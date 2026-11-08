# 3회차 제출 — hyeongjun

> 계정: `cc01` · 작성일: 2026-11-08
> 주제: 노드 사이를 건너면 — vm2만 통신이 안 된다 (East-West, VXLAN, OVS flow)

## 0. 내 환경 메모 (정상 기준선)

| 항목 | 값 |
| --- | --- |
| 프로젝트 / 네트워크 | `study-cc01` / `cc01-net` |
| **VNI** (`provider:segmentation_id`) | **813** (flow 표기 `tun_id=0x32d`) — API에선 권한 때문에 `None`, cmp01·cmp02 qvo의 `other_config`에서 확인 |
| vm1 (cmp01, 192.168.0.21) | 내부 IP: `10.101.0.15` · 포트 앞 11자리: `3a5bc23b-2d` · FIP: `203.0.113.214` |
| vm2 (cmp02, 192.168.0.22) | 내부 IP: `10.101.0.7` · 포트 앞 11자리: `a49896ed-e5` · FIP: `203.0.113.178` |
| vm3 (cmp01) | 내부 IP: `10.101.0.180` · 포트 `6925aa2d-0f1d-47d8-a078-9a566bb79db1` (MAC `fa:16:3e:c4:a7:88`) |
| Local VLAN (cmp01, vm1의 qvo tag) | **2** |
| Local VLAN (cmp02, vm2의 qvo tag) | **2** |
| cmp01 `br-tun` OFPort | `1(patch-int)` · `2(vxlan-c0a8000c)`→net01 · `4(vxlan-c0a80016)`→cmp02 |
| cmp02 `br-tun` OFPort | `1(patch-int)` · `4(vxlan-c0a8000c)`→net01 · `5(vxlan-c0a80015)`→cmp01 |

## 1. 읽기 핵심 (한 장)

### Learning OpenStack Networking 4장 (p.102–112) — OVS, internal connections, flow rules

```
[cmp01]                                                         [cmp02]
vm1 → tap → qbr → qvb ═ qvo → br-int ─patch─ br-tun ─vxlan-c0a80016══╗
                              (VLAN 2)       (VLAN 2 → VNI 813        ║ underlay 192.168.0.21 → .22
                                              캡슐화, table 20/22)    ║ UDP 4789, VNI 813
                                                                      ╚══vxlan-c0a80015(port 5)─ br-tun ─patch─ br-int → qvo ═ qvb → qbr → tap → vm2
                                                                         (table 0 → 4: VNI 813 → VLAN 2         (VLAN 2)
                                                                          디캡슐화 → table 10 → patch-int)
```

| 개념 | 어디서 쓰이나 | 메모 |
| --- | --- | --- |
| `br-int` | 노드 안 통합 브리지. 모든 VM의 qvo가 붙음 | 같은 노드 VM끼리(vm1↔vm3)는 **여기서 끝남** — br-tun까지 안 감 |
| `br-tun` | 노드 사이 터널 브리지. 상대 노드마다 `vxlan-<IP16진수>` 포트 | 캡슐화(송신: table 2→20/22)·디캡슐화(수신: table 0→4→10)를 flow로 처리 |
| `patch-int` / `patch-tun` | br-int ↔ br-tun을 잇는 OVS 내부 패치 포트 | 리눅스 veth가 아니라 OVS끼리의 연결. br-tun 쪽 OFPort는 1 |
| Local VLAN | **노드 안**(br-int)에서 네트워크를 구분하는 번호 | 노드마다 독립적으로 할당 — 이번엔 우연히 양쪽 다 2 |
| VNI (`tun_id`) | **노드 사이** VXLAN 헤더에서 네트워크를 구분하는 번호 | 네트워크 전체에서 하나 (813 = `0x32d`) |
| flow `priority` / `n_packets` | 여러 규칙이 매칭되면 **priority가 가장 높은 규칙만** 실행 | `n_packets`·`idle_age`로 "지금 이 규칙이 패킷을 처리 중인지" 확인 가능 |

- 이번 장애와 연결되는 포인트:
  - **같은 노드 통신(br-int에서 끝남)과 다른 노드 통신(br-tun·VXLAN 경유)의 경로 차이**를 알았기 때문에, vm1↔vm3 성공 + vm1↔vm2 실패만으로 의심 구간을 br-tun·터널로 좁힐 수 있었다.
  - br-tun의 정상 수신 경로는 `table 0 (in_port=터널) → table 4 (tun_id → mod_vlan_vid) → table 10 → patch-int`. 이 경로 **앞에** 더 높은 priority의 규칙이 끼어들면, 포트·터널·정상 flow가 모두 있어도 패킷은 버려진다.
  - Local VLAN과 VNI의 번호 체계가 다르다는 걸 알아야 flow의 `dl_vlan=2`(노드 안)와 `tun_id=0x32d`(노드 사이)를 각각 우리 네트워크로 식별할 수 있다.

