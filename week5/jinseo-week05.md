# 5회차 제출 — 최진서

> 계정 `cc04` / 작성일 2026-11-22

## 1. 읽기 핵심

- 같은 네트워크에 있는 VM끼리 내부 IP로 통신할 때는 가상 라우터의 외부 인터페이스를 거치지 않는다. 그래서 내부 ping이 성공해도 인터넷 연결은 따로 확인해야 한다.
- VM이 외부로 보낸 패킷은 라우터의 내부 인터페이스 `qr`로 들어온다. 라우터는 목적지로 가는 경로를 찾고 외부 인터페이스 `qg`로 패킷을 내보낸다.
- 이때 SNAT 규칙에 따라 출발지 IP가 바뀐다. 
- 반대 방향에서는 외부 호스트가 먼저 ARP로 FIP에 해당하는 MAC 주소를 알아낸다. 패킷이 라우터에 도착하면 DNAT 규칙에 따라 목적지가 VM의 내부 IP로 바뀐다.
- 그래서 이번에는 NAT 규칙이 있는지와 실제 패킷이 그 규칙을 거칠 수 있는지를 나눠 확인했다.
- SRE에서 읽은 대로 정상인 결과도 기록했다. 내부 IP로 SSH와 ping이 된다는 결과 덕분에 외부 통신에만 쓰이는 구간을 먼저 살펴볼 수 있었다.

## 2. 받은 증상

VM 세 대 모두 `ACTIVE`였고 VM끼리는 내부 IP로 통신할 수 있었다. 하지만 세 VM 모두 외부 통신에 실패했고, 외부에서 VM1과 VM2의 FIP로 SSH 접속하는 것도 실패했다.

| VM | 내부 IP | Floating IP |
| --- | --- | --- |
| `cc04-vm1` | `10.104.0.52` | `203.0.113.104` |
| `cc04-vm2` | `10.104.0.88` | `203.0.113.246` |
| `cc04-vm3` | `10.104.0.46` | 없음 |

| 확인한 것 | 실행 위치와 명령 | 결과 |
| --- | --- | --- |
| VM1 → VM2, VM3 | VM1에서 `ping -c 2 -W 2 10.104.0.88`과 `ping -c 2 -W 2 10.104.0.46` | 각각 2개 송신, 2개 응답 |
| VM2, VM3 → VM1 | VM2와 VM3에서 각각 `ping -c 2 -W 2 10.104.0.52` | 각각 2개 송신, 2개 응답 |
| VM → 외부 게이트웨이 | 세 VM에서 각각 `ping -c 2 -W 2 203.0.113.1` | 세 VM 모두 목적지 응답 없음. 내부 라우터 `10.104.0.1`이 `Destination Net Unreachable`을 보냄 |
| VM → 인터넷 IP | 세 VM에서 각각 `ping -c 2 -W 2 8.8.8.8` | 세 VM 모두 목적지 응답 없음. 내부 라우터가 도달 불가 메시지를 보냄 |
| VM → 외부 웹사이트 | 세 VM에서 각각 `curl -s -m 5 -o /dev/null -w '%{http_code}\n' http://example.com` | VM1과 VM2는 `000`을 출력했고 VM3는 응답이 없어 테스트를 종료했다. 세 VM 모두 HTTP 응답을 받지 못함 |
| 외부 → VM1, VM2 FIP | net01에서 `ping -c 2 -W 2 203.0.113.104`와 `ping -c 2 -W 2 203.0.113.246` | 두 FIP 모두 응답 없음. net01의 `203.0.113.1`이 `Destination Host Unreachable`을 보냄 |
| VM1 FIP SSH | ops01에서 `ssh -o ConnectTimeout=5 -i ~/cc04-key.pem -o IdentitiesOnly=yes -J cc04@192.168.0.12 cirros@203.0.113.104 true` | net01에서 FIP에 연결하는 단계에서 `No route to host`로 실패 |
| VM2 FIP SSH | ops01에서 `ssh -o ConnectTimeout=5 -i ~/cc04-key.pem -o IdentitiesOnly=yes -J cc04@192.168.0.12 cirros@203.0.113.246 true` | net01에서 FIP에 연결하는 단계에서 `No route to host`로 실패 |

