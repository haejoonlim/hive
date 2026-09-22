# hive × Jev — 의사결정 오케스트레이터로의 진화 (v2 제안서)

날짜: 2026-09-22 · 현재 hive: v0.2 (watch·dispatch·digest·stats 운영 중)
대상: hive를 "명령 전달 CLI" → "상태를 보고 다음 행동을 결정하는 오케스트레이터"로

---

## 0. 한 줄 요약

지금 hive는 **결정은 사람(HEAD)이 하고, 전달만 기계**가 한다. 여기에 Jev(TypeSafe의
System One 결정 전용 모델)를 결정 계층으로 끼워 넣으면, hive가 워커 풀의 상태를 스스로
보고 **누구에게·지금·무엇을 시킬지·끝났는지** 판단하는 구조가 된다. Jev는 답변을
생성하는 모델이 아니라 **Choice/Score/Noul 타입 결정**을 70~500ms에 반환하고,
출력 토큰이 무료($0.042/MTok 입력, 출력 $0)라서 watch 루프 같은 고빈도 판단에 최적.
(출처: TypeSafe 공식 발표, 2026-09-15)

---

## 1. 왜 Jev가 hive에 맞는가 — hive 실측과 대조

hive v0.2가 이미 갖고 있는 관측 데이터:

| hive가 이미 관측하는 것 | 코드 근거 | Jev 판단에 쓸 수 있는 형태 |
|---|---|---|
| 탭별 turnState (running/idle) | `turn_state()` → `/api/thread/:id` | 라우팅·컨티뉴 결정의 1차 신호 |
| 탭별 메시지 스냅샷 (role/parts) | `fb.snapshot()` | 마지막 답변이 "대기"인지, 실패 흔적인지 |
| 토큰 추정치·턴 수 | `_estimate_tokens()` (3.2자≈1tok) | 컨텍스트 임계·번레이트 판단 |
| 5h 윈도우 사용률 | `cmd_stats()` | budget 정책 |
| outbox 보고 (append 전용) | `.orch/outbox/W{n}.md` | judge — "완료 인정?" 판단 |
| 인박스 타임스탬프 | `dispatch` append 헤더 | 우선순위·aging 계산 |

즉 **Jev에게 줄 "structured program state"가 이미 다 모인다.** 억지로 붙이는 게
아니라 watch/dispatch/stats 사이에 결정 계층을 삽입하는 형태.

Jev 특성과의 궁합:

| Jev 특성 | hive에서의 의미 |
|---|---|
| 병렬 다중 질문 1콜 (70~500ms) | watch 루프 20초 주기마다 워커 6개 × 질문 5개도 즉시 |
| 출력 무료, 입력 $0.042/MTok | 24시간 감시해도 비용 사실상 0 |
| 타입 안전 (type error 불가능) | 파싱 실패·환각 없음 — Python이 그대로 실행 |
| 보정된 확률(calibrated) | "continue=0.94" 같은 숫자를 임계값 정책으로 바로 씀 |
| 문자열 생성 없음 | 임무문 작성·요약은 Freebuff 워커 몫 — 역할 충돌 없음 |

---

## 2. 아키텍처: 결정(deterministic)과 판단(Jev)의 경계선

이 제안서의 핵심 규칙. **모든 판단을 Jev에 넘기면 안 된다.**

```
Python이 반드시 처리 (결정론)          Jev가 처리 (확률적 판단)
─────────────────────────────         ─────────────────────────
if worker_idle                        어느 워커가 이 임무에 적합한가?
if cooldown_expired                   이 임무는 긴급한가? (우선순위)
if port_changed                       두 임무가 의미상 중복인가?
if file_exists                        지금 rotate하는 게 이득인가?
if budget > limit                     이 보고서는 완료로 인정할 만한가?
if exact_title_match                  retry vs human review?
```

```
        ┌─────────────┐
        │    HIVE     │  상태수집 / 큐 / 메모리 / Freebuff API / 실행
        └──────┬──────┘
               │  애매한 판단만 넘김
               ▼
        ┌─────────────┐
        │     JEV     │  Choice / Score / Noul — 타입 결정 반환
        └──────┬──────┘
               │  typed decision
               ▼
             HIVE → ACTION
```

