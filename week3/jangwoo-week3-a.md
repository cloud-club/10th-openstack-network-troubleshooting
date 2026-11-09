# 3회차 제출 — Jangwoo (3A)

> 계정: `test` · 작성일: 2026-11-06
> 주제: 노드 사이를 건너면 — vm2만 통신이 안 된다
> 장애 유형: **3A — VXLAN 수신 DROP** (cc01·cc03 유형을 test 슬롯에 직접 주입해 관찰)
> 관찰 시각: 정상 기준선 11:15 · 3A 주입 11:17:04 ~ 복구 11:18:49 (KST) · 쿠키 `0x57170e57`

## 0. 내 환경 메모

| 항목 | 값 |
| --- | --- |
| 프로젝트 | `study-test` |
| 네트워크 | `test-net` · VXLAN · **VNI 84 (`0x54`)** · MTU 1450 |
| vm1 (비교, `cmp01`) | `10.99.0.117` · MAC `fa:16:3e:b4:a1:46` · FIP `203.0.113.245` · 포트 앞 11자리 `717baa6a-81` |
| vm3 (비교, `cmp01`) | `10.99.0.143` · MAC `fa:16:3e:e2:6c:b8` · FIP 없음 |
| vm2 (증상, `cmp02`) | `10.99.0.248` · MAC `fa:16:3e:05:8e:61` · FIP `203.0.113.220` · 포트 앞 11자리 `af37a45a-c6` |
| 라우터 | `test-router` · `qrouter-3b32d4d1-4d09-4f55-a852-76fb6b12bf05` · 게이트웨이 `10.99.0.1` |
| 언더레이 | net01 `192.168.0.12` · cmp01 `192.168.0.21` · cmp02 `192.168.0.22` |
| Local VLAN | cmp01 `qvo717baa6a-81` **tag 1** · cmp02 `qvoaf37a45a-c6` **tag 1** (이번엔 두 노드가 우연히 같았음. 보장되는 값 아님) |
| cmp02 `br-tun` 포트 번호 | `1(patch-int)` · `4(vxlan-c0a8000c → net01)` · `5(vxlan-c0a80015 → cmp01)` |
| cmp01 `br-tun` 포트 번호 | `1(patch-int)` · `2(vxlan-c0a8000c → net01)` · `4(vxlan-c0a80016 → cmp02)` |

## 1. 읽기 핵심

### 1.0 기초 다지기

#### 네트워크 기초 용어

| 용어 | 뜻 |
| --- | --- |
| **L2 / L3** | L2(데이터 링크 계층)는 같은 네트워크 안에서 **MAC 주소**로 프레임을 전달하는 단계, L3(네트워크 계층)는 서로 다른 네트워크 사이를 **IP 주소**로 라우팅하는 단계다. 스위치는 L2, 라우터는 L3 장비다. |
| **MAC 주소** | NIC마다 붙는 48비트 하드웨어 주소(`fa:16:3e:…`). 같은 L2 구간 안에서 "누구에게 보낼지"를 정한다. |
| **IP 주소 · 서브넷** | IP는 L3 주소, 서브넷은 그 주소를 묶은 범위(`10.99.0.0/24`)다. 목적지가 같은 서브넷이면 L2로 바로 보내고, 다르면 게이트웨이로 보낸다. |
| **게이트웨이** | 다른 서브넷으로 나가는 출구 라우터 주소. 서브넷 밖이 목적지면 패킷을 일단 게이트웨이의 MAC으로 보낸다. |
| **브로드캐스트 도메인** | 브로드캐스트 프레임(`ff:ff:ff:ff:ff:ff`)이 닿는 범위 = 하나의 L2 구간. VLAN 하나, Neutron 네트워크 하나가 각각 브로드캐스트 도메인 하나다. |
| **ARP** | IP로 MAC을 알아내는 프로토콜. "이 IP 가진 사람?"(Request, 보통 브로드캐스트)에 주인이 "내 MAC은 이거"(Reply)로 답한다. |
| **ARP 캐시** | 알아낸 IP→MAC 짝을 잠시 저장한 표. 오래되면 다시 확인하는데, 이때는 브로드캐스트가 아니라 알고 있던 MAC으로 **유니캐스트 ARP**를 보내기도 한다. |
| **프레임 / 패킷** | L2 단위(이더넷 헤더 포함)를 프레임, L3 단위(IP 헤더부터)를 패킷이라 부른다. |
| **VLAN 태그** | 이더넷 프레임에 붙이는 12비트 번호(802.1Q). 같은 스위치에서 L2 구간을 나눈다. |

