# Octavia Amphora V2 + Jobboard 도입 — 구조 / 원리 / 테스트 케이스

> 대상 환경: DX-INDEV-MD (cluster-stack03) / OpenStack-Helm octavia (fork 2025.1.15-ktc)
> 브랜치: `T241INFRA6-1582-fix-ampv2`
> 작성 배경: V1(amphora) LB가 `PENDING_UPDATE` 에 박제되는 stuck 이슈를 amphorav2 + jobboard 로 구조적으로 해결

---

## 0. 한 줄 요약

V1 octavia는 failover 같은 작업(flow)의 상태를 **worker 메모리에만** 들고 있어서, worker가 flow 도중 죽으면 그 작업이 **증발**하고 LB가 `PENDING_UPDATE` 에 영구히 갇힌다.
amphorav2 + jobboard는 flow 상태를 **MariaDB(persistence) + Redis(jobboard)** 에 영속화해서, worker가 죽어도 **다른 worker가 job을 이어받아(resume)** 완료시킨다.

---

## 1. Before / After 아키텍처

### Before (V1 = amphora driver, jobboard 없음)

```
                  RPC cast (RabbitMQ)
openstack API  ───────────────────────►  octavia-worker
                                          ├─ failover flow 를 "프로세스 메모리"에서 실행
                                          └─ 상태 저장 X
                                          ✗ worker 죽으면 → flow 증발, 이어받을 주체 없음
                                          → LB provisioning_status = PENDING_UPDATE (영구)
                                          → health_manager: "immutable ... Skipping failover" 무한반복
                                          → 복구하려면 사람이 DB UPDATE 로 상태 강제 리셋
```

### After (V2 = amphorav2 driver + jobboard)

```
openstack API ──► Jobboard(Redis) ──► octavia-worker(conductor)
                  job 등록             ├─ job claim (TTL=30s, keepalive 갱신)
                  flow 상태는          ├─ flow 실행, 각 task 상태를
                  MariaDB(persistence) │   octavia-persistence DB 에 기록
                  에 영속              └─ ✗ worker 죽으면
                                          → claim TTL 만료
                                          → 다른 worker 가 job 재claim
                                          → persistence DB 의 마지막 상태부터 resume
                                          → LB 가 ACTIVE 로 정상 수렴 (사람 개입 불필요)
```

핵심 차이는 **"flow 상태가 어디에 사느냐"** 다.
- V1: worker 프로세스 메모리 (휘발성)
- V2: MariaDB + Redis (영속) → 그래서 복구 가능

---

## 2. Redis 구조 (ot-container-kit redis-operator)

genestack / vexxhost atmosphere 와 동일하게 **ot-container-kit(opstree) redis-operator** 를 사용한다.
`redis-systems` 네임스페이스에 3개 컴포넌트를 띄운다.

```
namespace: redis-systems
┌──────────────────────────────────────────────────────────────┐
│ redis-operator          (Deployment, replicas=1)              │
│   - RedisReplication / RedisSentinel CRD 를 watch & reconcile │
│   - master/replica role 라벨 관리                              │
├──────────────────────────────────────────────────────────────┤
│ RedisReplication "redis-replication" (clusterSize=3)          │
│   redis-replication-0  ← master  (redis-role=master)          │
│   redis-replication-1  ← replica                              │
│   redis-replication-2  ← replica                              │
│                                                               │
│   생성되는 Service:                                            │
│     redis-replication-master   ← 항상 현재 master pod 를 가리킴 │ ★ octavia 가 여기 연결
│     redis-replication-replica  ← replica 들                    │
│     redis-replication          ← headless                     │
├──────────────────────────────────────────────────────────────┤
│ RedisSentinel "redis-sentinel" (clusterSize=3, quorum=2)      │
│   - redis-replication 을 monitoring                           │
│   - master 장애 시 새 master 선출 (failover)                   │
│   - operator 가 그 결과로 master 라벨/서비스를 갱신            │
└──────────────────────────────────────────────────────────────┘
```

