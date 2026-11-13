# 3회차 제출 — 최진서

> 계정: `cc04` / 작성일: 2026-11-06

## 1. 읽기 핵심 (한 장)

- 같은 노드의 VM끼리는 `br-int`에서 전달할 수 있다. 그래서 vm1에서 vm3로 ping이 된다는 사실을 cmp01 내부 경로의 비교 기준으로 삼았다.
- 다른 노드의 VM으로 가는 패킷은 `patch-tun`과 `patch-int`를 통해 `br-tun`으로 넘어가 VXLAN으로 포장된다.
- Local VLAN은 각 노드의 OVS 내부에서 쓰이고 VNI는 노드 사이에서 쓰인다. 그래서 같은 네트워크라도 Local VLAN은 노드별로 확인해야 한다.
- 브리지와 포트가 연결돼 있어도 flow가 패킷을 버릴 수 있다. 어느 포트로 들어왔는지와 VLAN, 우선순위, 동작을 같이 봐야 한다.
- ping은 요청과 응답이 모두 오가야 성공한다. 실패하면 양쪽에서 패킷을 관찰해 어느 쪽까지 도착했는지 구분해야 한다.
- VM 상태나 에이전트 상태는 내부 상태를 보여주는 지표다. VM 사이의 실제 통신은 VM에서 직접 보낸 ping으로 확인했다.

## 2. 받은 증상

프로젝트는 `study-cc04`, 네트워크는 `cc04-net`이다. 아래 내부 IP는 OpenStack에 할당된 값이다.

| VM | 컴퓨트 노드 | 내부 IP | FIP |
| --- | --- | --- | --- |
| vm1 | cmp01 | `10.104.0.52` | `203.0.113.104` |
| vm2 | cmp02 | `10.104.0.88` | `203.0.113.246` |
| vm3 | cmp01 | `10.104.0.46` | 없음 |

| 확인한 것 | 명령                                                                                                                                                                    | 결과 |
| --- |-----------------------------------------------------------------------------------------------------------------------------------------------------------------------| --- |
| VM 상태 | ops01에서 `openstack server list`                                                                                                                                       | 세 VM 모두 `ACTIVE` |
| 같은 노드, vm1 → vm3 | vm1에서 vm3로 ping.<br/><br/>ops01에서 `ssh -o ConnectTimeout=6 -i ~/cc04-key.pem -o IdentitiesOnly=yes -J cc04@192.168.0.12 cirros@203.0.113.104 'ping -c 3 10.104.0.46'` | 3개 송신, 3개 응답 |
| 다른 노드, vm1 → vm2 | vm1에서 vm2로 ping.<br/><br/>ops01에서 `ssh -o ConnectTimeout=6 -i ~/cc04-key.pem -o IdentitiesOnly=yes -J cc04@192.168.0.12 cirros@203.0.113.104 'ping -c 3 10.104.0.88'`           | 3개 송신, 응답 없음 |
| vm2 FIP SSH | vm2의 FIP로 SSH 접속<br/><br/>ops01에서 `ssh -o ConnectTimeout=6 -i ~/cc04-key.pem -o IdentitiesOnly=yes -J cc04@192.168.0.12 cirros@203.0.113.246 'ping -c 3 10.104.0.52'`                           | `Connection timed out during banner exchange`. SSH 접속 단계에서 실패해 이 명령의 ping은 실행되지 않음 |
| 다른 노드, vm2 → vm1 | VM2에서 VM1로 ping<br/><br/>cmp02에서 `sudo docker exec -it nova_libvirt virsh console a4bdaf52-462a-4231-91c1-fc7b020c2154`로 접속한 뒤 vm2에서 `ping -c 3 10.104.0.52`                                   | 3개 송신, 응답 없음. `169.254.107.93`에서 `Destination Host Unreachable` 출력 |

## 3. 가설과 검증 기록

