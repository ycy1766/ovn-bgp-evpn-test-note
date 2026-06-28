# v7 Type-2 FIP BGP-EVPN 테스트 런북 (현재 이미지 기준, pl-cyyoon02)

OVN Native BGP-EVPN을 사용하기 위해 kube-ovn 이미지 내 OVN 버전을 **26.03 이상**으로 올려, OpenStack Floating IP / provider-direct VM을 **EVPN Type-2 (MAC+IP)** 로 광고하는 테스트. 테스트 목적으로 내부 빌드 이미지를 배포 후 확인.

- **변경된 이미지 내용 (diff)**: <https://github.com/ycy1766/ovn/compare/31e11ad65...evpn-fip-type2-upstream-v7>
- **변경된 이미지**: `docker.io/ycy1766/kube-ovn:v1.15.11-evpn-fip-type2-v7`
  (digest `sha256:ddd150e0d38a9add600892c5d7ca8174093fabdc2ba854947b2ca5cc64f53e42`, OVN 26.03.90)

> v5 이미지 대비 v7은 SB `Advertised_MAC_Binding`의 `type` 컬럼이 제거되었다(스키마 = upstream 21.9.0). `advertised_mac` 조회에 `type`이 안 나오는 게 정상이며, FIP Type-2 광고 동작은 동일하다. (full 절차는 [05](./05-type2-fip-evpn-test-runbook.md)와 같고, 본 문서는 v7 이미지로 standalone 수행.)

---

## 1. 이미지 변경 / 배포

### 1.1 helm override 태그를 v7로
`/etc/genestack/helm-configs/kube-ovn/kube-ovn-helm-overrides.yaml`:
```yaml
global:
  registry:
    #address: docker.io/kubeovn
    address: docker.io/ycy1766
  images:
    kubeovn:
      repository: kube-ovn
      vpcRepository: vpc-nat-gateway
      #tag: v1.15.11
      tag: v1.15.11-evpn-fip-type2-v7
      support_arm: true
      thirdparty: true
networking:
  IFACE: "tun"
  ENABLE_SSL: true
  OVN_NORTHD_N_THREADS: 2
ipv4:
  POD_CIDR: "10.236.0.0/14"
  POD_GATEWAY: "10.236.0.1"
  SVC_CIDR: "10.233.0.0/18"
  JOIN_CIDR: "100.64.0.0/16"
```

### 1.2 배포
```bash
bash /opt/genestack/bin/install-kube-ovn.sh
```

### 1.3 OVN 버전 확인
```bash
kubectl-ko nbctl --version
#  ovn-nbctl 26.03.90          ← 버전 확인
#  Open vSwitch Library 3.7.2
#  DB Schema 7.18.0
```

---

## 2. kube-ovn 사전작업

ovn-controller Pod(ovs-ovn)가 각 호스트의 Netlink로 FDB 정보를 삽입해야 하므로 권한을 추가로 열어준다. (위키 `04. kcp bgp-evpn type-2 / Kube-OVN 사전 작업` 과 동일, Type-5에도 동일 적용.)

### 2.1 ovs-ovn securityContext 수정
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

### 2.2 소켓/로그 경로 권한 수정 (ovn-central 노드)
> pl-cyyoon02는 `kube_ovn_central_hosts = ovn_network_nodes` = oscompt01/02. ovn-central 노드 IP는 아래로 확인.
```bash
# ovn-central 노드 확인
kubectl -n kube-system get pod -l app=ovn-central -o wide

# ovn_network_nodes(oscompt01/02) api IP
for node in 10.21.1.27 10.21.1.28; do
    ssh $node 'chown -R nobody:nogroup /var/run/ovn /var/log/ovn'
done
```

### 2.3 변경된 권한 확인
```bash
kubectl -n kube-system exec ds/ovs-ovn -c openvswitch -- id
#  uid=0(root) gid=0(root) groups=0(root)

kubectl -n kube-system get pod -l app=ovn-central
#  ovn-central-... 1/1 Running (전부 Running)
```
> ⚠️ securityContext 변경 직후 `/var/run/ovn`·`/var/log/ovn`이 root 소유로 바뀌어 **ovn-central이 CrashLoopBackOff** 날 수 있다. 위 2.2 chown(nobody:nogroup) 적용하면 회복된다(실측: 3 replica 전부 Running 복귀).

