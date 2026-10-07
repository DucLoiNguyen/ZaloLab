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

## Keeping this file current

Update this file in the same change whenever the repo or the runtime changes: persona/rules in `SOUL.md`, config the bot depends on, deployment or operating steps, newly verified behavior, or a gotcha that bit us. Don't commit unless the owner asks.

## Repo contents

- [hermes/cskh/SOUL.md](hermes/cskh/SOUL.md): the canonical persona, guardrails and FAQ text. Deployed by copying to `~/.hermes/profiles/cskh/SOUL.md`; restart the gateway after changes. The FAQ is embedded here (Hermes only loads `SOUL.md`), so there's no separate FAQ file. Structure: 8 rules → forwarding template → "FAQ đã sẵn sàng" (answerable) → "FAQ chưa có nội dung" (forward-only table).
- [hermes/cskh/config.zalo.example.yaml](hermes/cskh/config.zalo.example.yaml): config snippet for the profile (no secrets).
- [docs/SETUP-VPS.md](docs/SETUP-VPS.md), [docs/GUARDRAILS.md](docs/GUARDRAILS.md).

## Operating notes (WSL)

The runtime is on the owner's machine; from some Claude sessions `wsl` reports "not installed", in which case logs can't be inspected, so ask the owner to run commands and paste output. Otherwise run WSL commands from PowerShell with `wsl -e bash <script>`; inline `bash -c '...'` loses quotes and `$VARS` to PowerShell, so put multi-line commands in a script file under the scratchpad.

- Start 9router: `9router -H 127.0.0.1 -n -l --skip-update` (needs a keep-alive process; it exits without a TTY unless `-n -l` are given). Scheduled task `9router` runs it at logon.
- Start gateway: `hermes gateway run`, from the **default** profile. Hermes allows one host gateway for all profiles; `hermes -p cskh gateway run` is refused. Stop with `hermes gateway stop`.
- Inspect the bot: `hermes -p cskh chat -q "..."`, logs at `~/.hermes/profiles/cskh/logs/{gateway,agent}.log`. A healthy FAQ turn logs `tool_turns=0`.
- Release a persona change (the repo's `SOUL.md` is the source, the deployed copy is what runs): back up `~/.hermes/profiles/cskh/SOUL.md` (`SOUL.md.bak-<timestamp>`), copy `/mnt/d/ZaloLab/hermes/cskh/SOUL.md` over it, test with `hermes -p cskh chat -q` (an answerable FAQ, a forward-only FAQ, a legal question, a prompt injection; each should log `tool_turns=0`), then `hermes gateway stop` and `hermes gateway run`, confirm `zalo connected` in `gateway.log`, and have existing Zalo chats send `/new` (see the history gotcha below). `chat -q` reads `SOUL.md` fresh each call; only the gateway needs a restart.

## Gotchas

- The host gateway does **not** load the `cskh` profile's `.env`. Plugin policy (`ZALO_DM_POLICY`, `ZALO_ALLOWED_USERS`) set in `.env` is ignored; set `platforms.zalo.extra.dm_policy` / `allowed_users` in the profile's `config.yaml`. `ZALO_BOT_TOKEN` in `.env` does work.
- Tool lockdown: `platform_toolsets.<platform>` must be `[]` for every platform **including `zalo`**; if the key is missing Hermes uses the full default toolset (terminal, file, code execution). `hermes tools disable --platform X` only worked for `cli`, so edit `config.yaml` directly and verify with `hermes -p cskh tools list --platform zalo`.
- The `custom` provider ignores `OPENAI_API_KEY`; the 9router key goes in `model.api_key` in the profile's `config.yaml`. Don't also put it in `OPENAI_API_KEY` (other features may send it to OpenAI).
- `dm_policy: pairing` asks for a captcha whose "passed" state is in memory only, so every gateway restart re-prompts. Use `allowlist` or `open`.
- The 9router `cc/` (Claude Code) provider injects its own ~2000-token system prompt per call.
- Hermes slash commands (e.g. `/stop`) and the plugin's admin commands (`/kick`, `/warn`...) are available to anyone in `allowed_users`, so keep that list to admins. Verified: `/stop` from Zalo only interrupts that chat's running agent turn (`gateway.log`: `STOP for session ... agent interrupted`); it does not stop the gateway. If the bot goes silent, run `hermes gateway status` / `pgrep -af "hermes|9router"` and restart per the notes above.
- A gateway started with `run_in_background` from a Claude session dies when that session ends (WSL then has nothing keeping the gateway alive), so the bot goes silent until restarted. Only the `HermesGateway` scheduled task makes it durable. The `9router` task is registered and survives.
- **Persona changes don't apply to chats that already have history.** Each Zalo chat keeps one long session (`agent:cskh:zalo:PRIVATE:<uid>`) and the full history is re-sent every turn, so the model imitates earlier replies written under the old persona. Seen after the `SOUL.md` rewrite: bot kept answering `[CHUYEN_NGUOI]` (`response_len=14` in `gateway.log` is the tell) even though the deployed `SOUL.md` had no such string and CLI tests (fresh session) were correct. Fix: the user sends `/new` in that Zalo chat (fresh session, old one kept in the DB); `hermes sessions delete` also works but is destructive. After every persona release, have each existing chat send `/new` (or test via a new chat) before judging the result. Check `history=N` in `agent.log` turn lines.
- The old `[CHUYEN_NGUOI]` escalation marker (from `bot.js`) is gone from `SOUL.md`; Hermes has no code to intercept it, so customers saw it verbatim. Escalation is now the polite template plus a `[Chuyển: ...]` label.
- Group behavior is undocumented: the plugin mentions a per-group "silent" mode (reply only when @tagged or called by name), but whether the bot receives all group messages by default is unverified. Test in a throwaway group and read `gateway.log` before relying on it; don't make the bot answer every group message.
- Windows-side Node (for local checks only) lives at `C:\Users\Loind\Documents\node-v24.21.0-win-x64` and was added to the user PATH; it isn't used by the runtime (that's nvm Node inside WSL).
- Never put real keys, tokens or Zalo user IDs in the repo or commit them. The 9router key and Zalo bot token were pasted into a chat during setup; recommend rotating them.
- Not yet done: the `HermesGateway` scheduled task (auto-start gateway at logon) was blocked and left for the owner to run; no tag filter or escalation to an admin (the bot has no tools, so it can't notify anyone; it only appends a `[Chuyển: CS|Product|Pháp lý]` label).
- Forwarded questions must end with a `[Chuyển: CS|Product|Pháp lý]` label: it's the only way to audit them in the logs. Rule 3 originally let legal-validity questions ("Văn bản này còn hiệu lực không?") slip through with a generic reply and no label; it now forbids asking for the document name and requires the template plus label. Verified via CLI (7 prompts: legal ×3, FAQ-006, forward-only, complaint, injection; all `tool_turns=0`, labels present). Not yet verified end-to-end on Zalo.
- Cannot be done in the persona (needs plugin/bridge code): tag filtering (`requiredTags`), keyword escalation to an admin, per-user cooldown, "asked the same thing twice" detection, notifying anyone.
- FAQ content in `SOUL.md` comes from the `FAQ_Request` sheet (not `FAQ_Master`). Only FAQ-006 has an approved answer; FAQ-001..004 and one unnumbered question are forward-only until the owner supplies approved answers. FAQ-005 is cancelled, don't add it. Never invent product steps/UI names to fill them.
