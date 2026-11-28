# 5회차 제출 — hyeongjun

> 계정: `cc01` · 작성일: 2026-11-28 (마감 11/22 이후 늦은 제출)
> 상태: **장애 재현 불가 — 세션(11/23) 복구 이후 점검 기록**

세션에 참석하지 못했고 점검은 11/28에 했다. 그 사이 운영진이 5회차 장애를 복구한 것으로 보여 원래 증상은 재현하지 못했다. 이 문서는 ① 복구 후 North-South 경로의 정상 기준선과 ② 점검 중 만난 outbound 실패를 5회차 장애로 오해했다가 다른 원인(6회차 SG 장애)으로 구분한 과정을 기록한다.

## 0. 내 환경 메모

| 항목 | 값 |
| --- | --- |
| 라우터 | `cc01-router` · UP / ACTIVE · 외부 GW **203.0.113.100** · `enable_snat: true` |
| 라우터 네임스페이스 (net01) | `qrouter-5ec32c60-6a9b-47cc-bb1d-9266a120bf27` |
| 바깥쪽 포트 `qg-ead8ef8d-f0` | `203.0.113.100/24` + `203.0.113.214/32` + `203.0.113.178/32` (FIP는 이 포트에 붙는다) |
| 안쪽 포트 `qr-fff49bcc-fa` | `10.101.0.1/24` |
| vm1 | 10.101.0.15 · **FIP 203.0.113.214** (포트 `3a5bc23b-2d`) |
| vm2 | 10.101.0.7 · **FIP 203.0.113.178** (포트 `a49896ed-e5`) |
| vm3 | 10.101.0.180 · **FIP 없음** |

## 1. 읽기 핵심 (한 장)

### Learning OpenStack Networking 6장 — NAT, Floating IP, 패킷 흐름

```
                       qrouter-5ec32c60-… 네임스페이스
 net01 호스트           ┌──────────────────────────────────────────────┐
 veth-gw 203.0.113.1 ─ br-ex ─ qg-ead8ef8d-f0                qr-fff49bcc-fa ─ br-int … VM
                       │  203.0.113.100 / .214 / .178     10.101.0.1   │
                       │                                               │
 ③ inbound   dst 203.0.113.214 ──PREROUTING DNAT──▶ dst 10.101.0.15    │
 ② outbound  src 10.101.0.15   ◀─POSTROUTING SNAT── src 203.0.113.214  (FIP 있는 VM: float-snat)
             src 10.101.0.180  ◀─POSTROUTING SNAT── src 203.0.113.100  (FIP 없는 VM: snat)
                       └──────────────────────────────────────────────┘
```

- **주소가 바뀌는 위치:** 전부 라우터 네임스페이스 **안**에서 바뀐다. qg 바깥(br-ex, net01 호스트)에서 보면 FIP나 `203.0.113.100`이고, qr 안쪽(br-int, VM)에서 보면 fixed IP다. 그래서 qr과 qg에서 같은 조건으로 tcpdump하면 NAT 전후가 비교된다.
- **들어오는 방향 (DNAT):** `neutron-l3-agent-PREROUTING -d 203.0.113.214/32 -j DNAT --to-destination 10.101.0.15`. 라우팅 결정 **전에** 목적지를 바꿔야 qr 쪽으로 나간다.
- **나가는 방향 (SNAT):** `neutron-postrouting-bottom → neutron-l3-agent-snat → neutron-l3-agent-float-snat`. FIP 있는 VM은 `-s 10.101.0.15 -j SNAT --to-source 203.0.113.214`(자기 FIP), 그 외는 `-o qg-… -j SNAT --to-source 203.0.113.100`(라우터 GW IP 하나를 공유). 라우팅 결정 **후**, 나가는 인터페이스가 qg로 정해진 뒤에 바뀐다.
- **FIP는 VM 안에 없다:** VM의 `ip addr`에는 fixed IP만 있다. FIP는 qg 포트의 `/32` 주소와 DNAT·SNAT 규칙 한 쌍으로만 존재한다. "FIP가 있다"와 "FIP로 나간다"는 **다른 규칙**(DNAT vs float-snat)이 책임진다. 그래서 inbound만 되거나 outbound만 되는 장애가 가능하다.
- 우리 환경과의 차이: 책의 compute 쪽은 LinuxBridge라 라우터 네임스페이스 부분만 대입했다. 책의 물리 라우터 대신 실습망은 net01 호스트의 `veth-gw 203.0.113.1`이 외부 GW 역할을 한다.

### SRE 12장 (In Practice ~ Negative Results Are Magic)

- 실패 범위를 가장 빨리 줄이는 실험은 **경로의 중간을 자르는 실험**이었다. "라우터 네임스페이스 자신이 밖으로 나가는가"(라우터 바깥 전체를 한 번에 판정)와 "VM 요청이 qr에 도착하는가"(라우터 안 전체를 한 번에 판정) 두 개로 라우터·NAT 가설을 모두 버렸다.
- 가설을 버리게 한 관찰은 **기대와 반대인 결과**였다. 라우터 장애라면 VM 요청이 qr까지는 와야 하는데 0건이었다. 이 부정적 결과가 다음 주(6회차) 과제의 출발점이 됐다.

