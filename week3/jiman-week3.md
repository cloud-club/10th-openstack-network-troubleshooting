# 3회차 읽기 정리 (OVS / flow)

- Denton 4장 PDF p.102 ~ 112 (책 85 ~ 95)
- SRE 6장 PDF p.84 ~ 88 (책 56 ~ 59)

2회차가 qbr/qvb/qvo였으면 이번엔 그 다음. br-int부터 노드 밖으로 나가는 구간.

---

## Denton 4장

### OVS 구성

커널 모듈 / ovs-vswitchd / OVSDB.
- 커널 모듈이 실제 패킷 처리 (ASIC 비유)
- vswitchd는 유저스페이스 데몬, 커널 모듈을 어떻게 돌릴지 결정
- OVSDB는 호스트마다 하나. 브리지·포트 구성이 여기

도구도 둘로 나뉨
- `ovs-vsctl` → OVSDB. 브리지·포트
- `ovs-ofctl` → OpenFlow. flow 규칙

명령이 갈라져 있다는 게 곧 "구성"과 "패킷 처리"가 다른 레이어라는 뜻. vsctl에서 멀쩡해도 ofctl은 다른 얘기를 할 수 있음.

kolla라 둘 다 `sudo docker exec openvswitch_vswitchd ...` 로.

### 가상 장치 5종

tap, Linux bridge, veth, OVS bridge, OVS patch port.
- veth = Linux bridge ↔ OVS bridge (qvb↔qvo)
- patch port = OVS bridge ↔ OVS bridge (patch-tun↔patch-int). 서로 peer 지정
- 케이블 종류 보면 여기가 OVS 영역인지 알 수 있음

### VM → NIC

책(provider 기준)은 9개 장치
tap → qbr → qvb/qvo → br-int → int-br-ethX/phy-br-ethX → br-ethX → ethX

우리는 br-tun
tap → qbr → qvb/qvo → br-int → patch-tun/patch-int → br-tun → NIC

SG 켜져 있으면 VM이 br-int에 직접 안 붙고 qbr 거침. OVS 포트 tap엔 iptables 못 걸어서 생긴 우회 (iptables_hybrid 유지해야 하는 이유).

노드마다 br-int·br-tun 따로. cmp01 br-int와 cmp02 br-int는 서로 모르고 물리망으로만 이어짐.

### ovs-ofctl show

포트 목록 앞 숫자가 OFPort 번호. 책 예시 br-int: 1(patch-tun), 2(int-br-eth1), 5(qvo04c49e4a-a6), LOCAL(br-int).
flow의 in_port=, output: 은 전부 이 번호라서 flow 읽기 전에 대응표부터. qvo 뒤 11자리 = neutron 포트 UUID 앞부분.

### Local VLAN

br-int 포트는 전부 그 호스트 로컬 VLAN을 받음 (`ovs-vsctl show`의 `tag: N`).
- 같은 호스트 + 같은 neutron 네트워크 = 같은 tag
- OVS가 임의 부여, 호스트마다 DB 따로, 물리 VLAN과 무관
- OVS 재시작하면 바뀔 수 있음

→ cmp01 tag 3이라고 cmp02도 3 아님. 노드별로 따로 봐야 함.

### flow 규칙

LinuxBridge는 eth1.30 같은 서브인터페이스로 태깅, OVS는 그런 거 없이 flow로 태그를 바꾸고 떼고 붙임. drop도 가능.
flow 만드는 건 neutron-openvswitch-agent. 그 노드에 해당 네트워크 VM 없으면 flow도 없음.

```
cookie=0x0, duration=6100.376s, table=0, n_packets=15, n_bytes=1904,
idle_age=709, priority=3,in_port=2,dl_vlan=30 actions=mod_vlan_vid:1,NORMAL
```
- n_packets : 이 규칙에 걸린 패킷 수. **패킷이 실제로 이 규칙을 탔다는 유일한 증거**
- priority : 여러 개 매칭되면 높은 게 이김
- actions : mod_vlan_vid, strip_vlan, output, NORMAL, drop ...