### 왜 이 3개가 다 필요한가

| 컴포넌트 | 역할 | 없으면 |
|---|---|---|
| **redis-operator** | CRD 해석 + 라이프사이클 관리 | RedisReplication/Sentinel CR이 아무 동작 안 함 |
| **RedisReplication** | master 1 + replica 2 (데이터 복제) | jobboard 데이터 저장소 자체가 없음 |
| **RedisSentinel** | master 장애 자동 failover + master 서비스 갱신 | master 죽으면 `redis-replication-master` 가 죽은 pod 가리킴 → jobboard 마비 |

### octavia 가 Redis 에 연결되는 경로

octavia 는 sentinel 프로토콜이 아니라 **`redis-replication-master` 서비스(:6379)** 에 직접 연결한다.
이 서비스가 "항상 현재 master 를 가리키도록" operator/sentinel 이 관리하므로, redis master 가 바뀌어도 octavia 입장에선 같은 주소로 계속 쓰면 된다. (HA 가 서비스 레이어에서 추상화됨)

```
octavia-worker
  └─ jobboard_backend_hosts = redis-replication-master.redis-systems.svc.<cluster-domain>
     jobboard_backend_port  = 6379
     (redis master 가 0→1 로 failover 돼도 이 주소는 그대로 → 새 master 를 가리킴)
```

### 우리 배포 파일 (stack-deployments-indev-md)

```
openstack-infra/
├── redis-operator/
│   ├── namespace.yaml                      # redis-systems 네임스페이스
│   ├── redis-operator-helmrelease.yaml     # operator (values 인라인 override)
│   └── kustomization.yaml
├── redis-replication/
│   ├── redis-replication-helmrelease.yaml  # clusterSize 3, local-path 1Gi, 평문
│   └── kustomization.yaml
├── redis-sentinel/
│   ├── redis-sentinel-helmrelease.yaml     # quorum 2, redisReplicationName=redis-replication
│   └── kustomization.yaml
└── apps/
    ├── redis-operator.yaml                  # ArgoCD Application (자동 discover)
    ├── redis-replication.yaml
    └── redis-sentinel.yaml
```

> **Phase 1 = 평문(no auth/TLS)**. octavia `endpoints.valkey.password: null` 과 일치.
> TLS 는 Phase 2 (redis Certificate + `jobboard_redis_backend_ssl_options`).

---

## 3. Jobboard 원리 (TaskFlow)

octavia 컨트롤러는 OpenStack `taskflow` 라이브러리로 작업을 **flow** 로 모델링한다.

```
flow (예: failover_amphora_flow)
 ├─ atom: GetAmphoraeNetworkConfigs
 ├─ atom: AmphoraePostNetworkPlug
 ├─ atom: ListenersUpdate
 └─ atom: MarkLBActiveInDB
```

jobboard 를 켜면 두 개의 영속 저장소가 붙는다:

| 저장소 | 무엇을 저장 | 우리 환경 |
|---|---|---|
| **Persistence backend** (MariaDB) | flow/atom 의 실행 상태 (logbook, flowdetails, atomdetails) | `octavia-persistence` DB |
| **Jobboard** (Redis) | "이 job 을 누가 claim 했는가" + job 메타 | `redis-replication-master` |

### 동작 모델 (conductor / claim)

```
1. API/health_manager 가 작업을 jobboard(redis)에 job 으로 POST
2. octavia-worker = "conductor" 가 jobboard 를 폴링하다 job 을 claim
   - claim 에는 TTL = jobboard_expiration_time(=30s)
   - 작업 중인 worker 는 keepalive 로 claim 을 계속 갱신 (조기 release 방지)
3. flow 를 실행하며 각 atom 상태를 persistence DB(MariaDB)에 기록
4-a. 정상 완료 → job 을 jobboard 에서 제거, LB = ACTIVE
4-b. worker 가 죽음 → keepalive 멈춤 → claim TTL 만료
     → 다른 conductor(worker)가 그 job 을 "abandoned" 로 보고 재claim
     → persistence DB 의 마지막 atom 상태부터 flow 를 resume
     → 완료 → LB = ACTIVE
```

