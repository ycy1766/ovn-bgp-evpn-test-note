# Type-5 직접 광고 시도와 실패

OVN+FRR만 가지고 compute에서 EVPN Type-5로 FIP `/32`를 직접 광고하려고 했던 첫 시도.
이전 lab(pl-cyyoon04)에서 Type-2는 코드 패치(머지된 4268d3ec1, 91376604d)로 검증됐는데, Type-5는 main 브랜치 기능만으로 가능할 줄 알았음. 결론은 안 됐다.

## 구성

ASN gw=65000 / compute=65001 (eBGP), VNI=10, dstport 4789.

| 노드 | services iface |
| --- | --- |
| kdvmd-pl-cyyoon05-gw | 172.16.1.110 |
| kdvmd-pl-cyyoon05-oscompt01 | 172.16.1.108 |
| kdvmd-pl-cyyoon05-oscompt02 | 172.16.1.109 |

테스트 대상: test-router-az1 + region01-vm1 (FIP 172.16.1.114, oscompt02) + region01-vm2 (FIP 172.16.1.115, oscompt01).

## 사전 작업 (kube-ovn 측)

이전 Type-2 lab 진행할 때 발견했던 사항인데 Type-5에도 동일하게 필요.

1. `ovs-ovn` daemonset의 `openvswitch` 컨테이너에 `runAsUser: 0, privileged: true` 패치. 안 그러면 ovn-controller가 netlink로 VRF/route/FDB 못 만짐.
2. patch 후 `/var/run/ovn`, `/var/log/ovn` 소유자가 root로 바뀌면 ovn-central CrashLoopBackOff. 모든 ovn-central 노드에서 `chown -R nobody:nogroup` 으로 원복.
3. 가장 중요한 함정: **patch 적용 후 `ovs-ovn` pod를 명시적으로 강제 재기동해야** 함. daemonset rolling update만으로는 ovn-controller가 좀비 상태로 남아서 ARP responder OF flow가 안 깔린다. 결과적으로 FIP/router-gw ARP가 응답 안 와서 ping이 죽음. pod 강제 delete 후 회복됨.

```bash
kubectl -n kube-system patch daemonset ovs-ovn --type=json -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/securityContext","value":{"runAsUser":0,"privileged":true}}
]'

# 모든 노드 chown
for n in $(kubectl get pod -n kube-system -l app=ovn-central -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}'); do
  ssh $n 'chown -R nobody:nogroup /var/run/ovn /var/log/ovn'
done

# 강제 재기동
for n in kdvmd-pl-cyyoon05-oscompt01 kdvmd-pl-cyyoon05-oscompt02; do
  kubectl -n kube-system delete pod -l app=ovs --field-selector spec.nodeName=$n
done
```

## 호스트 인터페이스 (compute, gw 양쪽)

Type-5는 VRF에 묶인 kernel route를 EVPN으로 export해야 한다고 봐서 VRF 트리오를 만들었다.

```bash
# linux-modules-extra 설치 (Ubuntu 24.04 stock kernel은 vrf 모듈이 extra에)
apt install -y linux-modules-extra-$(uname -r)
modprobe vrf vxlan
echo vrf   > /etc/modules-load.d/vrf.conf
echo vxlan > /etc/modules-load.d/vxlan.conf
sysctl -w net.ipv4.ip_forward=1

VNI=10
LOCAL_IP=<자기 services IP>
TABLE_ID=100

ip link add vrf-evpn type vrf table $TABLE_ID
ip link set vrf-evpn up
ip link add br-evpn type bridge
ip link set br-evpn master vrf-evpn
ip link set br-evpn up
ip link add vxlan-$VNI type vxlan id $VNI local $LOCAL_IP dstport 4789 nolearning
ip link set vxlan-$VNI master br-evpn
ip link set vxlan-$VNI up
```

## FRR (compute)

```
frr defaults datacenter
hostname kdvmd-pl-cyyoon05-oscompt01
!
vrf vrf-evpn
 vni 10
exit-vrf
!
router bgp 65001
 bgp router-id 172.16.1.108
 no bgp default ipv4-unicast
 neighbor 172.16.1.110 remote-as 65000
 address-family l2vpn evpn
  neighbor 172.16.1.110 activate
  advertise-all-vni
 exit-address-family
exit
!
router bgp 65001 vrf vrf-evpn
 bgp router-id 172.16.1.108
 address-family ipv4 unicast
  redistribute kernel
 exit-address-family
 address-family l2vpn evpn
  advertise ipv4 unicast
 exit-address-family
exit
```

