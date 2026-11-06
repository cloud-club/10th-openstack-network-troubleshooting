# 2회차 제출 — Jangwoo

> 계정: `test` · 작성일: 2026-11-02
> 주제: Linux bridge — VM 바로 옆에서 끊기면

## 0. 내 환경 메모

| 항목 | 값 |
| --- | --- |
| 프로젝트 | `study-test` |
| vm1 (정상 비교용) | 내부 IP `10.99.0.117` · FIP `203.0.113.245` |
| vm2 (장애 대상) | 내부 IP `10.99.0.248` · FIP `203.0.113.220` |
| vm2 포트 | `af37a45a-c6f3-4bab-b469-bc7ad4c9b2ae` |
| vm2 MAC | `fa:16:3e:05:8e:61` |
| cmp02 장치 접미사 | 포트 ID 앞 11자리 `af37a45a-c6` |
| 라우터 | `test-router` · `qrouter-3b32d4d1-4d09-4f55-a852-76fb6b12bf05` |

## 1. 개념 및 이론 학습

### 1.1 Linux Bridge와 TAP

Linux bridge는 여러 네트워크 인터페이스를 하나의 L2 구간으로 연결하는 소프트웨어 스위치다. 수신한 Ethernet frame의 출발지 MAC 주소를 학습하고, 목적지 MAC 주소에 해당하는 포트로 프레임을 전달한다. 목적지를 모르면 연결된 포트로 flooding한다.

TAP은 VM의 가상 NIC가 호스트 네트워크와 Ethernet frame을 주고받게 하는 가상 인터페이스다. 예를 들어 VM의 `eth0`에서 전송한 프레임은 호스트의 TAP 장치로 나오고, TAP이 Linux bridge의 포트라면 같은 브리지에 연결된 다른 인터페이스로 전달된다.

인터페이스가 존재하고 UP인 것과 브리지에 연결된 것은 다른 상태다. 예를 들어 TAP과 veth가 모두 UP이어도 한쪽이 bridge port에서 빠져 있으면 두 장치 사이의 L2 전달은 이루어지지 않는다. 따라서 `ip link`의 상태와 함께 `bridge link`의 master 관계를 확인해야 한다.

### 1.2 Open vSwitch와 OVS Hybrid 구조

Open vSwitch(OVS)는 가상화 환경을 위한 소프트웨어 스위치다. Bridge, Port, Interface, Flow Rule을 기반으로 패킷을 전달하며 VLAN, VXLAN 등의 기능을 지원한다. Linux bridge가 기본적인 L2 switching을 제공한다면 OVS는 오버레이 네트워크와 세밀한 flow 처리를 함께 담당할 수 있다.

OVS Hybrid 구성은 OVS와 Linux bridge를 함께 사용한다. OVS가 가상 네트워크의 switching과 터널 연결을 담당하고, VM 가까이의 Linux bridge에서는 iptables 기반 Security Group을 적용한다. 일반적인 패킷 경로는 다음과 같다.

```text
VM eth0
   │
  tap      ← VM의 가상 NIC와 호스트 연결
   │
  qbr      ← VM별 Linux bridge, Security Group 처리
   │
  qvb ─── qvo   ← Linux bridge와 OVS를 잇는 veth pair
            │
          br-int  ← OVS integration bridge
            │
          br-tun  ← VXLAN 터널 연결
            │
         다른 노드
```

| 장치 | 역할 |
| --- | --- |
| `tap` | VM의 vNIC와 호스트 네트워크를 연결한다. |
| `qbr` | VM별 Linux bridge이며 Security Group 처리 경로가 된다. |
| `qvb` | veth pair의 Linux bridge 쪽 끝이다. |
| `qvo` | veth pair의 OVS 쪽 끝이며 br-int에 연결된다. |
| `br-int` | VM 포트와 OVS 내부 경로를 연결하는 통합 브리지다. |
| `br-tun` | VXLAN을 통해 다른 노드의 가상 네트워크와 연결한다. |

