<p align="center">
  <img src="https://github.com/user-attachments/assets/37be4bfc-60e5-496f-bd02-abe0adf3979c" width="300">
</p>

# somtum.fun

somtum.fun is an [osu!](https://osu.ppy.sh) private server from Thailand. Services communicate via a shared MySQL database and Redis cache, all hosted on OVH and fronted by Cloudflare.

**Website:** [somtum.fun](https://somtum.fun)

## Service Architecture

```mermaid
flowchart TB
    subgraph Clients["External Clients"]
        osu(["osu! Game Client"])
        web(["Web Browser"])
        dclient(["Discord"])
    end

    subgraph CF["Cloudflare (DNS · CDN · DDoS · SSL)"]
        cf{{"Proxy & Cache"}}
    end

    subgraph OVH["OVH VPS — Podman Containers"]
        ng{{"Caddy — Reverse Proxy & Rate Limiting"}}

        subgraph Public["Public Services"]
            bancho["bancho.py<br/>(Python/FastAPI) :10000"]
            gukarkka["gukarkka<br/>(Next.js 14) :3000"]
            payments["payments-service<br/>(Python/FastAPI) :8001"]
        end

        subgraph Internal["Internal Services"]
            circlecore["Circlecore-somtum<br/>(Python/FastAPI) :8555"]
        end

        subgraph Bots["Bots"]
            bot["Discord-Bot-Somtum<br/>(discord.py)"]
        end

        subgraph Data["Data Stores"]
            mysql[("MySQL 9.4<br/>Shared Database")]
            redis[("Redis 8.2<br/>Cache & Sessions")]
            files[("Static Files<br/>Avatars & Replays")]
        end

        subgraph Backup["Backups"]
            timer["somtum-backup.timer<br/>(systemd · hourly)"]
        end
    end

    subgraph Offsite["Offsite — Thailand"]
        miyabi[("Syncthing Replica<br/>Hourly Snapshots")]
    end

    subgraph ExternalAPIs["External APIs"]
        osuapi["osu! API v1/v2"]
        discordwh["Discord Webhooks"]
        truemoney["TrueMoney / PromptPay"]
        stripe["Stripe"]
    end

    %% Client connections
    osu -->|"c.somtum.fun"| cf
    web --> cf --> ng
    dclient <--> bot

    %% Caddy routing
    ng -->|"somtum.fun"| gukarkka
    ng -->|"session / api .somtum.fun"| bancho
    ng -->|"a.somtum.fun"| files

    %% Inter-service HTTP
    gukarkka -->|"player & session data"| bancho
    gukarkka -->|"donations"| payments
    bancho -->|"replay analysis"| circlecore
    payments -->|"grant_donator"| bancho
    bot -->|"player actions"| bancho
    bot -->|"replay checks"| circlecore

    %% External APIs
    bancho --> osuapi
    bancho --> discordwh
    circlecore --> discordwh
    payments --> truemoney
    payments --> stripe
    bot --> discordwh

    %% Data stores
    bancho --> mysql & redis
    gukarkka --> redis
    payments & bot --> mysql

    %% Backups (side branch off the data stores)
    mysql & redis & files -->|"hourly dump"| timer
    timer -->|"Syncthing"| miyabi

    %% Invisible links to keep External APIs as the bottom layer
    miyabi ~~~ osuapi & discordwh & truemoney & stripe

    %% Styling
    classDef client fill:#e1f5fe,stroke:#01579b,color:#01579b
    classDef proxy fill:#fff3e0,stroke:#e65100,color:#e65100
    classDef public fill:#e8f5e9,stroke:#2e7d32,color:#1b5e20
    classDef internal fill:#f3e5f5,stroke:#7b1fa2,color:#4a148c
    classDef data fill:#fce4ec,stroke:#c2185b,color:#880e4f
    classDef external fill:#eceff1,stroke:#546e7a,color:#37474f
    classDef bot fill:#e3f2fd,stroke:#1565c0,color:#1565c0
    classDef backup fill:#fff8e1,stroke:#f9a825,color:#f57f17

    class osu,web,dclient client
    class cf,ng proxy
    class bancho,gukarkka,payments public
    class circlecore internal
    class mysql,redis,files data
    class osuapi,discordwh,truemoney,stripe external
    class bot bot
    class timer,miyabi backup
```

## Services

| Service | Language | Purpose |
|---|---|---|
| **bancho.py** | Python/FastAPI | Core game server (osu! protocol, scoring, multiplayer) |
| **gukarkka** | Next.js 14/TypeScript | Web frontend — profiles, leaderboards, beatmap browser & uploads |
| **Discord-Bot-Somtum** | Python/discord.py | Community bot — stats, moderation, beatmap workflow |
| **Circlecore-somtum** | Python/FastAPI | Anti-cheat — replay analysis & similarity detection |
| **payments-service** | Python/FastAPI | Donation processing (TrueMoney, PromptPay, Stripe) |

## Communication Patterns

- **Shared Database**: bancho.py, Discord bot, and payments-service all connect to the same MySQL instance
- **Redis**: bancho.py for sessions and cache; gukarkka for caching
- **HTTP REST**: All inter-service communication

### Inter-Service HTTP Calls

| Caller | Target | Purpose |
|---|---|---|
| gukarkka | bancho.py | Player sessions, scores, leaderboards |
| gukarkka | payments-service | Donation redeem, PromptPay, Stripe checkout, admin approve/reject |
| bancho.py | Circlecore-somtum | Queue replay for analysis after score submit |
| payments-service | bancho.py | `POST /internal/grant_donator` — sync donator status |
| Discord-Bot-Somtum | bancho.py | Player management (restrict, whitelist, etc.) |
| Discord-Bot-Somtum | Circlecore-somtum | On-demand replay checks from staff commands |

### Score Submission Flow

1. Client `POST`s encrypted score data to bancho.py via `c.somtum.fun`
2. bancho.py decrypts and validates the score
3. Score and stats written to MySQL
4. Replay queued to Circlecore-somtum for async analysis
5. Circlecore fires Discord webhook alert if suspicious
6. First-place announcements sent via Discord webhook

### Beatmap Upload Flow

somtum.fun lets players host their own beatmaps (a custom feature not in vanilla bancho.py). Uploads are supported both **in-game** (via the standard `osu-osz2-bmsubmit` endpoints on `osu.somtum.fun`) and through the **web frontend**:

1. Player uploads an `.osz` to bancho.py
2. bancho.py validates the session, parses the zip, and SHA-256-dedups each difficulty against existing maps
3. Each diff is cross-checked against the osu! API v1 to reject maps already submitted to osu!
4. A fresh somtum set ID is allocated; `BeatmapID`/`BeatmapSetID` lines are rewritten and the map is persisted to disk
5. `mapsets`/`maps` rows are inserted as `somtum_only` with status `Pending`, awaiting nomination
6. Upload is audited via Discord webhook

## Tech Stack

| Category | Technologies |
|---|---|
| **Languages** | Python, TypeScript, SQL |
| **Web Frameworks** | FastAPI, Next.js 14, React 18 |
| **Databases** | MySQL 9.4, Redis 8.2 |
| **Styling** | Tailwind CSS, shadcn/ui |
| **Discord** | discord.py 2.4+ |
| **Anti-cheat** | circleguard 5.4.3 |
| **Containers** | Podman / Podman Compose |
| **Reverse Proxy** | Caddy |
| **External APIs** | osu! API v1/v2, Discord, TrueMoney, PromptPay, Stripe |

## Infrastructure

| Provider | Purpose |
|---|---|
| **OVH** | VPS — hosts all services |
| **Cloudflare** | DNS, CDN, DDoS protection, SSL certificates |
| **Caddy** | Reverse proxy, automatic HTTPS, rate limiting |
| **Podman** | Container runtime & orchestration |

### Backups & Disaster Recovery

A `somtum-backup.timer` systemd user timer fires **hourly** on the OVH VPS, running `scripts/backup.sh` to snapshot:

- `/var/www` (avatars, banners, replays, hosted beatmaps) → `www.tar.gz`
- MySQL via `mysqldump --single-transaction` → gzipped SQL dump
- Redis via `SAVE` + a copy of `dump.rdb`

The last **24 hourly snapshots** are retained and older ones are pruned automatically. The entire `server-somtum` directory — backups included — is continuously replicated by **Syncthing** from the production VPS to an offsite machine in Thailand, giving geographically separated, near-real-time copies of all production data independent of OVH and Cloudflare.

### Domain Routing

| Domain | Service | Notes |
|---|---|---|
| `somtum.fun` | gukarkka :3000 | Web frontend |
| `c` / `ce` / `c4` / `osu` / `b` / `api` / `session` `.somtum.fun` | bancho.py :10000 | osu! client + API endpoints (rate-limited on c/ce/c4/osu) |
| `payment.somtum.fun` | payments-service :8001 | Donation processing |
| `a.somtum.fun` | Static files | Avatars |
| `assets.somtum.fun` / `assets1.somtum.fun` | Static files | Banners, clan assets, replays |

All subdomains sit behind the Cloudflare proxy; Caddy handles routing and automatic HTTPS at the OVH VPS.
