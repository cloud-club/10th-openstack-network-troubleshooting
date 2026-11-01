# 2회차 제출 — hyeongjun

> 계정: `cc01` · 작성일: 2026-11-02
> 주제: Linux bridge — VM 바로 옆에서 끊기면

## 0. 내 환경 메모

| 항목                | 값                                                                                                 |
| ------------------- | -------------------------------------------------------------------------------------------------- |
| 프로젝트            | `study-cc01`                                                                                       |
| vm1 (비교용, cmp01) | 내부 IP: `10.101.0.15` · FIP: `203.0.113.214`                                                      |
| vm2 (증상, cmp02)   | 내부 IP: `10.101.0.7` · FIP: `203.0.113.178`                                                       |
| vm2 포트 ID         | `a49896ed-e537-44c9-8fc4-5a1707454cfd` (앞 11자리: `a49896ed-e5`)                                  |
| 장치 이름           | `tapa49896ed-e5` · `qbra49896ed-e5` · `qvba49896ed-e5` · `qvoa49896ed-e5`                          |
| 라우터              | `cc01-router` · 네임스페이스 `qrouter-5ec32c60-6a9b-47cc-bb1d-9266a120bf27` (net01 = 192.168.0.12) |
| 라우터 인터페이스   | `qr-fff49bcc-fa` 10.101.0.1/24 (내부) · `qg-ead8ef8d-f0` 203.0.113.100 + FIP .214/.178 (외부)      |

## 1. 읽기 핵심 (한 장)

### Learning OpenStack Networking 4장 (p.90–102)

컴퓨트 노드 안에서 VM 패킷이 지나는 경로:

```
vm2 eth0 → tapXXX → qbrXXX (Linux bridge, SG iptables 적용) → qvbXXX ⇄ qvoXXX (veth) → br-int (OVS) → br-tun/br-ex → …
```

| 접두어   | 무엇인가                                                                                                                         | 어디에 붙어 있나                             |
| -------- | -------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------- |
| `tap`    | VM의 vNIC에 대응하는 호스트 쪽 TAP 장치 (MAC `fe:16:3e…` = 포트 MAC의 첫 바이트만 다름)                                          | VM ↔ qbr                                     |
| `qbr`    | VM마다 하나씩 만드는 **리눅스 브리지**. 시큐리티 그룹 iptables가 여기서 적용됨 (OVS 포트엔 iptables를 걸 수 없어서 둔 중간 단계) | tap과 qvb를 묶음                             |
| `qvb`    | veth 쌍의 **브리지(qbr) 쪽** 끝 ("q-veth-bridge")                                                                                | qbr의 멤버                                   |
| `qvo`    | veth 쌍의 **OVS 쪽** 끝 ("q-veth-ovs")                                                                                           | br-int 포트, 네트워크별 로컬 VLAN tag가 붙음 |
| `br-int` | 노드의 통합 OVS 브리지. 로컬 VLAN tag ↔ VXLAN VNI 변환은 br-tun과 함께 처리                                                      | qvo들과 br-tun을 연결                        |

- 이번 장애와 연결되는 포인트:
  - 장치 이름 = 접두어 + **포트 ID 앞 11자리**라서, API에서 포트 ID만 확보하면 어느 노드에서든 해당 VM의 장치를 바로 찾을 수 있었다.
  - 경로가 tap → qbr → qvb⇄qvo → br-int로 "직렬"이라 각 장치에서 tcpdump를 걸면 **패킷이 사라지는 첫 지점**이 곧 끊긴 지점이 된다.
  - 장치가 전부 UP이어도 **연결 관계(브리지 멤버십)** 가 끊기면 통신이 안 된다 — `ip link` 상태만 보면 놓친다.

### SRE 12장 (p.159–166) — Effective Troubleshooting

- "가장 그럴듯한 원인부터 고치기"가 위험한 이유:
  - 그럴듯한 원인은 대개 **최근에 겪었거나 익숙한 원인**일 뿐, 증거로 고른 것이 아니다. 틀린 곳을 고치면 시간만 쓰고, 원래 원인 위에 변경이 하나 더 얹혀 상황이 더 복잡해진다.
  - 12장의 흐름은 "증상 관찰 → 가설 → **검증으로 기각/유지** → 범위 좁히기"이고, 고치기는 원인이 증거로 특정된 뒤의 일이다.
