# OpenStack Nova RPC 중단 — RabbitMQ Reply Queue 소실 근본원인 분석

> 본 문서는 **근본 원인**을 구성도(다이어그램) 중심으로 설명한다. 조치/테스트는 후반부.

| 항목 | 내용 |
|---|---|
| 리전 / 클러스터 | DX-G-GB / STACK |
| 서비스 | OpenStack Nova (2025.1, Genestack), RabbitMQ 3.13 (cluster-operator) |
| oslo.messaging | **16.1.0** (배포 확인) |
| 증상 | 인스턴스 BUILD 행(hang), 특정 compute 노드 RPC 전면 timeout |
| 표면 원인(원 보고서) | RabbitMQ policy로 reply queue exclusive=false + x-expires 만료 |
| **실제 근본원인** | **oslo `rabbit_quorum_queue: true` + 기본 transient TTL(1800s) → reply queue가 connection과 분리된 휘발성 큐 + oslo 자가복구 부재.** RabbitMQ policy는 무관(라이브 검증). |
| 취약 범위 | nova 한정 아님 — **9개 서비스 전부** 동일 구성 |
| 작성일 | 2026-05-28 |

---

## 0. TL;DR

1. oslo가 `rabbit_quorum_queue: true` 때문에 reply queue를 **`exclusive=false · auto_delete=false · x-expires=30분`** 인 classic 큐로 선언한다.
2. 이 플래그 조합은 큐 수명을 **connection·consumer와 분리**시킨다 → 주인 프로세스가 죽어도, 혹은 살아있어도 큐가 독립적으로 사라질 수 있다.
3. **좀비 큐**(양성)와 **노드 마비**(장애)는 *동일한 삭제 메커니즘*의 두 얼굴이다. 차이는 "삭제될 때 큐 주인이 살아서 응답을 기다리고 있었느냐" 뿐.
4. RabbitMQ **policy(`.*` quorum)는 무관** — 라이브에서 reply queue에 policy가 붙지도 않음을 확인. 원 보고서의 policy 수정안은 효과 0.
5. 조치는 oslo 4종 세트(`rabbit_quorum_queue` + `rabbit_transient_quorum_queue` + `use_queue_manager` + `rabbit_stream_fanout` = 모두 true). #2110957 버그는 16.1.0에 이미 fix 포함됨을 검증.

---

## 1. 시스템 구성도 (정상 동작)

### 1.1 OpenStack ↔ RabbitMQ RPC 토폴로지
```
   nova-api ──────┐
   nova-scheduler ┼──TLS 5671──▶  ┌──────────────────────────────────┐
   nova-conductor ┤               │      RabbitMQ cluster (3 nodes)    │
   nova-compute ──┘               │      vhost: nova                   │
                                  │   - RPC 큐 (quorum, durable)       │
                                  │   - reply_xxxxx (classic, 휘발성)  │  ← 문제의 큐
                                  │   - fanout 큐                      │
                                  └──────────────────────────────────┘
   ※ neutron/cinder/glance/keystone/manila/octavia/placement/barbican 도 동일 구조
```

### 1.2 RPC call/reply 정상 흐름
RPC `call`(응답 필요)은 호출자가 **자기만의 reply queue**를 만들어 거기로 응답을 받는다.
```
 nova-compute(호출자)              RabbitMQ                 conductor(처리자)
      │                              │                          │
      │ ① reply_AAA 선언 + consumer 등록(대기)                  │
      │─────────────────────────────▶│                          │
      │ ② call (reply_to=reply_AAA)  │                          │
      │─────────────────────────────▶│─────────────────────────▶│
      │                              │              ③ 처리       │
      │                              │◀─────────────────────────│
      │ ④ reply 를 reply_AAA 로 전달  │                          │
      │◀─────────────────────────────│                          │
      │ consumer 수신 → call() 반환   │                          │
```
정상 환경의 reply queue는 호출자 프로세스 생존 동안 유지되고, connection이 끊기면 같이 정리·재생성되어 **자가복구**된다. ← 우리 환경은 이게 깨진다.

---

## 2. 근본 원인 (핵심)