veth pair는 한쪽으로 들어온 패킷이 다른 쪽으로 나오는 가상 케이블처럼 동작한다. qvb와 qvo 자체가 정상이어도 qvb가 qbr에 연결되지 않거나 qvo가 br-int에서 빠지면 전체 경로는 끊긴다.

### 1.3 Neutron Port와 장치 이름

Neutron Port는 VM의 가상 NIC에 필요한 MAC 주소, Fixed IP, Security Group, binding 상태 등을 나타내는 논리 리소스다. OVS Hybrid 환경에서는 Port UUID의 앞 11자리가 `tap`, `qbr`, `qvb`, `qvo` 뒤에 공통으로 사용된다.

예를 들어 포트 ID가 `123456789ab…`라면 `tap123456789ab`, `qbr123456789ab`, `qvb123456789ab`, `qvo123456789ab` 형태의 장치를 예상할 수 있다. 따라서 OpenStack API에서 포트 ID를 먼저 구한 다음 컴퓨트 노드에서 같은 접미사의 장치를 추적할 수 있다.

### 1.4 SRE 12장 — Effective Troubleshooting

Troubleshooting의 출발점은 증상과 원인을 구분하는 것이다. 사용자가 말하는 “접속이 안 된다”는 문제 보고이며 원인 자체는 아니다. 같은 SSH timeout도 서버 정지, 잘못된 IP, 방화벽 차단, 라우팅 오류, L2 연결 단절 등 서로 다른 원인으로 발생할 수 있다.

SRE에서 설명하는 기본 흐름은 다음과 같다.

| 단계 | 확인할 내용 | 예시 |
| --- | --- | --- |
| Problem Report | 사용자가 겪은 현상을 구체화한다. | 실패 시각, 대상, 오류 메시지를 확인한다. |
| Triage | 영향 범위와 심각도를 판단한다. | 전체 서버인지 특정 VM 한 대인지 비교한다. |
| Examine | 상태와 데이터를 수집한다. | API 상태, 로그, 메트릭, 패킷을 조회한다. |
| Diagnose | 가설을 검증하고 원인을 좁힌다. | 관찰 결과와 맞지 않는 가설을 기각한다. |

좋은 가설은 검증 가능해야 한다. 예를 들어 “방화벽 문제다”라는 가설을 세웠다면 허용 규칙의 존재, 패킷이 방화벽 지점까지 도착하는지, 다른 프로토콜도 같은 영향을 받는지를 확인할 수 있어야 한다. 예상과 다른 결과가 나오면 가설을 유지하지 않고 기각해야 한다.

가장 익숙하거나 최근 경험한 원인을 먼저 고치는 방식은 확증 편향에 빠질 수 있다. 재부팅이나 설정 변경부터 수행하면 일시적으로 증상이 사라져도 원인을 설명하지 못하고, 기존 증거까지 없어질 수 있다. 따라서 조회와 관찰로 가설을 검증한 뒤 최소한의 복구를 선택하는 것이 안전하다.

Divide and Conquer는 긴 요청 경로의 중간 지점을 확인해 조사 범위를 절반씩 줄이는 방법이다. 예를 들어 `클라이언트 → 라우터 → 터널 → 컴퓨트 노드 → VM` 경로에서 라우터 이후 패킷이 보인다면 클라이언트와 라우터 앞 구간의 우선순위를 낮출 수 있다. 이후 컴퓨트 노드의 OVS와 VM 인접 구간을 다시 나누어 패킷이 사라지는 첫 지점을 찾는다.

### 1.5 현재 프로젝트에 적용할 관점

현재 프로젝트에서는 먼저 정상 VM과 증상 VM을 비교해 공통 경로와 개별 경로를 구분한다. 그다음 OpenStack이 보는 Port·FIP·Security Group 상태를 확인하고, 라우터 네임스페이스와 컴퓨트 노드 순서로 이동한다.

