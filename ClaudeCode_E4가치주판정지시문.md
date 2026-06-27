# ClaudeCode 지시문 — E4: 미국 가치주 저평가 판정 (FMP+yfinance)

> 목적: 미국 종목의 밸류에이션을 여러 지표로 종합 판정(5단계 등급+근거). FMP 1차+yfinance 교차검증.
> 보안: FMP 키 마스킹. 조회 전용. 보고는 맨 끝 1회. 막히면 멈추고 한 줄 질문.

## 0. 🔒 보안
- FMP 키는 조회용이나 유출 시 호출한도 도용 위험 → 토스·KIS와 같은 보안: `.env`에만(`FMP_API_KEY`), 평문 금지, `.gitignore` 확인, `_redact`로 **`apikey=값` 마스킹**(FMP는 키를 URL 쿼리에 붙이므로 로그·에러에 URL이 찍히면 키 노출 → 반드시 가림). ClaudeCode 키 입력·출력 금지.

## 1. 작업 전 필독
- **폴더**: `C:\Users\dlwn7\OneDrive\Desktop\주식자동화` → 직행.
- **읽을 것**: yfinance 사용 구간(미국 시세), `_redact`, `log`, US_PORTFOLIO/감시종목, 미국 브리핑 구간. 반복 통독 금지.
- **확인된 사실(채팅 검증)**:
  - FMP 엔드포인트: `https://financialmodelingprep.com/api/v3/` 또는 stable. **핵심 = `ratios-ttm/{symbol}?apikey=`**(최근12개월 비율) 또는 `key-metrics-ttm`. 응답에 priceToEarningsRatioTTM(PER)·priceToBookRatioTTM(PBR)·priceToSalesRatioTTM(P/S)·priceToEarningsGrowthRatioTTM(PEG)·enterpriseValueMultipleTTM(EV/EBITDA)·debtToEquityRatioTTM·netProfitMarginTTM·freeCashFlowPerShareTTM 등.
  - **GOOGL 검증값**(채팅서 FMP·yfinance 대조 일치): PER~25.5, PBR~8.5, P/S~9.7, 순이익률~37.9%, D/E 낮음(건전). **단 PEG는 FMP 0.55 vs yfinance 1.36 불일치(계산방식 차이)** → PEG는 보조·주석.
  - 무료플랜 일250회 제한 가능성 → **결과 캐시**(같은 종목 짧은시간 반복호출 금지).

## 2. 구현: 가치주 판정 함수

### [2-A] `fmp_get_ratios(symbol)` — FMP 재무지표
- FMP `ratios-ttm`(또는 key-metrics-ttm) 호출, `?apikey={FMP_API_KEY}`. PER·PBR·P/S·PEG·EV/EBITDA·D/E·순이익률·FCF·ROE 파싱.
- **결과 캐시**(예: 메모리 dict 또는 디스크, TTL 수시간). 실패 시 추정 없이 (None,사유)+log. timeout=10.
- 키 미설정 시 "FMP 키 미설정" 안전중단(yfinance 폴백 가능하면 그쪽).

### [2-B] `yf_get_ratios(symbol)` — yfinance 교차검증/폴백
- yfinance `.info`에서 trailingPE·priceToBook·priceToSalesTrailing12Months·trailingPegRatio·debtToEquity·profitMargins 등 파싱. FMP 대조용+FMP 실패 시 폴백.

### [2-C] `assess_valuation(symbol)` — 종합 판정 (핵심)
- FMP 1차로 지표 수집, yfinance로 **교차검증**(PER·PBR·P/S가 두 소스 ±5~10% 이내면 신뢰, 벗어나면 ⚠️ 플래그하고 어느 소스가 튀는지 기록 — 추측으로 단정 금지).
- **5단계 등급**: 저평가 / 약간 저평가 / 적정 / 약간 고평가 / 고평가.
  - **단일 절대기준 금지**(PER 15 같은 건 기술주에 안 맞음). **업종/섹터 평균 대비**(FMP `sector-PE-snapshot`·`industry-PE-snapshot` 활용 가능) 또는 종목 과거평균 대비 상대평가 우선. 절대기준만 쓸 경우 "절대기준 한계" 주석.
  - 지표마다 ✓(좋음)/△(애매)/⚠️(주의·불일치) 표시.
