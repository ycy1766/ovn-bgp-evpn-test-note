# v7 재테스트 런북 — Type-2 FIP BGP-EVPN (현재 빌드 기준)

[05](./05-type2-fip-evpn-test-runbook.md)의 절차를 **현재(v7) 빌드**로 다시 돌리기 위한 런북. 이미지·스키마·검증 부분만 v7 기준으로 갱신했고, 호스트 인터페이스/FRR/OVN 활성화 같은 **이미지 무관 단계는 05를 그대로 따른다**.

- **대상 이미지**: `docker.io/ycy1766/kube-ovn:v1.15.11-evpn-fip-type2-v7`
  digest `sha256:ddd150e0d38a9add600892c5d7ca8174093fabdc2ba854947b2ca5cc64f53e42` (레지스트리 게시 완료)
- **OVN 코드**: `kt-cloud-stack/ovn @ a4f6e9c46` (v7 = upstream/main 위 3커밋), OVN 26.03.90
- **PR**: <https://github.com/kt-cloud-stack/ovn/pull/1>

---

## 0. v7가 v5 이미지와 다른 점 (재테스트 전 필독)

| 항목 | v5 이미지 (`-v5`) | **v7 이미지 (`-v7`)** |
| --- | --- | --- |
| SB 스키마 | 21.9.0 **+ `type` 컬럼** | **21.9.0 (type 없음 = upstream 동일)** |
| `Advertised_MAC_Binding` | `type=ip\|nat` 컬럼 존재 | **`type` 컬럼 없음** |
| ovn-controller 광고 로직 | type별 게이팅 | **있는 행 전부 Type-2 neighbor 광고, `fdb`만 별도** |
| 기능 결과 (FIP Type-2) | 동작 | **동일하게 동작** |

> 코드 경로만 정리됐고 **FIP가 Type-2로 광고되는 최종 동작은 동일**하다. 검증 절차에서 `type` 컬럼 조회만 빠진다.

### ⚠️ 스키마 다운그레이드 주의 (v5 → v7 in-place 업그레이드 시)

v5(21.9.0**+type**) → v7(21.9.0**−type**)은 **같은 버전번호·다른 내용**이라 ovsdb-server가 자동 변환을 안 할 수 있다.
- **예상**: ovn-central은 기존 SB DB를 그대로 로드, v7 northd/controller는 `type`을 안 읽으므로 잔여 컬럼은 무해 → 정상 동작 가능성 높음.
- **ovn-central이 스키마 불일치로 crash/refuse 시** leader에서 명시 변환:
  ```bash
  kubectl -n kube-system exec -it <ovn-central-leader> -c ovn-central -- \
    ovsdb-tool convert /etc/ovn/ovnsb_db.db /kube-ovn/ovn-sb.ovsschema
  ```
  (RAFT면 leader 변환 후 follower 재동기화. 경로는 이미지 내 실제 위치로 확인.)
- **깨끗하게 가려면**: 재테스트 랩이면 v7로 먼저 배포하고 OVN DB를 새로 올리는 게 가장 단순. NB는 스키마 변경 없음(논리 토폴로지 보존).

---

## 1. 환경 변수 (한 번 설정)

랩에 맞게 채운다. 아래는 pl-cyyoon04 검증값(05 기준).
```bash
export IMG_TAG=v1.15.11-evpn-fip-type2-v7
export VNI=10
# 노드 services IP (VTEP/BGP)
export GW_IP=172.16.1.160          # gw, ASN 65000
export C1_IP=172.16.1.157          # oscompt01, ASN 65001
export C2_IP=172.16.1.159          # oscompt02, ASN 65001
# provider LS (external net 매핑) — kubectl-ko nbctl show 로 확인
export LS=neutron-73dbaf22-d6f7-44c7-82cf-42411c61c71c
```

---

## 2. 이미지 배포