`frr defaults traditional` 이면 EVPN 자체가 차단되니 반드시 `datacenter`. FRR 10.x 가 깔리는데 conf 안 버전 label은 무관.

## OVN NB 옵션

```bash
ROUTER_LR=neutron-c26ce272-3006-4ee3-9f26-caf465fae13b
EXT_LRP=lrp-48e53ee8-56b5-48b7-ba25-c2564ed9febf

kubectl-ko nbctl set logical_router "$ROUTER_LR" \
  options:dynamic-routing=true \
  options:dynamic-routing-vrf-id=100 \
  options:dynamic-routing-vrf-name=vrf-evpn \
  options:dynamic-routing-redistribute-local-only=true

kubectl-ko nbctl set logical_router_port "$EXT_LRP" \
  options:dynamic-routing-redistribute=nat \
  options:dynamic-routing-maintain-vrf=true
```

> 처음엔 internal LRP에 잘못 걸어서 디버깅하는 시간이 좀 있었음. external LRP(router_gateway device_owner) 에 걸어야 함.

## 동작한 부분

- SB Advertised_Route에 172.16.1.113/114/115 정상 등록
- ovn-controller가 vrf-evpn (table 100)에 RTPROT_OVN(=84) blackhole route install (한쪽 chassis만)
- BGP EVPN 세션 정상 (gw <-> oscompt01/02 모두 Established)
- `show evpn vni` 에 VNI 10 / Type L3 / vrf-evpn / br-evpn 정상

## 막힌 부분

gw가 광고를 못 받음. compute side에서 `show ip route vrf vrf-evpn` 했더니:

```
K   172.16.1.113/32 [0/100] via 172.16.1.113, services (vrf default) inactive
K   172.16.1.113/32 [0/100] unreachable (blackhole) inactive
K   172.16.1.114/32 [0/100] via 172.16.1.113, services (vrf default) inactive
K   172.16.1.114/32 [0/100] unreachable (blackhole) inactive
K>* 172.16.1.115/32 [0/1000] unreachable (blackhole)
```

`inactive`. FRR가 nexthop을 못 풀어서 BGP가 redistribute 안 함. 115만 우연히 entry가 하나뿐이라 best로 잡혀서 광고됐고, gw가 받는 prefix가 시간/옵션 토글에 따라 들쭉날쭉했다.

`(vrf default)` 표시가 결정적. nexthop 172.16.1.113은 services iface 위에 있는데 services는 default VRF 소속. vrf-evpn 안에선 reachable하지 않다고 zebra가 판정 → inactive → BGP가 advertise 거부. 이건 FRR 버그가 아니라 표준 routing daemon 동작 (nexthop 검증 통과 못 하는 route는 광고 X).

`dynamic-routing-v4-prefix-nexthop=172.16.1.113` 줘봤지만 같은 결과. nexthop은 잡혔는데 device가 default VRF라서.

추가로 발견한 부수 이슈:
- `dynamic-routing-redistribute-local-only=true` 로 두면 광고 자체가 사라지는 transition. install/remove 반복 사이에 FRR가 active 상태를 잃는다. PoC 중에는 일단 `false` 로 토글해서 봤다.
- oscompt01의 ovn-controller가 vrf-evpn 인터페이스를 인식 못 하는 시점이 있었음. vrf를 삭제 후 다시 만들고 pod 강제 재기동하니 잡힘. `maintain-vrf=true` 옵션은 이 ycy1766 빌드에서 실효 없음 (controller가 VRF 자동 생성 안 함).
- ovn-controller 로그에 FDB unique constraint violation이 가끔 떴다. 수동으로 만든 br-evpn FDB와 OVN의 FDB가 같은 MAC/dp_key로 충돌. 광고 동작 자체에 결정적 영향은 안 본 듯.

## 데이터 평면 측면

ping 자체는 vrf-evpn 컨텍스트로 보내면 무조건 drop (blackhole). 기본 VRF로 ping 172.16.1.114는 정상 동작 (OVN의 기존 br-provider 경로 + cr-lrp ARP). 즉 EVPN 광고는 data plane과 무관, 진짜 데이터는 OVN이 그대로 처리한다는 게 확인됨.

## 정리

- 광고 자체를 안정화하려면 OVN의 routing context가 EVPN underlay와 통합되어야 하는데 main에 그 코드가 없다.
- 다음 노트에서 왜 그런지, upstream은 어떻게 가고 있는지 정리.