**fallback 필수**: Jev는 클라우드 API라 네트워크 의존이 생긴다. Freebuff 자체는
로컬이라는 hive의 성격을 지키려면 `JEV_ENABLED=false` 또는 Jev 무응답 시
**결정론 fallback 정책**(현재 v0.2의 규칙 그대로)으로 즉시 강등해야 한다.
watch는 Jev 없이도 동작해야 한다 — Jev는 "더 똑똑해지는 옵션", 필수 의존성 아님.

---

## 3. 구현할 기능 (hive v0.3~v1.0)

### v0.3 — Smart Watch (Jev 첫 접점, 가장 효과 큼)

현재 watch: `20초 → 상태확인 → idle이면 continue`. 문제: 왜 idle인지 모른다.
미완료 작업이 끊긴 건지, 할 일이 없어서인지, 실패를 반복 중인지 같은 "idle"이다.

Jev에게 한 콜로 병렬 질문:

```
state: {tab, turnState, last_role, last_text, token_est, errors_recent, outbox_last}
Q1 continue_worthwhile : Noul   ← 이어서 할 일이 있는가
Q2 likely_stuck        : Noul   ← 실패 반복·막힘 상태인가
Q3 should_rotate       : Noul   ← 컨텍스트 갱신이 필요한가
Q4 urgency             : Score  ← 얼마나 급한가 (0~1)
```

정책 (코드로 명시, Jev 결정 → 임계값은 운영 데이터로 조정):

| 판정 | 조건 | 액션 |
|---|---|---|
| CONTINUE | `continue>0.85 and stuck<0.3` | 기존대로 컨티뉴 전송 |
| ROTATE | `rotate>0.7` | v0.5 rotate 트리거 (당장은 HEAD 탭에 제안 메시지만) |
| WAIT | `continue<0.15` | 아무것도 안 함 — 현재 "대기" 스킵 로직의 확장판 |
| ASK_HEAD | `stuck>0.6` | HEAD 탭에 "W{n} 막힘 추정, 확인 요망" 전송 |

**fallback**: Jev 타임아웃(500ms)·에러 시 기존 v0.2 규칙(idle+쿨다운→continue)로 즉시 강등.

### v0.4 — Worker Registry + 소유권 배분 (dispatch 개량)

- 워커별 성과 기록 (`.orch/memory/workers.json`): 완료율·평균 턴·평균 토큰·재시도율·
  태스크 타입(코딩/문서/리서치/테스트)별 성공률.
- `dispatch`가 임무 분배 시 Jev에게:
  `이 임무(Unity C# 버그픽스)에 누가 적합? W1=코딩 92% / W2=문서 88% / W3=리서치 91%`
  → Choice로 워커 선택 + 근거 확률. 같은 파일 중복 배분은 Python이 하드 차단(결정론).
- **모델 재학습 아님** — 관측→성과 기록→state→Jev 판단→기록의 closed loop.

### v0.5 — Jev Router + rotate 자동화 (절감 최대점)

- `hive rotate "<탭>"`: TokenPilot 근거의 컨텍스트 리셋 (제안서 v0.5 항목 확정안).
  마지막 어시스턴트 메시지 요약 → `POST /api/threads` 새 탭 → 요약 심기 → 구탭 close.
- rotate 실행 여부를 "토큰>300k" 같은 하드코딩 대신 Jev Noul(`rotate_probability=0.86`)
  로 판단: 컨텍스트 크기 + 턴 수 + 에러율 + idle 시간 + 진행률을 state로 주면 됨.
- Smart Watch의 ROTATE 판정이 실제 rotate를 트리거.

### v0.6 — Jev Judge (완료 인정 계층)

워커가 `W1 완료`라고 보고해도 그대로 믿지 않는다 (sp-verification 원칙과 일치):

```
state: outbox 보고서 + 실제 변경(diff stat) + 검증 결과
Q1 complete      : Noul
Q2 needs_review  : Noul
Q3 retry         : Noul
```

