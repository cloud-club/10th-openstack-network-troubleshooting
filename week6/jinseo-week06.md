# 6회차 제출 — 최진서

## 1. 읽기 핵심

- 시큐리티 그룹은 VM의 Neutron 포트에 연결된다. 그룹에 등록한 허용 규칙은 VM이 올라간 컴퓨트 노드의 방화벽 규칙으로 반영된다.
- 그래서 OpenStack에 SSH 허용 규칙이 보이면, 실제로 VM이 있는 노드에서도 그 규칙까지 패킷이 도달하는지 확인해야 한다.
- 우리 환경에서 VM으로 들어오는 패킷은 `br-int → qvo → qvb → qbr → tap → VM` 순서로 이동한다. `qbr`를 지날 때 `iptables` 규칙이 적용된다.
- 체인은 규칙을 위에서부터 검사한다. SSH를 허용하는 줄이 있어도 그보다 앞에서 같은 패킷을 `DROP`하면 SSH 요청은 VM에 도착하지 않는다.
- SRE에서 읽은 내용도 이번 증상과 연결됐다. 사용자가 하려는 일이 SSH 접속이라면, ping과 별도로 실제 SSH 접속 성공률을 측정해야 한다.

## 2. 받은 증상

**신고 A였다.** VM1과 VM2 모두 외부에서 보낸 ping에 응답했지만, VM2로 들어가는 SSH만 시간 초과로 실패했다. VM2는 FIP로 접속해도, 라우터에서 내부 IP로 직접 접속해도 같은 결과였다.

| VM | 컴퓨트 노드 | 내부 IP | FIP | Neutron 포트 ID |
| --- | --- | --- | --- | --- |
| VM1 | `cmp01` | `10.104.0.52` | `203.0.113.104` | `4310a234-1d27-4315-bdbd-d96686a9468d` |
| VM2 | `cmp02` | `10.104.0.88` | `203.0.113.246` | `1ea2b1af-d3cf-4e2b-9bd8-251a77d7d1fe` |
| VM3 | `cmp01` | `10.104.0.46` | 없음 | `b9454d8d-3ac4-48db-94ad-342504f40c12` |

| 확인한 것 | 실행 위치와 명령 | 결과 |
| --- | --- | --- |
| 외부 → VM1 FIP ping | net01에서 `ping -c3 -W1 203.0.113.104` | 3개 송신, 3개 응답 |
| 외부 → VM2 FIP ping | net01에서 `ping -c3 -W1 203.0.113.246` | 3개 송신, 3개 응답 |
| 외부 → VM1 FIP SSH | ops01에서 `ssh -o BatchMode=yes -o ConnectTimeout=5 -i ~/cc04-key.pem -o IdentitiesOnly=yes -J cc04@192.168.0.12 cirros@203.0.113.104 true && echo OK` | `OK` |
| 외부 → VM2 FIP SSH | ops01에서 `ssh -o BatchMode=yes -o ConnectTimeout=5 -i ~/cc04-key.pem -o IdentitiesOnly=yes -J cc04@192.168.0.12 cirros@203.0.113.246 true && echo OK` | `Connection timed out during banner exchange` |
| 라우터 → VM1 내부 IP SSH | net01에서 `sudo ip netns exec qrouter-89c8239b-94db-4284-aa6e-4b24ebb1bc7f ssh -o ConnectTimeout=5 cirros@10.104.0.52 'ping -c3 -W2 10.104.0.1; ping -c3 -W2 10.104.0.88; ping -c3 -W2 8.8.8.8'` | SSH 로그인 성공. VM1 안에서 아래 세 가지 ping 실행 |
| 라우터 → VM2 내부 IP SSH | net01에서 `sudo ip netns exec qrouter-89c8239b-94db-4284-aa6e-4b24ebb1bc7f ssh -o ConnectTimeout=5 cirros@10.104.0.88 true` | `port 22: Connection timed out` |
| VM1 → 게이트웨이 | VM1에서 `ping -c3 -W2 10.104.0.1` | 3개 송신, 3개 응답 |
| VM1 → VM2 내부 IP | VM1에서 `ping -c3 -W2 10.104.0.88` | 3개 송신, 3개 응답 |
| VM1 → 인터넷 | VM1에서 `ping -c3 -W2 8.8.8.8` | 3개 송신, 3개 응답 |

## 3. 가설과 검증 기록