#### NAT · MASQUERADE 맛보기 (자세한 동작은 5회차)

| 용어 | 뜻 |
| --- | --- |
| **NAT** | 패킷이 지나갈 때 IP 주소(때로는 포트)를 바꿔 쓰는 것. 사설 주소를 쓰는 내부망이 외부와 통신하려고 쓴다. |
| **SNAT** | **출발지** 주소를 바꾼다. 내부 VM이 밖으로 나갈 때 라우터의 외부 주소로 바꿔 내보내는 방식. |
| **DNAT** | **목적지** 주소를 바꾼다. 외부에서 공인 주소로 들어온 요청을 내부 VM 주소로 바꿔 넣어주는 방식. |
| **MASQUERADE** | SNAT의 한 종류. 바꿀 주소를 고정값으로 적지 않고 **나가는 인터페이스의 현재 주소**로 자동으로 쓴다. 주소가 바뀔 수 있는 출구(가정용 공유기, 실습 환경의 외부 veth)에 쓴다. |
| **Floating IP (FIP)** | Neutron이 VM 하나에 1:1로 붙여주는 외부 주소. 들어올 때는 DNAT, 나갈 때는 SNAT가 이 주소로 짝지어 걸린다. 처리는 라우터 네임스페이스(qrouter)에서 한다. |

#### 캡슐화 용어 (이번 주 범위에서 필요한 만큼만)

| 용어 | 뜻 |
| --- | --- |
| **언더레이 / 오버레이** | 언더레이는 실제 노드끼리 연결된 물리(또는 호스트) 네트워크(`192.168.0.0/24`), 오버레이는 그 위에 터널로 만든 가상 네트워크(`10.99.0.0/24`)다. |
| **VXLAN** | L2 프레임 전체를 UDP(목적지 포트 **4789**)에 담아 언더레이로 보내는 터널 방식. 바깥 IP 헤더에는 노드 주소, 안쪽에는 VM 주소가 들어간다. |
| **VNI** | VXLAN 헤더에 들어가는 24비트 네트워크 번호. 노드 사이 터널에서 "어느 Neutron 네트워크의 프레임인지"를 구분한다. Neutron에서는 `provider:segmentation_id`, OVS flow에서는 `tun_id`로 보인다. |
| **Local VLAN** | 노드 안 `br-int`에서만 쓰는 VLAN 번호. 같은 Neutron 네트워크라도 노드마다 따로 정해지고 서로 상관없다. 터널로 나갈 때 VNI로 바뀌고 들어올 때 다시 Local VLAN으로 바뀐다. |

#### OVS flow 읽는 법

flow 한 줄은 이렇게 생겼다.

```text
cookie=0x9c8f…, table=0, n_packets=741, priority=1,in_port=5 actions=resubmit(,4)
└ 누가 넣었나  └ 몇 번 표  └ 몇 개 처리했나 └ 우선순위 └ 조건(match)  └ 할 일(actions)
```

| 항목 | 뜻 |
| --- | --- |
| **table** | flow를 묶은 표. 패킷은 **table 0**에서 시작해 `resubmit(,N)` / `goto_table:N`으로 다음 표로 넘어간다. |
| **priority** | 한 표 안에서 조건에 맞는 flow가 여러 개면 **priority가 가장 높은 하나만** 실행된다. 숫자가 클수록 먼저다. 줄 순서와는 상관없다. |
| **match** | 적용 조건. 자주 보는 것: `in_port`(들어온 포트 번호), `dl_vlan`(VLAN 태그), `dl_dst`/`dl_src`(목적지/출발지 MAC), `tun_id`(VNI), `arp`. 조건이 없으면 전부에 맞는다. |
| **actions** | 맞았을 때 할 일. `resubmit`·`goto_table`(다음 표로), `mod_vlan_vid:N`(VLAN 태그 붙이기/바꾸기), `strip_vlan`(태그 떼기), `load:…->NXM_NX_TUN_ID[]`(VNI 쓰기), `output:N`(포트로 내보내기), `learn`(MAC 학습 flow 만들기), `NORMAL`(일반 학습 스위치처럼 동작), `drop`(버림). |
| **cookie** | flow를 넣은 주체가 붙이는 꼬리표. 같은 쿠키끼리 골라 보거나 지울 수 있다(`dump-flows br-tun cookie=0x…/-1`). |
| **n_packets / n_bytes** | 그 flow가 지금까지 처리한 패킷 수. **같은 명령을 테스트 전후로 두 번** 실행해 늘어난 flow가 실제로 패킷이 지나간 길이다. |