## 3. 가설과 검증 기록

| # | 가설 | 확인 방법 | 결과 | 판단 |
| --- | --- | --- | --- | --- |
| 1 | VM 자체의 네트워크나 SSH에 문제가 있을 수 있다. | net01의 라우터 네임스페이스에서 세 VM의 내부 IP로 SSH 접속한 뒤, VM끼리 양방향으로 ping을 보냈다. | 세 VM 모두 내부 IP로 SSH 접속에 성공했다. 같은 노드의 VM1과 VM3 사이, 다른 노드의 VM1과 VM2 사이에서 ping도 성공했다. | 기각. 게스트 SSH와 VM 사이의 내부 통신은 동작했다. |
| 2 | FIP 연결이나 NAT 규칙이 빠졌을 수 있다. | ops01에서 `openstack floating ip list`로 FIP 연결을 확인했다. net01에서 `sudo ip netns exec qrouter-89c8239b-94db-4284-aa6e-4b24ebb1bc7f iptables -t nat -S`로 NAT 규칙을 확인했다. | VM1과 VM2의 FIP가 각각 올바른 내부 IP에 연결돼 있었다. 각 FIP의 DNAT와 SNAT 규칙, VM3가 라우터 외부 IP를 쓰도록 하는 SNAT 규칙도 있었다. | 규칙 누락 가설은 기각. 다음으로 패킷이 라우터의 어느 인터페이스까지 가는지 확인했다. |
| 3 | VM의 요청이 라우터에 도착해도 라우터가 외부로 내보내지 못할 수 있다. | net01에서 아래 명령으로 라우터 인터페이스와 경로를 조회하고, VM의 ping에 맞춰 `tcpdump`로 패킷을 확인했다. | 내부 인터페이스 `qr`에 VM의 요청이 도착했다. 라우터는 도달 불가 메시지를 돌려보냈다. 외부 인터페이스 `qg`는 `DOWN`이었고 외부망 경로와 기본 경로도 없었다. | 확인. VM에서 라우터까지 도착한 요청이 외부로 나가는 단계에서 멈췄다. |

**라우터의 외부 경로 확인**

net01에서 `cc04` 라우터의 외부 인터페이스 상태, 라우팅 테이블, `8.8.8.8`로 나가는 경로를 확인했다.

```bash
sudo ip netns exec qrouter-89c8239b-94db-4284-aa6e-4b24ebb1bc7f ip link show qg-12c55304-e6
sudo ip netns exec qrouter-89c8239b-94db-4284-aa6e-4b24ebb1bc7f ip -4 route
sudo ip netns exec qrouter-89c8239b-94db-4284-aa6e-4b24ebb1bc7f ip route get 8.8.8.8
```

`qg-12c55304-e6`에는 `UP` 플래그가 없었고 `state DOWN`으로 표시됐다. 라우팅 테이블에는 내부망 `10.104.0.0/24` 경로만 있었다. `8.8.8.8`로 가는 경로를 조회했을 때도 `Network is unreachable`이 나왔다.

**VM에서 외부로 나가는 방향**

```bash
# net01에서 VM2의 요청과 라우터 응답 확인
sudo ip netns exec qrouter-89c8239b-94db-4284-aa6e-4b24ebb1bc7f timeout 14s tcpdump -l -nn -i qr-f0d253a5-87 -c 8 'icmp and host 10.104.0.88'

# net01에서 FIP가 없는 VM3도 같은 방식으로 확인
sudo ip netns exec qrouter-89c8239b-94db-4284-aa6e-4b24ebb1bc7f timeout 12s tcpdump -l -nn -i qr-f0d253a5-87 -c 6 'icmp and host 10.104.0.46'
```

VM2와 VM3이 `8.8.8.8`로 보낸 요청은 모두 내부 인터페이스 `qr`에 도착했다.

-> VM에서 라우터까지는 통신 가능