이게 V1 에 없던 **"중단된 flow 의 인계(handoff)"** 메커니즘이다.

---

## 4. 무엇이 해결되는가 — 두 가지 failure mode 구분 (중요)

stuck 이슈를 정확히 이해하려면 **두 모드를 구분**해야 한다. jobboard 가 직접 고치는 건 Mode A 다.

| | **Mode A** | **Mode B** |
|---|---|---|
| 메커니즘 | controller(worker)가 flow **도중 사망** (revert도 못 돎) | flow 안의 task 가 **실패 → revert** (controller 는 생존) |
| V1 결과 | flow 증발 → **PENDING_UPDATE 영구 박제** | revert 가 LB 를 ERROR 로 떨궈야 하는데, revert 불완전/중단 시 PENDING 박제 |
| jobboard 효과 | **직접 해결** — 다른 worker 가 resume | **간접** — V2 revert 가 더 견고, 보통 ERROR 로 깔끔히 종료 (PENDING 안 갇힘) |
| 실제 그 incident(6f628ed7)? | ❌ (worker 9일 무재시작) | ✅ (AmphoraePostNetworkPlug revert, controller 생존) |

### 핵심 통찰

- **jobboard 는 "근본적으로 실패하는 task" 를 성공시키지 못한다.** (예: 백엔드 9443 이 진짜 막혀 있으면 V2 라도 결국 실패)
- 하지만 V2 의 실질 이득은 **"PENDING 에 안 갇히고 ERROR 로 떨어진다"** 는 것. ERROR 는 mutable 이라 사람이 DB 손 안 대고 `loadbalancer failover` 로 재시도 가능. V1 의 PENDING 박제는 immutable 이라 무조건 수동 DB reset 이 필요했다.
- 그 incident 의 진짜 1차 원인(listener 1/2 mismatch, 멤버 전원 ERROR = MKS NodePort 32124 백엔드 문제)은 octavia 밖의 문제이므로 **jobboard 와 별개로 따로 봐야** 한다.

---

## 5. 우리 환경 config — 무엇을 켰나 (정합성 검증 완료)

`octavia/common/config.py` 원본과 대조해 검증한 결과:

| 설정 | 값 | 검증 |
|---|---|---|
| `api_settings.default_provider_driver` | `amphorav2` | 기본 `amphora` → 변경 필요 ✅ |
| `api_settings.enabled_provider_drivers` | `amphorav2, amphora` | 기존 V1 LB 호환 위해 amphora 유지 ✅ |
| `task_flow.jobboard_enabled` | `true` | 기본 `false` ✅ |
| `task_flow.jobboard_backend_driver` | `redis_taskflow_driver` | `choices` 의 기본값, 유효 ✅ |
| `task_flow.jobboard_expiration_time` | `30` | claim TTL 기본값 ✅ |
| `task_flow.persistence_connection` | (미지정) | 차트가 `endpoints.oslo_db_persistence` 에서 자동주입 ✅ |
| `jobboard_backend_hosts/port/password` | (미지정) | 차트가 `endpoints.valkey` 에서 자동주입 ✅ |
| `octavia-persistence` DB + `upgrade_persistence` | 사전생성됨 | db-sync job 이 자동 마이그레이션 ✅ |

> **provider 는 LB 생성 시점에 고정**된다. 따라서 default 를 amphorav2 로 바꿔도:
> - 기존 V1 LB → 그대로 V1 유지 (영향 없음)
> - 신규 LB → amphorav2 로 생성
> 기존 LB 를 V2 로 옮기려면 별도 failover 마이그레이션 필요 (다운타임 검토).

### 변경 방식: base values.yaml 무수정, 전부 HelmRelease override

