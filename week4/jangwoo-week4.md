# 4회차 제출 — 정장우

> 계정: `test` · 작성일: 2026-11-16
> 

## 1. 읽기 핵심 (한 장)

- **포트와 VM 연결**
    - Neutron이 VM 부팅 전에 포트를 만들고 IP·MAC을 DB에 저장 → DHCP는 그 값을 게스트에 **전달**할 뿐이라, Horizon의 ACTIVE·IP와 게스트가 실제로 받은 값은 따로 확인해야 함
    - 포트의 IP·MAC과 다른 소스 트래픽은 컴퓨트 노드 iptables에서 조용히 DROP → 패킷이 **포트·장치 단위로** 말없이 버려질 수 있음
- **DHCP와 메타데이터**
    - DHCP: `qdhcp-<네트워크 ID>` 네임스페이스의 dnsmasq가 DISCOVER → OFFER → REQUEST → ACK로 주소와 옵션(경로·DNS·MTU)을 줌
    - 메타데이터 경로는 책 기준 두 가지
        - 라우터 경로(p.159–160): 게이트웨이가 Neutron 라우터면 qrouter PREROUTING이 80 → 9697로 REDIRECT, 9697 proxy가 받음
        - DHCP 경로(p.160–161): `enable_isolated_metadata = True` **이고 서브넷에 게이트웨이가 없을 때만** option 121로 `169.254.169.254 via <DHCP IP>`를 주고, qdhcp의 proxy가 80에서 직접 받음
    - 뒷단은 같음: proxy → metadata agent → Nova metadata API. 공개 키는 **부팅할 때** 여기서 받아 `authorized_keys`에 씀
    - 신고 A, B에 해당
- **MTU**
    - 1500 − VXLAN 50 = 1450이 기준, DHCP agent가 IPv4 VM에 이 값을 전달한다
    - VM의 `eth0` mtu, 네트워크 `mtu`, 경로 장치 mtu 중 하나라도 다르면 큰 패킷만 막힐 수 있기에 장치별 mtu를 비교
- **SRE**
    - 12장: 기각한 가설도 결과다. 남겨야 같은 길을 다시 안 가고, 원인 범위가 좁혀짐
    - 6장: 블랙박스 프로브는 **자기가 지나는 길만** 본다. 프로브가 안 지나는 경로는 화이트박스 신호로 메워야 함

vm4의 메타데이터 요청이 지나는 경로

```
vm4(eth0) ─(169.254.169.254 via 10.99.0.2)→ tap → qbr → qvb ─ qvo → br-int → VXLAN
  → net01 br-int → qdhcp tap → haproxy:80 → metadata agent → Nova metadata API   (vm1과 비교)
```

## 2. 받은 증상

**내가 받은 신고: B**

신고 판별 근거: `openstack server list`에 `test-vm4`가 있음 (A·B 대상). DHCP로 IP를 받았고 비밀번호 로그인이 되며 키 인증만 실패 → 신고 B

| 확인한 것 | 명령 | 결과 |
| --- | --- | --- |
| 서버 목록, vm4 존재 | `openstack server list` | test-vm1~3 ACTIVE, **test-vm4 ACTIVE** (10.99.0.139, FIP 203.0.113.116, cmp02) → 신고 B |
| vm4 키 SSH | net01 경유 `ssh -i test-key.pem cirros@203.0.113.116` | `Permission denied (publickey,password)` |
| 같은 키로 vm1 | net01 경유 `ssh -i test-key.pem cirros@203.0.113.245` | 접속 성공 |
| vm4 비밀번호 로그인 | `cirros` / `gocubsgo` | 로그인 성공 |
| vm4 FIP ping | net01에서 `ping -c2 203.0.113.116` | 2 received |

## 3. 가설 · 검증 기록 ★

**가장 중요한 부분입니다.** 순서대로, 틀린 가설도 지우지 말고 남기세요.