### 2.1 설정 → 큐 속성 인과 지도
어떤 GitOps 설정이 어떤 큐 속성을 만드는지의 인과 사슬:
```
[GitOps] openstack/<svc>/values.yaml  →  conf...oslo_messaging_rabbit:

   rabbit_quorum_queue: true ───────────────────────┐
   rabbit_transient_quorum_queue: false ──┐          │
   (rabbit_transient_queues_ttl 미설정)   │          │
            │                             │          │
            ▼                             ▼          ▼
   x-queue-type=classic 명시      reply queue 선언 방식:
   (vhost 기본 quorum 무시)        exclusive=false, auto_delete=false,
            │                      x-expires=1800000(30분), 발행 시 mandatory flag
            └──────────────┬───────────────┘
                           ▼
   reply queue = [ classic · exclusive=false · auto_delete=false · x-expires=30m ]
                           │
                           │  ◀── RabbitMQ Policy(.* quorum)는 여기 관여 안 함
                           │       (definition=target-group-size뿐 = quorum 전용,
                           │        classic 큐엔 무효. 라이브: reply 큐 policy 컬럼 ∅)
                           ▼
   "connection·consumer와 분리되어 오직 x-expires TTL 로만 청소되는 휘발성 큐"
                           │
                           ▼
              ★ 근본 원인의 씨앗 ★
```

### 2.2 큐 삭제 트리거 — 플래그가 결정한다
RabbitMQ에서 큐가 자동 삭제되는 조건은 플래그가 정한다. 라이브에서 본 우리 reply queue는 아래 표의 마지막 줄에 해당한다.

| 플래그 조합 | 삭제 트리거 | 자가복구성 | 우리 reply queue |
|---|---|---|---|
| `exclusive=true` | declaring connection 닫히면 **즉시 삭제** | 좋음(connection 종속) | ✗ |
| `auto_delete=true` | 마지막 consumer 떠나면 **즉시 삭제** | 보통 | ✗ |
| `exclusive=false`+`auto_delete=false`+`x-expires=N` | **consumer=0 + 미사용 N ms 후** 삭제 | **나쁨(독립 수명)** | ✓ **(현재)** |

→ 우리 큐는 connection이 끊겨도, consumer가 0이 되어도 **즉시 삭제되지 않고**, 오직 "consumer 없는 채 30분"이라는 TTL로만 삭제된다. 이 "독립 수명"이 모든 문제의 출발점이다.

### 2.3 왜 좀비 큐가 생기나 (양성 케이스)
주인이 **죽은** 큐가 TTL까지 남는 현상:
```
 health-probe.py (liveness/readiness, 노드당 ~90초 주기)
   │ 시작 → connection open → reply_BBB 선언 → call('ping') → 응답 수신
   │ 프로세스 종료
   ▼
 [종료 순간]
   connection close ──▶ (exclusive=false 이므로) ──╳ 삭제 안 됨
   consumer = 0     ──▶ (auto_delete=false 이므로) ─╳ 삭제 안 됨
                           │
                           ▼
              reply_BBB 가 "주인 없는 빈 큐"로 잔존  ← 좀비
                           │  30분 경과 (x-expires)
                           ▼
                     RabbitMQ가 삭제
   ⇒ 정상 상태에서도 항상 (probe빈도 × 30분 × 노드수) 만큼 누적  (관측: 1,760개)
```
**좀비는 양성**이다 — 주인이 이미 떠나 아무도 응답을 기다리지 않으므로, 큐가 남든 사라지든 RPC에 영향이 없다. 단지 메타데이터/메모리를 잠깐 점유할 뿐. 하지만 **이 삭제 메커니즘이 consumer=0 큐에 실제로 작동한다는 증거**다.

### 2.4 왜 노드가 마비되나 (장애 케이스)
주인이 **살아있는** 큐가 똑같이 삭제되는 현상:
```
 nova-compute (장기 프로세스, reply_CCC 를 사용 중)
   │
   │ (1차 트리거) 채널/IO blip 등으로 reply_CCC 의 consumer=0 이 됨
   │             그러나 TCP connection·heartbeat 는 정상  ← "반쪽 연결"
   ▼
 consumer=0 상태가 30분 지속
   │
   ▼  x-expires 만료  (← 좀비와 완전히 동일한 삭제 경로)
 RabbitMQ가 reply_CCC 삭제
   │
   ▼  그런데 connection은 살아있음
 oslo: reconnect 미발동 → reply_CCC 재선언 안 함 → 이름 캐시한 채 계속 사용
   │
   ▼
 conductor가 reply_CCC 로 응답 발행 → 큐 없음 → mandatory flag 로 반송
      "reply_CCC doesn't exist, drop reply" / "failed to send ... Abandoning"
   │
   ▼
 nova-compute: 보내는 모든 call 이 영구 60초 timeout
   │
   ▼
 노드 사실상 작동 불능 (Pod 재시작 전까지 자가복구 불가)
```
> ※ 1차 트리거(consumer가 0이 된 최초 원인 — 채널 cancel / 스토리지·IO stall 등)는 로그 보존 한계로 **미확정**. 그러나 트리거가 무엇이든, 그것을 "영구 장애"로 키운 것은 위의 구조(독립 수명 큐 + oslo 자가복구 부재)다.

