# 4회차 제출 — hyeongjun

> 계정: `cc01` · 작성일: 2026-11-15
> 받은 신고: **A — 새로 만든 vm4가 네트워크에 안 붙는다** (DHCP)

## 0. 내 환경 메모 (정상 기준선)

| 항목 | 값 |
| --- | --- |
| 네트워크 / ID | `cc01-net` / `c1043167-7748-40fd-b0ce-193c8f5799c7` · mtu **1450** |
| 서브넷 | `cc01-subnet` 10.101.0.0/24 · `enable_dhcp True` · GW 10.101.0.1 · pool .2–.254 |
| DHCP 포트 | `6b4156b7-7a` · 10.101.0.2 · MAC `fa:16:3e:36:d3:f7` → net01 `qdhcp-c1043167-…` 안 `tap6b4156b7-7a` |
| 라우터 포트 | `fff49bcc-fa` · 10.101.0.1 · MAC `fa:16:3e:bb:10:e9` |
| vm1 | 10.101.0.15 · MAC `fa:16:3e:c9:a6:77` · FIP 203.0.113.214 |
| vm2 | 10.101.0.7 · MAC `fa:16:3e:fe:23:c8` · FIP 203.0.113.178 |
| vm3 | 10.101.0.180 · MAC `fa:16:3e:c4:a7:88` |
| **vm4 (신고 대상)** | 10.101.0.187 (API상) · 포트 `5b3f90d2-d8` · MAC **`fa:16:3e:f8:2a:a2`** · key `cc01-key` |

## 1. 읽기 핵심 (한 장)

### Learning OpenStack Networking 5장 — 포트·서브넷·DHCP·메타데이터

```
vm4 eth0 ── DHCPDISCOVER (0.0.0.0:68 → 255.255.255.255:67, 브로드캐스트)
   → tap/qbr(SG) → br-int → br-tun ══VXLAN══▶ net01 br-tun → br-int
   → qdhcp-<네트워크ID> 네임스페이스의 tap6b4156b7-7a
   → (netfilter INPUT) → dnsmasq :67 ── DHCPOFFER ──▶ 같은 길로 vm4에게
```

- **포트의 IP는 누가 언제 정하나:** 포트를 만드는 순간 Neutron(IPAM)이 서브넷 풀에서 골라 DB에 적는다. 그 값을 dhcp-agent가 dnsmasq의 `host` 파일(`MAC,이름,IP`)로 옮기고, 게스트는 **부팅 때 DHCP로 받아야** 실제로 그 IP를 쓴다. 즉 `openstack server list`의 IP는 "배정"이지 "수령"이 아니다. 이번 장애의 핵심 구분이었다.
- **dnsmasq는 아는 MAC에게만 답한다.** 그래서 처음엔 "host 파일에 vm4가 빠졌다"를 의심했지만, 파일엔 있었다. 그 사이(tap과 dnsmasq 소켓 사이)에 **netfilter INPUT 체인**이 있다는 게 다음으로 볼 지점이었다.
- **메타데이터 두 경로 (책 p.141·p.160):** 책은 라우터가 있으면 qrouter 쪽 프록시가, 라우터 없는 격리 서브넷이면(`enable_isolated_metadata`) DHCP 네임스페이스가 `169.254.169.254`를 처리한다고 설명한다. 우리 환경은 `force_metadata = true`라 **라우터가 있어도 DHCP 네임스페이스가 처리**한다. 실제로 qdhcp의 tap에 `169.254.169.254/32`가 붙어 있었다. → 이번 장애처럼 qdhcp 네임스페이스가 막히면 DHCP뿐 아니라 메타데이터(키 주입)까지 같은 곳에 영향을 받는다.
- **MTU 1450 (공식 문서):** VXLAN 오버헤드 50바이트(바깥 Ethernet 14 + IP 20 + UDP 8 + VXLAN 8)를 1500에서 빼서 1450. VM에는 **DHCP 옵션 26**으로 전달된다. → vm4처럼 DHCP를 못 받으면 IP뿐 아니라 MTU·라우트·DNS도 못 받는다.

### SRE 12장 (Negative Results Are Magic) · 6장 (Black-box vs White-box)

- 기각한 가설(SG, 포트 DOWN, 서브넷 DHCP off, 요청 미도착, dnsmasq 미동작, host 파일 누락)을 하나씩 지운 기록이 있어야 "남은 곳은 tap과 dnsmasq 사이"라는 결론이 근거를 갖는다. 각각이 다음 사람의 조사 시간을 줄이는 **재사용 가능한 부정적 결과**다.
- Black-box 프로브는 **이미 있는 대상(vm1~3 FIP)** 만 두드린다. 이번 장애는 "새 MAC의 첫 DHCP"에서만 드러나므로 기존 프로브가 지나가는 길 위에 장애가 없다. 그래서 Grafana가 전부 초록이다.

