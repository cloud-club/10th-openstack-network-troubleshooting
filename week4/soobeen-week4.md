# 4회차 제출 — 손수빈

> 계정: `cc03` · 작성일: 2026-11-16
> 

## 1. 읽기 핵심 (한 장)

- **포트와 VM 연결**
    - Neutron이 VM 부팅 때 포트를 만들고 IP·MAC을 묶어 DB에 저장 → DHCP는 그 값을 게스트에 **전달**할 뿐이라, Horizon의 ACTIVE·IP와 게스트가 실제로 받은 값은 따로 확인해야 함
    - 포트의 IP·MAC과 다른 소스 트래픽은 컴퓨트 노드 iptables에서 조용히 DROP → 패킷이 **포트·장치 단위로** 말없이 버려질 수 있음
- **DHCP와 메타데이터**
    - DHCP: `qdhcp-<네트워크 ID>` 네임스페이스의 dnsmasq가 DISCOVER → OFFER → REQUEST → ACK로 주소와 옵션을 줌
    - 메타데이터: router 네임스페이스는 PREROUTING으로 9697 포트 proxy에, DHCP 네임스페이스는 80 포트 proxy에 보내고, VM은 option 121로 받음
    - 신고 A, B에 해당
- **MTU**
    - 1500 − VXLAN 50 = 1450이 기준, DHCP agent가 IPv4 VM에 이 값을 전달한다
    - VM의 `eth0` mtu, 네트워크 `mtu`, 경로 장치 mtu 중 하나라도 다르면 큰 패킷만 막힐 수 있기에 장치별 mtu를 비교

vm1이 지나는 경로

```
vm1(eth0) → tap → qbr → qvb ─ qvo → br-int → … → VXLAN → cmp02 …   (vm2와 장치별 비교)
```

## 2. 받은 증상

**내가 받은 신고: C**

신고 판별 근거: `openstack server list`에 `cc03-vm1~3`만 ACTIVE로 있고 `cc03-vm4`가 없음

| 확인한 것 | 명령 | 결과 |
| --- | --- | --- |
| 서버 목록, vm4 존재 | `openstack server list` | cc03-vm1, vm2, vm3 ACTIVE (cc03-net), **vm4 없음** → 신고 C |
| net01에서 FIP로 ping, 작은 크기 | `ping -c2 -s 1200 203.0.113.205` (IP 패킷 1228) | 2 received |
| net01에서 FIP로 ping, 큰 크기 | `ping -c2 -s 1400 203.0.113.205` (IP 패킷 1428) | **0 received** |

## 3. 가설 · 검증 기록 ★

**가장 중요한 부분입니다.** 순서대로, 틀린 가설도 지우지 말고 남기세요.

| # | 가설 (어디가 문제일 것 같다) | 확인 방법 (명령) | 결과 | 판단 |
| --- | --- | --- | --- | --- |
| 1 | 서버 환경이 문제다 | net01에서 vm1·vm2 FIP로 ping, vm2 FIP로 큰 패킷 ping | vm1, vm2 FIP 모두 기본 ping 응답 | 기각 |
| 2 | OpenStack이 보는 상태(포트, SG, 키, 서브넷)가 vm1만 vm2와 다르다 | ops01에서 `openstack server show`(vm1, vm2), `openstack port list --network cc03-net`, `openstack security group rule list cc03-sg` | vm1·vm2 모두 status ACTIVE, flavor `m1.study` 등으로 같음 | 기각 |
| 3 | 출력이 긴 명령이면 vm1에서 멈춘다 | vm1에서 `seq 1 50000` | **17967까지 출력되고 멈춤** | 유지 |
| 4 | vm1 경로에서 큰 패킷만 막힌다 | net01에서 `for n in 1250 1300 1350 1400; do ping -c2 -W1 -M do -s $n 203.0.113.205; done`, 이어서 `for n in 1272 1273; do …; done` (IP 패킷 = N+28) | vm1 FIP: `-s 1250` 2/2 응답, `-s 1300`·`-s 1350`·`-s 1400` 0/2. 경계 정밀화: `-s 1272`(IP 1300) 2/2 응답, `-s 1273`(IP 1301) 0/2 (100% loss). vm2 FIP: `-s 1250`, `-s 1400` 모두 2/2 응답 | 유지 |
| 5 | net01 qrouter(FIP 경로)의 인터페이스 mtu가 낮다 | `sudo ip netns exec qrouter-10322d7c-163e-4c09-a7db-a2acdab5f36b ip -o addr`, `… ip -o link` | FIP 203.0.113.205는 `qg-5801a520-01`(mtu 1500)에 있음. 내부측 `qr-bd669f9f-3f` mtu 1450 | 기각 |
| 6 | 경로 장치 중 하나의 mtu가 다르다 | cmp01에서 `ip -o link | grep b88849a1-1c`, cmp02에서 `ip -o link | grep eb99e8d2-ed` | vm1(cmp01): `qbrb88849a1-1c`, `qvob88849a1-1c`, `qvbb88849a1-1c` mtu **1300**, `tapb88849a1-1c` mtu **1450**. vm2(cmp02): `qbreb99e8d2-ed`, `qvoeb99e8d2-ed`, `qvbeb99e8d2-ed`, `tapeb99e8d2-ed` 모두 mtu **1450** | 유지 |
| 7 | VM 안 값이나 DHCP가 준 값이 vm1만 다르다 | vm1에서 `ip link show eth0`, ops01에서 `openstack network show cc03-net -c mtu` | vm1 `eth0` mtu **1450**, 네트워크 mtu **1450** | 기각 |
| 8 | cmp01에서 실제로 패킷이 버려지는 기록이 있다 | cmp01에서 `sudo dmesg | grep -i mtu | tail` | `qvob88849a1-1c: dropped over-mtu packet: 1450 > 1300`(2건), `1328 > 1300`(2건), `1378 > 1300`(2건), `1428 > 1300`(2건), `1301 > 1300`(2건) | 유지 |