컴퓨트 노드에서는 포트 ID를 기준으로 `tap → qbr → qvb/qvo → br-int → br-tun` 연결을 확인한다. 장치 상태만으로 결론 내리지 않고 `bridge link`, `ovs-vsctl`, `tcpdump`를 함께 사용해 패킷이 마지막으로 관찰되는 지점과 실제 연결 관계가 일치하는지 검증한다.

## 2. 받은 증상

| 확인한 것 | 명령 | 결과 |
| --- | --- | --- |
| vm2 FIP ping | (net01) `ping -c 3 -W 1 203.0.113.220` | 3/3 손실, 100% loss |
| vm2 SSH | `ssh -J ubuntu@192.168.0.12 cirros@203.0.113.220` | `Connection timed out` |
| vm1 FIP ping | (net01) `ping -c 3 -W 1 203.0.113.245` | 3/3 응답 |
| vm2 내부 IP ping | (qrouter) `ping -c 3 -W 1 10.99.0.248` | 3/3 손실 |
| vm1 내부 IP ping | (qrouter) `ping -c 3 -W 1 10.99.0.117` | 3/3 응답 |
| Horizon/API | `openstack server list --long` | vm1·vm2 모두 ACTIVE / Running |

vm1은 같은 라우터와 외부망을 사용하면서 정상이고 vm2만 ICMP와 TCP 22가 함께 실패했다. 따라서 net01 전체나 외부망보다 vm2 전용 경로에서 프로토콜 공통으로 발생하는 L2 단절을 우선 의심했다.

## 3. 가설 · 검증 기록 ★

경로: **ops01 → net01(qrouter) → VXLAN → cmp02(br-tun/br-int) → qvo/qvb → qbr → tap → vm2**

먼저 vm2 자체나 제어면 설정이 잘못되었다고 가정했다. `openstack server list --long`, `port show`, `floating ip list`를 확인한 결과 vm2와 포트는 ACTIVE였고 FIP도 `203.0.113.220 → 10.99.0.248`로 정상 연결되어 있었다. `test-sg`에도 ingress ICMP와 TCP 22 규칙이 있어 서버 상태, FIP, 보안 그룹 가설은 기각했다.

다음으로 공통 외부 경로를 의심해 net01에서 vm1과 vm2의 FIP를 비교했다. vm1은 3/3 응답했지만 vm2만 100% 손실이었다. 같은 라우터의 qrouter에서도 vm1 내부 IP는 응답하고 vm2 내부 IP만 실패했으므로 net01 전체나 FIP NAT보다는 cmp02 이후 vm2 전용 경로에 문제가 있다고 판단했다.

cmp02에서는 포트 ID 앞 11자리인 `af37a45a-c6`로 장치를 찾았다. qbr·qvb·qvo는 모두 UP이었고 tap도 LOWER_UP이었다. qvo는 br-int에 tag 1로 정상 연결되어 있어 장치 소실이나 OVS 연결 문제는 기각했다. 그러나 `bridge link show`에는 tap만 qbr의 멤버로 표시됐고, qvb에는 `master qbraf37a45a-c6` 항목이 없었다. 여기서 qvb의 브리지 분리를 원인 후보로 유지했다.

마지막으로 구조만 보고 결론 내리지 않고 실제 패킷으로 검증했다. qrouter에서 vm2 내부 IP로 ping을 보내며 cmp02에서 tcpdump를 수행하자 요청이 VXLAN, qvo, qvb까지는 보였지만 qbr와 tap에서는 보이지 않았다. 연결 정보와 패킷 단절 지점이 모두 qvb–qbr 사이를 가리켜 원인을 확정했다.

### 패킷 관찰 결과

