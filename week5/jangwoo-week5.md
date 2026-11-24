# 5회차 제출 — 정장우

> 계정: `test` · 작성일: 2026-11-22

## 1. 읽기 핵심 (한 장)

- **기초 다지기**
    - **네임스페이스**: 리눅스가 인터페이스·라우팅 테이블·iptables·ARP 캐시를 통째로 나눠 주는 공간. Neutron 라우터 하나 = net01의 `qrouter-<라우터ID>` 하나. 안을 보려면 `ip netns exec`가 필요함
    - **qr / qg**: 라우터의 내부망 쪽 / 외부 게이트웨이 쪽 인터페이스. 접두사 q는 Neutron의 옛 이름 Quantum, 뒤 11자는 포트 UUID 앞부분(인터페이스 이름 15자 한도)
    - **라우팅 테이블·기본 경로**: 목적지별 출구표. 가장 구체적으로 맞는 줄을 고르고, 나머지는 `default`를 따라감
    - **ARP·이웃 캐시**: 같은 링크에서 "이 IP의 MAC은?"을 묻고, 그 주소를 가진 쪽만 답함. 답은 **물어본 쪽**의 이웃 캐시에 저장되고(`ip neigh`), 상태는 REACHABLE → STALE → PROBE → REACHABLE 또는 FAILED. GARP는 "이 주소는 이 MAC"이라고 먼저 알리는 ARP
    - **NAT 순서**: 들어오자마자 PREROUTING에서 목적지를 바꾸고(DNAT), 라우팅 결정 뒤 나가기 직전 POSTROUTING에서 출발지를 바꿈(SNAT)
    - **conntrack**: nat 규칙은 연결의 첫 패킷에만 적용되고, 결과가 conntrack에 기록됨. 응답은 규칙이 아니라 이 기록을 보고 되돌려짐
- **NAT 두 가지 (책 PDF p.171–172)**
    - FIP가 없는 VM: 라우터 게이트웨이 주소(우리 `203.0.113.234`) 하나를 포트로 나눠 쓰는 many-to-one(PAT). 안에서 시작한 연결만 됨
    - FIP: 외부 주소 하나 ↔ VM 주소 하나의 static NAT. 밖에서 먼저 들어올 수 있음. **라우터의 외부 인터페이스(qg)에 설정됨**
- **FIP가 동작하는 조건 (책 PDF p.188–190)**
    - FIP는 qg의 **/32 보조 주소**로 붙음 → 그래서 라우터가 그 FIP에 대한 **ARP 요청에 응답할 수 있음**
    - 같은 FIP에 대해 DNAT(들어올 때)과 SNAT(나갈 때) 규칙이 추가됨
    - 즉 FIP 하나는 ① qg의 주소 ② iptables NAT 규칙 ③ Neutron DB 세 곳에 흔적을 남김
- **공식 문서 (OVS Self-service North-south)**
    - 시나리오 1(fixed IP): 라우터에서 SNAT 후 gateway 인터페이스로 나감
    - 시나리오 2(FIP): gateway 인터페이스로 들어와 DNAT 후 self-service 인터페이스로. 문서 그림에는 FIP의 ARP 단계가 없음
- **SRE 12장**
    - 좋은 실험은 가설을 둘로 나눔. "일부가 안 된다"는 신고는 **어느 일부인지 표로 만드는 것**이 첫 실험
    - 기각한 가설도 결과다(Negative Results Are Magic). 남겨야 같은 곳을 두 번 안 봄

vm2 FIP로 들어오는 경로 (+ 장애 지점)

```
밖(net01 호스트 veth-gw 203.0.113.1)
  ─ ARP: "203.0.113.220 주인 누구?" → qg-3771c5f1-a9 가 답해야 함   ← 여기서 막힘
  → qg → PREROUTING DNAT(.220 → 10.99.0.248) → qr-d2af90a5-8d
  → br-int → br-tun ═VXLAN═ cmp02 → qvo/qvb → qbr → tap → vm2
```

## 2. 받은 증상

**내가 받은 신고: C**

