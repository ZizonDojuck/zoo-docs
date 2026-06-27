# ClaudeCode 지시문 — G2 버그수정 + 조회완성 + G3 양도세 시뮬레이션

> **단계는 많지만 보고는 맨 끝에 딱 한 번.** 이주혁 시험 중이라 중간 보고 없이 끝까지 처리.
> 단 중간에 **막히면** 멈추고 한 줄로 질문(읽기 루프 금지).

## 0. 작업 전 필독

- **폴더**: `C:\Users\dlwn7\OneDrive\Desktop\주식자동화` → 검색 말고 직행.
- **읽을 것**: `CLAUDE.md` + `stock_auto.py`의 토스 구간(`_toss_get`, `toss_get_accounts`, `toss_get_holdings`, `toss_get_candles`, `_redact`, `log`, `US_PORTFOLIO`)만. **전체/반복 통독 금지.**
- **이미 확인된 사실 (그대로 신뢰)**:
  - G2a/G2b/G2c 함수 구현됨: `_toss_get`(account_seq 옵션 포함), `toss_get_accounts`, `toss_get_holdings`, `toss_get_candles`. 계좌번호 마스킹 작동.
  - G2b 검수에서 **버그 발견**: `toss_get_holdings`가 보유종목을 `result.holdings`로 파싱 → **0건**. 실제 토스 스펙 필드명은 **`result.items`**.

## 1. [버그수정] toss_get_holdings 필드명 교정 (최우선)

- `toss_get_holdings`에서 보유종목 배열 파싱을 **`result.holdings` → `result.items`** 로 수정.
- 각 item 필드(스펙 기준): `symbol`, `name`, `marketCountry`(KR/US), `currency`, `quantity`, `lastPrice`, `averagePurchasePrice`, `profitLoss.amount`, `profitLoss.rate`, `cost.commission`, `cost.tax`.
- 요약은 기존대로 `totalPurchaseAmount`, `marketValue`, `profitLoss`, `dailyProfitLoss`.
- 수정 후 재조회하여 **이번엔 종목이 실제로 들어오는지** 확인. 부록A 16종목과 대조(일치/차이 보고만, **US_PORTFOLIO 자동수정 금지**).

## 2. [조회완성] 주문 전 정보 3종 추가 (조회 전용, 계좌헤더 필요)

`_toss_get(..., account_seq=seq)` 재활용. 전부 GET. timeout=10. 실패 시 추정 없이 `(None,사유)`+log.

1. `toss_get_buying_power(account_seq, currency)` — `GET /api/v1/buying-power?currency=KRW|USD` → `cashBuyingPower`
2. `toss_get_sellable_qty(account_seq, symbol)` — `GET /api/v1/sellable-quantity?symbol=` → `sellableQuantity`
3. `toss_get_commissions(account_seq)` — `GET /api/v1/commissions` → marketCountry별 `commissionRate`

→ 이로써 토스 **조회 API는 주문 빼고 전부 커버**. (주문 생성/정정/취소는 영구 보류)

## 3. [G3 양도세] 시뮬레이션 + 한도 추적 (대안1+2)

**중요 — 토스 API로 실현손익 자동집계 불가(CLOSED 미지원 확정).** 따라서:

### 3-A. 시스템 자동: 미실현 손익 → "지금 팔면 세금" 시뮬레이션
- `toss_tax_simulate(account_seq)` 함수:
  - `toss_get_holdings`의 **US(해외) 종목만** 필터(marketCountry=="US"). 국내(KR)는 양도세 한도 대상 아님 → 제외.
  - 각 해외종목의 미실현 손익(`profitLoss.amount`, KRW 환산 기준 또는 USD→환율 환산)으로 "지금 전량 매도 시 예상 양도차익" 계산.
  - **이건 '시뮬레이션'이지 실현이 아님** 명확히 표기.

