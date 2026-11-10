# 4회차 — 포트 · DHCP · 메타데이터 · MTU: VM은 ACTIVE인데요

> 세션: 2026-11-16 (월) 22:00 디스코드 · 제출 마감: 2026-11-15 (일) 23:59

## 0. 시작 전 확인

- 3회차 복구 이후 본인 VM의 정상 기준선(vm1~3 FIP SSH, 내부 ping)을 다시 확인하고 메모하세요.
- 본인 **네트워크 ID**(`openstack network show <본인ID>-net -f value -c id`)를 미리 적어두세요. 이번 주 확인 지점에서 계속 쓰입니다.
- 이번 주 신고 세 건 중 **하나**가 본인 환경에 들어가 있습니다.
- 신고 A·B는 "새로 만든 VM"이 대상입니다. 운영진이 본인 프로젝트에 **`<본인ID>-vm4`** 를 새로 만들어 두었습니다(vm1~3과 같은 네트워크·SG·키). vm4가 보이면 그게 신고 대상이고, 직접 VM을 만들 필요는 없습니다. 신고 C는 기존 vm1이 대상이라 vm4가 없습니다.

## 1. 읽기 범위

| 교재 | 범위 | 읽으면서 볼 것 |
| --- | --- | --- |
| Learning OpenStack Networking | **5장 PDF p.128–130** (Network management), **p.140–141** (Subnets), **p.153** (Ports), **p.154–162** (Attaching instances ~ metadata) | 포트의 IP는 **누가 언제** 정하는지. VM이 `169.254.169.254`로 가는 길은 **어디를 거치는지** — 책이 말하는 두 경로와 그 조건 |
| 공식 문서 | [Neutron 2026.1: MTU considerations](https://docs.openstack.org/neutron/2026.1/admin/config-mtu.html) | 1450이 어떻게 계산되고 VM에 **어떤 방법으로** 전달되는지 |
| SRE | **12장 PDF p.172–176** (Negative Results Are Magic ~ Making Troubleshooting Easier) · **6장 Black-Box vs White-Box** 재독 (3회차 범위) | 기각한 가설을 왜 기록해야 하는지. 블랙박스 프로브가 **못 보는 길**은 어디인지 |

> 쪽수는 **PDF 뷰어 페이지 기준**입니다. 1판의 개념은 활용하되, 명령과 구현은 실습 환경의 실제 구성을 우선합니다. 책의 `neutron` CLI는 우리 환경에서 `openstack` CLI입니다.

## 2. 장애 신고

세 건 중 **본인이 DM으로 받은 신고 하나**만 다룹니다. 다른 두 건은 읽어만 두세요.

> **신고 A — 스터디원:** 어제 vm4를 하나 새로 만들었는데 **네트워크에 안 붙습니다.** Horizon에선 ACTIVE고 IP도 적혀 있는데 SSH가 안 되고, 라우터 네임스페이스에서 ping도 안 갑니다. 기존 vm1~3은 전부 정상이에요. 뭐가 다른 거죠?

> **신고 B — 스터디원:** 어제 vm4를 새로 만들었는데 **키로 SSH 로그인이 안 됩니다.** `Permission denied (publickey)`가 나요. 그런데 콘솔에서 비밀번호(cirros/gocubsgo)로는 로그인이 되고, 들어가 보면 IP도 받았고 ping도 됩니다. 키 파일은 vm1~3에 잘 되는 그 파일이에요.

> **신고 C — 스터디원:** vm1에 **SSH 접속은 되는데**, `dmesg`처럼 출력이 긴 명령을 치거나 파일을 받으면 **화면이 멈춥니다.** 한참 기다려도 안 풀려서 끊고 다시 붙으면 또 들어가져요. ping도 됩니다. vm2는 멀쩡하고요.

**대상**

- 프로젝트: `study-<본인ID>`
- A·B: `<본인ID>-vm4` (운영진이 새로 만들어 둔 VM) — 비교용 `<본인ID>-vm1~3`
- C: `<본인ID>-vm1` (`cmp01`) — 비교용 `<본인ID>-vm2` (`cmp02`)

**참고**

- 원인은 **한 군데**이고, 사람마다 다릅니다. 옆 사람 답이 내 답이 아닙니다.
- **미끼 주의.** A·B는 기존 VM이 멀쩡해서 "환경은 정상"으로 보이고, C는 ping·SSH가 되어서 "네트워크는 정상"으로 보입니다. 그 인상에서 멈추지 마세요. 2회차 문장 그대로 — 가설은 증상을 **전부** 설명해야 합니다. "기존 VM은 왜 정상인가", "왜 작은 것만 되는가"까지 설명되어야 끝입니다.
- Grafana는 이번 주 전부 초록으로 보이는 게 맞습니다. 그 이유가 세 질문 ③의 재료입니다.

## 3. 요구

증상이 나타난 지점에서 멈추지 말고, **패킷(또는 DHCP·메타데이터 요청)이 출발해 응답을 받기까지 지나는 경로를 순서대로** 따라가며 어느 구간에서 끊기는지 짚으세요.

- A: DHCP 요청은 어디서 출발해 어디로 가서 **누가** 답하는가. 요청은 도착했는가, 응답은 나갔는가.
- B: `169.254.169.254` 요청은 VM 안 라우팅 테이블에서 **어디를 향하고**, 그 끝에서 **누가 80번을 듣고** 있는가.
- C: 되는 것과 안 되는 것의 **차이가 무엇인지**(어떤 조건을 바꾸면 결과가 바뀌는지) 먼저 특정하고, 그 조건과 관련된 값을 경로 장치마다 비교하세요.
- 마지막으로 **끊긴 지점이 왜 이 증상으로 보였는지**(미끼가 왜 정상으로 보였는지) 한 문단으로 설명하세요.

## 4. 확인 지점 체크리스트

위에서부터 순서대로 좁혀가세요. 각 지점에서 **무엇을 확인했고 결과가 어땠는지**를 제출물에 남깁니다. 기각한 가설도 남기세요 — 이번 주 SRE 범위 제목이 "부정적 결과는 마법이다"입니다.

| # | 지점 | 쓸 만한 확인 방법 |
| --- | --- | --- |
| ① | OpenStack이 보는 상태 — 포트·IP·MAC·SG·키·네트워크 mtu | `openstack server show <VM>` (`key_name`, `addresses`) · `openstack port list --network <본인ID>-net -c ID -c "MAC Address" -c "Fixed IP Addresses" -c device_owner` · `openstack subnet show <본인ID>-subnet` · `openstack network show <본인ID>-net -c mtu` · `openstack security group rule list <본인ID>-sg` |
| ② | 게스트가 실제로 받은 것 — 노드 접속 없이 콘솔 로그로 | `openstack console log show <VM>` — `dhcpcd` 줄(`soliciting a DHCP lease` 다음에 `leased …`가 있는지), `=== network info ===` 블록의 `if-info`·`ip-route`, `=== datasource: … ===` 줄. 비교용으로 **vm1의 콘솔 로그**도 같이 보세요. 필요하면 Horizon 콘솔(또는 `openstack console url show`)에서 `cirros`/`gocubsgo`로 로그인해 `ip -4 addr`, `ip route`, `ip link show eth0` |
| ③ | 조건을 바꿔 비교 — **어떤 조건에서 되고 안 되는가** | 기존 VM vs vm4 · 비밀번호 vs 키 · net01에서 `ping -c2 -s 1200 <FIP>` vs `ping -c2 -s 1400 <FIP>` (크기를 바꿔가며 경계 찾기) · VM 안에서 `curl -m 3 http://169.254.169.254/latest/meta-data/` |
| ④ | net01의 DHCP 네임스페이스 — 요청이 **도착하는지**, **누가 듣는지**, **사이에 무엇이 있는지** | `sudo ip netns list` · `sudo ip netns exec qdhcp-<네트워크ID> ip -4 addr` · `sudo ip netns exec qdhcp-<네트워크ID> tcpdump -ni <tap이름> -c 20 'port 67 or port 68 or port 80'` (vm4 재부팅 `openstack server reboot <본인ID>-vm4` 또는 VM 안 curl과 동시에) · `sudo ip netns exec qdhcp-<네트워크ID> ss -ltunp` · `sudo ip netns exec qdhcp-<네트워크ID> iptables -L INPUT -n -v --line-numbers` (조회만, 두 번 실행해 카운터 변화 비교) · `sudo docker exec neutron_dhcp_agent cat /var/lib/neutron/dhcp/<네트워크ID>/host` · `… /opts` · `sudo tail -50 /var/log/kolla/neutron/dnsmasq.log` |
| ⑤ | cmp 노드의 VM 경로 장치 — 연결 상태와 **MTU 값** 비교 | `openstack port list --server <VM> -c ID` → 앞 11자리 · 해당 cmp에서 `ip link show tap<11자리>` · `qbr` · `qvb` · `qvo` 각각의 `mtu` · `sudo dmesg \| grep -i mtu \| tail` · 필요하면 `sudo tcpdump -ni tap<11자리> -c 10 icmp` |
| ⑥ | 결론 | 요청이 **마지막으로 확인된 지점**과 **처음 사라진 지점**, 그 사이에서 찾은 값, 그리고 미끼(기존 VM 정상 / ping·SSH 정상)가 왜 정상으로 보였는지 |

> 콘솔 로그는 부팅이 끝나야 다 찍힙니다. vm4는 사정에 따라 로그인 프롬프트까지 **10분 넘게** 걸릴 수 있으니, 끝까지 안 나왔으면 기다렸다가 다시 보세요. `169.254.x.x` 주소는 DHCP 응답이 없을 때 게스트가 **스스로 지어내는 링크로컬(IPv4LL) 주소**입니다 — 주소가 있다고 "받았다"는 뜻은 아닙니다.

> 네임스페이스 안 명령은 `ip netns exec qdhcp-<네트워크ID>`를 앞에 붙이면 그 안에서 실행됩니다. `tcpdump`, `ss`, `iptables -L/-S`는 **조회**입니다. `iptables -A/-I/-D`처럼 바꾸는 명령은 쓰지 마세요.

> 우리 환경은 `force_metadata = true`라 책의 설명과 한 군데가 다릅니다. 책 p.141·p.160의 조건문과 VM 안 `ip route` 결과를 대조해 보세요 — 그 차이가 B의 경로를 결정합니다.

> `ping -s N`은 ICMP 페이로드 크기입니다. 실제 IP 패킷은 N + 28 바이트입니다.

## 5. 해결 조건

- 요청(DHCP / 메타데이터 / 큰 패킷)이 **어디까지 갔고 어디서 사라졌는지** 근거와 함께 특정할 것
- **되는 것과 안 되는 것의 차이**(기존 VM vs vm4, 키 vs 비밀번호, 작은 패킷 vs 큰 패킷)가 원인으로 설명될 것 — 미끼가 왜 정상으로 보였는지 포함
- 책의 설명과 우리 환경의 차이를 한 군데 이상 짚을 것 (p.141·p.160의 조건 / 프록시 프로세스 이름 / 명령어)
- SRE: 이 장애를 사용자 신고 전에 잡으려면 **기존 프로브로 가능한지**, 안 되면 어떤 Black-box 프로브와 White-box 신호가 필요한지 한 줄씩 제안할 것
- 복구 방법은 문서로만 제안하되, **정상 구성은 건드리지 않고 원인만 되돌리는** 방법인지 설명할 것
- (못 풀었다면) 어디까지 확인했고 어디서 막혔는지, 기각한 가설이 무엇인지 기록이 있을 것

## 6. 규칙

- 허용: **조회, 제한된 ping/curl/SSH 확인, tcpdump(-c로 짧게), 네임스페이스 안 `ss`·`iptables -L/-S` 조회, 파일·로그 읽기.** 가능한 한 본인 리소스로 대상을 한정합니다.
- 허용 (A·B 한정): **본인 vm4 재부팅** `openstack server reboot <본인ID>-vm4` — DHCP 요청·메타데이터 요청을 다시 보게 하려면 이게 필요합니다. 콘솔 로그는 재부팅 후 다시 쌓입니다.
- 금지: VM 생성·삭제(vm4 포함), 네트워크·서브넷·라우터·포트·SG·FIP 변경, `ip link set`, `ip route add/del`(VM 안 포함), `ovs-vsctl add-port/del-port`, `ovs-ofctl add-flow/del-flows`, `iptables`/`nft` 규칙 변경, 네임스페이스·컨테이너·에이전트 재시작, vm1~3 재시작, `reconfigure`, 서브넷 DHCP 끄기/켜기.
- 공용 노드에서 대량 트래픽을 발생시키거나 전체 패킷을 장시간 캡처하지 마세요.
- 복구는 실행하지 않고 **방법만 제안**합니다. 실제 복구와 vm4 정리는 운영진이 세션 후에 진행합니다. 이번 주 장애는 **네트워크 단위**라 한 사람이 고치면 같은 네트워크의 다른 확인이 바뀝니다.
- 노드 접속은 본인 계정으로만 합니다. 권한이 없는 확인 지점은 운영진에게 요청하고 그 사실을 기록하세요.

## 7. 제출

- `week4/<본인이름>.md` — 루트 `template.md`를 복사해 작성합니다. 2번에 **본인이 받은 신고(A·B·C)** 를 적어주세요.
- 브랜치: `week4/<본인이름>` → PR 제목: `[week4] <본인이름>` (루트 README의 **Git으로 제출하기** 참고).
- 제출 마감: **2026-11-15 (일) 23:59**.
- 세션에서 3명 랜덤 발표합니다. **정답이 없어도 검증 기록과 기각한 가설이 있으면 발표할 수 있습니다.**
