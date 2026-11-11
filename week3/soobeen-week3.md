# 3회차 제출 — 손수빈

> 계정: `cc03` · 작성일: 2026-11-09
> 

## 1. 읽기 핵심 (한 장)

- 같은 노드와 다른 노드 간 통신 경로
    - 같은 노드: tap~br-int 안에서 끝남 (Local VLAN만 사용)
        - vm1→vm3가 되는 건 vm1의 송신 경로와 cmp01 안에서의 통신만 배제
    - 다른 노드: br-int → patch-tun으로 br-tun → VXLAN으로 encap(VNI) → 상대 노드 br-tun에서 decap → br-int → VM
        - 변환 번호가 2가지(Local VLAN ↔︎ VNI)
- 브리지와 포트가 모두 정상이어도 **flow 규칙이 패킷을 통과시킬지 버릴지를 결정**
- flow 규칙: OVS 브리지가 패킷을 만났을 때 어떻게 처리할지 정해 둔 조건과 동작의 쌍
    - priority=3, in_port=2, dl_vlan=30  → mod_vlan_vid:1, NORMAL   (VLAN 30으로 들어오면 Local VLAN 1로 바꿔 전달)
    - priority=2, in_port=2   → drop   (같은 포트에서 위에 안 맞는 나머지는 버림)
    - priority=1 → NORMAL (그 외는 일반 스위치처럼 전달)

```
priority=3, in_port=2, dl_vlan=30   actions=mod_vlan_vid:1,NORMAL
└──── 매칭 조건(어떤 패킷?) ────┘   └──── 동작(어떻게 처리?) ────┘
```

| 구성 | 뜻 | 예 |
| --- | --- | --- |
| 매칭 조건 | 어느 포트에서 들어왔고 어떤 태그, 번호를 달고 있는가 | `in_port`(들어온 포트 번호), `dl_vlan`(Local VLAN), `tun_id`(VNI) |
| `actions` | 맞는 패킷에 할 일 | `mod_vlan_vid`(VLAN 번호 바꿈), `output`(특정 포트로 내보냄), `NORMAL`(일반 스위치처럼 학습해서 전달), `drop`(버림) |
| `priority` | 여러 규칙에 동시에 맞을 때 우선순위. 숫자가 큰 규칙의 `actions`가 실행됨 |  |
| `n_packets` | 이 규칙에 맞은 패킷 수(카운터) | 숫자가 오르면 패킷이 이 규칙을 타고 있다는 증거 |
- 블랙박스는 지금 실제로 안 되는 걸 사용자 관점에서 보는 것이고 화이트박스는 내부 지표를 보는 것
    - 에이전트 Alive는 프로세스가 살아 있다는 신호일 뿐 패킷이 통한다는 증거가 아님 → 증상(프로브)과 내부 카운터(flow 카운터)를 함께 봐야 함

## 2. 받은 증상

| 확인한 것 | 명령 | 결과 |
| --- | --- | --- |
| vm2 상태 | `openstack server show cc03-vm2` | ACTIVE |
| vm1 ↔ vm2 ping | `ping -c 3 10.103.0.x` | 0 received |
| vm1 → vm3 ping | `ping -c 3 10.103.0.x` | 3 received |

## 3. 가설 · 검증 기록 ★

**가장 중요한 부분입니다.** 순서대로, 틀린 가설도 지우지 말고 남기세요.