현재 Neutron OVS 에이전트가 `br-tun`에 만드는 표는 대략 이렇게 나뉜다.

| table | 역할 |
| --- | --- |
| 0 | 입구 분기. `patch-int`에서 온 것(노드에서 나가는 것)은 table 2, `vxlan-*`에서 온 것(터널로 들어온 것)은 table 4. 나머지는 drop |
| 2 | 나가는 프레임 분류. 유니캐스트는 20, 브로드캐스트·멀티캐스트는 22, 브로드캐스트 ARP는 21 |
| 4 | 들어온 프레임의 `tun_id`(VNI)를 Local VLAN으로 바꿔 붙이고 10으로 |
| 10 | 출발지 MAC을 학습(`learn`)한 뒤 `patch-int`로 내보냄 → `br-int` |
| 20 | 목적지 MAC별로 어느 터널 포트로 보낼지. Local VLAN을 떼고 VNI를 써서 `output` |
| 21 | ARP responder. 알고 있는 IP의 ARP 요청에 터널로 보내지 않고 직접 답함 (l2population) |
| 22 | 브로드캐스트를 그 네트워크가 있는 모든 터널 포트로 복제해 보냄 |

조회 도구:

| 명령 | 보는 것 |
| --- | --- |
| `ovs-vsctl show` | 브리지·포트 구성과 `tag`(Local VLAN). "무엇이 꽂혀 있나" |
| `ovs-ofctl show <br>` | 포트 **번호**와 이름의 짝. flow의 `in_port=5`가 어느 포트인지 여기서 찾는다 |
| `ovs-ofctl dump-flows <br>` | flow 전체와 카운터. "패킷을 어떻게 처리하나" |
| `ovs-appctl ofproto/trace <br> '<조건>'` | 가상의 패킷 하나가 어느 표·어느 flow를 거쳐 어떻게 끝나는지 시뮬레이션. 실제 패킷은 보내지 않는다 |

### 1.1 Learning OpenStack Networking 4장 (PDF p.102–112)

- **OVS 세 구성요소**: 커널 모듈(실제 패킷 처리 = 데이터 플레인), `ovs-vswitchd`(커널 모듈을 어떻게 프로그래밍할지 정하는 데몬), OVSDB(설정 저장). Neutron은 `ovs-vsctl`(DB)과 `ovs-ofctl`(flow)로 OVS를 다룬다.
- **장치 다섯 종류**: tap, Linux bridge, veth, OVS 브리지, OVS 패치 포트. 패치 포트는 **OVS 브리지끼리**, veth는 **OVS ↔ Linux bridge**를 잇는다. 우리 환경의 `patch-int` ↔ `patch-tun`이 패치 포트다.
- **노드 안 경로**: 책은 provider 브리지 기준(`tap → qbr → qvb/qvo → br-int → int-br-ethX/phy-br-ethX → br-ethX → ethX`)으로 9개 장치를 센다. 우리 환경(VXLAN)은 provider 브리지 자리에 터널 브리지가 들어가 `… → br-int → patch-tun/patch-int → br-tun → vxlan-* → eth0`이 된다.
- **포트 식별**: `ovs-ofctl show <브리지>`로 포트 번호를 본다. flow 조건의 `in_port=N`은 이 번호다.
- **Local VLAN**: `br-int`에 꽂힌 포트는 노드마다 독립적인 VLAN에 들어간다. 각 노드의 OVSDB가 **따로** 관리하고, 같은 노드·같은 네트워크면 같은 번호를 받는다. 번호는 임의로 정해지고 재시작하면 바뀔 수 있다. `ovs-vsctl show`의 `tag`로 확인한다.
- **flow가 실제 처리를 정한다**: OVS 에이전트는 Neutron DB의 네트워크 정보를 flow로 바꿔 넣고 계속 유지한다. flow는 태그를 바꾸거나 붙이거나 떼고, 특정 조건이면 **drop**할 수도 있다.
- **VLAN 매핑 예**: 들어올 때 `in_port=2,dl_vlan=30 → mod_vlan_vid:1`(물리 VLAN 30 → Local VLAN 1), 나갈 때 `dl_vlan=1 → mod_vlan_vid:30`. 조건에 안 맞는 나머지는 `priority=2,in_port=2 actions=drop`으로 버린다. 우리 환경에서는 같은 역할을 `br-tun`이 VNI로 한다(`tun_id=0x54 → mod_vlan_vid:1`, `dl_vlan=1 → load:0x54->TUN_ID`).
- **NORMAL**: 학습 스위치처럼 동작하라는 action. 목적지를 모르면 모든 포트로 내보내고 MAC 표(FDB)를 채운다. `br-int`는 대부분 이걸로 전달하고, VLAN 태그는 flow가 아니라 포트의 `tag`로 붙는다.
- **노드에 flow가 없을 수도 있다**: 그 노드에 해당 네트워크의 인스턴스·자원이 없으면 그 네트워크용 flow도 없다.
- **1판과 다른 점**: 책은 flow를 "위에서 아래로 처리한다"고 설명하지만, 실제로는 한 표 안에서 **priority가 가장 높은 맞는 flow**가 실행된다. 책은 provider 브리지(VLAN·flat)만 다루고 `br-tun`의 VXLAN 표(0/2/4/10/20/21/22)는 다루지 않으므로 1.0의 표와 실제 `dump-flows`로 보강했다.

