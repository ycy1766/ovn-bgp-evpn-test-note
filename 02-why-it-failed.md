# 왜 막혔나, upstream은 어디까지 왔나

01에서 막힌 이유와 upstream(Neutron, OVN-Kubernetes, kube-ovn)의 진행 상황 정리.

## Type-2 vs Type-5 — 결정적 차이

이전 lab(pl-cyyoon04)의 Type-2는 동작했는데 Type-5는 안 됐다. 외형은 둘 다 BGP EVPN `l2vpn evpn` address-family인데, 광고 가능 여부를 결정하는 의미론이 다르다.

Type-2 (MAC+IP)는 L2 reachability. "이 MAC이 이 VTEP 뒤에 있다"가 전부다. nexthop 검증이 없다. FRR `advertise-all-vni` 가 host bridge FDB를 그대로 EVPN으로 emit.

Type-5 (IP prefix)는 L3 routing. nexthop이 필수다. FRR `redistribute kernel` 은 RIB의 active route만 광고한다. active가 되려면 nexthop이 같은 VRF 안에서 reachable해야 한다 (RFC 4271). 이건 어떤 routing daemon으로 바꿔도 같다.

OVN이 install하는 route는:
- blackhole route (nexthop 없음) → 무조건 inactive
- `dynamic-routing-v4-prefix-nexthop` 줘도 nexthop이 default VRF의 device를 가리킴 → cross-VRF nexthop → inactive

즉 OVN이 vrf-evpn(table 100) 안에 route를 넣어도, 그 route의 nexthop을 vrf-evpn 안에서 풀 길이 없다 (br-evpn은 EVPN underlay 전용이라 외부 nexthop 도달성 제공 안 함).

## OVN 측 통합 코드의 비대칭

Type-2는 OVN이 데이터 평면까지 직접 만져준다:

- `controller/evpn-binding.c` — SB datapath ↔ br-int의 ovn-evpn-* port 매핑
- `controller/evpn-fdb.c` — SB Advertised_MAC → host bridge FDB inject
- `controller/evpn-arp.c` — ARP responder OF flow
- `controller/neighbor.c` (`91376604d` 패치) — FDB MAC을 광고 후보로 수집

거기에 `ovs-vsctl set Open_vSwitch . external_ids:ovn-evpn-local-ip=<VTEP>` 하면 br-int에 `ovn-evpn-4789` VXLAN port가 자동 생성된다. OVN이 LS의 L2 도메인을 EVPN VXLAN underlay로 직접 노출.

Type-5에 대응되는 통합 코드가 없다:

- `controller/route.c` — kernel route entry 준비
- `controller/route-exchange-netlink.c` — kernel RTM_NEWROUTE with RTPROT_OVN(=84)

여기까지가 끝. EVPN underlay와 LR routing context 사이 통합점 없음. br-evpn / vxlan / nexthop reachability는 사용자가 만들어야 한다. 만들어줘도 위에 적은 routing 의미론 때문에 inactive에 걸린다.

## VRF model과 OVN datapath의 mismatch

Linux VRF (table N)는 strict L3 격리. 한 VRF 안의 route lookup은 그 VRF의 device들 사이에서만 수행되고, nexthop이 다른 VRF의 device를 가리키면 cross-VRF로 분류돼 BGP routing 결정에서 inactive.

OVN datapath는 자체 logical network. br-int의 OF flow가 LR/LS의 모든 처리를 함. 외부에서 보면 black-box. kernel VRF와 매핑할 표준 인터페이스가 없다. `dynamic-routing-vrf-id` 는 단지 "어느 table에 route 적을지" 알려주는 hint일 뿐, OVN datapath와 kernel VRF의 routing context를 연결하지 않는다.

## OpenStack Neutron spec

