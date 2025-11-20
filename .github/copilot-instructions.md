# Octopets AI Agent Brief
## Architecture
- AppHost (`apphost/AppHost.cs`) is the source of truth: run `aspire run` or `dotnet run --project apphost` to spin up backend, React frontend, and three Python agents with shared telemetry and env wiring.
- AppHost injects URLs (`REACT_APP_AGENT_API_URL`, `LISTINGS_AGENT_URL`, etc.) and flips `ENABLE_CRUD`/`ERRORS` based on publish mode—never hardcode service origins inside app code.
- The backend (`backend/`) is a .NET 9 Minimal API; endpoints live in `Endpoints/*Endpoints.cs` and rely on repository interfaces in `Repositories/Interfaces`.
- React SPA in `frontend/` uses `agentService.ts` + `appConfig.ts` to decide between orchestrator/listings/sitter agents and whether to call `/api` or mock data.
- Python agents (`agent/`, `sitter-agent/`, `orchestrator-agent/`) run via uv; the orchestrator delegates between listings and sitter agents (see `docs/multi-agent-orchestration.md`).
## Data & Models
- Backend persistence is in-memory EF Core (`AppDbContext.cs`) with JSON conversions for list fields; seeding lives directly in `SeedData`.
- Frontend mock listings are kept in `frontend/src/data/listingsData.ts`; keep schema in sync with backend models (`backend/Models/Listing.cs`, `Review.cs`).
- Config flag `REACT_APP_USE_MOCK_DATA` (default true in dev) is read in `frontend/src/config/appConfig.ts`.
## API & Backend Patterns
- Add endpoints via extension methods (`MapListingEndpoints`, `MapReviewEndpoints`) and call them from `Program.cs`; they expect DI-bound repositories.
- Repositories follow async CRUD signatures (`IListingRepository` etc.); use them instead of DbContext in endpoints.
- Honour feature flags: `ENABLE_CRUD` blocks POST/PUT/DELETE and `ERRORS` toggles the stress-test path inside `ListingEndpoints`.
- OpenAPI + Scalar are wired by `builder.Services.AddOpenApi()`; keep annotations via `.WithName/.WithDescription/.WithOpenApi` on the route builder.
## Frontend Patterns
- Routing is React Router; feature pages live under `frontend/src/pages`, atomic UI in `components/`.
- Shared UI data (icons, constants) is under `frontend/src/data/constants.ts` and `constantsJsx.tsx`; reuse instead of duplicating copy.
- Use `agentService.sendMessage(message, AgentType)` when talking to agents; defaults to orchestrator and logs target URL.
- Styling relies on plain CSS modules in `frontend/src/styles`; animations like the polaroid orbit follow the directives in `.github/instructions/polaroid.instructions.md`.
## Agents
- Python services use FastAPI (`app.py`) plus uv for dependency management; run `uv sync` once per agent dir, then `uv run python app.py`.
- Orchestrator instructions and tool wiring live in `orchestrator-agent/orchestrator.py`; extend by adding a new async tool and registering it with `ChatAgent`.
- Sitter agent data source is `sitter-agent/data/pet-sitter.json`; keep new sitter fields reflected in both the JSON and tool filter signature.
## Developer Workflows
- Canonical start: `aspire run` (or `dotnet run --project apphost`) from repo root to get all services + dashboard; this also publishes env vars expected by the frontend and agents.
- Frontend standalone dev: `cd frontend; npm install; npm start`; tests via `npm test` (Jest) and `npm run test:e2e` (Playwright config in `playwright.config.ts`).
- Backend standalone dev: `cd backend; dotnet run`; Swagger/Scalar mounts automatically in Development, hitting `/scalar/v1`.
- Python agents can be run individually for debugging (`uv run python orchestrator.py` for CLI mode, `uv run python app.py` for HTTP).
- Aspire deploy guidance lives in `azure.yaml`; use `azd up` inside `apphost/` for Azure provisioning.
## References
- Read `docs/multi-agent-orchestration.md` before touching cross-agent flows; it documents the routing heuristics and env expectations.
- Use `.github/instructions/polaroid.instructions.md` for landing-page image work; no new files or hover animations allowed.
- The repo root `README.md` lists required tooling versions (Aspire CLI, uv, Node 18, .NET 9); align dev containers accordingly.