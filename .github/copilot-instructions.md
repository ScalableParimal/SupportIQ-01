# SupportIQ — Copilot Repository Instructions

SupportIQ is a multi-tenant AI customer-support portal: .NET 8 Web API (Minimal APIs), React + TypeScript (Vite), Azure SQL via EF Core, Azure Blob Storage, Azure AI Search, Azure AI model endpoints, Key Vault, Application Insights, hosted on Azure App Service, CI on GitHub Actions.

## 0. Precedence and governance (read first)

1. The three SDLC Masters win over everything else, including this file:
   `docs/ai/sdlc/SDLC_GOVERNANCE.md`, `docs/ai/sdlc/SDLC_ARTIFACT_GOVERNANCE.md`, `docs/ai/sdlc/SDLC_TRACEABILITY_AND_CHANGE_CONTROL.md`.
2. Then the current **approved** Requirement and **approved** Implementation Plan.
3. Then the engineering standards in this file. If an approved Plan specifies a different engineering choice, follow the Plan and mention the difference. Exception: the 80% coverage gate (section 6), the `REQ-ID` tagging rules (section 2), and the stop gates (section 8) are mandatory team rules; a Plan may make them stricter but never weaker.

This file is a navigation and engineering-standards aid. It creates no approval, gate, or governance authority.

- Distinguish product SDLC work, governance-framework work, and informational requests. "Explain" or "inspect" is not authorization to change files or start a governed process.
- Determine the current approved Requirement and Plan from verification, resolved findings, explicit human approval, and supersession evidence in the repository. A newer filename or a draft does not establish approval. Example: `docs/requirement/SupportIQ_Product_Requirement_v1.13.md` is marked as an unapproved draft.
- Implement only within the approved Requirement, the approved Plan, and the task in scope. A self-check never replaces Implementation Verification or Implementation Review.
- Never claim, infer, or reuse Human Requirement Approval, Human Review Approval, Human Plan Approval, or Human Acceptance. Commit or push governed implementation only after Human Acceptance of that exact state.
- Never turn a finding directly into a code change. Use Finding Investigation → Corrective Change Decision → required approval → implementation → re-verification → review.
- Never edit an existing file in `docs/requirement/`. Changes go through the Requirement Update process, which creates a new immutable version.
- Do not edit `docs/ai/**`, `.github/agents/**`, or `.github/skills/**` as part of product work; those changes follow the governance amendment route. Change this file only through Project Instructions Setup (`.github/agents/project-instructions-setup.agent.md`), with Human Instructions Approval of the exact content.
- When the SDLC Process Orchestrator is used, get fresh, explicit Process Initiation Approval for every start, retry, and transition. It is not a formal human gate.
- If authority, evidence, scope, or a governance interpretation is unclear, stop, explain the gap, and ask the human. Do not invent evidence or silently resolve a conflict.

## 1. Repository layout

Target layout. Create folders only when an approved Plan task needs them; never create files outside this structure without asking.

```text
src/backend/SupportIQ.Api/
  Program.cs                         # composition root (shared file, see section 8)
  Features/<Feature>/<UseCase>/      # vertical slice: endpoint, request/response DTOs, validator, handler
  Features/<Feature>/<Feature>Endpoints.cs   # MapGroup registration for the feature
  Domain/<Aggregate>/                # entities, value objects, domain rules (no EF or ASP.NET references)
  Infrastructure/Persistence/        # AppDbContext, Configurations/, Migrations/
  Infrastructure/<Integration>/      # Blob, Search, AI model, Email adapters behind interfaces
  Common/                            # cross-cutting: auth policies, tenancy, errors, filters (shared)
src/frontend/
  src/app/                           # router, providers, query client (shared)
  src/features/<feature>/            # components/, api/ (query hooks), types.ts, *.test.tsx
  src/shared/                        # api client, Fluent UI wrappers, utilities
tests/backend.Tests/                 # unit tests, mirroring Features/<Feature>/<UseCase>/
tests/backend.IntegrationTests/      # SQL, Blob, Search, and API integration tests
docs/                                # governance, requirements, SDLC records
```

