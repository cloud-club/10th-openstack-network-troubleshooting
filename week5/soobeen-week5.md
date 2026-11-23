# 5회차 제출 — 손수빈

> 계정: `cc03` · 작성일: 2026-11-23

## 1. 읽기 핵심 (한 장)


- **NAT 종류**: FIP는 one-to-one(static NAT), FIP가 없는 VM은 many-to-one(PAT, Source NAT)로 나간다. 라우터가 `qg` 쪽으로 나가며 **출발지를 자기 외부 주소로 바꾸고**, 응답이 오면 목적지를 되돌린다. 그래서 FIP 없는 VM은 먼저 시작한 통신만 밖과 오갈 수 있다
- **FIP 위치**: VM 안이 아니라 라우터 네임스페이스의 **`qg`에 붙은 secondary 주소 + NAT 규칙**이다. 그래서 OpenStack DB(`floating ip list`)와 네임스페이스의 실제 상태는 따로 확인해야 한다
- **qg와 qr**: `qg`는 라우터의 바깥쪽(외부) 인터페이스, `qr`은 안쪽 인터페이스(서브넷 `gateway_ip`). 같은 패킷이 `qr`과 `qg`에서 **다른 주소로 보이는 것이 정상**이다(출발지 또는 목적지가 NAT에서 바뀜)
- **SRE 12장**: 경로의 가운데를 반으로 나눠 보는 **bisection**, 시험한 위치가 결과를 바꾸는 **교란 요인**, 기각한 가설과 "변화 없음"도 남기는 **Negative results** -> 이번 주에는 `qr`과 `qg`를 가운데 지점으로 잡아 비교함

경로:

```
vm → qr-bd669f9f-3f ─[ qrouter 안 NAT ]─ qg-5801a520-01 → br-int → int-br-ex/phy-br-ex → br-ex → veth-ext → veth-gw(203.0.113.1)
```

**책/공식 문서와 우리 환경(OVS)의 차이:**
- **qg의 위치:** 책은 LinuxBridge라서 `qg`의 반대쪽이 외부 VLAN 브리지에 들어간다. 우리 환경은 `qg`와 `qr`이 둘 다 `br-int`에 있고, `br-ex`에는 `phy-br-ex`와 `veth-ext`만 있다. `int-br-ex`↔`phy-br-ex`가 `br-int`와 `br-ex`를 잇는다
- **외부 게이트웨이:** 책은 Cisco ASA이고 우리는 net01 호스트의 `veth-gw`
- **NAT 규칙의 모양:** 우리 환경은 체인이 `neutron-l3-agent-snat`와 `float-snat`로 나뉘고 SNAT에 `--random-fully`가 붙으며, SNAT 후 ICMP id도 바뀐다(`39836` → `7901`)


## 2. 받은 증상 · VM별 결과 표

**신고:** "같은 네트워크 안에서는 되는데, 외부에서 내 VM으로 들어오거나 VM에서 밖으로 나가는 것 중 일부가 실패한다."

| VM | fixed IP | FIP | ① 내부 ping | ② 게이트웨이 `203.0.113.1` | ② `8.8.8.8` | ② 도메인 (`curl example.com`) | ③ FIP inbound (ping / SSH) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| vm1 | `10.103.0.75` | `203.0.113.205` | vm2, vm3 OK (3/3) | OK | OK | OK | OK / OK |
| vm2 | `10.103.0.118` | `203.0.113.156` | vm1, vm3 OK (3/3) | OK | OK | OK | OK / OK |
| vm3 | `10.103.0.195` | **없음** | vm1, vm2 OK (3/3) | **실패** | **실패** | **실패** | FIP 없음 |

## 3. 가설 · 검증 기록 ★

| # | 가설 (어디가 문제일 것 같다) | 확인 방법 (명령) | 결과 | 판단 |
| --- | --- | --- | --- | --- |
| 1 | 환경 구성(FIP 연결, 라우터)이 틀렸다 | ops01 `openstack server list`, `floating ip list`, `router show cc03-router` | vm1·vm2에 FIP(205, 156) 연결, vm3은 없음. 라우터 ACTIVE, 외부 주소 203.0.113.107, enable_snat: true | 기각 |
| 2 | 라우터 네임스페이스의 인터페이스, 경로가 비정상이다 | net01 `ip netns exec qrouter-10322d7c-… ip -br addr`, `ip route` | qg: 203.0.113.107/24과 FIP 2개. qr: 10.103.0.1/24. default via 203.0.113.1 | 기각 |
| 3 | 환경 전체/외부 전체가 문제다 | vm1·vm2의 outbound 3단계, net01에서 FIP ping/SSH | vm1·vm2는 게이트웨이, 8.8.8.8, 도메인, FIP ping·SSH 모두 OK | 기각 |
| 4 | [vm3] VM의 게이트웨이 설정이 틀렸다 | vm3 안 `ip route` | `default via 10.103.0.1 dev eth0` | 기각 |
| 5 | [vm3] 요청이 라우터에 도착하지 않는다 | `qr-bd669f9f-3f`에서 `tcpdump -nei … -c 6 'icmp and host 8.8.8.8'`, vm3에서 `ping -c3 8.8.8.8` | qr에 요청 3개(출발지 10.103.0.195 → 8.8.8.8), 응답 없음 | 기각(요청이 qr까지 도착해 라우터가 받음) |
| 6 | [vm3] 라우터 바깥(`qg`, `br-ex`, 호스트)에서 막힌다 | `qg-5801a520-01`에서 같은 tcpdump, `veth-gw`에서 `tcpdump -nei veth-gw 'icmp and host 10.103.0.195'`, `ovs-vsctl list-ports br-ex` | qg에서 출발지 10.103.0.195 그대로 나감. veth-gw(203.0.113.1)에도 사설 출발지로 도착, 응답 없음. br-ex 포트는 phy-br-ex, veth-ext | 기각(패킷이 veth-gw까지 도달함) |
| 7 | [vm3] FIP 없는 VM의 기본 SNAT가 적용되지 않는다 | net01 `ip netns exec qrouter-10322d7c-… iptables -t nat -S`, vm1과 비교 | neutron-l3-agent-snat에 float-snat 점프 한 줄뿐. float-snat에는 vm1·vm2 규칙만 있고 vm3용·기본 SNAT 없음. 대조: vm1은 qr 10.103.0.75 → qg 203.0.113.205로 바뀌고 응답 3/3 | 유지 |


