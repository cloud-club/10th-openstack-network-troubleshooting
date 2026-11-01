# 1회차 스터디

- Learning OpenStack Networking (Denton, 1판): 1장 p.27–38 / 3장 p.70–73, p.80–89 (에이전트 역할만)
- Site Reliability Engineering: 6장 Monitoring Distributed Systems p.81–92

---

## 1. Denton 1장 - OpenStack Networking 개요

**Neutron이란**

- 독립 설치 가능한 서비스. `neutron-server`가 API를 노출하고, 요청을 설정된 **플러그인**으로 넘겨 처리
- 네트워크 구성을 영구 저장하기 위해 DB 필요
- 여러 호스트에 분산 가능(복원력·이중화) / 단일 노드에도 가능

**제공 기능**

| 기능 | 구현 방식 |
| --- | --- |
| Switching (L2) | Linux bridge, Open vSwitch(OVS). VLAN 태깅, GRE/VXLAN 오버레이 |
| Routing (L3) | IP forwarding + iptables + **네트워크 네임스페이스** (chroot의 네트워크판 → 테넌트 간 서브넷 겹침 허용) |
| Load balancing | LBaaS, HAProxy 기반 |
| Firewalling | 시큐리티 그룹(컴퓨트 노드 iptables) / FWaaS(라우터 단, 실험적) |
| VPN | VPNaaS, IPSec (실험적) |

**트래픽 4종류**

| 종류 | 용도 | 비고 |
| --- | --- | --- |
| Management | 호스트 간 내부 통신 (메시징, DB) | 모든 호스트가 이 망으로 서로 통신 |
| API | Keystone·Neutron·Glance·Horizon 엔드포인트 노출 | 보통 Management와 같은 IP/인터페이스 |
| External | Neutron 라우터의 외부 접근 → **Floating IP, LB VIP의 출처** | 인터넷에서 도달 가능해야 함 |
| Guest | 인스턴스 트래픽 전용 | local / flat / VLAN / GRE·VXLAN |
- 4종류가 각각 전용 인터페이스를 요구하진 않음 — 보통 하나의 인터페이스에 합쳐짐

**물리 서버 연결**

- 단일 인터페이스: 제어·데이터 플레인 통합. 게스트가 대역폭을 다 쓰면 호스트 관리 불가(DoS) → 테스트/PoC 전용
- 다중 인터페이스: 최소 2개 — ① Mgmt+API ② External+Guest. 프로덕션 권장
- 본딩: 링크 집선(대역폭↑) 또는 active/passive(이중화). 물리 스위치 지원 필요

**노드 간 서비스 분리 (3가지 배치 모델)**

| 모델 | 특징 |
| --- | --- |
| 컨트롤러 1 + 컴퓨트 N (L3 agent 없음) | Neutron은 L2만 제공, 라우팅은 외부 라우터 |
| 컨트롤러 1 + 컴퓨트 N (L3 agent 있음) | 소프트웨어 라우터가 컨트롤러에 상주 |
| 컨트롤러 1 + **네트워크 노드** 1 + 컴퓨트 N | L3·DHCP·metadata 에이전트를 네트워크 노드로 분리. `neutron-server`(API)는 컨트롤러에 남음. 컨트롤러의 자원 포화 위험↓ |

---

## 2. Denton 3장 — Basic Neutron constructs

**Neutron API 핵심 리소스**

| 리소스 | 정의 |
| --- | --- |
| Network | 격리된 L2 브로드캐스트 도메인. API의 중심 — subnet·port는 반드시 network에 속함 |
| Subnet | IPv4/IPv6 주소 블록(CIDR). 한 network에 여러 subnet 가능, DHCP 할당 범위 지정 가능 |
| Port | 논리 가상 스위치의 포트. VM 인터페이스가 매핑되고 **MAC·IP가 여기서 정해짐**. DB에 저장 → 플러그인 에이전트가 이걸 보고 가상 스위칭 구성 |

**네트워크 네임스페이스로 겹치는 네트워크 지원**

- 네임스페이스 = 네트워크 스택의 논리 복사본(자체 라우팅 테이블·iptables·인터페이스)
- 테넌트가 만드는 network / router / LB마다 네임스페이스 하나
- 명명 규칙
    - `qdhcp-<network UUID>` — 해당 네트워크의 DHCP 서비스
    - `qrouter-<router UUID>` — 라우터, 연결된 서브넷의 트래픽 라우팅
    - `qlbaas-<LB UUID>` — HAProxy 등 로드밸런서
- 보통 컨트롤러 또는 네트워크 노드에만 존재
- 확인: `ip netns` / 실행: `ip netns exec <NAME> <command>` (ip, route, iptables …)

