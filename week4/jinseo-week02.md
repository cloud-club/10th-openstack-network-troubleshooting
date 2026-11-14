# 4회차 제출 — 최진서

> 계정: `cc04` / 작성일: 2026-11-14

## 1. 읽기 핵심 (한 장)

- OpenStack에 VM의 IP가 표시돼도 VM 운영체제가 그 IP를 설정했는지는 따로 확인해야 한다.
- Neutron은 포트에 IP와 MAC을 할당하고, VM은 DHCP를 통해 실제 IP 설정을 받는다. 그래서 포트 정보와 콘솔 로그를 비교했다.
- DHCP 요청은 VM에서 출발해 net01의 qdhcp 네임스페이스로 전달되고, 그 안의 dnsmasq가 응답한다.
- dnsmasq가 실행 중이어도 요청이 중간의 방화벽 규칙에서 막힐 수 있다. 요청 도착 여부와 INPUT 규칙의 카운터를 함께 확인했다.
- 기존 VM은 이미 받은 IP 설정으로 통신할 수 있어, 기존 VM이 정상이라는 사실만으로 새 VM의 DHCP까지 정상이라고 판단할 수 없다.
- 메타데이터는 라우터 또는 DHCP 네임스페이스를 통해 접근한다. 우리 환경은 force_metadata 설정으로 게이트웨이가 있어도 DHCP를 통해 메타데이터 경로를 전달한다.
- 기각한 가설도 기록하면 어떤 구간을 확인했고 왜 다음 지점으로 넘어갔는지 설명할 수 있다.
- 프로브가 검사하는 대상과 조건을 알아야 한다. 기존 VM의 통신이 정상이더라도 새 VM의 DHCP 실패는 별도 검사로 찾아야 한다.

## 2. 받은 증상

- 네트워크 ID는 `ff0d19a8-d633-4baa-8e66-3112d1c4134c`
- VM4에 할당된 내부 IP는 `10.104.0.68`
- VM4의 MAC 주소는 `fa:16:3e:3f:bd:8b`
- VM4의 Neutron 포트 ID는 `e0fa5dca-6ecd-4483-bef4-7fc68114e683`

| 확인한 것 | 명령 | 결과 |
| --- | --- | --- |
| VM 상태와 할당 IP | ops01에서 `openstack server list` | VM1~4 모두 `ACTIVE`. VM4에는 `10.104.0.68`이 표시됨 |
| VM4 포트 | ops01에서 `openstack port list --server cc04-vm4` | 포트는 `ACTIVE`, MAC은 `fa:16:3e:3f:bd:8b` |
| VM4가 실제 설정한 IP | ops01에서 `openstack console log show cc04-vm4` | `soliciting a DHCP lease` 이후 IPv4LL 주소 `169.254.141.114`를 설정한 기록 확인. 네트워크 정보에도 이 주소가 표시됨 |
| 기존 VM의 IP 설정 | ops01에서 `openstack console log show cc04-vm1` | VM1은 `10.104.0.52`와 기본 게이트웨이 `10.104.0.1`을 설정한 기록 확인 |
| VM1 → VM3 | ops01에서 `ssh -o ConnectTimeout=6 -i ~/cc04-key.pem -o IdentitiesOnly=yes -J cc04@192.168.0.12 cirros@203.0.113.104 'ping -c 2 10.104.0.46'` | 2개 송신, 2개 응답 |
| VM1 → VM2 | ops01에서 `ssh -o ConnectTimeout=6 -i ~/cc04-key.pem -o IdentitiesOnly=yes -J cc04@192.168.0.12 cirros@203.0.113.104 'ping -c 2 10.104.0.88'` | 2개 송신, 2개 응답 |
| VM1 → VM4 | ops01에서 `ssh -o ConnectTimeout=6 -i ~/cc04-key.pem -o IdentitiesOnly=yes -J cc04@192.168.0.12 cirros@203.0.113.104 'ping -c 2 10.104.0.68'` | 2개 송신, 응답 없음 |
| 라우터 → VM4 | net01에서 `sudo ip netns exec qrouter-89c8239b-94db-4284-aa6e-4b24ebb1bc7f ping -c 2 -W 2 10.104.0.68` | 2개 송신, 응답 없음 |

VM4에는 FIP가 없었다. 이번 확인에서는 VM4의 내부 ping과 DHCP 실패를 확인했다.

## 3. 가설과 검증 기록

