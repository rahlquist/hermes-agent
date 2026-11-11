---
name: music-assistant-ha
description: "Music Assistant: playback, volume, queue, search, announce."
version: 0.1.0
author: Richard (rahlquist), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [Smart-Home, Music, Audio, Media, Music-Assistant]
    homepage: https://github.com/rahlquist/music-assistant-ha
    related_skills: []
required_environment_variables:
  - name: MASS_SERVER_URL
    prompt: Music Assistant server base URL
    help: "http://<ma-host>:8095 (a trailing /api and any web-UI path/fragment are stripped)"
    required_for: all operations
  - name: MASS_ACCESS_TOKEN
    prompt: Music Assistant long-lived access token
    help: "Music Assistant Settings -> Security -> create a long-lived token"
    required_for: all operations
  - name: MASS_DEFAULT_PLAYER
    prompt: Default player id (optional)
    help: "Run `mactl players` to list ids. If unset, the player is auto-detected and cached."
    required_for: optional
    optional: true
---

# Music Assistant Skill

Control a Music Assistant (MA) server — playback, volume, queues, library search,
favorites, multi-room grouping/transfer, and announcements — through `scripts/mactl.py`.
The CLI wraps the official `music-assistant-client` library over WebSocket; each invocation
opens one connection, runs one command, and disconnects. Do not hand-roll API calls except
for the documented direct-access fallback.

## When to Use

- "Play/pause/skip my music", "turn it down to 30", "mute it"
- "Play some Miles Davis" / search the MA library and play a result
- Queue management: list, add, remove, move, shuffle, repeat
- "Move the music to the kitchen" (transfer), grouping players
- "Say dinner is ready" / play an audio URL as an announcement
- Don't use for: adding music providers, MA server administration, or players MA does not expose.

## Prerequisites

The helper needs `music-assistant-client` in a dedicated venv (one-time, run via `terminal`):

```bash
PY="$HOME/.cache/hermes-music-assistant-venv/bin/python"
if [ ! -x "$PY" ]; then
  python3 -m venv "$HOME/.cache/hermes-music-assistant-venv" && \
  "$PY" -m pip install -q --disable-pip-version-check music-assistant-client
fi
SCRIPT="$HOME/.hermes/skills/smart-home/music-assistant-ha/scripts/mactl.py"
```

Environment: `MASS_SERVER_URL` (API base URL, e.g. `http://<ma-host>:8095`), `MASS_ACCESS_TOKEN`
(long-lived token from MA Settings → Security), optional `MASS_DEFAULT_PLAYER`. The CLI reads
only environment variables; export them first. The token is never printed.

## How to Run

Canonical invocation through the `terminal` tool (after the setup block above):

```bash
"$PY" "$SCRIPT" players
"$PY" "$SCRIPT" status [--player ID]
```

Global flags, accepted before or after the command:
- `--json`: machine-readable output. Success prints `{"ok": true, ...}`; failure prints
  `{"ok": false, "error": ..., "exit_code": ...}` on stdout.
- `--quiet`: hides confirmation text only; data and errors still print.
- `--player ID`: target player for this call; overrides `MASS_DEFAULT_PLAYER`.

Exit codes: `0` success, `1` operation failed, `2` configuration/usage error,
`3` connection or authentication failure.

## Quick Reference

| Area | Command |
|---|---|
| Status | `players` · `status` |
| Transport | `play-pause` · `pause` · `stop` · `next` · `previous` |
| Volume | `volume-set N` (0–100) · `mute` · `unmute` |
| Queue | `shuffle on\|off` · `repeat off\|all\|one` · `queue-list [--limit N]` · `queue-clear` · `queue-add URI` · `queue-remove ITEM` · `queue-move QUEUE_ITEM_ID SHIFT` · `queue-play-index INDEX` |
| Play | `play URI [--option play\|replace\|next\|replace_next\|add]` |
| Search | `search QUERY [--type track\|album\|artist\|playlist\|radio] [--limit N] [--library-only]` · `search-play QUERY [...]` |
| Library | `recent [--limit N]` · `sync [--media-type T] [--provider ID]` |
| Favorites | `favorite-add URI` · `favorite-remove MEDIA_TYPE ITEM_ID` |
| Multi-room | `group PLAYER TARGET` · `ungroup PLAYER` · `transfer SOURCE TARGET [--auto-play\|--no-auto-play]` |
| Announce | `announce --text "..." [--tts-engine E] [--language L] [--volume N]` · `announce --url URL [--volume N]` · `announce-engines` (needs API schema 101) |

Voice-style mapping: "play Blue in Green" → `search-play "Blue in Green" --type track`;
"turn it down to 30" → `volume-set 30`; "what's playing?" → `status`; "say dinner is ready"
→ `announce --player <id> --text "Dinner is ready"`.

## Pitfalls

- **`transfer` REPLACES the target's queue with the source's.** An empty source CLEARS a
  populated target — live-verified destructive; the server offers no queue undo. Never
  transfer from a player you have not inspected.
- **Player ids are not stable across installs** — discover with `players`, never hardcode.
- **Ambiguous target:** with several available players and no default, commands refuse to
  guess (exit 1). Pass `--player` or set `MASS_DEFAULT_PLAYER`.
- **Web-UI URLs:** `http://<ma-host>:8095/#/settings` and trailing `/api` are accepted and
  sanitized, but the API base URL is what the client needs; availability of players changes
  during a session — re-list rather than cache names.
- **CAST-style receivers lag in state reporting**: a pause may still report `playing` for
  up to a minute. Assert by polling state, not by exit code.
- **`stop` does not clear the queue** on current servers; use `queue-clear` (destructive).
- **Announcements** interrupt active playback and restore it afterward; players are powered
  on for the announcement and powered back off.
- **`announce-engines`** requires API schema 101; older servers return a clean refusal (exit 1).
  The announce command itself works from schema 46/47 upward.
- **Queue ids equal player ids** on typical MA setups; queue commands target the player id.
- **Non-ASCII** track names print as-is.

## Verification

Prove the skill end-to-end without side effects:

```bash
"$PY" "$SCRIPT" players && echo "connection ok"          # exit 0, >=1 player
"$PY" "$SCRIPT" search "test" --type track --limit 1     # exit 0 (or empty result)
```

State-changing verification needs an owner-approved target player, a machine-readable
baseline (`status --json`), an assert-and-restore step per command, and a final baseline
re-assert. See references/PROVENANCE.md for the live-verification record.

## Direct API access (fallback only)

Use this only for an operation the CLI does not cover.

- Endpoint: `POST <MASS_SERVER_URL>/api`, JSON body `{"command": "<name>", "args": {...}}`.
- Header: `Authorization: Bearer <token>`. Read the token from the environment. Never print it.
- The full command list is at https://www.music-assistant.io/api and the live spec at
  `<MASS_SERVER_URL>/api-docs`.
