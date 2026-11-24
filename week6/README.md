# 6회차 — 규칙은 열려 있는데요

> 세션: 2026-11-30 (월) 22:00 디스코드 · 제출 마감: 2026-11-29 (일) 23:59

## 0. 시작 전 확인

- `<본인ID>`는 본인 슬롯 이름입니다 (예: `cc01` → 프로젝트 `study-cc01`, 시큐리티 그룹 `cc01-sg`, VM `cc01-vm1`). 제출 파일·브랜치에는 `<본인이름>`을 씁니다.
- **수요일(11/25)까지** 본인 `vm1`, `vm2`, `vm3`의 **포트 ID**와 올라가 있는 노드를 기록합니다. 이번 주 장치 이름과 체인 이름이 전부 포트 ID에서 나옵니다.
- 밖에서 VM으로 들어오는 통신과 VM 안에서 밖으로 나가는 통신을 **서로 다른 테스트**로 분리합니다.
- Grafana(`localhost:3001/d/slot-probe`)에서 본인 슬롯 줄의 프로브 색을 기록합니다. 표가 옆으로 길어서 오른쪽 끝까지 스크롤해야 전부 보입니다. **회색(측정 불가)은 실패(빨강)와 다릅니다.**

## 1. 읽기 범위

| 교재 | 범위 | 읽으면서 볼 것 |
| --- | --- | --- |
| Learning OpenStack Networking | **8장 PDF p.240–244** (시큐리티 그룹, iptables 기초), **p.248–253** (규칙 구현, 체인 따라가기) | 시큐리티 그룹 규칙이 **무엇으로 바뀌어 어느 노드에** 깔리는지, 패킷이 **어느 체인을 어떤 순서로** 지나는지, 새 시큐리티 그룹에 **처음부터 들어 있는 규칙**이 무엇인지 |
| SRE | **4장 PDF p.63–74** (Service Level Objectives) | SLI·SLO·SLA의 차이, 지표를 사용자가 신경 쓰는 것에서 고르는 이유, 목표를 100%로 잡지 않는 이유 |
| 보강 자료 ( 선택 ) | [Neutron 2026.1: Open vSwitch Native Firewall Driver](https://docs.openstack.org/neutron/2026.1/admin/config-ovsfwdriver.html) 첫 문단 | VM과 `br-int` 사이에 왜 Linux bridge(`qbr`)가 끼어 있는지 |

> 쪽수는 **PDF 뷰어 페이지 기준**입니다. 책 내용을 우리 환경에 대입할 때 주의할 점:
>
> - 책 예시는 LinuxBridge 플러그인이라 체인 이름이 `neutron-linuxbri-*`입니다. 우리 클러스터(OVS)에서는 `neutron-openvswi-*`이고 구조는 같습니다.
> - `neutron security-group-*` 명령은 옛 버전 CLI입니다. 우리 환경에서는 `openstack security group …`입니다.
> - p.240의 FWaaS와 p.252–253의 대시보드 화면은 훑어만 보세요. 핵심은 **p.249–251**입니다.

## 2. 장애 신고

> **신고자:** 스터디원
>
> **내용:** ping은 되는데 **제 VM에 SSH로 들어가거나, VM 안에서 밖으로 나가는 동작 중 일부가 실패합니다.** 시큐리티 그룹 규칙은 확인했는데 22번도 ICMP도 열려 있습니다. Horizon에서는 VM과 네트워크가 정상으로 보입니다.

본인 환경의 증상은 아래 둘 중 **하나**입니다. 어느 쪽인지는 알려드리지 않으니 3번의 테스트로 직접 확인하세요.

- **A:** `vm2`에 SSH가 안 됩니다(timeout). ping은 되고, 같은 시큐리티 그룹을 쓰는 `vm1`은 SSH가 됩니다.
- **B:** VM에 SSH로 들어가는 건 되는데, VM 안에서 밖으로 나가는 게 안 됩니다. `vm1`~`vm3`이 전부 그렇습니다.

**대상**

- 프로젝트: `study-<본인ID>`
- 시큐리티 그룹: `<본인ID>-sg`
- 비교 대상: `vm1`(`cmp01`)과 `vm2`(`cmp02`), 들어오는 방향과 나가는 방향
- 공통 경로: `br-int → qvo<11자> → qvb<11자> → qbr<11자> → tap<11자> → VM`, 반대 방향도 비교 (`<11자>`는 포트 ID 앞 11자)
- 원인은 **한 군데**이고 사람마다 다릅니다. 지난주(5회차) 증상과 닮아 보일 수 있습니다.

## 3. 요구

**① 밖에서 VM으로 들어오는 통신 ② VM 안에서 밖으로 나가는 통신**을 구분해서 테스트하세요. 실패를 발견했다면 VM이 올라간 compute 노드에서 **패킷이 마지막으로 보인 장치와 처음 안 보인 장치**를 짚고, 그 사이에 걸린 규칙을 책 p.249–251 순서대로 따라가 **어느 줄에서 패킷의 운명이 정해졌는지** 설명하세요.

### 테스트 방법

**들어오는 방향(①)** — **net01 호스트**에서 `vm1`, `vm2` 각각 실행합니다.

```bash
ping -c3 <FIP>
ssh -o ConnectTimeout=5 -i <키> cirros@<FIP> true && echo OK
```

출발지를 바꿔서도 확인합니다. NAT를 거치지 않는 경로입니다.

```bash
# net01, 라우터 네임스페이스에서 fixed IP로
sudo ip netns exec qrouter-<라우터ID> ssh -o ConnectTimeout=5 -i ~/<본인ID>-key.pem cirros@<VM fixed IP> true && echo OK
```

**나가는 방향(②)** — VM 안에서 세 단계를 각각 기록합니다. VM에 들어가는 방법은 5회차와 같습니다. `vm2`에 못 들어가면 `vm1`에서 합니다.

| 단계 | 명령 (VM 안) | 확인하는 구간 |
| --- | --- | --- |
| 본인 게이트웨이 | `ping -c3 <게이트웨이 IP>` | 같은 네트워크의 라우터까지 |
| 옆 VM | `ping -c3 <다른 VM fixed IP>` | 같은 네트워크 안 |
| 인터넷 IP | `ping -c3 8.8.8.8` | 라우터 바깥까지 |

> 게이트웨이 IP는 `openstack subnet show <본인ID>-subnet -c gateway_ip`로 확인합니다. `1.1.1.1`은 정상일 때도 실패하니 쓰지 마세요.

## 4. 확인 지점 체크리스트

| # | 지점 | 쓸 만한 확인 방법 |
| --- | --- | --- |
| ① | 방향을 구분한 증상 표 | VM별로 `FIP ping / FIP SSH / 네임스페이스 SSH / 나가는 ping 3단계` 성공·실패를 각각 기록 |
| ② | OpenStack이 알고 있는 규칙 | `openstack security group rule list <본인ID>-sg` (**Direction 열**에서 ingress·egress를 각각 세기) · `openstack security group show <본인ID>-sg` · `openstack port show <포트ID> -c security_group_ids -c port_security_enabled` |
| ③ | 포트 ID와 장치 이름 | `openstack port list --server <본인ID>-vmN -f value -c ID` (**vm1~vm3 모두**). 장치는 앞 11자(`tap…`, `qbr…`, `qvb…`, `qvo…`), 체인은 앞 10자 |
| ④ | 장치별로 패킷이 어디까지 보이는지 | VM이 있는 노드에서 `sudo tcpdump -nei qvo<11자> -c 10 'tcp port 22 or icmp'` 를 `qvb<11자>`, `tap<11자>`에서도 같은 조건으로 실행해 비교 |
| ⑤ | 장치 사이에 있는 것 | `ip link show master qbr<11자>` · `sysctl net.bridge.bridge-nf-call-iptables` |
| ⑥ | 체인 따라가기 (변경 금지) | `sudo iptables -S FORWARD` → `sudo iptables -S neutron-openvswi-sg-chain \| grep <11자>` → `sudo iptables -S neutron-openvswi-i<10자>` · `…-o<10자>` · `…-s<10자>` → `sudo iptables -S neutron-openvswi-sg-fallback` |
| ⑦ | 어느 줄에 걸렸는지 | `sudo iptables -L neutron-openvswi-i<10자> -n -v --line-numbers` (o 체인도 같은 방법). 테스트 패킷 **전후로 두 번** 실행해 `pkts`가 늘어난 줄 찾기. 정상인 VM의 체인과 나란히 비교 |
| ⑧ | 결론 | ②의 API 규칙과 ⑥의 노드 규칙을 한 줄씩 짝지은 표, 어긋난 줄, 방향별 판정과 근거 |

> `iptables`는 VM이 올라간 compute 노드에서 조회합니다(`vm1`은 `cmp01`, `vm2`는 `cmp02`). 체인 안에 보이는 `DROP`이 전부 장애는 아닙니다. 정상 구성에도 `DROP`이 있습니다.
>
> 캡처와 카운터에는 Grafana 프로브 패킷(net01에서 FIP로 보내는 ping과 22번 접속)이 함께 잡힙니다. 본인이 보낸 패킷과 구분해서 읽으세요.
>
> 책 p.251은 체인 이름에 포트 ID "앞 9자"가 들어간다고 쓰지만 예시(`c2a46367-1`)는 10자입니다. 우리 환경도 10자입니다.

## 5. 해결 조건

- 실패 범위를 들어오는 SSH만 / VM이 시작하는 통신 전부 등으로 구체화할 것
- 패킷이 **마지막으로 보인 장치**와 **처음 안 보인 장치**를 제시할 것
- API의 규칙과 노드의 규칙을 짝지은 표에서 **어긋난 줄**(남는 줄 또는 빠진 줄)을 특정할 것
- 되는 것(ping, 들어오는 SSH, 다른 VM)이 **왜 되는지**까지 원인으로 설명할 것
- "지난주와 같은 라우터·NAT 문제" 가설을 세웠다면 그와 이번 장애를 구별할 관찰 결과를 적을 것
- SRE: 이 장애를 잡는 **SLI를 한 문장으로 정의**하고(무엇을, 어디서, 얼마 간격으로, 어떤 값이면 정상) **SLO 숫자 하나**를 이유와 함께 붙일 것. 기존 프로브 중 무엇이 반응했는지도 적을 것

## 6. 규칙

- 허용: **조회, 제한된 ping/SSH 확인, tcpdump, `iptables -S`/`-L` 조회, 로그·통계 읽기**. 가능한 한 본인 리소스로 대상을 한정합니다.
- 금지:
  - **시큐리티 그룹 규칙 추가·삭제·수정** (본인 `<본인ID>-sg` 포함), 포트의 시큐리티 그룹 교체, 새 시큐리티 그룹 생성
  - **VM 재부팅** (`vm1`~`vm3` 전부), VM 생성·삭제
  - `iptables`/`nft` 규칙 변경, `conntrack -D/-F`
  - 네트워크·라우터·포트·FIP 변경, `ip link set`, `ovs-vsctl add-port/del-port`
  - 컨테이너·에이전트 재시작, `reconfigure`
- 시큐리티 그룹을 하나라도 건드리면 **증상이 바뀌거나 사라질 수 있습니다.** 그렇게 사라진 경우는 원인을 설명하지 못하면 푼 것으로 보지 않습니다. 이유는 세션에서 다룹니다.
- 공용 노드에서 대량 트래픽을 만들거나 전체 패킷을 오래 캡처하지 마세요. `tcpdump -c`와 `host` 필터로 짧게 관찰합니다.
- 복구는 실행하지 않고 **방법만 제안**합니다. 실제 복구는 운영진이 세션에서 진행합니다.
- 노드 접속은 본인 계정으로만 합니다. 권한이 없는 확인 지점은 운영진에게 요청하고 그 사실을 기록하세요.

## 7. 제출

- `week6/<본인이름>.md` — 루트 `template.md`를 복사해 작성합니다. 2번에 본인 증상이 **A·B 중 무엇이었는지** 적어주세요.
- 브랜치: `week6/<본인이름>` → PR 제목: `[week6] <본인이름>` (루트 README의 **Git으로 제출하기** 참고).
- 제출 마감: **2026-11-29 (일) 23:59**.
- 세션에서 3명 랜덤 발표합니다. **정답이 없어도 검증 기록과 가설 변경 과정이 있으면 발표할 수 있습니다.**