### 3-B. 이주혁 입력값으로 한도 추적
- 설정 상수/변수로 **`TAX_REALIZED_2026 = 2_420_000`** (2026 해외주식 판매수익 통산 근사치, 토스 수익분석 화면 실측 기준) 정의. 주석에 "이주혁이 토스 내계좌>수익분석>년 화면 보고 갱신하는 값. 해외주식 판매수익만, 국내·배당·이자 제외" 명시.
- `TAX_LIMIT_2026 = 2_500_000` (연 비과세 한도), `TAX_REMAINING = LIMIT - REALIZED` ≈ 80,000원.
- `toss_tax_status()` 함수: 현재 실현 누계 / 한도 / 잔여 / "사실상 소진(잔여 8만)" 경고 반환.
- 3-A 시뮬레이션과 결합: "지금 X 팔면 양도차익 +Y → 누계 (242만+Y)가 한도 250만 초과/이내" 판정.

### 3-C 주의·면책 (코드 주석 + 출력에 명시)
- 242만은 **화면 추정 근사치**. 실제 신고는 토스 연간 거래내역서/양도소득자료 기준(신고 2027.5).
- 시스템 값은 **"추가 매도 자제" 경고용**이지 세무 신고용 아님.
- 양도세는 연간 **손익 통산**(이익-손실 합산) 후 과세. 250만 한도는 **해외주식 양도소득에만** 적용(국내상장주식·배당·이자 별도).

## 4. 안전·보안 (필수)

- **조회(GET)만. 주문 절대 금지.** 주문 코드 추가 0.
- 민감정보 마스킹: 토큰·키·계좌번호. 공개데이터(시세·OHLCV·종목명·손익률·양도세 계산값)는 평문 가능.
- timeout=10 / 수집 실패 시 추정 없이 (None,사유)+log(R7).
- **self_check 무겁게 금지**: 기존 토큰1+시세1 유지. holdings·tax·buying-power 등을 self_check에 넣지 말 것(Rate Limit·시간).
- US_PORTFOLIO 자동수정 금지(대조는 보고만).

## 5. 문서 갱신 (노트북 CLAUDE.md = SSOT)

- R20(양도세) 행: **🔶** — "토스 미실현 손익 기반 세금 시뮬레이션(toss_tax_simulate) + 이주혁 입력값(242만) 한도추적(toss_tax_status) 구현. 실현 자동집계는 토스 CLOSED 미지원으로 불가, 수동입력 방식 확정."
- R21 행: **🔶** 유지 — "조회 API 주문 빼고 전부 커버(holdings 버그수정·buying-power·sellable·commissions)."
- 10.3 게이트: G2 완료(조회 전부) → G3 양도세 시뮬 구현 → 검수 대기.
- 0.2 갱신이력 한 줄.

## 6. 검증 후 보고 (맨 끝 1회, 실측)

1. `py_compile` (stock_auto.py, self_check.py)
2. **holdings 버그수정 검증**: `toss_get_holdings(1)` 재조회 → 이번엔 종목 N건 들어오는가(symbol/name/수익률 일부), 요약. **부록A 16종목 대조 결과.**
3. `toss_get_buying_power(1,"KRW")` / `(1,"USD")` 각 1회 → 매수가능금액
4. `toss_get_sellable_qty(1,"005930")` 1회 → 매도가능수량 (보유 시)
5. `toss_get_commissions(1)` 1회 → KR/US 수수료율
6. `toss_tax_status()` → 누계 242만/한도 250만/잔여 8만/소진경고
7. `toss_tax_simulate(1)` → 해외종목만 필터됐는가, "지금 팔면 양도차익" 시뮬값 + 면책문구
8. self_check 1회 → 토스 표기·종합등(holdings·tax 미추가 확인)
9. 보안 스캔 → 토큰·키·계좌번호 노출 0건

**형식**: 실측값(추정 아님). 토큰·계좌번호 마스킹. 주문은 "코드 없음·미접촉". 양도세는 "시뮬레이션/경고용, 신고용 아님" 명시.

## 7. 금지 재확인
- ❌ 주문 생성·정정·취소(영구) ❌ 토큰·계좌번호 평문 ❌ US_PORTFOLIO 자동수정 ❌ self_check에 무거운 호출 추가 ❌ 양도세를 "실현 자동집계"로 구현(불가, 입력 방식) ❌ 전체 반복 통독

---
**완료 후**: 채팅 Claude 검수 → 통과 시 다음 우선순위(E1~E6 개선과제 또는 인프라 GitHub Actions).