**플러그인으로 기능 확장**

- nova-network(Linux bridge + VLAN + iptables) → Neutron은 서드파티 플러그인 지원
- LinuxBridge / Open vSwitch = **모노리식** 플러그인(동시에 하나만). Icehouse부터 **ML2**로 대체(여러 L2 메커니즘 동시 사용)

---

## 3. Denton 3장 — 에이전트별 역할

| 컴포넌트 | 위치 | 역할 |
| --- | --- | --- |
| `neutron-server` | 컨트롤러 | API 수신, DB에 네트워크 구성 저장, 플러그인 호출 |
| plugin agent (`linuxbridge-agent` / `openvswitch-agent`) | 모든 노드 | DB의 port 정의를 보고 **실제 가상 스위치를 구성·연결** |
| `neutron-dhcp-agent` | 컨트롤러/네트워크 노드 | DHCP 쓰는 네트워크마다 **dnsmasq** 프로세스 생성·관리. `qdhcp` 네임스페이스. `neutron-ns-metadata-proxy`도 띄움. 모든 플러그인에서 공통 사용 |
| `neutron-metadata-agent` | 컨트롤러/네트워크 노드 | DHCP/라우터 네임스페이스의 프록시 ↔ `nova-metadata-api` 중계. Neutron·Nova가 **shared secret**으로 서로 인증 |
| `neutron-l3-agent` | 컨트롤러/네트워크 노드 | 가상 라우터 제공. Linux IP 스택 + iptables로 **L3 포워딩·NAT**, Floating IP. `qrouter` 네임스페이스로 라우터별 격리 |
| `neutron-lbaas-agent` | 컨트롤러/네트워크 노드 | HAProxy 기반 LB. VIP마다 네임스페이스 |

**메타데이터 요청 흐름 (8단계)**

1. 인스턴스가 부팅 시 `http://169.254.169.254`로 HTTP 요청
2. 라우트에 따라 라우터 또는 DHCP 네임스페이스에 도착
3. 네임스페이스의 metadata proxy → Unix 소켓으로 `neutron-metadata-agent`에 전달
4. metadata agent → `nova-metadata-api`로 전달
5. Nova가 응답 → metadata agent
6. metadata agent → proxy
7. proxy → 인스턴스에 HTTP 응답
8. 인스턴스가 메타데이터 처리하고 부팅 계속

**DHCP agent 주요 옵션 (의미만)**

- `use_namespaces` — True면 네트워크마다 `qdhcp-` 네임스페이스. False면 테넌트 간 겹치는 네트워크 불가
- `enable_isolated_metadata` — Neutron 라우터가 게이트웨이가 아닌(flat/VLAN) 네트워크에서도 DHCP로 메타데이터 경로(static route) 제공
- `enable_metadata_network` — L3 agent와 metadata agent가 다른 호스트일 때, 169.254.0.0/16 서브넷을 메타데이터 네트워크로 취급
- `dhcp_domain` — DHCP로 내려주는 DNS 검색 도메인

> 3장 요약: **Neutron은 전체 네트워크 구조를 DB에 유지하고, 각 노드의 plugin agent가 그에 맞게 가상 네트워크를 구성한다. DHCP·metadata 서비스는 컨트롤러/네트워크 노드에서 부팅 시 IP와 인스턴스 정보를 내려준다.**
> 

---

## 4. SRE 6장 — Monitoring Distributed Systems

**용어 정의**

| 용어 | 뜻 |
| --- | --- |
| Monitoring | 시스템의 실시간 정량 데이터를 수집·처리·집계·표시 |
| White-box | 시스템 **내부** 지표 기반(로그, 내부 통계 HTTP 핸들러 등) |
| Black-box | **사용자가 보는 대로** 외부에서 동작을 테스트 |
| Dashboard | 핵심 지표 요약 뷰 |
| Alert | 사람이 읽으라고 보내는 알림 — 티켓 / 이메일 / **페이지** |
| Root cause | 고치면 "같은 방식으로 다시는 안 난다"고 확신할 수 있는 결함. 한 사고에 여러 개일 수 있음 |
| Push | 서비스의 소프트웨어·설정 변경 |

**왜 모니터링하나**

- 장기 추세 분석 · 시간/실험군 비교 · **알림** · 대시보드 · 사후 디버깅

**합리적 기대치**

- 모니터링은 손이 많이 가는 일 — Google SRE 팀(10~12명)에서 보통 1~2명이 전담
- 복잡한 의존 관계 규칙("DB가 느리면 DB 알림, 아니면 웹 알림")은 거의 안 씀 — 인프라가 계속 바뀌어서 깨짐
- 페이지로 가는 규칙은 **단순하고 견고하게**, 명확한 실패만

