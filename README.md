# English → Persian search

Next.js UI preserved from the source project, with a modular FastAPI backend.
Search reads a persistent index. Crawling runs as a separate bounded background workload.

    frontend/                 original UI, proxy and Playwright tests
    backend/app/
      api/                    typed routes and schemas
      core/                   settings, security, logging, limits, metrics
      domain/                 models and replaceable interfaces
      persistence/            operational metadata and repositories
      crawling/               safe fetch, discovery, durable frontier and scheduler
      indexing/               normalization, language, passages and publication
      retrieval/              query plans, lexical/dense/exact, RRF, reranking, gate
      answering/              evidence, answerability and citation checks
      integrations/           OpenSearch and configurable model clients
      services/ workers/      orchestration
    backend/migrations/       Alembic revisions
    backend/tests/            contracts, unit, retrieval and integration tests
    infra/                    reproducible PostgreSQL/OpenSearch dependencies
    scripts/                  bootstrap, reindex, evaluation, smoke
    storage/                  ignored runtime files
    docs/                     architecture, operations and verification records

## Windows setup

Install Python 3.12, uv, Node 24 and optionally Docker Desktop.
From this directory:

    powershell -ExecutionPolicy Bypass -File scripts/bootstrap.ps1

In a backend terminal:

    cd backend
    uv run --locked uvicorn app.main:app --host 127.0.0.1 --port 8050 --limit-concurrency 64

In a frontend terminal:

    cd frontend
    npm run dev -- -p 3050

Open http://localhost:3050. Linux/macOS can use sh scripts/bootstrap.sh.
SQLite development mode needs no Docker. Run backend commands from backend/.

## Native OpenSearch on Windows (no Docker)

The official Windows 3.3.2 distribution includes Java. From the project root:

    powershell -ExecutionPolicy Bypass -File infra/opensearch-windows.ps1 -Action install
    powershell -ExecutionPolicy Bypass -File infra/opensearch-windows.ps1 -Action start
    powershell -ExecutionPolicy Bypass -File infra/opensearch-windows.ps1 -Action status

The default installation is C:\OpenSearch\persian-search-engine; it needs an initial
35 GiB free, including watermark headroom. Installation paths must have no spaces.
The node listens on http://127.0.0.1:9200 and retains data across process restarts.
Use -Action stop or -Action restart for lifecycle control. Run -Action start after
Windows restarts; automatic startup is not installed. Security is disabled for this
loopback-only development node.

Set these values in the ignored root .env, then start the backend as above:

    SEARCH_BACKEND=opensearch
    OPENSEARCH_URL=http://127.0.0.1:9200

This local setup uses SQLite for operational metadata and OpenSearch for search storage.
The current machine has already been installed, started and configured this way.
For installation provenance, disk limits and verification, see docs/operations.md.

## PostgreSQL and OpenSearch

    docker compose -f infra/docker-compose.yml config --quiet
    docker compose -f infra/docker-compose.yml up -d --build

Compose migrates the database before starting the API. Named volumes own runtime data.
Ports are bound to localhost; the base Compose file is for local development.
Remote Docker contexts require tunnels to their loopback ports. Environment endpoints
inside containers must be reachable from the Docker host; localhost refers to the container.
See docs/operations.md for migration, backup, indexing and remote-host limitations.

## Whole-site crawling

Add a website root URL in Sources to discover its public pages recursively through
links and sitemaps. Pages become searchable while the crawl runs, and unfinished work
resumes after a backend restart. Refresh recrawls saved URLs; Continue resumes a crawl
that reached its configured page budget.

Defaults allow 1,000,000 pages/frontier entries, four concurrent requests per site,
eight globally, 250 ms request spacing, depth 12 and up to 1,000 sitemap files.
Tune CRAWL_* values in .env and restart the backend to apply them. Traversal stays
within the submitted host/path and respects robots, body limits and URL trap bounds.
Run one backend per database; a local scheduler lock rejects duplicate instances.

## Semantic providers

Configure a multilingual OpenAI-compatible embedding endpoint and optional /rerank endpoint
in root .env. The existing local chat settings LM_BASE_URL, LLM_MODEL and LLM_API_KEY work.
The migration reused those three settings in an ignored local .env; no secret is committed.

English retrieval uses Persian LLM rewrites and multilingual vectors independently.
Without either semantic provider, general English → Persian matching is unavailable;
Persian/Latin lexical search and supported entity aliases continue to work.
Model clients have timeouts, bounded concurrency, validation and fallbacks.
See docs/search-quality.md for model families, adapters and evaluation.

## Verification

From backend/:

    uv run --locked --extra dev pytest -q
    uv run --locked --extra dev ruff check app tests
    uv run --locked --extra dev mypy app
    uv run --locked alembic check
    uv run --locked python ../scripts/eval_search.py --fixture --offline
    uv run --locked python ../scripts/smoke_search.py

From frontend/:

    npm run lint
    npm run typecheck
    npm run build
    npx playwright install chromium
    npm test

Real PostgreSQL/OpenSearch tests are opt-in; commands and actual outcomes are in
docs/TEST_MATRIX.md. The smoke uses a controlled crawl fixture and a real configured LLM.
It is evidence of cross-language rewrite integration, not a general ranking benchmark.

## Git and runtime data

The root .gitignore excludes secrets, environments, caches, test/build output, logs,
SQLite files, database dumps, crawl artifacts, search indexes, model weights and backups.
Lockfiles, source, migrations, fixtures, documentation and storage/.gitkeep stay tracked.
Required large distributable assets should use Git LFS; model weights do not belong here.