```text
vxlan_sys_4789  ICMP echo request 10.99.0.1 > 10.99.0.248
qvoaf37a45a-c6 ICMP echo request 10.99.0.1 > 10.99.0.248
qvbaf37a45a-c6 ICMP echo request 10.99.0.1 > 10.99.0.248
qbr / tap      관찰되지 않음
```

VXLAN에서 보였으므로 net01에서 cmp02까지의 터널은 정상이다. qvo와 qvb 양쪽에서 같은 요청이 보였으므로 br-int와 veth pair도 정상이다. 바로 다음 구간인 qbr에서 패킷이 사라졌고, `bridge link`에서도 qvb의 멤버십이 없으므로 두 증거가 같은 지점을 가리킨다.

### 정상 구조와 장애 구조 비교

| 확인 항목 | 정상이라면 | test-vm2 현재 상태 |
| --- | --- | --- |
| qbr 멤버 | qvb와 tap 두 개 | tap 한 개만 존재 |
| qvb link 정보 | `master qbraf37a45a-c6` | master 항목 없음 |
| qvo의 OVS 소속 | br-int | br-int, 정상 |
| qvo VLAN tag | 네트워크 local VLAN | tag 1, 정상 |
| ICMP 요청 | qvo→qvb→qbr→tap | qvo→qvb까지만 관찰 |

## 4. 원인 · 복구 방안

복구는 **실행하지 않습니다.**

- **원인 (한 문장):** cmp02에서 vm2용 `qvbaf37a45a-c6`가 Linux bridge `qbraf37a45a-c6`의 멤버에서 빠져 있어 qvo에서 넘어온 프레임이 qbr와 tap을 거쳐 VM으로 전달되지 못한다.
- **끊긴 지점이 이 증상으로 보인 이유:** qvb–qbr 연결은 ICMP와 TCP보다 아래인 L2 공통 경로이므로 ping과 SSH가 함께 실패한다. Nova는 VM 프로세스가 실행 중이고 Neutron은 논리 포트를 정상으로 보기 때문에 Horizon과 API에는 계속 ACTIVE로 표시될 수 있다.
- **복구하려면 (운영진이 cmp02에서):** `sudo ip link set qvbaf37a45a-c6 master qbraf37a45a-c6`으로 qvb를 원래 브리지에 다시 연결한다. VIF 재연결이나 VM 재부팅도 구조를 재생성할 수 있지만 영향이 더 크므로 직접 멤버십 복구가 우선이다.
- **복구 후 확인:** `bridge link show`에서 qvb와 tap 두 장치가 모두 같은 qbr의 forwarding 멤버인지 확인한다. 이어 qrouter 내부 IP ping, net01 FIP ping, 외부 SSH 순서로 검증하고 tcpdump에서 request와 reply가 tap까지 왕복하는지 확인한다.

## 5. SRE 세 질문

1. **사용자가 무엇을 겪었나:** vm2에 ping과 SSH가 모두 되지 않지만 vm1은 정상이고 화면에는 vm2가 ACTIVE로 보여 실제 서비스 상태와 제어면 상태가 어긋났다.
2. **어떤 증상으로 알아챘나:** vm2 FIP와 내부 IP 모두 100% 손실이고 SSH는 연결 단계에서 timeout이 발생했다. 정상 vm1과 비교해 vm2 한 대의 네트워크 경로로 범위를 줄였다.
3. **운영자가 미리 볼 수 있었던 지표는:** 사용자 경로를 그대로 검사하는 FIP ICMP/TCP 22 블랙박스 프로브가 가장 직접적이다. 추가로 각 VM용 qbr의 멤버가 qvb와 tap 두 개인지 점검하면 API ACTIVE로 드러나지 않는 연결 해제를 조기에 탐지할 수 있다.

## 6. 한 줄 회고 (선택)

- 인터페이스의 UP 여부보다 패킷이 사라지는 첫 지점과 브리지 멤버십을 함께 보는 것이 원인 확정에 더 유효했다.