### SRE 6장 (p.84–88) — Symptoms vs Causes / Black-box vs White-box

- 에이전트 Alive ≠ 패킷이 통함인 이유:
  - `agent Alive`는 **에이전트 프로세스가 하트비트를 보낸다**는 white-box 신호일 뿐, 데이터플레인이 패킷을 전달한다는 뜻이 아니다. 에이전트가 직접 관리하지 않는 flow(쿠키가 다른 규칙)가 끼어들어도 에이전트는 계속 Alive다.
  - 6장 구분으로 보면 "vm2와 통신 불가"가 **증상(symptom)**, "cmp02 br-tun의 drop flow"가 **원인(cause)**. 사용자가 겪는 증상은 black-box(실제 ping)로만 바로 드러났고, API·에이전트 상태 같은 white-box 지표는 전부 "정상"이었다.
- 이번 과제에 적용한 점:
  - 상태값(ACTIVE, Alive, 포트 존재)을 정상 근거로 쓰지 않고, **실제 패킷(tcpdump)과 실제 처리 기록(flow 카운터)** 으로만 판단했다.
  - "구성이 있다(③)"와 "패킷이 전달된다(④⑤)"를 별도 가설로 나눠 검증했다.

## 2. 받은 증상 (비교 테스트 ②)

| 확인한 것 | 명령 | 결과 |
| --- | --- | --- |
| vm1 → vm3 (같은 노드) | (vm1 안에서) `ping -c 3 10.101.0.180` | **3/3 성공**, 0.45~0.70ms |
| vm1 → vm2 (다른 노드) | (vm1 안에서) `ping -c 3 10.101.0.7` | **3/3 손실** |
| **vm2 → vm1** (반대 방향) | (vm2 안에서) `ping -c 3 10.101.0.15` | **3/3 손실** · vm2 → vm3도 **3/3 손실** |
| vm2 → 게이트웨이 (net01) | (vm2 안에서) `ping -c 3 10.101.0.1` | **3/3 성공**, 0.65~2.25ms |
| ops01 → vm2 FIP | `ssh … -J cc01@192.168.0.12 cirros@203.0.113.178` | **SSH 접속 성공** → net01 ↔ cmp02 경로 정상 |
| ops01 → vm1 FIP (비교) | `ssh … cirros@203.0.113.214` | SSH 접속 성공 |
| API 상태 | `openstack server list` · `openstack network agent list` | VM 3대 모두 ACTIVE · agent list는 일반 계정 권한으로 출력 없음 (신고 내용상 Alive) |
| Grafana FIP 프로브 | `localhost:3001` | 대시보드에서 해당 프로브를 찾지 못해 확인하지 못함 |

> 정리: **cmp01 ↔ cmp02 사이만** 실패. 같은 노드(cmp01 안)와 net01 ↔ cmp02는 정상.

## 3. 가설 · 검증 기록 ★

경로 (vm1 → vm2): **vm1 → cmp01 br-int → br-tun (캡슐화, VNI 813) → VXLAN → cmp02 br-tun (디캡슐화, VNI→Local VLAN 2) → br-int → qvo → … → vm2**

