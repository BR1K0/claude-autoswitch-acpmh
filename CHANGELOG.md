# Changelog

## 0.3.8

- **The handoff tells part 2 to search the journal before answering "unknown".** In a new test on 9 real chats,
  Haiku and Sonnet almost never looked anything up and fell as low as 51% of re-reading; with this rule they beat
  compaction at 200k and roughly tie it at 500k and above. Opus already searched on its own.
- **New benchmark: handoff vs compaction across many chats** ([docs/HANDOFF_VS_COMPACTION.md](docs/HANDOFF_VS_COMPACTION.md)),
  with the script to rerun it on your own chats (`bench/handoff_vs_compaction.py`, subscription only). The README
  table now shows those measured results, replacing the earlier one-chat numbers and the 500k estimate.
- **Docs:** the repo is now `claude-autoswitch-acpmh` (was `claude-autoswitch`). Old links redirect. The plugin and
  marketplace names are unchanged, so `autoswitch-inline@claude-autoswitch` still installs.

## 0.3.7

- **Handoffs never use an API key.** The handoff ran ACPMH's Haiku fact extraction, billed to your API key, whenever
  `ANTHROPIC_API_KEY` was set in the shell, without saying so. A key being set is not consent: handoffs no longer
  run it, and the key is removed from ACPMH's processes. `acpmh.cli sync` itself now refuses unless
  `ACPMH_SYNC_API=on`.
- **Keys are redacted from everything ACPMH stores**, not only screenshot text: every event and tool output.
- **Docs:** the repo is now `claude-autoswitch` (was `claude-autoswitch-v2`); the README drops the v1/v2 names,
  says "credit savings" in the title, and compares handoffs with compaction at 266k, 500k and 1M.

## 0.3.6

- **The newest part is listed first.** A chat's sidebar group is now sorted by creation date, newest on top, so
  the part you're working in is the easiest to reach. Sorted by recency, part 1 sat on top because its
  "Continued in part N" reply was the last thing posted.

## 0.3.5

- **A dev server no longer holds a chat past the rollover point.** A running background command used to block the
  rollover until it was 30 minutes old, and each restart reset the clock. Now the chat rolls over anyway: the
  command keeps running, owned by the old part, and the new part's handoff names it and how to restart it.
  `AUTOSWITCH_ROLLOVER_WAIT_BACKGROUND=on` restores the wait.

## 0.3.4

- **Turns never fall back to an older Opus.** A chat made before Opus 5.5 kept Opus 4.7 as its app setting, and turns
  the router had no word on ran there, re-reading the whole chat into a new cache each flip. Every turn now runs on
  the newest version of its tier (a pin keeps its tier, on the current version).
- **Old parts stay quiet.** When a rolled-over chat's next part ends a turn, the app wakes the old part, which
  re-read its 200k-400k chat just to say "continued in part N". Those turns are now dropped before any model call.
- **Model switching uses the exact chat size** from the API usage, minus the chat's starting size (tools and system
  prompt). The old count read 30-35% low, so chats switched in place well past break-even.

## 0.3.3

- **A cheaper model never carries over between turns.** 0.3.2 kept unscored turns on the previous turn's model; after
  an easy message went to Haiku, a harder message the router was unsure about (or a router failure) could stay on
  Haiku. Now only a move up, or to a newer version of the same tier, carries over; every Haiku turn needs the
  router's fresh yes, and anything else goes back to the chat's model.

## 0.3.2

- **No more model bouncing in long chats.** Turns the router does not score (task notifications, background
  results, a router error) now stay on the model the previous turn ran on, instead of going back to the chat's
  model setting. Each bounce re-read the whole chat on the other model; in one measured project this cost more
  than running everything on Opus 5.5. The hook also decides from the model the plugin really runs the chat on.
  Picking a model by hand still takes over at once.
- **Long chats move to the tier's newest model too** (Opus 4.7 to Opus 5.5). It is cheaper per token, so the one
  re-read pays back in about 17 model calls whatever the chat's size. `AUTOSWITCH_UPGRADE_MAX_TOKENS` still sets a cap.

## 0.3.1

