# ClaudeCode 지시문 — G2b+G2c 통합: 보유주식 동기화 + 차트 조회

> **이 지시문은 작업을 여러 단계로 나누지만, 보고는 맨 끝에 딱 한 번만 한다.**
> 이주혁(고객)이 시험 공부 중이라 중간 보고 없이 끝까지 처리 후 1회 보고.
> 단, 단계 중간에 **막히면** 멈추고 한 줄로 질문(읽기 루프에 빠지지 말 것).

## 0. 작업 전 필독 (시간 낭비 방지)

- **프로젝트 폴더**: `C:\Users\dlwn7\OneDrive\Desktop\주식자동화` → 검색 말고 직행.
- **읽을 파일**: `CLAUDE.md` + `stock_auto.py`의 토스 구간(`get_toss_token`, `_toss_get`, `toss_get_price`, `_redact`, `log`, `US_PORTFOLIO`/감시종목 리스트 주변)만. **전체 반복 통독 금지. 같은 파일 두 번 통독 금지.**
- **이미 확인된 사실 (그대로 신뢰, 재확인 위해 코드 다시 읽지 말 것)**:
  - G2a에서 `_toss_get(path, params)` 공통 헬퍼 완성: 토큰→Bearer→GET→상태/파싱→실패시 `(None,사유)`+log, 429 처리, timeout=10. **이 헬퍼를 그대로 재활용**한다.
  - `get_toss_token` 발급 성공 실측됨. `_redact` 토스 마스킹 작동.
  - 토스 base = `https://openapi.tossinvest.com`.
  - **계좌 헤더가 필요한 API**는 `X-Tossinvest-Account: {accountSeq}` 헤더를 추가로 보내야 함.

## 1. 이번 작업 범위 (조회 전용 — 주문 절대 금지)

**4개 함수**를 구현한다. 전부 GET. 주문(POST) 코드는 추가하지 않는다.

### [1단계] 계좌번호 마스킹 먼저 (보안 선행)
- `_redact`에 **계좌번호(accountNo) 마스킹 패턴 추가**. 계좌번호는 보통 10~11자리 숫자 문자열.
  - 단, 종목코드(6자리)·가격·수량 등 다른 숫자를 오탐하지 않도록 **충분히 길고 구체적인 패턴**으로(예: 10자리 이상 연속 숫자, 또는 `accountNo` 키 리터럴 앵커). 오탐 방지를 G1 스캐너 때처럼 신중히.
  - `_toss_get_account_helper`가 만들어지면 그 경로로 나가는 응답 로깅도 마스킹되는지 확인.

### [2단계] 계좌 헤더용 공통 헬퍼
- `_toss_get_account(path, params, account_seq)` — `_toss_get` 골격에 `X-Tossinvest-Account: {account_seq}` 헤더를 추가한 변형. (또는 `_toss_get`에 선택적 account_seq 파라미터 추가 — 구현 편한 쪽)
- 실패 처리·timeout·마스킹은 `_toss_get`과 동일 원칙.

### [3단계] 계좌목록 → accountSeq 확보
- `toss_get_accounts()` — `GET /api/v1/accounts`
  - 응답 `result[]`에서 `accountSeq`, `accountType` 파싱. **accountNo는 마스킹 경유로만 로깅.**
  - 종합매매(BROKERAGE) 계좌의 `accountSeq` 반환.
  - 계좌 없으면 빈 배열 → 명확히 "계좌 없음" 반환(추정 금지).

### [4단계] 보유주식 조회
- `toss_get_holdings(account_seq)` — `GET /api/v1/holdings` + 계좌 헤더
  - 응답에서 종목별 `symbol`, `name`, `quantity`, `averagePurchasePrice`, `lastPrice`, `profitLoss.rate`, `cost(commission/tax)` 파싱.
  - 요약(`totalPurchaseAmount`, `profitLoss`)도 파싱.
  - **부록 A 16종목과 대조용**으로 보유 종목 리스트를 정리(단 대조 판단은 보고만 하고, 코드의 US_PORTFOLIO를 자동 수정하지는 말 것 — 이주혁 확인 후 별도 반영).

### [5단계] 차트(캔들) 조회 — R18 다중타임프레임 기반
- `toss_get_candles(symbol, interval, count=100)` — `GET /api/v1/candles?symbol=&interval=&count=`
  - `interval`은 `1m`(분봉) 또는 `1d`(일봉)만 지원(토스 스펙). 그 외 값은 호출 전에 거부.
  - 응답 `result.candles[]`에서 OHLCV(`openPrice/highPrice/lowPrice/closePrice/volume`)+`timestamp` 파싱. `nextBefore`(페이지네이션)도 보존.
  - **계좌 헤더 불필요**(토큰만).
  - 이건 R18(분봉/일봉/주봉 다중타임프레임) 확장의 토스측 소스다. 단, 토스는 주봉(`1w`) 미지원 → **주봉은 일봉을 집계하거나 기존 소스(네이버/yfinance) 사용**. 이번엔 토스 분봉·일봉 조회 함수까지만 만들고, 다중타임프레임 통합 분석 로직은 다음 단계로 남긴다(과욕 금지).