- Put each new use case in its own slice, for example `Features/Tickets/AssignTicket/`. Never add a global `Controllers/`, `Services/`, or `Repositories/` layer.
- A slice may depend on `Domain/`, `Infrastructure/` abstractions, and `Common/`. It must not reference another slice's internal types. Share through `Domain/` or an explicit contract.
- Example feature names: `Identity`, `Tenants`, `Teams`, `Tickets`, `Customers`, `Feedback`, `Knowledge`, `Assistant`, `Analytics`, `Jobs`, `Audit`, `Configuration`.

## 2. Requirement traceability (`REQ-ID`)

A **`REQ-ID`** is the canonical requirement ID from the current approved Requirement, written in the pattern `REQ-XXX-000`. In this repository, the concrete form is `FR-<AREA>-<NN>`, for example `FR-CA-02`, `FR-KB-10`, `FR-TK-04`, or `FR-TEAM-03`. Use the ID exactly as written in the approved Requirement. Never invent, renumber, or reformat a `REQ-ID`. If no applicable `REQ-ID` exists for the code you are writing, stop and ask.

Every test that verifies requirement behavior must carry its `REQ-ID`:

- **Backend (xUnit):** use exactly `[Trait("Requirement", "REQ-XXX-000")]`, with one attribute per `REQ-ID`.

  ```csharp
  [Fact]
  [Trait("Requirement", "FR-KB-10")]
  public async Task Retrieval_ExcludesRejectedVersions() { ... }
  ```

- **Frontend (Vitest):** wrap tests in `describe('REQ-XXX-000: <behavior>', ...)`.

  ```ts
  describe('FR-KB-10: rejected versions are excluded from retrieval', () => { ... });
  ```

- When the Plan task also has a canonical Work Item ID (`WI-<lowercase-uuidv4>`), add `[Trait("WorkItem", "WI-...")]` in backend tests. In frontend tests, append it to the `describe` text. Never invent a Work Item ID.
- Tests for authorization and tenant isolation must name the specific allowed and denied role or scope in the test name.
- Production code does not need `REQ-ID` comments. Add one only where the requirement is not obvious from the slice name.

**Traceability matrix maintenance:** at the end of **every** code-generation session, explicitly remind the developer to update `docs/traceability.md`. List each `REQ-ID` and Work Item ID touched, with the affected slices and tests. Do not edit `docs/traceability.md` yourself unless the developer asks.

`docs/traceability.md` is a non-authoritative developer view. It must not record or imply approval, verification, or acceptance status. The governed traceability report produced by the owning SDLC process (`SDLC_ARTIFACT_GOVERNANCE.md` §8.5) remains authoritative. If the two disagree, report the discrepancy and do not resolve it silently.

## 3. Backend (.NET 8) standards

- **Style:** vertical slices with Minimal APIs. Do **not** use MediatR or another mediator. Each use case has a plain handler class registered in DI and injected into its endpoint. Group endpoints with `MapGroup("/api/<feature>")`, and require authorization on the group.
- **Language:** nullable reference types and implicit usings are on, file-scoped namespaces are required, and warnings are treated as errors. Use `record` types for request and response DTOs. Never expose EF entities from an endpoint.
- **Validation:** use FluentValidation validators in the slice, run through a shared endpoint filter. Return `400` ValidationProblemDetails. Do not validate inside handlers.
- **Errors:**
  - Use one global `IExceptionHandler` plus `AddProblemDetails()`. Responses follow RFC 9457 `application/problem+json` and include `traceId`. Never return stack traces or internal messages.
  - Handle expected outcomes (not found, forbidden, conflict, invalid state transition) with typed results, not exceptions.
  - Status codes: `404` for resources outside the caller's tenant or scope (do not reveal existence), `403` for in-scope actions the role cannot perform, `409` for state or concurrency conflicts.