### 2.5 좀비 vs 장애 — 같은 메커니즘, 다른 피해자
```
            consumer=0  →  x-expires(30m)  →  큐 삭제      ← 동일한 메커니즘
                 │                                  │
        ┌────────┴────────┐                ┌────────┴────────┐
        ▼                 ▼                ▼                 ▼
   주인 = 죽은 probe                   주인 = 살아있는 nova-compute
   응답 기다리는 이: 없음               응답 기다리는 이: 있음(계속)
        │                                  │
        ▼                                  ▼
   큐 삭제돼도 영향 0                  큐 삭제 = 모든 reply 유실
   (받을 사람이 없으니)                (주인이 살아서 기다리는데)
        │                                  │
        ▼                                  ▼
   메타데이터만 잠깐 점유              RPC 영구 timeout → 노드 마비
      = 좀비 (양성)                      = 빌드 행/장애
```
**차이는 메커니즘이 아니라 "삭제되는 순간 큐 주인이 살아서 응답을 기다리고 있었느냐" 단 하나.**

### 2.6 정리: 두 구조적 결함
- **결함 A — 큐 수명이 connection과 분리됨.** `exclusive=false`라 connection이 살아있어도 큐가 독립적으로 사라질 수 있다. (exclusive=true였다면 connection 종속이라 끊기면 같이 삭제·재생성→자가복구)
- **결함 B — oslo가 큐 소실을 감지·재선언 못 함.** connection이 살아있으면 reconnect를 안 하고 reply queue를 다시 declare하지 않는다. 사라진 큐로 계속 발행 → mandatory 반송. 이 자가복구 부재가 "재시작 전 영구 timeout"의 직접 원인.

### 2.7 보고서의 policy 귀속 정정 (라이브 검증)
```
$ rabbitmqctl list_queues -p nova name type exclusive auto_delete durable policy arguments
 reply_42ba...  classic  false  false  false   <∅>   [{"x-expires",1800000},{"x-queue-type","classic"}]
                type     excl   auto_d durable policy  arguments(= oslo 가 선언한 것)

$ rabbitmqctl list_policies -p nova
 nova  nova-quorum-three-replicas  .*  queues  {"target-group-size":3}  0
```
- **reply queue의 policy 컬럼이 비어 있음(∅)** → `nova-quorum-three-replicas`(`.*`)가 reply 큐에 **붙지도 않는다**. definition이 `target-group-size`뿐 = quorum 전용이라 classic 큐엔 무효.
- `exclusive=false`/`x-expires`/`x-queue-type=classic`은 전부 **oslo가 선언한 arguments**다.
> **결론: 원 보고서 6.1의 policy 패턴 변경(`^(?!reply_).*`)은 효과 0.** 붙지도 않는 policy의 매칭을 바꾸는 것이라 TTL 삭제 경로를 못 건드린다.

### 2.8 취약 범위 — 9개 서비스 전부
`rabbit_quorum_queue: true` + 기본 TTL + `use_queue_manager: false` 가 **nova/neutron/cinder/glance/keystone/manila/octavia/placement/barbican 전부 동일**. nova가 먼저 터진 건 `health-probe.py`가 노드 수만큼 reply 큐를 양산해 모집단·교체율이 압도적이기 때문(좀비 1,760개). 같은 트리거가 오면 다른 vhost도 동일하게 무너진다.

---

## 3. "빌드 행(hang)"과의 연관성 — 장애의 가시적 형태

### 3.1 cast 는 되고 call 이 막힌다
```
 scheduler ──cast(build_and_run_instance)──▶ nova-compute
            (cast = 응답 불필요)               │  빌드 "시작" 됨
                                               │  → server event 에 start_time 만 찍힘
                                               ▼
 [빌드 진행 중 필요한 RPC들 — 전부 call(응답 필요)]
   nova-compute ──call──▶ conductor : instance.save / BDM 조회 / object_action ...
                  │ reply 필요
                  ▼
            reply_CCC (이미 삭제됨) ──▶ 응답 영원히 못 받음 ──▶ 60s timeout 반복
                  │
                  ▼
   빌드가 상태를 DB에 영속화·완료보고 불가
                  │
                  ▼
   API 상태: host=None, task_state=None, vm_state=building  → 영구 정지 (finish_time 없음)
```
즉 빌드 행은 별개 문제가 아니라 **reply queue 소실의 증상**이다.