### 2.1 helm override 태그를 v7로
`/etc/genestack/helm-configs/kube-ovn/kube-ovn-helm-overrides.yaml`:
```yaml
global:
  registry:
    address: docker.io/ycy1766
  images:
    kubeovn:
      repository: kube-ovn
      vpcRepository: vpc-nat-gateway
      tag: v1.15.11-evpn-fip-type2-v7      # ← v7
      support_arm: true
      thirdparty: true
```

### 2.2 배포 + 버전/스키마 확인
```bash
bash /opt/genestack/bin/install-kube-ovn.sh

# OVN 버전
kubectl-ko nbctl --version | head -1          # ovn-nbctl 26.03.90

# 이미지 반영
kubectl -n kube-system get ds ovs-ovn ovn-central -o jsonpath='{..image}{"\n"}' | tr ' ' '\n' | sort -u

# ovn-central 정상 + 스키마 (0절 주의)
kubectl -n kube-system get pod -l app=ovn-central          # 전부 Running
kubectl-ko sbctl --columns=_uuid,ip,mac,logical_port list Advertised_MAC_Binding
#  → type 컬럼이 조회에 안 나오면 v7 스키마 반영된 것 = 정상
```

---

## 3. kube-ovn 사전작업 (필수, 05 Phase 2와 동일)

```bash
# (1) ovs-ovn securityContext: runAsUser:0, privileged:true
kubectl -n kube-system edit daemonset ovs-ovn      # containers[name=openvswitch].securityContext

# (2) ovn-central 노드 host path 소유자
for node in 10.21.1.21 10.21.1.22 10.21.1.23; do
  ssh $node 'chown -R nobody:nogroup /var/run/ovn /var/log/ovn'
done

# (3) compute ovs-ovn pod 강제 재기동 (좀비 ovn-controller 방지)
for n in <compute-node-1> <compute-node-2>; do
  kubectl -n kube-system delete pod -l app=ovs-ovn --field-selector spec.nodeName=$n
done

# 확인
kubectl -n kube-system exec ds/ovs-ovn -c openvswitch -- id   # uid=0(root)
```

---

## 4. EVPN 인프라 — **05를 그대로 따름** (이미지 무관)

아래 단계는 v5/v7 동일하므로 [05](./05-type2-fip-evpn-test-runbook.md)의 해당 Phase를 그대로 수행:

- **05 Phase 3** — 커널 VRF 모듈 (`linux-modules-extra`, ip_forward) : gw + 모든 compute
- **05 Phase 4** — 테스트 VM (provider 직결 1대 + tenant+FIP 1대)
- **05 Phase 5** — 호스트 인터페이스 트리오 (`br-$VNI` / `vxlan-$VNI` / `lo-$VNI`), gw `/32` 라우트
- **05 Phase 6** — FRR (gw bgp 65000 / compute bgp 65001, `frr defaults datacenter`, `advertise-all-vni`)
- **05 Phase 7** — `vtysh -c 'show bgp l2vpn evpn summary'` / `show evpn vni` 세션 확인

> 핵심 함정(05와 동일): `br-$VNI`에 IP 금지, compute vxlan `nolearning`, `frr defaults datacenter`, provider LS에 localnet 포트 필수, ping src는 services IP 명시.

---

## 5. OVN Native EVPN 활성화

```bash
# provider LS에 dynamic-routing 키
kubectl-ko nbctl set Logical_Switch $LS \
  other_config:dynamic-routing-vni=$VNI \
  other_config:dynamic-routing-bridge-ifname=br-$VNI \
  other_config:dynamic-routing-vxlan-ifname=vxlan-$VNI \
  other_config:dynamic-routing-advertise-ifname=lo-$VNI
kubectl-ko nbctl set Logical_Switch $LS \
  other_config:dynamic-routing-redistribute=fdb,ip,nat

# compute OVS external_ids (ovn-evpn 포트 생성)
#  oscompt01
ovs-vsctl set Open_vSwitch . external-ids:ovn-evpn-local-ip=$C1_IP external-ids:ovn-evpn-vxlan-ports=4789
#  oscompt02
ovs-vsctl set Open_vSwitch . external-ids:ovn-evpn-local-ip=$C2_IP external-ids:ovn-evpn-vxlan-ports=4789
```
- `ip`=VIF/router-port, `nat`=분산 FIP(dnat_and_snat), `fdb`=FDB inject. **FIP는 `nat` 토큰 필수**.