- **Screenshot-only messages are routed by their text.** The `inline` plugin passes the turn's attached images
  (the app's `[Image: source: …]` files, first 3) to the hook, which reads their text with ACPMH's macOS Vision
  OCR (2 s budget) and scores it like a paste. An image with no readable text (under 20 characters) keeps the
  current model; an empty prompt is never scored. Image turns get a 3 s decision budget instead of 1.5 s.
- **The band names the exact model** that answered ("↪ Haiku 5.5 answered") and updates on every turn.

## 0.3.0

Two new speeds that answer the first message on the right model, with no stop notice and no wait.

- **`inline` (recommended, macOS):** the bundled Claude Code plugin `autoswitch-inline` (`plugin/`, marketplace in
  `.claude-plugin/`) picks each main-loop turn's model itself, from the installed hook's decision (`hint.py` with
  `AUTOSWITCH_DRY=1`). The hook stops, queues and shows nothing in those chats, and they need no Watcher. The
  plugin marks each chat it runs in (`~/.autoswitch/inline/<chat id>`); chats opened before it was installed fall
  back to `predictive`. Fails open: a router error or a decision slower than 1.5 s leaves the turn's model alone.
- **Budget bar:** with `inline`, a band above the prompt box shows the last turn's model ("↪ Opus 5.5 (kept)",
  "↪ Haiku 5.5 answered") and Save / Balanced / Quality buttons that write `~/.autoswitch/budget`.
- **`predictive` (macOS):** a read-only reader of the prompt box (`hook/typeahead.swift`, started by the Watcher)
  switches the chat while you type. At Enter, a message whose switch has not landed is held like `seamless` only for
  an up-switch or a message of 400+ characters (`AUTOSWITCH_HOLD_CHARS`); otherwise it goes through at once.
- **Same tier, newer model:** a chat on an older version of its tier (Opus 4.7, Sonnet 5, ...) moves to the tier's
  current model, except in long chats (`AUTOSWITCH_UPGRADE_MAX_TOKENS`, 100k), pinned chats and handoffs. New chats
  get the request's model from the plugin (`AUTOSWITCH_CURRENT_MODEL`).
- **`stoptext`:** `echo hide > ~/.autoswitch/stoptext` hides the "↪ Switched to X" text on a held message.
- `autoswitch speed [inline|predictive|instant|seamless]`; `inline` also runs `claude plugin marketplace add` and
  `claude plugin install autoswitch-inline@claude-autoswitch --scope user` (or prints them). `install.sh` copies the
  plugin and builds the typeahead reader; `uninstall.sh` removes both.
- The dry-run decision never writes state: no pin observation, no shadow log, no lasting-switch record, and the
  clipboard check no longer records what it saw.
- Docs: README "Speed" section for all four speeds, the band, the newer-model upgrade; new demo image of the band.
- **Windows: `inline`.** The plugin finds the home folder from `HOME`, then `USERPROFILE`, joins paths with the
  platform's separator (Windows is told by `OS=Windows_NT`) and runs the router with the launcher `install.ps1`
  recorded, then `py -3`, then `python` (the first that works is remembered; still one 1.5 s budget, fail-open). The
  decision log is appended through the engine's file API instead of `/usr/bin/tee`, keeping the last 2000 rows.
  `install.ps1` copies the plugin and its marketplace, `autoswitch speed inline` installs it as on macOS, and
  `uninstall.ps1` removes it. Not yet tried live in the Windows desktop app.
- **Windows: `predictive` is refused.** `autoswitch speed predictive` says it needs macOS and keeps the current speed;
  a chat opened before the plugin falls back to `instant` on Windows (not `predictive`).
- **CI:** `.github/workflows/tests.yml` runs the Python suite and the plugin's `claude plugin validate` / `test` on
  Ubuntu, macOS and Windows, and parses `install.ps1` / `uninstall.ps1` on Windows.

## 0.2.0

v2, everything off by default (`~/.autoswitch/handoff`, `~/.autoswitch/rollover`); with both off, decisions are
exactly v0.1's (tested).

- **Handoff to a new chat** for long chats, built by ACPMH (now bundled in `acpmh/`): v4 from 70k to 300k tokens, v5b
  above (balanced). Below ~70k the chat still switches in place. Only for lasting switches.
