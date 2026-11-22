# 5회차 — VM 밖으로 나가는 길이 사라졌다

> 세션: 2026-11-23 (월) 22:00 디스코드 · 제출 마감: 2026-11-22 (일) 23:59

## 0. 시작 전 확인

- `<본인ID>`는 본인 슬롯 이름입니다 (예: `cc01` → 프로젝트 `study-cc01`, 라우터 `cc01-router`, VM `cc01-vm1`). 제출 파일·브랜치에는 `<본인이름>`을 씁니다.
- **수요일(11/18)까지** 본인 `vm1`, `vm2`, `vm3`의 fixed IP와 FIP 부착 여부를 기록합니다.
- 내부 통신, 외부로 나가는 통신, 외부에서 들어오는 통신을 **서로 다른 테스트**로 분리합니다.
- Grafana(`localhost:3001`)에서 본인 FIP 프로브 변화 시각을 확인합니다. **변화가 없는 것**도 기록할 결과입니다.

## 1. 읽기 범위

| 교재                            | 범위                                                                                                                                                                                               | 읽으면서 볼 것                                                   |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------- |
| Learning OpenStack Networking | **6장 PDF p.171–190** (NAT, Floating IP, 패킷 흐름)                                                                                                                                                   | `qr-`*, `qg-*`, Floating IP, SNAT/DNAT 때 주소가 바뀌는 위치        |
| SRE                           | **12장 PDF p.163–171** (In Practice ~ Negative Results Are Magic)                                                                                                                                 | 원인 추정 전에 실패 범위를 가장 빠르게 줄이는 실험을 어떻게 고를지, 어떤 관찰이 가설을 버리게 하는지 |
| 보강 자료 ( 선택 )                  | [Neutron 2026.1: NAT](https://docs.openstack.org/neutron/2026.1/admin/intro-nat.html) · [OVS Self-service networks](https://docs.openstack.org/neutron/2026.1/admin/deploy-ovs-selfservice.html) | FIP가 있는 VM과 없는 VM의 **North-South** 경로 차이                   |

> 쪽수는 **PDF 뷰어 페이지 기준**입니다. 책 시연을 우리 환경에 대입할 때 주의할 점:
>
> - 실제 외부망은 실습용으로 구성되어 있습니다. 책의 물리 라우터(Cisco ASA) 예시를 그대로 대입하지 마세요.
> - 책의 compute 쪽 구성은 LinuxBridge 기반입니다. 우리 클러스터(OVS)에는 **라우터 네임스페이스(qr/qg/NAT) 부분만** 대입합니다.
> - `neutron ...` CLI, cirros 기본 비밀번호 등은 옛 버전 값입니다.

## 2. 장애 신고

> **신고자:** 스터디원
>
> **내용:** 같은 네트워크 안에서는 접속이 되는 것 같은데, **외부에서 제 VM으로 접속하거나 VM에서 바깥으로 나가려는 동작 중 일부가 실패합니다.** Horizon에서는 네트워크와 VM이 정상으로 보입니다.

**대상**

- 프로젝트: `study-<본인ID>`
- 네트워크·라우터: `<본인ID>-net`, `<본인ID>-router`
- 비교 대상: FIP가 연결된 VM과 FIP가 없는 VM (각자의 실제 구성 확인)
- 공통 경로: `net01 호스트(veth-gw, 203.0.113.1) → veth-ext → br-ex → qg-* → qrouter → qr-* → compute VM`, 반대 방향도 비교
- 외부망 게이트웨이 `203.0.113.1`은 net01 호스트에 있는 실습용 주소입니다. ops01은 FIP 대역(`203.0.113.0/24`)으로 라우팅되지 않습니다.

## 3. 요구

**① VM끼리의 내부 연결 ② VM→외부 ③ 외부(net01 호스트)→FIP→VM**을 구분해서 테스트하세요. 실패를 발견했다면 `net01`의 라우터 네임스페이스 안팎에서 **NAT 전후 주소/인터페이스가 어떻게 달라지는지** 설명하고, 마지막 정상 지점을 표시하세요.

### 테스트 방법

**VM 안에서 하는 테스트(①②)** — 외부 경로와 상관없이 들어갈 수 있도록 `net01`의 라우터 네임스페이스를 거쳐 접속합니다. FIP가 없는 VM도 이 방법으로 들어갑니다.

```bash
# net01, 본인 계정. 키는 본인 net01 홈에 두고 chmod 600
# 【확인 필요: 학생 키 위치/복사 방법】
sudo ip netns exec qrouter-<라우터ID> ssh -i ~/<본인ID>-key.pem cirros@<VM fixed IP>
```

**outbound(②)** — VM 안에서 세 단계를 각각 기록합니다.

| 단계 | 명령 (VM 안) | 확인하는 구간 |
| --- | --- | --- |
| 외부망 게이트웨이 | `ping -c3 203.0.113.1` | 라우터 바깥 첫 홉까지 |
| 인터넷 IP | `ping -c3 8.8.8.8` | 실습 외부망 너머까지 |
| 도메인 | `curl -s -m5 -o /dev/null -w '%{http_code}\n' http://example.com` | DNS + TCP |

> `1.1.1.1`은 클러스터 전체에서 ICMP가 막혀 있어 정상일 때도 실패합니다. 쓰지 마세요.

**FIP inbound(③)** — **net01 호스트**에서 실행합니다.

```bash
ping -c3 <FIP>
ssh -o ConnectTimeout=5 -i <키> cirros@<FIP> true && echo OK
```

> ops01에서 FIP로 보내는 ping은 정상일 때도 실패합니다(라우팅 없음). ops01의 SSH는 net01을 거쳐 들어가므로(ProxyJump) net01에서 시험한 것과 같은 경로입니다.

## 4. 확인 지점 체크리스트

| # | 지점 | 쓸 만한 확인 방법 |
| --- | --- | --- |
| ① | 외부와 내부를 구분한 증상 표 | 본인 VM별로 `내부 ping / outbound(게이트웨이·인터넷·도메인) / FIP inbound` 성공·실패를 각각 기록 |
| ② | OpenStack이 알고 있는 라우터·FIP | `openstack router show <본인ID>-router` · `openstack floating ip list` · `openstack port list --server <본인ID>-vmN` (**vm1~vm3 모두**) |
| ③ | 라우터 네임스페이스와 두 종류의 인터페이스 | `sudo ip netns list` · `sudo ip netns exec qrouter-<라우터ID> ip -br addr` · `sudo ip netns exec qrouter-<라우터ID> ip route` |
| ④ | 라우터 안쪽(qr)과 바깥쪽(qg)의 패킷 차이 | `sudo ip netns exec qrouter-<라우터ID> tcpdump -nei qr-<…> -c 20 '(icmp or arp or tcp port 22) and host <대상 IP>'` 를 `qg-<…>`에서도 같은 조건으로 실행해 비교 |
| ⑤ | 라우터 네임스페이스 바깥(호스트) | `net01`에서 `203.0.113.1`이 붙은 인터페이스(`ip -br addr \| grep 203.0.113.1`)나 `br-ex`의 veth 포트(`sudo docker exec openvswitch_vswitchd ovs-vsctl list-ports br-ex`)에서 `sudo tcpdump -nei <인터페이스> -c 20 'host <본인 FIP 또는 라우터 게이트웨이 IP>'`. 이웃 캐시는 `ip neigh show <본인 FIP>`. **공용 인터페이스이므로 본인 IP 필터 필수.** `br-ex` 이름으로 캡처하면 브리지를 지나가는 패킷이 거의 보이지 않습니다 |
| ⑥ | NAT 규칙을 읽어 보기 (변경 금지) | `sudo ip netns exec qrouter-<라우터ID> iptables -t nat -S` (또는 권한 허용된 동등한 조회) |
| ⑦ | `br-ex` 및 외부 연결 확인 | `sudo docker exec openvswitch_vswitchd ovs-vsctl show` (net01) · `ip -br link` (호스트 쪽 veth 상태. `qg-*` 상태는 ③에서) |
| ⑧ | 결론 | VM별 ①②③ 판정과 근거, inbound/outbound 차이, IP 변환 전후의 증거 |

> 캡처에는 Grafana 프로브 패킷(net01에서 FIP로 보내는 여러 크기의 ping과 SSH)이 함께 보입니다. 본인이 보낸 패킷과 구분해서 읽으세요.
>
> FIP가 없거나 특정 경로가 본인 실습 구성에 해당하지 않으면 "미적용"으로 적으세요. `openstack floating ip set/unset`, `openstack server add/remove floating ip`, 라우터 수정은 금지합니다.

## 5. 해결 조건

- 실패 범위를 외부 전체 / outbound만 / FIP inbound만 등으로 구체화할 것
- NAT와 라우터를 포함한 패킷 경로에서 **마지막 정상 지점**을 제시할 것
- "SG 문제" 가설을 세웠다면 그와 라우터 경로 장애를 구별할 관찰 결과를 적을 것
- SRE: 어떤 **외부 경로 synthetic probe**가 가장 먼저 실패했어야 하는지 제안할 것

## 6. 규칙

- 허용: **조회, 제한된 ping/curl/SSH 확인, tcpdump, 로그·통계 읽기**. 가능한 한 본인 리소스로 대상을 한정합니다.
- 금지:
  - 네트워크·라우터·포트·SG·FIP 변경
  - `ip link set`, `ip addr add/del`, `ip route add/del/replace`, `ip neigh` 변경 (**라우터 네임스페이스 안 포함**)
  - `arping`, `conntrack -D/-F`
  - `ovs-vsctl add-port/del-port`, `iptables`/`nft` 규칙 변경
  - VM·컨테이너·에이전트 재시작, `reconfigure`
- 공용 노드에서 대량 트래픽을 만들거나 전체 패킷을 오래 캡처하지 마세요. `tcpdump -c`와 `host` 필터로 짧게 관찰합니다.
- 복구는 실행하지 않고 **방법만 제안**합니다. 실제 복구는 운영진이 세션에서 진행합니다.
- 노드 접속은 본인 계정으로만 합니다. 권한이 없는 확인 지점은 운영진에게 요청하고 그 사실을 기록하세요.

## 7. 제출

- `week5/<본인이름>.md` — 루트 `template.md`를 복사해 작성합니다.
- 브랜치: `week5/<본인이름>` → PR 제목: `[week5] <본인이름>` (루트 README의 **Git으로 제출하기** 참고).
- 제출 마감: **2026-11-22 (일) 23:59**.
- 세션에서 2~3명 랜덤 발표합니다. **정답이 없어도 검증 기록과 가설 변경 과정이 있으면 발표할 수 있습니다.**