## 2. 받은 증상

### ① 내부 / ② outbound / ③ FIP inbound 분리 테스트 (11/28)

| VM | ③ FIP ping (net01 호스트) | ③ FIP SSH (net01 호스트) | 라우터 → VM SSH (qrouter→fixed IP) | ① 내부 (VM 안에서) | ② outbound (VM 안에서: 203.0.113.1 / 8.8.8.8 / curl) |
| --- | --- | --- | --- | --- | --- |
| vm1 | ✅ 3/3, ttl 63 | ✅ OK | ✅ | ❌ vm2·vm3·GW 모두 실패 | ❌ / ❌ / ❌ `000` |
| vm2 | ✅ 3/3, ttl 63 | ✅ OK | ✅ | — | ❌ / ❌ / ❌ `000` |
| vm3 | 미적용 (FIP 없음) | 미적용 | ✅ | — | ❌ / ❌ / ❌ `000` |

| 라우터 자체 | 명령 | 결과 |
| --- | --- | --- |
| 라우터 → 외부 GW | `sudo ip netns exec qrouter-… ping -c2 -W1 203.0.113.1` | ✅ 2/2 |
| 라우터 → 인터넷 | `sudo ip netns exec qrouter-… ping -c2 -W1 8.8.8.8` | ✅ 2/2, ttl 110 |
| Grafana FIP 프로브 | `localhost:3001` | 확인하지 못함 |

> ①② VM 안에서 시작하는 통신은 전부 실패했지만, 아래 3번에서 보듯 이 실패는 **라우터 경로가 아니라 VM 포트의 시큐리티 그룹**(6회차 장애, 11/24 변경)에서 생긴 것이다. 그래서 5회차 장애 판정에서는 ①②를 **판정 불가(다른 장애와 겹침)** 로 두고, 라우터·NAT 경로 자체는 아래 근거로 **정상**으로 판정했다.

## 3. 가설 · 검증 기록 ★

| # | 가설 | 확인 방법 (명령) | 결과 | 판단 |
| --- | --- | --- | --- | --- |
| ① | 장애가 이미 복구됐다 (세션 후 5일 경과) | 증상 재현부터 시도 | FIP inbound는 정상, VM outbound는 실패 → 일부 실패가 남아 보임 | 보류 → 남은 실패가 5회차 장애인지 확인 |
| ② | OpenStack 쪽 라우터·FIP 구성이 깨졌다 | `openstack router show cc01-router` · `openstack floating ip list` · `openstack server list` | 라우터 UP/ACTIVE, `enable_snat true`, GW 203.0.113.100. FIP 2개가 vm1·vm2 포트에 정상 연결 | **기각** |
| ③ | qg·qr 인터페이스나 라우팅이 이상하다 | `sudo ip netns exec qrouter-… ip -br addr` · `ip route` | qg에 GW IP + FIP 2개, qr `10.101.0.1/24`, `default via 203.0.113.1 dev qg-…` | **기각** |
| ④ | DNAT/SNAT 규칙이 빠지거나 바뀌었다 | `… iptables -t nat -S` (전체) | PREROUTING/OUTPUT DNAT 2쌍, float-snat 2줄, snat(GW IP) 2줄, `POSTROUTING → neutron-postrouting-bottom → snat → float-snat` jump 연결 모두 존재 | **기각** |
| ⑤ | NAT 테이블 밖(filter/mangle)에 끼워 넣은 규칙이 있다 | `… iptables -S` · `iptables -t mangle -S` · `iptables-save \| grep -i study` · `iptables-save \| grep -vE 'neutron\|…'` | 표준 Neutron 규칙만 있음 (`scope` DROP, mark/connmark는 기본 구성). 주석·비Neutron 규칙 없음 | **기각** |
| ⑥ | 라우터 바깥(qg → br-ex → veth → 203.0.113.1 → 인터넷)이 끊겼다 | 라우터 네임스페이스에서 `ping 203.0.113.1`, `ping 8.8.8.8` | 둘 다 성공 | **기각** → 라우터 바깥은 정상 |
| ⑦ | VM outbound 실패는 라우터 안(NAT/forward)에서 생긴다 | qr에서 `tcpdump -nei qr-fff49bcc-fa 'host 10.101.0.15 and (icmp or arp)'` + vm1 안에서 `ping 10.101.0.1` | **0 packets.** vm1 요청이 라우터에 도착조차 안 함. 라우터를 안 거치는 vm1→vm3(같은 노드)도 실패 | **기각** → 실패는 라우터보다 앞, VM 쪽 |
| ⑧ | 남은 실패는 VM 포트의 시큐리티 그룹 문제다 (5회차 장애 아님) | `openstack security group rule list cc01-sg` · `openstack security group show cc01-sg -c updated_at` · cmp01 tap/qvb tcpdump | egress 규칙 0개, SG 변경 **2026-11-24 12:58 KST** (5회차 세션 다음 날). tap에는 요청이 보이고 qvb엔 없음 | **확인** → 6회차 장애. 자세한 추적은 `week6/hyeongjun-week6.md` |

