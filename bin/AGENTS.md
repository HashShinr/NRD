# bin/ — AGENTS.md

## Purpose

Stores the prebuilt `czds-nrd` binary committed to the repo so CI runners can execute it without a Go toolchain.

## Ownership

- Built locally by the operator (HashShin) before committing
- Consumed by `.github/workflows/release.yml` at runtime

## Local Contracts

- **Only `bin/czds-nrd` is tracked** — the root `czds-nrd` is gitignored
- Binary must target `linux/amd64` with `CGO_ENABLED=0`
- Build command:
  ```bash
  CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o bin/czds-nrd ./...
  git add bin/czds-nrd && git commit -m "Rebuild binary"
  ```
- Never add a `go build` step to the CI workflow — the binary is always pre-committed

## Work Guidance

- Rebuild and recommit whenever any `.go` source file changes
- Do not add other files or scripts here; this directory is for the binary only

## Verification

- After rebuilding, confirm `file bin/czds-nrd` reports `ELF 64-bit LSB executable, x86-64`
- Run `go test ./...` from the repo root before rebuilding to catch regressions
