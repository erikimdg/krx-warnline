# krx-warnline

An End-of-Day CLI for Korea Exchange (KRX) market-alert stocks. Give it a stock name and it reports, for the **next trading day**:

- whether an **investment-warning** (투자경고) designation has already been lifted or is about to be
- if the judgment window is in progress, what closing price would trigger a **release** or a **trading halt**
- if the judgment window hasn't started yet, when the first judgment day is
- whether the stock has been escalated to **investment-risk** (투자위험)
- if the stock is currently a **warning-pre-designation** (투자주의), what closing price on the judgment day would push it into investment-warning

Output is a single multi-line block (in Korean, matching KRX terminology). Designed to be run after the close (e.g. after 9 PM), one stock name at a time, to preview the next day's scenarios. Uses only Naver Finance public APIs — **no external package dependencies**.

## Supported cases

- ✅ KOSPI / KOSDAQ common stocks
- ✅ **Preferred shares** (disclosures auto-fetched from the common-stock code; scenario computed on the preferred-share price series)
- ✅ **Trading-suspension days** auto-skipped (T-N indexing stays accurate on trading-day basis)
- ✅ **Investment-risk escalation** detection (analysis halts with a notice when escalated)
- ✅ **Re-designation / judgment-day deferral** (daily-rolling scenarios reflected automatically)
- ✅ **Trading-halt preview** gating (halt analysis only when a preview disclosure exists)
- ✅ **Pre-designation → designation thresholds** (from the 투자주의 stage: what judgment-day close lands the stock in investment-warning — per route: short-surge / short-rise·unsound / ultra-long-term surge)

## Requirements

- Python 3.8+ (standard library only: `urllib`, `json`, `re`, `html`, `datetime`)

## Install

```bash
git clone https://github.com/erikimdg/krx-warnline.git
cd krx-warnline
```

## Usage

### CLI

```bash
python3 check_warning.py "<stock_name>" [as_of_date YYYY-MM-DD]
```

Omit the date to use today. To backtest, pass a basis date as the second argument (e.g. `python3 check_warning.py "테스" 2026-06-10`). With a basis date, the tool only looks at disclosures issued up to that date and truncates price data to that day — and the judgment day T becomes the *next* trading day after it.

Stock name is passed verbatim — Korean/English/symbols/preferred-suffix all work. Examples:
- `"드림시큐리티"` (common)
- `"태영건설우"` (preferred)
- `"THE E&M"`, `"CSA 코스믹"` (with symbols)

### Import / backtest

```python
from check_warning import check_warning_stock
from datetime import date

print(check_warning_stock("드림시큐리티"))               # as of today
print(check_warning_stock("태영건설우", today=date(2026, 4, 29)))  # backtest
```

The `today` argument scopes both disclosures and price data to that point in time.

## Output examples

> Program output is in Korean (KRX terms). Captions below are English; the sample blocks show the actual output.

### 1) Judgment in progress — no halt risk

```
$ python3 check_warning.py "한국피아이엠"

종목명: 한국피아이엠
종목코드: 448900
실행일: 2026-05-22 (금)
다음 거래일: 2026-05-25 (월)

최신 공시: 한국피아이엠(주) 투자경고종목지정
지정일: 2026-05-11 (월)
최초 판단일: 2026-05-22 (금)
해제요건 임계: T-5 +45% / T-15 +75%
매매거래정지: 있음

=== 해제 시나리오 (판단일 진행 중) ===
T-5 종가: 107,000원  [2026-05-15]
cond1 (T-5 × 1.45): 155,150원
cond2 (T-15 × 1.75): 177,275원
cond3 (15일 최고가): 137,600원
해제 임계가 (cond3): 137,600원
다음 종가 해제 조건: < 137,600원 (+40.4% 미만 상승)
거래정지 위험: 없음 (매매거래정지 예고 공시 없음)
```

### 2) Pre-designation → designation thresholds (투자주의 stage)

When the latest relevant disclosure is `투자경고종목 지정예고`, the tool computes — per route — what judgment-day (T) close would trigger the investment-warning designation.

