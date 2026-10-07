# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

ZaloLab is a Vietnamese-language Zalo FAQ chatbot. **The repo contains no application code**, only config templates and docs. The runtime lives outside the repo, inside WSL Ubuntu on the owner's Windows machine:

```
Zalo ──► zalo-platform plugin ──► Hermes Agent (profile cskh) ──► 9router (localhost:20128/v1) ──► Claude
```

- **Hermes Agent** (Nous Research) installed at `~/.hermes/hermes-agent`, profile `cskh` at `~/.hermes/profiles/cskh/`.
- **zalo-platform** (community plugin, catalog-pinned SHA, official Zalo Bot API via `ZALO_BOT_TOKEN`) installed in the `cskh` profile.
- **9router** (npm global, Node via nvm in WSL) serves an OpenAI-compatible endpoint; the model combo is `ZaloLab`.

The previous `zca-js` bot (`bot.js`, nick phụ, file extraction → `.xlsx`) was removed; last version is commit `03e5478`. The official Bot API can't receive files, so that feature is gone.

README and docs are in Vietnamese; keep new user-facing text and docs in Vietnamese.

## Repo contents

- [hermes/cskh/SOUL.md](hermes/cskh/SOUL.md): the canonical persona, guardrails and FAQ text. Deployed by copying to `~/.hermes/profiles/cskh/SOUL.md`; restart the gateway after changes. The FAQ is embedded here, so there's no separate FAQ file.
- [hermes/cskh/config.zalo.example.yaml](hermes/cskh/config.zalo.example.yaml): config snippet for the profile (no secrets).
- [docs/SETUP-VPS.md](docs/SETUP-VPS.md), [docs/GUARDRAILS.md](docs/GUARDRAILS.md).

## Operating notes (WSL)

Run WSL commands from PowerShell with `wsl -e bash <script>`; inline `bash -c '...'` loses quotes and `$VARS` to PowerShell, so put multi-line commands in a script file under the scratchpad.

- Start 9router: `9router -H 127.0.0.1 -n -l --skip-update` (needs a keep-alive process; it exits without a TTY unless `-n -l` are given). Scheduled task `9router` runs it at logon.
- Start gateway: `hermes gateway run`, from the **default** profile. Hermes allows one host gateway for all profiles; `hermes -p cskh gateway run` is refused. Stop with `hermes gateway stop`.
- Inspect the bot: `hermes -p cskh chat -q "..."`, logs at `~/.hermes/profiles/cskh/logs/{gateway,agent}.log`. A healthy FAQ turn logs `tool_turns=0`.

## Gotchas

- The host gateway does **not** load the `cskh` profile's `.env`. Plugin policy (`ZALO_DM_POLICY`, `ZALO_ALLOWED_USERS`) set in `.env` is ignored; set `platforms.zalo.extra.dm_policy` / `allowed_users` in the profile's `config.yaml`. `ZALO_BOT_TOKEN` in `.env` does work.
- Tool lockdown: `platform_toolsets.<platform>` must be `[]` for every platform **including `zalo`**; if the key is missing Hermes uses the full default toolset (terminal, file, code execution). `hermes tools disable --platform X` only worked for `cli`, so edit `config.yaml` directly and verify with `hermes -p cskh tools list --platform zalo`.
- The `custom` provider ignores `OPENAI_API_KEY`; the 9router key goes in `model.api_key` in the profile's `config.yaml`. Don't also put it in `OPENAI_API_KEY` (other features may send it to OpenAI).
- `dm_policy: pairing` asks for a captcha whose "passed" state is in memory only, so every gateway restart re-prompts. Use `allowlist` or `open`.
- The 9router `cc/` (Claude Code) provider injects its own ~2000-token system prompt per call.
- Never put real keys, tokens or Zalo user IDs in the repo or commit them. The 9router key and Zalo bot token were pasted into a chat during setup; recommend rotating them.
- Not yet done: the `HermesGateway` scheduled task (auto-start gateway at logon) was blocked and left for the owner to run; no tag filter or escalation to an admin (the bot has no tools, so it can't notify anyone; it only appends a `[Chuyển: CS|Product|Pháp lý]` label).
- FAQ content in `SOUL.md` comes from the `FAQ_Request` sheet (not `FAQ_Master`). Only FAQ-006 has an approved answer; FAQ-001..004 and one unnumbered question are forward-only until the owner supplies approved answers. FAQ-005 is cancelled, don't add it. Never invent product steps/UI names to fill them.