- **가치함정 필터**: 싸 보여도 ①부채비율 과다 ②FCF 마이너스 ③순이익률 악화 ④매출/EPS 역성장이면 "저평가 아님(가치함정 의심)" 경고. 메모리 16번 4기둥(밸류에이션·펀더멘털·미래성장성·타이밍) 정신.
- **PEG는 보조**: 표시하되 "계산방식 따라 값 갈림(참고용)" 주석 필수.
- **충돌 경고**: 밸류에이션은 저평가인데 차트(E3)가 하락추세면 "차트 경고" 주석(메모리 16 철학).
- 출력은 **등급+지표별 근거** 구조(점수화 안 함 — 근거 투명성 위해). 단정 표현 자제, "~로 보임/참고" 수준.

### [2-D] 용어정리표 텍스트 `format_glossary()`
- 아래 표를 반환하는 함수(다음 단계 텔레그램 "용어정리표" 버튼에서 호출 예정):
  | 약어 | 뜻 | 한 줄 설명 | 방향 |
  - PER(주가수익비율): 주가가 순이익의 몇 배 / 낮을수록 쌈(너무 낮으면 의심)
  - PBR(주가순자산비율): 주가가 순자산의 몇 배 / 낮을수록 쌈
  - PSR·P/S(주가매출비율): 주가가 매출의 몇 배(적자기업도 평가) / 낮을수록 쌈
  - PEG(성장성감안PER): PER÷성장률 / 1 미만이면 저평가 신호(계산방식 갈림 주의)
  - EV/EBITDA(기업가치배수): 부채포함 기업값÷영업이익 / 낮을수록 쌈(가치함정 거름)
  - D/E(부채비율): 부채÷자본 / 낮을수록 안전
  - 순이익률: 순이익÷매출 / 높을수록 좋음
  - FCF(잉여현금흐름): 쓰고 남은 현금 / 플러스면 좋음
  - ROE(자기자본이익률): 내 돈으로 얼마 버나 / 높을수록 효율적
- 이 함수는 텍스트만 반환(고정). 매번 출력 아님 — 버튼/요청 시만.

## 3. 안전·범위
- 조회 전용·주문 없음. FMP 키 마스킹. timeout=10. 실패 시 (None,사유)+log(R7).
- **self_check 무겁게 금지**: FMP/판정 호출을 self_check에 넣지 말 것.
- 결과 캐시로 FMP 호출 절약(일250회).
- 외부 라이브러리 추가 없이 requests+기존 yfinance.

## 4. 문서 갱신 (CLAUDE.md = SSOT)
- E4: "fmp_get_ratios/yf_get_ratios/assess_valuation/format_glossary 구현. FMP 1차+yfinance 교차검증. 5단계 등급+지표별 ✓△⚠️ 근거+가치함정 필터+PEG 보조+충돌경고. 절대기준 아닌 업종/과거 대비 상대평가." 0.2 갱신이력. 추정 금지.

## 5. 보고 (맨 끝 1회, 실측)
1. `py_compile`
2. **FMP 키 마스킹 자가검증**: 가짜 apikey URL 투입 → 마스킹되는가 + 종목코드·숫자 오탐0
3. `fmp_get_ratios("GOOGL")` 실측(PER·PBR·P/S·PEG·D/E·순이익률) — 채팅검증값(PER~25.5 등)과 부합?
4. `yf_get_ratios("GOOGL")` + **FMP↔yfinance 대조**(PER·PBR·P/S 일치 여부, PEG 불일치 확인)
5. `assess_valuation("GOOGL")` → 5단계 등급+지표별 ✓△⚠️ 근거+가치함정 판정+PEG 주석 출력 샘플
6. `format_glossary()` 출력 샘플(용어표)
7. (선택) 보유종목 1~2개 더(NVDA 등) 등급 샘플
8. self_check / 보안스캔 노출0(apikey 평문0)

**형식**: 실측. 지표·등급 평문 가능. FMP 키 마스킹. 추정과 사실 구분. 등급은 "참고용" 톤.

## 6. 금지
- ❌ 단일 절대기준만으로 판정(업종/과거 대비 병행) ❌ PEG 단독 신뢰 ❌ 지표 불일치를 추측으로 단정 ❌ 점수화(등급+근거로) ❌ FMP 키 평문/로그 노출 ❌ self_check에 FMP ❌ 주문 ❌ 반복 통독

---
**완료 후**: 채팅 Claude 검수. 통과 시 다음=텔레그램 "용어정리표" 버튼 연결 + 가치주 등급을 미국 브리핑/불타기 우선순위에 반영.