신고 판별 근거: 같은 네트워크 안 통신은 모두 되고, **FIP가 있는 vm2 하나만** 밖에서 들어오는 것과 밖으로 나가는 것이 둘 다 실패. vm1(FIP 있음)·vm3(FIP 없음)는 정상 → FIP 하나에 국한된 장애(C)

| 확인한 것 | 명령 | 결과 |
| --- | --- | --- |
| VM 구성 | `openstack server list` · `openstack floating ip list` | vm1 10.99.0.117 / FIP 203.0.113.245, **vm2 10.99.0.248 / FIP 203.0.113.220 (cmp02)**, vm3 10.99.0.143 / FIP 없음 |
| ① 내부 | qrouter 경유로 VM 접속 후 `ping` vm1↔vm2, vm3→vm1 | 모두 성공 |
| ② vm1 외부 | vm1에서 `ping 203.0.113.1` · `ping 8.8.8.8` · `curl http://example.com` | 모두 성공 |
| ② **vm2 외부** | vm2에서 같은 3단계 | **게이트웨이 203.0.113.1까지 모두 실패** |
| ② vm3 외부 | vm3에서 같은 3단계 | 모두 성공 |
| ③ vm1 FIP | net01 `ping 203.0.113.245`, ops01 SSH(net01 경유) | 둘 다 성공 |
| ③ **vm2 FIP** | net01 `ping 203.0.113.220`, ops01 SSH(net01 경유) | **둘 다 실패** |
| 헤어핀 | vm1에서 `ping 203.0.113.220` (vm2의 FIP) | **성공** |
| 90초 뒤 재확인 | 위와 같음 | 결과 동일 (일시적 아님) |

## 3. 가설 · 검증 기록 ★

**가장 중요한 부분입니다.** 순서대로, 틀린 가설도 지우지 말고 남기세요.

| # | 가설 (어디가 문제일 것 같다) | 확인 방법 (명령) | 결과 | 판단 |
| --- | --- | --- | --- | --- |
| 1 | 외부 경로 전체(br-ex·외부망)가 끊겼다 | ops01에서 `ping` vm1·vm2 FIP, net01에서 `ping 203.0.113.245` | ops01 ping은 vm1·vm2 **둘 다 실패**. 그런데 장애 전 기준선에서도 ops01 ping은 실패였음(ops01은 FIP 대역으로 라우팅이 없음). net01에서 vm1 FIP는 성공, vm1·vm3 외부도 성공 | 기각 (ops01 ping은 원래 안 되는 시험이었음) |
| 2 | vm2 자체나 cmp02 경로가 문제다 | qrouter 경유 `ssh cirros@10.99.0.248`, vm1↔vm2 `ping` | 둘 다 성공. vm2와 VXLAN 구간은 정상 | 기각 |
| 3 | SG가 vm2를 막는다 | vm1과 SG 비교, 헤어핀 `ping 203.0.113.220` | vm1은 같은 `test-sg`로 FIP·외부 모두 정상. vm1→vm2 FIP 헤어핀이 성공 → 패킷이 vm2 포트까지 들어감. vm2의 egress(게이트웨이 ping)도 실패하는데 SG egress는 전체 허용 | 기각 |
| 4 | FIP 연결이 해제됐다 (Neutron 설정) | `openstack floating ip list --port <vm2 포트>` | `203.0.113.220`이 vm2 포트에 연결된 상태 그대로 | 기각 (DB는 정상) |
| 5 | 라우터의 qg 링크나 기본 경로가 문제다 | `sudo ip netns exec qrouter-3b32d4d1-… ip -br addr` · `ip route` | qg `UNKNOWN`(올라와 있음), `default via 203.0.113.1 dev qg-3771c5f1-a9` 있음. 그런데 **qg 주소가 `203.0.113.234/24 203.0.113.245/32`뿐, vm2 FIP `.220/32`가 없음** | 링크·경로는 기각. 새 단서: qg에 vm2 FIP가 없다 |
| 6 | vm2 FIP의 NAT 규칙(DNAT·float-snat)이 지워졌다 | 헤어핀 결과, `sudo ip netns exec qrouter-… tcpdump -nei qg-3771c5f1-a9 'arp or (icmp and host 8.8.8.8)'` (vm2→8.8.8.8 ping 중), SNAT 줄 수 | 헤어핀 성공 → DNAT 동작. qg에서 `203.0.113.220 > 8.8.8.8: ICMP echo request` → float-snat 동작. 기본 SNAT 줄 2개 그대로 | 기각 (NAT 규칙은 멀쩡함) |
| 7 | 바깥이 `.220`의 MAC을 못 알아낸다 (ARP 응답 없음) | net01 `ip neigh show 203.0.113.220`, `sudo ip netns exec qrouter-… tcpdump -nei qg-3771c5f1-a9 -c 15 'arp or (icmp and host 203.0.113.220)'` (net01에서 ping 중) | 이웃 캐시 **`203.0.113.220 dev veth-gw FAILED`**. 캡처에 **`Request who-has 203.0.113.220 tell 203.0.113.1` 6번, Reply 0번**, ICMP는 qg에 한 번도 오지 않음. vm2→8.8.8.8 캡처에서도 요청이 나갈 때마다 곧바로 `who-has 203.0.113.220`이 뒤따름 | 유지 |
| 8 | 일시적인 현상이다 | 90초 뒤 이웃 캐시·증상 재확인 | 여전히 `FAILED`, 증상 동일 | 기각 → 지속되는 장애 |