- 이번 과제에 적용한 점:
  - "SSH·ping이 안 된다"에서 가장 그럴듯한 건 시큐리티 그룹이었지만, 규칙을 먼저 **조회해서 기각**했다 (②).
  - net01에서 ARP reply가 0.2ms 만에 와서 "L2는 살아 있다"고 결론 내릴 뻔했다. 응답 속도가 vm1 RTT보다 너무 빨라 net01 br-tun의 ARP responder가 대신 답한 것으로 보고 증거에서 제외했고, 실제로 cmp02 qvo에선 ARP reply가 없었다.
  - 경로를 반으로 나누듯(net01 → 터널 → qvo → qvb → qbr → tap) tcpdump로 **패킷이 사라지는 첫 지점**을 찾았고, 정상 VM(cmp01)과 나란히 비교해 확정했다.

## 2. 받은 증상

| 확인한 것          | 명령                                                                                    | 결과                                                                                                                                                                                   |
| ------------------ | --------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| vm2 SSH            | `ssh -i ~/cc01-key.pem -o IdentitiesOnly=yes -J cc01@192.168.0.12 cirros@203.0.113.178` | `channel 0: open failed: connect failed: Connection timed out` — 점프 호스트(net01)까지는 접속됨, 그 뒤 vm2:22 연결이 timeout                                                          |
| vm2 ping           | (net01) `ping -c 3 203.0.113.178`                                                       | **3/3 손실 (100% loss)**                                                                                                                                                               |
| vm1 SSH (비교)     | `ssh … -J cc01@192.168.0.12 cirros@203.0.113.214`                                       | vm1 sshd의 **호스트 키 확인 + 비밀번호 프롬프트까지 도달** → TCP 22 연결 성립 (로그인 실패는 root 셸에서 실행해 `~/cc01-key.pem`이 `/root/…`로 잡혀 키를 못 쓴 것, 네트워크 문제 아님) |
| vm1 ping (비교)    | (net01) `ping -c 3 203.0.113.214`                                                       | **3/3 응답**, ttl=63, 1.1~6.3ms                                                                                                                                                        |
| Horizon / API 상태 | `openstack port list --server cc01-vm2`                                                 | 포트 ACTIVE, 신고 내용상 Horizon에서도 vm2 ACTIVE                                                                                                                                      |
| Grafana FIP 프로브 | `localhost:3001`                                                                        | 대시보드에서 vm2 프로브를 찾기 어려워 확인하지 못함                                                                                                                                                                            |

> 정리: 같은 FIP 대역·같은 라우터인데 vm2만 ICMP·TCP 22 모두 응답 없음. vm2 SSH는 **connect timeout**(SYN에 응답 자체가 없음), vm1은 **인증 단계까지 진행** — 차이가 네트워크 계층에서 갈린다는 첫 단서.

## 3. 가설 · 검증 기록 ★

경로: **ops01(FIP) → net01(qrouter) → cmp02 → tap/qbr/qvb/qvo/br-int → vm2**

