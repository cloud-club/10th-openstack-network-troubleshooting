# 2회차 제출 - 최진서

> 계정: `cc04` / 작성일: 2026-11-01

## 1. 읽기 핵심 (한 장)

- VM이 ACTIVE여도 실제 네트워크 연결은 끊겨 있을 수 있으므로 실행 상태와 통신 상태를 구분했다.
- 이번 환경에서는 `tap → qbr → qvb/qvo → br-int` 순서로 연결 관계를 확인해야 한다.
- 포트 UUID 앞 11자리로 VM2와 연결된 호스트의 인터페이스를 찾을 수 있었다.
- 인터페이스가 UP인지만 보는 것이 아니라 올바른 브리지에 연결돼 있는지도 확인해야 한다.
- SRE의 가설 검증 방식에 따라 FIP와 보안그룹을 확인한 뒤 라우터에서 VM2 내부 IP로 검사하며 범위를 좁혔다.
- 정상 VM1과 비교하며 검증했다.

## 2. 받은 증상

대상은 `cc04-vm2`, cmp02에서 실행 중. 내부 IP는 `10.104.0.88`, FIP는 `203.0.113.246`.

| 확인한 것 | 명령과 실행 위치 | 결과 |
| --- | --- | --- |
| VM 상태 | ops01: `openstack server list` | VM1과 VM2 모두 ACTIVE |
| VM1 SSH | ops01: `ssh -i ~/cc04-key.pem -o IdentitiesOnly=yes -J cc04@192.168.0.12 cirros@203.0.113.104` | 정상 접속 |
| VM1에서 VM3 ping | VM1: `ping -c 4 10.104.0.46` | 4회 모두 응답 |
| VM2 SSH | ops01: `ssh -i ~/cc04-key.pem -o IdentitiesOnly=yes -J cc04@192.168.0.12 cirros@203.0.113.246` | `No route to host`, `stdio forwarding failed` |
| VM1 FIP ping | net01: `ping -c 3 -W 2 203.0.113.104` | 3회 모두 응답 |
| VM2 FIP ping | net01: `ping -c 3 -W 2 203.0.113.246` | `Destination Host Unreachable`, 응답 없음 |

## 3. 가설과 검증 기록

| # | 가설 | 확인 방법 (명령) | 결과 | 판단 |
| --- | --- | --- | --- | --- |
| 1 | FIP가 VM2 포트에 잘못 연결됐을 것이다 | ops01:<br>`openstack port list --server cc04-vm2`<br>`openstack port show 1ea2b1af-d3cf-4e2b-9bd8-251a77d7d1fe`<br>`openstack floating ip list` | FIP `203.0.113.246`은 VM2 포트와 내부 IP `10.104.0.88`에 연결돼 있음. | FIP 연결 오류는 기각. |
| 2 | 보안그룹 규칙이 ping과 SSH를 차단했을 것이다 | ops01:<br>`openstack security group rule list cc04-sg` | IPv4 ICMP와 TCP 22가 `0.0.0.0/0`에서 허용됨 | 설정된 보안그룹 규칙 누락은 기각 |
| 3 | FIP를 거치지 않고 라우터에서 내부 IP로 접근하면 통신할 수 있을 것이다 | net01:<br>`sudo ip netns exec qrouter-89c8239b-94db-4284-aa6e-4b24ebb1bc7f ping -c 3 -W 2 10.104.0.52`<br>`sudo ip netns exec qrouter-89c8239b-94db-4284-aa6e-4b24ebb1bc7f ping -c 3 -W 2 10.104.0.88` | VM1 내부 IP는 응답하지만 VM2는 `Destination Host Unreachable`.  | 내부 통신도 가능하다는 가설은 기각. VM2까지의 L2 연결을 확인해야 함 |
| 4 | cmp02에서 VM2의 가상 인터페이스와 브리지 연결이 끊겼을 것이다 | cmp02:<br>`ip -d link show tap1ea2b1af-d3`<br>`ip -d link show qbr1ea2b1af-d3`<br>`ip -d link show qvb1ea2b1af-d3`<br>`ip -d link show qvo1ea2b1af-d3`<br>`sudo docker exec openvswitch_vswitchd ovs-vsctl show`<br>`sudo docker exec openvswitch_vswitchd ovs-vsctl port-to-br qvo1ea2b1af-d3` | TAP, qbr, qvb/qvo는 UP이며 TAP과 qvb는 qbr에 연결됨. 하지만 qvo가 br-int 포트 목록에 없고 `no port named qvo1ea2b1af-d3` 출력 | 원인으로 특정. qvo와 br-int 사이 연결 누락 |

### 끊긴 지점

```text
VM2 ↔ tap1ea2b1af-d3 ↔ qbr1ea2b1af-d3 ↔ qvb1ea2b1af-d3
                                           ↕
                                     qvo1ea2b1af-d3
                                           ×
                                         br-int
```

cmp02에서 확인한 결정적인 출력:

```text
$ sudo docker exec openvswitch_vswitchd ovs-vsctl port-to-br qvo1ea2b1af-d3
ovs-vsctl: no port named qvo1ea2b1af-d3
```

## 4. 원인과 복구 방안

- **원인 (한 문장):** cmp02의 `qvo1ea2b1af-d3`가 Linux 인터페이스로는 존재하지만 OVS `br-int`에 등록돼 있지 않아 VM2의 L2 연결이 끊겼다.
- **끊긴 지점이 이 증상으로 보인 이유:** VM 프로세스는 실행 중이므로 ACTIVE로 보이지만, OVS와 연결되지 않아 라우터가 VM2의 ARP 응답을 받지 못한다. 따라서 보안그룹이 허용돼 있어도 ping과 SSH가 함께 실패한다.
- **복구하려면:** 운영진이 해당 qvo를 br-int의 OVS 포트로 정상 복원하고, Neutron 포트 식별 정보와 필요한 VLAN 및 flow가 올바르게 반영되는지 확인한다. 
- **복구 후 확인:** qvo의 br-int 소속, Neutron 포트 상태, 내부 IP와 FIP ping, 실제 SSH를 순서대로 확인한다.

## 5. SRE 세 질문

1. **사용자가 무엇을 겪었나:** VM1은 접속되지만 VM2는 ACTIVE로 표시되면서도 SSH와 ping이 모두 실패했다.
2. **어떤 증상으로 알아챘나:** SSH의 `No route to host`, ping의 `Destination Host Unreachable`, Neutron 포트 DOWN과 ARP 응답 부재를 확인했다.
3. **운영자가 미리 볼 수 있었던 지표는:** FIP 프로브 성공 여부로 사용자 영향을 확인하고, Neutron 포트 상태와 OVS 인터페이스의 존재 여부로 내부 단서를 확인할 수 있다.

## 6. 한 줄 회고

- 인터페이스가 UP이어도 스위치 연결은 빠질 수 있었고, 정상 VM과 비교하며 실제 연결 관계를 따라가야 원인을 찾을 수 있었다.