### 3.2 현재 빌드 행이 이 원인인지 진단
빌드 행은 neutron port binding / cinder / glance / placement 등 다른 원인도 있으므로 구분 필요.
```bash
# (a) 멈춘 인스턴스의 target compute
openstack server show <uuid> -f value -c OS-EXT-SRV-ATTR:host -c status -c OS-EXT-STS:task_state
# (b) 그 compute 의 RPC 가 죽었는지 (이 이슈의 지문)
kubectl logs -n openstack <nova-compute-pod-on-host> -c nova-compute --tail=300 \
  | grep -iE 'MessagingTimeout|Timed out waiting for a reply'
# (c) conductor 가 그 노드 reply 큐로 못 보내는지
kubectl logs -n openstack -l application=nova,component=conductor --tail=500 \
  | grep -iE 'MessageUndeliverable|missing queue|Abandoning|doesn.t exist, drop reply'
```
(b)(c)가 그 host에 대해 뜨면 → 이 이슈 → 즉시조치는 해당 nova-compute Pod 재시작. 깨끗하면 다른 원인.

---

## 4. 왜 5개월 잘 쓰다 이제야 터졌나 (latent → trigger)
구조는 첫날부터 취약했으나, 발사에는 드문 조건이 정렬돼야 했다.
```
 [상시] 좀비 큐 누적 = 5개월 내내 진행 중 (양성이라 미인지)
 [희박] 노드 마비 = 아래 4개가 겹쳐야 가시화
   1. 침묵성   : compute service 는 fanout 기반이라 conductor 엔 계속 'up' → 빌드 시도 전엔 모름
   2. 첫 인지  : 초기엔 잦은 배포/튜닝으로 Pod 재시작 → 발생해도 조용히 리셋됐을 것
   3. 긴 uptime: 안정화로 무재시작 가동 길어짐 → 큐가 30분 나쁜 상태로 방치될 환경 형성
   4. 트리거   : 스토리지/채널 blip(mpi3mr0 등)이 그 노드에서 처음 정렬
```
→ "최근 회귀"가 아니라, **충분한 노드-가동시간이 쌓여 tail event 가 처음 눈에 띈 것.**

---

## 5. 조치 — oslo 4종 세트 (공식 권장)

### 5.1 변경 (9개 서비스 `oslo_messaging_rabbit`)
| 옵션 | 현재 | 변경 |
|---|---|---|
| `rabbit_quorum_queue` | true | 유지 |
| `rabbit_transient_quorum_queue` | false | **→ true** |
| `use_queue_manager` | false | **→ true** |
| `rabbit_stream_fanout` | (없음) | **→ true** |

### 5.2 해결 원리 (before → after)
```
BEFORE: reply 큐 = classic · exclusive=false · x-expires=30m · per-call random 이름
        └ 주인이 살아있어도 큐가 사라질 수 있고(결함 A), oslo 가 재생성 못 함(결함 B)

AFTER : transient_quorum=true → reply/fanout 큐 = quorum(3노드 복제·durable)   ⇒ 결함 A 제거
        use_queue_manager=true → 프로세스당 안정적 큐 풀 + 재선언 관리          ⇒ 결함 B 제거
        stream_fanout=true     → fanout = stream 큐
        └ 노드/채널 blip 에도 큐 보존 + manager 가 재선언 → 자가복구, 좀비 양산도 멈춤
```

### 5.3 16.1.0 안전성 (필수 검증 결과)
- `use_queue_manager: true` 엔 동반 버그 **#2110957**(queue manager+quorum 에서 RPC reply 가 엉뚱한 스레드로 → `No calling threads waiting` → 전 서비스 timeout). 공식 워크어라운드가 `use_queue_manager=false`(=현재 상태).
- fix(Change-Id `I8a31c0c…`)는 master 2025-02-16 머지. 배포된 **16.1.0** 에 대해 `compare 16.1.0...d3a4eedb8a14 → ahead_by=0` = **fix 포함 확인**.
- → 경로 viable. 단 부하 잔존 레이스 가능성 있어 **부하 회귀 테스트 필수**.

---

## 6. 개발환경 테스트
목표: (1) 현재 구성에서 장애 재현 → (2) 4종 세트 적용 후 미재현 → (3) #2110957 회귀 없음(부하).