| #   | 가설 (어디가 문제일 것 같다)                                                   | 확인 방법 (명령)                                                                                                        | 결과                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | 판단                                                                                                              |
| --- | ------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| ①   | API상 vm2 포트/FIP가 이상하다 (DOWN, 바인딩 실패, FIP 미연결)                  | `openstack port list --server cc01-vm2` · `openstack floating ip list`                                                  | vm2 포트 `a49896ed…` **ACTIVE**, 10.101.0.7 · FIP 203.0.113.178 → 10.101.0.7 정상 연결                                                                                                                                                                                                                                                                                                                                                                                       | 기각 (API상 정상)                                                                                                 |
| ②   | 시큐리티 그룹에 ICMP/22 규칙이 빠졌다 (ping과 SSH의 원인이 같은가?)            | `openstack security group rule list cc01-sg`                                                                            | ingress tcp 22 / icmp 모두 0.0.0.0/0 허용, egress 전체 허용                                                                                                                                                                                                                                                                                                                                                                                                                  | 기각 (규칙상 정상. 게다가 ④-c에서 패킷이 SG iptables가 적용되는 qbr에 **도달조차 못함** → SG는 원인이 될 수 없음) |
| ③   | net01 라우터에서 vm2 내부 IP까지 못 간다                                       | `ip netns exec qrouter-5ec32c60… ip -br a` · `… ping -c 3 10.101.0.15` · `… ping -c 3 10.101.0.7` · `… ip neigh`        | qg에 FIP 두 개 다 있음. **vm1(10.101.0.15) 3/3 응답, vm2(10.101.0.7) 0/3**. ARP `10.101.0.7 lladdr fa:16:3e:fe:23:c8 STALE` (MAC은 vm2 포트와 일치, 단 예전에 배운 캐시). qr에서 tcpdump: `ARP who-has 10.101.0.7` → **0.2ms 만에 reply** (is-at fa:16:3e:fe:23:c8), 이후 `ICMP echo request` seq 1~~5 나감, **echo reply 0건**. ARP 응답이 vm1 ping RTT(2~~8ms)보다 훨씬 빨라 br-tun ARP responder(l2population)가 net01에서 대신 답했을 가능성 → vm2 생존 증거로 쓰지 않음 | 유지 → 같은 라우터·같은 qr로 vm1은 되므로 **net01 라우터 자체는 정상, 문제는 그 너머(cmp02 쪽)**                  |
| 3-1 | net01 → cmp02 VXLAN 터널이 없다                                                | (net01) `docker exec openvswitch_vswitchd ovs-vsctl show \| grep -A2 vxlan`                                             | br-tun에 `vxlan-c0a80015`(→192.168.0.21 cmp01), `vxlan-c0a80016`(→192.168.0.22 cmp02) 둘 다 존재. 포트 이름 = remote IP의 16진수                                                                                                                                                                                                                                                                                                                                             | 기각 (터널 포트는 있음 · 실제 도착 여부는 ④에서)                                                                  |
| ④-a | cmp02의 vm2 장치(tap/qbr/qvb/qvo) 중 없거나 DOWN인 것이 있다                   | (cmp02) `ip -br link \| grep a49896ed-e5`                                                                               | qbr / qvo@qvb / qvb@qvo 모두 **UP**, tap은 UNKNOWN(tap의 정상 표기)·LOWER_UP                                                                                                                                                                                                                                                                                                                                                                                                 | 기각 (장치는 전부 살아 있음)                                                                                      |
| ④-b | qvo가 br-int에서 빠졌거나 tag가 틀렸다                                         | `docker exec openvswitch_vswitchd ovs-vsctl show \| grep -B1 -A3 qvoa49896ed-e5` · `ovs-vsctl list port qvoa49896ed-e5` | br-int에 `qvoa49896ed-e5` 있음, **tag 2**, `network_type=vxlan, segmentation_id=813`                                                                                                                                                                                                                                                                                                                                                                                         | 기각 (tcpdump의 VNI 813과 일치)                                                                                   |
| ④-c | 패킷이 cmp02의 어느 장치까지 오는지 (net01 qrouter에서 `ping 10.101.0.7` 지속) | `tcpdump -lni any udp port 4789` → `-i qvo…` → `-i qvb…` → `-i qbr…` → `-i tap…`                                        | eth0에 **VXLAN vni 813로 echo request 도착** → qvo에 request 보임 → **qvb에 request 보임** → **qbr 0건** → tap 0건. (qvo에서 ARP who-has도 보이지만 reply 없음)                                                                                                                                                                                                                                                                                                              | **qvb → qbr 사이에서 끊김**                                                                                       |
| ④-d | qvb가 qbr 브리지에 붙어 있지 않다                                              | `bridge link \| grep a49896ed-e5`                                                                                       | **`tapa49896ed-e5 … master qbra49896ed-e5`만 출력, qvba49896ed-e5 행이 없음** → qbr에는 tap만 있고 qvb는 브리지 멤버가 아님. `ip link show qvba49896ed-e5`에 **`master` 항목 없음** (비교: `ip link show tapa49896ed-e5`엔 `master qbra49896ed-e5`). cmp01의 vm1은 `bridge link`에 qvb·tap **두 줄 모두** master qbr3a5bc23b-2d                                                                                                                                              | **유지 → 원인 확정**                                                                                              |

### 비교 관찰 (vm1 / cmp01 vs vm2 / cmp02)

