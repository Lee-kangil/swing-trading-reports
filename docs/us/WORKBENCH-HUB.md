# Workbench Hub — US Swing (`us-swing-auto-trading`)

> **통합 SSOT:** `C:\Users\KangilLee\Projects\ai-workbench\projects\auto-trading\STATUS.md`  
> 이 파일은 **US 레포 관점** 요약. 허브와 불일치 시 **허브 STATUS 우선**.

---

## 30초 요약 (2026-09-06)

| 항목 | 값 |
|------|-----|
| 모드 | Alpaca **Paper** |
| 운영 | **3로직 split** — `macd_trend_confirm` / `momentum_absolute` / `short_term_reversal` |
| 슬롯 | `open+5`, `close-15` (ET) |
| 한도 | 로직당 **30%**, 종목당 **6%**, 로직당 **5종**, 전체 **90%** |
| cron | `strategy: **split**` (composite 폐기) |
| 레포트 | https://lee-kangil.github.io/swing-trading-reports/us/latest_day.html |

---

## 최근 완료 (09-06)

- [x] **로직 교체**: `ma_divergence`(live 0건, 08-14부터 관찰) → `macd_trend_confirm`
      — 2022(하락장) ROI +36.01%/승률 65.60%, 최근 1년(상승장) ROI +83.13%/승률 74.42%
        GHA 실백테스트로 검증 후 `SPLIT_LOGIC_IDS` 교체 + `buy_signal_strength` 강도 함수 추가
- [x] `composite_label()` fallback 문자열을 `SPLIT_LOGIC_IDS` 참조로 변경 (하드코딩 라벨 stale 방지)
- [x] `docs/REPORT-FORMAT.md` 표시 로직명 갱신

## 최근 완료 (08-14)

- [x] 레포트: 계좌·로직별 **매수원가 / 평가금액 / 투자 비중%**
- [x] legacy `split` · `composite` · **`correction`** 섹션 숨김
- [x] `split_correction.json` **applied** — 1회성 정리 종료
- [x] **30% 한도**: trade replay + Alpaca 실포지션 기준 `logic_headroom` (장부 stale 방지)
- [x] `logic_seed.json`: BAC·GE → momentum (orphan 매핑)
- [x] `EXTERNAL-CRON.md` → split 반영

---

## 열린 이슈

| # | 항목 | 조치 |
|---|------|------|
| 1 | **기존 초과 보유** (momentum ~37%, STR ~40%) | 신규 매수는 차단됨 → 시그널 매도 또는 수동 trim 관찰 |
| 2 | **macd_trend_confirm** 교체 직후 (09-06) | live 0건 재발 여부 2~4주 관찰 |
| 3 | **Live** | 미개설 — Paper N일 축적 후 |
| 4 | DST cron | `docs/reminders/2026-10-29-dst-cron.md` (10/29 전) |

---

## 핵심 파일

| 파일 | 역할 |
|------|------|
| `lumibot_strategies/pit_swing.py` | split 매매 (매도→매수, 한도) |
| `core/trading/logic_holdings.py` | 종목→로직, live deploy 합산 |
| `core/trading/position_book.json` | 가상 장부 (GHA cache) |
| `data/trading/logic_seed.json` | log 없을 때 로직 매핑 보조 |
| `data/trading/split_correction.json` | 1회성 정리 (applied) |
| `core/reporting/build.py` | 일일 레포트 데이터 |
| `.github/workflows/us-alpaca-trade.yml` | Paper trade |
| `.github/workflows/us-eod-report.yml` | HTML → Pages |

---

## 명령

```powershell
uv run python -m automation.probe
uv run python -m automation.trade --slot open+5 --strategy split --force
uv run python -m automation.generate_report --force
uv run pytest
```

외부 cron: [`docs/EXTERNAL-CRON.md`](EXTERNAL-CRON.md)  
레포트 형식: [`docs/REPORT-FORMAT.md`](REPORT-FORMAT.md)

---

## 관련 레포

| 레포 | 역할 |
|------|------|
| `swing-auto-trading` | KR (KIS/키움) — **분리** |
| `swing-trading-reports` | Pages (KIS / US HTML) |
| `ai-workbench/.../auto-trading` | 통합 STATUS · TODO |

---

## 용어 (레포트)

| 표시 | 의미 |
|------|------|
| **보유 매수원가** | 평단×수량 합 |
| **보유 평가금액** | 현재가×수량 합 |
| **투자 비중** | 매수원가 ÷ 총자산 (한도 30%) |
| **correction** | 운영 1회 정리 (레포트 비표시) |