- **Async:** all database, network, Blob, Search, AI, email, and file I/O is `async`, ends in `Async`, and accepts and forwards a `CancellationToken`. Never use `.Result`, `.Wait()`, `async void`, or `Task.Run` around I/O.
- **Security and tenancy:**
  - Use a central default-deny authorization model with resource/action policies that match the requirement's role matrix.
  - Resolve `TenantId` only from the authenticated principal, never from request input.
  - Enforce tenant, team, and role scope in the backend data query. Hiding something in the UI is not enforcement.
- **Integrations:** call Blob, Search, AI, and email only through interfaces in `Infrastructure/`. Use `IHttpClientFactory` and the Azure SDK clients with `DefaultAzureCredential`. Read settings through the Options pattern (`IOptions<T>` with validation on start).
- **AI safety:** treat retrieved content and user input as untrusted (prompt-injection). Use only allowlisted read-only tools. Validate structured model output against a schema. AI output is a suggestion and never auto-sends to a customer.
- **Logging:** use structured `ILogger` message templates. Never log secrets, tokens, full prompts with customer data, or unmasked customer emails.

## 4. Database (Azure SQL + EF Core)

- Entities in `Domain/` have **no data annotations** and no EF references. Map each entity in its own `IEntityTypeConfiguration<T>` under `Infrastructure/Persistence/Configurations/`. `AppDbContext` uses `ApplyConfigurationsFromAssembly`.
- Apply a global query filter on `TenantId` to every tenant-owned entity. Use `IgnoreQueryFilters()` only in reviewed platform-level code, with a comment explaining why.
- Read-only queries use `.AsNoTracking()` and project to DTOs with `.Select(...)`. Never load whole aggregates just to read a few fields.
- Paginate every list endpoint, and apply filtering before pagination and total count. Avoid N+1 queries; use `Include` or projection deliberately.
- Only use `FromSql` or `ExecuteSql` with interpolated parameters. Never concatenate SQL strings.
- Use optimistic concurrency (`rowversion`) where competing updates are possible, such as ticket assignment and status changes.
- **Migrations:** never generate, edit, or delete migrations or `AppDbContextModelSnapshot.cs` during feature work. Instead, describe the needed schema change in the summary. A designated developer creates a single migration after sequencing, to avoid snapshot collisions.

## 5. Frontend (React + TypeScript) standards

- TypeScript `strict` is on. Do not use `any`, non-null `!` without justification, or `@ts-ignore`.
- Use function components only. Define props as `interface <Component>Props`. Use one exported component per file, named in PascalCase.
- **Server state:**
  - Use TanStack Query exclusively. Put hooks in `features/<feature>/api/` (`useTicketsQuery`, `useAssignTicketMutation`) with a per-feature query-key factory.
  - Never fetch inside `useEffect`.
  - Poll background-job status through `refetchInterval` and stop when the job reaches a terminal state.
- **UI state:** use local `useState`/`useReducer`. Use React Context only for genuinely app-wide UI concerns such as the session user or theme. Do not use Redux.
- **UI library:**
  - Use Fluent UI React v9 (`@fluentui/react-components`) components and `makeStyles` with Fluent `tokens`.
  - Do not use hard-coded colors, raw pixel spacing, inline `style` for theming, or another UI or CSS framework.
- **Accessibility:** use labeled form controls and keyboard-reachable actions, and add an `aria-*` attribute when there is no visible label.
- **Data handling:** the UI reflects backend permissions but never relies on them for security. Render customer email masked unless the API returned an authorized unmasked value. Show AI output as editable drafts with their sources.
- All API calls go through the shared client in `src/shared/api/`, which attaches auth and maps problem-details responses to typed errors.

## 6. Testing and quality gates

- **Coverage gate (mandatory):** at least **80% line and 80% branch coverage**, enforced as a failing quality gate. Use coverlet (`/p:Threshold=80 /p:ThresholdType="line,branch"`) for the backend and Vitest `v8` coverage (`thresholds: { lines: 80, branches: 80 }`) for the frontend.
  - New and changed code must meet the gate.
  - Never lower a threshold or add coverage exclusions to pass. If the gate cannot be met, stop and report the gap instead of weakening it.