```
openstack/octavia/values.yaml            ← 무수정 (upstream 기준 유지)
openstack/octavia/octavia-helmrelease.yaml  ← spec.values.conf.octavia 에 override 추가
```
이미 사전작업으로 `octavia-persistence` DB·Grant, `oslo_db_persistence`/`valkey` endpoint,
`endpoints.valkey.port.server.sentinel: 6379` 까지 다 깔려 있어서, 실질 변경은 provider + jobboard **스위치 2개**.

---

## 6. 배포 순서 (중요)

octavia 가 **이미 떠 있는** 상태라 순서가 중요하다.

```
1) Redis 먼저 배포 (ArgoCD: redis-operator → redis-replication → redis-sentinel)
   확인: kubectl -n redis-systems get redisreplication,redissentinel
        kubectl -n redis-systems get svc | grep redis-replication-master   # 존재해야 함
2) Redis Ready 확인 후 octavia 변경 적용
   - 안 그러면 jobboard_enabled=true 인데 redis 없음 → worker CrashLoop
   확인: kubectl -n openstack logs job/octavia-db-sync | grep -i persistence
        kubectl -n openstack logs -l component=worker | grep -i jobboard
        openstack loadbalancer provider list   # amphorav2 보여야
```

미러링 필요 (내부 레지스트리):
- 차트: `redis-operator:0.24.0`, `redis-replication:0.17.0`, `redis-sentinel:0.16.12` (ot-helm)
- 이미지: `quay.io/opstree/redis-operator:v0.24.0`, `redis:v8.2.2`, `redis-sentinel:v8.2.2`

---

## 7. 테스트 케이스 (Before/After 비교)

> ⚠️ **반드시 테스트 LB 로만**. worker scale 0 은 그 순간 모든 LB 의 in-flight 작업에 영향 → 트래픽 적은 시점.
> 현재 토폴로지 `ACTIVE_STANDBY` → failover 가 amphora 2대 재생성 → flow 가 길어 중단 윈도우 넉넉.

### 사전: worker 가 Deployment 인지 DaemonSet 인지 확인

```bash
kubectl -n openstack get deploy,ds | grep octavia-worker
# fork 차트 postRenderer 가 'DaemonSet octavia-worker-default' 를 패치 → DaemonSet 일 가능성 높음
# DaemonSet 이면 scale 대신 delete pod 사용
```

### TC-1 — jobboard 핵심 (Mode A: controller 사망 중 flow)

**목적:** worker 가 flow 도중 죽어도 복구되는가

```bash
# 준비
LBID=$(openstack loadbalancer create --name jb-tc1 --vip-subnet-id <subnet> --wait -f value -c id)
openstack loadbalancer show $LBID -c provider     # BEFORE=amphora / AFTER=amphorav2

# 실행: failover 트리거 → 즉시 worker 전멸 → 복구
openstack loadbalancer failover $LBID
# (Deployment)
kubectl -n openstack scale deploy octavia-worker --replicas=0 && sleep 5 && \
kubectl -n openstack scale deploy octavia-worker --replicas=3
# (DaemonSet)
# kubectl -n openstack delete pod -l application=octavia,component=worker --force --grace-period=0

# 관찰
watch -n3 'openstack loadbalancer show '$LBID' -c provisioning_status -c operating_status'
kubectl -n openstack logs -l component=health_manager | grep -i "immutable\|Skipping failover"
kubectl -n openstack logs -l component=worker | grep -i "jobboard\|claim\|resum"
```

| 관찰 항목 | BEFORE (V1) | AFTER (V2+jobboard) |
|---|---|---|
| worker kill 후 LB 최종 상태 | **PENDING_UPDATE 영구 박제** | ~30s 후 **ACTIVE 자동복귀** |
| 자동 복구 시간 | ∞ (수동 DB reset 필요) | ~`jobboard_expiration_time`(30s)+α |
| health_manager 로그 | "Skipping failover" 반복 | 정상 |
| worker 로그 | (resume 없음) | job claim / resume |
| 사람 개입 | 필수 | 불필요 |

