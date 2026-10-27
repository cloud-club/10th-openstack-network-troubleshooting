# 10th-openstack-network-troubleshooting

# CloudClub OpenStack 시즌2 — "ping은 되는데요?"

구축된 OpenStack(Neutron ML2/OVS) 클러스터에서 **매주 한 가지 네트워크 장애를 받아 원인이 어디인지 추적**하는 8주 스터디입니다.
복구는 운영진이 세션에서 합니다. 평가는 **어떤 관찰로 어디까지 좁혀갔는지**입니다.

- 세션: 매주 월요일 22:00 디스코드, 1시간
- 교재: James Denton, *Learning OpenStack Networking* (1판) / Google *SRE* (sre.google)
- 실습 환경 접속: 노션 「실습 환경 접속 가이드」 참고

## 저장소 구조

```
README.md              ← 이 문서 (운영 방식, Git 제출 방법)
template.md            ← 제출 양식 — 복사해서 쓰세요
week1/ … week8/
  README.md            ← 그 주 과제지 (읽기 범위·신고 내용·체크리스트)
  <본인이름>.md         ← 본인 제출물
  solution.md          ← 세션 후 운영진이 올리는 정답·해설
```

## 주간 사이클

| 요일 | 할 일 |
| --- | --- |
| 월 (세션 직후) | 다음 회차 과제지 공개, 각자 환경에 장애 주입됨 |
| 화 ~ 일 | 읽기 + 장애 추적. 체크리스트는 과제지에 처음부터 포함 |
| 일 23:59 | `weekN/<본인이름>.md` PR 제출 |
| 월 세션 | 2~3명 랜덤 발표(10분) → 운영진 정리·정답 공개 → 운영진이 복구 → 다음 장애 |

## 발표 양식 (제출물과 동일)

1. 읽기 핵심 한 장
2. 받은 증상
3. 가설·검증 기록
4. 원인·복구 방안 (실행은 하지 않음)
5. SRE 세 질문 (사용자가 겪은 것 / 알아챈 증상 / 미리 볼 수 있었던 지표)

못 풀었어도 **3번(가설·검증 기록)이 있으면 발표가 성립**합니다. 평가는 결론이 아니라 좁혀간 과정입니다.

## 규칙

- 진단(조회·tcpdump·네임스페이스 안에서 ping)은 자유, **설정 변경·복구는 하지 않습니다** — 복구는 세션에서 운영진이 합니다
- 특히 컨테이너 재시작 · `kolla-ansible reconfigure` · 시큐리티 그룹 변경 · FIP 재할당 · `ip link` / `ovs-vsctl`로 상태 바꾸기 금지 (공용 노드라 다른 사람 환경에 영향이 갑니다)
- 노드 접속은 본인 계정으로만 (명령 기록이 8회차 포스트모템 재료가 됩니다)
- 비밀번호 · SSH 개인 키 · openrc 내용은 절대 커밋하지 않습니다

---

## Git으로 제출하기

처음이면 아래를 순서대로 따라 하면 됩니다. Windows는 PowerShell, Mac/Linux는 터미널에서 실행합니다.

### 최초 1회

```bash
git clone https://github.com/cloud-club/10th-openstack-network-troubleshooting.git
cd 10th-openstack-network-troubleshooting
git config user.name "본인이름"
git config user.email "본인@이메일"
```

> 이 저장소는 운영진이 collaborator로 초대합니다. GitHub 초대 메일을 수락한 뒤 clone 하세요.

### 매주 제출 (예: 2회차, 홍길동)

```bash
# 1. 최신 main 받기
git checkout main
git pull

# 2. 내 브랜치 만들기  (규칙: weekN/<본인이름>)
git checkout -b week2/홍길동

# 3. 제출 양식을 복사해서 작성
cp template.md week2/홍길동.md
#    → 편집기로 week2/홍길동.md 작성

# 4. 커밋 & 푸시
git add week2/홍길동.md
git commit -m "week2: 홍길동 제출"
git push -u origin week2/홍길동
```

### PR 열기

1. 푸시 후 터미널에 뜨는 링크를 열거나, GitHub 저장소 페이지의 **Compare & pull request** 클릭
2. base: `main` ← compare: `week2/홍길동` 인지 확인
3. 제목: `[week2] 홍길동`
4. **Create pull request**

제출 후 내용을 고치고 싶으면 **같은 브랜치에서** 수정 → `git add` → `git commit` → `git push` 하면 PR에 자동 반영됩니다.
마감 후 운영진이 일괄 merge 합니다.

### 하지 말 것

- `main`에 직접 push
- 다른 사람 파일 수정
- 이미지 캡처에 비밀번호·키가 보이는 상태로 업로드

### 자주 막히는 것

| 상황 | 해결 |
| --- | --- |
| `git push` 할 때 로그인 창/토큰 요구 | GitHub 비밀번호 대신 **Personal Access Token**이 필요합니다. Settings → Developer settings → Tokens에서 발급 후 비밀번호 자리에 입력 |
| `fatal: not a git repository` | `cd`로 저장소 폴더 안에 들어와 있는지 확인 |
| 브랜치를 잘못 만들었음 | `git checkout main` 후 다시 `git checkout -b` |
| PR에 다른 주차 파일까지 올라감 | 그 파일은 `git rm --cached 경로` 후 커밋·푸시 |

막히면 오류 메시지를 그대로 복사해서 운영진에게 보내주세요.