## 2. 받은 증상

| 확인한 것 | 명령 | 결과 |
| --- | --- | --- |
| VM 상태 | `openstack server list` | vm1~4 모두 **ACTIVE**, vm4 = `cc01-net=10.101.0.187` (FIP 없음) |
| vm4 포트 | `openstack port show 5b3f90d2-…` | `admin_state_up UP` · `status ACTIVE` · `10.101.0.187` |
| **vm4 게스트가 실제로 받은 것** | `openstack console log show cc01-vm4` | `soliciting a DHCP lease` → 5초 뒤 `probing for an IPv4LL address` → **`using IPv4LL address 169.254.160.45`**. `leased …` 줄 **없음** |
| vm4 링크 | 같은 콘솔 로그 | `eth0: carrier acquired` — 링크(포트 연결)는 정상 |
| 기존 VM (비교) | vm1~3 FIP SSH / 내부 ping (3회차 복구 이후 기준선) | 정상 |
| 내 라우터 찾기 | net01: qrouter 네임스페이스 중 `qr-fff49bcc-fa`를 가진 것 | `qrouter-5ec32c60-6a9b-47cc-bb1d-9266a120bf27` (`qr-fff49bcc-fa 10.101.0.1/24`) |
| 라우터 → vm1 (비교) | `ip netns exec qrouter-5ec32c60-… ping -c2 -W1 10.101.0.15` | **2/2 성공** (1.2~4.2ms) |
| **라우터 → vm4** | `ip netns exec qrouter-5ec32c60-… ping -c2 -W1 10.101.0.187` | **2/2 손실 (100%)** — 신고 재현 |
| 라우터의 vm4 ARP | `ip netns exec qrouter-5ec32c60-… ip neigh \| grep 10.101.0.187` | `lladdr fa:16:3e:f8:2a:a2 REACHABLE` — **ARP는 풀림** (아래 설명) |

> 정리: OpenStack은 vm4에 10.101.0.187을 **배정**했지만, 게스트는 DHCP 응답을 못 받아 **169.254.x.x 링크로컬 주소를 스스로 만들어** 쓰고 있다. 그래서 10.101.0.187로는 ping·SSH가 닿지 않는다.
>
> **ARP가 REACHABLE인데 ping이 안 되는 이유 (추정):** l2population의 br-tun **ARP 응답기(table 21, 3회차)** 가 Neutron DB의 "10.101.0.187 = fa:16:3e:f8:2a:a2"를 보고 게스트 대신 OVS에서 ARP에 답한다. 그래서 라우터는 vm4가 그 IP를 가졌다고 믿고 ICMP를 보내지만, 실제 게스트는 그 IP가 없어 응답하지 않는다. "ARP까지 되니 L2·포트는 정상"이라는 **두 번째 미끼**. ARP 응답은 DB 기준이고, ping 응답은 게스트 기준이다. (net01 br-tun table 21 flow로 직접 확인하지는 않음)

## 3. 가설 · 검증 기록 ★

경로: **vm4 → cmp tap/qbr(SG) → br-int → br-tun → VXLAN → net01 br-tun → br-int → qdhcp `tap6b4156b7-7a` → netfilter INPUT → dnsmasq :67 → (응답) → vm4**