- **Budget modes** quality / balanced / save, optional spending limit with pacing, calibration hook.
- **Context-aware scoring** for short messages.
- **Real history size** from the transcript.
- **Exactly-once delivery** of the handed-off message, settings carried over (never more), "please resend" and
  "Autoswitch is paused" notes, one Watcher at a time.
- **Automatic rollover** of chats over 200k into a part 2, with jobs, crash recovery and a loop guard; optional
  `hook/stop.py` Stop hook.
- Benchmark in `docs/BENCHMARK.md`.

## Unreleased

- Instant mode is a first-class setting. `autoswitch speed [instant|seamless]` writes `~/.autoswitch/speed` (no argument prints the current value) and `autoswitch status` shows it. `install.sh` / `install.ps1` write `speed=instant` on a fresh install only (no `mode` file yet) and never overwrite an existing `speed` file; fresh installs still start in shadow mode, and upgrades keep `seamless` unless you change it. `WATCHER.md` and `hook/watch.py` spell out `send: false` (switch only, nothing sent or started) on macOS and Windows. README: new "Speed: instant vs seamless" section; trade-off: with `instant` the first message after a change of need is answered by the current model. Tests for instant with and without a target, pinned chats, new chats and screenshots, and for the CLI on both platforms.
- Faster switches: the Watcher now makes only the two app calls (set model, send) and no longer runs `finish.py done` on success. The request is closed by `finish.py settle` (started by `watch.py`) once the re-sent prompt reaches the hook, or as an error with a "please resend" note after 120 s. `watch.py` polls every 0.05 s (was 0.1 s). Older Watcher prompts keep working.
- Windows support: `install.ps1` / `uninstall.ps1`, a Task Scheduler task for the scorer, `autoswitch.cmd` on the user PATH, screenshots detected via the Windows clipboard (no OCR yet). Paths come from one helper (`hook/paths.py`): `%APPDATA%\Claude` on Windows. The Watcher loop is now cross-platform Python (`hook/watch.py`; `watch.sh` is a one-line wrapper) and `WATCHER.md` is rendered per platform (`python3` on Mac, `py -3`/`python` on Windows). Also: queue files are replaced with `os.replace`, hook input is read as UTF-8, settings.json keeps its line endings. macOS behaviour is unchanged.
- Manual pins: a model change made by hand (the picker or `/model`) pins the chat; pinned chats are never switched. `autoswitch unpin` clears pins.
- The unclear-prompt move to Sonnet only applies to new or short chats; unknown history size never passes a down-switch.

- Switch text is exactly "↪ Switched to Haiku" (Sonnet / Opus likewise). The app adds its own prefix; no hook output avoids it in the CLI.
- New speed setting: `AUTOSWITCH_SPEED=instant` (or the file `~/.autoswitch/speed`). The hook never holds the prompt: the current message answers on the current model and the switch applies from the next one. Default stays `seamless` (hold, switch, re-send). Trade-off: with `instant` the first message after a change of need runs on the old model.
- Watcher polls every 0.1 s and is told to make set-model, re-send and file-record calls in one message (ordering not verified live).

## 0.1.1

- Switching no longer shows "blocked": the hook stops the turn with a plain "Switching to Haiku..." text.
- Leaner Watcher prompt: re-send and file the record in one step.
- A request that cannot be parsed still leaves a "please send it again" note; `finish.py done` no longer crashes on odd request files; the Watcher files `error` when its fallback worker cannot start.

## 0.1.0

First public release.

- Local scorer (haiku / sonnet / opus) trained on public data: synthetic chat prompts plus SWE-bench issues labelled by what actually solved them.
- `UserPromptSubmit` hook with policy: confidence gates, dwell guard for long chats, hand-picked models respected, Sonnet as the default for unclear prompts.
- Watcher flow: hold the prompt, switch the chat's model, re-send; fallbacks and "send it again" notices so nothing is lost silently.
- Screenshot awareness: pasted images are detected, read with macOS Vision and never held.
- Never blocks outside the desktop app, agent-to-agent messages, tool notifications, or when no Watcher is running.
- `./install.sh` (private venv, launch agent, safe idempotent settings edit with backup, shadow mode by default) and `./uninstall.sh`.
- `autoswitch on|off|shadow|hint|status|report|watcher`.
- Hook latency about 70 ms.
