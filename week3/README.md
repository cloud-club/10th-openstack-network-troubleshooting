# 3회차 — 노드 사이를 건너면: vm2만 통신이 안 된다

> 세션: 2026-11-09 (월) 22:00 디스코드 · 제출 마감: 2026-11-08 (일) 23:59

## 0. 시작 전 확인

- 2회차 복구 이후 본인 VM의 정상 기준선을 다시 확인합니다. `vm1(cmp01)`, `vm2(cmp02)`, `vm3(cmp01)`의 위치와 IP를 메모하세요.
- 본인 네트워크의 **VNI**(`provider:segmentation_id`)를 미리 적어두세요. 이번 주 확인 지점에서 계속 쓰입니다.
- Grafana(`localhost:3001`)에서 본인 FIP 프로브의 변화 시점을 확인하세요.

## 1. 읽기 범위

| 교재                            | 범위                                                                                                                       | 읽으면서 볼 것                                                                                               |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| Learning OpenStack Networking | **4장 PDF p.102–112** (Open vSwitch, internal connections, flow rules)                                                    | `br-int`, `br-tun`, `patch-int`/`patch-tun`과 flow가 VM 트래픽을 어떻게 이어주는지. **Local VLAN과 VNI가 각각 어디서 쓰이는지** |
| SRE                           | **6장 PDF p.84–88** (Symptoms vs Causes / Black-box vs White-box)                                                         | 에이전트가 살아 있다는 지표와 실제 패킷이 통한다는 사실이 왜 다른지                                                                 |
| 보강 자료 (선택)                    | [Neutron 2026.1: OVS Self-service networks](https://docs.openstack.org/neutron/2026.1/admin/deploy-ovs-selfservice.html) | 특히 **East-west scenario 1**의 컴퓨트 노드 간 패킷 경로, 캡슐화·디캡슐화 지점과 VXLAN 헤더                                     |

> 쪽수는 **PDF 뷰어 페이지 기준**입니다. 1판의 개념은 활용하되, 명령과 구현은 실습 환경의 실제 구성을 우선합니다.

## 2. 장애 신고

> **신고자:** 스터디원
>
> **내용:** 같은 프로젝트의 vm1과 vm3는 서로 통신하는데, **다른 컴퓨트 노드에 있는 vm2와는 통신이 되지 않습니다.** Horizon에서 VM은 모두 ACTIVE이고, 네트워크 에이전트도 Alive로 표시됩니다. 제가 건드린 건 없습니다.

**대상**

- 프로젝트: `study-<본인ID>`
- 비교: `<본인ID>-vm1`·`<본인ID>-vm3` (`cmp01`)
- 증상: `<본인ID>-vm2` (`cmp02`)

**참고**

- 원인은 **한 군데**이고, 사람마다 다를 수 있습니다. 같은 "vm2 통신 불가"라도 **어느 방향·어느 경로가 실패하는지**는 다를 수 있으니 옆 사람 답이 내 답이 아닐 수 있습니다.
- vm2의 FIP 접속 같은 다른 경로의 결과도 사람마다 다를 수 있습니다. 되는 것과 안 되는 것을 모두 기록하세요.
- 본인 환경에서 증상이 재현되지 않는다면, 그 사실과 정상 경로를 확인한 증거를 그대로 기록하면 됩니다.

## 3. 요구

`vm1(cmp01) ↔ vm2(cmp02)`의 **East-West 경로**를 **양방향으로** 쫓아, 마지막으로 정상 확인된 위치와 처음 실패한 위치를 표시하세요.

- 실패가 **송신 쪽**(vm2에서 나가는 패킷이 캡슐화되기 전)인지 **수신 쪽**(cmp02에 도착한 VXLAN 패킷이 vm2로 전달되기 전)인지 구분하세요.
- `vm1 → vm3`가 된다면 이 비교가 어떤 구간을 배제하는지 설명하세요.
- 실패 지점, 판단 근거, 복구 제안(미실행)을 제출합니다.

## 4. 확인 지점 체크리스트

위에서부터 순서대로 좁혀가세요. 각 지점에서 **무엇을 확인했고 결과가 어땠는지**를 제출물에 남깁니다.

| # | 지점 | 쓸 만한 확인 방법 |
| --- | --- | --- |
| ① | VM·포트 배치와 네트워크 VNI | `openstack server show <본인ID>-vm2` · `openstack port list --server <본인ID>-vm2` · `openstack network show <본인ID>-net` (`provider:segmentation_id`) |
| ② | 비교 테스트: 같은 노드 / 다른 노드 / 방향 / FIP | vm1→vm3, vm1→vm2, **vm2→vm1** ping 비교 · ops01→vm2 FIP 점검 · Grafana 프로브 시각 기록 |
| ③ | `cmp02`에서 vm2 포트의 **Local VLAN**과 `br-int`↔`br-tun` 연결, `br-tun` 포트 번호 | `sudo docker exec openvswitch_vswitchd ovs-vsctl show` (`qvo<포트ID앞11자리>`의 `tag`, `patch-int`/`patch-tun`) · `sudo docker exec openvswitch_vswitchd ovs-ofctl show br-tun` (`patch-int`·`vxlan-*` 포트의 OFPort 번호) |
| ④ | 노드 간 VXLAN 패킷을 실제로 보았는가 (양쪽 노드, 양방향) | `sudo tcpdump -ni any -c 30 'udp port 4789'` (cmp01/cmp02에서 짧게 비교, 터널 양끝 IP·VNI 확인) |
| ⑤ | `cmp02`의 `br-tun` flow — `in_port`·VNI(`tun_id`)·Local VLAN(`dl_vlan`) 매칭 조건, **`priority`**, **패킷 카운터** | `sudo docker exec openvswitch_vswitchd ovs-ofctl dump-flows br-tun` (`table=0`부터 순서대로 읽고, 같은 명령을 ping 전후로 두 번 실행해 `n_packets` 변화 비교. `in_port` 번호는 ③의 OFPort와 대조) |
| ⑥ | `cmp02`의 `br-int`에서 vm2로 이어지는 경로 | `sudo docker exec openvswitch_vswitchd ovs-ofctl dump-flows br-int` · `ip link` |
| ⑦ | 결론 | 송신·캡슐화·수신·디캡슐화 중 어느 **구간까지** 증거가 있는지, 실패 방향과 그 구간을 식별한 값(VNI 또는 Local VLAN)을 명시 |

> Local VLAN은 **노드 안에서만** 쓰이는 번호이고, VNI는 **노드 사이 터널**에서 쓰이는 번호입니다. 같은 네트워크라도 cmp01과 cmp02의 Local VLAN 값은 다를 수 있으니 ③·⑤에서 노드별로 따로 확인하세요.

> `openstack network agent list`나 `ovs-vsctl show`에서 브리지·포트가 모두 정상으로 보여도, 패킷을 실제로 어떻게 처리할지는 **flow 규칙**이 결정합니다. OVS는 여러 flow가 매칭되면 `priority`가 가장 높은 규칙의 `actions`를 실행합니다. "구성이 존재한다"와 "패킷이 전달된다"를 구분해서 확인하세요.

> `tcpdump -i any`와 `dump-flows`는 다른 참가자의 트래픽도 함께 보입니다. 특정 패킷·flow를 본인 것이라 단정하지 말고, VM IP·터널 양끝 IP·VNI·Local VLAN·시각을 함께 대조합니다. 명령은 **조회 전용**으로 사용하세요.

## 5. 해결 조건

- 같은 노드 통신과 다른 노드 통신의 **경로 차이**를 설명할 수 있을 것
- 실패가 **송신 방향인지 수신 방향인지**, 그리고 OVS의 어느 단계에서 패킷이 멈췄는지 근거와 함께 제시할 것
- `agent Alive`와 `dataplane 정상`을 구분하고, 장애가 **구성 요소(브리지·포트)가 사라진 문제인지 패킷 처리 규칙의 문제인지** 근거와 함께 판단할 것
- SRE: 이 장애를 사용자 신고 전에 잡으려면 **어떤 Black-box 프로브(양방향)와 White-box 신호(flow 카운터, 인터페이스 통계 등)**가 필요한지 한 줄씩 제안할 것
- 복구 방법은 문서로만 제안하되, **정상 구성은 건드리지 않고 원인만 되돌리는** 방법인지 설명할 것
- (못 풀었다면) 어디까지 확인했고 어디서 막혔는지 기록이 있을 것

## 6. 규칙

- 허용: **조회, 제한된 ping/curl/SSH 확인, tcpdump,** `ovs-vsctl show`**·**`ovs-ofctl dump-flows` **등 OVS 조회, 로그·통계 읽기**. 가능한 한 본인 리소스로 대상을 한정합니다.
- 금지: 네트워크·라우터·포트·SG·FIP 변경, `ip link set`, `ovs-vsctl add-port/del-port`, `ovs-ofctl add-flow/del-flows/mod-flows`, `iptables`/`nft` 규칙 변경, VM·컨테이너·에이전트 재시작, `reconfigure`.
- 공용 노드에서 대량 트래픽을 발생시키거나 전체 패킷을 장시간 캡처하지 마세요. `tcpdump -c` 등으로 짧게 관찰하세요.
- 복구는 실행하지 않고 **방법만 제안**합니다. 실제 복구는 운영진이 세션에서 진행합니다.
- 노드 접속은 본인 계정으로만 합니다. 권한이 없는 확인 지점은 운영진에게 요청하고 그 사실을 기록하세요.

## 7. 제출

- `week3/<본인이름>.md` — 루트 `template.md`를 복사해 작성합니다.
- 브랜치: `week3/<본인이름>` → PR 제목: `[week3] <본인이름>` (루트 README의 **Git으로 제출하기** 참고).
- 제출 마감: **2026-11-08 (일) 23:59**.
- 세션에서 2~3명 랜덤 발표합니다. **정답이 없어도 검증 기록과 가설 변경 과정이 있으면 발표할 수 있습니다.**