**증상 vs 원인**

- 모니터링이 답해야 할 두 질문: **무엇이 고장났나(증상) / 왜(원인)**
- 예: "HTTP 500" ← "DB가 연결 거부" / "응답이 느림" ← "CPU 과부하 또는 랙 밑에 눌린 이더넷 케이블(부분 패킷 손실)"
- 다층 시스템에선 한 사람의 증상이 다른 사람의 원인 — DB SRE에게 "느린 읽기"는 증상, 프런트엔드 SRE에게는 원인

**Black-box vs White-box**

- Google: white-box를 많이 + black-box를 **적지만 중요하게**
- Black-box = 증상 지향, **지금 실제로** 깨진 것만 → 페이징에 좋음(사람을 깨울 땐 진짜 문제만). 임박한 문제엔 무용
- White-box = 내부를 들여다봄 → 임박한 문제, 재시도에 가려진 실패 탐지. 디버깅에 필수(웹서버가 보는 DB 속도 vs DB가 보는 자기 속도를 둘 다 알아야 "느린 DB"와 "네트워크 문제"를 구분)

**Four Golden Signals**

| 신호 | 의미 | 주의 |
| --- | --- | --- |
| **Latency** | 요청 처리 시간 | 성공 요청과 실패 요청의 latency를 **구분**. 빠른 500은 평균을 왜곡하고, 느린 에러는 더 나쁘다 |
| **Traffic** | 시스템에 걸리는 수요 | 웹: req/s, 스트리밍: 네트워크 I/O·동시 세션 |
| **Errors** | 실패율 | 명시적(500), 암묵적(200인데 내용이 틀림), 정책 위반(1초 넘긴 응답) |
| **Saturation** | 얼마나 "꽉 찼나" | 가장 제약되는 자원(메모리·I/O). 포화 전 성능 저하가 먼저 온다 → 이용률 목표 필요. 예측("4시간 뒤 디스크 참")도 포함 |
- 이 넷을 측정하고 **하나라도 문제면 페이지(또는 페이지 직전)** → 꽤 괜찮은 모니터링

**꼬리(tail)를 걱정하라**

- 평균은 꼬리를 숨긴다. 평균 100ms라도 1%는 5초 걸릴 수 있음
- 지연을 버킷(히스토그램)으로 나눠 분포를 보라 — 버킷 경계는 지수적으로

**측정 해상도 적절히**

- CPU는 초 단위, 디스크 사용률은 1~2분이면 충분. 세밀한 수집은 비싸다 → 내부에선 세밀하게 샘플링하되 저장은 집계

**가능한 한 단순하게, 그 이상은 아니게**

- 가장 자주 발생하는 사고를 잡는 규칙은 최대한 단순·예측 가능·신뢰성 있게
- 거의 안 쓰는 수집·집계·알림 설정은 제거 후보
- 수집은 되는데 대시보드에도 없고 알림에도 안 쓰는 신호도 제거 후보

**페이지 알림 설계 — 질문과 철학**

- 이 규칙이 아니면 놓쳤을, 긴급하고 조치 가능하며 **지금 사용자에게 보이는** 상태를 잡는가?
- 이 알림이 양성(benign)이라 무시해도 되는 경우가 있나? 그 경우를 규칙에서 뺄 수 있나?
- 정말 사용자에게 영향이 있나? (드레인된 트래픽, 테스트 배포처럼 걸러낼 수 있는 경우 아닌가)
- 조치 가능한가? 긴급한가 아침까지 미뤄도 되나? 자동화 가능한가? 장기 수정인가 임시 우회인가?
- 다른 사람도 같은 문제로 페이지 받고 있나? (중복)

→ **페이지는 ① 긴급하게 반응할 수 있는 것 ② 조치 가능한 것 ③ 사람의 지능이 필요한 것 ④ 새로운 문제**여야 한다. 이 넷을 만족하면 white-box든 black-box든 상관없음. **증상을 잡는 데 훨씬 많이 투자하고, 원인은 아주 확실하고 아주 임박한 것만.**

**장기적 관점의 모니터링**

- 오늘의 페이지 하나하나가 내일의 시스템 개선을 방해 → 장기 전망을 위해 단기 가용성을 희생할 때도 있음
- Bigtable 사례(over-alerting): 평균 기반 SLO가 꼬리 때문에 계속 깨짐 → 알림을 조정하고 SLO를 현실에 맞게 낮춰 팀이 근본 수정에 집중
- Gmail 사례: 스케줄러 버그로 태스크가 deschedule되는 알림이 반복 → 땜질 자동화 대신 근본 원인을 추적하기로 결정
