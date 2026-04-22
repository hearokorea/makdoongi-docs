# 변경 이력 (Changelog)

막둥이의 주요 버전별 변경 사항을 버전/날짜 역순으로 정리합니다.
세부 기능 설명과 운영 방법은 [MANUAL.md](MANUAL.md) 참조.

---

## V3.72.1 — 2026-04-22

### 🧬 유기체 진화 7-Stage 완성

살아 움직이는 생명체처럼 동작하도록 7단계 진화 완료.

- **Meta-Awareness** — AI 가 자기 시스템 상태 인지
  (system_self 자동 프롬프트 주입: 포지션 만석 / 연속 손실 / 매도 실패율 /
  regime_tag / 항상성 이탈 / 성공 패턴 매칭)
- **Growth 실증** — 일일 복기 → 전략 자동 튜닝 → SQLite `strategy_tuning_history` 영속
- **Homeostasis Controller (축 27)** — 5분 주기 목표 상태 측정 → 이탈 시 자동 복귀
  (신규 매수 차단 / 보수 모드 / 브로커 재연결) + AI 프롬프트 자동 주입
- **Response 속도** — NEWS_URGENT 등 긴급 이벤트 → 2초 내 AI 루프 트리거 (기존 180초 → 90× 빠른 반응)
- **Immunity 자동 치유** — SELL_FAIL_STREAK / HOT_RELOAD_FAILED / HEARTBEAT_STALE 등
  5종 CRITICAL 이벤트 자동 치유
- **Circulation 가시화** — 이벤트 핸들러 실패 통계 누적 → `/health` 노출
- **Reproduction (학습 루프 완성)** — 익절 5%+ 매도 시 성공 시그니처 자동 추출
  → `data/success_patterns.json` → 스크리너가 유사 종목에 보너스 점수 자동 부여

### 🎓 학습·자가 인식 5단계 100% 달성

| 단계 | 신설/강화 모듈 |
|---|---|
| 1. 누적 | `_tool_usage_stats` 영속화 |
| 2. 복기 | (유지 — 즉시/일일/주간/월간 자동) |
| 3. 학습 | **CustomPatternDetector 신설** — 통계 기반 신규 손실 패턴 자동 발견 |
| 4. 배우기 | DecisionContext 에 패턴 top3 자동 주입 + Screener 손실 페널티 |
| 5. 깨우치기 | **TuningExecutor 신설** — 이벤트 기반 자율 튜닝 + 3일 효과 측정 + 악화 시 자동 롤백 |
| Meta-cognition | **SystemSelfState 신설** — 24h 슬라이딩 윈도우 자기 진단 |

### 🛠 핵심 결함 수정

- **포지션 `[장기/이전보유]` 태그 재발 버그 근본 해결**
  (매수 시 group 누락 → 저장 시 "이전보유" 덮어쓰기 → 재시작 후 복원 반복)
  · ScalpPosition 생성 시 `group` + `strategy_id` 필수 주입
  · `buy_stock` 도구 description 에 group 기반 max_hold_sec 자동 결정 명시
  · ConsistencyRule `position_group_restore_from_trade_history` 신설

- **COM 매도 실패 연쇄(P1-7) 근본 수정**
  · `pending_orders` 교차검증을 `order_no` 없어도 ticker+side 기반 매칭
  · `_ghost_sell_counter` — 동일 종목 3중 미확인 3회 연속 시 `GHOST_POSITION_DETECTED` 마커
  · `trade_engine` 마커 감지 → 포지션 즉시 영구 제거 + 알림 1회 (스팸 차단)

- **자기 주문 → "외부거래" 오분류 버그 해결**
  · `ExternalTradeSync.register_own_order_intent()` — SendOrder 직전 의도 기록
  · 체잔 콜백 20분 지연 케이스도 ticker+qty 매칭으로 자기 주문 식별
  · `source="program_delayed"` 분류로 승률 통계 정상 반영

- **메모리 ↔ positions.json 동기화**
  · `PositionMonitor.reload_from_file()` 신설
  · 핫리로드 감지 시 자동 호출 — 유령 포지션 메모리 제거 + group/max_hold_sec 갱신

- **일일 결산 금액 버그**
  · `buy_total_amt` 이중 차감 공식 오류 수정
  · TR 성공 시 balance 캐시 + 실패 시 30분 캐시 우선 사용
  · 입출금 감지 임계 상향 (0.5%/20만 → 2%/100만)

### 🔒 상태 영속화 (재시작 안전)

| 대상 | 위치 |
|---|---|
| HomeostasisController off_targets · 최근 조치 | `data/homeostasis_state.json` |
| SystemIntegrityAxis 경보 쿨다운 · 일일 리셋 | `data/system_integrity_state.json` |
| `_sell_fail_count` · `_zombie_notified` | `ai_state.json` |
| `_tool_usage_stats` | `ai_state.json` |
| SystemSelfState 24h 히스토리 | `data/system_self_state.json` |
| CustomPatternDetector 발견 패턴 | `data/custom_patterns.json` |
| TuningExecutor 효과 감사 | SQLite `tuning_effectiveness` |

### 🔧 스키마 정합성 자동 관리

축 18 `data_integrity` 를 **파일 손상 감지**에서 **cross-field 의미 정합성 감사**까지 확장.
소유 모듈이 자기 cross-field 룰을 plugin 방식으로 등록(ConsistencyRule).
핫리로드 완료 직후 전수 감사 + 자동 교정 → **코드 업데이트 직후 레거시 저장값이 스스로 보정**.