| # | 가설 | 확인 방법                                                                                                                                                               | 결과 | 판단                                                                                 |
| --- | --- |---------------------------------------------------------------------------------------------------------------------------------------------------------------------| --- |------------------------------------------------------------------------------------|
| 1 | 지난주처럼 라우터 외부 경로나 FIP 변환에 문제가 있을 수 있다. | net01에서 두 FIP로 ping을 보냈다. FIP를 거치지 않도록 라우터에서도 VM2 내부 IP로 직접 SSH 접속했다.                                                                                               | 두 FIP 모두 ping에 응답했다. VM2의 SSH는 내부 IP로 직접 접속해도 시간 초과였다. VM1에서는 인터넷 ping도 성공했다. | 기각. FIP를 거치지 않는 경로에서도 VM2의 SSH만 실패했다.                                              |
| 2 | VM2 포트에 다른 시큐리티 그룹이 붙었거나 TCP 22번 허용 규칙이 빠졌을 수 있다. | ops01에서 `openstack security group rule list cc04-sg`로 규칙을 조회하고, VM1~3 포트의 `security_group_ids`(포트에 어떤 시큐리티 그룹이 연결)와 `port_security_enabled`(포트의 보안 기능 켜져있는지)를 비교했다. | 세 포트 모두 `cc04-sg`를 사용했고 포트 보안은 `True`였다. 그룹에는 IPv4 TCP 22번과 ICMP 수신 허용, IPv4와 IPv6 송신 허용이 있었다. | 기각. 등록된 그룹과 허용 규칙은 같았다.                                                            |
| 3 | VM2가 있는 `cmp02`에서 SSH 요청이 VM에 전달되기 전에 차단될 수 있다. | 라우터에서 VM2로 SSH 접속을 시도하면서 `cmp02`의 `qvo`, `qvb`, `tap`에서 동시에 캡처했다. 수신 체인의 규칙별 패킷 수도 접속 전후로 비교했다.                                                                     | 같은 SSH 연결 요청이 `qvo`와 `qvb`에서 각각 5개 보였고 `tap`에서는 0개였다. 같은 시간 동안 수신 체인 1번의 TCP 22번 `DROP`은 5개 증가했고, 5번의 SSH 허용 규칙은 그대로였다. | 확인. 마지막으로 요청이 보인 곳은 `qvb`, 처음 보이지 않은 곳은 `tap`이었다. 그사이에서 VM2 수신 체인의 1번 규칙이 요청을 버렸다. |

### 패킷이 어디까지 왔는지 확인

`cmp02`에서 아래 세 장치(`qvo`, `qvb`, `tap`)를 동시에 캡처했다. 라우터 `10.104.0.1`에서 VM2 `10.104.0.88`의 TCP 22번으로 보내는 패킷만 골랐다.

```bash
sudo timeout 15 tcpdump -nnli qvo1ea2b1af-d3 -c 5 'src host 10.104.0.1 and dst host 10.104.0.88 and tcp dst port 22'
sudo timeout 15 tcpdump -nnli qvb1ea2b1af-d3 -c 5 'src host 10.104.0.1 and dst host 10.104.0.88 and tcp dst port 22'
sudo timeout 15 tcpdump -nnli tap1ea2b1af-d3 -c 5 'src host 10.104.0.1 and dst host 10.104.0.88 and tcp dst port 22'
```

`qvo`와 `qvb`에서 출발지 포트 `60360`, TCP 순서 번호 `3316048291`이 같은 `SYN`이 보였다. SSH를 한 번 시도했지만 응답이 없어 같은 요청을 재전송한 것까지 5개가 잡혔다. 같은 시간에 `tap`에서는 이 요청이 하나도 잡히지 않았다.

### 어느 규칙에서 버렸는지 확인

`cmp02`에서 `qvb`와 `tap`이 모두 VM2의 `qbr`에 연결돼 있고, 브리지를 지나는 패킷에 `iptables`를 적용하는 설정이 `1`인 것을 확인했다. 이어서 `FORWARD → neutron-openvswi-FORWARD → neutron-openvswi-sg-chain` 순서로 확인했다.
VM2로 들어가는 SSH 요청에 실제로 어떤 규칙이 적용되는지 찾아보니, VM2 전용 수신 규칙 목록을 확인하면 됐다. ->
`sg-chain`에서 VM2의 `tap`으로 들어가는 패킷은 `neutron-openvswi-i1ea2b1af-d`로 보내고 있었다.

