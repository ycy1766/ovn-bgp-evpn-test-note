# Type-2 FIP BGP-EVPN 테스트 런북 (pl-cyyoon04)

OVN Native BGP-EVPN으로 OpenStack Floating IP / provider-direct VM을 **EVPN Type-2 (MAC+IP)** 로 광고하는 테스트의 전체 절차.

- **목적**: kube-ovn 기본 OVN(25.03) 대신 **OVN 26.03 이상**을 넣은 커스텀 이미지를 배포하고, provider LS에 `dynamic-routing` 키를 줘서 FIP가 EVPN Type-2로 광고되는지 gw↔compute FRR로 검증.
- **환경**: pl-cyyoon04 (genestack, kube-ovn 1.15.11, FRR-on-FRR / 외부 EVPN peer 없음).
- **이미지**: `docker.io/ycy1766/kube-ovn:v1.15.11-evpn-fip-type2-v5` (OVN 26.03.90). 변경 내용 PR: <https://github.com/kt-cloud-stack/ovn/pull/1>
  - 최신 이미지 태그는 갱신될 수 있다 (예: `-v7`). helm override의 tag만 교체하면 된다. 빌드 절차는 별도 노트 참고.

> 본 문서는 이미지를 바꿔 끼우고 이전 테스트 잔재(호스트 인터페이스 / 라우트 / OVN 키)를 지우면서 반복 검증할 수 있게 **정리(cleanup) → 구성 → 검증** 순으로 묶었다. IP/인터페이스 이름은 pl-cyyoon04 한정 값이다.

## 토폴로지 / 주소

| 역할 | 노드 | services IP (VTEP/BGP) | ASN |
| --- | --- | --- | --- |
| gw (EVPN 종단, ToR 시뮬) | kdvmd-pl-cyyoon04-gw | 172.16.1.160 | 65000 |
| compute | kdvmd-pl-cyyoon04-oscompt01 | 172.16.1.157 | 65001 |
| compute | kdvmd-pl-cyyoon04-oscompt02 | 172.16.1.159 | 65001 |

- VNI=10, provider subnet `172.16.1.0/24`, FIP/direct pool `172.16.1.151~156`, default GW `172.16.1.254`.
- 테스트 VM: `region01-vm1`(tenant net + **FIP 172.16.1.156**), `test-evpn-vm-direct`(provider 직결 **172.16.1.153**).

---

## Phase 1 — 커스텀 OVN 이미지 배포

### 1.1 helm override 태그 교체
`/etc/genestack/helm-configs/kube-ovn/kube-ovn-helm-overrides.yaml`:
```yaml
global:
  registry:
    address: docker.io/ycy1766
  images:
    kubeovn:
      repository: kube-ovn
      vpcRepository: vpc-nat-gateway
      tag: v1.15.11-evpn-fip-type2-v5     # ← 커스텀 OVN 26.03 이미지
      support_arm: true
      thirdparty: true
networking:
  IFACE: "tun"
  ENABLE_SSL: true
  OVN_NORTHD_N_THREADS: 2
```

### 1.2 배포 + OVN 버전 확인
```bash
bash /opt/genestack/bin/install-kube-ovn.sh

kubectl-ko nbctl --version
#  ovn-nbctl 26.03.90          ← 버전 확인 (핵심)
#  Open vSwitch Library 3.7.2
#  DB Schema 7.18.0
```

---

## Phase 2 — kube-ovn 사전작업 (필수)

ovn-controller(ovs-ovn) Pod가 각 호스트의 Netlink로 static FDB(RTM_NEWNEIGH)를 주입해야 하므로 권한을 열어준다. Type-5 테스트에도 동일 적용.

### 2.1 ovs-ovn securityContext
```bash
kubectl -n kube-system edit daemonset ovs-ovn
```
```yaml
spec:
  template:
    spec:
      containers:
      - name: openvswitch
        securityContext:
          runAsUser: 0
          privileged: true
```

### 2.2 소켓/로그 경로 소유자 (모든 ovn-central 노드)
```bash
for node in 10.21.1.21 10.21.1.22 10.21.1.23; do
  ssh $node 'chown -R nobody:nogroup /var/run/ovn /var/log/ovn'
done
```

