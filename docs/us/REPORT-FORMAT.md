# US Swing — 일일 결과 레포트

## HTML (GitHub Pages)

국내 [KIS 매매일지](https://lee-kangil.github.io/swing-trading-reports/latest_day.html)와 같은 public Pages 레포에 배포:

| URL | 설명 |
|-----|------|
| https://lee-kangil.github.io/swing-trading-reports/us/latest_day.html | **북마크용** — 항상 최근 거래일 |
| `swing-trading-reports/docs/us/reports/YYYY-MM-DD.html` | 일자별 아카이브 |
| https://lee-kangil.github.io/swing-trading-reports/ | 전체 허브 (KIS / 키움 / US) |

> `us-swing-auto-trading`은 private repo라 Pages 불가 → public `swing-trading-reports`의 `docs/us/`에 게시

생성:

```powershell
uv run python -m automation.generate_report --force
uv run python -m automation.report --html --force
```

Secrets: `REPORTS_DEPLOY_KEY` (국내 eod-report와 동일 deploy key)

---

## 계좌 요약 타일

| 타일 | 의미 |
|------|------|
| **총자산** | Alpaca equity (부제: 현금 · 주식 평가) |
| **보유 매수원가** | 전체 보유 종목 `평단×수량` 합 |
| **보유 평가금액** | 전체 보유 `현재가×수량` 합 (부제: 평가손익) |
| **누적손익** | 초기자본 대비 |
| **당일 실현손익** | 당일 매도 FIFO 실현 |
| **보유종목 평가손익** | 미실현 합 |

---

## 로직별 섹션 (split 3로직)

표시: `ma_divergence`, `momentum_absolute`, `short_term_reversal`

**숨김** (legacy/운영): `split`, `composite`, `correction`

| 타일/항목 | 의미 |
|-----------|------|
| **매수 금액** | 해당 로직 보유 `평단×수량` 합 |
| **투자 비중** | 매수금액 ÷ 총자산 (**한도 30%**) |
| **보유 평가금액** | 현재가×수량 합 (부제: 평가손익) |
| **실현/미실현/합계** | 로직별 손익 |
| **보유 종목** | trade replay 기준 로직 귀속 |
| **매매 내역** | 해당 logic_id 체결 |

종목→로직: trade log replay → `position_book` → `logic_seed.json`

---

## 분석용 매매일지 (별도)

| 파일 | 용도 |
|------|------|
| `logs/journal/trades_journal.jsonl` | 체결 1건당 1행 |
| `logs/journal/daily_summary.jsonl` | 일별 계좌·손익 |

---

## 자동 업데이트

| 워크플로 | 시점 |
|----------|------|
| `us-eod-report.yml` | 장 마감 후 (~17:05 ET) HTML + Pages |
| `us-alpaca-trade.yml` | open+5 / close-15 후 레포트 갱신 |

> Private Free repo: GitHub `schedule` 미동작 가능 → [`docs/EXTERNAL-CRON.md`](EXTERNAL-CRON.md) (`strategy: split`)

Markdown 레거시: `uv run python -m automation.report --live`

---

## 운영 한도 (Paper split)

| env | 기본 |
|-----|------|
| `PER_LOGIC_DEPLOY_PCT` | 0.30 |
| `PER_SYMBOL_SPLIT_PCT` | 0.06 |
| `MAX_POSITIONS_PER_LOGIC` | 5 |
| `MAX_DEPLOY_PCT` | 0.90 |
| `CASH_RESERVE` | 0.10 |

매수 직전 `logic_headroom`은 **trade replay + Alpaca 실포지션** 중 큰 값으로 deploy를 계산한다.
