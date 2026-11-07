# claude-video / watch skill

Agent Skills package that gives an agent a video input. Installable across Claude Code (most common host), Codex, Cursor, GitHub Copilot, and 50+ other [Agent Skills](https://agentskills.io) hosts. Pure-stdlib Python with two engines: **gemini** (sends the video to Google's model and passes on its timestamped answer) and **local** (`yt-dlp` + `ffmpeg` frames and a transcript, with an optional Whisper API or separately managed local WhisperX process).

## Structure

- `skills/watch/SKILL.md` — canonical skill contract the model reads when `/watch` fires. Source of truth for behavior across every host.
- `skills/watch/scripts/watch.py` — entry point. Picks the engine (`--engine` > `WATCH_ENGINE` > `auto`; `auto` = gemini when `GEMINI_API_KEY` resolves), then either calls `run_gemini` or runs the local pipeline.
- `skills/watch/scripts/gemini.py` — Gemini engine: YouTube URLs are passed by reference (no download, no binaries); other URLs are downloaded with yt-dlp and, like local files, uploaded to the Files API and deleted after the answer. Local-only options (`--fps`, `--whisper`, …) are reported as ignored, not rejected.
- `skills/watch/scripts/{download,frames,transcribe,whisper,local_whisperx,setup,config,runtime}.py` — yt-dlp wrapper, ffmpeg frame extraction + auto-fps, caption/Whisper transcription, preflight/installer, shared config.

## Local pipeline (non-obvious ordering)

1. **Captions first**: `fetch_captions` reads native subtitles via yt-dlp *without* downloading media. The video is downloaded only when frames or a transcription fallback need it (audio-only when frames aren't needed).
2. **Frames**: scene detection or uniform sampling (or keyframes) within the frame budget from `WATCH_DETAIL`/`--max-frames`. Extra frames at transcript cue timestamps are merged in, then deduplicated.
3. **Transcription fallback** only when there are no captions: backend `groq` / `openai` / `whisperx` / `none`. A failed WhisperX run never switches to a cloud backend by itself.
4. Output is Markdown on stdout for the model. Missing evidence goes in an `## Unavailable evidence` section. Exit 1 only when nothing (no frames, transcript, or caption track) was produced.

**No silent engine fallback**: a failed Gemini run reports the error; it never reruns locally by itself (the user may have chosen Gemini, or not want a long download). Keep this behavior in code and in SKILL.md.

## Configuration

`config.py` is the only reader/writer of settings. Precedence: process environment → `~/.config/watch/.env` → (API keys only) the cwd `.env`. Settings are `WATCH_*` (`ENGINE`, `DETAIL`, `WHISPER_BACKEND`, `GEMINI_MODEL`, `WHISPERX_*`, cookies, `SUB_LANG`). Writes are atomic and refuse to overwrite a malformed file. Keys are never printed or put in a shell command; SKILL.md tells the agent to have the user paste them, or to open the file for them.
- `skills/watch/scripts/build-skill.sh` — builds `dist/watch.skill` for claude.ai upload (dev-only).
- `hooks/` — Claude Code SessionStart setup-status hook (Claude Code only).
- `.claude-plugin/` — `plugin.json` + `marketplace.json` (Claude Code plugin + local marketplace).
- `.codex-plugin/plugin.json` — Codex/agents manifest; `"skills": "./skills/"` points the Agent Skills CLI at the self-contained skill folder.
- `.agents/plugins/marketplace.json` — agents marketplace listing pointing at the repo-root plugin.
- `CLAUDE.md` → `@AGENTS.md` — generic-agent entry point.
- `tests/` — pytest suite (ffmpeg-synthesized clips; no network).

## Orientation

- The product is the slash-command-invoked skill (`/watch <url-or-path> [question]`), not a CLI. `scripts/watch.py` is implementation. Features must work across every harness the skill installs into, not just Claude Code.
- **The skill is one self-contained folder: `skills/watch/`.** SKILL.md and `scripts/` are siblings inside it. This is what lets `npx skills add` copy a working skill as a unit — do NOT move SKILL.md or `scripts/` back to the repo root, or non-Claude installers will copy SKILL.md without the scripts.
- **Path resolution is harness-agnostic.** SKILL.md resolves `SKILL_DIR` as the directory of the SKILL.md the model just Read, then runs `${SKILL_DIR}/scripts/...`. Do NOT reintroduce `${CLAUDE_SKILL_DIR}` (Claude-Code-only) — it is unset on Codex/Cursor/agents and breaks every script call there.
- **No `commands/` wrapper.** `/watch` comes from `name: watch` (skills are user-invocable by default). A separate command file creates a duplicate slash command.
- **SKILL.md frontmatter uses only Agent Skills keys:** `name`, `description`, `license`, `allowed-tools`, `metadata`, `compatibility`. Claude's skill upload rejects anything else; the version lives in `metadata.version`.

## Install surfaces

| Surface | Install |
|---------|---------|
| Claude Code | `/plugin marketplace add bradautomates/claude-video` then `/plugin install watch@claude-video` |
| Codex | `$skill-installer` from `.../tree/main/skills/watch`, or `codex plugin marketplace add` + `codex plugin add watch@claude-video` |
| Cursor / Copilot / +50 | `npx skills add bradautomates/claude-video -g --skill watch` |

Not supported: Claude Chat (including uploading `dist/watch.skill`) and Cowork. `dist/watch.skill` is still built and attached to releases.

## Commands

```bash
# Tests: stdlib + pytest; ffmpeg/ffprobe needed for frame tests. No network or model downloads.
uv run --no-project --with pytest python -m pytest -q        # NixOS: no .venv, system python lacks pytest
uv run --no-project --with pytest python -m pytest -q tests/test_frames.py                       # one file
uv run --no-project --with pytest python -m pytest -q tests/test_release_regressions.py::test_uniform_capped_full_range  # one test

# Build the claude.ai upload bundle (archives skills/watch/ as the bundle root)
bash skills/watch/scripts/build-skill.sh   # → dist/watch.skill

# Dev: mirror the working tree into the installed Claude Code plugin cache
./dev-sync.sh                       # --dry-run to preview
```

### Testing local changes in each host

The published install routes test what's on GitHub, not your working tree. Use these instead:

- **Claude Code**: `claude --plugin-dir .`, or install with `claude plugin marketplace add <repo-path>`. A directory marketplace is read live from the repo; only GitHub installs need `dev-sync.sh`.
- **Codex**: `codex plugin marketplace add .` still installs from the GitHub URL in `.agents/plugins/marketplace.json`. To test local changes, symlink `skills/watch` into `~/.codex/skills/watch`. Run `codex exec --sandbox workspace-write`; the default read-only sandbox can't create the work dir. Add `-c sandbox_workspace_write.network_access=true` to test URLs.
- **OpenCode**: symlink `skills/watch` into `~/.config/opencode/skills/watch` (or a project's `.opencode/skills/watch`).
- Headless runs (`claude -p`, `codex exec`, `opencode run`) give a quick end-to-end check on a short `ffmpeg -f lavfi -i testsrc=duration=4` clip with `--engine local --no-whisper --out-dir ./out`.

## Rules

- Keep the version in sync across `skills/watch/SKILL.md` (`metadata.version`), `.claude-plugin/plugin.json`, and `.codex-plugin/plugin.json` when cutting a release.
- Releasing: tag `vX.Y.Z` and push the tag; `.github/workflows/release.yml` builds `dist/watch.skill` and attaches it to the GitHub release.
- CI (`.github/workflows/tests.yml`) runs on Ubuntu with Python 3.10 and 3.12, plus macOS and Windows, so keep code 3.10-compatible and Windows-safe: explicit UTF-8, bytes for fixtures, `runtime.configure_stdio()`.
- Tests are isolated by an autouse fixture in `tests/conftest.py`. It clears `WATCH_*`/API-key env vars, fakes `HOME`/cwd, repoints `config.CONFIG_FILE`, and blocks uploads. New config paths or env vars must be added there, or tests will read the developer's real config.
- `tests/test_packaging.py` enforces the SKILL.md frontmatter keys and matching versions, and checks that a copied `skills/watch/` runs outside the repo. It also pins the exact set of `scripts/*.py` files, so adding or removing a module means updating that list and SKILL.md's "Bundled scripts" line.
- Never commit real API keys or `.env` contents; keys live in `~/.config/watch/.env` (mode `0600`) at runtime.