| # | 가설 (어디가 문제일 것 같다) | 확인 방법 (명령) | 결과 | 판단 |
| --- | --- | --- | --- | --- |
| 1 | vm2가 vm1의 MAC을 알아내지 못한다(L2 문제) | vm2에서 `ip neigh show` | 10.103.0.75 FAILED(probes 3, MAC 없음), 10.103.0.1과 10.103.0.2는 MAC 있음 | 유지(ARP 응답 못 받음) |
| 2 | cmp02에서 vm2의 장치 연결이 끊겼다 | `ip -br link show master qbreb99e8d2-ed` | 장치 연결 정상(`qvbeb99e8d2-ed` UP, `tapeb99e8d2-ed` UNKNOWN) | 기각 |
| 3 | vm2의 qvo가 br-int에 붙어 있지 않다 | `sudo docker exec openvswitch_vswitchd ovs-vsctl show | grep -A3 qvoeb99e8d2-ed` | `Port qvoeb99e8d2-ed`, `tag: 4` | 기각(cmp02 Local VLAN=4) |
| 4 | cmp02 br-tun에 cmp01로 가는 터널 포트가 없다 | `sudo docker exec openvswitch_vswitchd ovs-ofctl show br-tun` | state 0 | 기각 |
| 5 | cmp02 br-tun에 VLAN 4의 flow가 없다 | cmp02에서 `ovs-ofctl dump-flows br-tun`을 `dl_vlan=4`, `mod_vlan_vid:4`로 필터 | 모든 구성 존재(VNI, 여러 table)  | 기각 |
| 6 | flow에 이상이 있다 | cmp02에서 `sudo docker exec openvswitch_vswitchd ovs-ofctl dump-flows br-tun table=0` | `priority=30000,tun_id=0x8a(138),in_port=5 actions=drop` (cookie 0x330303) 발견 | 유지 |
| 7 | priority=30000 규칙이 적용되어 패킷이 막힌다 | cmp02에서 `dump-flows br-tun table=0 | grep tun_id=0x8a` | 패킷 카운터 54→60, `idle_age` 9321→34초 | 유지 |
| 8 | 양방향 실패였으니 cmp01에도 같은 규칙이 있다 | cmp01에서 `dump-flows br-tun table=0`, `ovs-ofctl show br-tun`을 vxlan으로 grep | `priority=30000,tun_id=0x8a,in_port=4 actions=drop` (cookie 0x330303) 발견 | 유지 |

## 4. 원인 · 복구 방안

복구는 **실행하지 않습니다.** 어떻게 고치면 될지만 적습니다.

- **원인 (한 문장):** cmp02와 cmp01의 `br-tun` table 0에 같은 cookie의 `priority=30000,tun_id=0x8a actions=drop` 규칙이 하나씩 있어, 두 노드가 서로 상대 터널에서 도착한 VNI 138 패킷을 Local VLAN으로 변환하기 전에 버린다.
- **끊긴 지점이 이 증상으로 보인 이유:**
    - vm1→vm2: cmp01 송신, 캡슐화, 전송, cmp02 NIC 수신까지 tcpdump로 확인, cmp02 `br-tun` table 0에서 막힌다. (VNI 138, `in_port=5`, cmp02 Local VLAN 4)
    - vm2→vm1: cmp02 송신, 전송, cmp01 NIC 수신까지 tcpdump로 확인, cmp01 `br-tun` table 0에서 막힌다 (VNI 138, `in_port=4`)
- **복구하려면 (명령 또는 절차):**
    1. cmp02에서 그 규칙만 삭제: `sudo docker exec openvswitch_vswitchd ovs-ofctl --strict del-flows br-tun "cookie=0x330303/-1,table=0,priority=30000,tun_id=0x8a,in_port=5"`
    2. cmp01에서 그 규칙만 삭제: `sudo docker exec openvswitch_vswitchd ovs-ofctl --strict del-flows br-tun "cookie=0x330303/-1,table=0,priority=30000,tun_id=0x8a,in_port=4"`
- **복구 후 무엇으로 확인하면 되는지:**
    - `sudo docker exec openvswitch_vswitchd ovs-ofctl dump-flows br-tun table=0 | grep tun_id=0x8a`에서 drop 규칙이 사라졌는지 확인(두 노드 다)

## 5. SRE 세 질문

1. **사용자가 무엇을 겪었나:** 같은 프로젝트의 vm1·vm3는 서로 통신하는데 다른 컴퓨트 노드의 vm2와는 통신이 되지 않는다.
2. **어떤 증상으로 알아챘나:** cmp02와 cmp01의 `br-tun` table 0에 같은 cookie의 `priority=30000,tun_id=0x8a actions=drop` 규칙 확인 및 tcpdump
3. **운영자가 미리 볼 수 있었던 지표는:**
    - Black-box: 별도의 probe 확인하지 못함(대시보드 미존재?)
    - white-box 증거: cmp02 `br-tun` table 0의 `priority=30000,tun_id=0x8a,in_port=5 actions=drop` 규칙의 `n_packets`가 ping 전후로 54에서 60으로 늘고 `idle_age`가 9321초에서 34초가 됨.

## 6. 한 줄 회고

- 2주차 때처럼 장치 연결부터 확인해서 vm2 로컬 구간을 빨리 배제했지만, 이번 원인은 구성이 아니라 `br-tun` table 0의 `priority=30000` drop 규칙이었다. 빨리 찾으려고 VLAN과 VNI로만 grep했다가 `in_port` 기준 규칙은 놓칠 뻔했다. table 0부터 `priority`로 읽어야 찾을 수 있다는 걸 배웠다.