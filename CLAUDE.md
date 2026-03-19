# CLAUDE.md — OpenClaw AI Assistant Guide

> This file is the primary reference for AI coding assistants (Claude Code, etc.) working in this repository. See also `AGENTS.md` for detailed operational guidelines and `CONTRIBUTING.md` for contributor rules.

---

## Project Overview

**OpenClaw** is a personal, single-user AI assistant platform written in TypeScript (Node.js ≥22.12.0). It acts as a multi-channel messaging gateway and AI agent orchestration system, connecting to 20+ messaging platforms (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, Matrix, Teams, Zalo, LINE, etc.) and providing voice, Canvas UI, and multi-agent capabilities.

- **Version scheme:** Date-based (e.g. `2026.2.6-3`)
- **Package manager:** pnpm 10.23.0 (enforced via corepack); Bun also supported for dev/scripts
- **License:** MIT
- **Naming:** Use **OpenClaw** in headings/docs; use `openclaw` for CLI commands, binary, paths, config keys

---

## Repository Structure

```
openclaw/
├── src/                   # Core TypeScript source (~325k lines)
│   ├── agents/            # AI agent orchestration (Pi framework)
│   ├── acp/               # Agent Client Protocol implementation
│   ├── channels/          # Channel abstraction, plugins, allowlists
│   ├── cli/               # CLI entry point (program.ts, progress.ts)
│   ├── commands/          # CLI command implementations
│   ├── config/            # Configuration & session management
│   ├── discord/           # Discord integration
│   ├── gateway/           # Gateway server + protocol + RPC methods
│   ├── hooks/             # Bundled hooks system
│   ├── imessage/          # iMessage integration
│   ├── infra/             # Network, TLS, env, errors, binaries
│   ├── line/              # LINE messaging
│   ├── logging/           # Logging infrastructure
│   ├── macos/             # macOS-specific code
│   ├── media/             # Media handling pipeline
│   ├── media-understanding/  # Media analysis providers
│   ├── memory/            # Memory system (embeddings, vector search)
│   ├── plugin-sdk/        # Plugin SDK exports
│   ├── plugins/           # Plugin runtime management
│   ├── process/           # Process execution, RPC, child-process-bridge
│   ├── routing/           # Message routing
│   ├── sessions/          # Session management
│   ├── signal/            # Signal integration
│   ├── slack/             # Slack integration
│   ├── telegram/          # Telegram integration
│   ├── terminal/          # Terminal utilities (table.ts, palette.ts)
│   ├── tts/               # Text-to-speech
│   ├── tui/               # Terminal UI
│   ├── types/             # Shared type definitions
│   ├── utils/             # General utilities
│   ├── web/               # Web channel integration
│   ├── whatsapp/          # WhatsApp integration
│   └── wizard/            # Onboarding wizard
├── apps/                  # Platform apps (android, ios, macos, shared)
├── packages/              # Workspace packages (clawdbot, moltbot)
├── extensions/            # 21 channel/plugin extensions (workspace pkgs)
├── skills/                # Built-in skills (spotify, discord, obsidian, etc.)
├── ui/                    # Web UI (separate pnpm workspace)
├── docs/                  # Mintlify documentation site
├── scripts/               # Build, deploy, and utility scripts
├── test/                  # E2E fixtures, mocks, helpers, setup
└── vendor/                # Vendored dependencies
```

### Key Files

| File | Purpose |
|------|---------|
| `src/cli/program.ts` | CLI command tree entry point |
| `src/cli/progress.ts` | Progress/spinner utilities (use this, never hand-roll) |
| `src/terminal/palette.ts` | Shared color palette (no hardcoded ANSI colors) |
| `src/terminal/table.ts` | ANSI-safe table output |
| `src/gateway/protocol/` | Gateway protocol schema (auto-generates Swift types) |
| `openclaw.mjs` | Production entry point |
| `AGENTS.md` | Detailed operational guidelines (read this too) |

---

## Development Setup

```bash
# Install dependencies
pnpm install

# Install pre-commit hooks (runs same checks as CI)
prek install

# Run CLI in dev mode
pnpm dev
# or
pnpm openclaw ...

# Start gateway in dev with hot-reload
pnpm gateway:dev
pnpm gateway:watch
```

**Node ≥22.12.0** is required. Bun is preferred for TypeScript execution (scripts, dev, tests).

---

## Build Commands

```bash
pnpm build          # Full build (tsdown + plugin SDK + DTS + a2ui bundle)
pnpm tsgo           # TypeScript type-checking only
pnpm check          # Lint + format check (run before commits)
pnpm lint           # oxlint only
pnpm lint:fix       # Auto-fix lint issues
pnpm format         # oxfmt check
pnpm format:fix     # Auto-fix formatting
```