| # | 가설 (어디가 문제일 것 같다) | 확인 방법 (명령) | 결과 | 판단 |
| --- | --- | --- | --- | --- |
| 1 | vm4에 키가 지정되지 않았거나 키 파일이 다르다 | ops01에서 `openstack server show`(vm1, vm4) | 둘 다 `key_name test-key`, flavor `m1.study`, image `cirros`, SG `test-sg` 같음. 같은 키 파일로 vm1은 접속됨 | 기각 |
| 2 | SG·FIP 때문에 sshd까지 못 간다 | `openstack security group rule list test-sg`, 비밀번호 SSH | tcp 22 ingress·egress 전체 허용. `Permission denied`는 TCP 22 연결과 sshd 응답까지 성공했다는 뜻이고, 비밀번호 로그인도 됨 | 기각 |
| 3 | DHCP가 실패했다 | net01 `sudo tail /var/log/kolla/neutron/dnsmasq.log`(vm4 MAC), 콘솔 로그 `if-info` | `DHCPDISCOVER → OFFER → REQUEST → ACK 10.99.0.139 fa:16:3e:a6:2d:52`. 게스트 `eth0 10.99.0.139/24` | 기각 |
| 4 | 게스트가 부팅 때 키를 받지 못했다 | `openstack console log show`(vm4, vm1), vm4 안 `ls ~/.ssh` | vm4: `failed 1/20 … failed 20/20`, `failed to read iid from metadata`, **`=== datasource: None None ===`**, `~/.ssh` 없음. vm1: `successful after 5/20 tries`, **`=== datasource: ec2 net ===`**, `authorized_keys` 1줄 | 유지 |
| 5 | 메타데이터 요청이 라우터(qrouter 9697)로 가는데 그쪽이 문제다 | vm4 `ip route`, net01 `cat /var/lib/neutron/kolla/dhcp/<NET>/opts`, qrouter `iptables -t nat -S`, `ss -ltnp` | VM 경로 `169.254.169.254 via 10.99.0.2`(DHCP 포트). opts에 `classless-static-route,169.254.169.254/32,10.99.0.2,0.0.0.0/0,10.99.0.1`. qrouter의 REDIRECT 9697·haproxy는 있지만 VM이 그쪽으로 안 보냄 | 기각 |
| 6 | vm4가 있는 cmp02 경로가 문제다 | cmp02 `ip -o link | grep e37bbb42-c3`, `sudo tcpdump -ni tape37bbb42-c3 -c 10 'tcp port 80'`, vm4 `ping 169.254.169.254` | 장치 4개 UP·mtu 1450, SYN이 tap에서 정상적으로 나감. `ping 169.254.169.254` 2/2. vm2(cmp02)는 정상 | 기각 |
| 7 | 메타데이터 proxy나 agent가 죽었다 | `sudo ip netns exec qdhcp-<NET> ss -ltunp`, `openstack network agent list` | `169.254.169.254:80` haproxy LISTEN, `0.0.0.0:67` dnsmasq. Metadata agent `:-) UP` | 기각 |
| 8 | 요청이 qdhcp까지 오지만 proxy에 닿기 전에 버려진다 | vm4에서 curl 하면서 `sudo ip netns exec qdhcp-<NET> tcpdump -ni tap161d6fed-c5 -c 20 'port 80'`, `iptables -L INPUT -n -v --line-numbers` 두 번 | **SYN만 4번, SYN-ACK 없음**. INPUT 1번 `DROP tcp dpt:80 → 169.254.169.254 /* study */` 카운터 **534 → 538**(SYN 수와 같음). 2번 `neutron-dhcp-age-INPUT`은 539 그대로 | 유지 |
| 9 | vm4에만 생긴 문제다 | vm1에서 `curl -m 3 http://169.254.169.254/latest/meta-data/` | vm1도 **rc=28(타임아웃)**. 그런데 vm1 키 SSH는 됨 | 기각 → 네트워크 전체가 막힘. 기존 VM은 첫 부팅 때 받은 키로 버티는 중 |

## 4. 원인 · 복구 방안

복구는 **실행하지 않습니다.** 어떻게 고치면 될지만 적습니다.

- **원인 (한 문장):** net01 `qdhcp-<NET>` 네임스페이스 INPUT 체인 맨 앞에 `169.254.169.254:80/tcp` DROP 규칙이 있어서, VM의 메타데이터 요청이 haproxy에 닿기 전에 버려지고 새로 부팅한 vm4가 공개 키를 받지 못했다.
- **끊긴 지점이 이 증상으로 보인 이유:**
    - 마지막으로 확인된 지점은 qdhcp의 `tap161d6fed-c5`(SYN 도착)이고, 처음 사라진 지점은 그 바로 뒤 INPUT 1번 규칙이다. haproxy는 듣고 있었지만 요청을 받지 못했다.
    - 기존 vm1~3는 첫 부팅 때 이미 키를 받아 `authorized_keys`에 써 두었다. SSH는 이 파일만 보기 때문에 지금 메타데이터가 막혀 있어도(vm1 curl rc=28) 접속된다. 그래서 장애 뒤에 부팅한 vm4에서만 증상이 드러났다.
    - DHCP(UDP 67)와 ICMP는 규칙에 걸리지 않아 IP도 받고 ping도 된다. 메시지도 인증 단계의 `Permission denied`라 키 파일 문제처럼 보인다.
    - DROP은 응답이 없어서 cirros가 시도마다 타임아웃까지 기다렸다(약 49초 × 20회). 그래서 vm4는 login까지 13분쯤 걸렸다.
    - 확정 근거는 qdhcp tap의 SYN만 있는 tcpdump, 그리고 같은 순간 SYN 수만큼 오른 DROP 카운터
