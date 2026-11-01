# 2회차 제출 — 손수빈

> 계정: `cc03` · 작성일: 2026-11-01
> 

## 1. 읽기 핵심 (한 장)

- 컴퓨트 노드(cmp) 안에서 패킷들은 **tap → qbr → qvb/qvo → br-int → br-tun 또는 provider 브리지)** → 물리 NIC 순서
    - 밖으로는 net01 라우터에서 FIP NAT 거침
    → vm1은 FIP로 접속되므로 이 경로는 장애 대상이 아닐 거 같음
        
        ```python
        학생 개인 PC (Mac / Windows / Linux)
                  │
                  │ SSH
                  ▼
             ops01 (Bastion)
                  │
                  ├── OpenStack CLI
                  │
                  ├── Horizon / Grafana 접근
                  │
                  └── SSH Jump
                           │
                           ▼
                      net01
                           │
                    ┌──────┴──────┐
                    │             │
                    ▼             ▼
                  cmp01         cmp02
                    │             │
                 VM1, VM3         VM2
        ```
        
    - `tap` + `qbr` + `qvb`/`qvo` : Port와 관련
- 다양한 이름 접두어 → 장치마다 역할이 나뉘어 있어서, **어느 장치까지 패킷이 보이는지**로 끊긴 구간을 좁힐 수 있음
    - 예) tap에서는 보이는데 qvo에서 안 보이면 → 그 사이(qbr/보안 그룹 구간)가 의심 구간
    - 패킷 확인은 각 장치에서 `tcpdump -i <장치>`로
    - `tap`, `qbr`, `qvb`, `qvo` 뒤의 문자열은 **Neutron Port UUID 앞부분 11글자**
        
        → `openstack port show`로 얻은 port ID의 앞 11글자만 알면 그 VM의 장치 이름을 바로 알 수 있음
        
- **분할 정복**: 스택의 한쪽 끝에서 다른 끝으로 한 구성요소씩 보거나, 큰 시스템은 **절반으로 쪼개서(bisection)** 정상인 쪽을 제외해 나감

## 2. 받은 증상

```
VM1 ─ tap ─ qbr ─ qvb/qvo ─ br-int ─ br-tun ─ (노드 간 경로) ─ br-tun ─ br-int ─ qvo/qvb ─ qbr ─ tap ─ VM2
        (cmp01)                                                    (cmp02)
```

| 확인한 것 | 명령 | 결과 |
| --- | --- | --- |
| vm2 SSH | `ssh … cirros@203.0.113.x` | timeout |
| vm1 → vm2 ping | `ping -c 4 10.103.0.x` | 0 received |

## 3. 가설 · 검증 기록 ★

**가장 중요한 부분입니다.** 순서대로, 틀린 가설도 지우지 말고 남기세요.

| # | 가설 (어디가 문제일 것 같다) | 확인 방법 (명령) | 결과 | 판단 |
| --- | --- | --- | --- | --- |
| 1 | 포트 자체가 비정상 | `openstack port list --server cc03-vm2` 
`openstack port show <port ID>` | VM1과 동일 | 기각 |
| 2 | 보안 그룹 정책에서 SSH가 빠졌다 | `openstack security group rule list cc03-sg` | SSH, ICMP 모두 허용 | 기각 |
| 3 | 애초에 노드 간 통신이 끊겼다 | `ping -> ip neigh`  | `REACHABLE`/`STALE` | 기각 |
| 4 | cmp02의 vm2 qvo까지 패킷이 도착하지 않는다 | `sudo timeout 15 tcpdump -ni qvo<port ID> icmp` | 도착함 | 기각 |
| 5 | cmp02의 vm2 tap까지 패킷이 도착하지 않는다 | `sudo timeout 15 tcpdump -ni tap<port ID> icmp` | 도착하지 않음 | 유지 |
| 6 | qvo ~ tap 사이에서 연결이 끊겼다 | `ip -br link show master qbr<port ID>` | qvb가 안 나옴
→ **vm2의 qvb가 qbr 브리지에 붙어 있지 않다** | 유지 |

## 4. 원인 · 복구 방안

복구는 **실행하지 않습니다.** 어떻게 고치면 될지만 적습니다.

- **원인 (한 문장):** cmp02에서 vm2의 `qvb<port ID>`가 `qbr<port ID>` 브리지에서 분리되어, `qvo`까지 온 패킷이 `qbr`과 `tap`으로 넘어가지 못한다.
- **끊긴 지점이 이 증상으로 보인 이유:** 장치는 모두 UP이고 ICMP request가 `qvo`에서는 보이지만 `tap`에서는 보이지 않았다.(ping 보냈을 때 tcpdump로 확인)
- **복구하려면 (명령 또는 절차):** `sudo ip link set qvb<port ID> master qbr<port ID>`
- **복구 후 무엇으로 확인하면 되는지:** `ip -br link show master qbr<port ID>` 에 연결된 tap과 qvb가 모두 나오는지 확인 → ping

## 5. SRE 세 질문

1. **사용자가 무엇을 겪었나:** vm2에 FIP(203.0.113.156)로 SSH, ping 접속이 되지 않음. VM 내부(부팅, DHCP, 메타데이터, sshd)는 콘솔 로그상 정상이었고, 같은 프로젝트의 vm1과 보안 그룹, 포트 설정도 같았음.
2. **어떤 증상으로 알아챘나:** 사용자 신고로 장애 대응 **→** cmp02에서 tcpdump 시 qvo에는 ICMP 요청이 보이는데 tap에는 아무것도 보이지 않음 → `ip -br link show master qbr<port ID>` 로 확인해 보니 qvb와 연결이 끊겨 있었음
3. **운영자가 미리 볼 수 있었던 지표는:** 대시보드에서 확인하지 못함

## 6. 한 줄 회고

- 보안 그룹 정책이 정상이면 qbr는 원인이 아닐 거라 생각했는데 tap과 qvb의 연결 지점(브리지)이기도 하다는 걸 간과한 거 같다. 처음부터 장치 연결 확인을 먼저 했다면 더 빨리 좁혔을 것 같다.