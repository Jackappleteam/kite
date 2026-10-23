# 🪁 Kite

[![Release](https://github.com/merlinfuchs/kite/actions/workflows/release.yaml/badge.svg)](https://github.com/merlinfuchs/kite/releases)
[![Docker image](https://github.com/merlinfuchs/kite/actions/workflows/docker-push.yaml/badge.svg)](https://hub.docker.com/r/merlintor/kite)

[![Release](https://img.shields.io/github/v/release/merlinfuchs/kite)](https://github.com/merlinfuchs/kite/releases/latest)
[![MIT License](https://img.shields.io/github/license/merlinfuchs/kite)](LICENSE)
[![Discord Server](https://img.shields.io/discord/845800518458540083)](https://discord.gg/rNd9jWHnXh)

Make your own Discord Bot with Kite for free without a single line of code. With support for slash commands, buttons, select menus, events, and more.

![Flow Example](./example-flow.png)

Commands and event listeners are built as flows of blocks in a visual editor.

- Blocks for messages, members, roles, channels, threads, polls and voice
- Conditions, loops, error handling and stored variables for more complex logic
- A Discord API Request block for anything else in the Discord API
- HTTP Request and AI blocks to connect your bot to other services
- Event listeners that run on Discord events, on a schedule or when a webhook is called
- A flow AI that builds flows from a description
- Templates for moderation, modmail, a welcomer and more

Kite is actively developed and new blocks and integrations are added regularly. If something you need is missing, open an issue or ask on [Discord](https://discord.gg/rNd9jWHnXh).

## Repository layout

| Directory | What it is |
| --- | --- |
| [`kite-service`](./kite-service) | Go backend: API, flow engine, Discord gateway and database migrations |
| [`kite-web`](./kite-web) | Next.js dashboard and flow editor, can be embedded into the `kite-service` binary |
| [`kite-docs`](./kite-docs) | Docusaurus documentation site ([docs.kite.onl](https://docs.kite.onl)) |
| [`kite-support`](./kite-support) | Discord support bot that answers questions from the docs with `/ask` |
| [`deploy`](./deploy) | Scripts used to deploy the hosted version of Kite |
| [`design`](./design) | Design notes for upcoming features |

## Contributing

Contributions are welcome. Read [CONTRIBUTING.md](./CONTRIBUTING.md) before opening a PR, and [AGENTS.md](./AGENTS.md) for the checks CI runs and the checklists for adding blocks and event listeners.

## Self Hosting

> [!WARNING]
> Self-hosting is not officially supported. These instructions are provided as-is for people who are comfortable running Docker, Postgres and their own Discord application. We don't answer self-hosting questions in issues or on Discord, and they may be closed without a reply. Bug reports that also affect the hosted version of Kite are welcome.

The easiest way to self host Kite is a single binary that contains both the backend and the frontend. Prebuilt binaries are available on the [releases page](https://github.com/merlinfuchs/kite/releases/latest) and as the [`merlintor/kite`](https://hub.docker.com/r/merlintor/kite) Docker image.

Kite also needs a [Postgres](https://www.postgresql.org/) database and S3-compatible storage (like MinIO) for assets, so using `docker compose` is recommended.

### Create a Discord application

1. Create an application in the [Discord Developer Portal](https://discord.com/developers/applications).
2. Copy the client ID and client secret from the OAuth2 page.
3. Add `<api public base url>/v1/auth/login/callback` as a redirect, for example `http://localhost:8080/v1/auth/login/callback`.

This application is only used to log in to the dashboard. The bots themselves are added in the dashboard with their own tokens.

### Configure the server

Create a file called `kite.toml` in the directory you run Kite from:

```toml
[discord]
client_id = "..."     # Discord client ID used for OAuth2
client_secret = "..." # Discord client secret used for OAuth2

[encryption]
token_encryption_key = "..." # Hex encoded 32 byte AES key for encrypting bot tokens
```

Generate an encryption key with `openssl rand -hex 32`. Keep it safe, bot tokens saved with one key can't be decrypted with another.

Every value can also be set with an environment variable: `KITE_` followed by the section and key, separated by a double underscore. For example `KITE_DISCORD__CLIENT_ID` sets `client_id` in the `[discord]` section.

All other values have defaults, see [`default.toml`](./kite-service/internal/config/default.toml). Some optional settings worth knowing about:

| Setting | What it does |
| --- | --- |
| `api.public_base_url` / `app.public_base_url` | The URLs the API and dashboard are reachable at. Set both to your domain when not running on localhost |
| `api.secure_cookies` | Set to `true` when serving Kite over HTTPS |
| `openai.api_key` | Enables the AI blocks and the flow AI in the editor |
| `openrouter.api_key` | Enables AI blocks that make decisions, like AI Choice |
| `discord.proxy_url` | Sends Discord API requests through a proxy like [nirn-proxy](https://github.com/germanoeich/nirn-proxy) |
| `engine.http_proxy_url` | Sends requests from the HTTP Request block through a proxy |
| `user_limits.max_apps_per_user` | How many apps each user can create |
| `billing.plans` | Plans and their limits (commands, event listeners, credits, ...). Users without a subscription get the plan marked `default` |

### Using Docker (docker compose)

Install Docker and create a `docker-compose.yaml` file next to your `kite.toml`:

```yaml
services:
  postgres:
    image: postgres
    restart: always
    volumes:
      - kite-local-postgres:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: postgres
      POSTGRES_DB: kite
      PGUSER: postgres
      PGDATA: /var/lib/postgresql/data/pgdata
      POSTGRES_HOST_AUTH_METHOD: trust
    healthcheck:
      test: ["CMD", "pg_isready"]
      interval: 3s
      timeout: 30s
      retries: 3

  minio:
    image: pgsty/silo
    restart: always
    command: server --console-address ":9001" /data
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: kite
      MINIO_ROOT_PASSWORD: 1234567890
    volumes:
      - kite-local-minio:/data

  kite:
    image: merlintor/kite:latest
    restart: always
    ports:
      - "8080:8080"
    environment:
      - KITE_API__HOST=0.0.0.0
      - KITE_DATABASE__POSTGRES__HOST=postgres
      - KITE_DATABASE__POSTGRES__USER=postgres
      - KITE_DATABASE__POSTGRES__DB_NAME=kite
      - KITE_DATABASE__S3__ENDPOINT=minio:9000
      - KITE_DATABASE__S3__ACCESS_KEY_ID=kite
      - KITE_DATABASE__S3__SECRET_ACCESS_KEY=1234567890
      - KITE_DATABASE__S3__SECURE=false
      - KITE_APP__PUBLIC_BASE_URL=http://localhost:8080
      - KITE_API__PUBLIC_BASE_URL=http://localhost:8080
    volumes:
      - ./kite.toml:/root/kite.toml
    depends_on:
      postgres:
        condition: service_healthy
      minio:
        condition: service_started

volumes:
  kite-local-postgres:
  kite-local-minio:
```

Start it with `docker compose up -d`. The `kite.toml` file is mounted into the container, and database migrations run automatically on every start. Don't configure Postgres or S3 in `kite.toml`, the environment variables above point Kite at the containers.

Kite should now be accessible in your browser at [localhost:8080](http://localhost:8080).

> [!IMPORTANT]
> This example uses Postgres without a password and the default MinIO credentials. Only expose port 8080 publicly, and change the MinIO credentials (in both services) before running it on a server. When serving Kite on a domain, put it behind a reverse proxy with HTTPS and set both public base URLs to that domain.

To update, run `docker compose pull` followed by `docker compose up -d`.

## Local development

The [`docker-compose.yaml`](./docker-compose.yaml) in this repository only starts the services Kite depends on (Postgres, MinIO and nirn-proxy), so you can run the backend and frontend from source:

```shell
docker compose up -d
```

Set `PG_HOST_PORT` if port 5432 is already in use. To route Discord requests through the local nirn-proxy, set `discord.proxy_url = "http://localhost:8888"` in `kite.toml`.

### Building the website (kite-web)

Install Node.js 22 from [nodejs.org](https://nodejs.org/en/download/) and enable pnpm with `corepack enable`.

```shell
cd kite-web

# Install dependencies
pnpm install

# Start the development server (optional)
pnpm run dev

# Build for embedded use in kite-service (recommended)
OUTPUT=export pnpm run build   # PowerShell: $env:OUTPUT="export"; pnpm run build

# Build for standalone use
pnpm run build
```

The development server runs on [localhost:3000](http://localhost:3000), which matches the default `app.public_base_url`.

### Building the docs (kite-docs)

```shell
cd kite-docs

# Install dependencies
pnpm install

# Start the development server (optional)
pnpm run start

# Build for production use
pnpm run build
```

### Building the server (kite-service)

Install Go 1.26 or newer from [go.dev](https://go.dev/doc/install).

```shell
cd kite-service

# Configure the server (see "Configure the server" above), kite.toml goes in this directory

# Run database migrations
go run main.go database migrate postgres up

# Start the development server (optional)
go run main.go server start

# Build with the kite-web files embedded in the binary (build the website first)
go build --tags "embedweb"

# Build without the frontend files (you need to serve them yourself)
go build
```

### Building the Docker image

The [`Dockerfile`](./Dockerfile) builds the website and the server into a single image:

```shell
docker build -t kite .
```

## License

Kite is licensed under the [MIT License](./LICENSE).