---

## 3. 커널 VRF 모듈 (gw + 모든 compute 동일)

Ubuntu 기본 커널엔 vrf 모듈이 없어 `linux-modules-extra` 설치 필요.
```bash
## 기본 커널에 VRF 미존재
modprobe vrf vxlan
#  modprobe: FATAL: Module vrf not found ...

## modules-extra 설치 (vrf 포함)
apt update && apt install -y linux-modules-extra-$(uname -r)

## 모듈 로드 + 부팅 자동 로드 등록
echo vrf   | tee /etc/modules-load.d/vrf.conf
echo vxlan | tee /etc/modules-load.d/vxlan.conf

## ip_forward 활성화
sysctl -w net.ipv4.ip_forward=1
echo 'net.ipv4.ip_forward=1' | tee /etc/sysctl.d/99-evpn.conf

## 확인
lsmod | egrep 'vrf|vxlan'        # vrf 로드됨
sysctl net.ipv4.ip_forward       # = 1
```

---

## 4. 테스트 환경 구성

VM 1대를 tenant 네트워크에 띄우고 **FIP 부착**으로 Type-2 광고 테스트.

> 랩 **pl-cyyoon02** 기준 (VM이 `kdvmd-pl-cyyoon02-oscompt01`에 배치). 노드/IP 매핑:
>
> | 역할 | 호스트 | services(VTEP/BGP) IP | ASN |
> |------|--------|------------------------|-----|
> | gw | kdvmd-pl-cyyoon02-gw | 172.16.1.20 | 65000 |
> | compute (VM 위치) | kdvmd-pl-cyyoon02-oscompt01 | 172.16.1.27 | 65001 |
> | compute | kdvmd-pl-cyyoon02-oscompt02 | 172.16.1.28 | 65001 |
>
> provider(FIP) 풀 `172.16.1.31~39`, VIP `172.16.1.30`, services NIC `ens8`, VNI 10.

```bash
## tenant network/subnet
openstack network create --availability-zone-hint az1 region01-test-network-az1
openstack subnet create --network region01-test-network-az1 --gateway 192.168.20.1 \
  --subnet-range 192.168.20.0/24 region01-test-subnet-az1

## provider network (FIP 풀)
openstack network create --share --external --availability-zone-hint az1 \
  --provider-physical-network physnet1 --provider-network-type flat internal-provider-network
openstack subnet create --network internal-provider-network \
  --dns-nameserver 8.8.4.4 --gateway 172.16.1.254 \
  --subnet-range 172.16.1.0/24 internal-provider-subnet

## Router
openstack router create --availability-zone-hint az1 test-router-az1
openstack router add subnet test-router-az1 region01-test-subnet-az1
openstack router set --external-gateway internal-provider-network test-router-az1

## SG
openstack security group create region01-test-sg
openstack security group rule create --proto icmp region01-test-sg
openstack security group rule create --proto tcp  region01-test-sg

## VM (tenant + FIP)
openstack server create --flavor ktc-m1.tiny --network region01-test-network-az1 \
  --image cirros --security-group region01-test-sg --availability-zone az1 region01-vm1
```

```text
$ openstack server list
| 6af4180a-2b66-4a42-8ea4-55c22f7f2cd5 | region01-vm1 | ACTIVE | region01-test-network-az1=172.16.1.32, 192.168.20.183 | cirros | ktc-m1.tiny |

$ openstack server show region01-vm1
| OS-EXT-SRV-ATTR:host | kdvmd-pl-cyyoon02-oscompt01                           |
| addresses            | region01-test-network-az1=172.16.1.32, 192.168.20.183 |
```
- region01-vm1: tenant fixed **192.168.20.183**, FIP **172.16.1.32**, host **kdvmd-pl-cyyoon02-oscompt01**.

### FIP 생성/부착
```bash
openstack floating ip create internal-provider-network        # → 172.16.1.32
openstack server add floating ip region01-vm1 172.16.1.32
#  region01-vm1: 172.16.1.32, 192.168.20.183
```

---

## 5. EVPN 연동 — gw 노드

