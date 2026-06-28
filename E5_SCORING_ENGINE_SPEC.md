# E5 — 뉴스 점수화 엔진 설계 명세서 (SSOT)

> ZOO 프로젝트 / 미국 가치주 트랙 4기둥 ③미래성장성 연결 모듈
> 작성: 기획(채팅 Claude) → 구현: ClaudeCode
> 상태: 설계 확정, 구현 대기
> 원칙 준수: 수집실패시 추정금지·GO보류 / 실주문 절대금지(조언만) / 비용 캐시 필수 / 대조검증 후 승격

---

## 0. 목적 (한 문장)

미국 가치주 보유·감시 종목의 뉴스를 LLM으로 구조화 점수화하여,
4기둥 중 ③미래성장성(가이던스·신사업·메가트렌드·회사 분위기)에 정량 신호로 연결한다.
이로써 미국 가치주 트랙을 완성하고, 한국 단타와 함께 2트랙을 완성한다.

E5는 **신호 생성기이지 주문기가 아니다.** 출력은 "조언"이며 최종 판단은 이주혁 본인.

---

## 1. 아키텍처 — 하이브리드 (확정)

```
[뉴스 수집]  →  [1차 룰베이스 필터/우선순위 큐]  →  [2차 LLM 점수화(Sonnet)]  →  [③미래성장성 연결]
   FMP뉴스          LLM 없음·무료                    claude-sonnet-4-6           가치주 종합점수에 가중
                    호출 대상 선별·랭킹               캐시 필수·실패시 null
```

### 1.1 설계 철학 (왜 하이브리드인가 — 변경 금지 근거)
- 순수 LLM 전량 호출 = 정밀하나 호출량 폭증·비용 과다.
- 순수 룰베이스 = 무료지만 stance/theme 뉘앙스 못 잡아 ③미래성장성 목적 미달.
- 하이브리드 = 1차로 "점수화할 가치 있는 뉴스"만 골라 LLM 호출 1/5~1/10로 절감.
- 장애 격리: LLM 실패해도 해당 뉴스만 null 처리, 파이프라인 전체는 생존.

### 1.2 1차 필터는 "탈락"이 아니라 "우선순위 큐" (핵심 설계)
- 필터가 빡빡하면 과탈락(중요뉴스 누락), 느슨하면 절감 무력화 → 둘 다 위험.
- 해결: 뉴스를 버리지 않고 **중요도 점수로 랭킹**한다.
- 일 LLM 호출 한도(LLM_DAILY_BUDGET)까지 상위부터 처리.
- 한도 초과분은 폐기하지 않고 **다음 사이클 이월(carry-over)**.

---

## 2. 모델 계층 (확정)

| 단계 | 처리 | 모델/방식 | 비용 |
|------|------|-----------|------|
| 1차 필터·랭킹 | 호출 대상 선별 | 룰베이스(코드) | 무료 |
| 2차 점수화 | sentiment·stance·theme·takeaway 추출 | `claude-sonnet-4-6` | 유료(캐시로 절감) |
| 타깃 에스컬레이션 | 사후검증서 입증된 "어려운 유형"만 | 상위 모델 | 예외적·선별적 |

- **Opus는 기본값에서 제외.** 분류·라벨링 작업이라 Opus의 추론 강점이 발휘 안 됨 + 단가 약 5배.
- 에스컬레이션은 "전량 Opus"가 아니라, 검증으로 오라벨률 높다고 입증된 특정 뉴스 유형에만 타깃 적용.
- few-shot 예시 2~3개를 프롬프트에 넣어 Sonnet 정밀도를 끌어올린다(모델 격상보다 비용 효율 우위).

---

## 3. 데이터 스키마 (뉴스 데이터센터 1·2차에서 null 예약된 필드를 채움)

뉴스 1건 = 1 레코드. E5가 채우는 필드:

```json
{
  "news_id": "string (URL 해시 등 고유키, 캐시 키로도 사용)",
  "ticker": "string (NFLX, AMZN, TSM, GOOGL, AAPL ...)",
  "url": "string",
  "published_at": "ISO8601",
  "title": "string",

  // ── 1차 룰베이스 산출 ──
  "relevance_score": "float 0~1 (종목 관련도)",
  "priority_score": "float (LLM 호출 우선순위 랭킹값)",
  "rule_label": "enum: positive|neutral|negative (룰베이스 1차 라벨, 대조검증용)",

  // ── 2차 LLM 산출 (E5 핵심, 기존 null 예약 필드) ──
  "sentiment": "enum: strong_positive|positive|neutral|negative|strong_negative | null",
  "stance": "enum: bullish|neutral|bearish | null  (논조: 종목 미래에 대한 방향)",
  "theme": "enum: guidance|new_business|megatrend|earnings|regulation|management|macro|other | null",
  "takeaway": "string ≤140자, 한국어 한 줄 시사점 | null",
  "growth_signal": "float -1.0~+1.0 | null  (③미래성장성 연결용 정규화 점수)",

  // ── 메타 ──
  "scored_by": "enum: sonnet-4-6|escalated|rule_only | null",
  "scored_at": "ISO8601 | null",
  "score_status": "enum: ok|llm_failed|skipped_budget|carried_over",
  "fail_reason": "string | null"
}
```

### 3.1 신호 강도 매핑 (E4 등급·매매 공통틀과 통일)
`growth_signal` → 5단계 강도로 환산(단정 금지, 강도 표현):

