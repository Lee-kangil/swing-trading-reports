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
uv run python -m automation.generate_report --force   # 로컬
uv run python -m automation.report --html --force     # 동일
```

GitHub Pages: public 레포 `Lee-kangil/swing-trading-reports` → `docs/us/` (EOD workflow가 자동 push)

Secrets: `REPORTS_DEPLOY_KEY` (국내 eod-report와 동일 deploy key)

## 분석용 매매일지 (별도)

| 파일 | 용도 |
|------|------|
| `logs/journal/trades_journal.jsonl` | 체결 1건당 1행 (logic_id, notional 등) |
| `logs/journal/daily_summary.jsonl` | 일별 계좌·손익 요약 (수익 분석용) |

표시용 HTML과 분리되어 있어 pandas/Notebook으로 후속 분석 가능.

## 자동 업데이트

| 워크플로 | 시점 |
|----------|------|
| `us-eod-report.yml` | 장 마감 후 (~17:05 ET) HTML 생성 + `docs/` push |
| `us-alpaca-trade.yml` | close-15 슬롯 후에도 `--force`로 당일 갱신 |

> **Private Free repo:** GitHub `schedule` cron이 안 돌 수 있음 → [`docs/EXTERNAL-CRON.md`](EXTERNAL-CRON.md) (cron-job.org → `workflow_dispatch`).

Markdown 레거시: `logs/reports/report_YYYYMMDD.md` (`uv run python -m automation.report --live`)