### 1.2 SRE 6장 (PDF p.84–88)

- **정의**: 모니터링은 시스템의 실시간 수치를 모으고 보여주는 것이다. **화이트박스**는 내부가 내보내는 지표(로그, 내부 통계)를 보는 것, **블랙박스**는 사용자가 보는 것처럼 바깥에서 동작을 시험하는 것이다. **근본 원인**은 고치면 같은 방식으로 다시 일어나지 않으리라 확신할 수 있는 결함을 말한다.
- **증상 vs 원인**: 모니터링은 "무엇이 고장 났나(증상)"와 "왜(원인, 중간 원인일 수도)"에 답해야 한다. 이 둘을 구분하는 것이 신호는 높이고 소음은 줄이는 핵심이다.
- **블랙박스 vs 화이트박스**: 블랙박스는 증상 중심이라 **지금 실제로 일어나는** 문제를 잡는다. 화이트박스는 내부를 들여다볼 수 있어서 곧 터질 문제나 재시도에 가려진 실패까지 잡는다.
- **누군가의 증상은 다른 누군가의 원인**이다. 층이 여러 개인 시스템에서는 같은 지표가 보는 사람에 따라 증상도 되고 원인도 된다.
- **호출(paging)은 블랙박스로**: 이미 진행 중이고 실제 증상을 만드는 문제일 때만 사람을 깨우게 해준다. 대신 아직 안 일어난 문제에는 블랙박스가 거의 쓸모없다.

### 1.3 보강 — Neutron 2026.1 OVS Self-service networks, East-west scenario 1

같은 네트워크의 인스턴스가 서로 다른 컴퓨트 노드에 있으면, 라우터(네트워크 노드)를 거치지 않고 **컴퓨트 노드끼리 직접** 통신한다.

**컴퓨트 노드 간 패킷 경로** (문서 단계를 우리 장치 이름으로 옮김)

| 노드 | 단계 | 무엇을 하나 | 우리 환경 |
| --- | --- | --- | --- |
| 보내는 노드 | 1~3 | 인스턴스 → veth → 보안 그룹 브리지(SG 규칙·연결 추적) → `br-int` | `tap → qbr → qvb/qvo → br-int` |
| | 4 | `br-int`가 **내부 VLAN 태그**(Local VLAN)를 붙인다 | `qvo` 포트의 `tag: 1` |
| | 5~6 | 내부 VLAN 태그를 **내부 터널 ID**로 바꾸고, 패치 포트로 `br-tun`에 넘긴다 | `patch-tun → patch-int` |
| | 7 | `br-tun`이 VNI를 붙여 **캡슐화** | table 20/22: `strip_vlan, load:0x54->TUN_ID, output:<vxlan>` |
| | 8 | 오버레이용 물리 인터페이스가 상대 노드로 보낸다 | `eth0`, `192.168.0.21 → .22` UDP 4789 |
| 받는 노드 | 1 | 물리 인터페이스 → `br-tun` | `eth0 → vxlan-c0a80015` |
| | 2~3 | `br-tun`이 **디캡슐화**하고 터널 ID를 내부 VLAN 태그로 바꾼다 | table 0 → 4: `tun_id=0x54 → mod_vlan_vid:1` |
| | 4 | 패치 포트로 `br-int`에 넘긴다 | table 10 → `output:1(patch-int)` |
| | 5 | `br-int`가 내부 VLAN 태그를 뗀다 | `qvo` 포트(tag 1)로 나가며 태그 제거 |
| | 6~8 | 보안 그룹 브리지 → veth → 인스턴스 | `qvo/qvb → qbr → tap → vm` |

