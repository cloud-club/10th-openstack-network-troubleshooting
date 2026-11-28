# 6회차 제출 — hyeongjun

> 계정: `cc01` · 작성일: 2026-11-28
> 본인 증상: **B — VM에 들어가는 건 되는데, VM 안에서 밖으로 나가는 게 안 된다 (vm1~vm3 전부)**

## 0. 내 환경 메모

| 항목 | 값 |
| --- | --- |
| 시큐리티 그룹 | `cc01-sg` · id `2ed34f53-1341-46f8-9058-b4640faf36a5` · revision 5 · **updated_at 2026-11-24T03:58:19Z (KST 11/24 12:58)** |
| 서브넷 / 게이트웨이 | `cc01-subnet` 10.101.0.0/24 / 10.101.0.1 (`qr-fff49bcc-fa`) |
| 라우터 네임스페이스 | `qrouter-5ec32c60-6a9b-47cc-bb1d-9266a120bf27` (net01) |
| vm1 (cmp01) | 10.101.0.15 · FIP 203.0.113.214 · 포트 `3a5bc23b-2dd5-…` → 장치 `tap/qbr/qvb/qvo3a5bc23b-2d` · 체인 `…3a5bc23b-2` · MAC `fa:16:3e:c9:a6:77` |
| vm2 (cmp02) | 10.101.0.7 · FIP 203.0.113.178 · 포트 `a49896ed-e537-…` → 장치 `…a49896ed-e5` · 체인 `…a49896ed-e` |
| vm3 (cmp01) | 10.101.0.180 · FIP 없음 · 포트 `6925aa2d-0f1d-…` → 장치 `…6925aa2d-0f` · 체인 `…6925aa2d-0` |
| 세 포트 공통 | `security_group_ids = 2ed34f53-…` (cc01-sg) · `port_security_enabled True` |

## 1. 읽기 핵심 (한 장)

### Learning OpenStack Networking 8장 — 시큐리티 그룹이 노드에서 구현되는 방식

```
VM eth0 → tap3a5bc23b-2d ─┐
                          qbr3a5bc23b-2d (Linux bridge, bridge-nf-call-iptables=1 → iptables 통과)
                          └─ qvb3a5bc23b-2d ═veth═ qvo3a5bc23b-2d → br-int

iptables (cmp01, filter):
FORWARD → neutron-openvswi-FORWARD
        → (physdev-in  tap…) neutron-openvswi-sg-chain → neutron-openvswi-o3a5bc23b-2  (VM이 보내는 방향 = egress)
        → (physdev-out tap…) neutron-openvswi-sg-chain → neutron-openvswi-i3a5bc23b-2  (VM이 받는 방향 = ingress)
o 체인: DHCP 허용 → s 체인(IP/MAC 스푸핑 검사) → DHCP 위장 차단 → RELATED,ESTABLISHED RETURN → INVALID DROP
        → [ SG egress 규칙이 바뀐 줄들 ] → sg-fallback (DROP)
```

- **SG 규칙은 무엇으로 바뀌어 어디에 깔리나:** API의 SG 규칙 한 줄이 VM이 올라간 **compute 노드의 iptables 한 줄**(i/o 체인의 `-j RETURN`)로 바뀐다. vm1은 cmp01, vm2는 cmp02에 각각 깔린다. OVS 포트에는 iptables를 걸 수 없어서(hybrid 방식) VM과 br-int 사이에 **Linux bridge `qbr`** 를 끼워 넣고, `bridge-nf-call-iptables=1`로 브리지를 지나는 프레임이 iptables를 타게 한다.
- **체인 순서가 결과를 정한다:** o/i 체인은 위에서부터 처음 맞는 줄에서 끝난다. 상태 기반이라 `RELATED,ESTABLISHED` 줄이 규칙 줄들보다 **앞에** 있다. 그래서 한 번 허용된 연결의 응답은 SG 규칙과 상관없이 통과하고, **새 연결(NEW)만** 규칙 줄들로 판정된다. 규칙에 안 맞으면 마지막 `sg-fallback`의 DROP으로 떨어진다.
- **새 SG에 처음부터 들어 있는 규칙:** SG를 만들면 **egress IPv4 any, egress IPv6 any** 두 줄이 기본으로 들어 있다. 그래서 평소엔 egress를 신경 쓸 일이 없고, "22번·ICMP 열었는지"만 확인하게 된다. 이번 장애는 바로 그 기본 두 줄이 빠진 상태였다.
- 책과 우리 환경의 차이: 체인 접두어 `neutron-linuxbri-*` → `neutron-openvswi-*`. 체인 이름의 포트 ID는 10자(`3a5bc23b-2`), 장치 이름은 11자(`3a5bc23b-2d`). CLI는 `neutron security-group-*` → `openstack security group …`.