## 4. 원인 · 복구 방안

복구는 **실행하지 않습니다.** 어떻게 고치면 될지만 적습니다.

- **원인 (한 문장):** net01 `qrouter-3b32d4d1-…`의 `qg-3771c5f1-a9`에서 vm2 FIP `203.0.113.220/32` 주소가 빠져서, 라우터가 그 FIP의 ARP 요청에 응답하지 못하고 바깥(net01 호스트)이 패킷을 qg로 보내지 못한다.
- **끊긴 지점이 이 증상으로 보인 이유:**
    - **들어오는 방향:** 마지막 정상 지점은 net01 호스트의 veth-gw. `.220`으로 보내려면 MAC이 필요한데, qg에 그 주소가 없으니 아무도 ARP에 답하지 않음(이웃 캐시 FAILED). 패킷은 qg에 한 번도 도착하지 않음
    - **나가는 방향:** vm2의 요청은 float-snat으로 출발지가 `.220`으로 바뀌어 qg를 정상으로 나감(캡처로 확인). 응답을 돌려주려면 바깥이 다시 `.220`의 MAC을 알아야 해서 같은 곳에서 멈춤. 게이트웨이 ping까지 실패한 이유
    - **헤어핀은 성공:** vm1→vm2 FIP는 라우터 안에서 DNAT만 거치고 qg 밖으로 나가지 않아 ARP가 필요 없음. NAT 규칙이 멀쩡하다는 증거
    - **설정 화면은 정상:** FIP 흔적 세 곳(qg 주소 / NAT 규칙 / Neutron DB) 중 qg 주소 하나만 빠짐. 그래서 Horizon과 `openstack floating ip list`로는 보이지 않음
    - **다른 VM은 정상:** vm1 FIP(`.245`)와 기본 주소(`.234`)는 qg에 그대로 있음. 세 주소 모두 qg의 같은 MAC(`fa:16:3e:43:b3:51`)으로 배달되는데, `.220`만 응답할 주소가 없음
- **책의 설명과 우리 환경의 차이:**
    - 책 p.170의 "FIP가 qg의 보조 주소라서 ARP에 응답할 수 있다"를 거꾸로 확인한 장애. 책은 외부 장비(Cisco ASA)에서 관찰하고, 우리는 net01 호스트의 `veth-gw`(203.0.113.1)와 `ip neigh`로 관찰함
    - 책은 qr/qg를 veth 쌍으로 설명하지만, 우리 `ip -br addr`에는 `@ifN`이 없고 상태가 `UNKNOWN` → OVS 내부 포트를 네임스페이스로 옮긴 형태로 보임(추정, `ovs-vsctl show`의 `type: internal`로 확정 필요)
    - 책은 neutron CLI, 우리는 openstack CLI. 외부망도 물리 VLAN 50이 아니라 net01 안의 실습망 `203.0.113.0/24`
    - 공식 문서 그림(시나리오 2)에는 ARP 단계가 없어서, 그림만 보면 NAT만 맞으면 될 것처럼 보임