### 6.1 테스트 A — 격리 oslo.messaging 미니 재현 (권장 1차)
1. dev RabbitMQ에 `nova`와 동일 vhost(`defaultQueueType: quorum`)+policy 생성.
2. 최소 oslo.messaging RPC server/client 작성. `rabbit_transient_queues_ttl=60`(가속).
3. **재현**: server 기동 → client `call('ping')` 1회 정상 → server의 reply 큐를 강제 삭제
   `rabbitmqctl delete_queue -p <vhost> reply_<uuid>` (=만료 후 상태) → 다시 `call` → **MessagingTimeout + MessageUndeliverable** 재현.
4. server 재시작 → 복구.

### 6.2 테스트 B — dev OpenStack E2E
1. dev nova values `rabbit_transient_queues_ttl: 60` + 4-set off.
2. nova-compute 1대 reply 큐 `rabbitmqctl delete_queue` 삭제.
3. 그 노드 주기태스크 timeout + conductor MessageUndeliverable 확인. (선택) 그 노드로 인스턴스 스케줄 → **BUILD 행 재현**.

### 6.3 fix 검증
```bash
rabbitmqctl list_queues -p <vhost> name type durable | grep -E '^reply_|^fanout'
# 기대: type=quorum, durable=true, queue-manager 네이밍
```
- 6.1/6.2의 "큐 강제 삭제" 재시도 → **재시작 없이 RPC 자동 복구** = 핵심 합격 신호.

### 6.4 회귀(#2110957) 부하 테스트 — 필수
```bash
# 동시 다발 RPC(인스턴스 생성/삭제, neutron port 대량 생성 등) 중:
kubectl logs -n openstack -l application=nova --tail=500 \
  | grep -iE 'No calling threads waiting|MessagingTimeout' || echo "clean"
```
cinder-api↔scheduler, neutron-metadata↔server 등 cross-service 도 확인(이 버그가 특히 그쪽에서 발생).

### 6.5 합격 기준
| # | 항목 | 기대 |
|---|---|---|
| 1 | reply/fanout 큐 type | quorum, durable, manager 네이밍 |
| 2 | 큐 강제 삭제 후 | **재시작 없이** 자동 복구 |
| 3 | 부하 중 `No calling threads waiting` | 0건 |
| 4 | 부하 중 cross-service RPC timeout | 0건 |
| 5 | consumer=0 reply 큐 누적 | 멈춤/감소 |
| 6 | (E2E) 인스턴스 BUILD 완료 | 정상 |

---

## 7. 좀비 큐 스캔 (다른 환경 점검)
```bash
NS=openstack
RMQ=$(kubectl get pods -n $NS -l app.kubernetes.io/component=rabbitmq -o name | head -1)
for V in $(kubectl exec -n $NS $RMQ -c rabbitmq -- rabbitmqctl list_vhosts --no-table-headers); do
  kubectl exec -n $NS $RMQ -c rabbitmq -- \
    rabbitmqctl list_queues -p "$V" name consumers --no-table-headers 2>/dev/null \
  | awk -v v="$V" '$1 ~ /^reply_/ {t++; if($2==0) z++}
                   END {printf "%-12s reply_total=%-6d zombie(consumer=0)=%-6d\n", v, t+0, z+0}'
done
```
zombie 가 0이 아니면 그 환경도 동일 취약 구조. (좀비 자체는 양성 — §2.3. 실제 위험은 §3.2 로그로 판별.)

## 8. 롤아웃 (canary)
1. **nova 1개 서비스만** 먼저(별도 브랜치) → Flux → 검증(§6.3·6.4) → 나머지 8개.
2. 현재 브랜치 `revert/T241INFRA6-1346-...`는 무관 — 전용 브랜치 사용.

## 9. 라이브 증거 / 참고
- `list_policies -p nova` → `{"target-group-size":3}`뿐 / `list_queues ... reply_*` → classic·exclusive=false·**policy ∅**·x-expires=1800000
- `pip show oslo.messaging` → **16.1.0** / gerrit `change:941996` MERGED, master commit `d3a4eedb8a14`(2025-02-16) / `gh api compare/16.1.0...d3a4eedb8a14` → `ahead_by=0`
- Bug #2110957: https://bugs.launchpad.net/oslo.messaging/+bug/2110957
- oslo.messaging 2025.1 RabbitMQ guide: https://docs.openstack.org/oslo.messaging/2025.1/admin/rabbit.html
- gerrit 941996: https://review.opendev.org/c/openstack/oslo.messaging/+/941996