| # | 가설 | 확인 방법 (명령) | 결과 | 판단 |
| --- | --- | --- | --- | --- |
| ① | VM·포트 배치나 VNI가 이상하다 | `openstack server list` · `openstack port list --server cc01-vm3` · `openstack network show cc01-net` | vm1·vm2·vm3 모두 **ACTIVE**, vm3 = 10.101.0.180 (포트 ACTIVE). `OS-EXT-SRV-ATTR:host`, `provider:*`는 **None** — 일반 계정 권한으론 안 보이는 관리자 필드 | 판단 보류 → 노드·VNI는 데이터플레인(③)에서 확인 |
| ② | 같은 노드는 되고 다른 노드만 안 된다 (방향별 차이?) | vm1→vm3, vm1→vm2, vm2→vm1, vm2→vm3, vm2→GW ping · ops01→vm2 FIP SSH | cmp01 안(vm1↔vm3) ✅ · cmp01↔cmp02 ❌ 양쪽 ping 모두 실패 · net01↔cmp02 ✅ | 유지 → 의심 구간 = **cmp01 ↔ cmp02 터널 구간**. 단 ping은 왕복이라 "양쪽 다 실패" ≠ "양방향 고장" → 방향은 ④에서 판별 |
| ③ | Local VLAN / VNI / br-tun 터널 포트가 없거나 틀렸다 | `ovs-vsctl list port qvo…` · `ovs-ofctl show br-tun` | 양쪽 모두 `segmentation_id=813`, `tag=2`. br-tun에 patch-int(1)와 상대 노드행 vxlan 포트 존재 (cmp01: 4→cmp02, cmp02: 5→cmp01) | 기각 → **구성 요소는 다 있음**. 남은 후보는 패킷 처리 규칙(flow) |
| ④ | VXLAN 패킷이 노드 사이를 오가지 않는다 | vm1에서 `ping 10.101.0.7` 지속 + (cmp01) `tcpdump -lni any -c 20 'udp port 4789 and host 192.168.0.22'` · (cmp02) `… and host 192.168.0.21` | cmp01 `eth0 Out 192.168.0.21 > 192.168.0.22:4789 vni 813` / 안쪽 `10.101.0.15 > 10.101.0.7 echo request` · cmp02 `eth0 In` 같은 패킷 **도착** (seq 138·139가 양쪽에서 확인) · **cmp02 → cmp01 방향은 0건** (echo reply·ARP reply 없음) | 부분 기각 → cmp01 송신·캡슐화·underlay **정상**. 실패는 **cmp02 도착 이후** |
| ⑤ | cmp02 br-tun flow가 cmp01발 VXLAN을 버린다 | (cmp02) `ovs-ofctl dump-flows br-tun` | **table 0에 `cookie=0x330301, priority=30000, tun_id=0x32d, in_port=5 actions=drop`** — `n_packets=399`, **`idle_age=0`**(방금도 매칭). 같은 조건을 처리해야 할 정상 규칙 `priority=1, in_port=5 actions=resubmit(,4)`보다 우선순위가 높아 먼저 실행됨. 쿠키도 에이전트 규칙(`0x9c8f24fb32fbbef7`)과 다름. **ping 중 5초 간격 카운터 비교:** drop 규칙 `n_packets` 588 → **592 (+4, ping 1초당 1개)**, 정상 규칙 `priority=1,in_port=5`는 881 → **881 (변화 없음)**, `idle_age` 866 → 871 (약 14분간 한 번도 매칭 안 됨) | **유지 → 원인 확정** |
| ⑤-b | cmp01 br-tun에도 같은 종류의 규칙이 있다 (반대 방향) | (cmp01) `ovs-ofctl dump-flows br-tun table=0` | **`cookie=0x330301, priority=30000, tun_id=0x32d, in_port=4 actions=drop`** (n_packets=22, idle_age=1215). cmp01 `in_port=4` = cmp02에서 오는 터널. 정상 규칙 `priority=1,in_port=4`는 idle_age=18236 | **유지 → 같은 장애의 반대편.** 한 변경(같은 쿠키)이 두 노드 수신 쪽에 대칭으로 적용 |
| ⑥ | cmp02 br-int에서 vm2로 가는 경로가 이상하다 | ⑤로 원인이 설명되어 우선순위 낮춤 · ②에서 net01 → vm2(FIP SSH)가 되므로 br-int → vm2 구간은 정상 | — | 기각 (FIP 경로가 같은 br-int·qvo·tap을 지남) |

### br-tun table 0 읽기 (cmp02) — 왜 이 규칙이 원인인가

```
priority=30000, tun_id=0x32d, in_port=5   actions=drop            ← cmp01(5)에서 온 VNI 813만 정확히 매칭 → 버림
priority=1,     in_port=4                 actions=resubmit(,4)    ← net01(4)에서 온 것: 정상 처리 → FIP·GW 통신 OK
priority=1,     in_port=5                 actions=resubmit(,4)    ← cmp01(5)에서 온 것: 원래 여기로 가야 함 (가려짐)
priority=1,     in_port=1                 actions=resubmit(,2)    ← vm2가 보내는 것(송신): 정상 → table 20/22에서 output:5
```

- 매칭 조건이 **`in_port=5`(cmp01 터널) + `tun_id=0x32d`(우리 VNI)** 두 개라서 증상이 정확히 설명된다:
  - net01 → cmp02 (`in_port=4`): 매칭 안 됨 → vm2 FIP·게이트웨이 통신 정상
  - cmp01 → cmp02, 다른 VNI: 매칭 안 됨 → 다른 프로젝트는 영향 없음
  - cmp01 → cmp02, VNI 813: **drop** → vm1·vm3 → vm2 실패