### SRE 4장 — SLI·SLO

- SLI는 **사용자가 겪는 것**에서 골라야 한다. 이번 장애에서 사용자가 겪은 건 "VM 안에서 밖으로 못 나간다"인데, 기존 프로브(FIP ping·SSH)는 **밖에서 안으로** 들어오는 방향만 잰다. 그래서 프로브는 초록인데 사용자는 실패하는 상황이 된다.
- SLO를 100%로 잡지 않는 이유: 프로브 자체의 흔들림과 정상 작업(유지보수)까지 장애로 세면 알람이 신뢰를 잃는다. 대신 그 여유(에러 버짓)를 넘는 순간을 놓치지 않는 것이 목표다.

## 2. 받은 증상 (① 방향을 구분한 증상 표)

| VM | FIP ping (net01) | FIP SSH (net01) | 네임스페이스 SSH (qrouter→fixed IP) | → 게이트웨이 10.101.0.1 | → 옆 VM | → 203.0.113.1 | → 8.8.8.8 | curl / 메타데이터 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| vm1 (cmp01) | ✅ 3/3 | ✅ OK | ✅ | ❌ 0/2, 0/3 | ❌ vm2 0/2 · vm3 0/2 | ❌ 0/2 | ❌ 0/2 | ❌ `000` / ❌ `000` |
| vm2 (cmp02) | ✅ 3/3 | ✅ OK | ✅ | (첫 루프 결과, 출력 구분 불명확) | — | ❌ 0/3 | ❌ 0/3 | ❌ `000` |
| vm3 (cmp01) | 미적용 (FIP 없음) | 미적용 | ✅ | (첫 루프 결과, 출력 구분 불명확) | — | ❌ 0/3 | ❌ 0/3 | ❌ `000` |

- 명령: 들어오는 방향은 net01 호스트에서 `ping -c3 <FIP>` / `ssh -o ConnectTimeout=5 -i ~/cc01-key.pem cirros@<FIP> true`, `sudo ip netns exec qrouter-5ec32c60-… ssh … cirros@<fixed IP>`. 나가는 방향은 그 SSH로 VM 안에서 `ping -c2 -W1 <대상>`, `curl -s -m5 …`.
- vm2·vm3의 게이트웨이 ping은 첫 루프(`ping … | tail -1`)에서 실패 시 빈 줄만 남아 판독이 불명확했다. 이후 vm1에서 단계마다 이름표를 붙여 다시 측정했다.

> 정리: **들어오는 연결(ping·SSH)과 그 응답은 전부 성공, VM이 시작하는 연결은 대상이 라우터든 옆 VM이든 외부든 전부 실패.** 세 VM, 두 노드 모두 같다 → 증상 **B**.

## 3. 가설 · 검증 기록 ★

처음엔 이 작업을 **5회차(VM 밖으로 나가는 길, 라우터·NAT)** 과제로 시작했다. 그래서 가설 ①~④가 라우터 쪽이다. 그 가설들이 기각되면서 범위가 VM 포트로 좁혀졌고, SG 변경 시각을 보고 이게 6회차 장애라는 걸 알았다.

