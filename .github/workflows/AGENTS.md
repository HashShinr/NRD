# .github/workflows/ — AGENTS.md

## Purpose

CI/CD workflows for the NRD pipeline. The sole workflow (`release.yml`) downloads CZDS zone files daily, produces a Newly Registered Domains diff, and publishes it as a GitHub Release.

## Ownership

- Maintained by operator (HashShin)
- Consumes prebuilt binary from `bin/czds-nrd`

## Workflow: `release.yml` — "CZDS NRD Release"

### Triggers
| Trigger | Purpose |
|---------|---------|
| `push: tags: v*` | Manual tagged release |
| `workflow_dispatch` | Manual run from Actions UI |
| `schedule: 0 7 * * *` | Daily primary run (07:00 UTC) |
| `schedule: 0 9 * * *` | Daily backup run (09:00 UTC) |

### Phases
| Phase | Step | What it does |
|-------|------|-------------|
| Guard | Check if today's release already exists | Skips if `vYYYY-MM-DD` already published |
| Prep | Free disk space | Clears `/opt/hostedtoolcache`, Android SDK, .NET |
| Prep | Checkout | Checks out repo (binary + scripts); runs on both paths |
| Prep | Prepare binary | Copies `bin/czds-nrd` into the workspace; runs on both paths |
| 1 | Download & extract zones | Runs `./czds-nrd download` with CZDS creds |
| 2 | Clean & deduplicate | Runs `./czds-nrd clean` |
| 3 | Diff against recent releases | Crawls releases up to 500 MB baseline; diffs |
| 4 | Create GitHub Release | Creates `vYYYY-MM-DD` release with zip asset |
| 5 | Send to Telegram | Sends zip via Telegram bot, then writes a `Telegram-delivered:` receipt line into the release notes |
| 5b | Retry Telegram delivery | Runs only on the skip path; re-delivers the release asset when the notes carry no delivery receipt |

### Secrets used
| Secret | Used in |
|--------|---------|
| `CZDS_USERNAME` | Phase 1 |
| `CZDS_PASSWORD` | Phase 1 |
| `GITHUB_TOKEN` | Phase 3, Phase 4 (auto-provided) |
| `TELEGRAM_API_BASE` | Phase 5 |
| `TELEGRAM_BOT_TOKEN` | Phase 5 |
| `TELEGRAM_CHAT_ID` | Phase 5 |

### Permissions
- `contents: write` — needed for creating releases and pushing tags

### Delivery receipts

The guard only knows whether a release *exists*, not whether it was *delivered*.
Phase 5 therefore stamps `Telegram-delivered: <timestamp>` into the release
notes after a successful send, and Phase 5b runs on the skip path to re-deliver
any release that lacks that stamp. Without this, a run that failed between
release creation and delivery (e.g. a transient network error) would never be
retried: the primary run goes red and the backup run silently skips.

Re-delivery is intentionally idempotent — once the receipt is present, later
skipped runs log `nothing to do` and exit without sending.

## Work Guidance

- Never add a `go build` step — binary is pre-committed in `bin/`
- When editing, keep phases ordered 1–5 as defined in root AGENTS.md
- Guard step must always run first. It skips Phases 1–5 when the release already
  exists, but Checkout, Prepare binary and Phase 5b run on both paths so a
  missing Telegram delivery can still be repaired