- 문서는 "VLAN 태그를 터널 ID로 바꾸는" 단계를 integration bridge(`br-int`) 설명 안에 적어 두었다. 하지만 실제 `dump-flows`를 보면 이 교환은 **`br-tun`의 flow**(table 20/22, table 4)가 한다. 문서 문장과 실제 구현 위치가 다를 수 있으니, 어느 브리지에서 일어나는지는 flow로 확인한다.
- 그래서 east-west 장애는 **송신 노드의 캡슐화 전**(`br-int` ~ `br-tun` table 0/2/20), **언더레이**, **수신 노드의 디캡슐화 후**(`br-tun` table 0/4/10 ~ `br-int`) 세 구간으로 나눠 볼 수 있다.

**VXLAN 헤더 구조** (바깥에서 안쪽 순서)

```text
┌ 바깥 이더넷  : 노드 NIC MAC
├ 바깥 IP      : 출발 노드 IP → 도착 노드 IP        (언더레이 주소, 예: 192.168.0.21 → 192.168.0.22)
├ 바깥 UDP     : 출발 포트(흐름마다 다름) → 4789
├ VXLAN 헤더   : 8바이트 = 플래그(I 비트 = "VNI 유효") + 24비트 VNI + 예약 필드
└ 안쪽 프레임  : 원래 VM의 이더넷 프레임 그대로 (VM MAC, VM IP, ICMP/TCP …)
```

- `tcpdump`는 이 구조를 `IP 192.168.0.21.54503 > 192.168.0.22.4789: VXLAN, flags [I] (0x08), vni 84` 다음 줄에 안쪽 패킷을 찍는 식으로 보여준다.
- 바깥 UDP 출발 포트는 안쪽 흐름에 따라 달라진다. 그래서 같은 VM 쌍이라도 ICMP와 ARP의 출발 포트가 다를 수 있다.
- VNI는 UDP 페이로드 4~6바이트 위치에 있어서, 캡처할 때 `'udp port 4789 and (udp[12:4] >> 8) = <VNI>'`로 내 네트워크만 골라낼 수 있다.
- 캡슐화로 늘어난 헤더 크기가 MTU에 주는 영향은 4회차에서 다룬다.

## 2. 받은 증상

| 확인한 것 | 명령 | 결과 |
| --- | --- | --- |
| vm1 → vm3 (같은 노드) | (vm1) `ping -c 3 10.99.0.143` | **3/3 응답**, 0.4~1.1ms |
| vm1 → vm2 (다른 노드) | (vm1) `ping -c 5 10.99.0.248` | **5/5 손실** |
| vm2 → vm1 (반대 방향) | (vm2, FIP로 접속) `ping -c 5 10.99.0.117` | **5/5 손실** |
| qrouter → vm2 (net01 경유) | (net01) `ip netns exec qrouter-3b32… ping -c 5 10.99.0.248` | **5/5 응답** |
| net01 → vm2 FIP | (net01) `ping -c 3 203.0.113.220` | **3/3 응답** |
| ops01 → vm2 SSH | `ssh -i ~/test-key.pem cirros@203.0.113.220` | **접속 성공** |
| Nova 상태 | `openstack server list --long` | vm1·vm2·vm3 모두 `ACTIVE` |
| 에이전트 상태 | `openstack network agent list` | net01·cmp01·cmp02 전부 `:-)` `UP` |

> 정리: vm2는 살아 있고 FIP·SSH도 된다. 안 되는 것은 **cmp01에 있는 VM과 cmp02의 vm2 사이, 양방향 모두**다. 같은 vm2라도 net01을 거치는 길은 멀쩡하다. 그래서 vm2 자체나 라우터가 아니라 **cmp01 ↔ cmp02 노드 간 경로**를 의심했다.

## 3. 가설 · 검증 기록 ★

경로(vm1 → vm2): **vm1 → tap/qbr/qvb/qvo(cmp01) → br-int → patch-tun/patch-int → br-tun(캡슐화) → eth0 → 언더레이 → eth0(cmp02) → br-tun(디캡슐화) → patch-int/patch-tun → br-int → qvo/qvb/qbr/tap → vm2**

