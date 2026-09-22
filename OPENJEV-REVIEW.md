# OpenJev 검수 보고서 — 공식 Jev API 없이 hive의 결정 계층을 만들 수 있는가

날짜: 2026-09-23 · 검수 대상: OpenJev(현 SemIf) · 결론: **적용 가능 (조건부)**

---

## 1. 검수 대상 확인

| 항목 | 내용 |
|---|---|
| 원 이름 | **OpenJev** → 2026-09-18 **SemIf** 로 리네임 (`TheoLeeCJ/SemIf`, MIT, ★3.2k+) |
| 정체 | Jev(TypeSafe, 클로즈드)의 **인터페이스 패턴 재현**. 모델·학습 재현 아님 (README 명시) |
| 핵심 원리 | 답변을 생성하지 않음. **한 번의 forward pass에서 옵션 글자(A/B/C…)의 logit을 직접 읽어 확률로 반환** — JSON 파싱·디코딩 루프 없음 |
| 포크 | **Meanblock/JEV-CPU** — SemIf를 순수 CPU로 포팅 (GPU 강제 코드 1곳만 교체) |

## 2. 이 머신 실측 조건 (2026-09-23 확인)

| 항목 | 값 | 영향 |
|---|---|---|
| CPU | Intel i5-6500 4코어 3.2GHz | 결정 1건 ≈ 1초 (JEV-CPU 실측 기준) |
| RAM | 8 GB | 0.6B 모델 float32 2.4GB — 가능하나 여유 적음 |
| GPU | AMD R9 M390 2GB (Metal 2) | **사실상 못 씀** — CUDA 없음, MLX는 Apple Silicon 전용 |
| 런타임 | python3 3.9 (구버전), node 있음, bun 없음 | SemIf는 **Python 3.10+** 필요 → pyenv/uv로 3.10+ 설치 필요 |

## 3. 모델 선택지 (SemIf 공개 벤치마크 기준)

| 모델 | 용량 | 균형정확도 | 공식 Jev 대비 | 이 머신 판정 |
|---|---|---|---|---|
| Qwen3-0.6B Q8 | 639 MB | 0.440 | 낮음 | 가능하나 품질 부족 |
| **MiniCPM5-2B Q4** | **1.56 GB** | **0.686** | 중간 | **추천 — RAM·품질 균형** |
| Qwen3.5-4B Q4 | 3.01 GB | 0.813 | 근접 (Jev 0.883) | 가능하나 4코어에서 3~10초/건, RAM 압박 |
| 공식 Jev (클라우드) | — | 0.883 | — | 접근 불가 (유저 미보유) |

## 4. 결론: 조건부 적용 가능

**가능.** 추천 조합:

1. **엔진**: JEV-CPU(순수 CPU, 웹 UI·API 내장) 또는 SemIf llama.cpp 백엔드.
2. **모델**: MiniCPM5-2B Q4_K_M (1.56GB) — 8GB RAM에서 유일한 현실적 균형점.
3. **구동**: 로컬 HTTP 서버로 상시 띄움 → hive의 결정 계층이 localhost로 질의.
4. **보정**: SemIf에 workload별 온도 스케일링 내장 (raw ECE 0.208 → 0.069 실증) — 사용 전 소유 워크로드로 캘리브레이션 필수.

## 5. hive 연동 설계 (v0.3 반영)

```
hive watch/dispatch/judge
        │  결정 요청 (state + question + options)
        ▼
┌─────────────────────────────────────┐
│ hive decision provider (단일 모듈)   │
│  ① official  — TypeSafe Jev API     │  ← 키 생기면 자동 승격
│  ② local     — SemIf/JEV-CPU HTTP   │  ← 이 머신 (추천)
│  ③ fallback  — 결정론 규칙 (v0.2)    │  ← ①② 실패 시 항상 동작
└─────────────────────────────────────┘
```

- Jev 질문은 hive 워크로드에 맞춘 **소형 클로즈드 셋**: continue_worthwhile, should_rotate,
  likely_stuck, worker_fit, complete — 5종. 로컬 0.6B~2B 모델로도 판별 가능한 수준.
- fallback이 기본값 → SemIf 다운돼도 hive v0.2 동작 유지 (무중단).
- 결정 로그(`.orch/events.log`)에 백엔드·확률·소요시간 기록 → 정확도 실측 후 임계값 조정.

## 6. 리스크

| 리스크 | 판정 |
|---|---|
| 품질: 공식 Jev(0.883) 대비 0.69~0.81 | hive의 판단은 단순 이진 판단 다수 + fallback 병행 → 수용 가능 |
| 속도: 1초/건 (CPU) | watch 20초 주기엔 충분. dispatch 배분 등 다중 질문 시 수 초 — 허용 범위 |
| RAM: 8GB에서 Freebuff+모델 병행 | 2B 모델 1.56GB로 억제. 메모리 압박 시 스왑 주의 |
| Python 3.10+ 필요 | pyenv/uv로 별도 설치 — 시스템 3.9 건드리지 않음 |
| 프로젝트 유령화 가능성 | MIT + 단일 파일 인터페이스라 포크/대체 용이 |

## 7. 참조

- SemIf: github.com/TheoLeeCJ/SemIf (MIT, 공개 벤치마크·캘리브레이션 자료 포함)
- JEV-CPU: huggingface.co/Meanblock/JEV-CPU (CPU 실측: 로드 5~17초, 결정 ~1초)
- Hacker News 토론 (2026-09-20, 718 points): 인터페이스 재현임을 저자 명시, 캘리브레이션 한계 논의
