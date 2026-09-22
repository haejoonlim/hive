# hive v0.4 릴리스 노트

날짜: 2026-09-22
전 버전: v0.3

## 하이라이트

v0.4는 hive가 **"상태를 보고 다음 행동을 결정하는" 단계**로 넘어간 버전이다.
rotate 승인 루프, 배분 안전장치, 완료 보고 자동 회수, HEAD 명령 청취가 이번에 들어갔다.

## 신규 기능

### 1. rotate ↔ watch 승인 연동 (v0.5 로드맵 조기 달성)

- watch가 **컨텍스트 40만+ 탭**을 감지하면 `.orch/rotate-pending.txt`에 기록하고
  HEAD 탭에 승인 요청 메시지를 자동 전송 (`hive approve <id> 안내 포함`).
- `hive approve <탭id접두>` → 승인 파일 기록 → watch 다음 사이클에 **자동 rotate**
  (요약 심은 새 탭 생성 → 구탭 close → 세션 반납·시간제 회수).
- `hive pending` 으로 승인 대기 목록 확인. 재요청은 6시간 쿨다운.

### 2. dispatch 소유권 충돌 검증

- 같은 파일 경로가 **두 워커 인박스에 이중 배분**되면 전송 자체를 중단.
- 신규 임무문 경로 vs 기존 인박스 경로 교차 검사 + 인박스 간 중복 검사.
- 충돌 시 경로·보유 워커 목록을 보여주고 재배분을 유도.

### 3. 시간제 가드 — 소비 중 탭 과다 경고

- watch가 `turnState=running` 탭 수를 주기적으로 집계.
- **3개 초과** 시 로그에 경고 (30분 주기): "idle 탭은 15분 후 자동 반납 — 억지 컨티뉴 금지".
- 완료·대기·정체 탭 깨우기 금지 규칙과 함께 시간제(Freebucks/시간) 소비 억제.

### 4. 완료 보고 → HEAD 자동 검수 요약

- 워커가 "…완료"로 보고하면 watch가 `.orch/outbox/W{n}.md` 꼬리부(500자)를 읽어
  **HEAD 탭에 검수 요약을 자동 전송** + 전문 경로 안내.
- 5분 쿨다운으로 중복 보고 방지. 비워커 탭의 완료성 응답은 기존대로 스킵.

### 5. HEAD 탭 명령 청취

- HEAD(대가리) 탭에서 사람이 `hive pending` / `hive approve <id>` 를
  메시지로 보내면 watch가 감지해 **결과를 탭에 답변**으로 돌려준다.
  HEAD는 컨티뉴 대상에서 여전히 제외 — 깨우기 없이 명령만 실행.

## 개선·수정

- 시간제 경고 문구를 실행 지침형으로 개선.
- `--help` 에 `approve`/`pending` 문서화.

## 검증

- 단위 테스트 11건 (소유권 충돌 3, rotate 연동 2, 완료 보고 4, 상수/헬퍼 2) 통과.
- watch 데몬 재기동·실동작 확인 (인박스 자동 핑, sweep, 앱 대기 모드).

## 전체 명령어

```
threads states send send-id show dispatch digest stats meter rotate sweep
approve pending markers brain decide watch status
```

## 다음 로드맵

- v0.5+: Jev 라우팅(공식 API 키 또는 로컬 SemIf — RAM 제약으로 기본 꺼짐), 모델 라우팅