| # | 가설 | 확인 방법                                                                                                                                                                                                                                                                                                                                        | 결과                                                                                                                                                                                                                                                                | 판단 |
| --- | --- |----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------| --- |
| 1 | 지난주처럼 vm2의 포트가 `br-int`에서 빠졌을 수 있다. | VM2의 네트워크 포트를 확인:<br/>ops01에서 `openstack port list --server cc04-vm2`<br><br/>cmp02의 실제 가상 스위치 연결 구성을 확인:<br/>cmp02에서 `sudo docker exec openvswitch_vswitchd ovs-vsctl show`                                                                                                                                                                 | Neutron 포트는 `ACTIVE`였다. `qvo1ea2b1af-d3`는 `br-int`에 있고 태그는 `8`이었다. `patch-tun`과 `patch-int`도 연결돼 있었다.                                                                                                                                                             | 기각. 다음으로 실제 패킷이 어디까지 가는지 확인했다. |
| 2 | vm1에서 보낸 요청이 노드 사이 VXLAN 구간에서 유실될 수 있다. | cmp01과 cmp02의 `eth0`에서 VXLAN을 동시에 캡처했다:<br/>`sudo timeout 16s tcpdump -l -n -i eth0 -c 12 'udp dst port 4789 and host 192.168.0.21 and host 192.168.0.22 and udp[12:4] = 0x00024800'`<br/> <br/>cmp02의 vm2 `tap`에서 ICMP를 확인했다: <br/>`sudo timeout 12s tcpdump -l -n -i tap1ea2b1af-d3 -c 6 'icmp and host 10.104.0.52 and host 10.104.0.88'` | 두 노드에서 VNI `584`로 포장된 같은 ICMP 요청 3개를 확인했다. ICMP ID는 `45852`, seq는 1~3이었다. vm2의 `tap`에도 요청 3개가 들어왔고 응답은 보이지 않았다.                                                                                                                                                   | 요청 방향의 터널 유실 가설은 기각. vm1 → vm2 요청은 VM2의 `tap`까지 도착했다. |
| 3 | cmp02에서 vm2 쪽 트래픽을 터널로 보내기 전에 flow가 버릴 수 있다. | br-tun 가상 스위치에 연결된 포트의 이름과 번호, 상태를 확인:<br/>cmp02에서 `sudo docker exec openvswitch_vswitchd ovs-ofctl show br-tun`<br/><br/>패킷 처리 규칙 확인:<br/> `sudo docker exec openvswitch_vswitchd ovs-ofctl dump-flows br-tun`으로 포트 번호와 flow를 대조했다.      | `patch-int`는 OFPort `1`, cmp01 쪽 터널 포트는 `5`였다.<br/> <br/>패킷 처리 규칙은 `table=0`에 `in_port=1,dl_vlan=8`을 버리는 우선순위 `30000` 규칙이 있었다(1번 포트로 들어온 VLAN 8 패킷은 첫 처리 단계에서 버림). <br/>송신 trace는 여기서 `drop`(1번가설에서 cmp02의 Local VLAN 8인거 확인)| 송신 쪽 flow 문제 확인. 해당 조건에 적용되는 규칙은 OVS trace로 확인했다. |


## 4. 원인과 복구 방안

- **원인 (한 문장)** — cmp02의 `br-tun` table 0에서 Local VLAN `8`이 `patch-int`로 들어오면 우선순위 `30000`의 규칙이 일반 전달 규칙보다 먼저 적용돼 VXLAN으로 포장하기 전에 패킷을 버린다.
- **끊긴 지점이 이 증상으로 보인 이유** — vm1과 vm3의 통신은 cmp01 안에서 처리된다. vm2 쪽으로는 요청이 `tap`까지 도착했지만, vm2에서 나가는 VLAN 8 트래픽은 cmp02의 터널 입구에서 막힌다. 
- **복구하려면** — cmp02의 `br-tun`에서 cookie `0x330304`, table `0`, priority `30000`, `in_port=1,dl_vlan=8`인 drop 규칙만 확인해 제거한다. 
- **복구 후 확인** — vm2의 `ip -4 addr show dev eth0`로 IP를 확인하고, 2번 표의 vm1 → vm2, vm2 → vm1, vm1 → vm3 ping과 vm2 FIP SSH를 다시 확인한다.

## 5. SRE 세 질문

1. **사용자가 무엇을 겪었나** — 같은 노드의 vm1과 vm3는 통신했지만, vm2와는 양방향 내부 IP 통신이 실패했다. vm2 FIP SSH도 시간 초과가 났다.
2. **어떤 증상으로 알아챘나** — 같은 노드의 vm1과 vm3는 ping에 응답했지만, 다른 노드의 vm2와는 통신이 실패했다. 두 컴퓨트 노드의 OVS 에이전트 컨테이너는 정상 상태였고, 패킷 경로와 flow를 확인해 VM2의 송신을 막는 규칙을 찾았다.
3. **운영자가 미리 볼 수 있었던 지표는** 
    - **Black-box** — vm1과 vm2에서 상대 VM의 내부 IP로 각각 주기적으로 ping을 보내 성공률을 기록하고, vm1 → vm3 결과와 비교한다.
    - **White-box** — cmp02에서 Local VLAN별 drop flow 카운터와 VXLAN 포트의 송수신 카운터를 비교한다.

## 6. 한 줄 회고

- 양쪽에서 패킷을 보니 vm2로 가는 길과 vm2에서 나오는 길을 따로 판단할 수 있어서 장애를 찾을 수 있었다!