| 확인 항목                          | vm1 (정상)                                                                                  | vm2 (증상)                                         | 차이                        |
| ---------------------------------- | ------------------------------------------------------------------------------------------- | -------------------------------------------------- | --------------------------- |
| 장치 상태 (`ip -br link`)          | UP                                                                                          | 4개 모두 UP                                        | 없음                        |
| qbr 멤버 (`bridge link`)           | `qvb3a5bc23b-2d … master qbr3a5bc23b-2d` · `tap3a5bc23b-2d … master qbr3a5bc23b-2d` **2줄** | `tapa49896ed-e5 … master qbra49896ed-e5` **1줄뿐** | **qvb 누락**                |
| qvb의 master (`ip link show qvb…`) | qbr3a5bc23b-2d                                                                              | **없음**                                           | **qvb가 브리지에서 분리됨** |
| qvo가 br-int에 있는가 / tag        | 있음                                                                                        | 있음 / tag 2                                       | 없음                        |
| 라우터→VM ping                     | 3/3 응답                                                                                    | 0/3, 패킷이 qvb에서 멈춤                           | qbr로 못 넘어감             |

## 4. 원인 · 복구 방안

복구는 **실행하지 않습니다.**

- **원인 (한 문장):** cmp02에서 vm2용 veth의 한쪽 끝 `qvba49896ed-e5`가 리눅스 브리지 `qbra49896ed-e5`에서 빠져(brif 해제) 있어, br-int에서 qvo→qvb까지 온 패킷이 브리지를 통해 tap(vm2)으로 넘어가지 못한다.
- **끊긴 지점이 이 증상으로 보인 이유 (⑤):**
  - L2에서 경로 자체가 끊겼기 때문에 프로토콜과 무관하게 **ICMP도 TCP 22도 전부** 막힌다 (시큐리티 그룹처럼 프로토콜별로 갈리는 원인이 아님).
  - 장치는 모두 UP이고 qvo는 br-int에 정상 tag로 붙어 있어 OVS agent·Neutron 입장에선 포트가 **ACTIVE**, Nova도 VM이 실행 중이라 **ACTIVE**로 보인다. qbr 멤버십은 VM plug 시점에 os-vif가 설정할 뿐 이후 계속 감시하는 주체가 없다.
  - net01에서는 br-tun의 ARP responder가 대신 ARP에 답해 "ARP는 되는데 ping만 안 되는" 것처럼 보였다.
- **복구하려면 (명령 또는 절차):** (운영진, cmp02에서)
  - `ip link set qvba49896ed-e5 master qbra49896ed-e5` (또는 `brctl addif qbra49896ed-e5 qvba49896ed-e5`)
  - 대안: `openstack server reboot --hard cc01-vm2` — VIF를 다시 plug하면서 qbr/qvb 연결이 재구성됨 (VM 재시작이 동반되므로 영향이 더 큼)
- **복구 후 무엇으로 확인하면 되는지:**
  - `bridge link | grep a49896ed-e5`에 tap과 qvb **두 줄**이 나오는지
  - qrouter에서 `ping 10.101.0.7` 응답, cmp02 `tcpdump -i tapa49896ed-e5`에 request/reply 둘 다 보이는지
  - ops01에서 FIP `203.0.113.178`로 SSH 접속, Grafana vm2 FIP 프로브 복구

## 5. SRE 세 질문

1. **사용자가 무엇을 겪었나:** 어제까지 되던 vm2에 SSH·ping이 모두 안 됨. 그런데 Horizon에선 ACTIVE라 "내가 뭘 잘못했나?"부터 의심하게 되는 상태 — 서비스는 완전히 중단, 대시보드는 정상.
2. **어떤 증상으로 알아챘나:** FIP로 SSH connect timeout, ping 100% 손실. 같은 프로젝트·같은 라우터의 vm1은 정상이라 **vm2 한 대에 국한된** 장애로 보임.
3. **운영자가 미리 볼 수 있었던 지표는:**
   - Grafana의 vm2 FIP 프로브(블랙박스 ICMP/TCP 22) 실패 — 사용자 신고 전에 알 수 있었던 가장 직접적인 신호. _(대시보드에서 해당 프로브를 제대로 확인하지 못해 실패 시점은 기록하지 못함)_
   - API 상태(ACTIVE)는 이번 장애를 **전혀 반영하지 못함** → 화이트박스 상태값만으로는 부족하고, 사용자 경로를 그대로 따라가는 블랙박스 프로브가 필요하다는 예.
   - 추가로 둘 만한 지표: 컴퓨트 노드에서 "qbr 멤버 수 ≠ 2"인 브리지 탐지, 라우터 네임스페이스에서 내부 IP로의 주기적 ping.

## 6. 한 줄 회고 (선택)

- tcpdump는 정말 중요하다...