정책 예: `complete>0.85 → HEAD 승인 대기 / needs_review>0.6 → 검수 워커 투입 /
retry>0.5 → 재작업 지시`. 단, **Jev confidence는 정답 보증이 아니다** —
공식 문서도 명시. 최종 커밋 승인은 여전히 HEAD(사람)만.

### v0.7 — Semantic Dedup + Event Log

- 대기 큐의 임무들이 의미상 중복인지 Jev Score로 비교 → 중복 제거.
- `.orch/events.log`: 모든 결정(Jev 질문→답→액션) 기록 → digest가 "왜 그렇게
  움직였는지"까지 요약. 디버깅·신뢰 검증의 기반.

### v1.0 — Autonomous Hive (목표상)

```
HUMAN   목표/승인/예외
  ↓
HIVE    state·queue·memory·API
  ↓
JEV     routing·scoring·retry·priority
  ↓
FREEBUFF  Claude 등 에이전트 — 실제 reasoning·coding·tools
  ↓
RESULT → JEV (accept/retry/rotate) → MEMORY
```

---

## 4. 명령어 구조 (v2 목표)

```
현재 (v0.2)                v2 추가
hive threads               hive brain     ← 전체 상태 + Jev 권고 액션 원스톱
hive states                hive decide    ← 임의 질문 수동 실행 (디버깅용)
hive send                  hive rotate    ← 컨텍스트 리셋
hive dispatch              hive judge     ← outbox 보고 판정
hive digest                hive workers   ← 워커 평판 조회
hive stats                 hive budget    ← 5h 윈도우 예산 정책
hive watch                 hive events    ← 결정 로그
```

`hive brain` 출력 예 (제안서 안):

```
HIVE BRAIN
────────────────────────────────────
Workers       6   (running 3 · idle 2 · blocked 1)
Context       2.8M  ·  5h estimate 71%
JEV
  W3 → continue (0.94)
  W5 → rotate   (0.89)
  W6 → wait     (0.01)
Mission: GDD validation → W1 + W3 (충돌 없음)
Budget: SAFE
```

---

## 5. stats의 진화 — 번레이트 추정

현재 stats는 3.2자≈1토큰 근사 + 시스템 프롬프트·캐시 미포함 → **운영용 추정치**로만
유효. 여기에 추가:

- `context_growth` (시간당 토큰 증가), `turn_rate`, `idle_rate`, `failure_rate`
- `burn_rate = tokens / elapsed` → Jev에게 "이 속도면 5h 윈도우 몇 시간 남았나" 판단
- 윈도우 위험 예측 시 HEAD 탭에 자동 경고

## 6. 리스크와 완화

| 리스크 | 완화 |
|---|---|
| Jev 클라우드 의존 (Freebuff는 로컬인데) | fallback 결정론 정책 항상 유지, Jev는 옵션 |
| confidence ≠ 정답 | 임계값을 실제 성과 데이터로 검증, 커밋 승인은 사람 전용 |
| 워커 행동 변화로 평판 노후화 | 평판에 최근 N건 가중(지수 이동평균) |
| Jev API 조기 접근 변동성 | Jev 호출부를 단일 모듈로 격리 → 교체 쉽게 |

## 7. 구현 순서 (권장)

1. **Smart Watch** (v0.3) — Jev 첫 접점. 기존 watch에 decision hook만 추가.
2. **Jev Router + rotate** (v0.5) — 절감 최대점.
3. **Jev Judge** (v0.6) — 신뢰 계층.
이 세 개만 되어도 hive 성격이 바뀐다: "명령을 전달하는 도구" → "상태를 보고
다음 행동을 결정하는 오케스트레이터". 나머지(registry·dedup·budget)는 부착물.

---

*참조: TypeSafe AI — Introducing System One Models & Jev (2026-09-15).
Jev 입력 $0.042/MTok·출력 무료·70~500ms·Choice/Score/Noul 타입 출력·RLCD 학습.
DataCamp·Firecrawl·Beam AI의 2차 분석 자료도 동일 구조 확인.*