- **cmp01에도 같은 규칙이 있다** (같은 쿠키 `0x330301`): `priority=30000, tun_id=0x32d, in_port=4 actions=drop` — cmp01의 4번 포트 = `vxlan-c0a80016`(cmp02에서 오는 터널). 즉 장애는 **양쪽 노드의 수신 쪽에 대칭으로** 들어가 있다.
  - vm1 → vm2: request가 cmp02 table 0에서 drop (카운터 +4로 확인)
  - vm2 → vm1: vm2의 request는 정상 송신(table 2 → 20/22 → `output:5`)되지만 **cmp01 table 0에서 drop**. cmp01 규칙의 `n_packets=22`, `idle_age=1215`(약 20분 전 마지막 매칭)는 앞서 vm2 → vm1·vm3 ping을 보냈던 무렵과 맞는다 (시점 대조는 추정).
  - 처음엔 "vm2 → vm1은 reply가 cmp02에서 버려져서 실패"라고 가설을 세웠지만, cmp01을 확인하고 **request 단계에서 cmp01이 버린다**로 수정했다.
- 송신 경로 자체는 정상: cmp02 table 20에 vm1(`fa:16:3e:c9:a6:77`)·vm3(`fa:16:3e:c4:a7:88`)행 `load:0x32d → output:5` 규칙이 그대로 있음.

### 비교 관찰 (cmp01 vs cmp02)

| 확인 항목 | cmp01 | cmp02 | 차이 |
| --- | --- | --- | --- |
| Local VLAN (qvo tag) | 2 | 2 | 없음 (같은 값이지만 노드별로 독립) |
| br-tun OFPort (patch-int / 상대 노드행 vxlan) | 1 / 4(→cmp02) | 1 / 5(→cmp01) | 번호는 노드마다 다름 → flow 읽을 때 노드별로 대조 |
| VXLAN 패킷 (vm1 → vm2 ping 중) | `Out` → .22, vni 813 | `In` ← .21, vni 813 | 도착까지는 정상 |
| br-tun table 0 — 상대 노드 터널 + VNI 813 drop | **있음**: `cookie=0x330301, priority=30000, tun_id=0x32d, in_port=4 actions=drop` (n_packets=22, idle_age=1215) | **있음**: `cookie=0x330301, priority=30000, tun_id=0x32d, in_port=5 actions=drop` | **같은 쿠키·같은 형태가 양쪽에 대칭** |
| 정상 수신 규칙 (`priority=1,in_port=<상대 노드> → resubmit(,4)`) | in_port=4, idle_age=18236 (약 5시간 매칭 없음) | in_port=5, idle_age 866→871 | 양쪽 모두 가려져서 매칭 안 됨 |
| ping 중 5초간 `n_packets` 변화 (cmp02) | — | drop 규칙 **+4** / 정상 `in_port=5` 규칙 **+0** | 패킷이 전부 drop 규칙으로 감 |

> 참고: cmp02 table 0에는 `cookie=0x3303xx` 계열의 priority=30000 drop 규칙이 3개 더 있었다 (`tun_id=0x8a,in_port=5` / `in_port=1,dl_vlan=7` / `in_port=1,dl_vlan=8`). VNI·Local VLAN이 우리 값(0x32d / 2)과 달라 **다른 참가자의 장애**로 판단하고 제외했다 — `in_port=1,dl_vlan=…`은 수신이 아니라 **송신 쪽**을 막는 형태라, 과제지의 "사람마다 실패 방향이 다를 수 있다"와도 맞는다.

## 4. 원인 · 복구 방안 (⑦)

복구는 **실행하지 않습니다.**