| # | 가설 (어디가 문제일 것 같다) | 확인 방법 (명령) | 결과 | 판단 |
| --- | --- | --- | --- | --- |
| ① | (기준 정보) 배치와 VNI가 내 메모와 다르다 | `openstack server list --long` · `openstack port list --server test-vm2` · `openstack network show test-net -c provider:segmentation_id` | vm1·vm3 → cmp01, vm2 → cmp02, VNI **84** | 이후 캡처·flow를 VNI 84로 거름 |
| ② | vm2 자체(OS, 포트, SG)가 문제다 | qrouter → vm2 ping · net01 → vm2 FIP ping · ops01 → vm2 SSH | 셋 다 **성공**. cmp02 언더레이에 net01발 요청 `In`과 vm2 응답 `Out`이 짝지어 보임 | **기각** — vm2와 cmp02 내부의 vm2 쪽 경로는 정상 |
| ③ | vm1 쪽(cmp01의 tap~br-int)이 문제다 | vm1 → vm3 ping (같은 노드, 같은 Local VLAN 1) | **3/3 응답** | **기각** — vm1에서 cmp01 `br-int`까지는 정상. 이 비교로 **cmp01 노드 안 구간이 배제**되고 남은 건 "노드를 건너는 구간"(br-tun, 언더레이, 상대 노드) |
| ④ | 노드 간 터널 포트가 없거나 언더레이에서 UDP 4789가 막혔다 | cmp01·cmp02 `ovs-vsctl show`(vxlan 포트) · vm1 → vm2 ping 중 cmp02 `tcpdump -ni any 'udp port 4789 and (udp[12:4]>>8)=84'` | 양쪽 `br-tun`에 상대 노드 vxlan 포트 존재. cmp02 `eth0 In`으로 **VNI 84 프레임 8개 도착** (ICMP 요청 5 + ARP 요청 3), `Out`은 **0개** | **기각** — vm1의 요청은 캡슐화돼 cmp02까지 **도착함**. 송신 노드의 캡슐화와 언더레이는 정상 |
| ⑤ | cmp02에 도착한 뒤 vm2로 넘어가지 못한다 (수신 쪽) | 같은 시각 cmp02 `tcpdump -ni qvoaf37a45a-c6 'icmp or arp'` | **0개** (정상 기준선에서는 같은 테스트에 14개) | **유지** — 끊긴 곳은 cmp02의 `eth0` 이후 ~ `qvo` 이전 = `br-tun → patch → br-int` 구간 |
| ⑥ | 그 구간의 **구성 요소**가 빠졌다 (패치 포트, qvo, tag) | cmp02 `ovs-vsctl show` | `patch-int`(peer=patch-tun)·`patch-tun`(peer=patch-int) 있음, `qvoaf37a45a-c6` **tag 1**로 `br-int`에 있음. 에이전트도 전부 Alive | **기각** — 브리지·포트는 정상 상태 그대로 |
| ⑦ | 구성은 그대로인데 **flow 규칙**이 패킷을 버린다 | cmp02 `ovs-ofctl dump-flows br-tun`을 ping 전후로 두 번 · `ovs-ofctl show br-tun`으로 `in_port` 번호 대조 | table 0에 `cookie=0x57170e57, priority=30000, tun_id=0x54, in_port=5 actions=drop`. `in_port=5` = `vxlan-c0a80015`(cmp01에서 오는 터널). **n_packets 0 → 8** — ④에서 도착한 8개와 정확히 같음 | **유지 → 원인 확정 (수신)** |
| ⑧ | 반대 방향(vm2 → vm1)도 같은 원인이다 | vm2에서 vm1 ping 중 cmp01 `tcpdump`(VNI 84) · cmp01 `dump-flows br-tun` 전후 | cmp01 `eth0 In` **8개 도착**(ICMP 요청 5 + ARP 3). cmp01 table 0에 `priority=30000, tun_id=0x54, in_port=4 actions=drop`(`in_port=4` = cmp02에서 오는 터널), **n_packets 0 → 8** | **유지** — 양쪽 노드 모두 **상대 컴퓨트 노드 터널로 들어온 VNI 84**를 버림 |
| ⑨ | 송신 쪽에도 문제가 있을 수 있다 | cmp02 `ovs-appctl ofproto/trace br-tun` 세 가지 | cmp02 수신(`in_port=5,tun_id=0x54`) → **table 0에서 drop**. cmp02 송신(`in_port=1,dl_vlan=1` → vm1) → table 0 → 2 → 20 → `output:5`(**정상 캡슐화**). cmp01 수신(`in_port=4,tun_id=0x54`) → **table 0에서 drop** | **기각** — 송신 쪽은 정상. 문제는 수신 쪽 table 0 하나 |

헛짚을 뻔한 지점:

