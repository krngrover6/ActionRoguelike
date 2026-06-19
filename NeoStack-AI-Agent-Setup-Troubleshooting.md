# NeoStack AI (Unreal Plugin) — Agent Setup & Troubleshooting

Notes for getting the **Gemini** and **Codex** agents working inside the NeoStack AI
(UltimateCoPilot) Unreal Engine plugin on macOS.

- **Machine when this was solved:** Apple Silicon (Mac15,6), macOS 26.5, Node/npm in `/usr/local`
- **Date solved:** 2026-06-19
- **Plugin:** NeoStack AI / UltimateCoPilot (registers itself in `ActionRoguelike.uproject`)

There were **three separate problems**. Each is documented below with symptom → cause → fix → how to verify.

---

## 1. Gemini won't sign in ("migrate to Antigravity")

**Symptom**
- In the Gemini CLI / plugin auth, choosing **"Sign in with Google"** fails with:
  > Failed to sign in. This client is no longer supported for Gemini Code Assist for
  > individuals. To continue using Gemini, please migrate to the Antigravity suite...

**Cause**
- Google **discontinued the free "Sign in with Google" (Code Assist for individuals)** OAuth
  path for the Gemini CLI client. Retrying never works.
- A consumer **Gemini Pro / Advanced subscription does NOT work** with the CLI/API — it only
  covers the Gemini app/website. CLI + API access is a separate, key-based product.

**Fix — use a free Gemini API key instead of Google sign-in**
1. Create a free key: https://aistudio.google.com/apikey → "Create API key" (no credit card).
2. Save it where the CLI **and** the GUI-launched plugin can both read it:
   ```bash
   printf 'GEMINI_API_KEY=YOUR_KEY_HERE\n' > ~/.gemini/.env && chmod 600 ~/.gemini/.env
   ```
   > Why `~/.gemini/.env`: the Gemini CLI reads this file itself on every launch, regardless
   > of how it's started. macOS GUI apps (like Unreal launched from the Dock) do **NOT** read
   > `~/.zshrc`, so an `export` in your shell profile won't reach the plugin — but `.env` will.
3. (Optional, for plain terminal use too) `echo 'export GEMINI_API_KEY="YOUR_KEY_HERE"' >> ~/.zshrc`
4. In the Gemini CLI auth menu choose **"Use Gemini API Key"** (option 2), not "Sign in with Google".
5. Fully quit and reopen Unreal.

**Verify**
- Terminal: run `gemini` → header should say **"Authenticated with gemini-api-key"**.
- `cat ~/.gemini/settings.json` → `selectedType` should be `gemini-api-key`.

---

## 2. Codex crashes instantly: "Agent process exited with code 16777214"

This single error code was caused by **two independent problems**. Fix both.

### 2A. "Computer Use" helper killed by macOS

**Symptom**
- macOS crash dialog for **`SkyComputerUseClient`** (`com.openai.sky.CUAService.cli`),
  exception `EXC_CRASH (SIGKILL (Code Signature Invalid))`, dying ~13 ms after launch.
- Repeated `SkyComputerUseClient-*.ips` files in `~/Library/Logs/DiagnosticReports/`.

**Cause**
- OpenAI's Codex **"Computer Use"** desktop-automation helper is killed by macOS 26.5's
  runtime **code-signing monitor**, *even though the binary is validly signed and notarized*.
  This is an upstream OpenAI × macOS-26 incompatibility — **not fixable locally**.
- The Codex agent in the plugin does **not need** Computer Use (screen/mouse control), but the
  crash dragged the whole agent down.

**Fix — disable Computer Use in Codex config**
Edit `~/.codex/config.toml` (a backup is auto-saved as `~/.codex/config.toml.bak-*`):
```toml
# was: notify = [".../SkyComputerUseClient", "turn-ended"]
notify = []

[plugins."computer-use@openai-bundled"]
enabled = false        # was true
```

