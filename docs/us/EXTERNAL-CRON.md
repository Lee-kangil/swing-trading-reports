# US Swing — 외부 cron → GHA workflow_dispatch

> **배경:** GitHub Free + **private** repo에서는 `on.schedule` cron이 동작하지 않을 수 있다.  
> `workflow_dispatch`는 정상이므로, [cron-job.org](https://cron-job.org) 등 외부 스케줄러가 REST API로 워크플로를 호출한다.

앱 슬롯 가드(`automation.trade`)는 ET(`America/New_York`) 기준이므로, cron 시각은 **UTC 고정** + workflow `slot` 입력을 명시하는 방식이 안전하다.

---

## 1. GitHub PAT 발급

1. GitHub → **Settings → Developer settings → Personal access tokens**
2. **Fine-grained token** (권장) 또는 Classic token
3. Repository access: `Lee-kangil/us-swing-auto-trading` only
4. Permissions:
   - **Actions: Read and write**
   - **Metadata: Read** (기본)
5. 토큰을 복사 — **cron-job.org에만** 저장, 레포에 커밋 금지

Classic token 사용 시: `repo` + `workflow` scope.

---

## 2. cron-job.org 설정

[https://console.cron-job.org](https://console.cron-job.org) 가입 후 **Create cronjob** × 2 (open / close).

### 공통

| 필드 | 값 |
|------|-----|
| Request method | `POST` |
| URL | `https://api.github.com/repos/Lee-kangil/us-swing-auto-trading/actions/workflows/us-alpaca-trade.yml/dispatches` |
| Headers | `Accept: application/vnd.github+json` |
| | `Authorization: Bearer <YOUR_GITHUB_PAT>` |
| | `X-GitHub-Api-Version: 2022-11-28` |
| Body (JSON) | 슬롯별 아래 참고 |
| Timezone | **UTC** |

### Job 1 — open+5 (EDT, 3월~11월)

| 필드 | 값 |
|------|-----|
| Title | `us-swing open+5 (EDT)` |
| Schedule | `35 13 * * 1-5` (13:35 UTC = 09:35 ET) |
| Body | `{"ref":"main","inputs":{"slot":"open+5","strategy":"composite"}}` |

### Job 2 — close-15 (EDT)

| 필드 | 값 |
|------|-----|
| Title | `us-swing close-15 (EDT)` |
| Schedule | `45 19 * * 1-5` (19:45 UTC = 15:45 ET) |
| Body | `{"ref":"main","inputs":{"slot":"close-15","strategy":"composite"}}` |

### EST 전환 (11월~3월)

서머타임 종료 후 UTC를 **+1시간** 이동. 상세: [`docs/reminders/2026-10-29-dst-cron.md`](reminders/2026-10-29-dst-cron.md)

| 슬롯 | EST cron (UTC) |
|------|----------------|
| open+5 | `35 14 * * 1-5` |
| close-15 | `45 20 * * 1-5` |

---

## 3. 로컬에서 수동 트리거 (테스트)

PowerShell (PAT는 환경변수로):

```powershell
$env:GITHUB_TOKEN = "ghp_..."   # 또는 fine-grained token
.\scripts\dispatch_trade_workflow.ps1 -Slot open+5
```

또는 `gh` CLI:

```powershell
gh workflow run "US Alpaca Paper Trade" `
  --repo Lee-kangil/us-swing-auto-trading `
  --ref main `
  -f slot=open+5 `
  -f strategy=composite
```

(`gh auth login` 필요 — cron-job.org에는 PAT + curl 방식 권장)

---

## 4. 성공 확인

```powershell
gh run list --repo Lee-kangil/us-swing-auto-trading --workflow="US Alpaca Paper Trade" --limit 3
gh run view <run-id> --repo Lee-kangil/us-swing-auto-trading --log | Select-String "\[trade\]"
```

기대 로그:

- `[trade] running slot=open+5 ...` — 정상
- `[trade] skip: market closed` — cron UTC/EST 불일치 → 스케줄 수정
- `event: workflow_dispatch` — 외부 cron 경유 ✅

---

## 5. EOD 레포트

`us-eod-report.yml`도 private Free에서 schedule이 막힐 수 있다. 동일 패턴으로:

- URL: `.../actions/workflows/us-eod-report.yml/dispatches`
- Body: `{"ref":"main","inputs":{"force":"false"}}`
- EDT cron: `5 22 * * 1-5` (~17:05 ET)

---

## 6. 보안

- PAT는 cron-job.org **Encrypted** 필드 / Secrets에만 저장
- PAT 유출 시 GitHub에서 즉시 revoke
- Live 전환 전에도 동일 PAT로 paper workflow만 호출하도록 workflow 가드 유지 (`ALPACA_PAPER=true` in GHA env)