| # | 가설 | 확인 방법 (명령) | 결과 | 판단 |
| --- | --- | --- | --- | --- |
| ① | 서브넷 DHCP가 꺼져 있다 / 네트워크 설정이 이상하다 | `openstack subnet show cc01-subnet` · `openstack network show cc01-net -c mtu` | `enable_dhcp True`, mtu 1450, 풀 .2–.254 정상 | **기각** |
| ② | SG가 DHCP를 막는다 | `openstack security group rule list cc01-sg` | egress IPv4/IPv6 전체 허용, ingress 22·icmp. vm1~3도 같은 SG로 정상 | **기각** |
| ③ | vm4 포트가 DOWN이거나 연결이 안 됐다 | `openstack port show 5b3f90d2-…` · 콘솔 로그 `carrier` | `UP` / `ACTIVE`, 게스트도 `carrier acquired` | **기각** |
| ④ | vm4가 DHCP를 아예 안 보냈거나 받았는데 못 썼다 | `openstack console log show cc01-vm4` | `soliciting a DHCP lease` 후 응답 없이 **IPv4LL `169.254.160.45`**. `leased` 없음 | **유지** → 요청은 나갔고 응답이 안 왔다. 어디서 끊겼는지 경로를 따라감 |
| ⑤ | (a) 요청이 net01 qdhcp까지 **도착하지 않는다** (cmp → net01 경로 문제) | net01: `ip netns exec qdhcp-… tcpdump -lnei tap6b4156b7-7a -c 10 'ether host fa:16:3e:f8:2a:a2 and (port 67 or port 68)'` + ops01에서 `openstack server reboot cc01-vm4` | **Request 8개 도착** (`0.0.0.0.68 > 255.255.255.255.67, Request from fa:16:3e:f8:2a:a2`), 간격 5→8→15→32→64→64→64초(dhcpcd 재시도 backoff). **Reply는 0개** | **기각** → 요청은 tap까지 옴. 끊긴 곳은 **tap 이후** |
| ⑥ | (d) dnsmasq가 죽었거나 67번을 안 듣는다 | `ip netns exec qdhcp-… ss -lunp` | `dnsmasq pid=1301085`가 `0.0.0.0:67` 및 :53 LISTEN | **기각** |
| ⑦ | (c) dnsmasq가 vm4 MAC을 모른다 (host 파일 누락) | `docker exec neutron_dhcp_agent cat /var/lib/neutron/kolla/dhcp/<네트워크ID>/host` | `fa:16:3e:f8:2a:a2,host-10-101-0-187.openstacklocal,10.101.0.187` **있음** | **기각** → dnsmasq는 vm4에게 줄 IP를 알고 있다 |
| ⑧ | (b) tap과 dnsmasq 사이 **qdhcp 네임스페이스 iptables INPUT**이 막는다 | `ip netns exec qdhcp-… iptables -L INPUT -n -v --line-numbers` (두 번 실행해 카운터 비교) · `grep -i f8:2a:a2 /var/log/kolla/neutron/dnsmasq.log` | 아래 표. vm1~3 MAC만 `ACCEPT`, 나머지 udp/67은 **DROP** (`/* study-w4a-cc01 */`). DROP 카운터 **6423 → 6427 → 6428** 증가 중. dnsmasq.log에 vm4 MAC 기록 **없음** | **원인 확정** |

**⑧ qdhcp-c1043167-… INPUT 체인**

```
num  pkts  bytes  target  match                                          comment
1      10   3280  ACCEPT  udp dpt:67  MAC fa:16:3e:fe:23:c8   (vm2)       study-w4a-cc01
2      11   3608  ACCEPT  udp dpt:67  MAC fa:16:3e:c4:a7:88   (vm3)       study-w4a-cc01
3      11   3608  ACCEPT  udp dpt:67  MAC fa:16:3e:c9:a6:77   (vm1)       study-w4a-cc01
4    6423  2107K  DROP    udp dpt:67  (그 외 전부 → vm4 f8:2a:a2)         study-w4a-cc01   ← 범인
5     374  49088  neutron-dhcp-age-INPUT  (Neutron이 만든 정상 체인)
```

- **패킷 크기로 교차 확인:** DROP 2107K / 6423 ≈ **328바이트** = tcpdump의 프레임 342바이트 − Ethernet 헤더 14바이트. 즉 버려지는 패킷은 tcpdump에서 본 그 DHCP Request와 같은 크기다.
- **tcpdump는 보이는데 dnsmasq는 못 본 이유:** tcpdump는 인터페이스에서 netfilter보다 **먼저** 패킷을 복사한다. iptables INPUT의 DROP은 그 뒤, 소켓에 전달되기 **전에** 일어난다. 그래서 "tap에는 도착(⑤) / dnsmasq.log엔 흔적 없음 / Reply 0개"가 동시에 성립한다.
- comment `study-w4a-cc01`, 첫 번째 규칙 위치 → Neutron이 만든 규칙(5번 체인)이 아니라 **나중에 끼워 넣은 규칙**이다. 3회차의 cookie `0x3303xx` flow와 같은 성격.

## 4. 원인 · 복구 방안

- **원인 (한 문장):** net01 `qdhcp-c1043167-…` 네임스페이스의 iptables INPUT 맨 앞에 **기존 VM 3대의 MAC만 허용하고 나머지 udp/67을 모두 DROP하는 규칙**(`study-w4a-cc01`)이 들어 있어, 새로 생긴 vm4의 DHCP 요청이 dnsmasq에 닿기 전에 버려졌다.
- **요청이 마지막으로 확인된 지점 / 처음 사라진 지점:** 마지막 확인 = qdhcp `tap6b4156b7-7a` (tcpdump Request 8개). 처음 사라진 지점 = 같은 네임스페이스 iptables INPUT 4번 규칙 (DROP 카운터 증가, dnsmasq.log 기록 없음).
- **끊긴 지점이 이 증상으로 보인 이유 (미끼):**
  규칙이 **MAC 화이트리스트** 구조라서 이미 등록된 vm1~3은 lease 갱신 요청이 1~3번 ACCEPT를 타고 정상 통과한다. 카운터가 10~11인 것도 그 흔적이다(Neutron 기본 lease 24시간이면 반나절마다 갱신, 규칙이 들어간 11/10 이후 약 5일 × 하루 2회 ≈ 10회 — **추정**). 그래서 "기존 VM은 다 정상 → 환경은 정상"처럼 보였다. 반면 화이트리스트에 없는 **새 MAC(vm4)** 만 DROP에 걸린다. 게스트는 응답을 못 받아 169.254.x.x를 스스로 만들어 쓰고, OpenStack API는 IPAM이 **배정**한 10.101.0.187과 ACTIVE를 그대로 보여준다. 결국 "ACTIVE고 IP도 적혀 있는데 안 붙는" 증상이 된다. 같은 이유로 DHCP 옵션으로 오는 라우트·MTU도 받지 못한다.