- **복구하려면 (명령 또는 절차):**

    net01에서 빠진 주소 하나만 다시 붙이고, 바깥의 이웃 캐시를 갱신한다.
    1. `sudo ip netns exec qrouter-3b32d4d1-4d09-4f55-a852-76fb6b12bf05 ip addr add 203.0.113.220/32 dev qg-3771c5f1-a9`
    2. GARP로 바깥에 알림: `arping -U -I qg-3771c5f1-a9 -c 3 203.0.113.220` (qrouter 네임스페이스 안에서). net01 호스트에는 arping이 없으면 L3 에이전트 컨테이너 안에서 실행
    3. GARP를 보낼 수 없으면, 다음 패킷 때 ARP를 다시 묻게 되면서 이제 qg가 응답함

    L3 에이전트 재시작이나 FIP를 뗐다 붙이는 방법은 쓰지 않는다. 재시작은 net01의 모든 라우터를 다시 처리해서 다른 슬롯에도 영향이 가고, FIP 재연결은 DB를 바꾸는 방법이라 원인(qg 주소 누락)이 기록에서 사라진다.

- **복구 후 무엇으로 확인하면 되는지:**
    - qrouter `ip -br addr`의 qg에 `203.0.113.220/32`가 다시 보임
    - net01 `ip neigh show 203.0.113.220` → `lladdr fa:16:3e:43:b3:51 REACHABLE`
    - net01 `ping 203.0.113.220` 성공, ops01에서 vm2 FIP SSH 성공
    - vm2 → `203.0.113.1` · `8.8.8.8` · `http://example.com` 성공

## 5. SRE 세 질문

1. **사용자가 무엇을 겪었나:**
    - vm2에 FIP로 접속이 안 되고, vm2 안에서는 밖으로도 안 나감. 같은 프로젝트의 vm1·vm3와 내부 통신은 정상이고, Horizon에도 이상이 없음
2. **어떤 증상으로 알아챘나:** VM별·방향별 증상 표에서 vm2 줄만 실패(②③). 결정적 신호는 qg 주소 목록에 `.220`이 없다는 것과, qg 캡처에 응답 없는 `who-has 203.0.113.220`만 반복된 것
3. **운영자가 미리 볼 수 있었던 지표는:**
    - net01에서 FIP로 보내는 블랙박스 프로브(FIP ping·SSH)는 이 장애를 잡을 수 있다. 다만 **한계**가 있다. 프로브의 TCP가 계속 오가면 net01 이웃 캐시가 REACHABLE로 계속 갱신돼서, qg 주소가 빠진 뒤에도 한동안 패킷이 전달될 수 있다. 상태는 깨졌는데 프로브는 초록인 구간이 생긴다
    - **Black-box 제안:** FIP inbound 프로브에 더해 **FIP가 있는 VM → 외부** outbound 프로브. FIP 없는 VM의 outbound 프로브도 따로 둬야 기본 SNAT 경로까지 덮을 수 있음
    - **White-box 제안:** 라우터마다 **Neutron FIP 목록 ↔ qrouter qg의 실제 /32 주소**를 주기적으로 비교해서 하나라도 다르면 경보. net01에서 FIP 대역 이웃 항목이 `FAILED`인 개수도 지표로 둠

## 6. 한 줄 회고

- NAT 규칙도, Neutron 설정도 멀쩡했는데 FIP는 죽어 있었다. "규칙이 있다"와 "패킷이 거기까지 온다"는 다른 질문이고, 그 사이에 ARP라는 한 단계가 있다는 게 이번 주 배운 점이다.