---
name: video-use-install
description: Install video-use into the current agent (Claude Code as a plugin, or Codex/Hermes/Openclaw via a skills symlink) and wire up ffmpeg + the ElevenLabs API key so the user can start editing immediately.
---

# video-use install

Use this file only for first-time install or reconnect. For daily editing, read `skills/video-use/SKILL.md`. Always read `skills/video-use/helpers/` — that's where the scripts live.

## What you're doing

You're setting up a conversation-driven video editor for the user. After install, the user drops raw footage into any folder, runs their agent (`claude`, `codex`, etc.) there, and says "edit these into a launch video." You do the rest by reading `skills/video-use/SKILL.md`.

Three things must exist on this machine:

1. The video-use skill registered with the agent (Claude Code plugin, or a symlinked skills folder).
2. `ffmpeg` on `$PATH` (plus optional `yt-dlp` for online sources).
3. An ElevenLabs API key (env var `ELEVENLABS_API_KEY`, or `~/.config/video-use/.env`) for Scribe transcription.

## Install prompt contract

- Do everything yourself. Only ask the user for things you cannot generate — the ElevenLabs API key, and confirmation before `brew install` / `sudo`.
- The skill references helpers by bare name (`transcribe.py`, `render.py`). That works because `SKILL.md` and `helpers/` ship together inside `skills/video-use/` — keep them as siblings.
- After install, verify by running one real command against one real file. Don't declare success on file-existence checks alone.

## Steps

### 1. Register the skill with the current agent

- **Claude Code (recommended: plugin).** Inside Claude Code run:

    ```
    /plugin marketplace add cryptogus056-beep/video-use
    /plugin install video-use@video-use
    ```

    The repository is private, so the machine needs GitHub access to it (`gh auth login`, or git credentials that can read the repo). Claude Code caches the plugin; `/plugin marketplace update video-use` pulls new versions.

- **Codex / Hermes / Openclaw / any agent with a skills directory.** Clone and symlink the skill folder (not the repo root):

    ```bash
    test -d ~/Developer/video-use || git clone https://github.com/cryptogus056-beep/video-use ~/Developer/video-use
    mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
    ln -sfn ~/Developer/video-use/skills/video-use "${CODEX_HOME:-$HOME/.codex}/skills/video-use"
    ```

    If the agent has no skills directory, point its system prompt / config at `~/Developer/video-use/skills/video-use/SKILL.md`.

If you can't tell which agent you're in, ask the user once.

### 2. Install Python deps

```bash
python -c "import requests, librosa, matplotlib, PIL, numpy" 2>/dev/null \
  || pip install requests librosa matplotlib pillow numpy
```

No console scripts — helpers are invoked directly as `python <skill dir>/helpers/<name>.py`.

### 3. Install ffmpeg (+ optional yt-dlp)

`ffmpeg` and `ffprobe` are hard requirements. `yt-dlp` is only needed if the user wants to pull sources from URLs. Animation engines such as HyperFrames, Remotion, and Manim are installed lazily the first time a project actually needs them.

```bash
# macOS
command -v ffmpeg >/dev/null || brew install ffmpeg
command -v yt-dlp >/dev/null || brew install yt-dlp     # optional

# Debian / Ubuntu
# sudo apt-get update && sudo apt-get install -y ffmpeg
# pip install yt-dlp

# Windows
# winget install Gyan.FFmpeg

# Arch
# sudo pacman -S ffmpeg yt-dlp
```

If a package manager requires a sudo prompt, tell the user the exact command and wait. Do not invent a password.

### 4. ElevenLabs API key

Scribe (ElevenLabs) does all transcription. Without a key, nothing transcribes.

1. Check existing state in this order and stop at the first hit:

    ```bash
    [ -n "$ELEVENLABS_API_KEY" ] && echo "env"
    grep -q '^ELEVENLABS_API_KEY=..' ~/.config/video-use/.env 2>/dev/null && echo "dotenv"
    ```

2. If neither is set, ask the user once for a key from https://elevenlabs.io/app/settings/api-keys. Prefer a secure input mechanism if the agent has one; otherwise let the user write the file themselves. Store it outside the plugin folder (which is replaced on every plugin update):

    ```bash
    mkdir -p ~/.config/video-use
    printf 'ELEVENLABS_API_KEY=%s\n' "$KEY" > ~/.config/video-use/.env
    chmod 600 ~/.config/video-use/.env
    ```

    Never echo the key back in tool output. Never commit it.

3. Sanity check with a cheap, quota-free call:

    ```bash
    curl -s -o /dev/null -w '%{http_code}\n' \
      -H "xi-api-key: ${ELEVENLABS_API_KEY:-$(sed -n 's/^ELEVENLABS_API_KEY=//p' ~/.config/video-use/.env)}" \
      https://api.elevenlabs.io/v1/user
    ```

    `200` means the key works. `401` means a wrong/expired key — ask once more and stop. Anything else (network, 5xx), move on and verify during the first real transcription.

### 5. Verify end-to-end

```bash
python <skill dir>/helpers/timeline_view.py --help >/dev/null && echo "helpers OK"
ffprobe -version | head -1
```

Full transcription test is optional at install time — it burns Scribe credits. Wait until the user hands you their first clip.

### 6. Hand off

Tell the user, in one short message:

- That the skill is installed and how (plugin or symlink).
- That they should `cd` into their footage folder and start their agent there (e.g. `claude`).
- That a good first message is: *"edit these into a launch video"* or *"inventory these takes and propose a strategy."*
- That all outputs land in `<videos_dir>/edit/` — the skill folder stays clean.

## Keeping the skill current

- Plugin: `/plugin marketplace update video-use`, then restart Claude Code.
- Symlink: `cd ~/Developer/video-use && git pull --ff-only`.
- If `pyproject.toml` changed deps, re-run the pip install from step 2.

## Cold-start reminders

- Symlink the **`skills/video-use` folder**, not just `SKILL.md`. The helpers need to sit next to it.
- If a `.env` exists but the key is empty, treat it the same as missing.
- `ffmpeg` from static builds works fine. Any modern (≥ 4.x) build is enough.
- `yt-dlp` is optional. Install lazily the first time a user asks to pull from a URL.
- Node.js/npm are only needed for HyperFrames or Remotion slots. HyperFrames currently requires Node.js 22+.
- HyperFrames, Remotion, and Manim are optional animation engines. Pick the engine per animation slot in `SKILL.md`; install inside the slot directory, never globally during setup.
- Never run transcription as part of install verification unless the user explicitly asks — Scribe costs real money.
