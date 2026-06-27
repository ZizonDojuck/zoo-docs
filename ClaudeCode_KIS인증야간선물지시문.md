# ClaudeCode 지시문 — KIS(한국투자증권) 인증 + 야간선물 시세 조회

> **실전 앱키 사용. 보안 최상위.** 토스와 동급. 조회(시세) 전용·주문 코드 0줄.
> 보고는 맨 끝 1회. 막히면 멈추고 한 줄 질문. 단계 분기 구조(인증→야간선물 확인).

## 0. 🔒 보안 최우선 (실전 키 — 반드시 먼저 읽고 지킬 것)

- KIS 앱키/시크릿은 **실전 계좌를 움직일 수 있는 열쇠**다. 토스와 동급 최상위 보안.
- **조회(시세) 전용. 주문(매수/매도/정정/취소) 코드 절대 0줄.** 토스처럼 DRY_RUN 정신 동일 적용.
- 앱키·시크릿·계좌번호·토큰은 **`.env`에서만** 읽고, **`_redact`로 마스킹**(아래 1-A). 로그·출력·예외 어디에도 평문 금지.
- `.gitignore`에 `.env` 등록 확인.
- **ClaudeCode는 키를 발급·입력·출력하지 않는다**(이주혁이 이미 .env에 넣음). 키 값을 보고에 찍지 말 것.

## 1. 작업 전 필독 + 마스킹 선행

- **폴더**: `C:\Users\dlwn7\OneDrive\Desktop\주식자동화` → 직행.
- **읽을 것**: `stock_auto.py`의 토스 인증/헬퍼 구간(`get_toss_token`, `_toss_get` — KIS 인증을 같은 패턴으로 만들 참고), E2 야간선물 구간(`get_kospi_night_futures`, `get_us_futures`, `assess_kospi_gap_signal`), `_redact`, `log`. 반복 통독 금지.
- **KIS는 공식 샘플(kis_auth.py/kis_devlp.yaml)을 통째로 가져오지 말 것.** 우리 방식(.env + 직접 requests 헬퍼)으로 인증만 구현. 공식 GitHub는 엔드포인트·헤더 참고만.

### [1-A] `_redact`에 KIS 키 마스킹 추가 (코드 작성 전에 먼저)
- `KIS_APP_KEY`, `KIS_APP_SECRET`, KIS 접근토큰(Bearer), `KIS_ACCOUNT_NO` 마스킹 패턴 추가.
- 토스 마스킹과 같은 방식(키 이름 리터럴 앵커 + 토큰 형태). 종목코드·가격 등 오탐 0 유지(가짜키 자가검증).

## 2. KIS 환경/인증 (.env 기반)

### .env 변수 (이주혁이 입력함 — 읽기만)
```
KIS_APP_KEY, KIS_APP_SECRET, KIS_ACCOUNT_NO(선물옵션 8자리앞, 비어있을 수 있음),
KIS_ACCOUNT_PROD=03(국내선물옵션), KIS_ENV=real
```
- 도메인: `KIS_ENV=real` → **`https://openapi.koreainvestment.com:9443`** (모의 `openapivts`는 다름, 우리는 real).

### [2-A] 토큰 발급 `get_kis_token()`
- `POST /oauth2/tokenP`, body `{grant_type:"client_credentials", appkey, appsecret}` → `access_token`(+만료).
- 토스 `get_toss_token`과 동일 골격: 캐시(만료 전 재사용), force 옵션, 실패 시 추정 없이 (None,사유)+log, timeout=10.
- **토큰은 발급 1분당 1회 제한**(KIS 규정) → 캐시 필수. self_check에서 매번 재발급 금지.
- 키 미설정(.env 비었음) 시 "KIS 키 미설정" 안전중단.

### [2-B] 공통 호출 헬퍼 `_kis_get(path, params, tr_id)`
- 헤더: `authorization: Bearer {token}`, `appkey`, `appsecret`, `tr_id`(거래ID — KIS는 API마다 tr_id 다름), `custtype:"P"`(개인).
- 토스 `_toss_get` 골격 재사용(토큰→헤더→GET→파싱→실패시 (None,사유)+log, timeout=10, 429/유량초과 EGW00201 처리).

## 3. [핵심] 야간선물 시세 조회 시도

