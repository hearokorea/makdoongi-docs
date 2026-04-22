# 막둥이 (Makdoongi)

한국 주식 시장을 위한 **AI 기반 개인 자동 매매 프로그램**.

막둥이는 AI 가 시장을 분석하고 판단해서 자동으로 매매를 수행합니다.
사용자는 텔레그램으로 현황을 확인하고 명령을 내릴 수 있습니다.

---

## 📖 문서

| 문서 | 대상 | 내용 |
|---|---|---|
| [사용자 가이드](GUIDE.md) | 처음 사용자 | 빠른 시작, 기본 사용법 |
| [사용자 매뉴얼](MANUAL.md) | 모든 사용자 | 전체 기능, 명령어, 운영 안내 |
| [변경 이력](CHANGELOG.md) | 관심 사용자 | 버전별 주요 변경 사항 |

**처음이신가요?** → [사용자 가이드](GUIDE.md) 부터 읽어주세요.

---

## ✨ 주요 기능

### 1. AI 자율 매매
- 시장 분석 → 종목 스크리닝 → 매수/매도 자동 실행
- Claude Opus/Sonnet/Haiku 3계층 비용 최적화 (규칙 엔진으로 명확 케이스는 AI 호출 생략)
- 기술 지표(RSI/MACD/ADX/BB 등) 사전 계산 후 프롬프트 주입 — AI 는 판단에만 집중

### 2. 실시간 리스크 관리
- 강제 손절(-10%) · 1차 손절(-5% AI 판단) · 익절 · 트레일링 스탑
- 자투리 매수 차단 (건당 최소 주문 금액) + 자투리 포지션 자동 청산 (본전 회복 시)
- 항상성 컨트롤러 — 연속 손실 / 매도 실패율 / 포지션 만석 상태에서 자동 보수화

### 3. 텔레그램 연동
- 현황 보고 · 포지션 · 성과 · 결산 · 전략 실시간 조회
- 자기 진단(`/selfstate`) · 시스템 건강도(`/health`) · 수리 큐(`/repairs`) · 자율 튜닝(`/tune`)
- 자유 입력으로 AI 에게 직접 명령/질문 가능

### 4. 자가 복기·학습·자율 개선
- **즉시 복기** — 손절 직후 1초 내 패턴 분류 + 개선 제안
- **일일/주간/월간 복기** — 자동 실행 + 텔레그램 리포트
- **패턴 학습** — 손실/성공/오탐 3종 DB + 통계 기반 신규 패턴 자동 발견
- **자율 튜닝** — 이벤트 기반 파라미터 자동 조정 (safety gate + 3일 후 효과 측정 + 악화 시 자동 롤백)
- **자기 진단** — 24h 슬라이딩 윈도우로 "나는 지금 struggling 상태" 같은 자기 인식

### 5. 안정성·복원력
- 크래시 감지 + 무결성 감사 + 강제 재조정 (부팅 시 자동)
- 미완료 액션 24h 재개 + 유령 포지션 자동 정리 (3회 미확인 후)
- 외부 거래 / 지연 체결 자동 구분 (`order_no` + ticker·qty·side intent 매칭)
- 매 상태 이벤트 영속화 — 재시작해도 경보 쿨다운/항상성 이탈/매도 실패 카운터 모두 보존

### 6. 비용 의식형 수리 큐
- 고비용 AI 수정이 필요한 크래시·부채·오류는 자동 수리 대신 **큐에 누적**
- `/repairs` 로 운영자가 몰아서 승인 → 비용 폭주 방지

### 7. 컴팩트 알림
- 모바일 한 화면에 핵심 정보 (계좌/포지션/프로그램 상태) 표시

---

## 🧬 시스템 구조

### 25 AXIS + 4 Foundation Layer
모든 프로덕션 모듈이 **25개 축 + 4 Foundation** 에 100% 공식 소유
(orphan 0 · 중복 0 · 유령 경로 0).

| Tier | 축 | 역할 |
|---|---|---|
| 1 | `ai_brain` · `trade_engine` · `risk_management` | 핵심 매매 |
| 2 | `postmortem` · `market_intelligence` · `strategy_engine` · `decision_context` | 분석/판단 |
| 3 | `catchup` · `router` · `reconciler` · `external_trade_sync` | 실행 인프라 |
| 4 | `system_management` · `notifier` · `tech_debt` · `data_integrity` · `self_diagnostics` · `temporal_context` | 운영/품질 |
| 5 (메타) | `system_integrity` · `false_positive_audit` · `hot_reload` · `log_archive` · `homeostasis` | 자가 진화 |
| Foundation | `state_persistence` · `runtime_config` · `axis_meta` · `axis_event_bus` | 공통 기반 |

각 축에 `owns`(소유 모듈) + `tier` + `publishes`/`subscribes` 이벤트 명시.

### 7-Stage 유기체 진화
Meta-Awareness · Growth · Homeostasis · Response · Immunity · Circulation · Reproduction
— 복기 → 학습 → 행동 변화 → 효과 측정 → 롤백의 **닫힌 피드백 루프**로 작동.

### 학습·자가 인식 5단계
| 단계 | 담당 | 핵심 메커니즘 |
|---|---|---|
| 1. 누적 | StateManager · PatternDB | trades.json · postmortems · SQLite |
| 2. 복기 | PostmortemEngine | 즉시/일일/주간/월간 자동 실행 |
| 3. 학습 | PatternDB + SuccessPatternDB + FalsePositiveAudit + CustomPatternDetector | 손실·성공·오탐·신규 발견 4종 DB |
| 4. 배우기 | DecisionContext | 과거 패턴 top3 를 매 AI 판단 프롬프트에 자동 주입 + Screener 점수 반영 |
| 5. 깨우치기 | TuningExecutor | 이벤트 기반 자율 파라미터 조정 + 3일 효과 검증 + 자동 롤백 |
| Meta-cognition | SystemSelfState | 24h 슬라이딩 윈도우 자기 진단 (`struggling` / `recovering` / `excellent` 등 5 레벨) |

---

## 💻 주요 텔레그램 명령

| 명령 | 설명 |
|---|---|
| `/성과` `/포지션` `/전략` `/계좌` `/시장` `/결산` | 매매 현황 |
| `/health` | 25축 + 학습 5단계 + Foundation 건강도 |
| `/selfstate` | 24h 자기 진단 + 자동 발견 손실 패턴 top3 |
| `/tune` | 자율 튜닝 현황 (`/tune history`, `/tune params`) |
| `/repairs` | 수리 큐 목록 (`/repairs history`, `/repairs today`) |
| `/reload` | 코드 핫 리로드 |
| 자유 입력 | AI 에게 직접 명령/질문 |

전체 명령 → [MANUAL.md](MANUAL.md)

---

## 문의

이 저장소는 사용자 매뉴얼 공개용입니다.
프로그램 설치·계정 연동 등 운영 관련 문의는 운영자에게 하세요.

---

© Makdoongi. All rights reserved.
