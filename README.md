# hive 🐝 — Freebuff 멀티에이전트 오케스트레이터

**Freebuff 데스크톱 앱을 벌집(hive)처럼 굴리는 CLI.** 앱 수정 없이, 앱이 이미 가진
로컬 오케스트레이터 API를 발굴해 여러 탭(에이전트 세션)을 자동으로 깨우고, 조율하고,
소비를 감시한다.

> 한국어 사용자를 위한 도구지만 영어 코멘트도 환영합니다.
> Tested on: Freebuff Desktop 0.0.133 (macOS)

---

## 왜 필요한가

Freebuff 앱 안에는 여러 세션(탭)을 띄울 수 있지만:

1. **탭끼리 메시지를 보낼 방법이 없다** — 각각 다른 방에 있는 느낌.
2. **세션이 끊기면(크레딧 소진 등) 사람이 다시 엔터를 눌러야 한다.**
3. **소비량이 안 보인다** — 시간제(구독) 요금인데 어느 탭이 얼마나 먹는지 모른다.

hive는 이 세 가지를 모두 해결한다. 핵심 발견: Freebuff 앱(Electron)이 자식으로 띄우는
Bun 오케스트레이터 서버가 `127.0.0.1:<port>` 에서 **로컬 HTTP API** 를 노출하고 있고,
뮤테이션 인증용 launchId는 오케스트레이터 프로세스 환경변수(`FREEBUFF_LAUNCH_ID`)에 있다.
→ 앱 바이너리를 건드리지 않고도 **프로그래밍 방식으로 탭을 조종**할 수 있다.

## 설치

```bash
git clone https://github.com/haejoonlim/hive
cd hive
cp hive /usr/local/bin/hive   # 또는 PATH 안 폴더
chmod +x /usr/local/bin/hive
```

요구 사항: macOS + Freebuff 데스크톱 앱(실행 중) + python3.

## 빠른 시작

```bash
hive threads            # 열린 탭 목록
hive states             # 탭별 running/idle
hive send "워커 1" "인박스 확인"    # 탭에 메시지 (사용자 엔터와 동일)
hive show "워커 1" 5    # 탭 대화 엿보기
```

## 핵심 기능

### 1. watch — 무인 감시 (launchd 등록 권장)

- 탭이 `idle` 이 되면 **자동 컨티뉴**: 세션이 끊겨도 다음 주기에 이어서 일한다.
- `.orch/inbox/W*.md` 에 새 지시가 쓰이면 해당 워커 탭에 **인박스 확인을 자동 전송**.
- 탭별 5분 쿨다운 — 무한 크레딧 소모 방지.

```bash
hive watch 20           # 포그라운드
# 백그라운드 (로그인 시 자동 시작):
cp com.haejoon.hive-watch.plist ~/Library/LaunchAgents/
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.haejoon.hive-watch.plist
tail -f /tmp/hive-watch.log
```

### 2. dispatch — 한 줄 임무 배분

`.orch/inbox` 에 기록 + 워커 탭 전송까지 한 번에:

```bash
hive dispatch "소울 커먼더 문서 검수 3건: ①GDD_00 ②GDD_05 ③README"
```

### 3. digest — 아침에 어제 밤 활동 요약

```bash
hive digest 12          # 최근 12시간, 탭별 + .orch/outbox 보고서
```

### 4. stats — 소비 추정 (시간제 요금 대응)

```bash
hive stats
```

탭별 턴 수·토큰 추정치·5시간 윈도우 사용률 표시. 구독 한도가 언제 깨질지 가늠.

## .orch 파일 큐 (옵션)

hive는 `.orch/` 디렉토리가 있는 프로젝트에서 강력해진다:

```
.orch/
  CONTRACT.md        워커 행동 규약
  STATUS.md          상태표 (HEAD 전용)
  inbox/W{n}.md      HEAD → 워커 지시
  outbox/W{n}.md     워커 → HEAD 보고 (append 전용)
```

워커 탭을 한 번 부트시켜두면(`.orch/CONTRACT.md 읽고 따르라`) 이후 전부 hive가 중개한다.

## 문서

- [OPENJEV-REVIEW.md](OPENJEV-REVIEW.md) — 공식 Jev API 없이 로컬 결정 엔진(SemIf/JEV-CPU)을 쓸 수 있는지 검수한 보고서
- [JEV-PROPOSAL.md](JEV-PROPOSAL.md) — hive를 의사결정 오케스트레이터로 진화시키는 v2 설계
- [PROPOSAL.md](PROPOSAL.md) — 토큰·시간 절감 기술 리서치
- [RESEARCH.md](RESEARCH.md) — 리서치 근거 + 구현 로드맵

## 작동 원리 (기술 노트)

- `ps aux` → 오케스트레이터 프로세스 발견 (`orchestrator.js`)
- `ps eww <pid>` → `FREEBUFF_LAUNCH_ID` 환경변수 캡처
- `lsof -p <pid>` → 로컬 리스닝 포트 발견
- API:
  - `GET /api/projects` — 탭 목록
  - `GET /api/thread/:id` — 스냅샷 + `thread.turnState`
  - `POST /api/thread/:id/message` — 메시지 전송 (헤더 `x-freebuff-launch-id` 필요)
- 앱 업데이트로 API가 바뀌면 스크립트 상단의 경로만 수정하면 된다. 앱 수정이 없으므로
  자동 업데이트에도 설정이 지워지지 않는다.

## 한계

- Freebuff 앱이 켜져 있어야 동작 (오케스트레이터는 앱의 자식 프로세스).
- 컨티뉴 전송도 세션 크레딧을 소모한다 — 쿨다운을 늘리면(`hive watch 60`) 소모 감소.
- 개인용 도구로 만들었음. API가 비공개라 언제든 바뀔 수 있다.

## 라이선스

MIT