| # | 가설 | 확인 방법 (명령)                                                                                                                                                                                                                                                                                                                                                                                                        | 결과 | 판단 |
| --- | --- |-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------| --- | --- |
| 1 | VM4에 IP가 할당되지 않았거나 SG에서 통신을 막고 있을 수 있다. | VM4에 설정된 SSH 키, 보안 그룹, 할당 IP를 확인하고, 네트워크 포트 상태와 보안 그룹의 통신 허용 규칙을 조회하는 명령어<br/> <br/>ops01에서 `openstack server show cc04-vm4 -c key_name -c security_groups -c addresses`<br>`openstack port list --server cc04-vm4`<br>`openstack security group rule list cc04-sg`                                                                                                                                               | VM4에 IP와 MAC이 할당돼 있고 키는 `cc04-key`, SG는 `cc04-sg`였다. SG에는 ICMP와 TCP 22 허용 규칙이 있었다. 하지만 콘솔 로그에서는 할당된 IP 대신 링크 로컬 주소를 사용했다. | IP 미할당이나 ICMP·SSH 허용 규칙 누락 가설은 기각. VM이 DHCP 설정을 받는 과정을 확인했다. |
| 2 | DHCP가 꺼져 있거나 dnsmasq에 VM4 정보가 없을 수 있다. | 서브넷의 DHCP 활성화 여부, IP 할당 범위와 게이트웨이를 확인하고, DHCP 서버가 요청을 받을 준비가 돼 있는지와 VM4의 IP 예약 정보가 등록돼 있는지 조회하는 명령어<br/> <br/>ops01에서 `openstack subnet show cc04-subnet -c enable_dhcp -c allocation_pools -c gateway_ip`<br>net01에서 `sudo ip netns exec qdhcp-ff0d19a8-d633-4baa-8e66-3112d1c4134c ss -ltunp`<br>`sudo docker exec neutron_dhcp_agent cat /var/lib/neutron/kolla/dhcp/ff0d19a8-d633-4baa-8e66-3112d1c4134c/host` | DHCP는 켜져 있었다. dnsmasq가 UDP 67번을 듣고 있었고, host 파일에 VM4의 MAC과 `10.104.0.68`이 등록돼 있었다. | 기각. DHCP 서버와 예약 정보는 있으므로 요청이 도착해 처리되는지 확인했다. |
| 3 | DHCP 요청이 qdhcp에 도착해도 INPUT 방화벽에서 버려질 수 있다. | VM4의 DHCP 요청이 서버까지 도착하는지 패킷을 캡처하고, 방화벽 규칙과 차단된 패킷 수의 변화를 확인하는 명령어를 사용<br/> <br/>net01에서 아래 tcpdump로 VM4의 DHCP 요청을 관찰했다. 캡처 전후에 `sudo ip netns exec qdhcp-ff0d19a8-d633-4baa-8e66-3112d1c4134c iptables -L INPUT -n -v --line-numbers`를 실행하고, `sudo ip netns exec qdhcp-ff0d19a8-d633-4baa-8e66-3112d1c4134c iptables -S INPUT`으로 조건을 확인했다.                                                                             | VM4의 MAC에서 나온 DHCP 요청 1개가 qdhcp의 tap에 도착했다. 기존 VM1~3의 MAC만 먼저 허용하고 나머지 UDP 67 요청을 DROP하는 규칙이 있었다. 해당 DROP 카운터는 `5358 → 5359`로 증가했고, VM4로 향하는 응답은 캡처에서 보이지 않았다. | 원인 확인. 요청은 qdhcp까지 왔지만 dnsmasq에 전달되기 전 INPUT 규칙에서 차단됐다. |




확인한 경로를 정리하면 다음과 같다.

```text
VM4의 DHCP 요청
    → net01 qdhcp의 tap까지 도착
    → INPUT의 UDP 67 DROP 규칙에서 차단
    → dnsmasq가 요청을 처리하기 전에 중단
```

## 4. 원인과 복구 방안

- **원인 (한 문장)** — net01의 qdhcp 네임스페이스에서 기존 VM1~3의 MAC만 허용한 뒤 나머지 UDP 67 요청을 버리는 INPUT 규칙이 VM4의 DHCP 요청을 차단했다.
- **끊긴 지점이 이 증상으로 보인 이유** — Neutron은 VM4의 포트에 `10.104.0.68`을 할당했지만, VM4는 DHCP 응답을 받지 못해 링크 로컬 주소를 설정했다. 그래서 Horizon에는 ACTIVE와 IP가 표시돼도 해당 내부 IP로 통신할 수 없었다. 기존 VM1~3은 이미 IP가 설정돼 있고, DHCP 요청도 MAC별 ACCEPT 규칙에 포함돼 있어 정상으로 보였다.
- **복구하려면 (절차)** — 해당 qdhcp의 INPUT 체인에서 `study-w4a-cc04` 주석이 붙은 규칙을 확인하고 원래 규칙 구성으로 되돌린다. 다른 네트워크나 정상 Neutron 규칙에 영향이 없도록 대상 규칙만 복구한다.
- **복구 후 무엇으로 확인하면 되는지** — VM4의 DHCP 요청에 Offer와 ACK가 돌아오는지 확인한다. 필요하면 허용된 VM4 재부팅으로 다시 확인하고, 게스트에 `10.104.0.68`이 설정됐는지 확인한다. 이후 라우터와 VM1에서 VM4로 ping을 보내고, 접근 가능한 내부 경로에서 키 SSH도 확인한다.

## 5. SRE 세 질문

1. **사용자가 무엇을 겪었나** — 새 VM4는 ACTIVE이고 IP도 표시됐지만 할당된 내부 IP로 접근할 수 없었다. 기존 VM1~3의 통신은 정상적으로 동작했다.
2. **어떤 증상으로 알아챘나** — VM1에서 VM2와 VM3로는 ping이 됐지만 VM4로는 실패했다. VM4 콘솔 로그에서 DHCP lease를 받지 못하고 링크 로컬 주소를 설정한 기록을 확인했다. DHCP 요청은 qdhcp에 도착했지만 방화벽 DROP 카운터가 증가했다.
3. **운영자가 미리 볼 수 있었던 지표는** 
    - **Black-box** — 별도 테스트 게스트에서 DHCP 요청 후 정해진 시간 안에 올바른 IP와 게이트웨이를 받는지 검사한다.
    - **White-box** — qdhcp의 DHCP 요청·응답 수, dnsmasq lease 기록, INPUT DROP 카운터를 함께 관찰한다. 기존 경로 프로브가 초록이어도 DHCP 요청만 쌓이고 응답이 없다면 확인이 필요하다.

## 6. 한 줄 회고

- 주차가 늘어나며 조금씩 이해가 빨라지는것같다..