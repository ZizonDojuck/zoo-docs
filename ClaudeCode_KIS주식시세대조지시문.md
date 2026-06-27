# ClaudeCode 지시문 — KIS 국내주식 시세·일봉 + 3자 대조검증

> 목적: KIS(증권사)로 국내주식 현재가·일봉을 받고, **토스·네이버와 3자 대조**해 일치 확인 후 시세·차트 소스에 KIS를 1차로 승격 검토.
> 보안 최상위(실전키). 조회 전용·주문 0줄. 보고는 맨 끝 1회. 막히면 멈추고 한 줄 질문.

## 0. 🔒 보안 (변함없음)
- 조회·시세 전용. **주문 코드 0줄.** 키·토큰·계좌 마스킹, 보고에 키값 금지.
- REST 토큰 디스크 캐시(kis_token.json) 재사용. 재발급 최소화(KIS 1일1회). timeout=10.

## 1. 작업 전 필독
- **폴더**: `C:\Users\dlwn7\OneDrive\Desktop\주식자동화` → 직행.
- **읽을 것**: KIS 구간(`get_kis_token`, `_kis_get`), 토스 시세(`toss_get_price`, `toss_get_candles`), 네이버 시세·일봉 수집 함수, `analyze_timeframes`/`get_candles_multi`(E3), `_redact`, `log`. 반복 통독 금지.
- **확인된 KIS 엔드포인트(웹조사, 추측 아님)**:
  - **현재가 시세**: `GET /uapi/domestic-stock/v1/quotations/inquire-price`, tr_id **`FHKST01010100`**, params `FID_COND_MRKT_DIV_CODE='J'`(주식/ETF/ETN), `FID_INPUT_ISCD=종목6자리`. 응답 `output.stck_prpr`=현재가, `output.prdy_ctrt`=전일대비율 등.
  - **일봉(기간별)**: `GET /uapi/domestic-stock/v1/quotations/inquire-daily-itemchartprice`, tr_id **`FHKST03010100`**, params `FID_COND_MRKT_DIV_CODE='J'`, `FID_INPUT_ISCD=종목`, `FID_INPUT_DATE_1=시작YYYYMMDD`, `FID_INPUT_DATE_2=종료YYYYMMDD`, `FID_PERIOD_DIV_CODE='D'`(일/W주/M월/Y년), `FID_ORG_ADJ_PRC='0'`(수정주가) 또는 '1'(원주가). 응답 `output2`=일자별 배열(stck_bsop_date 일자, stck_oprc 시가, stck_hgpr 고가, stck_lwpr 저가, stck_clpr 종가, acml_vol 거래량).

## 2. 구현: KIS 시세·일봉 조회 함수

### [2-A] `kis_get_price(symbol)` — 국내주식 현재가
- `_kis_get`으로 inquire-price 호출(tr_id FHKST01010100). `output.stck_prpr`(현재가)·`prdy_ctrt`(등락률)·시각 파싱. float 변환.
- 실패 시 추정 없이 (None,사유)+log. timeout=10.

### [2-B] `kis_get_daily(symbol, count=100)` — 국내주식 일봉
- inquire-daily-itemchartprice(tr_id FHKST03010100), 기간=최근 count 영업일 근사(시작일=오늘-약 150일, 종료일=오늘). `FID_PERIOD_DIV_CODE='D'`.
- `output2` 배열에서 OHLCV+일자 파싱. **KIS는 보통 최신→과거(descending) 반환** → 토스 정렬버그 교훈대로 **반환 직전 일자 오름차순 정렬**(과거→최신, [-1]=최신).
- `FID_ORG_ADJ_PRC`는 **수정주가('0')** 기본(다른 소스와 비교 위해 일관). 실패 시 (None,사유)+log.

## 3. [핵심] 3자 대조검증 (토스 정렬버그 교훈 — 추측으로 승격 금지)

### [3-A] 현재가 3자 대조
- 같은 종목(예 005930 삼성전자, 000660 SK하이닉스)으로 **KIS·토스·네이버 현재가**를 동시 조회해 비교.
- 셋이 일치(허용오차 작게, 예 ±0.5% 또는 호가단위)하는지 확인. 토요일이라 직전거래일(금) 종가 기준일 수 있음 — 시각/기준일 표시.
- **불일치 시**: 어느 소스가 튀는지 기록(토스 정렬버그처럼 원인 추적). 추측으로 "KIS가 맞다" 단정 금지.

### [3-B] 일봉 최신값 3자 대조
- KIS 일봉 [-1](최신)·토스 일봉 최신·네이버 일봉 최신의 OHLC 비교. 정렬 방향(최신이 [-1]인지) 확인 — KIS도 descending이면 정렬 교정 적용됐는지 검증.

### [3-C] 판정
- **3자 일치 시**: "KIS 시세·일봉 검증 통과, 시세·차트 소스 우선순위에 KIS 1차 승격 가능" 결론. 단 **코드의 기본 소스를 자동 교체하지는 말 것** — 승격은 이주혁/채팅Claude 확인 후. 이번엔 검증+함수 추가까지.
- **불일치 시**: 원인 기록, 승격 보류, 어느 소스가 정답인지 미정으로 정직 보고.

## 4. 안전·범위
- 조회 전용·주문 0줄. 마스킹. timeout=10. 실패 시 추정 없이 (None,사유)+log(R7).
- **self_check 무겁게 금지**: KIS 시세·일봉 호출을 self_check에 넣지 말 것.
- 기존 토스/네이버 소스 자동 교체 금지(대조 결과만 보고, 승격은 별도 승인).
- 외부 라이브러리 추가 없이 requests로.

## 5. 문서 갱신 (CLAUDE.md = SSOT)
- KIS 주식시세: "kis_get_price/kis_get_daily 구현(tr_id FHKST01010100/FHKST03010100). 3자 대조검증 [통과/불일치] 결과. 승격 [가능/보류]." 0.2 갱신이력. 추정 금지(실측 근거).

## 6. 보고 (맨 끝 1회, 실측)
1. `py_compile`
2. `kis_get_price("005930")`·`kis_get_daily("005930")` 실측값(현재가, 일봉 최신 OHLCV, 정렬 방향)
3. **3자 대조표**: 005930·000660의 KIS/토스/네이버 현재가 + 일봉 최신 종가 비교 (일치/불일치, 기준일·시각)
4. **판정**: 3자 일치하는가 → KIS 승격 가능/보류 + 근거. 불일치면 어느 소스가 튀는지.
5. 토큰 재발급 안 했는지(캐시) / self_check / 보안스캔 노출0.

**형식**: 실측. 시세·종목코드 평문 가능. 키·토큰·계좌 마스킹. 주문 "코드 없음". 추정과 사실 구분. 승격은 "검증까지, 자동교체 안 함" 명시.

## 7. 금지
- ❌ 시세 불일치를 추측으로 "KIS가 맞다" 단정 ❌ 기존 소스 자동 교체(승격은 승인 후) ❌ 주문 ❌ 키·토큰·계좌 평문 ❌ self_check에 KIS 시세 추가 ❌ 토큰 불필요 재발급 ❌ 반복 통독

---
**완료 후**: 채팅 Claude 검수. 3자 일치 확인되면 KIS를 시세·차트 1차 소스로 승격 결정(E3 다중타임프레임에 KIS 차트 연결).
