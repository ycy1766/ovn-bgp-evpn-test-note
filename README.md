# ovn-bgp-evpn-test-note

KCP 환경에서 OVN Floating IP를 BGP EVPN으로 광고하는 작업을 진행하면서 남긴 노트.

## 작업 흐름

| 순서 | 파일 | 내용 |
| --- | --- | --- |
| 1 | [01-direct-type5-attempt.md](./01-direct-type5-attempt.md) | OVN+FRR만으로 compute에서 EVPN Type-5 직접 광고 시도. 결과는 막힘. |
| 2 | [02-why-it-failed.md](./02-why-it-failed.md) | 왜 막혔는지, Type-2(이전 lab)와 비교, upstream(Neutron, OVN-K8s, kube-ovn) 진행도 |
| 3 | [03-g1-via-tor.md](./03-g1-via-tor.md) | ToR에 EVPN을 위임하는 G1 방향 (compute IPv4 unicast → ToR Type-5). ToR 인프라 전이라 gw 노드로 시뮬레이션. |
| 4 | [04-type2-on-compute-type5-on-tor.md](./04-type2-on-compute-type5-on-tor.md) | Hybrid - compute는 Type-2 (pl-cyyoon04 검증 path), 상단은 Symmetric IRB로 Type-5 변환. EVPN 표준 패턴. |

## 환경

- pl-cyyoon05 lab (개인 개발용)
- genestack OpenStack 2025.1 on Kubespray
- kube-ovn 1.15.11 (사내 빌드 `ycy1766/kube-ovn:v1.15.11-evpn` = OVN 26.03.90)
- FRR 10.6.1
- Ubuntu 24.04 (kernel 6.8.0-78)

자세한 노드/IP 구성은 사내 위키 `21. pl-cyyoon05 개발환경 초기 구성` 참고.

## 핵심 결론

- OVN main의 `dynamic-routing` 옵션만으로 EVPN Type-5 광고는 안정적으로 안 됨. 광고 대상 prefix가 nexthop 검증을 통과 못 해서 FRR가 광고 거부.
- Type-2는 OVN의 `controller/evpn-*` 가 데이터 평면을 직접 통합해서 동작. Type-5에는 그런 통합이 없음.
- Neutron 2026.2의 EVPN Service Plugin이 그 gap을 메우려는 작업인데 아직 skeleton 단계.
- 현실적 해결: EVPN encapsulation을 ToR로 위임. compute는 BGP IPv4 unicast만 광고.