| # | 가설 | 확인 방법 (명령) | 결과 | 판단 |
| --- | --- | --- | --- | --- |
| ① | 라우터 외부 게이트웨이·qg·FIP 구성이 깨졌다 | `openstack router show cc01-router` · `openstack floating ip list` · `ip netns exec qrouter-… ip -br addr` · `ip route` | 라우터 UP/ACTIVE, `enable_snat true`, qg에 `203.0.113.100` + FIP 2개, qr `10.101.0.1/24`, `default via 203.0.113.1 dev qg-…` | **기각** |
| ② | SNAT/DNAT 규칙이 빠지거나 끼어 있다 | `ip netns exec qrouter-… iptables -t nat -S` · `iptables -S` · `iptables -t mangle -S` · `iptables-save \| grep -i study` | 세 테이블 모두 표준 Neutron 규칙만 있음 (DNAT·float-snat·snat·jump 연결 정상, 끼워 넣은 규칙 없음) | **기각** |
| ③ | 라우터 바깥(br-ex·veth·외부망)이 끊겼다 | `ip netns exec qrouter-… ping -c2 203.0.113.1` · `ping -c2 8.8.8.8` | 라우터 자신은 둘 다 **성공** | **기각** → 라우터에서 밖으로는 나감 |
| ④ | VM 요청이 라우터까지 와서 라우터 안에서 사라진다 | qr에서 `tcpdump -nei qr-fff49bcc-fa 'host 10.101.0.15 and (icmp or arp)'` 하면서 vm1 안에서 `ping 10.101.0.1` | **0 packets.** vm1 ARP 테이블엔 라우터 MAC이 이미 있음(`DELAY`) | **기각** → 요청이 qr에 도착조차 안 함. **라우터·NAT 가설 전부 기각** (아래 "5회차 문제와 구별") |
| ⑤ | 라우터와 무관하게 VM 쪽에서 막힌다 | vm1 → vm3 (같은 cmp01, br-int 안에서 끝나는 길) · vm1 → 메타데이터 (TCP) | 둘 다 실패 | **유지** → VM 포트 바로 근처, 프로토콜 무관 |
| ⑥ | SG의 egress가 막혀 있다 (상태 기반이라 들어온 연결의 응답만 통과) | `openstack security group rule list cc01-sg` · `openstack port show <포트> -c security_group_ids -c port_security_enabled` (vm1~3) | 규칙 **ingress 2 / egress 0**. 11/15(4회차) 같은 명령 결과엔 있던 `ed43372d-…` (egress IPv4) · `b8ac5553-…` (egress IPv6) 이 **없음**. 세 포트 모두 같은 SG, port security True | **유지 → 노드에서 확인** |
| ⑦ | 노드의 vm1 egress 체인에서 NEW 패킷이 fallback DROP에 걸린다 | cmp01: `iptables -S \| grep 3a5bc23b-2` · `iptables -L neutron-openvswi-o3a5bc23b-2 -n -v --line-numbers` (여러 번) · `iptables -S neutron-openvswi-sg-fallback` | o 체인에 egress 허용 줄 없음. 7번(`→ sg-fallback`) **35177 → 35201 → 35236 → 35306** 계속 증가. fallback은 `DROP` 한 줄(105K pkts) | **원인 확정** |
| ⑧ | 장치 단위로 패킷이 사라지는 곳 | vm1에서 `ping 10.101.0.1`을 켠 채 cmp01에서 `tcpdump -lnei tap3a5bc23b-2d …` → `qvb3a5bc23b-2d` | **tap: echo request seq 12~15 보임** (14:42:59~14:43:02) · **qvb: 0 packets** (6초) | **확정** → 마지막으로 보인 장치 tap, 처음 안 보인 장치 qvb. 그 사이 = qbr의 iptables |

- qvo 캡처와 두 번째 카운터 조회는 명령을 한꺼번에 붙여넣는 바람에 실행되지 않았다. qvb에 없는 패킷이 그 뒤 qvo에 있을 수는 없으므로 qvo도 없음으로 판단했다(직접 측정은 아님).
- vm2(cmp02)의 체인은 직접 조회하지 않았다. 같은 SG라 같은 구조일 것으로 추정한다.

### 패킷의 운명이 정해진 줄 (cmp01, vm1 egress 체인 `neutron-openvswi-o3a5bc23b-2`)

vm1 → 10.101.0.1 ICMP echo request (NEW) 기준, 책 p.249–251 순서대로:

