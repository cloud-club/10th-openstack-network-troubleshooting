## 슬롯 경로 프로브 설명서

Grafana `Probes` 폴더 → **슬롯 경로 프로브** 대시보드 (`localhost:3001/d/slot-probe`)

### 1. 프로브가 뭔가요?

밖에서 주기적으로 찔러 보고 **응답이 오는지만** 확인하는 자동 점검입니다. 사람이 1분마다 `ping`, `ssh`를 해 보는 일을 프로그램이 대신 합니다.

- **블랙박스 감시**: 내부 상태(flow, 카운터, 로그)는 보지 않고 "사용자 입장에서 되는가"만 봅니다. → 증상과 시작 시각
- **화이트박스 감시**: 내부 지표를 직접 봅니다. → 원인 범위

대시보드는 위에서 아래로 블랙박스(경로 프로브) → 화이트박스(노드 데이터플레인) 순서입니다. 증상을 먼저 잡고 원인을 좁혀 가세요.

### 2. 무엇을 재나요?

모든 슬롯(`study-*` 프로젝트)을 자동으로 찾아서, 1분마다 슬롯별로 아래 경로를 확인합니다. 회차가 바뀌어도 그대로 쓰입니다.

| # | 프로브 | 출발 → 도착 | 보는 구간 |
|---|---|---|---|
| 1 | FIP ICMP | net01 → vm1·vm2 FIP | 외부 → VM (DNAT, net01↔컴퓨트 터널) |
| 2 | FIP SSH | net01 → vm1·vm2 FIP TCP 22 | 1번과 비교: ICMP만 문제인지 |
| 3 | FIP MTU1400 | net01 → vm2 FIP, 데이터 1400B + DF | 큰 패킷만 막히는지 |
| 4 | qrouter | net01 qrouter → vm1·vm2 내부 IP | NAT를 뺀 net01↔컴퓨트 |
| 5 | E-W 같은 노드 | vm1 → vm3 | 한 노드 안 (br-int) |
| 6 | E-W 다른 노드 | vm1 → vm2 | 컴퓨트 노드 사이 터널 |
| 7 | E-W 반대 방향 | vm2 → vm1 | 같은 터널, 반대 방향 |

- VM이 어느 노드에 있는지는 대시보드의 **VM 배치** 표에 나옵니다.
- 5~7번은 vm1·vm2 안에 SSH로 들어가서 ping을 보냅니다.

### 3. 색 읽는 법

| 색 | 값 | 뜻 |
|---|---|---|
| 🟩 OK | 1 | 경로가 됩니다 |
| 🟥 FAIL | 0 | 경로가 끊겼습니다 |
| ⬜ 측정 불가 | -1 | 출발점 VM에 SSH로 들어가지 못해 잴 수 없습니다. **실패와 다릅니다** |
| 🟦 일시정지 | -2 | 운영자가 그 슬롯 프로브를 멈췄습니다. 그동안은 기록이 없습니다 |

### 4. 실습할 때 주의할 점

1. **반영까지 최대 2~3분** — 프로브 주기 1분 + Prometheus 수집 1분. 장애 주입·복구 직후에는 아직 안 바뀝니다.
2. **프로브 패킷도 카운터에 잡힙니다** — 프로브마다 1분에 ping 2개를 보냅니다. flow `n_packets`나 인터페이스 카운터가 내 ping 말고도 늘어납니다.
   - 전후 측정은 짧은 간격으로 하세요.
   - 숫자를 정확히 맞춰야 하면 운영자에게 **일시정지**를 요청하세요 (아래 6번).
3. **tcpdump에서 구분하기** — 프로브 ping은 `-s 120`이라 **`length 128`**, MTU 프로브는 `length 1408`로 보입니다. 내 ping은 기본 크기(`length 64`)로 보내면 구분됩니다.
4. **짧은 끊김은 놓칠 수 있습니다** — 1분에 2번만 잽니다.
5. **모든 슬롯이 한꺼번에 회색**이면 장애가 아니라 프로버 문제일 수 있습니다. **프로버 상태** 패널의 "마지막 실행 후 경과"가 3분을 넘으면 운영자에게 알려 주세요.

### 5. 학생이 할 수 없는 것

프로브 설정, 일시정지, 재시작은 운영자만 할 수 있습니다 (학생 계정은 Grafana Viewer). 필요하면 운영자에게 요청하세요.

---

## 운영자용

### 구성

| 구성 요소 | 위치 |
|---|---|
| 프로브 서비스 | ops01 `slot-probe.service` (systemd, `ubuntu`) → `~/s2-env/slot-probe/slot_probe.py` |
| 지표 | `http://192.168.0.10:9199/metrics` (`slot_probe_success`, `slot_probe_rtt_seconds`, `slot_probe_paused`, `slot_probe_vm_info` …) |
| Prometheus | `/etc/kolla/config/prometheus/prometheus.yml.d/10-slot-probe.yml` (job `slot_probe`) |
| Grafana | 폴더 uid `probes`, 대시보드 uid `slot-probe` (원본 `~/s2-env/slot-probe/dashboard.json`) |

- 슬롯 탐지 조건: `study-<slot>` 프로젝트 안의 `<slot>-vm1~3`, `<slot>-router`, 그리고 ops01의 `~/<slot>-key.pem`
- 동시 ssh는 4개로 제한합니다. FIP 접속도 net01 sshd를 거치기 때문에, net01 `MaxStartups 10`을 넘기면 학생 접속까지 끊깁니다.

### 일시정지

```bash
cd ~/s2-env/slot-probe
touch pause/cc02     # cc02만 멈춤 (다음 주기부터)
ls pause/            # 멈춘 슬롯 목록
rm pause/cc02        # 재개
```

- 파일을 만든 지 **30분이 지나면 자동으로 지우고 재개합니다.** 끄는 걸 잊어도 감시가 계속 멈춰 있지 않습니다. 더 길게 멈추려면 30분 안에 `touch`를 다시 하세요.
- 멈춘 구간은 대시보드에 🟦 일시정지로 남습니다.

### 상태 확인

```bash
systemctl status slot-probe
journalctl -u slot-probe -n 30          # "출발점 접속 실패" = 그 VM에 SSH 실패(-1)
curl -s 192.168.0.10:9199/metrics | grep -E 'discovery|duration|timestamp'
```

### 수정 · 되돌리기

- 스크립트 수정 후: `sudo systemctl restart slot-probe`
- 대시보드 재등록: Grafana API `POST /api/dashboards/db`에 `dashboard.json`을 `folderUid: probes`, `overwrite: true`로 올립니다.
- 완전히 제거:
  ```bash
  sudo systemctl disable --now slot-probe && sudo rm /etc/systemd/system/slot-probe.service && sudo systemctl daemon-reload
  rm /etc/kolla/config/prometheus/prometheus.yml.d/10-slot-probe.yml
  kolla-ansible reconfigure -i ~/multinode --tags prometheus
  ```
  Grafana의 `Probes` 폴더는 UI에서 지웁니다.

### 알려진 한계

- kolla 기본 `prometheus-blackbox-exporter`는 수집 주소가 잘못 생성돼(ops01:9115로 수집하지만 실제로는 net01에서 실행) 대상 10개가 down입니다. 이 프로브와는 무관합니다.