- **Backend unit tests (`tests/backend.Tests`):**
  - Use xUnit and NSubstitute for service abstractions. Use the SQLite in-memory provider or the EF Core InMemory provider for EF behavior.
  - Prefer SQLite in-memory when constraints, foreign keys, or transactions matter, because the InMemory provider does not enforce them.
  - **No** real SQL connections, live HTTP, Azure services, AI model calls, or email in unit tests.
- **Backend integration tests (`tests/backend.IntegrationTests`):** SQL repositories, Blob, Search, jobs, and API tests run here via `WebApplicationFactory` against isolated test resources, never against shared or production data.
- **Frontend tests:**
  - Use Vitest and React Testing Library. Mock the network with MSW; do not mock `fetch` or internal hooks directly.
  - Query by role or label, not by test IDs or CSS selectors.
- **Mandatory cases:** every endpoint or feature that touches protected data needs tests for both the allowed role/scope and at least one denied cross-tenant, cross-team, or role case.
- Tests must be deterministic: no real clock (inject `TimeProvider`), no random data without a fixed seed, no ordering dependence.
- Run the relevant `dotnet test` and `npm test` / `vitest run` commands before finishing, and report the actual results, including anything not run.

## 7. Azure App Service runtime

- **Backend configuration:**
  - Defaults live in `appsettings.json`.
  - Environment overrides use App Service application settings with `Section__Key` names.
  - Secrets come from Key Vault references or Managed Identity. Never put secrets, keys, or connection-string passwords in source, `appsettings*.json`, or test fixtures.
- **Frontend configuration:**
  - `import.meta.env.VITE_*` values are **baked in at build time**. App Service settings do not change a built bundle.
  - Use only non-secret values, set by the CI build per environment.
  - Anything that must change at runtime comes from a backend config endpoint.
- **Storage:** never write to the local disk or to `wwwroot` for persistent data. Uploads and processed artifacts go to Azure Blob Storage. Temporary files must be stream-based or cleaned up in `finally`.
- **Statelessness:** assume multiple instances and restarts at any time. No in-memory caches or job queues as a source of truth. Background jobs are persisted in SQL, tenant-scoped, idempotent, and cooperatively cancellable.
- **Telemetry and health:** use Application Insights for request, dependency, and exception tracing with correlation IDs. Expose a lightweight `/health` endpoint for App Service health checks.

## 8. Team collaboration guardrails

- **Shared-file confirmation gate:** before modifying any of these files, stop, explain the exact change and why, and wait for explicit developer confirmation.
  - `Program.cs` and other DI or middleware registration
  - `AppDbContext.cs` and `Infrastructure/Persistence/Migrations/**`
  - `Common/**`
  - `Directory.Build.props`, `*.csproj` package references, `package.json`, and lockfiles
  - `src/frontend/src/app/**` (router, providers)
  - `.github/workflows/**`
  - authorization policy definitions
- **New dependencies:** do not add a NuGet or npm package without confirmation. State its purpose, license, and alternatives.
- **Breaking-change stop gate:** stop immediately and report instead of proceeding if a request would do either of the following, and route the issue through the governed finding or corrective path:
  - Change a published API contract: route, method, request or response shape, status code, or problem type, as defined by the approved Plan or the published OpenAPI document.
  - Break or weaken behavior traced to an existing (historical) `REQ-ID` or Work Item ID, including authorization, tenant isolation, approval, or audit rules, or remove or change a test that carries that `REQ-ID`.
- **Concurrent work:** assess shared surfaces and checkout/worktree context before parallel work. Serialize writes in a shared checkout. Isolated worktrees require an explicit integration step.
- **Scope and quality:**
  - Make focused changes within the task scope, preserve unrelated work, and reuse existing patterns before introducing new ones.
  - Surface errors and missing evidence explicitly, without silent fallbacks.
  - Update directly affected documentation.
- **Session summary:** end each session with:
  - the files changed
  - the `REQ-ID` and Work Item IDs touched, with the reminder to update `docs/traceability.md`
  - the tests run, their results, and coverage against the 80% gate
  - any needed migration or configuration change
  - any shared-file or breaking-change concerns