책 VLAN 예시 — br-eth1·br-int 양쪽에 같은 패턴 세 줄
1. priority 3~4, in_port=2, dl_vlan=X → mod_vlan_vid:Y, NORMAL (바꿔라)
2. priority 2, in_port=2 → drop (안 걸린 건 버려라)
3. priority 1 → NORMAL (나머지 통과)

"바꿔라 / 아니면 버려라 / 나머지 통과" 사다리가 기본 문법인 듯.

2번 drop은 **정상 구성의 일부**. dump-flows에서 actions=drop 보인다고 찾았다 하면 안 되고 매칭 조건·priority·카운터를 봐야 함. 스터디원들이 grep drop 해서 아무거나 집어올 수 있음 → 세션에서 짚기.

NORMAL = 일반 학습 스위치. MAC 모르면 플러딩, 배우면 FDB.

---

## SRE 6장

왜 모니터링하나 — 추세, 비교, 알림, 대시보드, 사후 디버깅. 핵심은 알림.
- 사람 깨우는 건 비쌈. 잦으면 흘려보기 시작하고 진짜가 소음에 묻힘
- "뭔가 좀 이상한데"로 울리면 안 됨
- 구글은 임계값 학습·인과 자동추론 같은 "마법" 피하고 단순·빠른 모니터링 + 좋은 사후분석 도구
- "DB 느리면 DB 알림, 아니면 웹 알림" 같은 의존성 규칙 거의 안 씀. 인프라가 계속 바뀌어서 유지 안 됨

Symptom vs Cause
- what(뭐가 고장) = 증상, why = 원인
- 예: HTTP 500 / DB 연결 거부, 응답 느림 / 랙 밑에 눌린 케이블(부분 패킷 손실)
- 이 구분이 신호 높고 소음 낮은 모니터링의 핵심

Black-box vs White-box
- Black-box: 밖에서 사용자처럼. 증상 지향, "지금 실제로 안 된다". 호출용 — 진짜 영향 있을 때만 깨움. 아직 안 터진 문제엔 무용
- White-box: 내부(로그, 메트릭, 카운터). 임박한 문제, 재시도에 가려진 실패. 디버깅 필수
- 다층 시스템에선 한 사람의 증상이 다른 사람의 원인. DB SRE에겐 "DB 느림"이 증상, 프론트 SRE에겐 원인
- 웹서버가 보는 DB 속도 + DB가 보는 자기 속도 둘 다 있어야 DB 문제인지 사이 네트워크 문제인지 가름 → **같은 구간을 양끝에서 측정.** tcpdump 양쪽 노드에서 찍는 거랑 같은 얘기

---

## 이번 장애랑

- agent list 전부 :-) 인데 통신 안 됨 → Alive는 white-box 신호, 데이터플레인 증상을 대신 못 함
- ovs-vsctl show 멀쩡 + flow에 drop → "구성이 있다" ≠ "패킷이 간다"
- priority 높은 drop이 끼면 정상 flow는 그대로고 카운터만 멈춤 → ping 전후 n_packets 비교
- Local VLAN 노드별로 다름 → cmp01 값으로 cmp02 flow 찾으면 못 찾음
- 같은 노드(vm1↔vm3)는 br-int 안에서 끝남 → 되면 cmp01 br-int까지 배제

세션 질문
1. agent list가 다 :-)인데 통신이 안 된다. 이 명령은 뭘 증명하고 뭘 못 하나
2. cmp01 tag 3, cmp02 tag 5. 같은 네트워크라는 건 뭘로 확인하나
3. actions=drop 보였다. 바로 범인인가. 더 봐야 할 세 가지는
4. 신고 전에 잡으려면 어떤 프로브가 필요한가
