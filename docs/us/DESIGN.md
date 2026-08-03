# Alpaca US Swing Auto-Trading — Design Spec

작성일: 2026-08-02  
상태: Phase 0 확정 (골격 생성)  
브로커: Alpaca (Paper → Live, 동일 API 스택)

---

## 1. 목표

- 미국 주식 스윙 자동매매를 **별도 Git 프로젝트**로 운영
- **Paper로 검증**한 코드·가드·ledger를 **Live에 그대로 이식**
- 국내 `swing-auto-trading` (KIS/키움)과 **완전 분리**

## 2. 아키텍처

```text
┌─────────────────────────────────────┐
│  us-swing-auto-trading              │
│  ┌─────────────┐  ┌──────────────┐  │
│  │ strategies  │→ │ daily_trade  │  │
│  └─────────────┘  └──────┬───────┘  │
│                          │          │
│  ┌─────────────┐  ┌──────▼───────┐  │
│  │ guards      │→ │ AlpacaClient │  │
│  └─────────────┘  └──────┬───────┘  │
└──────────────────────────┼──────────┘
                           │
              Paper API ───┴─── Live API (later)
              (paper=True)      (paper=False + LIVE_ENABLED)
```

### 2.1 브로커 추상화

- **지금**: `core/broker/alpaca_client.py` — `TradingClient` factory
- **나중**: `BrokerClient` protocol — Alpaca Live / (선택) IBKR 교체 시 어댑터만 교체

### 2.2 Paper / Live 분리 (fail-closed)

| env | Paper | Live |
|-----|-------|------|
| `ALPACA_PAPER` | `true` (default) | `false` |
| `ALPACA_LIVE_ENABLED` | `false` (default) | **`true` 필수** |
| API keys | Paper dashboard keys | Live dashboard keys |

`ALPACA_PAPER=false` 이고 `ALPACA_LIVE_ENABLED` 가 없으면 **모든 주문 경로 거부**.

## 3. 국내 레포와의 관계

| | swing-auto-trading | us-swing-auto-trading |
|---|-------------------|----------------------|
| 브로커 | KIS paper + Kiwoom real | Alpaca paper → live |
| 통화 | KRW | USD |
| 스케줄 | KST 09:05·15:15 | US ET (DST) |
| Secrets | `KIS_PAPER_*`, `KIWOOM_*` | `ALPACA_*` |
| 로그 | `automation/logs/` | `logs/` (본 프로젝트) |

**코드 복사 금지**, **패턴만 이식**: guards, cooldown, ledger, probe-first.

## 4. 현금·자본 (ledger)

국내 교훈: API `portfolio_value` / `equity` 를 주문 한도로 쓰지 않는다.

| 필드 | 용도 |
|------|------|
| `buying_power` | 매수 가능 상한 (주문 전 검증) |
| `cash` | 현금 잔고 |
| `US_INITIAL_CAPITAL` | 리포트 ROI 기준선 (프로브 후 설정) |
| `CASH_RESERVE` | 총자산 대비 현금 버퍼 (default 5%) |

Phase 3에서 `logs/trades_*.jsonl` 기반 ledger 추가 (국내 `daily_trade.py` 패턴).

## 5. 스케줄 (Phase 3)

미국 정규장: ET 09:30–16:00 (서머타임 적용).

초안 슬롯 (국내 2회/일 패턴 이식, ET 기준):

- Slot 1: 장 개시 + 35분 (10:05 ET)
- Slot 2: 장 마감 − 45분 (15:15 ET)

휴장: Alpaca `get_clock()` + US holiday calendar.

## 6. 구현 로드맵

### Phase 0 — 골격 ✅

- [x] `pyproject.toml`, `core/`, `automation/probe.py`
- [x] Paper/Live guards
- [x] `.env.example`, README, 본 DESIGN

### Phase 1 — Paper 프로브 (사용자 액션 필요)

- [ ] Alpaca Paper 가입 + API keys
- [ ] `uv run python -m automation.probe` — account / clock 확인
- [ ] `buying_power`, `cash`, `portfolio_value` 기록 → `US_INITIAL_CAPITAL` 설정
- [ ] (선택) 실제 체결 테스트: limit order at market-near price, verify fill + position

### Phase 2 — 전략·백테스트

- [ ] US 유니버스 (예: S&P500 / liquid large-cap)
- [ ] Historical bars (`StockHistoricalDataClient` or Polygon)
- [ ] 국내 momentum/ADX 로직 이식 여부 결정
- [ ] Walk-forward / PIT 유니버스 (국내 pit-universe spec 참고)

### Phase 3 — Paper 자동매매

- [ ] `automation/daily_trade.py` — signal → order → log
- [ ] `logs/trades_*.jsonl`, `logs/holdings_*.json`
- [ ] Exit cooldown, cash reserve, slot guard
- [ ] GitHub Actions (US market days, self-hosted or cloud)

### Phase 4 — Live

- [ ] Alpaca Live KYC + USD 입금
- [ ] Live API keys → GitHub Secrets (별도 이름: `ALPACA_LIVE_*`)
- [ ] `ALPACA_LIVE_ENABLED=true` — **수동 확인 후** 첫 실행
- [ ] Paper vs Live PnL 분리 리포트

## 7. Live 전환 체크리스트 (Phase 4)

- [ ] Paper N≥60 trading days 운영 로그
- [ ] Max drawdown / churn / slippage 기준 통과
- [ ] Guards unit test green
- [ ] Live keys in Secrets, Paper keys retained for regression
- [ ] First live run: 1 symbol, min qty, manual approval

## 8. 리스크

| 리스크 | 대응 |
|--------|------|
| Paper ≠ Live (슬리피지, 체결) | Live 초기 소액·소유니버스 |
| PDT (Pattern Day Trader) | `$25k` 미만 계좌 day-trade 제한 — 스윙은 보통 영향 적으나 확인 |
| API 장애 | fail-closed, 알림 |
| Live 키 오류 | `ALPACA_LIVE_ENABLED` + separate secrets |

## 9. 참고

- [Alpaca Paper Trading](https://alpaca.markets/learn/start-paper-trading)
- [alpaca-py SDK](https://alpaca.markets/sdks/python/getting_started.html)
- 국내 HANDOFF: `../swing-auto-trading/docs/superpowers/HANDOFF-2026-08-02-full.md`