## 4. 원인 · 복구 방안

- **마지막 정상 지점:** 요청이 `qr`로 들어와 라우터가 `qg`로 내보내고 호스트 `veth-gw`까지 도달하는 것(라우팅과 외부 연결). 처음 이상이 보이는 곳은 `qg` 출구의 **출발지 사설 IP 미변환**
- **원인 (한 문장):** `qrouter-10322d7c-…`의 NAT 테이블 `neutron-l3-agent-snat` 체인에 FIP 없는 VM이 라우터 외부 주소(`203.0.113.107`)로 나가게 하는 **기본 SNAT 규칙이 없어서**, vm3의 패킷이 사설 출발지 `10.103.0.195` 그대로 나가고 외부가 응답하지 못한다
- **이 증상으로 보인 이유:**
    - vm1·vm2는 FIP가 있어 `float-snat`의 `-s <fixed> -j SNAT --to-source <FIP>` 규칙을 지나므로 정상. vm3은 FIP가 없어 어떤 규칙에도 안 걸려 변환 없이 나간다.
    - 내부 통신과 FIP inbound는 NAT 규칙의 FIP 쪽(DNAT, `float-snat`)과 `qr`~`qg` 라우팅만 쓰므로 영향이 없다. 그래서 Horizon과 openstack 명령에는 모두 정상(`enable_snat: true`)으로 보였다

- **복구하려면 (명령 또는 절차):**
    1. **L3 agent가 이 라우터를 다시 동기화**하게 한다. DB에는 `enable_snat: true`가 있으므로 agent가 규칙을 다시 만들면 정상 구성으로 돌아간다.
    2. 삭제된 규칙 두 개를 같은 모양으로 추가한다. qrouter에서 `iptables -t nat -A neutron-l3-agent-snat -o qg-5801a520-01 -j SNAT --to-source 203.0.113.107 --random-fully`, `iptables -t nat -A neutron-l3-agent-snat -m mark ! --mark 0x2/0xffff -m conntrack --ctstate DNAT -j SNAT --to-source 203.0.113.107 --random-fully`
    
- **복구 후 무엇으로 확인하면 되는지:**
    - vm3 안에서 `ping -c3 203.0.113.1`, `ping -c3 8.8.8.8`, `curl -s -m5 -o /dev/null -w '%{http_code}\n' http://example.com`이 모두 성공
    - `qg-5801a520-01`에서 vm3의 `8.8.8.8` ping 출발지가 `203.0.113.107`로 보임
    - `ip netns exec qrouter-10322d7c-… iptables -t nat -S`의 `neutron-l3-agent-snat`에 `203.0.113.107`로 SNAT하는 규칙이 있는지

## 5. SRE 세 질문

1. **사용자가 무엇을 겪었나:** FIP 없는 VM(vm3)에서 밖으로 못 나간다.
2. **어떤 증상으로 알아챘나:** vm3의 패킷이 사설 출발지 10.103.0.195 그대로 나가고 외부가 응답하지 못했다. Grafana의 경로(DNAT과 FIP 쪽 SNAT)는 장애의 영향을 받지 않기 때문에 **모두 OK**였다.
3. **운영자가 미리 볼 수 있었던 지표는:**
    - 기존 프로브가 이 장애를 못 잡은 이유: **FIP 없는 VM이 밖으로 나가는 경로**(기본 SNAT)를 지나는 프로브가 없다. 1-3(FIP)은 DNAT 쪽, 4는 라우터 내부(`qrouter`→VM), 5-7은 VM 사이, 8은 메타데이터, 9(`SSH 대용량 vm→ops01`)는 outbound지만 **FIP가 있는 vm1, vm2에서만** 돌아서 `float-snat` 경로만 지난다.
    - Black-box: FIP 없는 VM에서 203.0.113.1로 ping하는 프로브가 가장 먼저 실패했어야 한다. qrouter→게이트웨이, FIP 없는 VM→게이트웨이/8.8.8.8/HTTP, FIP 있는 VM→8.8.8.8로 구간을 나누면 기본 SNAT만 문제인 걸 바로 확인 가능
    - White-box: enable_snat: true인 라우터마다 neutron-l3-agent-snat 체인에 외부 주소로의 SNAT 규칙이 있는지 대조하는 지표

## 6. 한 줄 회고

- vm1을 대조군으로 두고 DB, 네임스페이스, 경로, NAT 규칙 순서로 범위를 좁힌 것이 효과적이었다. 역시 NAT는 까다롭다..