### 2.3 확인
```bash
kubectl -n kube-system exec ds/ovs-ovn -c openvswitch -- id
#  uid=0(root) gid=0(root) groups=0(root)
kubectl -n kube-system get pod -l app=ovn-central     # 전부 Running
```
> ⚠️ rolling update만으로는 ovn-controller가 좀비로 남아 OF flow 미설치될 수 있다. 사전작업 마지막에 compute의 `ovs-ovn` pod를 강제 재기동(`kubectl -n kube-system delete pod -l app=ovs-ovn --field-selector spec.nodeName=<node>`)하는 것을 권장.

---

## Phase 3 — 커널 VRF 모듈 (gw + 모든 compute 동일)

Ubuntu 기본 커널엔 vrf 모듈이 없어 `linux-modules-extra` 설치 필요.
```bash
modprobe vrf vxlan        # FATAL: Module vrf not found → extra 설치 필요
apt update && apt install -y linux-modules-extra-$(uname -r)

echo vrf   | tee /etc/modules-load.d/vrf.conf
echo vxlan | tee /etc/modules-load.d/vxlan.conf

sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward=1' | tee /etc/sysctl.d/99-evpn.conf

# 확인
lsmod | egrep 'vrf|vxlan'         # vrf 로드됨
sysctl net.ipv4.ip_forward        # = 1
```

---

## Phase 4 — 테스트 환경 (OpenStack)

```bash
# tenant network/subnet
openstack network create --availability-zone-hint az1 region01-test-network-az1
openstack subnet create --network region01-test-network-az1 --gateway 192.168.20.1 \
  --subnet-range 192.168.20.0/24 region01-test-subnet-az1

# provider network (172.16.1.151~156)
openstack network create --share --external --availability-zone-hint az1 \
  --provider-physical-network physnet1 --provider-network-type flat internal-provider-network
openstack subnet create --network internal-provider-network \
  --allocation-pool start=172.16.1.151,end=172.16.1.156 \
  --dns-nameserver 8.8.4.4 --gateway 172.16.1.254 \
  --subnet-range 172.16.1.0/24 internal-provider-subnet

# router + external gw
openstack router create --availability-zone-hint az1 test-router-az1
openstack router add subnet test-router-az1 region01-test-subnet-az1
openstack router set --external-gateway internal-provider-network test-router-az1

# SG
openstack security group create region01-test-sg
openstack security group rule create --proto icmp region01-test-sg
openstack security group rule create --proto tcp  region01-test-sg

# VM (1) tenant + FIP, (2) provider 직결
openstack server create --flavor ktc-m1.tiny --network region01-test-network-az1 \
  --image cirros --security-group region01-test-sg --availability-zone az1 region01-vm1
openstack server create --flavor ktc-m1.tiny --image cirros \
  --network internal-provider-network \
  --availability-zone az1:kdvmd-pl-cyyoon04-oscompt01 test-evpn-vm-direct

# FIP → region01-vm1
openstack floating ip create internal-provider-network        # → 172.16.1.156
openstack server add floating ip region01-vm1 172.16.1.156
```
결과: `test-evpn-vm-direct`=172.16.1.153(provider 직결), `region01-vm1`=192.168.20.121 + FIP 172.16.1.156.

---

## Phase 5 — 호스트 인터페이스 (gw / compute)

### 5.1 이전 테스트 잔재 삭제 (모든 노드)
```bash
VNI=10
ip link del vxlan-${VNI} 2>/dev/null
ip link del lo-${VNI}    2>/dev/null
ip link del br-${VNI}    2>/dev/null
# gw: VM IP를 br-10으로 보내던 /32 라우트 제거
for ip in 151 152 153 154 155 156; do ip route del 172.16.1.${ip}/32 dev br-10 2>/dev/null; done
```

### 5.2 인터페이스 트리오 (br-10 / vxlan-10 / lo-10)
- **br-10**: L2 bridge. lo-10 + vxlan-10을 같은 VNI 도메인으로 묶음. **IP 부여 금지** (services NIC 172.16.1.0/24와 충돌).
- **lo-10**: dummy. ovn-controller가 `advertised_mac`의 VM MAC을 RTM_NEWNEIGH로 여기에 static FDB 주입.
- **vxlan-10**: remote VTEP 학습. compute는 `nolearning` (FRR static FDB와 자가학습 충돌 회피).