## 4. 원인 · 복구 방안

복구는 **실행하지 않습니다.** 어떻게 고치면 될지만 적습니다.

- **원인 (한 문장):** cmp01에서 vm1 포트의 `qbr`·`qvb`·`qvo`의 mtu가 1300으로 낮아져 있어(tap·VM `eth0`·vm2 장치는 1450), IP 패킷이 1300바이트를 넘으면 `qvo`에서 버려진다.
- **끊긴 지점이 이 증상으로 보인 이유:**
    - 기본 ping(64바이트)과 SSH 핸드셰이크는 1300보다 작아 통과한다. VM은 `eth0` mtu 1450만 알고 크기가 큰 패킷을 보내는데, 1300을 넘는 패킷은 veth·브리지 같은 L2 장치가 ICMP fragmentation needed를 돌려주지 않고 조용히 버려서 VM이 크기를 줄이지 못한다. 그래서 짧은 입력은 되고 긴 출력만 멈춘 것처럼 보인다.
    - vm2는 장치가 전부 1450이라 같은 크기가 통과해서 정상이다. 대시보드의 `MTU1400` 프로브는 vm2만 재고 vm1을 향한 큰 패킷 프로브가 없어서 계속 초록이다.
    - 증상 재현: vm1에서 `seq 1 50000`을 실행하면 17967에서 멈췄다. 처음에는 작은 패킷이 통과하다가 1300을 넘는 첫 큰 TCP 세그먼트가 버려지면 재전송도 같은 크기라 계속 버려져 멈추는 것 같다.
    - 확정 근거는 패킷 크기에 따른 ping 확인과 cmp01의 `dropped over-mtu packet` 기록
- **복구하려면 (명령 또는 절차):**
    
    cmp01에서 `qbr`·`qvb`·`qvo`의 mtu를 네트워크 값 1450으로 되돌린다: `sudo ip link set dev qvob88849a1-1c mtu 1450`, `sudo ip link set dev qvbb88849a1-1c mtu 1450`, `sudo ip link set dev qbrb88849a1-1c mtu 1450`
    
- **복구 후 무엇으로 확인하면 되는지:**
    - net01에서 `ping -c2 -W1 -M do -s 1400 203.0.113.205`가 2/2 응답
    - cmp01에서 `ip -o link | grep b88849a1-1c`의 네 장치가 모두 mtu 1450
    - cmp01에서 `sudo dmesg | grep -i mtu | tail`에 새 `dropped over-mtu packet` 줄이 늘지 않음

## 5. SRE 세 질문

1. **사용자가 무엇을 겪었나:**
    - SSH는 접속되는데 긴 출력이나 파일 전송에서 화면이 멈추고, 재접속하면 다시 들어간다. ping도 되고 vm2는 정상
2. **어떤 증상으로 알아챘나:** vm1에서 `seq 1 50000`을 실행하면 17967에서 멈췄다
3. **운영자가 미리 볼 수 있었던 지표는:**
    - 기존 프로브는 vm1 문제를 못 잡았다. 이유는 두 가지다. 
    ① 큰 패킷 프로브인 `3 FIP MTU1400`의 대상이 **vm2뿐**(`net01→vm2`)이고 vm1을 향하는 큰 패킷 프로브는 없었다. 
    ② vm1을 향하거나 vm1에서 나가는 나머지 프로브는 작은 패킷만 지나 1300 이하라 통과한다.
    - **Black-box 제안:** `3 FIP MTU1400`을 vm1에도 추가(`net01→vm1`)

## 6. 한 줄 회고

- 평소에 MTU가 중요한 건 알았지만, 딱히 문제가 되는 경우는 없어 와닿지는 않았는데 이렇게 패킷이 버려지는 경우를 경험해 보니 좋았다. 또한, 이제 구조가 익어 빠르게 발견할 수 있었다.