```
$ python3 check_warning.py "테스" 2026-06-10

종목명: 테스
지정예고일: 2026-06-11 (목)
최초 판단일: 2026-06-11 (목)
순연 마감(판단 종료): 2026-06-24 (수)

시장: KOSDAQ  |  판단일(T) = 2026-06-11 (목) 기준 (다음 거래일)
T-1 2026-06-10 (수) 종가 173,400원  |  T-5 2026-06-04 (목) 종가 135,900원
② 최근 15일 최고종가(T 제외): 173,400원 → T 종가가 이보다 높아야 충족

── 경로 [1] 단기급등 (T-5 +60%) ──
  ① T-5 135,900 × 1.60 = 217,440원  |  T-1 대비 +25.40%
  ③ 주가상승률 ≥ KOSDAQ 5배: 지수 5일 -9.34% → ✅충족 (T-1 근사)
  ▶ 가격 트리거(①): 217,440원 이상 | 상한가 225,000원 → 도달가능

── 경로 [2] 단기상승·불건전 (T-5 +45%) ──
  ① T-5 135,900 × 1.45 = 197,055원  |  T-1 대비 +13.64%
  ③ 매수관여율(특정계좌) — ⚠ 외부확인불가 (KRX 내부 감시데이터)
  ▶ 가격 트리거(①): 197,055원 이상 | 상한가 225,000원 → 도달가능

=== 안내 문구 (복붙용) ===
테스 : 금일 종가 기준 197,055원 (+13.64%) 마감 시 6/12부터 투경 가능성 있음 (소수 계좌 여부에 달림), 217,440원 (+25.40%) 이상 상승 마감시에는 소수 계좌에 관계없이 투경 확정
```

Routes and thresholds:

- **Short-surge (60%) / short-rise (45%)**: threshold = `T-5 close × (1 + pct)`; condition ② requires the close to exceed the highest close of the last 15 trading days.
- **Ultra-long-term surge (200% over 1 year)**: threshold = `1yr-ago close × (1 + 2.00 + index_return)`. The "composite index" is the stock's own market index (KOSPI for KOSPI names, KOSDAQ for KOSDAQ names) — confirmed by cross-checking against KRX pre-designation reasons.
- ⚠ **Condition ③ (account participation / specific accounts)** is non-public KRX surveillance data and cannot be computed externally — it is flagged `외부확인불가`. (The short-surge ③ is the "index ×5" rule, which *is* computable.)
- ⚠ **Condition ④ (market-cap rank outside top 100)** shows market cap only instead of an exact combined ranking (usually satisfied for alert targets); precise rank needs KRX.
- The `=== 안내 문구 (복붙용) ===` block is a one-line, copy-paste-ready summary. "확정" (confirmed) when the binding route's ③ is computable; "가능성 있음 (소수 계좌 여부에 달림)" (possible, depends on a few accounts) when ③ is the non-public account rule.

### 3) Already lifted

```
$ python3 check_warning.py "이오테크닉스"

최신 공시: (주)이오테크닉스 [투자주의]투자경고종목 지정해제 및 재지정 예고
해제일: 2026-05-18 (월)
상태: 이미 해제 상태 (2026-05-18 (월) 해제)
주의: 예고 공시 — 이후 재지정 가능성 확인 필요
```

### 4) Escalated to investment-risk

```
$ python3 check_warning.py "가온전선"

⚠ 투자위험종목 격상 감지: 2026-05-08 가온전선(주) 투자위험종목 지정
→ 투자경고 분석은 무효. 투자위험종목 별도 처리 필요.
```

### 5) Not a market-alert stock

```
$ python3 check_warning.py "삼성전자"

상태: 투자경고 관련 공시 없음
```

## How it works

### Data sources (all Naver Finance public endpoints)

| Purpose | URL |
|---------|-----|
| name → code | `ac.stock.naver.com/ac?q={name}&target=stock` |
| disclosure list | `m.stock.naver.com/api/stock/{code}/disclosure?page=1&size=20` |
| disclosure detail (HTML) | `m.stock.naver.com/api/stock/{code}/disclosure/{id}` |
| daily OHLCV | `fchart.stock.naver.com/siseJson.naver?symbol={code}&...` |
| market cap / market type | `m.stock.naver.com/api/stock/{code}/integration`, autocomplete `typeCode` |