**gw** (172.16.1.160, vxlan dstport 4789):
```bash
VNI=10; LOCAL_IP=172.16.1.160
ip link add br-10 type bridge && ip link set br-10 up
ip link add vxlan-10 type vxlan id ${VNI} local ${LOCAL_IP} dstport 4789
ip link set vxlan-10 master br-10 && ip link set vxlan-10 up
ip link add lo-10 type dummy
ip link set lo-10 master br-10 && ip link set lo-10 up
# VM IP(151~156)만 br-10으로. VTEP(157/159)·default GW는 services 그대로.
for ip in 151 152 153 154 155 156; do ip route add 172.16.1.${ip}/32 dev br-10 2>/dev/null; done
```

**oscompt01** (172.16.1.157) / **oscompt02** (172.16.1.159) — vxlan dstport **60010 nolearning**:
```bash
VNI=10; LOCAL_IP=172.16.1.157          # oscompt02는 159
ip link add br-10 type bridge && ip link set br-10 up
ip link add vxlan-10 type vxlan id ${VNI} local ${LOCAL_IP} dstport 60010 nolearning
ip link set vxlan-10 master br-10 && ip link set vxlan-10 up
ip link add lo-10 type dummy
ip link set lo-10 master br-10 && ip link set lo-10 up
```

---

## Phase 6 — FRR EVPN

### 6.1 gw 노드
```bash
apt update && apt install -y frr
# daemons: bgpd=yes, vtysh_enable=yes (나머지 no)
sed -i 's/^bgpd=no/bgpd=yes/' /etc/frr/daemons

cat > /etc/frr/frr.conf <<'EOF'
frr version 8.4.4
frr defaults datacenter
hostname kdvmd-pl-cyyoon04-gw
no ipv6 forwarding
service integrated-vtysh-config
!
router bgp 65000
 bgp router-id 172.16.1.160
 no bgp default ipv4-unicast
 neighbor EVPN peer-group
 neighbor EVPN remote-as 65001
 neighbor 172.16.1.157 peer-group EVPN
 neighbor 172.16.1.159 peer-group EVPN
 !
 address-family l2vpn evpn
  neighbor EVPN activate
  advertise-all-vni
 exit-address-family
!
EOF
systemctl restart frr
```

### 6.2 compute 노드 (oscompt01/02)
```bash
cat > /etc/frr/frr.conf <<'EOF'
frr version 8.4.4
frr defaults datacenter
hostname kdvmd-pl-cyyoon04-oscompt01      # 02는 hostname/router-id만 변경
no ipv6 forwarding
service integrated-vtysh-config
!
router bgp 65001
 bgp router-id 172.16.1.157                # 02는 172.16.1.159
 no bgp default ipv4-unicast
 neighbor 172.16.1.160 remote-as 65000
 !
 address-family l2vpn evpn
  neighbor 172.16.1.160 activate
  advertise-all-vni
 exit-address-family
!
EOF
systemctl restart frr
```
> `frr defaults datacenter` 필수 — `traditional`이면 정책상 EVPN 차단. `advertise-all-vni`는 로컬 VNI를 자동 EVPN 광고.

---

## Phase 7 — BGP-EVPN 세션 확인

```bash
# gw
vtysh -c 'show bgp l2vpn evpn summary'    # oscompt01/02 둘 다 Up, PfxRcd/PfxSnt > 0
vtysh -c 'show evpn vni'                   # VNI 10 / L2 / # Remote VTEPs 2

# compute
vtysh -c 'show evpn vni'                   # VNI 10 / L2 / # Remote VTEPs 1
```

---

## Phase 8 — OVN Native EVPN 활성화

### 8.1 이전 OVN 키 초기화 (재테스트 시)
```bash
# compute OVS external_ids 초기화
ovs-vsctl remove Open_vSwitch . external-ids ovn-evpn-local-ip 2>/dev/null
ovs-vsctl remove Open_vSwitch . external-ids ovn-evpn-vxlan-ports 2>/dev/null
ovn-appctl -t ovn-controller inc-engine/recompute 2>/dev/null

# provider LS dynamic-routing 키 초기화
LS=neutron-73dbaf22-d6f7-44c7-82cf-42411c61c71c
for k in dynamic-routing-vni dynamic-routing-bridge-ifname dynamic-routing-vxlan-ifname \
         dynamic-routing-advertise-ifname dynamic-routing-redistribute; do
  kubectl-ko nbctl remove Logical_Switch $LS other_config $k 2>/dev/null
done
kubectl-ko sbctl find advertised_mac        # 비어 있어야 함
```