- **원인 (한 문장):** cmp01·cmp02의 `br-tun` table 0에 "상대 컴퓨트 노드 터널에서 VNI 813으로 들어온 패킷은 drop"하는 `priority=30000` 규칙(쿠키 `0x330301`, cmp02 `in_port=5` / cmp01 `in_port=4`)이 추가되어, 노드를 건너는 패킷이 정상 디캡슐화 규칙(`priority=1 → resubmit(,4)`)에 도달하기 전에 버려진다.
- **실패 방향과 구간:** **양쪽 노드의 수신 방향.** 상대 노드에서 온 VXLAN 패킷이 **디캡슐화(table 4) 되기 전** br-tun table 0에서 멈춤. 송신·캡슐화·underlay는 양쪽 모두 정상 (cmp01 `Out` → cmp02 `In` 확인).
- **식별한 값:** VNI 813 (`tun_id=0x32d`), 터널 포트 cmp02 `in_port=5`(=`vxlan-c0a80015`, cmp01행) · cmp01 `in_port=4`(=`vxlan-c0a80016`, cmp02행), `priority=30000` > 정상 규칙 1, 두 규칙 모두 쿠키 `0x330301` (에이전트 쿠키 cmp02 `0x9c8f24fb32fbbef7` / cmp01 `0xb913967a8d099feb`와 다름) → **하나의 변경이 양쪽 노드에 대칭으로 들어간 것**.
- **구성 요소가 사라진 문제인가, 패킷 처리 규칙의 문제인가:** **패킷 처리 규칙의 문제.** 브리지·patch·vxlan 포트·VNI/VLAN 매핑·정상 flow는 모두 존재(③, table 4·20·22 정상)하고, 그 앞에 끼어든 높은 priority의 drop 규칙 하나가 원인. 에이전트가 관리하지 않는 쿠키의 규칙이라 에이전트는 계속 Alive로 보인다.
- **vm1 → vm3가 되는 것이 배제하는 구간:** 같은 노드 통신은 cmp01 br-int 안에서 끝나므로 vm1 자체, cmp01 br-int, 시큐리티 그룹, 서브넷/IP 설정을 배제한다. 남는 건 **br-tun → VXLAN → 상대 노드** 구간뿐.
- **복구하려면 (정상 구성은 건드리지 않고 원인만 되돌리는 방법):** (운영진, **cmp01·cmp02 양쪽**에서)
  ```bash
  # cmp02
  docker exec openvswitch_vswitchd ovs-ofctl --strict del-flows br-tun "table=0,priority=30000,tun_id=0x32d,in_port=5"
  # cmp01
  docker exec openvswitch_vswitchd ovs-ofctl --strict del-flows br-tun "table=0,priority=30000,tun_id=0x32d,in_port=4"
  ```
  - `--strict`로 **priority와 매칭 조건이 정확히 같은 규칙 하나만** 지운다 → 같은 table의 정상 규칙(`priority=1,in_port=4/5`)과 다른 참가자의 규칙(`0x330303`의 `tun_id=0x8a`, `dl_vlan=7/8`)은 그대로 남는다. (같은 효과로 `del-flows br-tun "cookie=0x330301/-1"`도 가능 — 쿠키가 우리 장애 규칙에만 쓰였음을 두 노드에서 확인)
  - 한쪽만 지우면 반쪽 복구: cmp02만 지우면 vm1 → vm2 request는 도착하지만 vm2의 reply가 cmp01에서 다시 버려져 ping은 여전히 실패한다.
- **복구 후 무엇으로 확인하면 되는지:**
  - 양쪽 노드에서 `dump-flows br-tun | grep 'cookie=0x330301'` 결과 없음
  - vm1 → vm2 ping 중 cmp02 `table=0, priority=1, in_port=5`와 `table=4, tun_id=0x32d`의 `n_packets`가 증가
  - 양쪽 노드 tcpdump에 `192.168.0.22 > 192.168.0.21` 방향 echo reply 등장, vm1 ↔ vm2 · vm3 ↔ vm2 양방향 ping 성공

## 5. SRE 세 질문

1. **사용자가 무엇을 겪었나:** 같은 프로젝트 안에서 vm2만 다른 VM(vm1·vm3)과 통신 불가. 그런데 vm2의 외부 접속(FIP)과 게이트웨이 통신은 되고, Horizon·에이전트는 모두 정상이라 "일부만 안 되는" 혼란스러운 상태.
2. **어떤 증상으로 알아챘나:** 다른 컴퓨트 노드에 있는 VM 사이의 ping 100% 손실(양쪽 방향 모두). 같은 노드 VM끼리는 정상.
3. **운영자가 미리 볼 수 있었던 지표는:**
   - **Black-box 프로브 (양방향):** 컴퓨트 노드 쌍마다 프로브 VM을 두고 **노드 간 VM↔VM ping을 양방향으로** 측정 (cmp01→cmp02, cmp02→cmp01). 현재 FIP 프로브는 net01 경유라 이번 장애를 못 잡는다.
   - **White-box 신호:** br-tun table 0에서 **에이전트 쿠키가 아닌 flow 탐지**(또는 `actions=drop` + 높은 priority 규칙의 `n_packets` 증가 알림), 터널 포트별 rx 대비 table 4 매칭 수 비교, `ovs-ofctl dump-flows`의 flow 수 변화 감시.
   - Grafana FIP 프로브: 확인하지 못함 (참고로 FIP 프로브는 net01 경유라, 확인했더라도 이번 장애는 정상으로 보였을 가능성이 높다)

## 6. 한 줄 회고 (선택)

- br-tun 구조를 보니 꽤 흐름이 스무스해졌다..