> KIS `domestic_futureoption`에 코스피200 **야간선물(야간세션)**이 있는지 실호출로 확인이 목적.

### [3-A] 국내선물옵션 시세 엔드포인트 조사·호출
- KIS 공식 문서/GitHub의 `domestic_futureoption` 기준으로 **선물 현재가/시세 조회 API**(예: 선물옵션 시세 inquire-price 류)의 path·tr_id 확인.
- **코스피200 선물 종목코드**로 시세 조회 시도. (정규선물 먼저 되는지 → 그다음 야간세션 구분 가능한지)
- 야간선물 종목코드/구분이 별도인지, timestamp나 필드로 주간/야간이 구분되는지 확인.
- 야간세션 데이터가 잡히면 → `get_kospi_night_futures()`를 **KIS 직접수집**으로 구현(종가/등락/시각, 소스="KIS 야간선물 직접").
- **안 잡히면**(정규선물만, 야간 미제공/미지원): 정직하게 "KIS 야간선물 미제공" 결론 → 미국선물 프록시 메인 유지.
- 추정 금지. 실호출·문서 근거로만 판정. 무리하게 "있는 것처럼" 만들지 말 것.

### [3-B] assess_kospi_gap_signal 소스 우선순위 갱신
- 야간선물 소스 우선순위: **KIS 직접(되면) > 미국선물 프록시(폴백)**.
- 출력에 소스 명확히 표기(KIS 야간선물 직접 / 미국선물 프록시). 단정 금지·시간대 인지 유지.

## 4. 안전·보안 (재확인)
- 조회·시세만. **주문 코드 0줄.** KIS 주문 API는 건드리지도 말 것.
- 토큰·앱키·시크릿·계좌번호 전부 마스킹. 보고에 키 값 금지.
- timeout=10, 실패 시 추정 없이 (None,사유)+log(R7).
- **self_check 무겁게 금지**: KIS 토큰/시세 호출을 self_check에 넣지 말 것(토큰 1분 1회 제한+유량).
- 외부 라이브러리 추가 없이 requests로. (있으면 requirements 반영)

## 5. 문서 갱신 (CLAUDE.md = SSOT)
- KIS 도입 섹션: "KIS 실전 인증(get_kis_token/_kis_get) 구현, 조회 전용·주문0. 야간선물 시세 [제공/미제공] 실호출 결과. 소스 우선순위 KIS직접>미국프록시."
- 보안: KIS 키 마스킹 추가 기록.
- 0.2 갱신이력. 추정 금지.

## 6. 보고 (맨 끝 1회, 실측)
1. `py_compile`
2. **KIS 키 마스킹 자가검증**: 가짜 앱키/시크릿/토큰/계좌번호 투입 → 마스킹되는가 + 종목코드·가격 오탐0
3. **토큰 발급**: `get_kis_token()` 성공했는가(HTTP·만료시간). **키·토큰 값은 마스킹 표기만**. 실패 시 사유(키미설정/IP/유량 등).
4. **야간선물 조회**: 국내선물옵션 시세 호출 결과 — 정규선물 잡히는가, 야간세션 잡히는가(실호출). 잡히면 get_kospi_night_futures() 값(종가/등락/시각), 안 잡히면 그 사실.
5. `assess_kospi_gap_signal()` → 채택 소스(KIS직접/미국프록시)+신호+시간대.
6. self_check 표기·종합등 / 보안스캔 노출0(앱키·시크릿·토큰·계좌 평문0).

**형식**: 실측. 키·토큰·계좌 **마스킹 필수**(보고에 평문 절대 금지). 주문 "코드 없음·미접촉". 추정과 사실 구분.

## 7. 금지
- ❌ 주문 생성·정정·취소 코드(영구) ❌ 앱키·시크릿·토큰·계좌 평문 출력/로깅 ❌ KIS 키 발급·입력 ❌ 공식 kis_auth.py 통째 복사 ❌ self_check에 KIS 호출 추가 ❌ 야간선물 "있는 것처럼" 추정 ❌ 반복 통독

---
**완료 후**: 채팅 Claude 검수. (야간선물 제공되면 KIS 직접 메인 승격, 미제공이면 미국선물 프록시 메인 확정.)
