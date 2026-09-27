# telegram-bot-api

A self-hosted [Telegram Bot API server](https://github.com/tdlib/telegram-bot-api) for the homelab, run with Docker Compose. It uses the community image [`aiogram/telegram-bot-api`](https://hub.docker.com/r/aiogram/telegram-bot-api) in `--local` mode.

## Why

Through the public cloud Bot API (`api.telegram.org`), bots can upload files only up to 50 MB. With a local server in `--local` mode the limit is **2000 MB**.

This is shared infrastructure: several independent bot projects on the same Docker host use this one server. The first is immich-tg. Bots reach the server over a pre-created external Docker network, `telegram-bot-api`, as **`http://telegram-bot-api:8081`**. No ports are published.

## One-time setup

1. **Get an `api_id` and `api_hash`.** Log in at [my.telegram.org](https://my.telegram.org), open *API development tools* and create an application. One pair serves every bot that uses this server.

2. **Create the shared network.** Do this once per Docker host:

   ```sh
   docker network create telegram-bot-api
   ```

3. **Fill in `.env`:**

   ```sh
   cp .env.example .env
   # set TELEGRAM_API_ID and TELEGRAM_API_HASH
   ```

   `.env` is git-ignored.

4. **Start the server:**

   ```sh
   docker compose up -d
   ```

   The container restarts automatically (`unless-stopped`). Its data lives in the named volume `telegram-bot-api_data`, mounted at `/var/lib/telegram-bot-api`.

## Adding a bot

1. **Log the bot out of the cloud Bot API, once,** before its first use on this server:

   ```sh
   curl -s "https://api.telegram.org/bot<TOKEN>/logOut"
   # {"ok":true,"result":true}
   ```

   From then on the bot works through this server. Telegram doesn't let a logged-out bot log back in to the cloud API for 10 minutes. To move a bot back to the cloud later, call `logOut` against this server instead.

2. **Point the bot at `http://telegram-bot-api:8081`** instead of `https://api.telegram.org`. The URL format is unchanged: `http://telegram-bot-api:8081/bot<TOKEN>/<method>`.

3. **Join the bot's stack to the shared network** in its `docker-compose.yml`:

   ```yaml
   services:
     my-bot:
       # ...
       networks:
         - telegram-bot-api

   networks:
     telegram-bot-api:
       external: true
   ```

   The bot's service can't be on the stack's default network any more once `networks:` is listed. If it also needs other services in its own stack, add `default` to its `networks:` too.

## Checking it works

From a throwaway container on the network:

```sh
docker run --rm --network telegram-bot-api curlimages/curl -s \
  "http://telegram-bot-api:8081/bot<TOKEN>/getMe"
# {"ok":true,"result":{"id":...,"is_bot":true,...}}
```

A made-up token returns `{"ok":false,"error_code":401,"description":"Unauthorized"}`. That still shows the server is up and reachable.

## Local-mode caveats for bots

- **`getFile` returns a path, not a download URL.** In `--local` mode, `file_path` is an absolute path on the server's disk, such as `/var/lib/telegram-bot-api/<TOKEN>/documents/file_0.pdf`. The server has already downloaded the file itself. `http://telegram-bot-api:8081/file/bot<TOKEN>/...` does not serve it.
- **Downloaded files are kept forever.** They stay on the data volume under `/var/lib/telegram-bot-api/<TOKEN>/`, and nothing cleans them up automatically.

A bot that downloads user files therefore needs some way to read that volume. One option is to mount it into the bot at the same path, so that the paths match:

```yaml
services:
  my-bot:
    volumes:
      - telegram-bot-api_data:/var/lib/telegram-bot-api:ro

volumes:
  telegram-bot-api_data:
    external: true
```

The other option is a small file server that exposes the volume on the shared network. Either way, the bot also needs some cleanup. Files are owned by uid/gid 101, so deleting them needs a writable mount and that uid (or root).

Bots that only *upload* files, like immich-tg, need none of this. They send files as ordinary HTTP multipart uploads.

## Operations

### Disk space

- **Temporary uploads.** The server writes each uploaded file to disk in full before sending it to Telegram, so each concurrent upload needs up to about **2 GB** free. The temp dir is `/var/lib/telegram-bot-api/temp`, on the data volume and not in the container's writable layer (it is set by `TELEGRAM_TEMP_DIR` in `docker-compose.yml`). Files are removed after the upload. Leftovers from a crash can be deleted while the server is stopped.
- **Downloaded files** pile up under each bot's directory (see above).

Check usage:

```sh
docker run --rm -v telegram-bot-api_data:/data alpine du -sh /data/*
```

### Security

The server has **no authentication** besides bot tokens: anyone who can reach port 8081 can use any token they know. It also holds each bot's session state on the data volume. **Never publish its ports.** Bots on the same Docker host reach it through the `telegram-bot-api` network, which is the only way in.

### Logs

```sh
docker compose logs -f telegram-bot-api
```

Logs go to stdout and are rotated by Docker (3 × 10 MB, set under `logging:` in `docker-compose.yml`). For more detail, add `TELEGRAM_VERBOSITY` (for example `3`) to `environment:`.

### Upgrading

The image is pinned to a version tag (`aiogram/telegram-bot-api:10.3`). To upgrade:

1. Find the new tag on [Docker Hub](https://hub.docker.com/r/aiogram/telegram-bot-api/tags). It tracks the Bot API version.
2. Change `image:` in `docker-compose.yml`.
3. Run:

   ```sh
   docker compose pull && docker compose up -d
   ```

The data volume keeps each bot's state across upgrades.

### Stats endpoint (optional)

The server can serve a plain-text stats page on port 8082. To turn it on, uncomment `TELEGRAM_STAT: "1"` in `docker-compose.yml`, then run `docker compose up -d`. Like 8081, keep this port unpublished and read the page from the network:

```sh
docker run --rm --network telegram-bot-api curlimages/curl -s http://telegram-bot-api:8082/
```