| 순서 | 줄 | 이 패킷은? |
| --- | --- | --- |
| 1 | `FORWARD -j neutron-openvswi-FORWARD` | qbr을 지나는 프레임이라 진입 |
| 2 | `neutron-openvswi-FORWARD … --physdev-in tap3a5bc23b-2d -j neutron-openvswi-sg-chain` | tap에서 들어왔으니(VM이 보냄) 진입 |
| 3 | `sg-chain … --physdev-in tap3a5bc23b-2d -j neutron-openvswi-o3a5bc23b-2` | egress 체인으로 |
| o-1 | `-s 0.0.0.0/32 -d 255.255.255.255/32 udp 68→67 RETURN` | DHCP 아님 → 통과 |
| o-2 | `-j neutron-openvswi-s3a5bc23b-2` | 출발지 `10.101.0.15` + MAC `fa:16:3e:c9:a6:77` 일치 → RETURN (스푸핑 아님) |
| o-3, o-4 | DHCP 허용 / DHCP 위장 DROP | 해당 없음 |
| o-5 | `state RELATED,ESTABLISHED RETURN` (1301K pkts) | **NEW라 해당 없음** ← 들어온 연결의 응답은 여기서 통과 |
| o-6 | `state INVALID DROP` (0 pkts) | 해당 없음 |
| (빠진 줄) | 정상이면 여기 `-j RETURN` (egress IPv4 any에서 나온 줄) | **없음** |
| **o-7** | `-j neutron-openvswi-sg-fallback` (35177→35306) | **여기서 운명 결정** |
| fb-1 | `sg-fallback -j DROP` "Default drop rule for unmatched traffic." | **DROP** |

### API 규칙 ↔ 노드 규칙 짝짓기 (⑧ 결론 표)

| API (`openstack security group rule list cc01-sg`) | 노드 (cmp01 vm1 체인) | 짝 |
| --- | --- | --- |
| ingress tcp 22 `279cbc1f-…` | i 체인 `-p tcp --dport 22 -j RETURN` | ✅ 일치 |
| ingress icmp `8745fca2-…` | i 체인 `-p icmp -j RETURN` | ✅ 일치 |
| (없음) — 기본값이면 egress IPv4 any `ed43372d-…` | o 체인 `-j RETURN` 줄 **없음** | ❌ **빠진 줄** (API·노드 모두 없음) |
| (없음) — 기본값이면 egress IPv6 any `b8ac5553-…` | ip6tables 쪽 대응 줄 없음 (미조회, 추정) | ❌ 빠진 줄 |
| (API에 대응 규칙 없음) | DHCP 허용·위장 차단, s 체인 스푸핑 검사, RELATED/ESTABLISHED, INVALID, fallback DROP | Neutron이 포트마다 자동으로 넣는 줄. 정상 구성의 DROP |

→ **노드 iptables는 API를 정확히 반영하고 있다.** 에이전트나 노드 문제가 아니다. 어긋난 곳은 **API의 SG 자체**다. 새 SG의 기본 egress 두 줄(4회차 기준선엔 존재)이 빠졌다. `cc01-sg`의 마지막 변경 시각은 2026-11-24 12:58 KST(revision 5)이고, 5회차 세션(11/23 22:00) 다음 날이다.

### 5회차(라우터·NAT) 문제와 구별한 관찰

| 관찰 | 라우터·NAT 장애라면 | 실제 |
| --- | --- | --- |
| 라우터 네임스페이스 자신의 외부 ping | 실패해야 함 | **성공** (203.0.113.1, 8.8.8.8) |
| vm1 ping 중 qr 캡처 | 요청은 보여야 함 (라우터에서 사라지므로) | **0건** |
| 라우터를 안 거치는 vm1 → vm3, vm1 → vm2 | 성공해야 함 | **실패** |
| 라우터 nat/filter/mangle | 빠지거나 끼어 있는 규칙 | 표준 규칙만 |
| tap vs qvb | — | tap ✅ / qvb ❌ → 라우터보다 훨씬 앞, VM 바로 옆 qbr에서 끊김 |

## 4. 원인 · 복구 방안