**Verify**
- After restarting Unreal and using Codex, **no new** `SkyComputerUseClient-*.ips` files appear
  in `~/Library/Logs/DiagnosticReports/`.

### 2B. The real cause of `16777214` — wrong/missing `codex-acp` version

**Symptom**
- Unreal log (`~/Library/Logs/Unreal Engine/ActionRoguelikeEditor/*.log`):
  ```
  ACPClient: Launching Codex CLI: /usr/local/bin/npx @zed-industries/codex-acp@0.16.0
  ACPClient: Process started successfully ...
  Error: ACPClient [Codex CLI]: Agent process exited with code 16777214
  ```

**Cause**
- The plugin launches the Codex agent via **`npx @zed-industries/codex-acp@0.16.0`** (Zed's ACP
  bridge — separate from the `codex` CLI). That **exact version wasn't installed**, so npx tried
  to **download it on every launch** and aborted in the plugin's non-interactive context →
  reported as exit code `16777214`.
- Diagnosed by running the bridge manually — it printed
  `npm warn exec The following package was NOT FOUND and will be installed: @zed-industries/codex-acp@0.16.0`
  then initialized fine once downloaded. So the bridge code is healthy; only availability was the problem.

**Fix — install the exact bridge version globally so npx resolves it instantly**
```bash
npm install -g @zed-industries/codex-acp@0.16.0
```
> If the plugin updates and asks for a different version later, install that exact version the
> same way. Check the version in the Unreal log line `Launching Codex CLI: ... codex-acp@X.Y.Z`.

**Verify**
```bash
# Should print a valid initialize response and NOT say "will be installed":
cd "/path/to/your/UnrealProject"
echo '{"jsonrpc":"2.0","id":0,"method":"initialize","params":{"protocolVersion":1,"clientCapabilities":{}}}' \
  | npx @zed-industries/codex-acp@0.16.0
```
Look for `"agentInfo":{"name":"codex-acp",...,"version":"0.16.0"}` and **no** "will be installed" warning.

---

## Reference: the Codex toolchain pieces (don't confuse them)

| Thing | What it is | Install |
|---|---|---|
| `codex` | the Codex CLI itself | `npm install -g @openai/codex` (per plugin docs) |
| `codex-acp` | Zed's **ACP bridge** the plugin actually launches; embeds codex | `npm install -g @zed-industries/codex-acp@<version>` |
| Computer Use | optional desktop screen-control helper (broken on macOS 26.5) | disabled in `~/.codex/config.toml` |
| Codex auth | ChatGPT login | stored in `~/.codex/auth.json` (`tokens`); no API key needed |

---

## Quick checklist if an agent breaks again

1. **Which agent?** Gemini → check `~/.gemini/.env` has `GEMINI_API_KEY`. Codex → continue.
2. **Read the Unreal log:** `~/Library/Logs/Unreal Engine/ActionRoguelikeEditor/` (newest `*.log`).
   Find the `Launching ... : npx ...@<version>` line and the error after it.
3. **`16777214` / instant exit?** The requested `codex-acp@<version>` probably isn't installed →
   `npm install -g @zed-industries/codex-acp@<version>`.
4. **`SkyComputerUseClient` crash in `~/Library/Logs/DiagnosticReports/`?** → confirm Computer Use
   is still disabled in `~/.codex/config.toml` (`notify = []`, plugin `enabled = false`).
5. **GUI vs terminal:** the plugin runs from Unreal (GUI) and does NOT inherit `~/.zshrc`. Put
   keys in the tool's own config file (`~/.gemini/.env`, `~/.codex/auth.json`), not just shell exports.
6. After any change, **fully quit (Cmd+Q) and reopen Unreal** so agents respawn with new config.

---

## Revert / backups
- Codex config backup: `~/.codex/config.toml.bak-YYYYMMDD-HHMMSS`
  - Restore: `cp ~/.codex/config.toml.bak-* ~/.codex/config.toml`
- To re-enable Computer Use later (once OpenAI ships a macOS-26-compatible build):
  set `enabled = true` and restore the original `notify` line.