| growth_signal | 강도 라벨 |
|---------------|-----------|
| +0.6 ~ +1.0 | 강한 긍정 |
| +0.2 ~ +0.6 | 약한 긍정 |
| -0.2 ~ +0.2 | 중립 |
| -0.6 ~ -0.2 | 약한 부정 |
| -1.0 ~ -0.6 | 강한 부정 |

---

## 4. 1차 룰베이스 필터 규칙

### 4.1 relevance_score (관련도)
- 제목·본문에 ticker/회사명/주요 제품명 직접 언급 → 가점.
- 단순 시황 나열·지수 보도·타 종목 중심 기사 → 감점.
- relevance_score < THRESHOLD_REL(잠정 0.3) 이면 LLM 호출 대상에서 제외(큐 진입 안 함).

### 4.2 priority_score (LLM 호출 랭킹)
가중 합산(잠정 가중치, 사후검증 조정 대상):
- 관련도 relevance_score × 0.4
- 최신성(published_at 가까울수록) × 0.3
- theme 키워드 강도(guidance/earnings/regulation 등 ③미래성장성 직결 키워드 가점) × 0.3

### 4.3 rule_label (대조검증용 1차 라벨)
- 감정 키워드 사전 기반 positive/neutral/negative.
- **이 라벨은 신호로 직접 쓰지 않는다.** 2차 LLM 라벨과 대조해 필터 품질을 사후검증하는 용도.

---

## 5. 2차 LLM 점수화

### 5.1 호출 정책
- priority_score 내림차순으로 LLM_DAILY_BUDGET(잠정 50회/일)까지 호출.
- 초과분 → score_status = "carried_over", 다음 사이클 우선 처리.
- 동일 news_id 캐시 히트 시 LLM 미호출(비용 0). FMP 캐시 패턴 재사용.

### 5.2 프롬프트 골격 (구조화 출력 강제)
- 시스템: "너는 미국 주식 뉴스 분석가다. 아래 뉴스를 읽고 JSON만 출력하라. 설명·서문·마크다운 금지."
- few-shot: 라벨링 예시 2~3건 포함(bullish/bearish/neutral 각 1건).
- 출력 필드: sentiment, stance, theme, takeaway, growth_signal.
- 파싱: ```json 펜스 제거 후 안전 파싱. 파싱 실패 시 1회 재시도 → 실패시 null.

### 5.3 ③미래성장성 연결
- growth_signal을 미국 가치주 종합 판정의 4기둥 ③미래성장성 입력으로 가중 반영.
- 4기둥 다른 축(밸류·펀더멘털·타이밍)과 충돌 시: 종합결론 + 주석 경고 병기(NFLX 선례 방식).

---

## 6. 실패 처리 (프로젝트 원칙 ① 엄격 준수)

| 상황 | 처리 |
|------|------|
| 뉴스 수집 실패 | score_status=llm_failed 아님 — 수집 단계 실패는 GO보류 + 텔레그램 "수집 실패" |
| LLM 호출 실패(API 오류) | 1회 재시도 → 실패시 sentiment 등 null, fail_reason 기록, GO보류 |
| JSON 파싱 실패 | 1회 재시도 → 실패시 null |
| 예산 초과 | carried_over (실패 아님, 이월) |

**추정으로 빈칸을 채우지 않는다.** null은 null로 두고 신호 보류한다.

---

## 7. 대조검증 & 승격 (토스 정렬버그 교훈)

- rule_label vs LLM stance 를 같은 뉴스에 동시 부여 → 불일치율 로깅.
- 1차 필터 과탈락(중요뉴스인데 큐 진입 실패)·과통과 사후 점검.
- 오라벨률 높은 특정 theme 발견 시 → 그 theme만 타깃 에스컬레이션 후보.
- 충분한 검증 데이터 축적 전까지 E5 growth_signal은 **보조 가중**으로만(메인 신호 단독 결정 금지).

---

## 8. 보안 (반복 검증된 원칙)

- ANTHROPIC_API_KEY 등 모든 키 .env에만. 평문 하드코딩 금지. .gitignore 확인.
- _redact로 로그의 키·토큰 마스킹.
- GitHub Actions 사용 시 Secrets로만.
- 캐시 파일에 키 값 저장 금지(뉴스 본문·점수만).

---

## 9. .env 신규 변수 (제안 — 값 입력은 이주혁만)

```
ANTHROPIC_API_KEY=        # E5 2차 점수화용
E5_MODEL=claude-sonnet-4-6
E5_LLM_DAILY_BUDGET=50    # 일 LLM 호출 상한(잠정, 조정 대상)
E5_REL_THRESHOLD=0.3      # 관련도 컷(잠정)
E5_ESCALATE_MODEL=        # 타깃 에스컬레이션용 상위 모델(비워두면 비활성)
```

---

## 10. 완료 정의 (DoD)

E5는 다음을 모두 충족할 때 완성:
1. 1차 필터·우선순위 큐·이월 동작 확인.
2. 2차 Sonnet 호출 → 스키마대로 JSON 산출 → 캐시 적중 확인.
3. 실패 3종(LLM오류·파싱실패·예산초과) 각각 명세대로 처리 확인.
4. growth_signal이 4기둥 ③미래성장성에 가중 반영됨.
5. rule_label vs LLM stance 불일치율 로깅 동작(대조검증 인프라).
6. 보안 5단계 스캔 PASS(키 노출 0).
7. DRY_RUN / 주문코드 0줄 유지(E5는 신호만, 주문 없음).

부분 동작을 완성으로 착각 금지.