## 2. 안전·보안 수칙 (반드시 준수)

- **조회(GET)만. 주문 절대 금지.** 이번 작업에 주문 코드 없음.
- **민감정보 전부 마스킹**: 토큰(JWT/Bearer), client_id/secret, **계좌번호(accountNo)**, 텔레그램 토큰, OpenDART 키 등. 전부 `log()`(=`_redact`) 경유. accountSeq(순번)도 보수적으로 다룸.
- **공개 시장데이터는 마스킹 안 함**: 시세·환율·종목명·OHLCV·평가손익률 등은 그대로 표기 가능(검수·디버깅 위해).
- **timeout=10** 전 호출.
- **수집 실패 시 GO 금지(R7)**: 실패 시 추정 없이 `(None,사유)` 반환+log.
- **Rate Limit**: 토스 그룹별 TPS 제한. 캔들은 `MARKET_DATA_CHART` 그룹(별도 한도). self_check에 차트·계좌 호출을 **넣지 말 것**(self_check는 기존대로 토큰1+시세1만 유지, 무겁게 금지).
- **US_PORTFOLIO 자동수정 금지**: 보유 대조 결과는 보고만. 코드 종목 리스트 변경은 이주혁 확인 후.
- **키 입력 금지**: 토스 키는 이주혁이 이미 .env에 넣음. ClaudeCode는 키 입력·수정·출력 안 함.

## 3. 문서 갱신 (노트북 원본 CLAUDE.md = SSOT)

- R21 행: `🔶` 유지(조회 확장됐으나 주문 미구현). 진척 메모에 "G2b 계좌목록·보유주식, G2c 차트(분봉/일봉) 조회 구현. 계좌번호 마스킹 추가. 조회 전용."
- 10.3 게이트 표: **G2b·G2c 구현 → 채팅 Claude 검수 대기**. (G3 양도세는 ⬜ 유지)
- 0.2 갱신이력: G2b+G2c 한 줄.
- 추정 금지: 실행해서 받은 것만 기록.

## 4. 검증 후 보고 (맨 끝 1회 — 실측 기준)

순서대로 실행 후, **한 번에** 보고한다:

1. `py_compile` 통과 (stock_auto.py, self_check.py)
2. **계좌번호 마스킹 검증**: 가짜 계좌번호 문자열로 `_redact` 테스트 → 마스킹되는가 + 종목코드(6자리)·가격 오탐 0건인가 (G1 스캐너 때처럼 가짜키 투입식 자가검증)
3. `toss_get_accounts()` 1회 → accountSeq 확보 여부, accountType. **accountNo는 마스킹 표기로만.**
4. `toss_get_holdings(account_seq)` 1회 → 보유 종목 수, 종목 리스트(symbol/name/수익률), 요약. **부록 A 16종목 대조 결과(일치/차이) 보고.**
5. `toss_get_candles("005930","1d",count=5)` 1회 → 받은 봉 개수, 최근 1봉 OHLCV
6. `toss_get_candles("005930","1m",count=5)` 1회 → 받은 봉 개수, 최근 1봉 OHLCV (정규장/시간외 timestamp 확인)
7. self_check 1회 → 토스 표기·종합등(차트·계좌 미추가 확인)
8. 보안 스캔 → 토큰·키·**계좌번호** 평문 노출 0건

**보고 형식**: 각 실측값(추정 아님). 토큰·계좌번호 마스킹 필수. 시세·OHLCV·종목명은 평문 가능. 주문은 "코드 없음·미접촉" 명시.

## 5. 금지사항 재확인

- ❌ 주문 생성·정정·취소 (영구 금지)
- ❌ 매수가능금액·매도가능수량(`/buying-power`,`/sellable-quantity`) — 이번 범위 아님(다음 단계)
- ❌ 토큰·계좌번호 평문 출력/로깅
- ❌ US_PORTFOLIO 자동 수정 (보고만)
- ❌ self_check에 차트·계좌 호출 추가 (무겁게 금지)
- ❌ stock_auto.py 전체 반복 통독

---
**완료 후**: 채팅 Claude 검수 → 통과 시 G3(양도세, 단 메모리상 토스 CLOSED 미지원으로 대안1+2 방식) 또는 다음 우선순위로.