If the local TLS chain can't be verified (e.g. behind a corporate proxy), `_http_get` retries once with an unverified context — these are public read-only endpoints.

### Flow

```
name → (autocomplete) → code
  → preferred? (name ends with 우/우B/우C) → disclosure_code / price_code split
  → disclosure list → filter by trailing "(XXX우)" suffix
  → escalation to investment-risk? → stop + notice
  → newest relevant disclosure:
      ├─ release          → parse release date → status
      ├─ designation      → parse designation/judgment date, pct1/pct2, no_halt
      │     ├─ next trading day < first judgment day → "not yet"
      │     └─ else → T-1/T-2/T-5/T-15 closes + 15-day high → cond1/2/3 → release threshold
      ├─ pre-designation  → parse routes → per-route designation thresholds for judgment day T
      └─ none → "no market-alert disclosure"
```

### Release conditions (lifts next day if any one fails)

For judgment day T:
- **cond1**: T close < T-5 close × (1 + pct1)
- **cond2**: T close < T-15 close × (1 + pct2)
- **cond3**: T close < highest close of the last 15 trading days (T-1…T-15)

`release threshold = min(cond1, cond2, cond3)`. `pct1`/`pct2` are stock-specific (`45%/75%` or `60%/100%`), parsed from the disclosure body via regex.

### Pre-designation (designation) conditions

The judgment day T is the next trading day; T-1 is the latest available close. For each route parsed from the `[1]`/`[2]` blocks of the notice:
- price threshold from ① (short: `T-5 × (1+pct)`; ultra-long: `1yr × (1 + 2.00 + index_return)`)
- ② the close must exceed the highest close of the last 15 trading days
- ③ index-×5 rule (computable) or account-participation rule (non-public → flagged)
- ④ market-cap rank outside the combined top 100 (market cap shown; exact rank needs KRX)

The binding price trigger is `max(① threshold, ② recent-15-day high)`; "<" vs ">" wording reflects which one binds.

### Trading-halt gating

A 1-day halt the next session requires all three: (1) +40% vs T-2, (2) T close > pre-designation-day close, (3) T close > T-1 close. KRX issues a **`매매거래정지 예고`** disclosure on T-1 when this is likely, so the tool only shows a halt trigger (`T-2 × 1.4`, with reachability vs the daily upper limit) when that preview exists. The conservative effective release threshold is `min(release threshold, halt trigger)`.

### Preferred shares

Preferred-share disclosures are filed under the common-stock code. So: detect preferred by name suffix; fetch disclosures from the common code (last digit → 0); keep only disclosures whose title ends with the matching `(XXX우)`; use the preferred code for prices.

### Trading-suspension & trading-day handling

fchart returns rows for suspension days too, with open/high/low/volume all 0 and close carried over — rows with `open==0` or `volume==0` are skipped so T-N indexing stays on a trading-day basis. "Next trading day" is +1 calendar day skipping weekends (public holidays not modeled).

### Disclosure classification

| Title pattern | Class |
|---------------|-------|
| `투자경고종목` + `지정해제` | release |
| `투자경고종목` + `지정예고` | pre-designation |
| title ends with `투자경고종목 지정[(…)]` (excl. preview/release) | designation |
| `매매거래정지` + `예고` | halt preview |
| `투자위험종목` + `지정` (excl. preview/release) | escalation |
| otherwise | ignored |

## Caching

Name → code mappings are cached in `stock_codes.json` next to the script (git-ignored).

## Disclaimer

- Depends on Naver Finance public APIs; may break if those change.
- Disclosure bodies are parsed by regex against the standard KRX template; format changes can break parsing.
- Data freshness: right after the close, fchart may not yet have the day's close — run **after ~9 PM**.
- Conditions ③ (account participation) and ④ (exact market-cap rank) for designation are non-public/approximate and are flagged as such.

## License

MIT
