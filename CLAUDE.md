# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

ZaloLab is a single-file Zalo bot ([bot.js](bot.js), ESM, Node LTS, one dependency: the unofficial `zca-js`). It runs on a secondary Zalo account and does two things:

1. Receives list files (xlsx/xls/csv/docx/pdf) from allowed uploaders, downloads them, runs an external Python extractor, and DMs the resulting `.xlsx` files to the admin.
2. Answers FAQ questions (from [faq.md](faq.md)) when mentioned / a trigger keyword is used in an allowed group, or in DMs from allowed users — via the Anthropic Messages API, falling back to a fixed reply.

The README ([README.md](README.md)) and code comments/logs are in Vietnamese; keep new user-facing text and logs in Vietnamese.

## Project roadmap (owner's original plan)

`bot.js` is the current Zalo-only implementation. The broader plan is to move to an agent-based chatbot:

1. Set up the Hermes agent on a VPS.
2. Set up 9router, add a ChatGPT Plus or Claude account to it, and point Hermes at that source.
3. Prepare the list of channels: Zalo, Teams, Facebook fanpage.
4. Set up guardrails: answering rules so Hermes, acting as chatbot, auto-replies only to customer questions carrying a chosen tag.

Guides: [docs/SETUP-VPS.md](docs/SETUP-VPS.md) (Hermes + 9router + channels) and [docs/GUARDRAILS.md](docs/GUARDRAILS.md). Only the `llm` config block in `bot.js` is implemented: when `config.llm.baseUrl` is set, `answerFaq` calls that OpenAI-compatible endpoint (e.g. 9router at `http://localhost:20128/v1`, key from the env var named by `llm.apiKeyEnv`); otherwise it calls the Anthropic API directly. Hermes, Teams and Fanpage integration don't exist in the repo yet.

## Commands

- Install: `npm install` (use `npm.cmd` on Windows PowerShell if script execution is blocked)
- Run: `node bot.js` — logs in via QR code on every start (no session persistence); scan with the secondary account
- FAQ via AI: set `$env:ANTHROPIC_API_KEY` in PowerShell before running (never put it in code or `config.json`)
- Stop: `Ctrl+C`
- No build, lint, or test tooling exists (`npm test` is a placeholder). `package.json` must keep `"type": "module"`.

## Architecture

Everything is in [bot.js](bot.js), driven by [config.json](config.json) (read once at startup; restart to apply changes). Flow of the `api.listener.on("message")` handler:

1. Debug/`logContent` logging, then drop self messages (`m.isSelf`).
2. `discoveryMode` — only prints thread/sender IDs and returns (auto-enabled if no `allowedGroups`/`allowedUsers` are set). Used to collect IDs for the config.
3. Allowlist gate: groups must be in `allowedGroups`, DM senders in `allowedUsers`; others are silently ignored.
4. File messages (`content` is an object with `href`/`title`): sender must be in `allowedUploaders`, extension in `allowedExt` → `downloadFile` (to `tmp/`, size-capped by `maxFileMB`) → `runExtractor` → `sendToAdmin` with every `.xlsx` found in a per-run `out/<timestamp>/` dir; source file is deleted in `finally`.
5. Text messages: in groups require a bot mention (compared against `ownId`) or a `triggerKeywords` match; DMs always qualify. Per-sender `userCooldownSec` applies; then `answerFaq` → `say`.

Cross-cutting design points:
- All outbound actions go through `enqueue()`: a serial promise queue with random delay (`minDelayMs`–`maxDelayMs`) and a `maxActionsPerMinute` cap. Over-limit tasks are **dropped**, not deferred. These exist to reduce account-ban risk — don't bypass them.
- `dryRun: true` makes `say`/`sendToAdmin` only print `[dryRun] ...`. The checked-in config defaults to `dryRun`, `debug`, `logContent`, and `discoveryMode` all `true`.
- The extractor is an external project (DataExtractionTool, Python) invoked with `execFile` (no shell). `extractor.args` supports placeholders `{input}`, `{outputDir}`, `{outputFile}`; `cwd` is a machine-specific absolute path.
- `answerFaq` calls the Anthropic API directly with `fetch` (no SDK). The system prompt constrains answers to `<faq>`, and the user question is wrapped in `<cau_hoi>` as untrusted data; the model returns `[CHUYEN_NGUOI]` to escalate, which maps to the fixed fallback.
- `out/` is cleaned hourly by `cleanupOldOutputs` (`outputRetentionHours`); `tmp/` holds transient downloads.

## Gotchas

- `zca-js` message shapes (`title`, `href`, `mentions`), `getOwnId`, listener event names, and the `attachments` option for sending files are unverified against real logs and may vary by version — check with `debug` output when something doesn't fire. File download may also need auth cookies.
- The files processed contain personal data: logs should record IDs/names/actions, not message content, unless `logContent` is deliberately enabled for testing. `config.json` holds real IDs once filled in — don't commit real ones or share it.
- Don't log in to Zalo Web with the bot's account while it runs (kills the listener).
- `.gitignore` currently ignores only `node_modules/` and `.env`; `tmp/`, `out/`, and `qr.png` (login QR) are not ignored.