### 5.1 FRR daemons 활성화
```bash
apt update && apt install -y frr
cat > /etc/frr/daemons <<'EOF'
bgpd=yes
ospfd=no
ospf6d=no
ripd=no
ripngd=no
isisd=no
pimd=no
ldpd=no
nhrpd=no
eigrpd=no
babeld=no
sharpd=no
pbrd=no
bfdd=no
fabricd=no
vrrpd=no
pathd=no
vtysh_enable=yes
zebra_options="  -A 127.0.0.1 -s 90000000"
bgpd_options="   -A 127.0.0.1"
staticd_options="-A 127.0.0.1"
EOF
systemctl restart frr
systemctl status frr | grep -E 'bgpd|zebra'      # zebra/bgpd up
```

### 5.2 이전 호스트 인터페이스 삭제
```bash
VNI=10
ip link del vxlan-${VNI} 2>/dev/null
ip link del lo-${VNI}    2>/dev/null
ip link del br-${VNI}    2>/dev/null
# VM/FIP IP를 br-10으로 보내던 /32 라우트 제거 (provider 풀 31~39)
for ip in $(seq 31 39); do ip route del 172.16.1.${ip}/32 dev br-10 2>/dev/null; done
ip route show | grep -E '172\.16\.1\.3[1-9]/32' || echo "clean"
```

### 5.3 호스트 인터페이스 생성
> br-10에 IP를 주면 services NIC와 같은 서브넷(172.16.1.0/24) 충돌. **br-10은 L2 bridge로만 사용, IP 미부여.** VM 통신 src IP는 gw services NIC의 172.16.1.20 사용.
```bash
VNI=10; LOCAL_IP=172.16.1.20          # gw services IP
ip link add br-10 type bridge && ip link set br-10 up
ip link add vxlan-10 type vxlan id ${VNI} local ${LOCAL_IP} dstport 4789
ip link set vxlan-10 master br-10 && ip link set vxlan-10 up
ip link add lo-10 type dummy
ip link set lo-10 master br-10 && ip link set lo-10 up
ip -d link show vxlan-10 | grep dstport       # dstport 4789
```

### 5.4 호스트 라우팅
> 테스트 환경상 service/광고 인터페이스가 한 NIC에 몰려있음 (실 환경은 분리 필요). FIP 풀(31~39)만 br-10으로, VTEP(27/28)·default GW는 services 그대로.
```bash
for ip in $(seq 31 39); do ip route add 172.16.1.${ip}/32 dev br-10 2>/dev/null; done
```

### 5.5 FRR 설정
> `frr defaults datacenter` 필수 (traditional이면 정책상 EVPN 차단). `advertise-all-vni` = 로컬 VNI 자동 EVPN 광고.
```bash
cat > /etc/frr/frr.conf <<'EOF'
frr version 8.4.4
frr defaults datacenter
hostname kdvmd-pl-cyyoon02-gw
no ipv6 forwarding
service integrated-vtysh-config
!
router bgp 65000
 bgp router-id 172.16.1.20
 no bgp default ipv4-unicast
 neighbor EVPN peer-group
 neighbor EVPN remote-as 65001
 neighbor 172.16.1.27 peer-group EVPN
 neighbor 172.16.1.28 peer-group EVPN
 !
 address-family l2vpn evpn
  neighbor EVPN activate
  advertise-all-vni
 exit-address-family
!
EOF
systemctl restart frr
```

---

## 6. EVPN 연동 — Compute 노드 (oscompt01 / oscompt02)

### 6.1 이전 호스트 인터페이스 삭제 (양 노드)
```bash
VNI=10
ip link del vxlan-${VNI} 2>/dev/null
ip link del lo-${VNI}    2>/dev/null
ip link del br-${VNI}    2>/dev/null
```

### 6.2 OVS external_ids 초기화 (양 노드)
```bash
ovs-vsctl remove Open_vSwitch . external-ids ovn-evpn-local-ip 2>/dev/null
ovs-vsctl remove Open_vSwitch . external-ids ovn-evpn-vxlan-ports 2>/dev/null
ovn-appctl -t ovn-controller inc-engine/recompute 2>/dev/null
ovs-vsctl show | grep -A3 evpn || echo "no evpn port"
ovn-appctl -t ovn-controller evpn/vtep-binding-list 2>/dev/null     # 비어 있어야
```