---

## Testing

**Framework:** Vitest 4.0.18 with V8 coverage

```bash
pnpm test                       # Unit tests
pnpm test:watch                 # Watch mode
pnpm test:coverage              # Coverage report (threshold: 70% lines/functions)
pnpm test:e2e                   # E2E tests
pnpm test:all                   # Full CI suite

# Live tests (require real API credentials)
CLAWDBOT_LIVE_TEST=1 pnpm test:live
LIVE=1 pnpm test:live           # Includes provider live tests

# Docker-based tests
pnpm test:docker:live-models
pnpm test:docker:live-gateway
pnpm test:docker:onboard
```

**Test file conventions:**
- Colocated with source: `foo.ts` → `foo.test.ts`
- E2E tests: `*.e2e.test.ts`
- Do not set test workers above 16
- Full test kit docs: `docs/testing.md`

---

## Linting & Formatting

| Tool | Purpose | Config |
|------|---------|--------|
| oxlint | JS/TS linting (Rust-based) | `.oxlintrc.json` |
| oxfmt | JS/TS formatting | `.oxfmtrc.jsonc` |
| shellcheck | Shell script linting | `.shellcheckrc` |
| markdownlint-cli2 | Markdown linting | `.markdownlint-cli2.jsonc` |
| swiftlint / swiftformat | Swift (macOS/iOS) | `.swiftlint.yml`, `.swiftformat` |
| actionlint | GitHub Actions | — |
| zizmor | GH Actions security | `zizmor.yml` |
| detect-secrets | Secret scanning | `.detect-secrets.cfg` |

**Linting scope:** oxlint/oxfmt cover `src/` only. `extensions/`, `skills/`, `apps/`, `dist/`, `vendor/` are excluded.

---

## Coding Conventions

- **Language:** TypeScript ESM; prefer strict typing; avoid `any`
- **File size:** Aim for ≤700 LOC; ≤500 LOC preferred. Split/refactor for clarity
- **Comments:** Add brief comments for tricky/non-obvious logic only
- **Helpers:** Extract helpers instead of duplicating; avoid "V2" copies
- **CLI options:** Follow patterns in existing commands; use `createDefaultDeps` for DI
- **Progress/spinners:** Always use `src/cli/progress.ts` (`osc-progress` + `@clack/prompts`)
- **Colors/ANSI:** Use `src/terminal/palette.ts`; never hardcode color codes
- **Tables:** Use `src/terminal/table.ts` for status output
- **SwiftUI:** Prefer `Observation` framework (`@Observable`, `@Bindable`) over `ObservableObject`

### Plugin / Extension Rules

- Plugin-only deps go in the extension's `package.json`, not root
- Plugin `dependencies` must be real npm packages (no `workspace:*` — breaks `npm install`)
- Put `openclaw` in `peerDependencies` or `devDependencies` in plugins (runtime resolves via jiti alias)
- When adding channels/extensions: update `.github/labeler.yml`, all UI surfaces, onboarding, and docs

### Dependency Rules

- Never edit `node_modules`
- Never update the Carbon dependency
- Patched deps (`pnpm.patchedDependencies`) must use exact versions (no `^`/`~`)
- Patching deps requires explicit approval — do not patch by default
- Any dep with a patch entry must keep Bun patch in sync when touching deps

---

## Commit & PR Workflow

```bash
# Commit (scoped staging)
scripts/committer "<msg>" <file...>

# Full gate before pushing
pnpm build && pnpm check && pnpm test
```

**Commit message style:** Concise, action-oriented, module-prefixed
Example: `CLI: add verbose flag to send`

**Changelog:** Keep latest released version at top (no `Unreleased` section). Include PR # and contributor thanks for external PRs.

**PR merge preference:** Rebase when commits are clean; squash when history is messy. Always add PR author as co-contributor. After merging a new contributor's PR, run `bun scripts/update-clawtributors.ts` and commit the regenerated README.

Full PR guidelines: `docs/help/submitting-a-pr.md`

---

## Multi-Agent Safety Rules

When multiple AI agents may be running concurrently:

- **Do not** create/apply/drop `git stash` entries unless explicitly asked (no `--autostash`)
- **Do not** create/remove/modify `git worktree` checkouts
- **Do not** switch branches unless explicitly asked
- **Do not** commit unrelated files; scope commits to your own changes
- When told "push": `git pull --rebase` first to integrate latest (never discard others' work)
- When told "commit all": commit in grouped, logical chunks
- If you see unrecognized files, keep going; focus on your changes only
- End reports with a brief note if other agents' files are present and relevant

---

## Gateway Protocol

The gateway uses a typed RPC protocol defined in `src/gateway/protocol/`. Schema is auto-generated:

```bash
pnpm protocol:check     # Validate schema
# Generation scripts:
scripts/protocol-gen.ts         # JSON schema
scripts/protocol-gen-swift.ts   # Swift types for macOS/iOS apps
```

---

## Security & Credentials

- Web provider credentials: `~/.openclaw/credentials/` — rerun `openclaw login` if logged out
- Pi sessions: `~/.openclaw/agents/<agentId>/sessions/*.jsonl`
- Config: `~/.openclaw/` (not configurable)
- **Never** commit real phone numbers, tokens, videos, or live config values
- Use obviously fake placeholders in docs, tests, examples
- Docker: runs as non-root, binds to loopback (`127.0.0.1`) by default

---

## Platform-Specific Notes

### macOS

- Restart gateway via the OpenClaw Mac app or `scripts/restart-mac.sh`
- Check gateway: `launchctl print gui/$UID | grep openclaw`
- Logs: `./scripts/clawlog.sh` (supports follow/tail/category filters; needs passwordless sudo for `/usr/bin/log`)
- **Do not** rebuild the macOS app over SSH — rebuilds must run directly on the Mac
- Packaging: `scripts/package-mac-app.sh` (defaults to current arch)
- Release: read `docs/platforms/mac/release.md` before any release work

### iOS / Android

- Prefer real connected devices over simulators/emulators
- iOS Team ID: `security find-identity -p codesigning -v`
- "Restart app" = rebuild + reinstall + relaunch (not just kill/relaunch)

### Docker

- Entrypoint: `node openclaw.mjs gateway --allow-unconfigured`
- Multi-stage builds, non-root user, loopback binding

---

## Documentation (Mintlify)

- Docs site: `docs/` → published at `https://docs.openclaw.ai`
- Internal doc links: root-relative, no `.md`/`.mdx` extension (e.g. `/configuration`)
- Anchors: root-relative with `#anchor` (avoid em dashes and apostrophes in headings — they break Mintlify anchors)
- README links: use full `https://docs.openclaw.ai/...` URLs (for GitHub rendering)
- Docs content must be generic — no personal hostnames, device names, or paths
- `docs/zh-CN/**` is generated — do not edit unless explicitly asked
- i18n pipeline: English → glossary → `scripts/docs-i18n` → targeted fixes only

---

## Release Channels

| Channel | Tag Pattern | npm dist-tag |
|---------|------------|--------------|
| stable | `vYYYY.M.D` | `latest` |
| beta | `vYYYY.M.D-beta.N` | `beta` |
| dev | head of `main` | `dev` |

**Version locations to update for a release:**
- `package.json`
- `apps/android/app/build.gradle.kts` (versionName/versionCode)
- `apps/ios/Sources/Info.plist` + `apps/ios/Tests/Info.plist`
- `apps/macos/Sources/OpenClaw/Resources/Info.plist`
- `docs/install/updating.md`

Do not change version numbers without explicit consent. Always ask before `npm publish`.

---

## AI/LLM Integration

Supported model providers:
- Anthropic Claude (Pro/Max)
- OpenAI
- AWS Bedrock
- xAI (Grok)
- Baidu Qianfan
- Perplexity
- Local: `node-llama-cpp` (optional peer dep)

Agent framework: `@mariozechner/pi-agent-core` / `pi-ai` / `pi-coding-agent` (Pi framework)

---

## Tool Schema Guardrails (Pi agent tools)

- Avoid `Type.Union` in tool input schemas — no `anyOf`/`oneOf`/`allOf`
- Use `stringEnum`/`optionalStringEnum` (Type.Unsafe enum) for string lists
- Use `Type.Optional(...)` instead of `... | null`
- Keep top-level tool schema as `type: "object"` with `properties`
- Avoid raw `format` as a property name in tool schemas (reserved keyword in some validators)

---

## Useful References

- `AGENTS.md` — Detailed operational guidelines, shorthand commands, VM ops
- `CONTRIBUTING.md` — Contributor rules and PR requirements
- `SECURITY.md` — Security policy and bug reporting
- `docs/testing.md` — Complete testing guide
- `docs/help/submitting-a-pr.md` — PR submission checklist
- `docs/reference/RELEASING.md` — Release process
- `docs/platforms/mac/release.md` — macOS release checklist
