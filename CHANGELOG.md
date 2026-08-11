## [Unreleased]

### Changed

- **Migrated from `@modelcontextprotocol/sdk` v1 to the v2 SDK (`@modelcontextprotocol/server` + `@modelcontextprotocol/node` 2.0.0-beta.5) with dual-era serving.** All three entrypoints now consume one shared per-request server factory (`AutotaskMcpServer.requestFactory()`), so the tool/resource/prompt surface can never drift between protocol eras:
  - **stdio** is served via `serveStdio(factory)` — the opening handshake pins the connection to its era (classic 2025 `initialize` or modern 2026-07-28).
  - **Node HTTP** is served via `createMcpHandler(factory, { legacy: 'stateless' })` wrapped with `toNodeHandler`. Modern 2026-07-28 envelope traffic is served natively; 2025-era traffic is served by the SDK's default stateless legacy fallback — a fresh server per request, the same per-request stateless idiom this server has always used. CORS, `/health`, and the gateway 401 credential gate are unchanged.
  - **Cloudflare Workers** uses the same `createMcpHandler` (memoized per isolate) through its web-standard `fetch` face, replacing the hand-wired `WebStandardStreamableHTTPServerTransport` per-request plumbing.
  - The gateway credential contract is byte-identical: `X-API-Key`, `X-API-Secret`, `X-Integration-Code`, and optional `X-API-Url` headers are read per request (now from the handler's `ctx.requestInfo`) and bind an isolated per-tenant `AutotaskService`; missing credentials in gateway mode still answer 401 before any MCP processing.
  - The tool surface is unchanged: same 98 tool names and input schemas from `tool.definitions.ts` served to both eras (pinned by `scripts/smoke-dual-era.mjs`).
  - Wire-visible behavior notes for legacy (2025-era) clients: successful POST responses now arrive as single-message `text/event-stream` SSE frames instead of plain JSON bodies (both are canonical Streamable HTTP; every spec-conforming client accepts SSE), and GET/DELETE session operations on `/mcp` answer 405 from inside the SDK's stateless fallback rather than from hand-rolled routing.
  - Internal API renames: `McpError`/`ErrorCode` → `ProtocolError`/`ProtocolErrorCode`, `setRequestHandler(SomeRequestSchema, …)` → `setRequestHandler('method/name', …)`. Local result types tightened to the v2 SDK's literal-typed wire shapes (`inputSchema.type: 'object'`, content `type: 'text'`, resource contents text-xor-blob).

### Added

- **Dual-era smoke test** (`scripts/smoke-dual-era.mjs`): boots the built HTTP server with dummy credentials and proves a hand-crafted 2025-era JSON-RPC client (classic `initialize` → `notifications/initialized` → `tools/list`) and a modern `@modelcontextprotocol/client@2` StreamableHTTP session both see the identical non-empty tool surface. Exits non-zero on any failure.

- **Interactive ticket card via MCP Apps (SEP-1865).** `autotask_get_ticket_details` results now render as an interactive card in MCP Apps hosts (Claude Desktop/web, and other hosts advertising the `io.modelcontextprotocol/ui` extension), instead of a wall of JSON. The card shows status/priority/queue as human-readable labels, company and assignee names, dates, and recent notes — and includes a working "Add note" round-trip that calls `autotask_create_ticket_note` from inside the card. Non-App hosts are unaffected: the tool's JSON payload is unchanged apart from a new `_card` field.
  - The card is **brand-neutral by default** (system fonts, neutral palette, no baked-in identity — this is a published server) and brandable without rebuilding: `MCP_BRAND_NAME`, `MCP_BRAND_LOGO_URL`, `MCP_BRAND_PRIMARY_COLOR`, `MCP_BRAND_ACCENT_COLOR`, `MCP_BRAND_BG`, and `MCP_BRAND_TEXT` env vars are injected as `window.__BRAND__` at serve time (a gateway can inject the same object per-org). A test pins the default bundle to zero brand identity and zero external font fetches.
  - The two renderable tools advertise the UI via `_meta` (`ui/resourceUri`, plus the nested `ui.resourceUri` form) pointing at a new `ui://autotask/ticket-card.html` resource served as `text/html;profile=mcp-app`. The card HTML is a self-contained vite single-file bundle embedded at build time (`src/generated/ticket-card-html.ts`, committed), so it serves identically from stdio, Node HTTP, and the fs-less Cloudflare Workers runtime.
  - The card's "Add note" defaults are resolved server-side from the tenant's `TicketNotes` picklists and only ever use an internal-visibility `publish` value; if no internal option exists the card renders read-only rather than risk a client-visible note (`src/handlers/card.builder.ts`).
  - The card payload builder is best-effort: picklist or note-fetch failures degrade the card (or drop it) without affecting the tool result. 10 new contract tests in `tests/mcp-apps.test.ts` pin the `_meta` advertisement, the `ui://` resource wire shape, and the card/note-default normalization.
  - New `npm run build:ui` regenerates the embedded HTML after editing `ui/` (requires the new `vite`, `vite-plugin-singlefile`, and `@modelcontextprotocol/ext-apps` devDependencies); plain `npm run build` and CI are unaffected.

### Changed

- **Migrated to ESLint 9/10 flat config** ([#141](https://github.com/wyre-technology/autotask-mcp/issues/141)). ESLint 9 dropped support for the legacy `.eslintrc.json` format, and `@typescript-eslint` v6 can't run under it — so the `eslint`, `@typescript-eslint/parser`, and `@typescript-eslint/eslint-plugin` Dependabot bumps were three coupled breaking changes that could only land together. Replaced `.eslintrc.json` with `eslint.config.mjs` (flat config) using the unified `typescript-eslint` package (bundles parser + plugin in version-lockstep), bumped `eslint` 8 → 10 and typescript-eslint 6 → 8, and added `@eslint/js` + `globals`. The ruleset is unchanged (`no-explicit-any`/`no-unused-vars` as warnings, `no-console` off, same ignores); lint output is identical at 0 errors / 193 warnings. `lint` script simplified from `eslint src --ext .ts` to `eslint src` (flat config infers extensions).
  - ESLint 10's `preserve-caught-error` rule surfaced three `catch` blocks that re-threw without preserving the original error. Fixed by attaching `{ cause: err }` to the rethrown `Error` in `autotask-http.ts` (network errors) and `config.ts` (zone-detection network + malformed-response errors), improving error chains for debugging. This required adding `ES2022.Error` to the tsconfig `lib` for the `Error(message, { cause })` constructor overload (emit target unchanged at ES2020; Node 18+ supports `Error.cause` at runtime).
  - Supersedes Dependabot PRs #113 (incorporated), #114 and #124 (parser/plugin replaced by the unified package). TypeScript 6 (#122) remains tracked separately in #141.

### Removed

- **Dropped Node.js 18 support.** ESLint 10 requires Node `^20.19 || ^22.13 || >=24`, and Node 18 reached end-of-life on 2025-04-30. Bumped `engines.node` to `>=20.0.0` and removed Node 18 from the CI test matrix and release pipeline (the Docker image already targets Node 22). Supported runtimes are now Node 20 and 22.

- **Published to GitHub Packages.** The npm package is now scoped to `@wyre-technology/autotask-mcp` (GitHub Packages rejects unscoped names) and `@semantic-release/npm` `npmPublish` is enabled. The release workflow now configures an authenticated `.npmrc` for `npm.pkg.github.com` and grants `packages: write`. The unscoped `bin` command name (`autotask-mcp`), the `io.github.wyre-technology/autotask-mcp` MCP Registry identifier, and the GHCR image name are unchanged.

### Added

- **`issueType` and `subIssueType` on `autotask_update_ticket`** ([#109](https://github.com/wyre-technology/autotask-mcp/issues/109)). Both fields are already accepted by the underlying payload builder (and have always been exposed on `autotask_create_ticket`), but the update tool's input schema didn't advertise them — so triage workflows couldn't change ticket issue classification on an existing ticket without falling back to `autotask_raw_request`. Added them as optional numeric picklist IDs with the same descriptions used on `autotask_create_ticket`. New tests in `tests/lazy-loading.test.ts` pin the schema shape and assert the handler forwards both fields to `updateTicket()`.

- **Structured rate-limit handling for Autotask API thresholds** ([#69](https://github.com/wyre-technology/autotask-mcp/issues/69), [#91](https://github.com/wyre-technology/autotask-mcp/issues/91)). Previously, when Autotask returned HTTP 429 (per-integration API threshold exceeded — ~10k req/hr soft, ~20k hard), the response bubbled up as a generic `Error` indistinguishable from a 500 or any other failure. LLM-driven workflows that fan out (e.g. "status report for all open projects with notes" — the canonical scenario @faspina reported) kept retrying the same expensive path while the user got threshold-exceeded emails from Autotask and saw disconnects.
  - New `AutotaskRateLimitError` (exported from `src/services/autotask-http.ts`) is thrown specifically on HTTP 429. Carries `retryAfterSeconds` parsed from the `Retry-After` response header (RFC 7231 — integer seconds or HTTP-date supported, falls back to 60s when missing/unparseable).
  - `AutotaskToolHandler.callTool` recognizes the typed error and returns a structured tool result with `error_type: "rate_limited"`, `retry_after_seconds`, and an explicit `instruction` field telling the LLM not to retry. Belt-and-suspenders for clients that don't parse `error_type` programmatically.
  - 5 new tests in `tests/rate-limit.test.ts` cover Retry-After parsing (integer, missing, garbage), non-429 errors staying as generic `Error`, and end-to-end propagation to the structured tool envelope.

- **Rate-limit scoping hints on fan-out tool descriptions.** Tools that LLMs commonly loop over (`autotask_search_ticket_notes`, `autotask_search_project_notes`, `autotask_search_company_notes`, `autotask_search_time_entries`, `autotask_search_ticket_attachments`) now include a "RATE LIMIT TIP" reminding the model to scope the parent record list before fanning out. The project_notes hint cites #69 by issue number so the model knows this is a known failure mode.

- **README "Rate limits" section** documents the per-integration thresholds, how to raise them in Autotask Admin (dedicated API user + Workflow Rules → API Tracking Identifier), and recommended scoping patterns.

- **Ticket history audit-trail tools** (`autotask_get_ticket_history`, `autotask_search_ticket_history`). Exposes the Autotask `/TicketHistory` entity (GET-only) so callers can answer "when did this ticket transition from status X to status Y", "who changed the priority", etc. `search` requires a `ticketId` (Autotask does not support unscoped history queries) and fails fast with a friendly error before any network round-trip if it's missing. Surfaced via the `tickets` category bundle.

### Fixed

- **Contact updates failed on zones where both PATCH and PUT are unavailable** ([#197](https://github.com/wyre-technology/autotask-mcp/pull/197), follow-up to [#133](https://github.com/wyre-technology/autotask-mcp/issues/133)). On some Autotask zone hosts, `updateContact()` had no working route: the collection-level `PATCH /Contacts` returns an HTML 404 (the Zone DE1 behaviour from #133) *and* the `PUT /Contacts/{id}` fallback is rejected with 405, so every contact update died with no workaround through `autotask_update_contact`. Contacts are a child entity of Companies, so `updateContact()` now retries through the documented child route `PATCH /Companies/{companyID}/Contacts` (via the existing `childUpdate()` helper) — resolving the parent `companyID` from the update payload when supplied, otherwise via `getContact(id)`. The retry is gated strictly on 404/405, so genuine 400/422 validation errors still surface unchanged, and it runs last so zones where `update()` already works (including DE1's PUT fallback) keep their existing behaviour. Verified in production against a US zone: 51 consecutive contact updates via the child route, plus an end-to-end smoke of the typed tool exercising the `getContact` fallback. Adds `tests/contact-update-child-route.test.ts`. Thanks to @pdlaskbis (Phillip Long).

- **Open-ticket searches returned Completed tickets** ([#193](https://github.com/wyre-technology/autotask-mcp/pull/193)). `searchTickets()`'s default "open tickets only" filter used `{ op: 'ne', field: 'status', value: 5 }`, but Autotask's REST query API has no `ne` operator — it uses `noteq`. Autotask silently dropped the invalid clause, so any `autotask_search_tickets` call without an explicit `status` returned tickets of every status, including Complete (status 5). Changed the operator to `noteq`. Verified against a live tenant: `ne` returned 500 rows (100% Complete — filter dropped); `noteq` returned only open tickets. Explicit `status` filters (which correctly use `eq`) were never affected. Two regression tests pin that the status-less default emits `noteq` (not `ne`) and that an explicit `status` uses `eq` without the default. Thanks to @sk1tt1sh (Troy Harshberger) for the fix and live verification.

- **`queueID` (and `priority`) were silently ignored by `autotask_search_tickets`** ([#208](https://github.com/wyre-technology/autotask-mcp/issues/208)). The [tool reference](website/src/content/docs/reference/tools.md) documented `queueID` and `priority` as searchable filters, but neither was declared in the tool's input schema, defined on `AutotaskQueryOptionsExtended`, nor translated into a query filter by `searchTickets()`. A call like `{ "queueID": 1, "pageSize": 5 }` dropped the filter entirely and fell through to the default `status ne 5` query — returning unrelated open tickets, or (as reported) an empty result the client surfaced as "No tickets found matching search criteria: queueID=1".
  - `searchTickets()` now pushes `{ op: 'eq', field: 'queueID' }` and `{ op: 'eq', field: 'priority' }` filters when those options are provided; both fields were added to the `autotask_search_tickets` input schema and to `AutotaskQueryOptionsExtended`.
  - The handler's zero-filter elicitation guard (`hasFilters`) now also counts `queueID`, `priority`, and `contactID` — previously a search filtered solely by any of those was treated as filterless and triggered the interactive date-range prompt. `contactID` was already a supported filter with the same latent gap.
  - 2 new tests in `tests/autotask-service.test.ts` assert `searchTickets` emits the `queueID` and `priority` `eq` filters in the query body.

- **Contact updates were completely broken on Autotask Zone 18 / DE1** ([#133](https://github.com/wyre-technology/autotask-mcp/issues/133)). `autotask_update_contact` (and every other `update()` caller) issues a collection-level `PATCH /{Entity}` with the id in the body — the only update route Autotask documents. On the European DE1 zone (`webservices18.autotask.net`) that route is not registered in IIS and returns an HTML **404**, while item-level `PATCH /{Entity}/{id}` is rejected with 405, so updates failed on every attempt with no workaround.
  - `AutotaskHttpClient.update()` now falls back to `PUT /{Entity}/{id}` (universally supported across zones) **only** when the collection PATCH returns a 404. The fallback is gated strictly on the numeric status, so genuine 400/422 validation errors still surface to the caller unchanged.
  - `AutotaskHttpClient.request()` now attaches the numeric `status` to thrown HTTP errors so callers branch on it reliably instead of substring-matching the message (the message embeds the response body, which can coincidentally contain a status-like number).
  - **`autotask_update_contact` now exposes `userDefinedFields`** (`[{name, value}]`), matching `autotask_update_ticket`. Contacts are `hasUserDefinedFields: true`, but the schema previously omitted the field, so custom-field updates were impossible. The handler already forwarded all args, so only the schema needed the addition.
  - **`autotask_raw_request` now allows `PUT`** — added to both the runtime method allowlist *and* the advertised `method` enum in the tool schema (they were inconsistent before this would have surfaced), so the item-level update route is reachable via the raw escape hatch and the LLM is actually told it exists.
  - 6 new tests in `tests/contact-update-zone18.test.ts`: PATCH→PUT fallback on 404, no-fallback on 400 (incl. a 400 whose body contains "404"), no spurious PUT when PATCH succeeds, the `userDefinedFields` schema shape, the `rawRequest` PUT enum/runtime consistency, and PUT acceptance through `rawRequest`.
  - Out of scope (no DE1 evidence, untested): `AutotaskHttpClient.childUpdate()` issues the same collection-style PATCH on nested `/{parent}/{id}/{child}` routes and would also 404 on DE1 if those routes are likewise unregistered. The top-level `update()` covers all 10 entity update paths in the service; nested child updates are a separate, unverified route shape and were intentionally left alone.

- **`listAllCompanies` and other `query()` callers now clamp the per-page `MaxRecords` body param to the Autotask API limit (500).** `AutotaskHttpClient.query()` previously passed the caller's `opts.maxRecords` value directly as the per-page `MaxRecords` field in the request body. `MappingService.refreshCompanyCache()` calls `listAllCompanies(20_000)`, which produced `MaxRecords: 20000` in the request — Autotask responds with HTTP 500 "maxCountOfRecordsToReturn must be between 1 and 500". The cache pre-warm worked fine on the legacy static-singleton MappingService because it ran exactly once per process and the failure was silently absorbed into an empty cache; once the cache was made per-request the same call ran on every request and surfaced the API error on every call.
  - Introduced an exported `AUTOTASK_MAX_PAGE_SIZE = 500` constant in `src/services/autotask-http.ts`.
  - `query()` now treats `opts.maxRecords` as the **total cap** (how many rows to collect across the whole pagination walk) and derives the **per-page size** as `min(totalCap, AUTOTASK_MAX_PAGE_SIZE)`. The cursor walk continues to follow `pageDetails.nextPageUrl` until the total cap is reached.
  - `childQuery()` clamps its per-page `MaxRecords` to `AUTOTASK_MAX_PAGE_SIZE` for the same reason. `childQuery` does not walk `nextPageUrl`, so only the clamp matters there.
  - 2 new regression tests in `tests/autotask-service.test.ts` under `Per-page MaxRecords clamping`:
    - `listAllCompanies` with the default 20_000 cap captures the actual `MaxRecords` value sent and asserts it is `<= 500`, while still returning all 700 records from a multi-page tenant.
    - `query()` honors a small caller cap (25) by sending exactly `MaxRecords: 25` — small queries don't pay the full 500-row page cost.

- **Per-instance `MappingService` for proper tenant isolation in gateway mode.** Aligns the `MappingService` lifecycle with the per-request `AutotaskToolHandler` introduced in the worker refactor so each request gets its own mapping cache scoped to the request's credentials.
  - Removed `static initPromise` and `private constructor`. Replaced the static `getInstance()` factory with a per-call `MappingService.create(autotaskService, logger, options)` that constructs a fresh instance bound to the supplied `AutotaskService` and awaits its cache initialization. Concurrent inits on the same instance still coalesce via a new per-instance `initPromise`.
  - `AutotaskToolHandler.getMappingService()` now calls `MappingService.create(...)`. Because `AutotaskToolHandler` is already constructed per-request in gateway mode (via `McpServer.buildPerRequestHandlers`, added in the worker refactor), each request now ends up with its own isolated `MappingService`.
  - Gateway-mode `/mcp` now rejects requests missing any of `X-API-Key` / `X-API-Secret` / `X-Integration-Code` with HTTP 401 (JSON-RPC error `-32001`) instead of falling through to the env-configured `this.toolHandler`.
  - 4 new regression tests in `tests/mapping.test.ts` under `tenant isolation`: independent instances per tenant, no cross-pollution under concurrent init, isolated cache-clear semantics, and a 10-tenant high-fan-out parallel-init each-sees-only-own-data assertion. The `should return the same instance on subsequent calls` test was inverted to `should return a DISTINCT instance on each call` to guard against re-introducing the previous singleton shape.

- **deploy:** clarified that the one-click DigitalOcean deploy needs **no**
  GitHub Packages token. Unlike the other WYRE MCP servers, `autotask-mcp` has
  no private `@wyre-technology/*` GitHub Packages dependency — its only WYRE
  dependency is the `autotask-node` SDK, declared as a git dependency on the
  **public** `wyre-technology/autotask-node` repo, which `npm install` resolves
  anonymously. Added a README note so operators don't add an unnecessary build
  variable. (No `.npmrc` is created, because the package is not on the GitHub
  Packages npm registry.)

- **`autotask_create_ticket_note` exposed internal notes to clients via wrong picklist labels** ([#126](https://github.com/wyre-technology/autotask-mcp/issues/126)). The tool's `noteType` and `publish` descriptions hardcoded labels like `1=Internal Only, 2=All Autotask Users, 3=Everyone` — but Autotask picklist IDs are tenant-configurable, and in many tenants `1` actually maps to *All Autotask Users* (the opposite of "Internal Only"). The handler also silently defaulted both fields to `1` when the LLM omitted them. Notes intended to be internal were being published externally.
  - Fix: replaced the hardcoded labels with descriptions that direct the caller to `autotask_get_field_info` (entity `TicketNotes`, fields `noteType` and `publish`) — same pattern used by every other picklist field in the codebase. Made both fields `required` in the schema and added explicit handler-side guards so omission fails fast with an actionable error message instead of silently defaulting.
  - 4 new tests in `tests/create-ticket-note-picklist.test.ts` pin the schema's `required` array, the absence of the wrong baked-in labels, and the discovery-hint error messages.
  - Out of scope (same bug pattern, separate fix): `autotask_create_project_note` has identical hardcoded labels at `tool.definitions.ts:1309-1316`. Flagged but not changed here.

- **Four `search*` tools advertised filter params they silently dropped** ([#104](https://github.com/wyre-technology/autotask-mcp/issues/104), [#105](https://github.com/wyre-technology/autotask-mcp/issues/105)). `searchContracts`, `searchConfigurationItems`, `searchInvoices`, and `searchTasks` all published `inputSchema` properties like `companyID`, `searchTerm`, `status`, `assignedResourceID` — every property described as "Filter by …" — but the service methods read only `options.filter` and `options.pageSize`. The advertised properties were accepted, logged at debug level, then discarded. Callers got the same MATCH_ALL page-1 slice regardless of what they passed.
  - Effect: typed search tools were useless for any non-trivial query. Workaround was `autotask_raw_request`, which defeats the purpose of having schema-typed tools and isn't discoverable by MCP clients reading the catalog.
  - Fix: each method now mirrors the `searchProjects` pattern that was already in place — translates schema-shaped args into `QueryFilter[]` entries, with the `options.filter` escape hatch preserved for advanced callers. Field-name mappings per Autotask REST entity:
    - **Contracts**: `companyID`, `status`, `contractName` (for `searchTerm`)
    - **ConfigurationItems**: `companyID`, `isActive`, `productID`, `referenceTitle` (for `searchTerm`)
    - **Invoices**: `companyID`, `invoiceNumber`, `isVoided`
    - **Tasks**: `projectID`, `status`, `assignedResourceID`, `title` (for `searchTerm`)
  - `searchTasks` also dropped its advertised `page` argument — same bug class as the #101 fix to `searchCompanies`. Applied the same fetch-and-slice pattern over `http.query`'s cursor pagination.
  - 6 new tests in `tests/autotask-service.test.ts` mock `fetch` directly and assert the request body's `filter` array reflects each translated property. The "no filter args sends MATCH_ALL" test pins the no-regression case.
  - Defensive grep confirmed the contained scope: no other `search*` method on this service has the broken `Array.isArray(options.filter)`-only pattern. `getTimeEntries` has the same shape but isn't exposed via a search tool (`searchTimeEntries` is the real handler at line 2078).

- **`serverInfo.version` reported hardcoded `"1.0.0"` regardless of the running release** ([#94](https://github.com/wyre-technology/autotask-mcp/issues/94)). Both `src/utils/config.ts` and `src/mcp/server.ts` fell back to a literal `'1.0.0'` string instead of the actual build version. Effect: every release through `v2.x.x` reported `1.0.0` in the MCP `initialize` handshake. Clients that surface `serverInfo.version` (Claude Code's `claude mcp list`, etc.) showed `1.0.0` regardless of which image was actually running, making operator triage / bug-report attribution unreliable.
  - Fix:
    - Both fallback chains now read from the bundled `package.json` (TypeScript's `resolveJsonModule` was already enabled). Priority order: `MCP_SERVER_VERSION env > packageJson.version > 'unknown'`.
    - The Dockerfile patches `package.json`'s `version` field at build time using the existing `VERSION` build arg before `npm run build`. This is necessary because branch protection silently drops `@semantic-release/git`'s push-back on this repo — `package.json` on `main` stays stale, so the release pipeline has to inject the real version at image-build time. Local builds (where `VERSION="unknown"`) skip the patch so the checked-in `package.json` version is preserved.
    - Production stage now copies `package.json` from the builder (with the patch) rather than the build context.
  - `/health` endpoint now includes a `version` field so operators can `curl` for the running build without going through the MCP handshake.

- **`searchCompanies` silently dropped the `page` parameter** ([#101](https://github.com/wyre-technology/autotask-mcp/issues/101)). The method's `AutotaskQueryOptions` interface accepts `page`, but the implementation only forwarded `pageSize` to `http.query` — every call returned the first page regardless. `MappingService.refreshCompanyCache` looped 1..100 expecting offset pagination, hammered the same page 100×, and ended up with at most 200 companies cached after burning ~100 pagination API calls.
  - Effect: every tool call hung ~80s on first invocation (and every 30 min on TTL expiry) while the broken loop ran. On tenants with more than 200 companies, IDs past the first page fell through to single-record `getCompany(id)` direct-get. Symptom from the user side: `autotask_test_connection`, `autotask_search_companies` etc. hung on first call; only meta-tools worked.
  - Fix:
    - `searchCompanies` now honors `page` by fetching up to `page * pageSize` records via `http.query` (which already walks Autotask's cursor pagination internally) and slicing the target window. Wasteful at high page numbers, but Autotask's REST API is cursor-based — there is no native offset, and this matches what callers expect.
    - New `listAllCompanies(maxRecords = 20_000)` method for bulk pre-warm: single `http.query` call with no filtering and a large `maxRecords`, letting `http.query`'s built-in `nextPageUrl` walker do its job. Logs a warning if the cap is hit.
    - `MappingService.refreshCompanyCache` now calls `listAllCompanies()` once instead of looping. Atomic-swap-on-success semantics from the prior fix are preserved.
  - Test layer up: the previous pagination tests in `tests/mapping.test.ts` mocked `searchCompanies` directly and asserted call signatures — they passed because the mock observed the `page` argument production was dropping. New tests in `tests/autotask-service.test.ts` mock `fetch` and prove pagination behavior end-to-end through `http.query`.

- **`LAZY_LOADING` env var was dead code in `MappingService`** ([#101](https://github.com/wyre-technology/autotask-mcp/issues/101)). The flag is parsed in `src/utils/config.ts` and passed to `AutotaskToolHandler`, where it filters the `listTools()` output to hide non-meta tools. It never reached `MappingService.initializeCache`, so the eager company/resource cache pre-warm ran on every server start regardless. There was no documented way to opt out of the 80-second startup cost.
  - Fix: `MappingService.getInstance` now accepts `{ lazyLoading }` and forwards it to the constructor. When set, `initializeCache()` returns immediately and `refreshCacheIfNeeded()` is a no-op. `getCompanyName()` and `getResourceName()` fall through to their existing per-record direct-get paths. Trade-off: one extra API call per unique ID per response, no startup hang.
  - Wired through from `tool.handler.ts` so the env var flows: `LAZY_LOADING=true` → `AutotaskMcpServer` → `AutotaskToolHandler` → `MappingService`.

- **`autotask_get_ticket_attachment` silently dropped the `includeData` flag** — the tool advertised an `includeData` parameter and the service method accepted it, but the implementation always called the child endpoint `GET /Tickets/{id}/Attachments/{aid}` which **never** populates the `data` field. Callers asking for the binary got back metadata with `data: undefined` 100% of the time, which surfaced in the community as "issues pulling attachments consistently."
  - Root cause: two distinct Autotask endpoints. The child endpoint (`/Tickets/{id}/Attachments/{aid}`) returns metadata only; the top-level entity endpoint (`/TicketAttachments/{id}`) returns metadata **plus** the base64 binary in `data`. The service was hardcoded to the former regardless of `includeData`.
  - Fix: `getTicketAttachment` now routes to `/TicketAttachments/{id}` when `includeData=true`, with three safety nets: (1) the returned attachment's `ticketID` is verified to match the requested `ticketId` (returns null on mismatch — defense against cross-ticket reads if a caller passes a wrong parent), (2) oversized binaries are stripped before return with a `dataOmittedReason` field explaining why (default cap 1,000,000 base64 bytes ≈ 750 KB raw, configurable per call via `maxInlineBase64Bytes`), and (3) the metadata-only path still uses the cheaper child endpoint when `includeData` is false/omitted.
  - Why the size cap: Autotask attachments can be up to 3 MB raw (~4 MB base64). MCP client tool-result limits are typically ~1 MB. Without the cap, large attachments would arrive truncated/garbled at the client even with the correct endpoint, reproducing the "didn't work" outcome through a different failure mode.
  - Test coverage: 5 new endpoint-routing tests in `tests/autotask-service.test.ts` covering metadata-only path, data-returning path, size guard, cross-ticket isolation, and explicit size override.

- **`MappingService` company cache was silently truncated to 25 entries** — `refreshCompanyCache()` called `searchCompanies({})` assuming "no pageSize = fetch all pages", but the underlying `searchCompanies` defaults to `pageSize: 25, page: 1` and returns only the first page. The cache was then logged as `"COMPLETE dataset"`, which was incorrect.
  - Effect: for any company whose ID wasn't on the first page (~everything past ID ~198 in a typical tenant), `getCompanyName()` fell through to a single-record `getCompany(id)` direct-lookup. When the Autotask REST direct-get returned a stale or renamed name (observed for at least one merged/renamed company in the wild), that wrong name was written into the cache and served to every downstream consumer — `autotask_search_tickets`, `autotask_search_projects`, notes, time entries, etc. — for a 30-minute cache window. Surface: the `company` field on enriched responses displayed the wrong tenant name, which looks like cross-tenant data leakage even though the underlying IDs and ownership were correct.
  - Fix: `refreshCompanyCache()` now actually paginates — loops `searchCompanies({page, pageSize: 200})` until a short page is returned, building a fresh `Map` and atomic-swapping it into the cache only after full success (partial failures keep the prior cache rather than replacing it with a shorter one). Safety cap of 100 pages (20k companies) logs a warning rather than running forever.
  - Hardening: `getCompanyName()` still falls back to single-record `getCompany(id)` for companies added between refresh windows, but the fallback result is no longer written to the cache. This prevents a stale/wrong direct-get from poisoning the cache and being served to every subsequent caller.
  - Added `tests/mapping.test.ts` coverage: multi-page pagination, early-stop on short page, and fallback-does-not-poison-cache.

# [2.18.0](https://github.com/wyre-technology/autotask-mcp/compare/v2.17.2...v2.18.0) (2026-04-08)


### Features

* **attachments:** add autotask_create_ticket_attachment tool ([#55](https://github.com/wyre-technology/autotask-mcp/issues/55)) ([#62](https://github.com/wyre-technology/autotask-mcp/issues/62)) ([8ff325e](https://github.com/wyre-technology/autotask-mcp/commit/8ff325e1f11d24097aab0f1a5c7a5968bc4d409d))
* **billing:** add invoice details tool and billing item filters ([#55](https://github.com/wyre-technology/autotask-mcp/issues/55)) ([#61](https://github.com/wyre-technology/autotask-mcp/issues/61)) ([cc9354f](https://github.com/wyre-technology/autotask-mcp/commit/cc9354f9703e23c9dde3eaed3dee0e9f065e3a88))
* **checklist:** add ticket checklist items CRUD tools ([#55](https://github.com/wyre-technology/autotask-mcp/issues/55)) ([#59](https://github.com/wyre-technology/autotask-mcp/issues/59)) ([78e0f78](https://github.com/wyre-technology/autotask-mcp/commit/78e0f7805e36ce9c69e0ecd3b6c7326e95d51615)), closes [#33](https://github.com/wyre-technology/autotask-mcp/issues/33) [#32](https://github.com/wyre-technology/autotask-mcp/issues/32)
* **config:** auto-detect Autotask API zone from username ([#55](https://github.com/wyre-technology/autotask-mcp/issues/55)) ([#60](https://github.com/wyre-technology/autotask-mcp/issues/60)) ([01a3bae](https://github.com/wyre-technology/autotask-mcp/commit/01a3bae888c6cb3cb45d91afb50e83b2f0eed6c1))
* **projects:** add autotask_update_project tool ([#55](https://github.com/wyre-technology/autotask-mcp/issues/55)) ([#57](https://github.com/wyre-technology/autotask-mcp/issues/57)) ([1efeead](https://github.com/wyre-technology/autotask-mcp/commit/1efeead572b60b1513bd7c773ca684eb774e36c0))
* **tickets:** expand create/update_ticket field coverage ([#55](https://github.com/wyre-technology/autotask-mcp/issues/55)) ([#58](https://github.com/wyre-technology/autotask-mcp/issues/58)) ([16614ff](https://github.com/wyre-technology/autotask-mcp/commit/16614ff2209e5b82d57fbc83bbfbef0cc8e24080))

## [2.0.0](https://github.com/ITW-Sunshine/autotask-mcp/compare/v1.0.0...v2.0.0) (2026-08-11)


### ⚠ BREAKING CHANGES

* All 35 MCP tools now use 'autotask_' prefix to prevent naming collisions when multiple MCP servers are connected.

### Features

* add create_expense_item tool and fix expense report accessors ([0cf9f09](https://github.com/ITW-Sunshine/autotask-mcp/commit/0cf9f094e09d48fb4ed94e23242cfd354ab7f908))
* add DigitalOcean and Cloudflare deploy infrastructure and badges ([b68bad5](https://github.com/ITW-Sunshine/autotask-mcp/commit/b68bad5b7406ea17a0de4fe16ce65d75f7cb14d0))
* Add ESLint configuration file ([3a8ef4d](https://github.com/ITW-Sunshine/autotask-mcp/commit/3a8ef4d550f3ea8bf09640967709cf5e4823823d))
* Add gateway mode for hosted MCP deployments ([14d5682](https://github.com/ITW-Sunshine/autotask-mcp/commit/14d568223c9269de2d5e3e2eba5056351ca3e82d))
* add HTTP Streamable transport for remote MCP access ([2d31853](https://github.com/ITW-Sunshine/autotask-mcp/commit/2d3185348cb4387c5726892bb15d9c432279afa3)), closes [#7](https://github.com/ITW-Sunshine/autotask-mcp/issues/7)
* add MCP prompt handlers ([#67](https://github.com/ITW-Sunshine/autotask-mcp/issues/67)) ([6e37fb0](https://github.com/ITW-Sunshine/autotask-mcp/commit/6e37fb00666862c244759598d55699d03c7f0186))
* add MCPB (MCP Bundle) packaging for desktop distribution ([e7601b1](https://github.com/ITW-Sunshine/autotask-mcp/commit/e7601b1d158c261a6607530f59267dff99b06ba8))
* Add picklist discovery tools and elicitation support ([93c68f2](https://github.com/ITW-Sunshine/autotask-mcp/commit/93c68f20acf31c0a8cc661689f820bf7e3518393))
* add Regular Time entry support with name resolution ([f1947a8](https://github.com/ITW-Sunshine/autotask-mcp/commit/f1947a85c49790f55476d6c0c2f397b53172a0f0))
* add Service Call support (11 new tools) ([11b682e](https://github.com/ITW-Sunshine/autotask-mcp/commit/11b682e678c42c89c3ef963b1ca826ff653ea6b6)), closes [#38](https://github.com/ITW-Sunshine/autotask-mcp/issues/38)
* add Service Calls CRUD support ([#38](https://github.com/ITW-Sunshine/autotask-mcp/issues/38)) ([163d5eb](https://github.com/ITW-Sunshine/autotask-mcp/commit/163d5eb36c6936b416d1d0fb127641975a2c4c9e))
* add TicketCharges CRUD support ([b61aaf0](https://github.com/ITW-Sunshine/autotask-mcp/commit/b61aaf0b0e113121b73a851b17970164ada9edf4))
* add TicketCharges CRUD support ([#36](https://github.com/ITW-Sunshine/autotask-mcp/issues/36)) ([de2d684](https://github.com/ITW-Sunshine/autotask-mcp/commit/de2d684223047d6d87f4682ca2459ed8e02f06fa))
* **attachments:** add autotask_create_ticket_attachment tool ([#55](https://github.com/ITW-Sunshine/autotask-mcp/issues/55)) ([#62](https://github.com/ITW-Sunshine/autotask-mcp/issues/62)) ([8ff325e](https://github.com/ITW-Sunshine/autotask-mcp/commit/8ff325e1f11d24097aab0f1a5c7a5968bc4d409d))
* **billing:** Add BillingItems and BillingItemApprovalLevels support ([4c88034](https://github.com/ITW-Sunshine/autotask-mcp/commit/4c880348d7a930b5277a810b89a3c54cddedb509))
* **billing:** add invoice details tool and billing item filters ([#55](https://github.com/ITW-Sunshine/autotask-mcp/issues/55)) ([#61](https://github.com/ITW-Sunshine/autotask-mcp/issues/61)) ([cc9354f](https://github.com/ITW-Sunshine/autotask-mcp/commit/cc9354f9703e23c9dde3eaed3dee0e9f065e3a88))
* bump autotask-node to v2.2.0 with Regular Time entry support ([00f1f1e](https://github.com/ITW-Sunshine/autotask-mcp/commit/00f1f1e3078736f56cba1f4943ffec802a18197d))
* **checklist:** add ticket checklist items CRUD tools ([#55](https://github.com/ITW-Sunshine/autotask-mcp/issues/55)) ([#59](https://github.com/ITW-Sunshine/autotask-mcp/issues/59)) ([78e0f78](https://github.com/ITW-Sunshine/autotask-mcp/commit/78e0f7805e36ce9c69e0ecd3b6c7326e95d51615))
* **ci:** add configurationItemType and configurationItemCategoryID filters to search_configuration_items ([#180](https://github.com/ITW-Sunshine/autotask-mcp/issues/180)) ([9070072](https://github.com/ITW-Sunshine/autotask-mcp/commit/9070072a86d82901f4eacc878ab913a3562aaecc))
* **companies:** expose webAddress/paymentTerm/taxRegion/invoiceTemplate/taxExempt fields on update_company ([#73](https://github.com/ITW-Sunshine/autotask-mcp/issues/73)) ([e1a84cf](https://github.com/ITW-Sunshine/autotask-mcp/commit/e1a84cf61e35067428f3f67f0e1bddb4c25018f4))
* **config:** auto-detect Autotask API zone from username ([#55](https://github.com/ITW-Sunshine/autotask-mcp/issues/55)) ([#60](https://github.com/ITW-Sunshine/autotask-mcp/issues/60)) ([01a3bae](https://github.com/ITW-Sunshine/autotask-mcp/commit/01a3bae888c6cb3cb45d91afb50e83b2f0eed6c1))
* **contacts:** add autotask_update_contact tool ([#74](https://github.com/ITW-Sunshine/autotask-mcp/issues/74)) ([8a400ac](https://github.com/ITW-Sunshine/autotask-mcp/commit/8a400acc0e67a1990df891daba276ccda772c5ad))
* **contracts:** add CRUD tools for Contracts and ContractServices ([#75](https://github.com/ITW-Sunshine/autotask-mcp/issues/75)) ([bbdef76](https://github.com/ITW-Sunshine/autotask-mcp/commit/bbdef76a0dc34e66a30409825eb683fbac22dc6a))
* Initialize Taskmaster and fix build setup ([5e22e0c](https://github.com/ITW-Sunshine/autotask-mcp/commit/5e22e0cf9d51043f7d348bdba5eb66886843a8cf))
* interactive ticket card via MCP Apps (SEP-1865) ([#202](https://github.com/ITW-Sunshine/autotask-mcp/issues/202)) ([13123d9](https://github.com/ITW-Sunshine/autotask-mcp/commit/13123d9941608f0d227473b5f87e2c4d005fc98a))
* migrate to MCP SDK v2 with dual-era (backward-compatible) serving ([#219](https://github.com/ITW-Sunshine/autotask-mcp/issues/219)) ([e4b6961](https://github.com/ITW-Sunshine/autotask-mcp/commit/e4b69619d60d5b28dc1151883c655214a385a14c))
* Namespace all MCP tools with 'autotask_' prefix (v2.0.0) ([1190101](https://github.com/ITW-Sunshine/autotask-mcp/commit/119010121974b991f8f0438d1ee7eca8c990cd81))
* **projects:** add autotask_update_project tool ([#55](https://github.com/ITW-Sunshine/autotask-mcp/issues/55)) ([#57](https://github.com/ITW-Sunshine/autotask-mcp/issues/57)) ([1efeead](https://github.com/ITW-Sunshine/autotask-mcp/commit/1efeead572b60b1513bd7c773ca684eb774e36c0))
* Quote Builder - create quotes with line items from cost sheets ([#29](https://github.com/ITW-Sunshine/autotask-mcp/issues/29)) ([5da3a98](https://github.com/ITW-Sunshine/autotask-mcp/commit/5da3a98895446b051923afa3aa568f51bc423f23))
* **rate-limit:** surface Autotask 429/threshold errors as structured ([#69](https://github.com/ITW-Sunshine/autotask-mcp/issues/69), [#91](https://github.com/ITW-Sunshine/autotask-mcp/issues/91)) ([#107](https://github.com/ITW-Sunshine/autotask-mcp/issues/107)) ([3fad766](https://github.com/ITW-Sunshine/autotask-mcp/commit/3fad766ece80da8e836691ab3689e1a4f392c524))
* **registry:** publish to MCP Registry on release ([#85](https://github.com/ITW-Sunshine/autotask-mcp/issues/85)) ([5c81c5b](https://github.com/ITW-Sunshine/autotask-mcp/commit/5c81c5bfe6e86fec3a8204cd9a772fa288ab0267))
* Regular Time entry support with name resolution ([02e2e9a](https://github.com/ITW-Sunshine/autotask-mcp/commit/02e2e9a59583392f54e11d90b9511a521aa11a5d))
* releaseplease & auto deploy to azure container app ([f6779de](https://github.com/ITW-Sunshine/autotask-mcp/commit/f6779de3ddf11c287bc76a818ed42ff761ac07ce))
* releaseplease & auto deploy to azure container app ([533d315](https://github.com/ITW-Sunshine/autotask-mcp/commit/533d315205d0d7f08e20241eaf4b3c89a9309e5b))
* **search:** Add compact response format, smart defaults, and pagination ([00aa4b9](https://github.com/ITW-Sunshine/autotask-mcp/commit/00aa4b91e7329e833c60545d9d5e081f5a8f374c))
* **site-config:** add company site configuration read and update tools ([#55](https://github.com/ITW-Sunshine/autotask-mcp/issues/55)) ([#63](https://github.com/ITW-Sunshine/autotask-mcp/issues/63)) ([bb26f1a](https://github.com/ITW-Sunshine/autotask-mcp/commit/bb26f1a796496b9ec13cc50168cd731f486a4dba))
* Switch Docker publishing to GitHub Container Registry ([97fa1ac](https://github.com/ITW-Sunshine/autotask-mcp/commit/97fa1aca3226782b7110984a712fa01ee50e7915))
* **tickets:** add contactID filter to autotask_search_tickets ([#79](https://github.com/ITW-Sunshine/autotask-mcp/issues/79)) ([dc3f88a](https://github.com/ITW-Sunshine/autotask-mcp/commit/dc3f88ade9a32a0683fafb35134b30f5cb648a32))
* **tickets:** expand create/update_ticket field coverage ([#55](https://github.com/ITW-Sunshine/autotask-mcp/issues/55)) ([#58](https://github.com/ITW-Sunshine/autotask-mcp/issues/58)) ([16614ff](https://github.com/ITW-Sunshine/autotask-mcp/commit/16614ff2209e5b82d57fbc83bbfbef0cc8e24080))
* **tickets:** expose TicketHistory audit trail as MCP tools ([#95](https://github.com/ITW-Sunshine/autotask-mcp/issues/95)) ([0c74c4a](https://github.com/ITW-Sunshine/autotask-mcp/commit/0c74c4ace4e1a0e1a8de3f680033ab0c34e1ad91))
* **time-entries:** add approvalStatus filter for un-posted entries ([d27f0ab](https://github.com/ITW-Sunshine/autotask-mcp/commit/d27f0ab1fe8ba169069e3fb7de7010ead4b26636))
* **tools:** add autotask_raw_request escape hatch (security-hardened) ([#76](https://github.com/ITW-Sunshine/autotask-mcp/issues/76)) ([eb3e246](https://github.com/ITW-Sunshine/autotask-mcp/commit/eb3e246fa398a705c56e8b4f34af103b8fa66ac7))
* **tools:** add autotask_update_ticket tool ([b73c3f9](https://github.com/ITW-Sunshine/autotask-mcp/commit/b73c3f9faee84083510bcc7e07130c0577520a34)), closes [#28](https://github.com/ITW-Sunshine/autotask-mcp/issues/28)
* **tools:** expose issueType + subIssueType on autotask_update_ticket ([#110](https://github.com/ITW-Sunshine/autotask-mcp/issues/110)) ([b0bbd76](https://github.com/ITW-Sunshine/autotask-mcp/commit/b0bbd76981addc75ec7621db53ed08107a535922))
* **tools:** wire autotask_update_ticket handler ([13fe142](https://github.com/ITW-Sunshine/autotask-mcp/commit/13fe142082f04f95c2fefc6f48d51f485dab12b4))
* **worker:** serve full MCP on Cloudflare Workers via Web Standard transport ([4481c92](https://github.com/ITW-Sunshine/autotask-mcp/commit/4481c92ff32017d09d74f4b7ce136bdb7230b677))


### Bug Fixes

* add CLI bin entry and enforce test failures in CI ([10ce1c7](https://github.com/ITW-Sunshine/autotask-mcp/commit/10ce1c71324f5b301a6b41e151f187f377cd6793)), closes [#4](https://github.com/ITW-Sunshine/autotask-mcp/issues/4)
* add missing required fields to task, project note, and project creation ([df72fb4](https://github.com/ITW-Sunshine/autotask-mcp/commit/df72fb413e6e50e460a5864fd656d318ead622e1))
* **add-to-project:** call shared reusable workflow ([#90](https://github.com/ITW-Sunshine/autotask-mcp/issues/90)) ([8c75f7e](https://github.com/ITW-Sunshine/autotask-mcp/commit/8c75f7ed8f757488b258e2ef5eab2c673ace7012))
* Address GitHub issues [#3](https://github.com/ITW-Sunshine/autotask-mcp/issues/3), [#8](https://github.com/ITW-Sunshine/autotask-mcp/issues/8), [#9](https://github.com/ITW-Sunshine/autotask-mcp/issues/9) - search filtering and API endpoints ([b11256b](https://github.com/ITW-Sunshine/autotask-mcp/commit/b11256bab28df8e98f29017ebc5f20059253df65))
* allow unauthenticated tool discovery in gateway mode ([7e8bdc5](https://github.com/ITW-Sunshine/autotask-mcp/commit/7e8bdc50456e7c65856a2eeccb0f18647e950107))
* allow unauthenticated tools/list in gateway mode ([afa5120](https://github.com/ITW-Sunshine/autotask-mcp/commit/afa51208a9c1dd06625809b8339d11be65770ee2))
* **attachments:** make autotask_get_ticket_attachment actually return data ([#96](https://github.com/ITW-Sunshine/autotask-mcp/issues/96)) ([f58f83f](https://github.com/ITW-Sunshine/autotask-mcp/commit/f58f83fc2f52f9ad1d08ce81b3b62d0b314825d2))
* autotask_test_connection sets isError=true when connection fails ([#82](https://github.com/ITW-Sunshine/autotask-mcp/issues/82)) ([9166dbc](https://github.com/ITW-Sunshine/autotask-mcp/commit/9166dbcae5a559ac175e80e68ac2b0ee83fcb461))
* **autotask:** clamp per-page MaxRecords to Autotask API limit (500) ([#132](https://github.com/ITW-Sunshine/autotask-mcp/issues/132)) ([7e45e1b](https://github.com/ITW-Sunshine/autotask-mcp/commit/7e45e1bf831f888733b10c25988ba61cc169a377))
* **cache:** solve thundering herd bug resolving company mapping limits ([ffa318a](https://github.com/ITW-Sunshine/autotask-mcp/commit/ffa318a108455df1007eef46a3810764165444c4))
* **ci:** add GITHUB_TOKEN to version detection step ([99e2b29](https://github.com/ITW-Sunshine/autotask-mcp/commit/99e2b29c772e1fd80995888126b849629d8cb088))
* **ci:** add multi-platform Docker builds for arm64 support ([9e565c6](https://github.com/ITW-Sunshine/autotask-mcp/commit/9e565c6f2a046834df10b1774562867766e5a647))
* **ci:** Add proper permissions for release and security scan jobs ([d60e138](https://github.com/ITW-Sunshine/autotask-mcp/commit/d60e138684c214dcab6196cffe977fb581bc20eb))
* **ci:** bump mcp-assert reusable-workflow pin to pick up the 403 fix ([#228](https://github.com/ITW-Sunshine/autotask-mcp/issues/228)) ([437aade](https://github.com/ITW-Sunshine/autotask-mcp/commit/437aadec23fd020d4d68a3a5754c077002943bc1))
* **ci:** deploy :latest tag, force revision via env var bump ([dd31423](https://github.com/ITW-Sunshine/autotask-mcp/commit/dd3142390e20236a77bf784b79856a8c28a11f1f))
* **ci:** Disable npm publishing in semantic-release ([ae11880](https://github.com/ITW-Sunshine/autotask-mcp/commit/ae118800add292aaf5aa626aef29cc61e9d8cff9))
* **ci:** fix broken YAML in Discord notification step ([bc3b34a](https://github.com/ITW-Sunshine/autotask-mcp/commit/bc3b34a9e1479b6a2a8ddb5a0658378270a0fd71))
* **ci:** gate docker/deploy on actual release to stop :latest tag regression ([#128](https://github.com/ITW-Sunshine/autotask-mcp/issues/128)) ([72e6729](https://github.com/ITW-Sunshine/autotask-mcp/commit/72e6729636fc7045155b84d495da0f9977c648a8))
* **ci:** move Discord notification into release workflow ([113d526](https://github.com/ITW-Sunshine/autotask-mcp/commit/113d52638bada126ec8bfe081012870aaa2f053a))
* **ci:** pack MCPB bundle after semantic-release version bump ([53c952e](https://github.com/ITW-Sunshine/autotask-mcp/commit/53c952ea61c6b3f16b5f4405d5b9143e214d4b53))
* **ci:** repair MCPB pack/upload after package scope rename ([c72dd9e](https://github.com/ITW-Sunshine/autotask-mcp/commit/c72dd9e8317bad00d39389319a42a4b8aa92a07b))
* **ci:** repair MCPB pack/upload after package scope rename ([251d9e2](https://github.com/ITW-Sunshine/autotask-mcp/commit/251d9e2fbc724e286c3aad234d5534c1f8606e11))
* **ci:** replace dist file uploads with MCPB bundle in releases ([280127f](https://github.com/ITW-Sunshine/autotask-mcp/commit/280127f8f4541549b7f44fc68c0cd67807a91c5b))
* **ci:** Replace local file dependency with git dependency for autotask-node ([828bf1a](https://github.com/ITW-Sunshine/autotask-mcp/commit/828bf1abb4872ecc40c0b64ea080c6126ecee2ed))
* **ci:** Resolve npm ci prefer-offline/online conflict in Dockerfile ([b994baf](https://github.com/ITW-Sunshine/autotask-mcp/commit/b994bafa68d59b371bf3bdfefcf7406b3d4bbda8))
* **ci:** Resolve npm ci prefer-offline/online conflict in Dockerfile ([60734c8](https://github.com/ITW-Sunshine/autotask-mcp/commit/60734c8cc935b3913c3c310a1d039efdd5d41254))
* **ci:** use Node 22 in release job for semantic-release v25 compatibility ([f8d96eb](https://github.com/ITW-Sunshine/autotask-mcp/commit/f8d96eb929a87aeed702ac6192dcaeabb09020fe))
* **companies:** correct field names + add missing billing/invoice/quote fields on update_company ([#77](https://github.com/ITW-Sunshine/autotask-mcp/issues/77)) ([1ea8ea6](https://github.com/ITW-Sunshine/autotask-mcp/commit/1ea8ea6784aaf08c77c62243363097f3c7b54028))
* **companies:** correct field names + add missing billing/invoice/quote fields on update_company ([#87](https://github.com/ITW-Sunshine/autotask-mcp/issues/87)) ([7a4ef57](https://github.com/ITW-Sunshine/autotask-mcp/commit/7a4ef5726650318ecdbaea920f30e45e6d8d11d4))
* **contacts:** PUT fallback for Zone 18 PATCH 404 + userDefinedFields + raw PUT ([#133](https://github.com/ITW-Sunshine/autotask-mcp/issues/133)) ([#134](https://github.com/ITW-Sunshine/autotask-mcp/issues/134)) ([5308fb3](https://github.com/ITW-Sunshine/autotask-mcp/commit/5308fb38cfff5bad161f2526dffc4539861a3b53))
* **deploy:** authenticate GitHub Packages in one-click cloud builds ([6d3938d](https://github.com/ITW-Sunshine/autotask-mcp/commit/6d3938dd4c17fbda0757b92884870a3cc5ab72c6))
* **deploy:** bump to publish per-instance MappingService changes from [#131](https://github.com/ITW-Sunshine/autotask-mcp/issues/131) ([c1d975e](https://github.com/ITW-Sunshine/autotask-mcp/commit/c1d975ed1a81ccdc5e6325c29c272112f8b7da5b))
* **deploy:** correct ACA container name — gwp-autotask → mcpgw-prod-autotask ([3d7dc0c](https://github.com/ITW-Sunshine/autotask-mcp/commit/3d7dc0cdc01fc82482251b69f3164e308ddbd00e))
* **deploy:** one-click deploy npm-auth audit (no GitHub Packages dependency) ([920cea8](https://github.com/ITW-Sunshine/autotask-mcp/commit/920cea8ce2fabf6312477f4a8104fdb4e4c229f5))
* **deploy:** replace node_compat with nodejs_compat for Wrangler v4 ([65cb979](https://github.com/ITW-Sunshine/autotask-mcp/commit/65cb979206aa11a633843dbbbd612dfc39c96495))
* **deploy:** replace node_compat with nodejs_compat for Wrangler v4 ([e0bb78d](https://github.com/ITW-Sunshine/autotask-mcp/commit/e0bb78d414fe0ff6c2a571440c9ef63623043541)), closes [#50](https://github.com/ITW-Sunshine/autotask-mcp/issues/50)
* **deps:** bump autotask-node to main branch with endpoint semaphore fix ([35cef9c](https://github.com/ITW-Sunshine/autotask-mcp/commit/35cef9c659f9e0b399f03ccabcd9c9ea75878228))
* **deps:** consume autotask-node via GitHub Packages registry, not git tag ([#226](https://github.com/ITW-Sunshine/autotask-mcp/issues/226)) ([68ee782](https://github.com/ITW-Sunshine/autotask-mcp/commit/68ee7825adce8142b603a0244c3f3fe2571d45ff))
* **deps:** demote unused autotask-node to devDependencies and refresh vulnerable transitives ([bfce543](https://github.com/ITW-Sunshine/autotask-mcp/commit/bfce54335390ae742997ae567fcead20d3ce976f))
* **deps:** update autotask-node to compiled semaphore fix ([f282e21](https://github.com/ITW-Sunshine/autotask-mcp/commit/f282e21385e9b2640b8de7747df6070a8aecc605))
* **deps:** Update autotask-node to v2.0.6 ([1a2e08e](https://github.com/ITW-Sunshine/autotask-mcp/commit/1a2e08e3f9d808b0e424ea4c8bcc46a07727d784))
* **deps:** update autotask-node with gzip compression body fix ([62a6373](https://github.com/ITW-Sunshine/autotask-mcp/commit/62a6373960b0b5d9a45719b5ba06c233697fc88c))
* **deps:** Update package-lock.json with correct autotask-node v2.0.6 hash ([7c0ff90](https://github.com/ITW-Sunshine/autotask-mcp/commit/7c0ff90eb5623734c9f09643cccd46582d8c9568))
* **deps:** upgrade semantic-release to ^25.0.0 for github plugin compatibility ([41e88f1](https://github.com/ITW-Sunshine/autotask-mcp/commit/41e88f1651d30287a5d2a3b9ec534c0f4b606d15))
* disable broken gzip compression and fix create ID parsing ([d6889f3](https://github.com/ITW-Sunshine/autotask-mcp/commit/d6889f395b8b5327f976ac6cdda81ace4b6f6eed))
* **docker:** apk upgrade in production stage to pull OpenSSL 3.5.7-r0 ([660b98b](https://github.com/ITW-Sunshine/autotask-mcp/commit/660b98b54e627ae537875f074ed25ba8487bd38c))
* **docker:** drop arm64 platform to fix QEMU build failures ([038f21c](https://github.com/ITW-Sunshine/autotask-mcp/commit/038f21cfc547e3c915db6bb13f3324702f53b44b))
* **docker:** Fix build and runtime failures in Dockerfile ([c6e37e2](https://github.com/ITW-Sunshine/autotask-mcp/commit/c6e37e266c1eccf531247bd6110bfc7e06f75819))
* **docker:** pin npm to v10 to fix git dep install failure ([bab0dc8](https://github.com/ITW-Sunshine/autotask-mcp/commit/bab0dc8f009011286231935f1833694fb4ef9377))
* **docker:** strip npm CLI from production image ([8292d1e](https://github.com/ITW-Sunshine/autotask-mcp/commit/8292d1efaef6b8fa145a7b77b0c7183abda8bee2))
* **docs:** Add base path prefix to content links for GitHub Pages ([be4b661](https://github.com/ITW-Sunshine/autotask-mcp/commit/be4b66172c2f000e09a8d887b051d4bd2bb8ad05))
* **docs:** Use npx for Claude Code instructions instead of bundle extraction ([e5c7a01](https://github.com/ITW-Sunshine/autotask-mcp/commit/e5c7a01937ba323ce2463c2ce3c9e9c6eae65bd3))
* don't crash on missing credentials, return tool-level errors instead ([cd9294c](https://github.com/ITW-Sunshine/autotask-mcp/commit/cd9294c900350eab5f91ce6152121e5571abb88c))
* **enrich:** bound search-result name enrichment concurrency ([#169](https://github.com/ITW-Sunshine/autotask-mcp/issues/169)) ([32ad2a8](https://github.com/ITW-Sunshine/autotask-mcp/commit/32ad2a8193d297e54b3f4405477eae4e108e320b))
* **expenses:** correct weekEnding field name and required params for expense report creation ([226d38d](https://github.com/ITW-Sunshine/autotask-mcp/commit/226d38d3e1f52509421fba6fe3cfa31fca9c3046))
* fix release please ([91bd48b](https://github.com/ITW-Sunshine/autotask-mcp/commit/91bd48badf6bac2432c4db846971400cddee32ff))
* fixing an unintended changed tool use definition ([190f6c6](https://github.com/ITW-Sunshine/autotask-mcp/commit/190f6c6b0b7450bf436da5b10b1c20d877458e0d))
* **gateway:** skip autotask-node connection test in stateless mode ([67643c6](https://github.com/ITW-Sunshine/autotask-mcp/commit/67643c691d233ca0c4b760f99474203e3584a6ae))
* github repo secrets can not be start with GITHUB_ so renaming the variable in node auth ([000d2b4](https://github.com/ITW-Sunshine/autotask-mcp/commit/000d2b4f5742a7aff31534d4f92b933372399d30))
* **http:** add CORS preflight handling for browser-based MCP clients ([#70](https://github.com/ITW-Sunshine/autotask-mcp/issues/70)) ([82515e6](https://github.com/ITW-Sunshine/autotask-mcp/commit/82515e669f8fdd4864c8d4c5095fb99d93a70e93))
* **http:** fallback to GET for child entities that don't support /query ([93b21d5](https://github.com/ITW-Sunshine/autotask-mcp/commit/93b21d552e9badc336cc3fbceba52c3f1d9be4d3))
* **http:** paginate query() with POST on nextPageUrl, not GET ([#170](https://github.com/ITW-Sunshine/autotask-mcp/issues/170)) ([b0a24a2](https://github.com/ITW-Sunshine/autotask-mcp/commit/b0a24a2a2d2876920784d2510eec82137f7d81dc))
* load .env file at startup for credential configuration ([192c52c](https://github.com/ITW-Sunshine/autotask-mcp/commit/192c52c5b324ee485c07c73367f7d80da236f73d))
* **mapping:** coalesce concurrent cache refreshes and fix fetch-all fallback ([ea08f80](https://github.com/ITW-Sunshine/autotask-mcp/commit/ea08f80379898d10f1447a2399ab429151790756))
* **mapping:** honor page in searchCompanies; wire LAZY_LOADING through ([#101](https://github.com/ITW-Sunshine/autotask-mcp/issues/101)) ([#102](https://github.com/ITW-Sunshine/autotask-mcp/issues/102)) ([234f130](https://github.com/ITW-Sunshine/autotask-mcp/commit/234f130bced4a9e9199172779760fa420b952701))
* **mapping:** paginate company cache and stop caching direct-get fallbacks ([#68](https://github.com/ITW-Sunshine/autotask-mcp/issues/68)) ([b0812eb](https://github.com/ITW-Sunshine/autotask-mcp/commit/b0812eb15cf4934a49425361370122607637b526))
* **mcpb:** Add bundle signing, size reduction, and Claude Desktop compatibility ([89a4711](https://github.com/ITW-Sunshine/autotask-mcp/commit/89a471172a7486f56aadffaa8881a7ff96c87930))
* **mcpb:** Fix bundle runtime errors and add automated test harness ([c3beb22](https://github.com/ITW-Sunshine/autotask-mcp/commit/c3beb221bdacf949aa543d846188ab1fb85639d2))
* new search type object for contacts ([2d1460b](https://github.com/ITW-Sunshine/autotask-mcp/commit/2d1460bb201d6bf06c29a50f132caa7a9d8e0bb3))
* **notes:** require explicit noteType/publish, point LLM to get_field_info ([#126](https://github.com/ITW-Sunshine/autotask-mcp/issues/126)) ([#127](https://github.com/ITW-Sunshine/autotask-mcp/issues/127)) ([3a1cfb1](https://github.com/ITW-Sunshine/autotask-mcp/commit/3a1cfb1fea7c7ba08cc9b06b01828ce4ccf71333))
* **notes:** use sub-resource URL /Tickets/{id}/Notes for create_ticket_note ([#33](https://github.com/ITW-Sunshine/autotask-mcp/issues/33)) ([d9a26a0](https://github.com/ITW-Sunshine/autotask-mcp/commit/d9a26a021c34e3966b5cff6bda931f15b6372747))
* prevent dotenv stdout pollution in MCP stdio transport ([8818749](https://github.com/ITW-Sunshine/autotask-mcp/commit/8818749b2ec6979eddca0d45f7dd13a3c7c60756))
* prevent stdout pollution from autotask-node's dotenv.config() ([abc61fd](https://github.com/ITW-Sunshine/autotask-mcp/commit/abc61fdcd46f3891fe4501d226986163fe0dec95))
* **projects:** build searchProjects filters from individual params ([6e30860](https://github.com/ITW-Sunshine/autotask-mcp/commit/6e30860f3fc475df7bd3a6623856bc6330d035f6))
* quote item creation 404 and quote creation 500 errors ([#30](https://github.com/ITW-Sunshine/autotask-mcp/issues/30)) ([7f8e64f](https://github.com/ITW-Sunshine/autotask-mcp/commit/7f8e64f89350dba7962a56f2fa461b7db6af26ec))
* **release:** deploy to gwp-autotask by image digest ([#97](https://github.com/ITW-Sunshine/autotask-mcp/issues/97)) ([fe046ce](https://github.com/ITW-Sunshine/autotask-mcp/commit/fe046ce4e68808e519b4670d083b0c0da3ff886c))
* **release:** docker tag uses release output, not package.json ([#86](https://github.com/ITW-Sunshine/autotask-mcp/issues/86)) ([08c7d6a](https://github.com/ITW-Sunshine/autotask-mcp/commit/08c7d6aa2f8ff9d592d3a54cc72024c7891fefc6))
* **release:** drop @semantic-release/git to unblock releases ([#64](https://github.com/ITW-Sunshine/autotask-mcp/issues/64)) ([98171ab](https://github.com/ITW-Sunshine/autotask-mcp/commit/98171abd837da9454c66a2676642bc56bca1a3d7))
* **release:** grant id-token: write to deploy job ([#100](https://github.com/ITW-Sunshine/autotask-mcp/issues/100)) ([d89e5c0](https://github.com/ITW-Sunshine/autotask-mcp/commit/d89e5c0a3c127a7501a56b840714816a0f2b33f0))
* **release:** re-pin shared deploy workflow to d6ace52 ([#99](https://github.com/ITW-Sunshine/autotask-mcp/issues/99)) ([1dde178](https://github.com/ITW-Sunshine/autotask-mcp/commit/1dde17849e6b7d190314f9e664d5d78150cce5bc))
* rename duplicate step id 'version' to 'release-version' in docker job ([5e093cb](https://github.com/ITW-Sunshine/autotask-mcp/commit/5e093cb019b5fdc457b91c67efb807ce00207cd2))
* resolve .env relative to script location as fallback ([367eb0d](https://github.com/ITW-Sunshine/autotask-mcp/commit/367eb0d9a4bbcf0ec2b73e95ab96737145f586ac))
* resolve Docker build failure caused by prepare script running before source copy ([87b0a0c](https://github.com/ITW-Sunshine/autotask-mcp/commit/87b0a0cfde754bfe0f98980893834db6afd0eed1))
* resolve merge conflict markers from squash merges ([6c152eb](https://github.com/ITW-Sunshine/autotask-mcp/commit/6c152ebf7cb91a4fb19869d000ec6d1b93ad6a78))
* route contact updates through Companies/{companyID}/Contacts child route ([#212](https://github.com/ITW-Sunshine/autotask-mcp/issues/212)) ([ff77beb](https://github.com/ITW-Sunshine/autotask-mcp/commit/ff77beb4e8f3d152057e236ea7f8aa585726f271)), closes [#197](https://github.com/ITW-Sunshine/autotask-mcp/issues/197)
* **search:** honor filter params in 4 search tools ([#104](https://github.com/ITW-Sunshine/autotask-mcp/issues/104), [#105](https://github.com/ITW-Sunshine/autotask-mcp/issues/105)) ([#106](https://github.com/ITW-Sunshine/autotask-mcp/issues/106)) ([5f589a0](https://github.com/ITW-Sunshine/autotask-mcp/commit/5f589a000bc4d5aeb058b0ebe1be200085e59a11))
* **security:** address code scanning vulnerabilities ([9fba187](https://github.com/ITW-Sunshine/autotask-mcp/commit/9fba1879186a4c4c31482776a9a26152e163d7fe))
* **security:** enforce gateway S2S verification (gateway[#377](https://github.com/ITW-Sunshine/autotask-mcp/issues/377) parity) ([#225](https://github.com/ITW-Sunshine/autotask-mcp/issues/225)) ([f9099a8](https://github.com/ITW-Sunshine/autotask-mcp/commit/f9099a859b0e7022cfeb03d49ba396c2edf67890))
* **security:** isolate per-request credentials in gateway mode ([#66](https://github.com/ITW-Sunshine/autotask-mcp/issues/66)) ([c7f582c](https://github.com/ITW-Sunshine/autotask-mcp/commit/c7f582caea32977212293ea6de1ea0c93517ca00))
* **security:** SHA-pin auto-add-to-project.yml [@main](https://github.com/main) -&gt; [@6ae1533dd72f](https://github.com/6ae1533dd72f) (warden C-4) ([#187](https://github.com/ITW-Sunshine/autotask-mcp/issues/187)) ([f1bb155](https://github.com/ITW-Sunshine/autotask-mcp/commit/f1bb1559ad5e4a116dbe9941c5583cba99a76e04))
* **security:** update brace-expansion for CVE-2025-5889 ([#19](https://github.com/ITW-Sunshine/autotask-mcp/issues/19)) ([619d8d8](https://github.com/ITW-Sunshine/autotask-mcp/commit/619d8d88553f95263f84bdb4dc9be5ff95d3230c))
* **security:** update Docker base image for npm CVEs ([#16](https://github.com/ITW-Sunshine/autotask-mcp/issues/16)) ([cef86ba](https://github.com/ITW-Sunshine/autotask-mcp/commit/cef86ba8f5397f04612f54c128e373e01392b09c)), closes [#13](https://github.com/ITW-Sunshine/autotask-mcp/issues/13)
* **security:** update lodash for CVE-2025-13465 ([#18](https://github.com/ITW-Sunshine/autotask-mcp/issues/18)) ([99e856d](https://github.com/ITW-Sunshine/autotask-mcp/commit/99e856d114fed5e4665bb69600d18977d7f67762))
* **security:** update MCP SDK for CVE-2026-0621, CVE-2025-66414 ([#17](https://github.com/ITW-Sunshine/autotask-mcp/issues/17)) ([33a7d81](https://github.com/ITW-Sunshine/autotask-mcp/commit/33a7d81ce1181aaad74017ab32ba8918411c3e4e))
* **security:** update tar for CVE-2026-23950, CVE-2026-23745 ([6890ca6](https://github.com/ITW-Sunshine/autotask-mcp/commit/6890ca60688db539020eac3c02132ebc8858463c)), closes [#12](https://github.com/ITW-Sunshine/autotask-mcp/issues/12)
* **service:** replace autotask-node SDK with native fetch throughout ([6357573](https://github.com/ITW-Sunshine/autotask-mcp/commit/6357573e1c3ae9ed1f08143f3b25a812072c8adc))
* set released=true only when semantic-release creates a new version ([cb81aeb](https://github.com/ITW-Sunshine/autotask-mcp/commit/cb81aeb8f74351a0e4b1592a210456af971ca9b0))
* support queueID and priority filters in autotask_search_tickets ([#209](https://github.com/ITW-Sunshine/autotask-mcp/issues/209)) ([ea0aff5](https://github.com/ITW-Sunshine/autotask-mcp/commit/ea0aff57a5a80c6a07d6c02f74f65f7c2cba4a19)), closes [#208](https://github.com/ITW-Sunshine/autotask-mcp/issues/208)
* surface Autotask API validation errors instead of generic 500 ([#32](https://github.com/ITW-Sunshine/autotask-mcp/issues/32)) ([b8e2453](https://github.com/ITW-Sunshine/autotask-mcp/commit/b8e2453fb13208dcbb022c6c028c76d009c51414))
* sync manifest.json version from package.json at pack time ([c7a9724](https://github.com/ITW-Sunshine/autotask-mcp/commit/c7a97241777c47f28bfcaf3cb4a4f6392d68d3b3))
* task creation endpoint, ticket notes, field info + phase tools ([9d7fa42](https://github.com/ITW-Sunshine/autotask-mcp/commit/9d7fa42a1ce5d9e08651ca5b524a86d9b29a404b)), closes [#46](https://github.com/ITW-Sunshine/autotask-mcp/issues/46) [#47](https://github.com/ITW-Sunshine/autotask-mcp/issues/47)
* **test:** Run all MCPB tests in single server session to avoid rate limits ([7b425cf](https://github.com/ITW-Sunshine/autotask-mcp/commit/7b425cfbbba7a0ceeb0d6681dc84fb4a22ea421a))
* **tests:** Resolve ESM compatibility and rewrite mapping tests ([a294a7c](https://github.com/ITW-Sunshine/autotask-mcp/commit/a294a7c390a5ae56b70c269f5f6aaf0c3ff224e5))
* **tests:** update expense item test assertions to match refactored API ([9affc62](https://github.com/ITW-Sunshine/autotask-mcp/commit/9affc62d0769cb3ce7d079f1fbfae0e3c694ff2b)), closes [#27](https://github.com/ITW-Sunshine/autotask-mcp/issues/27)
* **tests:** update expense item tests to match implemented API signatures ([7fac014](https://github.com/ITW-Sunshine/autotask-mcp/commit/7fac014eedcbc27bf9753e719b042163e770f5ff))
* **tests:** update test expectations for implemented billing code methods ([1833d02](https://github.com/ITW-Sunshine/autotask-mcp/commit/1833d02fd217d82348167067dd43d938b40f31c3))
* **tickets:** update autotask-node lockfile with PATCH collection endpoint fix ([4156635](https://github.com/ITW-Sunshine/autotask-mcp/commit/4156635778bbd5be427bb55c8df3472ae6a62ca5))
* **tickets:** update autotask-node to PATCH collection endpoint fix ([0cf01d8](https://github.com/ITW-Sunshine/autotask-mcp/commit/0cf01d87b1063d6b27a98990a297de045fe17679))
* **tickets:** use fetch-based PATCH /Tickets (body has id) — fixes 405 ([29dc8b6](https://github.com/ITW-Sunshine/autotask-mcp/commit/29dc8b6042bdccef9641045a9f245faa8d0097b8))
* **tickets:** use noteq operator so open ticket search excludes Complete ([#211](https://github.com/ITW-Sunshine/autotask-mcp/issues/211)) ([f625398](https://github.com/ITW-Sunshine/autotask-mcp/commit/f6253989866205370adcb92c04ee3e83a99419fe)), closes [#193](https://github.com/ITW-Sunshine/autotask-mcp/issues/193)
* **tickets:** use PATCH instead of PUT for updateTicket ([c0a0b7b](https://github.com/ITW-Sunshine/autotask-mcp/commit/c0a0b7bf046831982a39e699857b278474258588))
* **time-entries:** bypass broken list() filter — call /TimeEntries/query directly ([31ed876](https://github.com/ITW-Sunshine/autotask-mcp/commit/31ed8760e545d04cd126263035ae114c30fd60f9))
* **time-entries:** bypass broken list() filter by calling /TimeEntries/query directly ([146e94d](https://github.com/ITW-Sunshine/autotask-mcp/commit/146e94d0b415c339d5201071e2e56ff1b6d4429c)), closes [#53](https://github.com/ITW-Sunshine/autotask-mcp/issues/53)
* tool updates to use first & last name for searching contacts ([4011861](https://github.com/ITW-Sunshine/autotask-mcp/commit/4011861e271c7d4f2d98491c3995ebc3a886e015))
* **tools:** remove duplicate tool definitions and dispatch entries ([#78](https://github.com/ITW-Sunshine/autotask-mcp/issues/78)) ([d21647c](https://github.com/ITW-Sunshine/autotask-mcp/commit/d21647c9661ba7bde08658750bf06a663961170c))
* **tools:** return explicit not-found errors to prevent LLM hallucination ([#31](https://github.com/ITW-Sunshine/autotask-mcp/issues/31)) ([7f628db](https://github.com/ITW-Sunshine/autotask-mcp/commit/7f628db8991954f3fbaadeef796b7521702532cd))
* Tuned for personalized company use ([ffa59c7](https://github.com/ITW-Sunshine/autotask-mcp/commit/ffa59c77a5ac6e9cb80c4ed2b70a7782a7761464))
* upgrade autotask-node to v2.0.3 (removes dotenv dependency) ([1a5727b](https://github.com/ITW-Sunshine/autotask-mcp/commit/1a5727b709a84a3741adf15b51f26502d9a4c5c7))
* upgrade autotask-node to v2.0.4 (graceful logger) ([213db40](https://github.com/ITW-Sunshine/autotask-mcp/commit/213db40377852ab3dfe6971daf57cbf9f71f5e02))
* upgrade autotask-node to v2.0.5 (stderr-only logging) ([a01588b](https://github.com/ITW-Sunshine/autotask-mcp/commit/a01588b1144bcb2adfae44c102dd7879225000c3))
* use autotask-node v2.1.0 parent-child URL pattern for note/time entry creates ([6397094](https://github.com/ITW-Sunshine/autotask-mcp/commit/6397094fad52f2afef72f0f92d4e523af65b1f1a))
* use block scalar for PRE_VERSION capture to avoid shell quoting issue ([50057d2](https://github.com/ITW-Sunshine/autotask-mcp/commit/50057d2b9a8afdbe795314a6ad38850220c1065c))
* use correct parent-child URL patterns for child entity creation ([#24](https://github.com/ITW-Sunshine/autotask-mcp/issues/24)) ([47f2a75](https://github.com/ITW-Sunshine/autotask-mcp/commit/47f2a75b16de3af6b0f7581079f22fde575fe9d9))
* use stateless per-request server pattern for HTTP transport ([e8c6326](https://github.com/ITW-Sunshine/autotask-mcp/commit/e8c6326e3bc26aa6eba773b298ae4a72336b8ba5))
* variable setting for npm packages ([e104816](https://github.com/ITW-Sunshine/autotask-mcp/commit/e1048166d5fa121d4616c539ca59107afb46de8a))
* **version:** report actual package.json version in serverInfo + /health ([#94](https://github.com/ITW-Sunshine/autotask-mcp/issues/94)) ([#103](https://github.com/ITW-Sunshine/autotask-mcp/issues/103)) ([5881577](https://github.com/ITW-Sunshine/autotask-mcp/commit/58815773526b3b332db9edaf9fba3bd87a54ceb6))


### Performance Improvements

* trim tool descriptions for token reduction ([#92](https://github.com/ITW-Sunshine/autotask-mcp/issues/92)) ([b42c46e](https://github.com/ITW-Sunshine/autotask-mcp/commit/b42c46e4c7ced6da39e695d1901863e5b04efecf))


### Reverts

* remove arm64 platform and QEMU, keep linux/amd64 only ([d2f4d6a](https://github.com/ITW-Sunshine/autotask-mcp/commit/d2f4d6a19eb729e74dd48cf6cb38c477b3f00afe))

## [Unreleased]

### Added

- **tickets:** expanded field coverage on `autotask_create_ticket` and `autotask_update_ticket`. Both tools now accept `ticketCategory`, `ticketType`, `issueType`, `subIssueType`, `source`, `billingCodeID`, `queueID`, `serviceLevelAgreementID`, `estimatedHours`, `projectID`, `ticketAdditionalContacts`, `resolution`, and `userDefinedFields` (REST-native `{name, value}[]` shape). `autotask_update_ticket` is now exposed as a first-class tool. (#55)

## [2.7.3](https://github.com/wyre-technology/autotask-mcp/compare/v2.7.2...v2.7.3) (2026-02-23)


### Bug Fixes

* rename duplicate step id 'version' to 'release-version' in docker job ([5e093cb](https://github.com/wyre-technology/autotask-mcp/commit/5e093cb019b5fdc457b91c67efb807ce00207cd2))

## [2.7.2](https://github.com/wyre-technology/autotask-mcp/compare/v2.7.1...v2.7.2) (2026-02-17)


### Bug Fixes

* **docker:** drop arm64 platform to fix QEMU build failures ([038f21c](https://github.com/wyre-technology/autotask-mcp/commit/038f21cfc547e3c915db6bb13f3324702f53b44b))

## [2.7.1](https://github.com/wyre-technology/autotask-mcp/compare/v2.7.0...v2.7.1) (2026-02-15)


### Bug Fixes

* use stateless per-request server pattern for HTTP transport ([e8c6326](https://github.com/wyre-technology/autotask-mcp/commit/e8c6326e3bc26aa6eba773b298ae4a72336b8ba5))

# [2.7.0](https://github.com/wyre-technology/autotask-mcp/compare/v2.6.0...v2.7.0) (2026-02-13)


### Features

* add DigitalOcean and Cloudflare deploy infrastructure and badges ([b68bad5](https://github.com/wyre-technology/autotask-mcp/commit/b68bad5b7406ea17a0de4fe16ce65d75f7cb14d0))

# [2.6.0](https://github.com/wyre-technology/autotask-mcp/compare/v2.5.3...v2.6.0) (2026-02-10)


### Bug Fixes

* **security:** address code scanning vulnerabilities ([9fba187](https://github.com/wyre-technology/autotask-mcp/commit/9fba1879186a4c4c31482776a9a26152e163d7fe))
* use autotask-node v2.1.0 parent-child URL pattern for note/time entry creates ([6397094](https://github.com/wyre-technology/autotask-mcp/commit/6397094fad52f2afef72f0f92d4e523af65b1f1a))
* use correct parent-child URL patterns for child entity creation ([#24](https://github.com/wyre-technology/autotask-mcp/issues/24)) ([47f2a75](https://github.com/wyre-technology/autotask-mcp/commit/47f2a75b16de3af6b0f7581079f22fde575fe9d9))


### Features

* Add gateway mode for hosted MCP deployments ([14d5682](https://github.com/wyre-technology/autotask-mcp/commit/14d568223c9269de2d5e3e2eba5056351ca3e82d))
* **billing:** Add BillingItems and BillingItemApprovalLevels support ([4c88034](https://github.com/wyre-technology/autotask-mcp/commit/4c880348d7a930b5277a810b89a3c54cddedb509)), closes [#21](https://github.com/wyre-technology/autotask-mcp/issues/21)
* **time-entries:** add approvalStatus filter for un-posted entries ([d27f0ab](https://github.com/wyre-technology/autotask-mcp/commit/d27f0ab1fe8ba169069e3fb7de7010ead4b26636)), closes [#21](https://github.com/wyre-technology/autotask-mcp/issues/21)

## [Unreleased] - Wyre Technology Fork

### Added

* **Gateway Mode**: Support for hosted MCP Gateway deployments with header-based credential injection
  - New `AUTH_MODE` environment variable (`env` or `gateway`)
  - Per-request credential extraction from `X-API-Key`, `X-API-Secret`, `X-Integration-Code` headers
  - Health endpoint now reports `authMode` in gateway mode
* **Migration Guide**: Documentation for migrating from local to hosted deployment (`docs/MIGRATION_GUIDE.md`)

### Changed

* Docker image registry changed to `ghcr.io/wyre-technology/autotask-mcp`
* GitHub repository moved to `wyre-technology/autotask-mcp`
* Container labels updated for Wyre Technology branding

---

## [2.7.2](https://github.com/asachs01/autotask-mcp/compare/v2.7.1...v2.7.2) (2026-02-10)


### Bug Fixes

* use autotask-node v2.1.0 parent-child URL pattern for note/time entry creates ([6397094](https://github.com/asachs01/autotask-mcp/commit/6397094fad52f2afef72f0f92d4e523af65b1f1a))

## [2.7.1](https://github.com/asachs01/autotask-mcp/compare/v2.7.0...v2.7.1) (2026-02-10)


### Bug Fixes

* use correct parent-child URL patterns for child entity creation ([#24](https://github.com/asachs01/autotask-mcp/issues/24)) ([47f2a75](https://github.com/asachs01/autotask-mcp/commit/47f2a75b16de3af6b0f7581079f22fde575fe9d9))

# [2.7.0](https://github.com/asachs01/autotask-mcp/compare/v2.6.1...v2.7.0) (2026-02-06)


### Features

* **time-entries:** add approvalStatus filter for un-posted entries ([d27f0ab](https://github.com/asachs01/autotask-mcp/commit/d27f0ab1fe8ba169069e3fb7de7010ead4b26636)), closes [#21](https://github.com/asachs01/autotask-mcp/issues/21)

## [2.6.1](https://github.com/asachs01/autotask-mcp/compare/v2.6.0...v2.6.1) (2026-02-05)


### Bug Fixes

* **security:** address code scanning vulnerabilities ([9fba187](https://github.com/asachs01/autotask-mcp/commit/9fba1879186a4c4c31482776a9a26152e163d7fe))

# [2.6.0](https://github.com/asachs01/autotask-mcp/compare/v2.5.3...v2.6.0) (2026-02-05)


### Features

* **billing:** Add BillingItems and BillingItemApprovalLevels support ([4c88034](https://github.com/asachs01/autotask-mcp/commit/4c880348d7a930b5277a810b89a3c54cddedb509)), closes [#21](https://github.com/asachs01/autotask-mcp/issues/21)

## [Unreleased]

### Features

* **time-entries:** Add `approvalStatus` filter to find un-posted time entries ([#21](https://github.com/asachs01/autotask-mcp/issues/21))
  - Use `approvalStatus: "unapproved"` to find labor items not yet posted
  - Use `approvalStatus: "approved"` to find already-posted entries
  - Also added `billable` filter for billable/non-billable filtering
  - Added `billingApprovalDateTime`, `billingApprovalLevelMostRecent`, `billingApprovalResourceID` to TimeEntry interface

### Security

* **deps:** Add npm override for @isaacs/brace-expansion@^5.0.1 to fix CVE-2026-25547
* **docker:** Add explicit npm update in Dockerfile to fix base image CVEs (CVE-2026-24842, CVE-2026-0775, CVE-2026-23950, CVE-2026-23745, CVE-2025-64756)

## [2.5.3](https://github.com/asachs01/autotask-mcp/compare/v2.5.2...v2.5.3) (2026-01-27)


### Security

* **deps:** Update @modelcontextprotocol/sdk to 1.25.3 for CVE-2026-0621 (ReDoS) and CVE-2025-66414 (DNS rebinding)
* **deps:** Add npm override for tar@^7.0.0 to fix CVE-2026-23950 and CVE-2026-23745 (arbitrary file overwrite)
* **deps:** Add npm override for lodash@^4.17.23 to fix CVE-2025-13465 (prototype pollution)
* **deps:** Add npm override for brace-expansion@^2.0.1 to fix CVE-2025-5889 (ReDoS)
* **deps:** Add npm override for diff@^7.0.0 to fix CVE-2026-24001 (jsdiff vulnerability)
* **docker:** Update base image from node:20-alpine to node:22-alpine for CVE-2025-64756 (glob) and CVE-2024-21538 (cross-spawn)

## [2.5.2](https://github.com/asachs01/autotask-mcp/compare/v2.5.1...v2.5.2) (2026-01-24)


### Bug Fixes

* **docs:** Use npx for Claude Code instructions instead of bundle extraction ([e5c7a01](https://github.com/asachs01/autotask-mcp/commit/e5c7a01937ba323ce2463c2ce3c9e9c6eae65bd3))

## [2.5.1](https://github.com/asachs01/autotask-mcp/compare/v2.5.0...v2.5.1) (2026-01-24)


### Bug Fixes

* **docs:** Add base path prefix to content links for GitHub Pages ([be4b661](https://github.com/asachs01/autotask-mcp/commit/be4b66172c2f000e09a8d887b051d4bd2bb8ad05))

# [2.5.0](https://github.com/asachs01/autotask-mcp/compare/v2.4.0...v2.5.0) (2026-01-24)


### Features

* **docs:** Add Astro Starlight documentation site with prompt examples and GitHub Pages deployment ([71a5a88](https://github.com/asachs01/autotask-mcp/commit/71a5a88))


### Code Refactoring

* Simplify codebase with dispatch table, schema extraction, and DRY patterns ([c1eff86](https://github.com/asachs01/autotask-mcp/commit/c1eff86))
  - Extract 39 tool schemas to declarative tool.definitions.ts
  - Replace 300-line switch with dispatch table Map
  - Merge enhanced handler into base handler (single class)
  - Generic note methods (9 methods → 3 generic + 9 thin wrappers)
  - Simplify MappingService singleton to cached-promise pattern
  - Delete unused wrapper.ts and dead code
  - tool.handler.ts reduced from 1,616 → 445 lines (72%)

# [2.4.0](https://github.com/asachs01/autotask-mcp/compare/v2.3.4...v2.4.0) (2026-01-24)


### Features

* **search:** Add compact response format, smart defaults, and pagination ([00aa4b9](https://github.com/asachs01/autotask-mcp/commit/00aa4b91e7329e833c60545d9d5e081f5a8f374c))

## [2.3.4](https://github.com/asachs01/autotask-mcp/compare/v2.3.3...v2.3.4) (2026-01-24)


### Bug Fixes

* **test:** Run all MCPB tests in single server session to avoid rate limits ([7b425cf](https://github.com/asachs01/autotask-mcp/commit/7b425cfbbba7a0ceeb0d6681dc84fb4a22ea421a))

## [2.3.3](https://github.com/asachs01/autotask-mcp/compare/v2.3.2...v2.3.3) (2026-01-24)


### Bug Fixes

* **mcpb:** Fix bundle runtime errors and add automated test harness ([c3beb22](https://github.com/asachs01/autotask-mcp/commit/c3beb221bdacf949aa543d846188ab1fb85639d2))

## [2.3.2](https://github.com/asachs01/autotask-mcp/compare/v2.3.1...v2.3.2) (2026-01-23)


### Bug Fixes

* **mcpb:** Add bundle signing, size reduction, and Claude Desktop compatibility ([89a4711](https://github.com/asachs01/autotask-mcp/commit/89a471172a7486f56aadffaa8881a7ff96c87930))

## [2.3.1](https://github.com/asachs01/autotask-mcp/compare/v2.3.0...v2.3.1) (2026-01-23)


### Bug Fixes

* **docker:** Fix build and runtime failures in Dockerfile ([c6e37e2](https://github.com/asachs01/autotask-mcp/commit/c6e37e266c1eccf531247bd6110bfc7e06f75819))

# [2.3.0](https://github.com/asachs01/autotask-mcp/compare/v2.2.13...v2.3.0) (2026-01-23)


### Features

* Add picklist discovery tools and elicitation support ([93c68f2](https://github.com/asachs01/autotask-mcp/commit/93c68f20acf31c0a8cc661689f820bf7e3518393))

## [2.2.13](https://github.com/asachs01/autotask-mcp/compare/v2.2.12...v2.2.13) (2026-01-23)


### Bug Fixes

* **deps:** Update package-lock.json with correct autotask-node v2.0.6 hash ([7c0ff90](https://github.com/asachs01/autotask-mcp/commit/7c0ff90eb5623734c9f09643cccd46582d8c9568))

## [2.2.12](https://github.com/asachs01/autotask-mcp/compare/v2.2.11...v2.2.12) (2026-01-23)


### Bug Fixes

* **deps:** Update autotask-node to v2.0.6 ([1a2e08e](https://github.com/asachs01/autotask-mcp/commit/1a2e08e3f9d808b0e424ea4c8bcc46a07727d784))

## [2.2.11](https://github.com/asachs01/autotask-mcp/compare/v2.2.10...v2.2.11) (2026-01-23)


### Bug Fixes

* upgrade autotask-node to v2.0.4 (graceful logger) ([213db40](https://github.com/asachs01/autotask-mcp/commit/213db40377852ab3dfe6971daf57cbf9f71f5e02))
* upgrade autotask-node to v2.0.5 (stderr-only logging) ([a01588b](https://github.com/asachs01/autotask-mcp/commit/a01588b1144bcb2adfae44c102dd7879225000c3))

## [2.2.10](https://github.com/asachs01/autotask-mcp/compare/v2.2.9...v2.2.10) (2026-01-23)


### Bug Fixes

* **ci:** add GITHUB_TOKEN to version detection step ([99e2b29](https://github.com/asachs01/autotask-mcp/commit/99e2b29c772e1fd80995888126b849629d8cb088))

## [2.2.9](https://github.com/asachs01/autotask-mcp/compare/v2.2.8...v2.2.9) (2026-01-23)


### Bug Fixes

* resolve .env relative to script location as fallback ([367eb0d](https://github.com/asachs01/autotask-mcp/commit/367eb0d9a4bbcf0ec2b73e95ab96737145f586ac))

## [2.2.8](https://github.com/asachs01/autotask-mcp/compare/v2.2.7...v2.2.8) (2026-01-23)


### Bug Fixes

* load .env file at startup for credential configuration ([192c52c](https://github.com/asachs01/autotask-mcp/commit/192c52c5b324ee485c07c73367f7d80da236f73d))

## [2.2.7](https://github.com/asachs01/autotask-mcp/compare/v2.2.6...v2.2.7) (2026-01-23)


### Bug Fixes

* don't crash on missing credentials, return tool-level errors instead ([cd9294c](https://github.com/asachs01/autotask-mcp/commit/cd9294c900350eab5f91ce6152121e5571abb88c))

## [2.2.6](https://github.com/asachs01/autotask-mcp/compare/v2.2.5...v2.2.6) (2026-01-23)


### Bug Fixes

* **ci:** pack MCPB bundle after semantic-release version bump ([53c952e](https://github.com/asachs01/autotask-mcp/commit/53c952ea61c6b3f16b5f4405d5b9143e214d4b53))

## [2.2.5](https://github.com/asachs01/autotask-mcp/compare/v2.2.4...v2.2.5) (2026-01-23)


### Bug Fixes

* sync manifest.json version from package.json at pack time ([c7a9724](https://github.com/asachs01/autotask-mcp/commit/c7a97241777c47f28bfcaf3cb4a4f6392d68d3b3))

## [2.2.4](https://github.com/asachs01/autotask-mcp/compare/v2.2.3...v2.2.4) (2026-01-23)


### Bug Fixes

* upgrade autotask-node to v2.0.3 (removes dotenv dependency) ([1a5727b](https://github.com/asachs01/autotask-mcp/commit/1a5727b709a84a3741adf15b51f26502d9a4c5c7))

## [2.2.3](https://github.com/asachs01/autotask-mcp/compare/v2.2.2...v2.2.3) (2026-01-23)


### Bug Fixes

* prevent stdout pollution from autotask-node's dotenv.config() ([abc61fd](https://github.com/asachs01/autotask-mcp/commit/abc61fdcd46f3891fe4501d226986163fe0dec95))

## [2.2.2](https://github.com/asachs01/autotask-mcp/compare/v2.2.1...v2.2.2) (2026-01-23)


### Bug Fixes

* prevent dotenv stdout pollution in MCP stdio transport ([8818749](https://github.com/asachs01/autotask-mcp/commit/8818749b2ec6979eddca0d45f7dd13a3c7c60756))

## [2.2.1](https://github.com/asachs01/autotask-mcp/compare/v2.2.0...v2.2.1) (2026-01-23)


### Bug Fixes

* **ci:** replace dist file uploads with MCPB bundle in releases ([280127f](https://github.com/asachs01/autotask-mcp/commit/280127f8f4541549b7f44fc68c0cd67807a91c5b))

# [2.2.0](https://github.com/asachs01/autotask-mcp/compare/v2.1.0...v2.2.0) (2026-01-23)


### Features

* add MCPB (MCP Bundle) packaging for desktop distribution ([e7601b1](https://github.com/asachs01/autotask-mcp/commit/e7601b1d158c261a6607530f59267dff99b06ba8))

# [2.1.0](https://github.com/asachs01/autotask-mcp/compare/v2.0.3...v2.1.0) (2026-01-23)


### Features

* add HTTP Streamable transport for remote MCP access ([2d31853](https://github.com/asachs01/autotask-mcp/commit/2d3185348cb4387c5726892bb15d9c432279afa3)), closes [#7](https://github.com/asachs01/autotask-mcp/issues/7)

## [2.0.3](https://github.com/asachs01/autotask-mcp/compare/v2.0.2...v2.0.3) (2026-01-23)


### Bug Fixes

* add CLI bin entry and enforce test failures in CI ([10ce1c7](https://github.com/asachs01/autotask-mcp/commit/10ce1c71324f5b301a6b41e151f187f377cd6793)), closes [#4](https://github.com/asachs01/autotask-mcp/issues/4) [#4](https://github.com/asachs01/autotask-mcp/issues/4)

## [2.0.2](https://github.com/asachs01/autotask-mcp/compare/v2.0.1...v2.0.2) (2026-01-23)


### Bug Fixes

* **tests:** Resolve ESM compatibility and rewrite mapping tests ([a294a7c](https://github.com/asachs01/autotask-mcp/commit/a294a7c390a5ae56b70c269f5f6aaf0c3ff224e5))

## [2.0.1](https://github.com/asachs01/autotask-mcp/compare/v2.0.0...v2.0.1) (2026-01-21)


### Bug Fixes

* **ci:** Add proper permissions for release and security scan jobs ([d60e138](https://github.com/asachs01/autotask-mcp/commit/d60e138684c214dcab6196cffe977fb581bc20eb))
* **ci:** Disable npm publishing in semantic-release ([ae11880](https://github.com/asachs01/autotask-mcp/commit/ae118800add292aaf5aa626aef29cc61e9d8cff9))
* **ci:** Replace local file dependency with git dependency for autotask-node ([828bf1a](https://github.com/asachs01/autotask-mcp/commit/828bf1abb4872ecc40c0b64ea080c6126ecee2ed)), closes [asachs01/autotask-node#v2](https://github.com/asachs01/autotask-node/issues/v2)

# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Changed
- **License**: Changed from MIT to Apache 2.0

### Added
- **CLA**: Added Contributor License Agreement for contributors

## [2.0.0] - 2026-01-21

### Breaking Changes
- **Tool Namespacing**: All 35 MCP tools now use `autotask_` prefix to prevent naming collisions when multiple MCP servers are connected
  - `search_companies` → `autotask_search_companies`
  - `create_ticket` → `autotask_create_ticket`
  - `test_connection` → `autotask_test_connection`
  - All other tools follow the same pattern: `autotask_<original_name>`
- **Migration Required**: Update all tool calls in your MCP client configuration to use the new namespaced names

### Changed
- All tool definitions in `tool.handler.ts` updated with `autotask_` prefix
- Documentation updated to reflect new tool names

## [1.0.2] - 2026-01-21

### Fixed
- **Issue #9: Inaccurate Endpoints, Excessive Calls**: Upgraded to `autotask-node` v2.0.2 which fixes the critical `maxRecords` casing bug. The Autotask REST API is case-sensitive and was silently ignoring `MaxRecords` (uppercase M), causing all records to be returned instead of paginated results.
- **Issue #8: Claude Desktop and Docker unable to return results**: Added `searchTerm` → filter transformation for Companies, Contacts, and Resources
  - Company searches now filter on `companyName` field instead of fetching all companies
  - Contact searches now filter across `firstName`, `lastName`, and `emailAddress` fields
  - Resource searches now filter across `email`, `firstName`, and `lastName` fields
  - When searching with `searchTerm`, limits pagination to 100 results for efficiency
- **Issue #3: Autotask MCP out of sync with REST schema**: Fixed by upgrading to `autotask-node` v2.0.2 which corrects:
  - `MaxRecords` → `maxRecords` (lowercase m) across all 214 entity files
  - Proper POST `/query` endpoint usage for all list operations

### Changed
- **Dependency Upgrade**: Updated `autotask-node` to v2.0.2 with critical pagination fix
- **Search Efficiency**: When `searchTerm` is provided, searches return filtered results directly from API instead of paginating through all records

### Fixed (Previous)
- **🚨 CRITICAL DATA ACCURACY FIX**: Implemented pagination-by-default to eliminate massive ticket undercounts
  - **Root Cause**: Default page size was limited to 25-50 tickets, causing severe data accuracy issues
  - **Solution**: All search tools now paginate through ALL results by default for complete datasets
  - **Impact**: Fixes undercounting from 26 tickets to actual counts (e.g., 97+ tickets)
  - **User Control**: Only specify `pageSize` parameter when you actually want to limit results
- **CRITICAL: Massive Ticket Undercount**: Fixed automatic company filter that was severely limiting ticket search results (was showing only ~10 tickets instead of 97+)
- **Critical Unassigned Ticket Search Issue**: Fixed inability to search for unassigned tickets that was causing discrepancies between UI and API results
- **Parameter Mapping Issue**: Fixed `companyID` to `companyId` parameter mapping in `search_tickets` tool handler
- Enhanced ticket filtering logic to properly handle all filter parameters including assignment status

### Changed
- **Default Behavior**: `search_tickets` and all search tools now return complete datasets via automatic pagination
- **Performance**: Increased page size to 500 tickets per API request for efficiency while paginating
- **Safety**: Added pagination safety limit of 100 pages (50,000 tickets) to prevent infinite loops
- **Tool Descriptions**: Updated all search tool descriptions to clarify pagination-by-default behavior
- **Status Filtering**: Improved open ticket definition (status < 5) for accurate filtering

### Added
- **Data Accuracy Guarantee**: All search operations now provide complete, paginated results by default
- **Enhanced ID-to-Name Mapping**: Comprehensive mapping service with intelligent caching
  - New tools: `get_company_name`, `get_resource_name`, `get_mapping_cache_stats`, `clear_mapping_cache`, `preload_mapping_cache`
  - Automatic enhancement of search results with `_enhanced` field containing resolved names
  - 30-minute cache expiry with graceful fallback for missing data
- **Pagination Testing**: Added test scripts to verify complete data retrieval (`npm run test:pagination`)
- **Unassigned Ticket Support**: Added `unassigned` boolean parameter to `search_tickets` tool to search for tickets without assigned resources
- **Enhanced Tool Handler**: `EnhancedAutotaskToolHandler` with automatic ID-to-name resolution

### Fixed (Additional)
- **CRITICAL: Incomplete Company/Resource Mapping**: Fixed mapping cache that was limited to 500 records, causing "Customer 624" style names instead of proper company names
- **All Search Methods Now Complete**: Applied pagination-by-default to `searchCompanies`, `searchContacts`, and `searchResources` to ensure mapping cache includes ALL entities
- **Graceful Mapping Fallback**: Enhanced mapping service to not throw errors on cache failures, allowing direct API lookups as fallback

## [1.1.1] - 2025-06-10

### Fixed
- **Critical**: Resolved "result exceeds maximum length" errors in ticket searches by implementing aggressive data optimization
- Limited ticket search results to maximum 3 tickets per query to stay under 1MB MCP response limit
- Reduced ticket data from 76 fields (~2KB per ticket) to 18 essential fields (~685 characters per ticket)
- Added service-level result limiting as safety measure since Autotask API may ignore pageSize parameter
- Improved null handling in ticket data optimization to prevent runtime errors

### Changed
- Updated `search_tickets` tool description to clarify field limitations and recommend `get_ticket_details` for full data
- Reduced maximum pageSize for ticket searches from 100 to 3 due to API response size constraints
- Enhanced ticket data truncation with clear indicators to use `get_ticket_details` for full content

### Added
- N/A

### Fixed
- N/A

## [1.1.0] - 2024-12-10

### Added
**Phase 1: High-Priority Entity Support**
- **Notes Management**: Support for ticket, project, and company notes
  - New tools: `get_ticket_note`, `search_ticket_notes`, `create_ticket_note`
  - New tools: `get_project_note`, `search_project_notes`, `create_project_note`
  - New tools: `get_company_note`, `search_company_notes`, `create_company_note`
- **Attachments Management**: Support for ticket attachments
  - New tools: `get_ticket_attachment`, `search_ticket_attachments`
- **Expense Management**: Support for expense reports
  - New tools: `get_expense_report`, `search_expense_reports`, `create_expense_report`
- **Quotes Management**: Support for sales quotes
  - New tools: `get_quote`, `search_quotes`, `create_quote`
- **Extended Type Definitions**: New interfaces for all supported entities
  - `AutotaskNote`, `AutotaskTicketNote`, `AutotaskProjectNote`, `AutotaskCompanyNote`
  - `AutotaskAttachment`, `AutotaskTicketAttachment`
  - `AutotaskExpenseReport`, `AutotaskExpenseItem`
  - `AutotaskQuote`, `AutotaskBillingCode`, `AutotaskDepartment`
  - Extended query options with `AutotaskQueryOptionsExtended`
- **Comprehensive Testing**: Full test coverage for all new entity methods

### Enhanced
- **Tool Count**: Expanded from 18 to 27 total MCP tools
- **Entity Support**: Now supports 10+ Autotask entities with comprehensive CRUD operations
- **Error Handling**: Improved error messages for unsupported operations
- **API Coverage**: Enhanced coverage of autotask-node library capabilities

### Notes
- Expense items, billing codes, and departments marked as not directly supported in current autotask-node version
- All new tools follow existing pagination and optimization patterns
- Backward compatibility maintained with all existing functionality

## [1.0.4] - 2025-01-09

### Added
- **Data Optimization for Large Responses**: Implemented comprehensive data optimization to prevent "result exceeds maximum length" errors
  - Added field filtering for ticket searches to return only essential fields
  - Implemented automatic text truncation for large description fields (tickets: 500 chars, tasks: 400 chars)
  - Added pagination limits with sensible defaults (tickets/projects/tasks: 25 default, 100 max; companies/contacts: 50 default, 200 max)
  - Created `get_ticket_details` tool for retrieving full ticket data when needed
  - Added data optimization for projects and tasks with similar field filtering

### Changed
- **Ticket Search Optimization**: `search_tickets` now returns optimized data by default
  - Essential fields only: id, ticketNumber, title, description (truncated), status, priority, etc.
  - Removed large arrays like userDefinedFields
  - Truncated resolution and description fields to prevent oversized responses
- **Project and Task Search Optimization**: Applied similar optimization strategies
  - Field filtering for essential data only
  - Description truncation with "... [truncated]" indicators
  - Reduced pagination limits for better performance
- **Tool Descriptions**: Updated tool descriptions to clarify optimization behavior
- **Pagination Limits**: Reduced maximum page sizes across all entity searches for better performance

### Fixed
- **TypeScript Compilation**: Fixed type compatibility issues with optimization functions
- **Response Size Management**: Eliminated "result exceeds maximum length" errors for ticket searches

### Technical Details
- Added `optimizeTicketData()`, `optimizeProjectData()`, and `optimizeTaskData()` methods
- Implemented field filtering using `includeFields` parameter where supported
- Enhanced error handling and logging for optimization processes

## [1.0.3] - 2025-06-09

### Added
- **Major Entity Expansion**: Added support for 8 additional Autotask entities:
  - **Projects**: Search, create, and update project records
  - **Resources**: Search for users/employees in Autotask
  - **Configuration Items**: Search for managed assets and devices
  - **Contracts**: Search for service contracts (read-only)
  - **Invoices**: Search for billing invoices (read-only)
  - **Tasks**: Search, create, and update project tasks
- **Enhanced Tool Coverage**: Expanded from 9 to 17 available MCP tools
- **Comprehensive Type Definitions**: Added TypeScript interfaces for all new entities
- **Status Enums**: Added helpful enums for project, task, opportunity, and contract statuses

### Improved
- **Better Error Handling**: Enhanced type casting for compatibility with autotask-node library
- **Code Organization**: Structured service methods by entity type for better maintainability

## [1.0.2] - 2025-06-09

### Fixed
- **Critical MCP Protocol Fix**: Enhanced stdout wrapper to completely filter all non-JSON-RPC output, eliminating "invalid union" errors in Claude Desktop
- **Critical Authentication Fix**: Removed extra quotes from AUTOTASK_SECRET in .env file that were causing 401 Unauthorized errors
- **Environment Variable Loading**: Updated docker-compose.yml to explicitly use `env_file` directive for proper environment variable handling
- **Lazy Initialization**: Implemented lazy initialization of Autotask client to prevent MCP timeout issues during server startup
- **Container Restart Issues**: Fixed Docker container to start quickly without blocking on Autotask API connection
- **Winston Logger Output**: Fixed Winston logs leaking to stdout by implementing comprehensive stdout interception

### Improved
- **MCP Compliance**: Now fully compliant with JSON-RPC protocol - only valid MCP messages on stdout
- **Error Diagnostics**: Enhanced credential validation and error reporting
- **Development Experience**: Faster development iteration with immediate container startup

## [1.0.1] - 2024-12-09

### Fixed
- **Stdout Interference**: Added TypeScript wrapper script to redirect all non-MCP stdout output to stderr
- **Logger Output**: Fixed logging to use stderr instead of stdout for Claude Desktop compatibility  
- **Third-party Library Output**: Prevented autotask-node library output from interfering with MCP JSON-RPC protocol
- **Build Process**: Fixed wrapper compilation by converting to TypeScript (.ts) for proper build inclusion
- **Docker Image Tag**: Updated documentation to use correct Docker image tag
- **MCP Protocol**: Resolved JSON-RPC parsing errors when connecting to Claude Desktop

### Documentation
- Enhanced Quick Start guide with system-specific configuration examples
- Added troubleshooting section for common Claude Desktop connection issues

## [1.0.0] - 2024-12-09

### Added
- Initial project setup and architecture
- MCP server implementation with full protocol compliance
- Autotask service layer with comprehensive API coverage
- Docker and docker-compose configuration for easy deployment
- Comprehensive test suite with 80%+ coverage requirement
- Structured logging with configurable levels and formats
- TypeScript types for all Autotask entities and MCP protocol
- Complete CI/CD ready setup

### Changed
- N/A (initial release)

### Deprecated
- N/A (initial release)

### Removed
- N/A (initial release)

### Fixed
- N/A (initial release)

### Security
- Implemented secure credential handling through environment variables
- Added non-root user in Docker container for security
- Configured proper resource limits for container deployment

## [1.0.0] - 2024-12-09

### Added
- **🔌 MCP Protocol Compliance**: Full Model Context Protocol implementation
- **🛠️ Autotask Integration**: Complete integration with Kaseya Autotask PSA via autotask-node
- **📚 Resource Access**: Read-only access to companies, contacts, tickets, and time entries
- **🔧 Tool Operations**: CRUD operations for core Autotask entities
- **🔍 Advanced Search**: Powerful search capabilities with filters
- **🐳 Container Support**: Docker and docker-compose configuration
- **📊 Logging System**: Winston-based structured logging
- **🧪 Test Framework**: Jest-based testing with coverage requirements
- **📝 Documentation**: Comprehensive README and API documentation
- **⚙️ Configuration**: Environment-based configuration management

### Core Features
- **Autotask Entities**: Companies, Contacts, Tickets, Time Entries
- **MCP Resources**: Structured read access to Autotask data
- **MCP Tools**: Interactive operations for data manipulation
- **Authentication**: Secure API credential management
- **Error Handling**: Comprehensive error handling with proper MCP error codes
- **Type Safety**: Full TypeScript implementation

### Development Features
- **Hot Reload**: Development server with hot reload capability
- **Testing**: Unit, integration, and API tests
- **Linting**: ESLint configuration with TypeScript support
- **Building**: TypeScript compilation pipeline
- **Docker**: Multi-stage Dockerfile for optimized containers

### Security
- Non-root container execution
- Environment variable credential management
- Input validation and sanitization
- Resource limits and health checks

---

## Release Process

### Version Numbering
This project follows [Semantic Versioning](https://semver.org/):
- **MAJOR**: Incompatible API changes
- **MINOR**: New functionality in a backwards compatible manner
- **PATCH**: Backwards compatible bug fixes

### Release Notes Format
Each release includes:
- **NEW FEATURES**: Major new functionality
- **IMPROVEMENTS**: Enhancements to existing features
- **FIXES**: Bug fixes and stability improvements
- **BREAKING CHANGES**: Any breaking changes and migration guides

### Upcoming Features (Roadmap)
- HTTP transport option for MCP
- Additional Autotask entities (Projects, Assets, etc.)
- Webhook support for real-time updates
- Advanced filtering and sorting options
- Bulk operations for data manipulation
- Performance optimizations and caching
- GraphQL interface for advanced queries