---

## 6. v7 검증

### 6.1 SB advertised_mac (v7: type 컬럼 없음)
```bash
kubectl-ko sbctl find advertised_mac
#  ip / mac / logical_port / datapath 만 (type 컬럼 없음)
#  → FIP external_ip/mac 행 + provider 직결 VM 행 등장
```

### 6.2 호스트 FDB/neighbor (FIP 호스팅 chassis)
```bash
ip neigh show dev br-$VNI | grep <FIP>            # Type-2 MAC+IP neighbor
bridge fdb show dev vxlan-$VNI | grep <FIP-MAC>   # FDB
```

### 6.3 FRR EVPN
```bash
vtysh -c "show evpn mac vni $VNI" | grep <FIP-MAC>
vtysh -c 'show bgp l2vpn evpn' | grep <FIP>       # Type-2 (RT-2)
```

### 6.4 데이터 평면 (gw에서)
```bash
ping -I $GW_IP <direct-VM-IP> -c 5     # ttl=64
ping -I $GW_IP <FIP> -c 5              # ttl=63 (NAT 1홉), 0% loss 기대
```
✅ FIP ttl=63 / direct ttl=64, 둘 다 0% loss → **v7로 Type-2 FIP 광고 재현 완료**.

---

## 7. cleanup (재테스트 반복 시, 05 Phase 5.1 / 8.1)

```bash
# OVN 키 제거
for k in dynamic-routing-vni dynamic-routing-bridge-ifname dynamic-routing-vxlan-ifname \
         dynamic-routing-advertise-ifname dynamic-routing-redistribute; do
  kubectl-ko nbctl remove Logical_Switch $LS other_config $k 2>/dev/null
done
# compute OVS external_ids 제거 + recompute
ovs-vsctl remove Open_vSwitch . external-ids ovn-evpn-local-ip
ovs-vsctl remove Open_vSwitch . external-ids ovn-evpn-vxlan-ports
ovn-appctl -t ovn-controller inc-engine/recompute
# 호스트 인터페이스/라우트 삭제 (05 Phase 5.1)
```

---

## 부록 A — v7 이미지 빌드 재현 (이미 푸시됨, 필요 시만)

```bash
cd ~/Documents/git/ycy1766/kube-ovn
export V=v1.15.11-evpn-fip-type2-v7
# Dockerfile.base의 OVN clone은 a4f6e9c46 핀, OVS 핀 bdb95cc… (수정 불필요)

# (1) base (QEMU amd64, OVS+OVN 컴파일, 수십분)
docker buildx build --platform linux/amd64 --build-arg ARCH=amd64 \
  -t ycy1766/kube-ovn-base:$V -o type=docker -f dist/images/Dockerfile.base dist/images/
# (2) go 바이너리 (호스트 크로스컴파일)
VERSION=$V make build-go
# (3) 최종 이미지
docker buildx build --platform linux/amd64 -t ycy1766/kube-ovn:$V \
  --build-arg VERSION=$V -o type=docker -f dist/images/Dockerfile dist/images/
# (4) push
docker push ycy1766/kube-ovn:$V
```
> base 태그는 `-amd64` 접미사 없이(메인 Dockerfile `FROM …:$BASE_TAG`와 맞춤). 브랜치명이 v6와 같아 Dockerfile.base가 SHA(`a4f6e9c46`) checkout으로 캐시 버스트.

## 부록 B — 참고

- 전체 상세 절차: [05-type2-fip-evpn-test-runbook.md](./05-type2-fip-evpn-test-runbook.md)
- v7 코드 PR: <https://github.com/kt-cloud-stack/ovn/pull/1>
- upstream 리뷰(ovs-dev v6 → v7 반영): patchwork series 509976, Ales Musil 리뷰 3코멘트 모두 v7 반영(type 제거 + controller 단순화 + 핸들러 분리).