- **원인 (한 문장):** `cc01-sg`에서 기본 egress 허용 규칙(IPv4·IPv6 any)이 삭제되어, VM이 시작하는 모든 NEW 패킷이 compute 노드의 egress 체인 끝 `sg-fallback` DROP에 걸린다.
- **마지막으로 보인 장치 / 처음 안 보인 장치:** `tap3a5bc23b-2d` (echo request 보임) / `qvb3a5bc23b-2d` (0건). 그 사이 `qbr3a5bc23b-2d`에서 iptables `neutron-openvswi-o3a5bc23b-2`의 7번 줄 → `sg-fallback` DROP.
- **되는 것이 왜 되는지:**
  - **ping·SSH로 들어오는 것:** ingress 규칙(icmp, tcp 22)이 i 체인에 그대로 있다.
  - **들어온 연결에 대한 VM의 응답:** o 체인 5번 `RELATED,ESTABLISHED RETURN`이 egress 규칙 **앞에** 있어서, SG에 egress 규칙이 하나도 없어도 응답은 통과한다(1301K pkts). 그래서 SSH로 들어가 명령을 치는 것까지는 된다.
  - **IP는 계속 유지:** DHCP 클라이언트 트래픽은 SG 규칙과 별개로 o 체인 1·3번이 허용한다. lease 갱신이 되니 VM은 IP를 잃지 않는다.
  - **ping으로 확인하면 정상처럼 보이는 이유:** 신고자가 "22번도 ICMP도 열려 있다"고 확인한 건 전부 **ingress** 규칙이다. 빠진 건 평소 아무도 손대지 않는 **기본 egress** 규칙이었다.
- **복구하려면 (운영진 실행, 문서로만 제안):** 빠진 기본 규칙 두 줄만 다시 만든다. ingress 규칙과 포트의 SG 지정은 건드리지 않는다.
  ```bash
  openstack security group rule create --egress --ethertype IPv4 cc01-sg
  openstack security group rule create --egress --ethertype IPv6 cc01-sg
  ```
  - 다른 SG를 새로 만들어 포트에 붙이거나 VM을 재부팅하는 방법은 쓰지 않는다. 원인이 SG 규칙 두 줄뿐이고, 에이전트가 SG 변경을 받아 각 노드 o 체인에 `-j RETURN` 줄을 다시 깔아 준다.
- **복구 후 무엇으로 확인하면 되는지:**
  1. `openstack security group rule list cc01-sg`에 egress 2줄 (Direction 열 ingress 2 / egress 2)
  2. cmp01 `iptables -S neutron-openvswi-o3a5bc23b-2`에서 INVALID DROP 다음, fallback 앞에 `-j RETURN` 줄
  3. vm1 안 ping 중 7번(`→ sg-fallback`) 카운터가 멈추고, qvb·qvo·qr에서 echo request/reply가 보임
  4. vm1~3 안에서 게이트웨이, 옆 VM, 8.8.8.8, curl 모두 성공

## 5. SRE 세 질문

1. **사용자가 무엇을 겪었나:** VM에 SSH로 들어가는 건 되는데, 그 안에서 패키지 설치·외부 호출·옆 VM 접속 같은 **VM이 시작하는 모든 통신**이 timeout. 세 VM 모두.
2. **어떤 증상으로 알아챘나:** 신고와 직접 테스트. 라우터, NAT, FIP, Horizon은 전부 정상이었다. VM 안에서 나가는 ping을 단계별로 쳐 보고서야 드러났다.
3. **운영자가 미리 볼 수 있었던 지표는:**
   - **기존 프로브:** Grafana `slot-probe`는 이번에 확인하지 못했다. 다만 기존 FIP 프로브는 net01 → FIP로 **들어가는** ping·SSH라 ingress 규칙과 ESTABLISHED 응답만으로 성공한다. 오늘 net01에서 같은 경로로 친 FIP ping·SSH도 전부 성공했으니, 그 프로브는 **반응하지 않았을 것**으로 본다(추정).
   - **SLI (한 문장):** 각 슬롯 네트워크의 카나리 VM이 **VM 안에서 시작한** 연결로 1분마다 ① 게이트웨이 ping ② 8.8.8.8 TCP 53 connect를 시도해, 둘 다 3초 안에 성공한 측정의 비율. 1이면 정상이다.
   - **SLO:** **7일 기준 99.5%** (약 50분/주의 실패 허용). 1분 간격 측정에서 프로브 자체의 흔들림과 계획된 작업 몇십 분은 버짓으로 흡수하되, 이번처럼 SG가 바뀌어 **연속 실패**가 나면 한 시간 안에 버짓을 크게 갉아먹어 알람이 울린다. 100%로 잡으면 순간적인 패킷 손실에도 알람이 울려 아무도 보지 않게 된다.
   - **White-box 보조 신호:** 노드별 `neutron-openvswi-sg-fallback` DROP 카운터 증가율, 그리고 SG 규칙 중 **egress 0개인 SG** 탐지(API 조회로 가능).