- **복구하려면 (운영진 실행, 문서로만 제안):** `study-w4a-cc01` 주석이 붙은 **4개 규칙만** 삭제하고, 5번 `neutron-dhcp-age-INPUT` 체인과 dnsmasq·에이전트는 건드리지 않는다.
  ```bash
  # net01
  NS=qdhcp-c1043167-7748-40fd-b0ce-193c8f5799c7
  # 1) 지울 규칙의 정확한 스펙을 확인
  ip netns exec $NS iptables -S INPUT | grep study-w4a-cc01
  # 2) 위 출력의 각 줄에서 -A 를 -D 로 바꿔 실행 (번호가 아닌 스펙으로 지워서 다른 규칙 오삭제 방지)
  #    최소 복구는 DROP 한 줄: ip netns exec $NS iptables -D INPUT <… -j DROP 줄의 스펙>
  ```
  - 왜 이 방법인가: 원인은 끼워 넣은 규칙뿐이고 Neutron 정상 구성(5번 체인, host 파일, dnsmasq)은 멀쩡하다. **DROP 한 줄만 지워도** vm4는 5번 체인으로 정상 처리된다. 다만 ACCEPT 3줄은 남겨 둘 이유가 없으니 함께 지워 원상태로 되돌린다.
  - dhcp-agent 재시작이나 네임스페이스 재생성은 하지 않는다. 같은 net01의 다른 참가자 네트워크에까지 영향을 주고, 이 규칙이 그 과정에서 지워질지 남을지도 보장되지 않는다.
- **복구 후 무엇으로 확인하면 되는지:**
  1. `iptables -L INPUT -n -v`에 `study-w4a-cc01`이 없고 5번 체인만 남았는지
  2. 같은 tcpdump에서 `Request from fa:16:3e:f8:2a:a2` 다음에 `10.101.0.2.67 > … Reply`가 찍히는지 (dhcpcd가 64초 간격으로 재시도 중이라 재부팅 없이도 1분 안에 보여야 함)
  3. `dnsmasq.log`에 `DHCPDISCOVER/DHCPOFFER/DHCPREQUEST/DHCPACK … fa:16:3e:f8:2a:a2` 기록
  4. vm4 콘솔 로그에 `leased 10.101.0.187`. 이후 라우터 네임스페이스에서 `ping 10.101.0.187`과 SSH 성공

## 5. SRE 세 질문

1. **사용자가 무엇을 겪었나:** 새로 만든 VM이 Horizon에선 ACTIVE에 IP까지 표시되는데, 그 IP로 SSH·ping이 전혀 안 된다. 기존 VM은 멀쩡해서 "내가 뭘 잘못 만들었나"로 보인다.
2. **어떤 증상으로 알아챘나:** 사용자 신고. API·포트 상태·기존 VM은 전부 정상이었고, 게스트 콘솔 로그의 `IPv4LL 169.254.160.45`(= DHCP 응답 없음)가 첫 직접 증거였다.
3. **운영자가 미리 볼 수 있었던 지표는:**
   - **기존 프로브로는 불가.** Grafana는 전부 초록이었다. 프로브가 두드리는 vm1~3은 화이트리스트에 있어 정상 경로를 타고, "새 MAC의 첫 DHCP"라는 길은 아무 프로브도 지나가지 않는다.
   - **Black-box 제안:** 네트워크마다 주기적으로 **카나리 포트(새 MAC)로 DHCPDISCOVER를 보내 OFFER가 오는지** 확인한다(또는 카나리 VM을 띄워 `leased`까지 걸린 시간 측정). "새 VM이 IP를 받는다"를 사용자 관점에서 직접 검사하는 프로브다.
   - **White-box 제안:** qdhcp 네임스페이스별 iptables **DROP 카운터(udp/67) 증가율** + Neutron이 만들지 않은 규칙(주석·체인 밖 규칙) 존재 알람. 그리고 dnsmasq 로그의 **"DB에는 있는 MAC인데 DHCPDISCOVER가 N분째 0건"** 신호(포트 ACTIVE 이후 DHCPACK이 없는 compute 포트 수).
