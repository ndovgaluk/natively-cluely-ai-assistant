# Windows local setup (without the private premium submodule)

Notes from setting Natively up from source on Windows 11 on 2026-11-19. They cover
install, build, run and first configuration for a clone that **cannot** fetch the
private `premium` submodule (`git ls-remote` on it returns "Repository not found").

## Prerequisites

| Tool | Needed | Tested with |
|---|---|---|
| Node.js | `>=22.13.0` (`package.json` `engines`) | 24.18.0 |
| npm | — | 11.16.0 |
| Python (node-gyp) | 3.x | 3.14.6 |
| Visual Studio Build Tools, C++ workload + Windows SDK | builds `better-sqlite3`, `keytar` and the Rust module's C code | Build Tools 2022 (17.14), SDK 10.0.26100 |
| Rust (rustup, MSVC toolchain) | builds `native-module` (mic + system audio) | rustc / cargo 1.99.1 |
| git | — | 2.55.0 |

## Install

Run from the repo root in PowerShell. Scripts are skipped at install and the
`postinstall` steps are run by hand, so each one is visible.

```powershell
Copy-Item .env.example .env          # then edit: keep only keys you have, delete placeholder lines
npm ci --ignore-scripts              # packages from the lockfile, no install scripts

npx patch-package                    # applies patches/ (better-sqlite3, keytar)
npm rebuild sharp
node scripts/rebuild-native-electron.js   # compiles better-sqlite3 + keytar for Electron (C++)
node scripts/download-models.js      # ~150 MB from huggingface.co, sha256-verified; required
npm run build:native                 # cargo build of native-module -> native-module\index.win32-x64-msvc.node
```

- `scripts/ensure-sqlite-vec.js` is not needed on Windows: `sqlite-vec-windows-x64`
  is installed by `npm ci`; the script only adds the macOS packages.
- Red `NativeCommandError` text in Windows PowerShell 5.1 is PowerShell wrapping
  stderr; it is not a failure. Check the last line ("rebuilt dependencies
  successfully", "Rebuild Complete").
- npm 11 warns that packages "have install scripts not yet covered by
  allowScripts". Expected with `--ignore-scripts`; do not run
  `npm approve-scripts` unless you want those scripts to run.
- `.env`: the app saves what `.env` holds into its credential store, so do not
  leave placeholders like `DEEPGRAM_API_KEY=your_deepgram_key_here`.

## Run (debug)

`premium/` is empty, so the normal build fails with
`Could not resolve "../premium/electron/knowledge/types"` (and two more).
Use core-smoke mode, which marks `../premium/...` imports as external
(`scripts/build-electron.js`):

```powershell
$env:NATIVELY_CORE_SMOKE = "1"
npm run app:dev
```

- Vite dev server on `http://127.0.0.1:5180` (`--strictPort`), then Electron.
- Main-process log: `%USERPROFILE%\Documents\natively_debug.log`.
- App data: `%APPDATA%\natively` (settings, encrypted credentials, SQLite DB,
  local telemetry in `logs\telemetry.jsonl`).
- Agent/CDP debugging: `npm run dev:agent` (free CDP port, worktree-local userData).
- Stop with Ctrl+C (`concurrently --kill-others` stops both processes).

Calendar warnings at build time (`GOOGLE_CALENDAR_CLIENT_SECRET`,
`MICROSOFT_CALENDAR_CLIENT_ID` not set) only mean calendar sync is off.

## Configure

### Speech-to-text (Settings → Audio → Speech Provider)

The default provider is **Google Cloud**, which needs a Service Account JSON
(not an API key). Without it, or when the chosen provider has no key, the overlay
shows **"Transcription Stopped"** and the log shows:

```
[WARN] [Main] No API key for Deepgram STT, falling back to GoogleSTT
[ERROR] [Main] STT (user) Error: Google speech-to-text has no working Service Account JSON.
```

Pick a provider, paste its key, **Save**, **Test Connection**, then start a new
meeting (the provider is chosen when a meeting starts):

| Provider | Needs |
|---|---|
| Local Models (`local-whisper`) | nothing; runs on the PC, downloads a model first |
| Groq Whisper | Groq API key |
| Deepgram Nova-3 (used here, works) | Deepgram API key |
| OpenAI Whisper | OpenAI **API** key (a Codex/ChatGPT sign-in does not work) |
| ElevenLabs, Azure, IBM Watson | that vendor's key (+ region for Azure/IBM) |
| Google Cloud | Service Account JSON (`GOOGLE_APPLICATION_CREDENTIALS` in `.env`) |
| Apple Speech | macOS only, hidden on Windows |

### Answer model

- Change it in the **model picker at the bottom-left of the meeting overlay**, or
  in **Settings → AI Providers** (default model and the Fast Response model).
- Keys go in **Settings → AI Providers** (Gemini, Groq, OpenAI, Claude,
  OpenRouter, …).
- "I couldn't reach the AI provider — this looks like an API key or rate-limit
  issue" is shown for any provider error. Check the log for the real cause; here
  it was `Codex upstream 429 ... usage_limit_reached` (ChatGPT plan quota used up
  for the OpenAI Codex / `codex-cli` model).
- The Fast Response model must produce a first word within 4 s
  (`no first word within 4000ms`); a small fast model (Gemini Flash, Groq) suits
  it better than a large one.

### Hotkeys (Settings → Keybinds)

If another app owns a shortcut, the log repeats
`Could not recover shortcut CommandOrControl+Shift+Space ... (OS conflict?)` and the
overlay offers **Rebind**. Seen here for Ctrl+Shift+Space and Ctrl+Shift+H.

## What does not work without `premium/`

- The **grid icon** in the Launcher header opens **Modes**; it renders an empty
  panel (`src/components/settings/ModesSettings.tsx` returns a NullComponent
  without `premium/`). Meetings run on the built-in default mode.
- Licence/Pro features, knowledge (résumé/JD) profile, company research
  (`electron/services/resolveCompanySearchProvider.ts`) may throw
  `Cannot find module '../premium/...'` when used.

## Things to know

- **Open at login** is on by default for fresh installs (commit `7d05eb05`). Its
  settings toggle was not located; Windows Settings → Apps → Startup also turns
  it off.
- The renderer sends usage analytics to Google Analytics 4
  (`src/lib/analytics/analytics.service.ts`). PostHog/Sentry/Axiom are only used
  when their env keys are set.
- `.husky/pre-commit` runs `npx --yes react-doctor@latest` when `react-doctor` is
  not installed locally (unpinned package). Set `HUSKY=0` to skip it when committing.

## Clean up

Delete `node_modules`, `dist`, `dist-electron`, `native-module\target`,
`resources\models` (downloaded models), `%APPDATA%\natively`, the stored keys in
Windows Credential Manager (keytar), and the open-at-login entry.