등록 룰 3종:
- `position_group_hold_sec_alignment` — AI-초단타 그룹 max_hold_sec 자동 교정
- `position_group_restore_from_trade_history` — trades.json 기반 group 복원
- `trades_buy_group_max_hold_sec_completeness` — BUY 거래 레거시 필드 bulk 보정

신규 AI 도구 `audit_data_consistency` — AI 자발 감사 호출.

### 📢 오탐 맥락화 (경보 폭주 해소)

- `NO_TRADE_DURING_ACTIVE` — 점심시간 skip + 안전장치 교차검증 + 포지션 만석 skip + 임계 3h/6h 상향
- `LOOP_STALL_CRITICAL` — 장마감 +30분 grace
- `MIDLONG_STALL_CRITICAL` — 오늘 첫 midlong 실행 후 120분 grace + 최신 파일 참조 버그 수정

### 💰 자투리 매매 근절

- **신규 차단** `min_order_amount` (기본 50만원) — 건당 주문 금액 하한
- **기존 청산** `tiny_liquidation_enabled` — 본전 회복 시 자동 매도
- AI 도구 `buy_stock` description 에 명시 → AI 가 스스로 회피

### 🚀 부팅 안정성

- ActionQueue 미완료 액션 자동 재개 (24h 이내) + 만료 자동 거부 (24h 초과)
- OvernightCollector 부팅 15초 후 1회 강제 수집 — 08:50 리포트 데이터 확보

### 💻 운영 명령 신설

- `/selfstate` — 24h 자기 진단 조회
- `/tune` · `/tune history` · `/tune params` — 자율 튜닝 현황·이력·안전 파라미터
- `/health` 에 학습·자가인식 5단계 지표 통합

### 🗂 체계 분류

- PARAM_GROUPS — 143/143 파라미터, 14 그룹
- TOOL_CATEGORIES — 107/107 도구, 22 카테고리
- 메타 축 (tier 5) `meta_group` 부여 (self_healing/audit/infra/archive)
- strategy_engine 에 custom_pattern_detector · system_self_state · tuning_executor 편입

---

## V3.72 — 2026-04-21

### 축 체계 완성 + 메타 자가 진화

- **23축 + 3 Foundation Layer 완성도 100%**
  (R1→R2→R3 3차례 감사, 75/75 모듈 공식 소유)
- **축 22 TemporalContext** — 과거(손실 패턴) + 현재(시장/매크로/뉴스) + 미래(경제 캘린더/실적/공시) 통합 컨텍스트 매 AI 루프 자동 주입
- **축 23 SystemIntegrity (5분 주기)** — 무거래·validator 과엄격·수집기 노쇠·루프 정체·판단-실행 괴리 자동 감지
- **축 24 FalsePositiveAudit (10분 주기)** — 축 23 경보가 사후 오탐으로 드러나는 5종 패턴 감사
- **축 25 HotReload** — 30초 주기 코드 리로드 + 인스턴스 재생성 + 참조 재주입 통합 축
- **runtime_wiring** — main.py 핫리로드 제약 우회. 참조 주입 로직을 별도 모듈로 분리 → 핫리로드 때마다 자동 재주입
- **Foundation Layer 3개** — state_persistence · runtime_config · axis_meta
- **/health 명령** — 23축 + Foundation 건강도 Tier별 표시 + 역참조 경고
- **포지션 보호** — group_policy 모듈화, 블랙리스트 + 옵트인 2중 안전
- **AI 도구 validator 관대화** — 타입 auto-coerce + fuzzy enum (한/영 매핑, str→int/dict/list)
- **즉시 복기 (Phase 7)** — 강제 손절 / -3%↓ 직후 매도 체결 1초 내 패턴 분류 + 개선 제안 전송, AI 호출 0
- **경로 100% 동적화** — BASE_DIR 하나로 전 경로 파생, Config.XXX 경유 강제
- **data/ 명명 규칙** — 11 카테고리 분류 + `.bak` 관리 규칙

---

## V3.71 — 2026-04 초중반

### APP 식별자 중앙화

- 애플리케이션 정체성 단일 진실 공급원 (APP_NAME / APP_DISPLAY_NAME / APP_VERSION / APP_LOCK_FILE 등 .env 파생)
- 마이그레이션: `alphatrace.lock` → `makdoongi.lock` (APP_NAME_LOWER 기반 자동 파생)

---

## V3.50~V3.70 (요약)

- V3.50+ 기술부채 레지스트리 + 자동 수리 큐 (비용 의식형)
- V3.60+ SQLite 통합 스토어 (UnifiedStateStore) + 주간 catchup
- V3.70+ 자체 계산 비용 최적화 (RSI/MACD/ADX/BB 사전 계산, TOOLS 캐싱)

---

## V3.1~V3.44 (초기 주요)

- V3.44 — 복기 + 자가 개선 체계 (손실 패턴 DB, improvement_proposal)
- V3.40 — 중장기 루프 "액션 전용" 도구셋 (AI 호출 15회 → 3~5회로 축소)
- V3.39 — 능동 수익 극대화 (HOLD 연속 한도, trailing 자동 상향, 연속 손실 크기 축소)
- V3.30 — 전략별 성과 추적 (strategy_id)
- V3.26 — 리스크 관리 자금 보존 최우선 (일일 손실 한도, Drawdown, Circuit Breaker)
- V3.22 — 야간 인텔리전스 수집 + 프리마켓 리포트
- V3.18 — 실시간 뉴스 이벤트 엔진
- V3.10.x — 전략 개편 · 매도 하이브리드 AI · 브로커 3중 동기화 · 텔레그램 모니터링 블록
- V3.9 — 시장 분석 모듈
- V3.7 — RuntimeConfig 동적 파라미터