- **책의 설명과 우리 환경의 차이:**
    - 책은 DHCP 경로가 "isolated metadata + 게이트웨이 없는 서브넷"일 때만 쓰인다고 하지만, 우리는 `force_metadata = true`라서 라우터 게이트웨이(10.99.0.1)가 있어도 DHCP가 option 121로 `169.254.169.254 via 10.99.0.2`를 준다. 그래서 라우터 경로(9697)는 살아 있어도 쓰이지 않는다.
    - 책의 `neutron-ns-metadata-proxy` → 우리 환경은 `haproxy -f /var/lib/neutron/kolla/ns-metadata-proxy/<ID>.conf`
    - DHCP 파일 경로는 `/var/lib/neutron/dhcp/<NET>/`가 아니라 `/var/lib/neutron/kolla/dhcp/<NET>/`
- **복구하려면 (명령 또는 절차):**
    
    net01에서 원인 규칙 1줄만 지운다: `sudo ip netns exec qdhcp-a54b43af-b91e-409d-aee0-afcc778781ab iptables -D INPUT -d 169.254.169.254/32 -p tcp -m tcp --dport 80 -m comment --comment study -j DROP`. 이어서 장애 동안 부팅한 vm4를 재부팅한다(키는 부팅할 때 받으므로). 에이전트가 관리하는 `neutron-dhcp-age-INPUT`, DHCP 에이전트, vm1~3는 건드리지 않는다. 장애로 추가된 규칙과 그동안 부팅한 VM만 되돌리는 방법이다.
    
- **복구 후 무엇으로 확인하면 되는지:**
    - net01에서 `sudo ip netns exec qdhcp-<NET> iptables -S INPUT`에 `-A INPUT -j neutron-dhcp-age-INPUT`만 남음
    - vm1에서 `curl -m 3 http://169.254.169.254/latest/meta-data/`가 목록을 돌려줌(rc=0)
    - vm4 재부팅 후 콘솔 로그에 `=== datasource: ec2 net ===`, 키로 SSH 접속 성공

## 5. SRE 세 질문

1. **사용자가 무엇을 겪었나:**
    - 새로 만든 vm4가 ACTIVE이고 IP·ping도 되는데 키 SSH만 `Permission denied`. 비밀번호로는 들어가지고, 같은 키로 vm1~3는 정상
2. **어떤 증상으로 알아챘나:** vm4 콘솔 로그의 `datasource: None None`과 `~/.ssh`가 없다는 것. vm4 안에서 `curl http://169.254.169.254/` rc=28
3. **운영자가 미리 볼 수 있었던 지표는:**
    - 기존 프로브는 이 문제를 못 잡았다. 이유는 두 가지다.
    ① 프로브(`slot_probe.py`)는 슬롯의 **vm1~3만** 대상으로 삼아서 새 VM(vm4)은 보지 않는다.
    ② 키 SSH 프로브(`2 FIP SSH`)는 vm1·vm2가 첫 부팅 때 받아 둔 키로 성공해서 계속 초록이다. 메타데이터를 직접 묻는 프로브는 없다.
    - **Black-box 제안:** 기존 vm1에서 주기적으로 `curl -m 3 http://169.254.169.254/latest/meta-data/instance-id` (새 VM을 안 만들어도 장애 즉시 실패)
    - **White-box 제안:** 네트워크별 메타데이터 요청 수(haproxy·metadata agent)를 VM 부팅 이벤트와 비교해, 부팅은 있는데 요청이 0이면 경보

## 6. 한 줄 회고

- 키 오류처럼 보이는 메시지가 실제로는 부팅 때 사라진 HTTP 요청 하나였다. "기존 VM은 왜 정상인가"의 답을 지금이 아니라 첫 부팅 시점에서 찾아야 한다는 게 이번 주 배운 점이다.