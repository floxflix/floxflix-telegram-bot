# floxflix-telegram-bot

Telegram bot that receives movies and episodes, stores them tidily on disk under
`uploads/<category>/<title>/` and registers them in the FloxFlix catalog.

It uses a [local Bot API server](https://github.com/tdlib/telegram-bot-api) to
lift the 20 MB limit of the official API: with the local server files can reach
2 GB and, since the bot shares the volume with the container, they are moved
with a `rename` instead of being downloaded over HTTP.

## Structure

```
cmd/floxflix-bot/      entrypoint: loads config, builds the bot and handles signals
internal/config/       environment variables, validation and path translation
internal/storage/      safe names and atomic writes to disk
internal/bot/          Bot API client, wizard, sessions, downloads and catalog calls
docker-compose.yml     local Bot API server
ROADMAP.md             what is still missing (edit, replace, delete)
```

The flow of an upload: `handlers.go` drives the wizard and keeps its state in
`sessionStore`, `download.go` hands the file to a goroutine, `storage` puts it
in its final place without overwriting anything, and `catalog.go` posts what was
saved to the FloxFlix API.

## Getting started

```sh
cp .env.example .env     # fill in FLOXFLIX_BOT_TOKEN, FLOXFLIX_ALLOWED_USER_IDS,
                         # FLOXFLIX_TELEGRAM_*, FLOXFLIX_API_URL and
                         # TELEGRAM_API_ID / TELEGRAM_API_HASH
make up                  # start the local Bot API server
make run                 # run the bot
```

`make help` lists the remaining targets (`build`, `test`, `cover`, `vet`,
`tidy`, `clean`, `down`, `logs`).

## Usage

| Command   | What it does                                            |
| --------- | ------------------------------------------------------- |
| `/start`  | choose **Movie**, **Tv Show** or **New Ep.**            |
| `/cancel` | forget the upload in progress                           |
| `/skip`   | leave the optional field being asked empty              |
| `/help`   | show the help                                           |

The three flows ask for one field at a time:

- **Movie** — title, description, year, genres, rating, backdrop, then the file.
- **Tv Show** — the same, then season, episode number, episode title, the
  skip-intro label and its start/end, and when the next-episode button should
  appear, then the file. The session stays on the file step afterwards, so the
  rest of the season can be sent in a row and the episode number increments.
- **New Ep.** — the show title, a numbered list of matching titles already in
  the catalog, then the episode fields and the file.

`/skip` only works on the optional fields: description, skip-intro label and
times, next-episode time. Everything else is required and validated — a bad
year, genre number, rating or timecode is rejected and the same question is
asked again. The backdrop takes an uploaded image or a URL.

Files are named after the title, lowercased with spaces, dots and special
characters replaced by `_` — `blade_runner_(1982).mkv` — and episodes go under
`tv-shows/<title>/S01/E02/` with the show title as the file name. A name that
already exists becomes `… (1)`, `… (2)` instead of overwriting the earlier file.
An uploaded backdrop is saved as `background.<ext>` in the title folder.

## Publishing to FloxFlix

With `FLOXFLIX_API_URL` set, every saved file is registered through
`POST /v1/api/bot/videos` (or `…/{uuid}/new-episode` for **New Ep.**), with paths
relative to `FLOXFLIX_DOWNLOAD_DIR`. The API writes the catalog row, queues the
transcode and moves the item from `In-Progress` to `Ready`.

The API must see the same directory the bot writes to — in Kubernetes both
deployments mount `videos-pvc` at `/app/videos`, and the API is configured with
`UPLOAD_FOLDER=videos/`. If `FLOXFLIX_API_URL` is empty the files are still
saved, but nothing is published and only a log line says so.

Those `/v1/api/bot/*` routes are unauthenticated today, so the bot and the API
belong on a private network.

## Configuration

Everything comes from environment variables; see `.env.example` for the full
list. Bot variables are prefixed `FLOXFLIX_`; the `TELEGRAM_API_ID` /
`TELEGRAM_API_HASH` pair in the same file is read by the local Bot API
container, not by the bot.

Required:

- `FLOXFLIX_BOT_TOKEN` — from BotFather.
- `FLOXFLIX_ALLOWED_USER_IDS` — the authorized numeric IDs, comma-separated. If
  it is left empty the bot **does not start**, and a malformed ID also aborts
  startup: a list that cannot be parsed would let anyone in.
- `FLOXFLIX_TELEGRAM_API_ENDPOINT` / `FLOXFLIX_TELEGRAM_FILE_ENDPOINT` — both
  must contain exactly two `%s` (token and path). The Dockerfile supplies the
  in-cluster values.
- `TELEGRAM_API_ID` / `TELEGRAM_API_HASH` — for the local Bot API server.

Optional, with defaults: `FLOXFLIX_DOWNLOAD_DIR` (`./uploads`),
`FLOXFLIX_TELEGRAM_DATA_DIR` (`./telegram-data`),
`FLOXFLIX_TELEGRAM_SERVER_WORK_DIR` (`/var/lib/telegram-bot-api`),
`FLOXFLIX_DEBUG` (`false`), `FLOXFLIX_API_URL` (empty — no publishing).

## Security

- The token lives only in gitignored files: `.env` locally, a Secret manifest in
  Kubernetes. It is never in the code.
- The local Bot API server listens on `127.0.0.1` only, because it exposes the
  token in file URLs.
- `FLOXFLIX_DEBUG=true` prints every Bot API request, token included: do not
  leave it on.
- Titles and file names come from the user and are sanitized before touching
  the disk (`internal/storage`, `internal/bot/download.go`), so they cannot
  escape `uploads/`.
- The bot is an ingestion tool: anyone who can reach `/v1/api/bot/*` can write to
  the catalog. Keep both on a private network until the routes require a shared
  secret.

## Tests

```sh
make test
```

They cover what can break silently: the wizard's step/field mapping, the JSON
sent to the API, name sanitizing (traversal, truncation), path translation from
container to host, destination deduplication and the file move.

## Roadmap

The create path works end to end. Editing, replacing and deleting items already
in the catalog does not — see [`ROADMAP.md`](ROADMAP.md).
