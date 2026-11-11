# Provenance and Verification Record: music-assistant-ha

## Clean-room attestation

This skill was authored clean-room. **No pre-existing Music Assistant skill, community
integration, or skill-registry/hub package was searched, cloned, or read.** Sources consulted:

1. A live Music Assistant server (2.10.9, API schema 65): `/api-docs`, `/api-docs/openapi.json`,
   `/info`, and read-only calls (`players/all`, `music/search`, `music/recently_played`,
   `player_queues/get_queue_items`, `players/tts_engines`).
2. https://www.music-assistant.io/integration/announcements (official docs)
3. https://github.com/music-assistant/client (official client README)
4. https://github.com/music-assistant/server (controller directory listing)
5. Installed `music-assistant-client` 1.8.0 source (PyPI)
6. https://www.music-assistant.io/api (official command reference)

Transport decision: official client over WebSocket for every capability (maintainer preference,
built-in auth/schema guards); raw `POST /api` documented only as fallback.

License: MIT (see `LICENSE` of the source project and this skill's frontmatter).

## Live verification record (2026-11-10)

Unit tests: 36 pass on Python 3.11 (syntax/compile), 3.14.7, and 3.14.9 against
`music-assistant-client` 1.8.0 — no network (mocked client).

Live, read-only: valid auth; bad token exit 3; unreachable host exit 3 after 30 s timeout;
`players`, `status`, `search`, `recent`, `queue-list`, `announce-engines` clean schema-101
refusal (exit 1).

Live, state-changing (owner consent, one target player, machine-readable baseline recorded
via `status --json`, assert-and-restore per step, final baseline re-asserted):
- Verified working: `volume-set`, `mute`/`unmute`, `shuffle`, `repeat`, `queue-add`,
  `queue-list`, `queue-move`, `queue-remove`, `queue-clear`, `play-pause`, `next`, `previous`,
  `stop` (does not clear the queue), `play` (replace mode), `search-play`, `favorite-add`/
  `favorite-remove`, `sync`, `announce --text` and `announce --url` (owner confirmed audible;
  announce interrupted active playback and restored the same track/index).
- **Destructive semantics discovered live:** `transfer` REPLACES the target queue with the
  source queue; an empty source CLEARS a populated target and the server offers no queue
  undo (live api-docs has no history/snapshot/restore command). This is why the warning is a
  first-class pitfall in SKILL.md.
- Device limitation: `group`/`ungroup` are accepted by the API (exit 0) but were never applied
  on the tested CAST-style receiver + Shield pair — provider-side support varies per player.
- CAST-style receivers lag in state reporting (~60 s); assert by polling `--json status`,
  not by exit code.
- Ambiguous multi-player setups refuse to auto-pick (exit 1) by design.

## Upstream project

Full verification table and incident notes: https://github.com/rahlquist/music-assistant-ha
(`docs/VERIFICATION.md`), personal identifiers sanitized.
