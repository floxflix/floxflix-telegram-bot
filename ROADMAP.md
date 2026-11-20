# Roadmap — floxflix-telegram-bot

**Goal:** send a movie, a TV show or a new episode to the bot and have it land in the FloxFlix catalog,
transcoded by the API, without touching a shell.

Status: **the create path works end to end.** The bot can create a movie, create a show with its first
episode, and append an episode to an existing show picked by title. It cannot change anything it has
already published. Owner: @elvus.

---

## 1. What works today

| Flow | Entry point | What lands |
| --- | --- | --- |
| Create a movie | `/start` → **Movie** | `uploads/movies/<title>/<file>` + `POST /v1/api/bot/videos` |
| Create a show with its first episode | `/start` → **Tv Show** | `uploads/tv-shows/<title>/S01/E01/<file>` + the same `POST`, with one season and one episode |
| Append an episode to a show | `/start` → **New Ep.** → title list | `POST /v1/api/bot/videos/{uuid}/new-episode` (the API creates the season if it is missing, and answers `409` if the episode already exists) |
| Backdrop | Wizard step, upload or URL | `background.<ext>` next to the media, or the URL as given |

Every flow ends the same way: the API writes the catalog row, the conversion queue transcodes the file,
and the row moves from `In-Progress` to `Ready`. Several files can follow one another for the same
title, because the session stays open on the file step.

## 2. TODO — the bot is create-only: no edit, replace or delete

Everything a user gets wrong on the way in can only be corrected from the UI or by hand in Mongo.

| Wanted | Today | Missing piece |
| --- | --- | --- |
| Fix a movie/show field (title, description, year, genres, rating, backdrop) | impossible | the API has no show-level update route — `floxflix/internal/handlers/videos.go:174-181` is episode-level only |
| Fix an episode's title, skip-intro or next-episode times | impossible from the chat | `PUT /v1/api/videos/{uuid}/season/{season}/episode/{episode}` exists but is JWT-only; no `/v1/api/bot/*` twin |
| Re-send an episode with a better file | the API answers `409 Season X episode Y already exists` — *after* the file was already written as `<title> (1).mkv` | an explicit replace flow: look the episode up before saving, or accept the file and swap `file_path` + re-queue |
| Move an episode to another season/number | impossible | the same episode PATCH, with the array filters it already uses |
| Delete an episode or a whole item | impossible | the API has no `DELETE` for `catalog` anywhere |
| Retry a failed transcode | impossible | re-queue by uuid (`queue/start` only kicks the worker) |

Two consequences worth writing down rather than discovering later:

- The skip-intro bug tracked in the floxflix roadmap (§3.3) is already in published rows. An edit path is
  how those get fixed without a Mongo script.
- Re-sending a file currently leaves an orphan: `saveVideo` writes first and only then learns the API
  rejected the episode (`internal/bot/handlers.go:246-289`).

**Minimum useful slice:** the episode PATCH twin plus a show-level PATCH. Replace and delete can follow;
they are the ones that need a UI decision (does deleting an item delete its files?).

## 3. Prerequisite: the bot routes are unauthenticated

Every row in §2 is a *new write route* on `/v1/api/bot/*`, which is currently reachable without
authentication from the public ingress (`api.floxflix.io`). Adding edit and delete on top of that turns a
data-integrity hole into a destructive one.

So the shared-secret middleware from the floxflix roadmap (§3.1) lands first, and each new route is
registered inside the protected group from day one — not "later, once it works".

## 4. Other known issues (bot side)

Details and file references live in the floxflix roadmap, §3; this is the bot's own checklist:

- [ ] Skip-intro and next-episode times are all published from the start time; two tests are red
      (`go test ./...`) — floxflix §3.3.
- [ ] `.env.example` and README document bare `BOT_TOKEN`/`ALLOWED_USER_IDS`, while `config.go` reads
      `FLOXFLIX_*` — copying the example fails at startup. floxflix §3.4.
- [ ] `Dockerfile` ends with `USER floxflix` but never creates the account; only the k8s
      `runAsUser: 101` hides it. floxflix §3.5.
- [ ] No duplicate protection: `update_id` is ignored and `FileUniqueID` only names fallbacks, so a
      redelivered update ingests twice. floxflix §3.6.
- [ ] No size or free-space preflight before a multi-GB download. floxflix §3.7.
- [ ] The bot token is plaintext in `.env` and `k8s/secret-telegram-bot.yaml` (both gitignored, never
      committed). Move to a managed secret. floxflix §3.2.
- [ ] README drift: "everything but the title can be skipped" (four steps are not skippable), and the
      episode filename described there is not the one `saveMedia` writes. floxflix §3.10.

## 5. Not in this repo

Progress messages, a persisted conversion queue, `/download <url>`, webhook mode and the MTProto
alternative all need the API to change. They stay in
[`docs/TELEGRAM_INGEST_ROADMAP.md`](https://github.com/floxflix/floxflix/blob/main/docs/TELEGRAM_INGEST_ROADMAP.md).

### References

- FloxFlix roadmap: [`TELEGRAM_INGEST_ROADMAP.md`](https://github.com/floxflix/floxflix/blob/main/docs/TELEGRAM_INGEST_ROADMAP.md)
- [Bot API — getFile](https://core.telegram.org/bots/api#getfile)
- [tdlib/telegram-bot-api](https://github.com/tdlib/telegram-bot-api) — local server, `--local`