- **다른 drop 규칙이 먼저 눈에 들어옴**: `grep priority=30000`을 하면 정상 기준선 단계부터 `cookie=0x330301`(`tun_id=0x32d`), `0x330303`(`tun_id=0x8a`), `0x330302`·`0x330304`(`dl_vlan=7`·`8`)가 보인다. 전부 VNI나 Local VLAN이 내 것(`0x54`, `1`)과 달라서 다른 슬롯 규칙으로 보고 제외했다. **`tun_id`와 `in_port`를 내 값과 대조하지 않았으면 남의 장애를 내 원인으로 적을 뻔했다.**
- **8개 중 3개는 ping이 아님**: cmp02 도착 8개 중 3개는 `ARP, Request who-has 10.99.0.248 tell 10.99.0.117`였다. ping이 실패하는 동안 vm1이 vm2의 MAC을 **유니캐스트 ARP**로 다시 확인한 것이다. 유니캐스트라 cmp01 `br-tun`의 ARP responder(table 21, 브로드캐스트 ARP만 처리)를 거치지 않고 table 20을 거쳐 터널로 나갔다. 그래서 drop 카운터도 ICMP 5가 아니라 8이다.
- **`br-int` flow에서 Local VLAN을 찾으려다 빈손**: cmp02 `dump-flows br-int`에서 `dl_vlan=1`을 grep하면 아무것도 안 나온다. `br-int`는 table 60에서 `NORMAL`로 전달하고, Local VLAN은 flow가 아니라 **qvo 포트의 `tag`**로 붙기 때문이다. Local VLAN 근거는 `ovs-vsctl show`의 `tag: 1`로 잡았다.

### 패킷 관찰 결과 (vm1 → vm2, 3A)

```text
cmp02 eth0  In   192.168.0.21 > 192.168.0.22.4789  VXLAN vni 84  10.99.0.117 > 10.99.0.248  ICMP echo request seq 1~5
cmp02 eth0  In   192.168.0.21 > 192.168.0.22.4789  VXLAN vni 84  ARP who-has 10.99.0.248 tell 10.99.0.117  ×3
cmp02 eth0  Out  (없음)
cmp02 qvoaf37a45a-c6     0 packets captured
cmp02 br-tun table=0  cookie=0x57170e57 priority=30000,tun_id=0x54,in_port=5 actions=drop   n_packets 0 → 8
```

언더레이에서는 보이고 qvo에서는 안 보인다. 그 사이에서 카운터가 정확히 같은 수만큼 오른 drop 규칙이 있다. 세 증거가 모두 cmp02 `br-tun` table 0을 가리킨다.

### 정상 / 3A 비교

| 확인 항목 | 정상 기준선 (11:15) | 3A (11:17~11:18) |
| --- | --- | --- |
| vm1 → vm3 | 성공 | 성공 |
| vm1 → vm2 | 5/5 응답 | 5/5 손실 |
| cmp02 언더레이 (VNI 84, vm1 → vm2 중) | `In` 7 / `Out` 7 (요청·응답 짝) | `In` 8 / `Out` 0 |
| cmp02 `qvo` 캡처 | 14개 (요청·응답·ARP) | 0개 |
| qrouter → vm2 · vm2 FIP | 성공 | 성공 |
| `br-tun` table 0 `priority=30000`, `tun_id=0x54` | 없음 | cmp02 `in_port=5`, cmp01 `in_port=4`에 drop |
| `agent list` / `ovs-vsctl show` | 정상 | 정상 (변화 없음) |

## 4. 원인 · 복구 방안

복구는 과제 규칙상 학생 입장에서는 **실행하지 않는다.** (이번 관찰은 운영진 권한으로 test 슬롯에 직접 주입·복구했고, 아래 '복구 후 확인'은 실제 결과다.)

- **원인 (한 문장):** cmp02와 cmp01의 `br-tun` table 0에, 상대 컴퓨트 노드 터널 포트(cmp02 `in_port=5`, cmp01 `in_port=4`)로 들어온 VNI 84(`tun_id=0x54`) 프레임을 버리는 `priority=30000` drop flow가 추가돼 있다. 이 flow가 정상 경로(`priority=1,in_port=… → resubmit(,4)`)보다 먼저 선택되어, 노드 사이를 건너온 vm1↔vm2 프레임이 디캡슐화 단계에서 버려진다.
- **끊긴 지점이 이 증상으로 보인 이유:**
  - **수신 쪽 drop**이라 보내는 쪽은 정상이다. 요청은 캡슐화되어 상대 노드 `eth0`까지 도착하지만 `br-tun`에서 사라진다. 그래서 양쪽 언더레이에 `In`만 있고 `Out`이 없다.
  - 조건이 `in_port=<상대 컴퓨트 터널>`이라 **net01에서 오는 터널(cmp02 `in_port=4`)은 해당하지 않는다.** 그래서 qrouter → vm2, FIP ping, SSH는 그대로 된다. "vm2만 안 된다"가 아니라 "**cmp01 ↔ cmp02 사이만** 안 된다"가 정확한 범위다.
  - 같은 노드 통신(vm1 → vm3)은 `br-int` 안에서 끝나 `br-tun`에 가지 않으므로 영향이 없다.
  - 브리지·포트·패치·tag는 그대로이고 정상 flow도 지워지지 않았다. **구성 요소가 사라진 문제가 아니라 패킷 처리 규칙의 문제**다. 그래서 `agent list`는 Alive, `ovs-vsctl show`도 정상으로 보인다.
  - 이 장애를 식별한 값은 **VNI(`tun_id=0x54`)와 터널 `in_port`**다. Local VLAN은 이 규칙의 조건에 쓰이지 않았다.