---

## 부록 A — 운영 로그 시그니처 (Loki, 2026-05-25/26 KST)

본 incident 의 로그 증거. Loki 쿼리 → conductor 컨테이너의 oslo.messaging ERROR + 같은 시간대 nova-compute 의 periodic_task ERROR 가 1:1 짝을 이루고 모두 **동일한 reply queue 하나**(`reply_2facd0a5f40c4021867af6e4ceb5bc24`)를 가리킨다. = §2.4 outage chain 의 실측 재현.

### A.1 쿼리
```logql
{service_name="nova", level="ERROR"} != `ERROR neutron.agent.ovn.metadata.server_socket`
```
시간창 2026-05-25 23:02 ~ 2026-05-28 16:27 KST · Total: 6.45K · log_volume = error.

### A.2 두 줄짜리 시그니처 (이 한 쌍이 이 incident 의 지문)
publish 측(conductor) — reply 큐가 사라져 반송:
```
ERROR oslo_messaging._drivers.amqpdriver
  The reply <msg_id> failed to send after 60 seconds due to a missing queue
  (reply_2facd0a5f40c4021867af6e4ceb5bc24). Abandoning...
  : oslo_messaging.exceptions.MessageUndeliverable
```
consumer 측(nova-compute) — 응답 영원히 못 받아 timeout:
```
ERROR oslo_service.periodic_task
  Error during ComputeManager.<task>:
  oslo_messaging.exceptions.MessagingTimeout: Timed out waiting for a reply
  to message ID <msg_id>
```
주기적 task 가 줄줄이 같은 패턴으로 실패: `_instance_usage_audit` / `_sync_power_states` / `_reclaim_queued_deletes` / `update_available_resource` / `_run_pending_deletes` / `_check_instance_build_time`.

### A.3 로그가 확정하는 사실
| 관측 | 의미 |
|---|---|
| 모든 `MessageUndeliverable` 가 **같은 reply queue** `reply_2facd0a5...` 를 가리킴 | 죽은 큐는 **단 한 개** — 한 노드(=cnode003-itl) 의 한 reply queue 가 모든 timeout 의 진원 |
| 죽은 큐 주인 = nova-compute **PID 22555**, `req-e29ce3ba-431a-4ca6-83f6-49c9f41f26d8` | 보고서가 지목한 cnode003-itl PID 22555 와 일치. 큐 주인이 **살아있는 장기 프로세스**임이 로그로 확정 (= §2.5 의 "주인 살아있는" 케이스) |
| conductor 다중 worker(PIDs 7·8·9·10·11·12·22) 가 **모두** 같은 dead 큐로 reply 시도 | 어느 conductor 가 처리하든 결국 그 compute 의 단일 reply 큐로 보내야 함 → conductor 측 문제 아님이 확정 |
| 1분 주기로 다른 periodic task 가 timeout | compute 의 모든 RPC 가 마비된 "전면" 상태 (특정 호출만 깨진 게 아님) |
| 60초 timeout → Abandoning | `direct_mandatory_flag` 의 의도된 mandatory 반송 동작 (impl_rabbit.py:1713 + amqpdriver.py:229-234) |

### A.4 우리 분석과의 1:1 매핑
| 분석 단계 (§2.4) | Loki 로그 |
|---|---|
| reply_CCC 가 삭제됨 | conductor: `missing queue (reply_2facd0a5...)` |
| connection 살아있어 oslo 재선언 안 함 | 같은 큐 이름으로 **분 단위 무한 재시도**(이름 캐시) |
| conductor mandatory 반송 | `MessageUndeliverable ... Abandoning` |
| compute 영구 timeout | `MessagingTimeout` × periodic task 전부 |

### A.5 알람·탐지용 시그니처 (재발 조기 탐지)
```logql
# (P1) 죽은 reply 큐 발견 — 가장 강한 신호
{service_name="nova", logger="oslo_messaging._drivers.amqpdriver", level="ERROR"}
  |~ "failed to send after .* due to a missing queue \\(reply_"

# (P2) 같은 compute 의 동시다발 periodic task timeout
{service_name="nova", logger="oslo_service.periodic_task", level="ERROR"}
  |~ "MessagingTimeout: Timed out waiting for a reply"
```
권장 알람: (P1) 동일 `reply_<id>` 가 **3분 내 5건 이상** = 노드 영구 마비 확정 신호 → 해당 reply_id 의 주인 nova-compute Pod 재시작(즉시조치) + 4종 세트 적용 진행.