### 6.3 OVN LS dynamic-routing 키 초기화 (ctrl 노드)
```bash
# provider LS 이름 = neutron-<provider-net-uuid> (접두사 neutron- 필수!)
# openstack CLI는 openstack-client pod 전용이라 ctrl 노드에선 UUID 직접 지정.
LS=neutron-e6a34482-31cb-4bdb-86b7-926c7c7ee28a
#  net UUID 확인(openstack-client pod): openstack network show internal-provider-network -f value -c id
#  OVN에서 확인(ctrl 노드):              kubectl-ko nbctl show | grep -B1 'type: localnet'
echo $LS
for k in dynamic-routing-vni dynamic-routing-bridge-ifname \
         dynamic-routing-vxlan-ifname dynamic-routing-advertise-ifname \
         dynamic-routing-redistribute; do
    kubectl-ko nbctl remove Logical_Switch $LS other_config $k 2>/dev/null
done
kubectl-ko nbctl get Logical_Switch $LS other_config     # 키 없어야
kubectl-ko sbctl find advertised_mac                     # 비어 있어야
```

### 6.4 호스트 인터페이스 (oscompt01: LOCAL_IP=27 / oscompt02: 28)
> br-10: L2 학습/광고 bridge. lo-10: static FDB 광고용 dummy(ovn-controller가 `advertised_mac`의 VM MAC을 RTM_NEWNEIGH로 주입). vxlan-10: remote VTEP 학습, `nolearning`으로 FRR static FDB와 자가학습 충돌 회피.
```bash
VNI=10; LOCAL_IP=172.16.1.27          # oscompt02는 172.16.1.28
ip link add br-10 type bridge && ip link set br-10 up
ip link add vxlan-10 type vxlan id ${VNI} local ${LOCAL_IP} dstport 60010 nolearning
ip link set vxlan-10 master br-10 && ip link set vxlan-10 up
ip link add lo-10 type dummy
ip link set lo-10 master br-10 && ip link set lo-10 up
ip -d link show vxlan-10 | grep dstport       # dstport 60010
```

### 6.5 FRR 설정 (oscompt01 예시; oscompt02는 hostname/router-id만 172.16.1.28로)
```bash
cat > /etc/frr/frr.conf <<'EOF'
frr version 8.4.4
frr defaults datacenter
hostname kdvmd-pl-cyyoon02-oscompt01
no ipv6 forwarding
service integrated-vtysh-config
!
router bgp 65001
 bgp router-id 172.16.1.27
 no bgp default ipv4-unicast
 neighbor 172.16.1.20 remote-as 65000
 !
 address-family l2vpn evpn
  neighbor 172.16.1.20 activate
  advertise-all-vni
 exit-address-family
!
EOF
systemctl restart frr
```

---

## 7. BGP-EVPN 세션 확인

```bash
# gw
vtysh -c 'show bgp l2vpn evpn summary'
#  oscompt01(172.16.1.27) ... State/PfxRcd 1   PfxSnt 3
#  oscompt02(172.16.1.28) ... State/PfxRcd 1   PfxSnt 3
#  Total number of neighbors 2
vtysh -c 'show evpn vni'
#  10  L2  vxlan-10  ...  # Remote VTEPs 2  default

# compute (각 노드)
vtysh -c 'show evpn vni'
#  10  L2  vxlan-10  ...  # Remote VTEPs 1  default
```

---

## 8. OVN Native EVPN 활성화

### 8.1 provider LS에 dynamic-routing 키 추가
OVN northd가 NB Logical_Switch 설정을 읽어 SB에 광고 엔트리를 생성.
```bash
# provider LS 확인 (localnet 포트 보유) — ctrl 노드에서 OVN으로 직접 확인
kubectl-ko nbctl show | grep -B1 'type: localnet'
#  switch <uuid> (neutron-e6a34482-31cb-4bdb-86b7-926c7c7ee28a) (aka internal-provider-network)

LS=neutron-e6a34482-31cb-4bdb-86b7-926c7c7ee28a
kubectl-ko nbctl set Logical_Switch $LS \
    other_config:dynamic-routing-vni=10 \
    other_config:dynamic-routing-bridge-ifname=br-10 \
    other_config:dynamic-routing-vxlan-ifname=vxlan-10 \
    other_config:dynamic-routing-advertise-ifname=lo-10
kubectl-ko nbctl set Logical_Switch $LS \
    other_config:dynamic-routing-redistribute=fdb,ip,nat

# 확인
kubectl-ko nbctl get Logical_Switch $LS other_config
#  {... dynamic-routing-redistribute="fdb,ip,nat", dynamic-routing-vni="10", ...}
```
- `ip`=VIF/router-port, `nat`=분산 FIP(dnat_and_snat), `fdb`=FDB inject. **FIP는 `nat` 토큰 필수.**