[bgp_evpn_type_5_route_support](https://specs.openstack.org/openstack/neutron-specs/specs/2026.2/bgp_evpn_type_5_route_support.html) — Neutron 2026.2 target.

spec 도입부에 그대로 명시되어 있다:

> Using OVN and FRR alone is not enough to deploy EVPN Type-5 route advertisement on OpenStack. It is necessary to add a new EVPN Service Plugin to configure OVN with new options. It is also necessary to add a new OVN Agent EVPN Extension to prepare the Linux environment for FRR EVPN as well as to enable EVPN in FRR on each compute/network node.

즉 01에서 막힌 거랑 같은 결론을 spec 작성자들이 먼저 내렸고, 그 gap을 채우기 위해 두 새 컴포넌트(Service Plugin + Agent Extension)를 만드는 게 spec의 본체.

spec의 핵심 architecture:
- Tenant LR ↔ dummy LS (`ls-evpn-$VNI`) 매개
- dummy LS가 OVN 안의 EVPN VNI anchor
- host 인터페이스 (`br-evpn` VLAN-aware, `vxlan-evpn` shared, `vlan-$VID` SVI in VRF) 를 Agent가 자동 provisioning
- Agent FSM (`WAITING_FOR_MAC`/`WAITING_FOR_VRF`/`ADVERTISING`) 으로 Port_Binding event와 VRF netlink event 사이 race 처리

내가 01에서 빠뜨린 핵심이 dummy LS. LR에만 옵션 줬다. spec 보고 나서 회고하면, Type-2가 LS에 옵션을 줘서 동작한 것이고 Type-5에도 같은 anchor가 필요한데, spec은 그걸 dummy LS로 풀고 있다.

### 구현 진행도 (2026-05-27 시점)

GitHub openstack/neutron 검색해보면:

| 컴포넌트 | 상태 |
| --- | --- |
| spec 문서 | 머지 완료 (2026-05-20) |
| neutron-lib `evpn` API definition | 머지 완료 (2026-04-30 ~ 05-06) |
| `neutron/agent/ovn/extensions/evpn/` | skeleton + FSM + netlink monitor 머지 (2026-04-13 ~ 04-30) |
| `neutron/services/evpn/plugin.py` | skeleton 머지 (2026-05-21). 코드 안에 `_fake_db_call()` TODO 다수 |

open review 7개가 2026-05-22~26 사이 update:
- DB model 구현 (#987250)
- OVN portion 구현 (#989626)
- FRR driver (#988158)
- Single VxLAN Device management (#990169)
- port_binding events (#989471)
- common constants (#990136)
- BGP plugin 정비 (#988539)

한 달 안에 spec의 핵심 구현이 review까지 다 올라온 셈. Red Hat 주도 (코드 카피라이트 + 핵심 author `jlibosva`).

타임라인 추정: 2026.2 GA (2026-10) 에 dev preview, RHOSP backport 후 2027.1~2027.2 에 stable. KCP 운영 적용 가능한 안정성까지는 2027 하반기 이후.

방향성 변화도 명확. 2025-10에 spec `OVN BGP Agent EVPN Advertisement API Extension` 이 abandoned, 2026-04에 Neutron native EVPN extension 작업 시작. 즉 사이드카(ovn-bgp-agent) 모델은 폐기되고 Neutron native로 통합되는 흐름.

ovn-bgp-agent 측에선 EVPN 관련 마지막 commit이 2025-05 (1년 전). Type-5 native 지원은 0.

## OVN-Kubernetes (별도 프로젝트)

KCP가 쓰는 kube-ovn(Aliyun) 과는 다른 프로젝트지만 architecture 참조용. [OKEP-5088 EVPN Support](https://ovn-kubernetes.io) 가 OVN-Kubernetes의 EVPN spec.

Neutron spec과 거의 같은 방향:
- Single VxLAN Device 모델 (MVD 명시적 비채택)
- FRR-K8S로 FRR config 자동 관리
- node가 EVPN VTEP (= leaf 역할)
- VTEP CRD (managed/unmanaged) 로 VTEP IP 관리
- CUDN의 `evpnConfiguration` 으로 macVRF/ipVRF 정의
- Type-2 + Type-5 모두 지원 (Neutron spec 보다 광범위)
- local gateway mode 만 지원 (shared mode는 future)

두 upstream의 architecture가 거의 일치. node=VTEP, SVD, FRR 자동 관리, leaf-spine 패턴. 차이는 K8s/OpenStack 추상화 정도.

## kube-ovn (KCP 사용 중)

kube-ovn 자체에도 BGP/EVPN 작업이 있다.

| 버전 | 기능 |
| --- | --- |
| v1.15.x | BGP via vpc-nat-gateway, BGP speaker, BgpConf CRD |
| v1.16.0 (2026-04-09) | vpc-egress-gateway가 BGP+EVPN L3VPN 지원 (PR #6224). FRR가 egress-gateway Pod 안에서 동작 |

핵심: **OpenStack VM의 FIP를 BGP/EVPN으로 직접 광고하는 path는 미지원**.

Issue #5010 (2025-08-10 closed) 에서 정확히 이 use case를 요청했는데 kube-ovn 메인테이너 oilbeater가:

> OVN doesn't natively support BGP for EIP, FIP, and SNAT. This means we might need to introduce a new set of agents or controllers and possibly modify the network flow. This isn't an easy feature.

라고 답변. 즉 kube-ovn 측도 OpenStack VM/FIP의 직접 BGP 광고는 별도 컴포넌트 필요하다고 인식. 메인테이너 zbb88888이 ovn-bgp-agent dive into 한다고 했지만 그 후 진행은 미정.

## 정리

01에서 내가 막힌 게 알고 보니 OpenStack과 K8s 양쪽 upstream이 모두 인지하고 작업 중인 ‘known gap’ 이었다. 운영 적용 가능한 솔루션이 나오기까지 1~2년. KCP는 그동안 다른 path가 필요하다. 그게 다음 노트의 G1.

| 후보 | 평가 |
| --- | --- |
| Neutron 2026.2 spec 대기 | 1~2년. spec 모델(compute=VTEP)이 KCP fabric 전략과 다름 |
| ovn-bgp-agent 채용 | EVPN Type-5 native 미지원 (Type-2 anycast만). 우리 use case 안 맞음 |
| kube-ovn 자체 EVPN 확장 대기 | OpenStack VM 직접 광고는 메인테이너도 별도 컴포넌트 필요하다고 인정 |
| OVN-Kubernetes 마이그레이션 | 너무 큰 변경 |
| **G1: ToR가 EVPN 종단, compute는 BGP IPv4만** | 즉시 가능. upstream 의존 0. KCP 네트워크팀의 ToR-EVPN 결정과 정렬 |
