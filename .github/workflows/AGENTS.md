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
| Prep | Checkout | Checks out repo (binary + scripts) |
| 1 | Download & extract zones | Runs `./czds-nrd download` with CZDS creds |
| 2 | Clean & deduplicate | Runs `./czds-nrd clean` |
| 3 | Diff against recent releases | Crawls releases up to 500 MB baseline; diffs |
| 4 | Create GitHub Release | Creates `vYYYY-MM-DD` release with zip asset |
| 5 | Send to Telegram | Sends zip via Telegram bot |

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

## Work Guidance

- Never add a `go build` step — binary is pre-committed in `bin/`
- When editing, keep phases ordered 1–5 as defined in root AGENTS.md
- Guard step must always run first and skip all subsequent steps when release already exists