### 8.2 SB advertised_mac 자동 등록 확인 (v7: type 컬럼 없음)
```bash
kubectl-ko sbctl find advertised_mac
```
```text
_uuid        : <uuid>
datapath     : <provider LS datapath>
ip           : "172.16.1.32"            ← FIP (Type-2 광고 타깃, oscompt01)
logical_port : <provider LS router port>
mac          : "fa:16:3e:2f:a7:aa"      ← FIP external_mac
```
> v7 출력엔 `type` 컬럼이 없다(스키마에서 제거됨). FIP 행(172.16.1.32 / fa:16:3e:2f:a7:aa)이 정상 등록되면 OK.
> external_mac 확인: `kubectl-ko nbctl find nat external_ip=172.16.1.32` (NAT type=dnat_and_snat).

### 8.3 ovn-evpn 포트 활성화 (compute)
`ovn-evpn-local-ip` + `ovn-evpn-vxlan-ports`를 OVS external_ids에 넣으면 ovn-controller가 br-int에 `ovn-evpn-4789` 포트를 자동 생성.
```bash
# oscompt01
ovs-vsctl set Open_vSwitch . \
  external-ids:ovn-evpn-local-ip=172.16.1.27 \
  external-ids:ovn-evpn-vxlan-ports=4789
# oscompt02
ovs-vsctl set Open_vSwitch . \
  external-ids:ovn-evpn-local-ip=172.16.1.28 \
  external-ids:ovn-evpn-vxlan-ports=4789
```

---

## 9. VM 통신 테스트

ping src IP는 gw services 대역(172.16.1.20) 명시. `-I br-10`은 br-10에 IP가 없어 src가 api 대역으로 잡혀 응답이 안 돌아온다. FIP는 NAT 1홉 경유라 ttl=63.

```bash
# gw 에서 FIP → ttl=63 (NAT 1홉 경유)
ping -I 172.16.1.20 172.16.1.32 -c 5
#  64 bytes from 172.16.1.32: ttl=63 ...
#  5 packets transmitted, 5 received, 0% packet loss
```

✅ FIP(172.16.1.32) ttl=63, 0% loss → **v7 이미지로 Type-2 FIP 광고 재현 완료.**

---

## 부록 — 핵심 함정

1. **OVN 버전**: stock(25.03) 아닌 26.03 이미지 필수. `nbctl --version`으로 26.03.90 확인.
2. **ovs-ovn 권한**: `runAsUser:0 + privileged:true` 없으면 RTM_NEWNEIGH 실패 → static FDB inject 안 됨.
3. **br-10 IP 금지** (services 서브넷 충돌). ping src는 gw services IP(172.16.1.20).
4. **compute vxlan `nolearning`** (FRR static FDB 충돌 회피).
5. **frr defaults datacenter** (traditional이면 EVPN 차단).
6. **provider LS에 localnet 포트 필수** (없으면 NAT이 distributed로 안 잡혀 FIP 광고 안 됨).
7. **재테스트 cleanup 순서**: OVN 키 제거 → OVS external_ids 제거 → 호스트 인터페이스/라우트 삭제.
8. **v7 스키마**: `Advertised_MAC_Binding`에 `type` 컬럼 없음. v5(type 有) 위에 v7 in-place 배포 시 ovsdb 스키마 처리 주의(필요 시 `ovsdb-tool convert`).

## 부록 — 참고
- 변경 diff: <https://github.com/ycy1766/ovn/compare/31e11ad65...evpn-fip-type2-upstream-v7>
- v5 기준 상세 런북: [05-type2-fip-evpn-test-runbook.md](./05-type2-fip-evpn-test-runbook.md)
- 코드 PR: <https://github.com/kt-cloud-stack/ovn/pull/1>
