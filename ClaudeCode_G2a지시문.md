# ClaudeCode 지시문 — G2a: 토스 시세·환율 조회 연동

## 0. 작업 전 필독 (시간 낭비 방지)

- **프로젝트 폴더**: `C:\Users\dlwn7\OneDrive\Desktop\주식자동화`
  → 폴더 검색하지 말고 이 경로로 직행.
- **읽을 파일은 딱 2개**: `CLAUDE.md`, 그리고 `stock_auto.py`의 토스 관련 구간(`get_toss_token`, `get_toss_creds`, `_redact`, `_tg_post`, `log` 헬퍼 주변)만.
  → `stock_auto.py` 전체를 반복 통독하지 말 것. 같은 파일을 두 번 이상 통독 금지.
- **이미 확인된 사실 (그대로 신뢰, 재확인 위해 코드 다시 읽지 말 것)**:
  - `get_toss_token(force=False)` 는 G1에서 **실측 발급 성공**(HTTP 200, expires_in≈24h). 토큰 캐시·만료 30초 전 재사용 골격 있음. timeout=10 걸려 있음.
  - `_redact` 는 토스 JWT/Bearer/c_·s_접두사/키이름리터럴 4종을 마스킹. **이미 작동 확인됨**.
  - 토스 base URL = `https://openapi.tossinvest.com`. 인증 = `Authorization: Bearer {access_token}`.
  - 시세·환율 엔드포인트는 **계좌 헤더 불필요**(토큰만으로 호출 가능).

## 1. 이번 작업 범위 (G2a — 여기서 끝, 넘지 말 것)

토스 토큰으로 **시세·환율 조회 2종만** 구현한다. **계좌 정보(계좌목록/보유주식)·주문은 일절 건드리지 않는다** (G2b 이후).

구현할 함수 2개 (둘 다 GET, 조회 전용):

1. **`toss_get_price(symbol)`** — 현재가 조회
   - `GET /api/v1/prices?symbols={symbol}` (예: `005930`)
   - 헤더: `Authorization: Bearer {get_toss_token()}`
   - 응답에서 `result[0].lastPrice`, `currency`, `timestamp` 파싱하여 dict 반환
   - timeout=10

2. **`toss_get_fx_usdkrw()`** — USD→KRW 환율 조회
   - `GET /api/v1/exchange-rate?baseCurrency=USD&quoteCurrency=KRW`
   - 헤더: `Authorization: Bearer {get_toss_token()}`
   - 응답에서 `result.rate`, `rateChangeType`, `validUntil` 파싱하여 dict 반환
   - timeout=10

## 2. 안전·보안 수칙 (G2 전체 공통, 반드시 준수)

- **조회(GET)만.** 주문(POST /orders 등) 절대 호출 금지. 이번 작업에 주문 코드 추가 없음.
- **토큰 평문 노출 0**: 모든 요청·응답·예외 로깅은 `log()`(=`_redact` 경유)로만. `print(token)` 류 금지. Authorization 헤더 값이 로그에 찍히지 않도록 주의.
- **timeout=10** 전 호출에 적용.
- **수집 실패 시 GO 금지 원칙 유지**: 호출 실패(타임아웃/4xx/5xx/파싱실패) 시 예외를 삼키지 말고, 실패를 명확히 반환(None 또는 에러 dict)하고 `log()`에 사유 기록. 추정값으로 채우지 말 것.
- **Rate Limit 주의**: 토스는 그룹별 TPS 제한 있음(429 시 `Retry-After` 헤더). 이번엔 단발 호출이라 문제없지만, self_check에 넣을 때 과다 호출 금지(아래 4항).
- **키 입력 금지**: `.env`의 토스 키는 이주혁이 이미 넣음. ClaudeCode는 키를 입력·수정·출력하지 않는다.

## 3. self_check 반영 (가볍게)

- self_check의 토스 점검에 **시세 1건(`toss_get_price("005930")`)을 추가**하여 "조회 연동 정상" 여부를 표기.
  - 성공 시: `토스 조회 OK(시세 1건)` 같은 표기.
  - 실패 시: `토스 조회 실패(사유)`.
- **과다 호출 금지**: self_check에서 토스 호출은 **토큰 발급 1회 + 시세 1건**까지만. 환율·차트 등 추가 호출로 self_check를 무겁게 만들지 말 것. (self_check는 자주 돌리므로 Rate Limit·시간 부담 회피)

## 4. 문서 갱신 (노트북 원본 CLAUDE.md = SSOT)

- R21 행: 상태 `🔶` 유지하되, 진척 메모에 **"G2a 시세·환율 조회 구현(toss_get_price/toss_get_fx_usdkrw), 조회 전용·계좌 미접촉"** 추가.
- 10.3 게이트 표: **G2a 시세·환율 조회 구현 → 채팅 Claude 검수 대기**. (G2b 보유주식, G2c 차트는 ⬜ 유지)
- 0.2 갱신이력: G2a 한 줄.
- 추정 금지: 실제로 구현·실행한 것만 기록. "잘 될 것"이 아니라 "실행해서 200 받음"으로.

## 5. 검증 후 보고 (실측 기준)

다음을 실제로 실행하고 결과를 보고한다 (각 1회만, 조회 전용):

1. `py_compile` 통과 여부 (stock_auto.py, self_check.py)
2. `toss_get_price("005930")` 1회 실행 → HTTP 상태, 파싱된 현재가·통화·시각 (토큰은 마스킹)
3. `toss_get_fx_usdkrw()` 1회 실행 → HTTP 상태, 파싱된 환율·등락·유효시각
4. self_check 1회 실행 → 토스 조회 표기, 종합등
5. 보안 스캔 → 토큰·키 평문 노출 0건 확인

**보고 형식**: 각 항목 실측값(추정 아님). 시세·환율 값은 그대로 보여줘도 무방(민감정보 아님). **단 토큰은 반드시 마스킹.** 호출 0건이어야 하는 항목(계좌·주문)은 "미접촉" 명시.

## 6. 금지사항 재확인 (G2a)

- ❌ 계좌목록·보유주식·매수가능금액·매도가능수량 조회 (G2b 이후)
- ❌ 차트/캔들 조회 (G2c 이후)
- ❌ 주문 생성·정정·취소 (영구 금지, 별도 승인 전까지)
- ❌ 토큰 평문 출력/로깅
- ❌ stock_auto.py 전체 반복 통독

---
**완료 후**: 채팅 Claude(기획자) 검수 → 통과 시 G2b(계좌목록→보유주식)로.