### 8.2 provider LS에 dynamic-routing 키 설정
provider LS = external net 매핑 LS (localnet 포트 보유 → distributed FIP 광고 가능).
```bash
LS=neutron-73dbaf22-d6f7-44c7-82cf-42411c61c71c
kubectl-ko nbctl set Logical_Switch $LS \
  other_config:dynamic-routing-vni=10 \
  other_config:dynamic-routing-bridge-ifname=br-10 \
  other_config:dynamic-routing-vxlan-ifname=vxlan-10 \
  other_config:dynamic-routing-advertise-ifname=lo-10
kubectl-ko nbctl set Logical_Switch $LS \
  other_config:dynamic-routing-redistribute=fdb,ip,nat
```
- `ip` = VIF/router-port 주소, `nat` = 분산 FIP(dnat_and_snat), `fdb` = FDB inject. **FIP는 `nat` 토큰으로 게이팅**.

### 8.3 SB advertised_mac 자동 등록 확인
```bash
kubectl-ko sbctl find advertised_mac
#  ip "172.16.1.156" mac fa:16:3e:1e:22:45   ← FIP (Type-2 광고 타깃)
#  ip "172.16.1.153" mac fa:16:3e:df:48:35   ← provider 직결 VM
```

### 8.4 ovn-evpn 포트 활성화 (compute)
`ovn-evpn-local-ip` + `ovn-evpn-vxlan-ports`를 OVS external_ids에 넣으면 ovn-controller가 br-int에 `ovn-evpn-4789` 포트를 자동 생성.
```bash
# oscompt01
ovs-vsctl set Open_vSwitch . \
  external-ids:ovn-evpn-local-ip=172.16.1.157 \
  external-ids:ovn-evpn-vxlan-ports=4789
# oscompt02
ovs-vsctl set Open_vSwitch . \
  external-ids:ovn-evpn-local-ip=172.16.1.159 \
  external-ids:ovn-evpn-vxlan-ports=4789
```

---

## Phase 9 — VM 통신 검증

ping src는 **services 대역(172.16.1.160)** 으로 명시. `-I br-10`은 br-10에 IP가 없어 src가 api 대역으로 잡혀 VM 응답이 안 돌아온다.
```bash
# provider 직결 VM → ttl=64
ping -I 172.16.1.160 172.16.1.153 -c 5      # 0% loss, ttl=64

# FIP → ttl=63 (NAT 1홉 경유)
ping -I 172.16.1.160 172.16.1.156 -c 5      # 0% loss, ttl=63
```
✅ FIP(156) ttl=63, direct(153) ttl=64, 둘 다 0% loss → **Type-2 FIP 광고 실증 완료**.

---

## 핵심 주의 / 함정

1. **OVN 버전**: kube-ovn 1.15.11 stock = OVN 25.03. Type-2 FIP는 26.03 필요 → 커스텀 이미지 필수. `nbctl --version`으로 26.03.90 확인.
2. **ovs-ovn 권한**: `runAsUser:0 + privileged:true` 없으면 Netlink RTM_NEWNEIGH 실패 → static FDB inject 안 됨.
3. **br-10에 IP 금지**: services NIC와 같은 서브넷(172.16.1.0/24) 충돌. br-10은 L2 only, src IP는 services(.160) 사용.
4. **vxlan nolearning** (compute): FRR이 BGP로 주입한 static FDB와 패킷 자가학습 충돌 회피.
5. **frr defaults datacenter**: traditional이면 EVPN 차단.
6. **provider LS에 localnet 포트 필수**: 없으면 DGP peer가 chassisredirect가 되어 NAT이 distributed로 안 잡혀 FIP 광고 안 됨.
7. **redistribute 토큰**: FIP는 `nat`, VIF/router-port는 `ip`, FDB inject는 `fdb`. FIP만 광고하려면 `fdb,nat`.
8. **재테스트 시 cleanup 순서**: OVN 키 제거 → OVS external_ids 제거 → 호스트 인터페이스/라우트 삭제. (Phase 5.1 / 8.1)

---

## 참고

- 이미지 변경 PR: <https://github.com/kt-cloud-stack/ovn/pull/1>
- 빌드 절차/함정: 별도 메모 (kube-ovn base 이미지 OVS submodule 핀, QEMU amd64 빌드).
- Type-5 (LR prefix) 경로는 본 Type-2(L2)와 다름 — `01`~`04` 노트 참고.