그다음 같은 인터페이스에서 라우터 `10.104.0.1`이 각 VM에 도달 불가 메시지를 돌려보내는 것도 확인했다. 

-> “외부로 보낼 길이 없다”고 응답. 장애 범위를 "라우터 → 외부 구간"으로 좁힘.

외부 인터페이스 `qg`에서는 `That device is not up`으로 캡처 자체가 시작되지 않아 SNAT 이후의 패킷은 관찰하지 못했다.

**외부에서 VM의 FIP로 들어오는 방향**

```bash
# net01에서 두 FIP에 대한 ARP 요청과 응답 확인
sudo timeout 14s tcpdump -l -nne -i veth-gw -c 10 '(host 203.0.113.104 or host 203.0.113.246) and (arp or icmp or tcp port 22)'
```

net01의 `veth-gw`에서 두 FIP의 MAC 주소를 묻는 ARP 요청을 확인했다. 캡처하는 동안 ARP 응답은 없었다. 외부 호스트가 FIP로 패킷을 보내는 데 필요한 MAC 주소를 얻지 못해 라우터의 DNAT 단계까지 진행하지 못했다.


-> 정상이라면 가상 라우터는 FIP에 대한 ARP 요청에 외부 인터페이스 qg의 MAC 주소로 응답한다. 이번에는 qg가 DOWN 상태였고, net01에서 두 FIP에 대한 ARP 요청만 관찰됐다. 따라서 외부에서 들어오는 통신은 ARP 단계에서 막혔고, ping과 SSH 패킷은 DNAT가 수행되는 라우터까지 도달하지 못했다.
## 4. 원인과 복구 방안

- **원인** — `cc04-router`의 외부 인터페이스 `qg-12c55304-e6`가 관리상 `DOWN`이었고, 라우터에 외부망 경로와 기본 경로가 없었다.
- **이 증상으로 보인 이유** — 같은 내부망의 VM끼리는 `qg`를 거치지 않아 통신할 수 있었다. 반면 세 VM이 외부로 나갈 때는 모두 이 인터페이스가 필요해서 FIP 유무와 관계없이 실패했다. 외부에서 접속할 때도 FIP에 대한 ARP 응답을 얻지 못해 SSH 연결이 시작되지 않았다.
- **복구하려면** — `cc04-router`의 `qg-12c55304-e6`를 활성화한 뒤, 외부망 `203.0.113.0/24` 경로와 `203.0.113.1`을 통한 기본 경로를 확인한다. 빠진 경로가 있으면 해당 라우터의 정상 설정에 맞춰 복원한다. 
- **복구 후 확인** — FIP가 있는 VM2와 없는 VM3에서 각각 외부 게이트웨이, `8.8.8.8`, `example.com` 순서로 통신을 확인한다. 반대 방향은 net01에서 VM1과 VM2의 FIP로 ping과 SSH가 되는지 확인한다.

## 5. SRE 세 질문

1. **사용자가 무엇을 겪었나?** VM끼리는 내부 IP로 통신할 수 있었지만 VM에서 인터넷을 사용할 수 없었다. 외부에서 VM1과 VM2의 FIP로 SSH 접속하는 것도 실패했다.
2. **어떤 증상으로 알아챘나?** VM끼리 보낸 ping에는 응답이 왔다. 외부로 보낸 ping에는 내부 라우터 `10.104.0.1`이 직접 `Destination Net Unreachable`을 돌려줬다. 두 결과를 비교해 라우터의 외부 경로를 확인했다.
3. **운영자가 미리 볼 수 있었던 지표는?** 외부에서 VM의 FIP로 보내는 ping과 VM 안에서 외부로 보내는 ping 및 HTTP 요청의 성공 여부를 각각 측정하면 어느 방향의 통신이 실패하는지 알 수 있다(블랙박스). 라우터 내부에서는 외부 인터페이스 qg가 UP인지, 외부로 나가는 기본 경로가 있는지 확인할 수 있다(화이트박스).

## 6. 한 줄 회고

네트워크 쪽은 봐도 봐도 항상 새롭고 어려운 것 같다...