### TC-2 — incident 충실 (Mode B: task 실패 → revert)

**목적:** task 가 실패해 flow 가 revert 될 때 LB 가 PENDING 박제 vs ERROR 정상종료

```bash
LBID=$(openstack loadbalancer create --name jb-tc2 --vip-subnet-id <subnet> --wait -f value -c id)

# 새 amphora 의 9443(amphora-agent) 도달을 막아 AmphoraePostNetworkPlug 강제 실패
# amp_secgroup_list = 0ffd1159-a822-4135-b3ab-d367b32f29c9 의 9443 ingress 임시 삭제
openstack security group rule delete <9443-rule-id>

openstack loadbalancer failover $LBID    # plug 단계에서 connectivity 실패 → revert
watch -n3 'openstack loadbalancer show '$LBID' -c provisioning_status'
kubectl -n openstack logs -l component=worker | grep -iE "traceback|revert|FAILURE"

# 복구 후 반드시 9443 룰 원복!
```

| 관찰 항목 | BEFORE (V1) | AFTER (V2) |
|---|---|---|
| revert 후 LB 상태 | PENDING_UPDATE 박제 가능 (그 incident) | 보통 **ERROR 로 정상 종료** (PENDING 안 갇힘) |
| 재시도 가능성 | 불가 (immutable) | `loadbalancer failover` 로 재시도 가능 (단, 9443 막힌 채면 또 실패→ERROR) |

> TC-2 결론: jobboard 가 실패 task 자체를 고치진 못하지만, **PENDING 박제 → ERROR 종료** 로 바뀌어
> 수동 DB reset 없이 운영자가 재시도할 수 있게 된다 = 그 incident 의 실질 개선.

### 정리(cleanup) — V1 박제 LB 복구

```sql
-- octavia DB (현재값 SELECT 백업 후)
UPDATE load_balancer SET provisioning_status='ACTIVE' WHERE id='<LBID>';
UPDATE listener SET provisioning_status='ACTIVE' WHERE load_balancer_id='<LBID>';
UPDATE pool   SET provisioning_status='ACTIVE' WHERE load_balancer_id='<LBID>';
UPDATE member SET provisioning_status='ACTIVE'
  WHERE pool_id='<POOL_ID>' AND provisioning_status LIKE 'PENDING_%';
UPDATE amphora SET status='ALLOCATED' WHERE id IN ('<amp1>','<amp2>');
```
또는 `openstack loadbalancer delete <LBID> --cascade` (PENDING 이면 DB 에서 상태 푼 뒤 삭제).

---

## 8. 빠른 검증 체크리스트

```bash
# Redis
kubectl -n redis-systems get pods                                   # operator/replication x3/sentinel x3 Running
kubectl -n redis-systems get svc | grep redis-replication-master    # 서비스 존재
kubectl -n redis-systems exec redis-replication-0 -- redis-cli ping # PONG

# octavia jobboard 연결
kubectl -n openstack logs -l component=worker | grep -i jobboard    # 연결 로그
# persistence 스키마
mysql ... -e "SHOW TABLES IN \`octavia-persistence\`;"               # logbooks/flowdetails/atomdetails

# provider
openstack loadbalancer provider list                                # amphorav2 노출
openstack loadbalancer create ... && openstack loadbalancer show <id> -c provider  # amphorav2
```

---

## 참고 (검증 소스)

- Octavia install-amphorav2 (공식): https://docs.openstack.org/octavia/latest/install/install-amphorav2.html
- octavia/common/config.py (task_flow opts): https://github.com/openstack/octavia/blob/master/octavia/common/config.py
- Bug #2036952 — LB stuck in PENDING_UPDATE: https://bugs.launchpad.net/bugs/2036952
- Bug #2043360 — revert of LB flow may be incomplete: https://bugs.launchpad.net/octavia/+bug/2043360
- 레퍼런스: rackerlabs/genestack, vexxhost/atmosphere (둘 다 redis-systems + ot-container-kit redis-operator)