- **복구하려면 (명령 또는 절차):** (운영진, cmp01·cmp02 각각)
  - `sudo docker exec openvswitch_vswitchd ovs-ofctl del-flows br-tun cookie=0x57170e57/-1`
  - 쿠키로 **이 drop flow만** 지우고, 그 아래 있던 정상 flow는 건드리지 않는다. 정상 구성을 바꾸지 않고 원인만 되돌리는 방법이다.
  - 피할 것: `neutron_openvswitch_agent` 재시작. 에이전트가 resync하면서 자기 쿠키가 아닌 flow를 정리할 수 있는데, 그러면 같은 노드에 걸린 **다른 슬롯의 drop flow까지** 사라질 수 있다. 영향 범위가 내 장애보다 커진다.
- **복구 후 무엇으로 확인하면 되는지:**
  - 양쪽 노드 `dump-flows br-tun cookie=0x57170e57/-1`이 비어 있는지 → **실제로 비어 있음**
  - vm1 → vm2 ping → **3/3 응답** (0.76~3.85ms)
  - 반대 방향 vm2 → vm1 ping, cmp02 언더레이에 `Out`이 다시 찍히는지
  - `scripts/60-verify-slot.sh test` → **전 항목 OK** (FIP SSH, vm1→vm2 노드 간, vm1→vm3, SNAT, MTU 1400, 하이브리드 장치, 네임스페이스, SG 체인, VXLAN 터널 포트)

## 5. SRE 세 질문

1. **사용자가 무엇을 겪었나:** 같은 프로젝트에서 vm1과 vm3는 서로 되는데 다른 노드의 vm2와는 양방향 모두 통신이 안 됐다. vm2에 FIP로 SSH 접속은 되니 "vm2는 살아 있는데 왜 내부끼리만 안 되지?"가 된다. Horizon은 ACTIVE, 에이전트는 Alive라 화면상으로는 아무 문제가 없다.
2. **어떤 증상으로 알아챘나:** 노드를 건너는 내부 통신만 100% 손실이고, 같은 노드와 net01 경유는 정상이었다. 이 차이가 범위를 "cmp01 ↔ cmp02 east-west"로 좁혔다.
3. **운영자가 미리 볼 수 있었던 지표는:**
   - **Black-box**: 컴퓨트 노드 쌍마다 **양방향** east-west 프로브(`cmp01 VM → cmp02 VM`, `cmp02 VM → cmp01 VM`). 이번 장애는 **FIP 프로브로는 잡히지 않는다.** FIP 트래픽은 net01 터널로 들어오고, drop 조건은 컴퓨트 노드 간 터널만이었기 때문이다. 사용자 경로마다 프로브가 따로 있어야 한다.
   - **White-box**: `br-tun` table 0에서 Neutron 에이전트 쿠키가 아닌 flow나 `actions=drop` flow의 `n_packets` 증가, VNI별 터널 수신(`In`) 대비 송신(`Out`) 비율 급변.
   - **Alive ≠ 정상**: `agent list` Alive는 에이전트 프로세스가 RPC로 보고한다는 뜻일 뿐, 데이터 플레인이 패킷을 전달한다는 뜻이 아니다. SRE 6장 식으로 보면 Alive는 화이트박스 지표인데, 이번 장애의 증상과도 원인과도 닿아 있지 않다.
   - (Grafana 프로브 기록은 cc01~cc05·test 대시보드를 새로 구성한 뒤 보강 예정)

## 6. 한 줄 회고 (선택)

- "브리지와 포트가 다 있다"와 "패킷이 지나간다"는 다른 말이었다. 언더레이 캡처, qvo 캡처, drop 카운터의 숫자가 맞아떨어질 때 비로소 확정할 수 있었다.