```bash
# cmp02에서 VM2 수신 체인의 규칙 순서와 패킷 수 확인
sudo iptables -L neutron-openvswi-i1ea2b1af-d -nvx --line-numbers

# cmp01에서 정상인 VM1의 수신 규칙과 비교
sudo iptables -S neutron-openvswi-i4310a234-1
```

| OpenStack API 규칙 | VM2가 있는 노드에서 확인한 규칙 | 판단                                                             |
| --- | --- |----------------------------------------------------------------|
| IPv4 수신 TCP 22 허용 | 수신 체인 **5번**에 `tcp dpt:22 RETURN` | 허용 규칙은 있었지만 테스트 전후 카운터가 `1670 → 1670`이었다.                      |
| IPv4 수신 ICMP 허용 | 수신 체인 **6번**에 `icmp RETURN` | ping은 통과.                                                      |
| IPv4 송신 허용 | VM2의 IPv4 송신 체인에 일반 `RETURN` 규칙이 있었다. | 이번 SSH 요청은 VM2의 수신 체인에서 차단됐다.                                  |
| IPv6 송신 허용 | 이번 검증은 IPv4 패킷과 `iptables`를 대상으로 했다. | IPv6 노드 규칙은 별도로 확인하지 않았다.                                      |
| 대응하는 API 규칙 없음 | 수신 체인 **1번**에 `tcp dpt:22 /* study */ DROP` | API와 어긋난 줄이다. 카운터가 `57079 → 57084`로 증가했다. 정상인 VM1에는 이 규칙이 없었다. |


## 4. 원인과 복구 방안

- **원인** — `cmp02`의 VM2 수신 체인 첫 줄에 TCP 22번을 버리는 규칙이 추가돼 있어, SSH 요청이 정상 허용 규칙에 도달하기 전에 차단됐다.
- **왜 이런 증상으로 보였는지** — FIP와 내부 IP 중 어느 쪽으로 접속해도 마지막에는 VM2의 같은 수신 체인을 지난다. 그래서 두 경로 모두 SSH가 실패했다. ping은 ICMP라서 첫 줄에 걸리지 않았고, VM1은 별도 수신 체인을 사용해 SSH도 정상 동작했다.
- **복구하려면** — `neutron-openvswi-i1ea2b1af-d`에서 TCP 목적지 22번, 주석 `study`, 동작 `DROP`인 규칙을 다시 확인한 뒤 해당 규칙 한 줄만 제거한다. 
- **복구 후 확인** — VM2의 FIP와 내부 IP 양쪽으로 SSH 접속을 다시 시도한다. `tap1ea2b1af-d3`까지 TCP 연결 요청이 도착하고 SSH 로그인이 성공하는지 확인한다. 기존에 되던 VM2 ping과 VM1 SSH도 함께 확인한다.

## 5. SRE 세 질문

1. **사용자가 무엇을 겪었나?** VM2는 ping에 응답했지만 SSH로 로그인할 수 없었다. 같은 보안 그룹을 쓰는 VM1에는 접속할 수 있었다.
2. **어떤 증상으로 알아챘나?** 같은 출발지에서 VM1과 VM2에 각각 ping과 SSH를 시도했다. 두 VM 모두 ping에 응답했지만 VM2의 SSH만 시간 초과였다. 라우터에서 내부 IP로 접속해도 같은 증상이 재현됐다.
3. **운영자가 미리 볼 수 있었던 지표는?** 기존 FIP SSH 프로브의 측정값은 VM1이 성공(`1`), VM2가 실패(`0`)였고, FIP ping 프로브는 둘 다 성공이었다. 이번 장애에 맞는 **SLI**는 “net01에서 각 VM의 FIP로 1분마다 SSH 접속을 시도해 `true` 명령까지 5초 안에 성공한 비율”이다. **SLO**는 “최근 7일간 VM별 SSH 성공률 99% 이상”. VM별로 계산해야 VM1의 성공에 VM2의 장애가 가려지지 않고, 한두 번의 일시적인 실패와 지속적인 접속 장애도 구분할 수 있다.

## 6. 한 줄 회고

22번 포트가 열려 있다는 것만 봤는데, 이번에는 그 허용 규칙보다 먼저 어떤 줄을 만나는지까지 봐야 한다는 걸 알게 됐다.