막힌 지점:
- 원래 5회차 장애는 복구 후라 **무엇이었는지 확인하지 못했다.** 세션 자료나 운영진 기록으로만 알 수 있다.
- qr·qg 동시 tcpdump로 NAT 전후 주소를 패킷으로 직접 보려 했으나, VM 안에서 시작하는 패킷이 6회차 SG 장애로 qr까지 오지 않아 outbound NAT는 캡처로 확인하지 못했다. inbound는 net01→FIP ping의 ttl 63(라우터 1홉)과 DNAT 규칙으로만 확인했다.

## 4. 원인 · 복구 방안

- **원인 (한 문장):** **미해결 / 재현 불가.** 점검 시점(11/28)에는 라우터·NAT·외부 경로가 모두 정상이었다. 5회차 장애는 세션(11/23) 복구 이후 사라진 것으로 판단한다.
- **실패 범위 구체화 (점검 시점):**

  | 방향 | 결과 | 원인 |
  | --- | --- | --- |
  | ③ 외부 → FIP → VM | 정상 | — |
  | 라우터 → VM, 라우터 → 외부 | 정상 | — |
  | ①② VM이 시작하는 통신 전부 | 실패 | 6회차 SG egress 규칙 삭제 (라우터 경로 아님) |

- **마지막 정상 지점 (North-South 경로 기준):**
  - inbound: net01 호스트 → `veth-gw` → br-ex → **qg (DNAT 203.0.113.214 → 10.101.0.15)** → qr → … → vm1까지 **전 구간 정상** (FIP ping ttl 63, SSH OK)
  - outbound: 라우터 네임스페이스 → qg → `203.0.113.1` → 8.8.8.8까지 정상. VM 쪽에서는 **`tap3a5bc23b-2d`가 마지막 정상 지점**이고 `qvb`에서 사라진다. 라우터 경로 밖의 문제다.
- **"SG 문제"와 "라우터 경로 장애"를 구별한 관찰:**
  1. 라우터 네임스페이스 자신은 외부로 나간다 → 라우터 바깥 정상
  2. VM ping 중 qr 캡처 0건 → 라우터에 도착조차 안 함
  3. 라우터를 거치지 않는 같은 노드 VM끼리도 실패
  4. tap ✅ / qvb ❌ → VM 바로 옆 qbr(iptables)에서 끊김
  5. 들어오는 연결은 되고 나가는 연결만 실패 → 상태 기반 SG의 전형적 모양 (NAT 장애라면 방향보다는 **FIP 유무**에 따라 갈렸을 것. 실제로는 FIP 있는 vm1·vm2와 없는 vm3이 똑같이 실패)
- **복구하려면:** 5회차 장애는 이미 복구됨. 점검 중 발견한 SG 장애의 복구 제안은 6회차 문서에 적었다 (`cc01-sg`에 egress IPv4/IPv6 기본 규칙 재생성).
- **복구 후 무엇으로 확인하면 되는지:** SG 복구 후 qr·qg에서 같은 조건으로 tcpdump하며 vm1 안 `ping 8.8.8.8`. qr에선 `10.101.0.15 > 8.8.8.8`, qg에선 `203.0.113.214 > 8.8.8.8`(float-snat)로 보이면 outbound NAT까지 정상이다. vm3는 qg에서 `203.0.113.100 > 8.8.8.8`(snat)로 보여야 한다.

## 5. SRE 세 질문

1. **사용자가 무엇을 겪었나:** (5회차 장애 자체는 재현 불가) 점검 시점엔 VM에 들어갈 수는 있지만 VM 안에서 밖으로 아무것도 안 되는 상태였다. 사용자 입장에서는 둘 다 "외부 통신이 일부 안 된다"로 똑같이 보인다.
2. **어떤 증상으로 알아챘나:** 방향을 나눈 테스트. inbound(FIP ping·SSH)만 봤다면 정상으로 결론 냈을 것이다.
3. **운영자가 미리 볼 수 있었던 지표는:** Grafana FIP 프로브는 확인하지 못했다. 그 프로브는 net01 → FIP로 **들어오는** 방향만 재므로, outbound 장애나 SG egress 장애에는 반응하지 않는다.
   - **가장 먼저 실패했어야 할 synthetic probe:** 슬롯마다 **FIP 있는 VM 하나와 FIP 없는 VM 하나**에서 1분마다 VM 안에서 시작하는 `ping 203.0.113.1` + `ping 8.8.8.8` + `curl http://example.com`. 두 VM을 함께 재야 float-snat(FIP)와 snat(GW IP) 경로를 각각 볼 수 있고, 단계별로 나눠야 실패 구간(라우터 바깥 첫 홉 / 인터넷 / DNS)이 바로 보인다.
   - 이 프로브는 이번처럼 라우터가 아닌 SG에서 막힌 경우에도 실패하므로, **inbound 프로브와 짝지어** 볼 때 "outbound만 실패 = 라우터 SNAT 또는 SG egress"로 범위를 줄일 수 있다.
