# Project N.O.M.A.D. — Codebase Map

> Node for Offline Media, Archives, and Data
> v1.29.1 · Apache 2.0 · Crosstalk Solutions, LLC

## What It Is

An offline-first knowledge/education server. The "Command Center" (AdonisJS 6 + Inertia/React) orchestrates Docker containers — Kiwix, Ollama, Kolibri, ProtoMaps, CyberChef, FlatNotes — and provides a unified management UI for installation, configuration, updates, benchmarking, and AI chat.

Targets Debian-based Linux. Installs to `/opt/project-nomad`. No auth by design. Zero telemetry.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│  Browser → http://HOST:8080  (Command Center)           │
├─────────────────────────────────────────────────────────┤
│  AdonisJS 6 (Node 22)                                   │
│    ├── Inertia.js SSR → React pages                     │
│    ├── REST API (/api/*)                                │
│    ├── Transmit (SSE broadcast)                         │
│    ├── BullMQ job queues (via Redis)                    │
│    └── Docker Engine API (via mounted socket)           │
├─────────────────────────────────────────────────────────┤
│  MySQL 8.0        │  Redis 7           │  Dozzle v10    │
│  (data store)     │  (queues/sessions) │  (log viewer)  │
├─────────────────────────────────────────────────────────┤
│  Managed Containers (installed on demand):              │
│    Kiwix :8090  │  Ollama :11434  │  Qdrant :6333/6334 │
│    CyberChef :8100  │  FlatNotes :8200  │  Kolibri :8300│
│    ProtoMaps (static file serving via admin)            │
└─────────────────────────────────────────────────────────┘
```

**Docker socket mount** is the critical integration point — the admin container talks directly to the host's Docker daemon to pull images, create/start/stop containers, and check status.

---

## Repo Structure

```
project-nomad/
├── admin/                    # AdonisJS 6 application (the Command Center)
│   ├── app/
│   │   ├── controllers/      # HTTP request handlers (14 controllers)
│   │   ├── exceptions/       # Error handling
│   │   ├── jobs/             # BullMQ async jobs (downloads, benchmarks, embeddings)
│   │   ├── middleware/       # Request middleware
│   │   ├── models/           # Lucid ORM models (MySQL)
│   │   ├── services/         # Business logic (16 services, ~6900 LOC total)
│   │   ├── utils/            # Shared utilities
│   │   └── validators/       # Request validation (Vine)
│   ├── bin/                  # Entry points (server, console, test)
│   ├── commands/             # Ace CLI commands (benchmark)
│   ├── config/               # AdonisJS config (db, queue, vite, cors, etc.)
│   ├── constants/            # Shared constants (service names, broadcast channels)
│   ├── database/
│   │   ├── migrations/       # 20 migrations (services, benchmarks, chat, collections, kv)
│   │   └── seeders/          # Service definitions (container images, configs, ports)
│   ├── docs/                 # Markdown docs served in-app via /docs
│   ├── inertia/              # Frontend (React + TypeScript)
│   │   ├── app/              # Inertia bootstrap
│   │   ├── components/       # Reusable UI (~40 components)
│   │   ├── context/          # React contexts (modal, notification)
│   │   ├── hooks/            # Custom hooks (~12, SSE subscriptions, polling)
│   │   ├── layouts/          # Page layouts (App, Docs, Maps, Settings)
│   │   ├── lib/              # Client utilities (api, navigation, collections)
│   │   ├── pages/            # Inertia pages (~4900 LOC)
│   │   └── providers/        # React context providers
│   ├── providers/            # AdonisJS providers (map static files)
│   ├── public/               # Static assets (logo, etc.)
│   ├── resources/views/      # Edge templates (Inertia root)
│   ├── start/                # Boot files (routes, env, kernel)
│   ├── tests/                # Test bootstrap
│   ├── types/                # Shared TypeScript types
│   └── util/                 # Server-side utilities (docs, files, zim)
├── collections/              # Curated content manifests (JSON)
│   ├── kiwix-categories.json
│   ├── maps.json
│   └── wikipedia.json
├── install/                  # Deployment scripts (~1300 LOC bash)
│   ├── install_nomad.sh      # Main installer (648 LOC, whiptail TUI)
│   ├── management_compose.yaml  # Docker Compose for core stack
│   ├── start_nomad.sh
│   ├── stop_nomad.sh
│   ├── update_nomad.sh
│   ├── uninstall_nomad.sh
│   ├── entrypoint.sh         # Container entrypoint (migrations, seeding, queue workers)
│   ├── collect_disk_info.sh
│   ├── run_updater_fixes.sh
│   └── sidecar-updater/      # Self-update sidecar container
├── .github/
│   ├── workflows/
│   │   ├── docker.yml        # Manual dispatch → GHCR image build
│   │   └── release.yml       # semantic-release
│   └── ISSUE_TEMPLATE/
├── Dockerfile                # Multi-stage Node 22 build
├── package.json              # Root (version source for semantic-release)
└── .releaserc.json           # semantic-release config
```

---

## Domain Map

### Services (Managed Containers)

The `Service` model + `service_seeder.ts` defines all installable apps. Each has:
- `container_image` — Docker image reference (pinned versions)
- `container_config` — JSON blob (port bindings, volumes, env vars)
- `ui_location` — port number or path for UI access
- `depends_on` — dependency chain (e.g., Ollama depends on Qdrant)
- `installation_status` — state machine: `idle` → `installing` → installed/failed

**Current services:** Kiwix, Ollama, Qdrant (dependency), CyberChef, FlatNotes, Kolibri

The `DockerService` (1021 LOC) handles all Docker Engine API calls — image pulls, container lifecycle, inspections. It talks to the Docker socket directly (no Docker SDK — raw HTTP).

### AI Chat

- `OllamaService` → talks to Ollama API (:11434) for model management and chat
- `RagService` (1212 LOC, largest service) → document upload, PDF/text extraction, chunking, embedding via Ollama, vector storage via Qdrant
- `ChatService` → session/message persistence, streaming responses
- Frontend: `chat/` component suite with sidebar, message bubbles, knowledge base modal

### Content Management

- `ZimService` → Kiwix ZIM file management (list, download, delete)
- `ZimExtractionService` → Wikipedia article extraction from ZIM files
- `CollectionManifestService` → curated content collections from `collections/*.json`
- `CollectionUpdateService` → checks for newer versions of installed content
- `MapService` → ProtoMaps region management (PMTiles download, static serving)

### System

- `SystemService` → hardware info, disk usage, service status aggregation
- `SystemUpdateService` → self-update orchestration via sidecar updater
- `BenchmarkService` (833 LOC) → hardware scoring (CPU, memory, disk, GPU/AI inference), leaderboard submission
- `ContainerRegistryService` → checks GHCR for newer container image tags

### Downloads & Queues

- `DownloadService` → coordinates file downloads with progress tracking
- `QueueService` → BullMQ queue management
- Job types: `run_download_job`, `download_model_job`, `embed_file_job`, `run_benchmark_job`, `check_update_job`, `check_service_updates_job`
- Queues: `downloads`, `model-downloads`, `benchmarks`
- Real-time progress via Transmit SSE on 4 broadcast channels

---

## Key Patterns

### Backend
- **AdonisJS 6** with ES modules, `#imports` aliases
- **Lucid ORM** with MySQL — models use `snake_case` columns
- **Vine validators** for request validation
- **BullMQ** via Redis for async job processing
- **Transmit (SSE)** for real-time broadcast to frontend (not WebSockets)
- **Docker Engine API** via raw HTTP to `/var/run/docker.sock`
- **No authentication** — all routes are open

### Frontend
- **Inertia.js** with React — server-driven navigation, no client router
- **Tailwind CSS** with custom desert theme colors
- **Tabler Icons** for iconography
- **Custom hooks** for SSE subscriptions (`useDownloads`, `useOllamaModelDownloads`, etc.)
- **Markdoc** for rendering in-app documentation

### Infrastructure
- **Docker Compose** for core stack (admin, MySQL, Redis, Dozzle, sidecar-updater)
- **Sidecar updater** container watches for update signals, pulls new admin image, recreates container
- **semantic-release** for versioning (root `package.json` is version source)
- **GHCR** (`ghcr.io/crosstalk-solutions/project-nomad`) for image distribution
- **Manual dispatch** workflow for Docker builds (not auto-triggered on push)

---

## API Surface

All API routes are under `/api/`:

| Prefix | Controller | Purpose |
|--------|-----------|---------|
| `/api/system/*` | SystemController, SettingsController | System info, service CRUD, settings, updates |
| `/api/ollama/*` | OllamaController | Model management, chat streaming |
| `/api/chat/*` | ChatsController | Chat session/message CRUD, suggestions |
| `/api/rag/*` | RagController | Document upload, embedding jobs, file management |
| `/api/zim/*` | ZimController | ZIM file management, Wikipedia selection |
| `/api/maps/*` | MapsController | Map region management, PMTiles download |
| `/api/downloads/*` | DownloadsController | Active download jobs |
| `/api/benchmark/*` | BenchmarkController | Benchmark execution, results, leaderboard |
| `/api/docs/*` | DocsController | Documentation listing |
| `/api/content-updates/*` | CollectionUpdatesController | Content version checking, updates |
| `/api/easy-setup/*` | EasySetupController | First-run wizard data |
| `/api/manifests/*` | EasySetupController | Manifest refresh |
| `/api/health` | inline | Health check |

### Pages (Inertia)

| Route | Page | Layout |
|-------|------|--------|
| `/` → `/home` | `home.tsx` | AppLayout |
| `/about` | `about.tsx` | AppLayout |
| `/chat` | `chat.tsx` | AppLayout |
| `/maps` | `maps.tsx` | MapsLayout |
| `/easy-setup` | `easy-setup/index.tsx` | — |
| `/docs/:slug` | `docs/show.tsx` | DocsLayout |
| `/settings/*` | `settings/*.tsx` | SettingsLayout |

---

## Database Models

| Model | Table | Key Fields |
|-------|-------|------------|
| `Service` | services | service_name, container_image, container_config, installed, installation_status, depends_on, available_update_version |
| `BenchmarkResult` | benchmark_results | scores, hardware info, builder_tag |
| `BenchmarkSetting` | benchmark_settings | benchmark preferences |
| `ChatSession` | chat_sessions | title, timestamps |
| `ChatMessage` | chat_messages | session_id, role, content |
| `CollectionManifest` | collection_manifests | manifest metadata |
| `InstalledResource` | installed_resources | tracks what content is installed |
| `KvStore` | kv_store | generic key-value settings |
| `WikipediaSelection` | wikipedia_selection | user's Wikipedia language/size picks |

---

## Environment Variables

From `start/env.ts`:

| Variable | Required | Purpose |
|----------|----------|---------|
| `NODE_ENV` | yes | development/production/test |
| `PORT` | yes | Server port (8080) |
| `APP_KEY` | yes | AdonisJS encryption key |
| `HOST` | yes | Bind address (0.0.0.0) |
| `URL` | yes | Public URL |
| `DB_HOST/PORT/USER/PASSWORD/DATABASE` | yes | MySQL connection |
| `REDIS_HOST/PORT` | yes | Redis connection |
| `NOMAD_STORAGE_PATH` | optional | Override storage dir (default: `/opt/project-nomad/storage`) |
| `NOMAD_API_URL` | optional | External API URL |
| `INTERNET_STATUS_TEST_URL` | optional | Override connectivity check URL |
| `DB_SSL` | optional | MySQL SSL (default: true) |
| `LOG_LEVEL` | yes | Logger level |

---

## Port Map

| Port | Service | Container Name |
|------|---------|---------------|
| 8080 | Command Center (admin) | nomad_admin |
| 3306 | MySQL | nomad_mysql |
| 6379 | Redis | nomad_redis |
| 9999 | Dozzle (log viewer) | nomad_dozzle |
| 8090 | Kiwix | nomad_kiwix_server |
| 11434 | Ollama | nomad_ollama |
| 6333/6334 | Qdrant | nomad_qdrant |
| 8100 | CyberChef | nomad_cyberchef |
| 8200 | FlatNotes | nomad_flatnotes |
| 8300 | Kolibri | nomad_kolibri |

---

## File Size Hotspots (by LOC)

### Backend Services
| File | LOC | Notes |
|------|-----|-------|
| `rag_service.ts` | 1212 | PDF extraction, chunking, embedding, vector search |
| `docker_service.ts` | 1021 | Raw Docker Engine API client |
| `benchmark_service.ts` | 833 | Hardware scoring, AI inference bench |
| `zim_service.ts` | 558 | ZIM library management |
| `system_service.ts` | 502 | System info aggregation |
| `container_registry_service.ts` | 484 | GHCR version checking |
| `map_service.ts` | 457 | ProtoMaps management |
| `ollama_service.ts` | 402 | Ollama API client |

### Frontend Pages
| File | LOC | Notes |
|------|-----|-------|
| `easy-setup/index.tsx` | 1218 | First-run setup wizard |
| `settings/benchmark.tsx` | 989 | Benchmark UI |
| `settings/update.tsx` | 685 | System update page |
| `settings/models.tsx` | 430 | AI model management |
| `settings/apps.tsx` | 430 | Installed apps management |

---

## Broadcast Channels (SSE)

| Channel | Purpose |
|---------|---------|
| `benchmark-progress` | Benchmark job progress |
| `ollama-model-download` | Model download progress |
| `service-installation` | Container install/uninstall progress |
| `service-updates` | Available update notifications |

---

## CI/CD

- **`release.yml`** — semantic-release on push to main, stamps version in root `package.json`
- **`docker.yml`** — manual dispatch, builds multi-stage Docker image, pushes to GHCR
- **`dependabot.yaml`** — dependency updates
- Image published as `ghcr.io/crosstalk-solutions/project-nomad:{version}`
- Fork remote: `trek-e/project-nomad` (upstream: `Crosstalk-Solutions/project-nomad`)

---

## Dev Quickstart

```bash
cd admin
cp .env.example .env  # fill in DB/Redis credentials
npm install
node ace migration:run
node ace db:seed
node ace serve --hmr     # dev server on :8080
# In another terminal:
node ace queue:work --all  # process background jobs
```

Requires MySQL 8 and Redis 7 running locally (or via Docker).
