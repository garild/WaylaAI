# Backend API — Tech Lead Build Guide & AI Prompts

**Role:** Senior .NET Developer / Tech Lead  
**Last updated:** 2026-07-09 (Minimal API)  
**Status:** Ready to implement — run prompts in order

This document is the **implementation playbook**. Each prompt (`B-00X`) is copy-paste ready for Cursor or any AI coding assistant. Run the **Master prompt once per session**, then execute prompts in sequence. Do not skip acceptance criteria.

**Related:** [decisions.md](../decisions.md) · [cicd.md](../../infrastructure/cicd.md) · [creator-assistant-prompt.md](../content/creator-assistant-prompt.md)

---



## How to use these prompts

1. Paste **Master prompt** into a new Cursor chat.
2. Run **B-001** through **B-021** in order unless noted.
3. After each prompt: `dotnet build`, `dotnet test`, commit when green.


| Phase                | Prompts     | Episode | Outcome                                                    |
| -------------------- | ----------- | ------- | ---------------------------------------------------------- |
| Prerequisites        | B-001       | 4       | Tooling verified                                           |
| Solution scaffold    | B-002–B-003 | 4       | Four-project onion solution                                |
| Domain & Application | B-004–B-007 | 4–5     | `GetCities` via MediatR + Conduit library + in-memory repo |
| API & local run      | B-008–B-010 | 4–5     | Swagger, health, unit tests                                |
| PostgreSQL & EF      | B-011–B-014 | 5–6     | Real data, seed, migrations                                |
| Catalog API complete | B-015–B-016 | 5       | All three read endpoints + error shape                     |
| Container & config   | B-017–B-018 | 4–6     | Dockerfile, secrets pattern                                |
| Auth                 | B-019       | 7       | Supabase JWT                                               |
| CI                   | B-020–B-021 | 16      | GitHub Actions + env config                                |


---



## Architecture reference (read before B-002)

Onion Architecture — four concentric layers, dependencies point inward


| Ring   | Project                  | Responsibility                                                      |
| ------ | ------------------------ | ------------------------------------------------------------------- |
| Center | `WaylaAI.Domain`         | Entities, value objects, domain rules — **zero dependencies**       |
| Ring 2 | `WaylaAI.Application`    | MediatR handlers, FluentValidation, DTOs, repository interfaces     |
| Ring 3 | `WaylaAI.Infrastructure` | EF Core, PostgreSQL, Redis, SQS — implements Application interfaces |
| Outer  | `WaylaAI.Api`            | Minimal API endpoint modules, middleware, DI bootstrap, OpenAPI     |


**Dependency rule:** Domain → nothing. Application → Domain only. Infrastructure → Application + Domain. Api → Application (+ Infrastructure for DI wiring only).

DDD bounded contexts · MediatR + FluentValidation pipeline

**Stack (locked):** .NET 10 · REST · **Minimal API** (no MVC controllers) · Onion + DDD · MediatR · FluentValidation · PostgreSQL + EF Core 10 · Supabase Auth (MVP) · AWS ECS · Terraform · GitHub Actions OIDC [D-004–D-016].

**Target repo layout:**

```
Wayla/
├── src/
│   ├── Conduit/                    # B-007 Part 1 — learning library (not used in Api)
│   ├── WaylaAI.Domain/
│   ├── WaylaAI.Application/
│   ├── WaylaAI.Infrastructure/
│   └── WaylaAI.Api/
│       ├── Program.cs
│       ├── Endpoints/              # Minimal API route modules — NO Controllers/
│       └── Middleware/
├── tests/                          # B-007 Part 1
│   ├── WaylaAI.Domain.Tests/
│   └── WaylaAI.Application.Tests/
├── docker-compose.yml
├── Dockerfile
└── WaylaAI.sln
```



### Minimal API conventions (locked — no controllers)

Wayla uses **endpoint modules** — static extension methods on `IEndpointRouteBuilder`. There is **no** `Controllers/` folder, no `AddControllers()`, no `MapControllers()`.


| Rule                            | Detail                                                                                             |
| ------------------------------- | -------------------------------------------------------------------------------------------------- |
| **One file per resource**       | `Endpoints/Catalog/CitiesEndpoints.cs`, `LocationsEndpoints.cs`, …                                 |
| **Thin routes**                 | Each `MapGet`/`MapPost` only builds a MediatR request and returns `Results.Ok` / `Results.Problem` |
| **No business logic in routes** | Validation → FluentValidation pipeline; orchestration → MediatR handler                            |
| **OpenAPI**                     | `MapGroup` + `.WithTags()` + `.WithName()` + `.Produces<T>()`                                      |
| **Auth**                        | `.RequireAuthorization()` on route or group — never `[Authorize]` attributes                       |
| **Binding**                     | `[AsParameters]` for query records, or explicit parameters                                         |


**Example shape (reference for B-008):**

```csharp
// WaylaAI.Api/Endpoints/Catalog/CitiesEndpoints.cs
public static class CitiesEndpoints
{
    public static RouteGroupBuilder MapCitiesEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/cities").WithTags("Catalog");

        group.MapGet("/", async (
            [AsParameters] GetCitiesQuery query,
            IMediator mediator,
            CancellationToken ct) =>
        {
            var result = await mediator.Send(query, ct);
            return Results.Ok(result);
        })
        .WithName("GetCities")
        .Produces<GetCitiesResult>(StatusCodes.Status200OK);

        return group;
    }
}
```

```csharp
// WaylaAI.Api/Program.cs (excerpt)
var app = builder.Build();
app.UseExceptionHandling(); // B-016
app.MapCitiesEndpoints();
app.Run();
```

---



## Master prompt (paste once per session)

```
You are a Senior .NET Developer and Tech Lead building Wayla — an AI travel companion SaaS.

## Locked constraints (never violate)

- Architecture: Onion / Clean Architecture — Domain / Application / Infrastructure / Api
- Patterns: DDD aggregates, MediatR (one handler per use case), FluentValidation in pipeline
- Runtime: .NET 10, C# latest
- API: REST via **ASP.NET Core Minimal API only** — endpoint modules in `WaylaAI.Api/Endpoints/`
- **Never** create MVC controllers, `AddControllers()`, or `MapControllers()`
- Database: PostgreSQL + EF Core 10 with migrations
- Auth (MVP): Supabase Auth JWT — API validates issuer/audience; domain stays auth-agnostic
- First endpoints: GET /api/cities, GET /api/locations, GET /api/categories [D-010]
- No Redis, SQS, or AI code unless I explicitly ask for that episode scope
- Repo root: WaylaAI/ (product code). Planning docs live outside repo.

## Dependency rules

- WaylaAI.Domain: zero project references
- WaylaAI.Application: references Domain only
- WaylaAI.Infrastructure: references Application + Domain
- WaylaAI.Api: references Application + Infrastructure (Infrastructure only for DI registration)

## Code standards

- sealed classes for entities, handlers, validators, endpoint mapping classes
- async/await with CancellationToken on all I/O
- DTOs in Application — domain entities never returned from HTTP responses
- Repository interfaces in Application; implementations in Infrastructure
- One folder per use case: Application/Catalog/Cities/GetCities/
- One endpoint module per resource: Api/Endpoints/Catalog/CitiesEndpoints.cs
- Add DependencyInjection.cs extension per layer (AddApplication, AddInfrastructure)
- Problem Details (RFC 7807) for validation and domain errors
- File-scoped namespaces
- Use Body expression
- Code formating before commit
- Remove not used reference, usings

## Output format

- Give complete file contents for every new/changed file
- List file paths in a tree after implementation
- End with: build command, test command, and one curl example per new endpoint
- Flag any decision that needs D-017+ in decisions.md — do not silently change stack choices
- If you are about to create a Controller class, stop — use an endpoint module in Api/Endpoints/ instead

When I give you a B-00X prompt, implement only that prompt's scope. Do not jump ahead to later episodes.
```

---



## Phase 0 — Prerequisites



### B-001 — Verify dev environment & repo

```
B-001 — Verify dev environment

As Senior .NET Tech Lead, verify and document the local toolchain for WaylaAI backend development.

Deliver:
1. Checklist (with exact version commands): .NET 10 SDK, Docker Desktop, Git, optional: dotnet-ef global tool
2. Create or update WaylaAI/README.md section "Backend — local setup" with:
   - Prerequisites
   - Clone + first run steps (placeholder until B-010)
   - Environment variable table (ConnectionStrings, Supabase JWT — values as placeholders)
3. .gitignore entries for: bin/, obj/, .vs/, appsettings.json secrets, .env
4. download and install all tools which is requrired

Do NOT scaffold the solution yet — verification and docs only.

Acceptance:
- `dotnet --version` shows 10.x
- `docker --version` works
- dotnet-ef global tool - ability to run migrations, add and remove
- README section is copy-paste ready for a new contributor
```

---



## Phase 1 — Solution scaffold



### B-002 — Create onion solution & project references

```
B-002 — Create WaylaAI.sln with four layers

Scaffold the Clean Architecture solution in the Wayla/ repo.

Create:
- WaylaAI.sln
- src/WaylaAI.Domain (class library, net10.0, no dependencies)
- src/WaylaAI.Application (class library, refs Domain)
- src/WaylaAI.Infrastructure (class library, refs Application + Domain)
- src/WaylaAI.Api (ASP.NET Core — use `dotnet new web` or `dotnet new webapi` with `--use-minimal-apis`; refs Application + Infrastructure)
- tests/WaylaAI.Domain.Tests (xUnit, refs Domain)
- tests/WaylaAI.Application.Tests (xUnit, refs Application; Moq or NSubstitute)

Configure:
- Nullable enable, implicit usings, treat warnings as errors in Directory.Build.props at solution root
- Correct project references enforcing onion dependency rule
- Solution folders: src/, tests/

Do NOT add MediatR, EF, or endpoint modules yet.

Acceptance:
- `dotnet build` succeeds with zero warnings
- Domain project has no PackageReference except analyzers if needed
- File tree printed in response
```



### B-003 — Install NuGet packages (version-locked)

```
B-003 — Add NuGet packages per layer

Add packages to the scaffolded solution. Use current stable 10.x / 12.x lines.

WaylaAI.Application:
- MediatR 12.*
- FluentValidation 11.*
- FluentValidation.DependencyInjectionExtensions 11.*

WaylaAI.Infrastructure:
- Microsoft.EntityFrameworkCore 10.*
- Npgsql.EntityFrameworkCore.PostgreSQL 10.*
- Microsoft.EntityFrameworkCore.Design 10.* (PrivateAssets=all)

WaylaAI.Api:
- Microsoft.AspNetCore.OpenApi (or Swashbuckle.AspNetCore 6.* if you need Swagger UI in Development)
- Do NOT add Microsoft.AspNetCore.Mvc or controller packages

WaylaAI.Application.Tests:
- xunit, Microsoft.NET.Test.Sdk, coverlet.collector, Moq or NSubstitute

Add empty DependencyInjection.cs stubs:
- Application: AddApplication(IServiceCollection)
- Infrastructure: AddInfrastructure(IServiceCollection, IConfiguration)
- Api Program.cs calls both

Acceptance:
- `dotnet restore && dotnet build` green
- No package in Domain project
```

---



## Phase 2 — Domain layer



### B-004 — Domain foundation (Catalog / City)

```
B-004 — Domain layer: Catalog bounded context start

Implement WaylaAI.Domain for the Catalog context — City aggregate only.

Create:
- Common/DomainException.cs
- Catalog/City.cs — factory Create(), private setters, parameterless ctor for EF
- Catalog/ namespace only; no Country/Location yet

Rules on City.Create:
- Name required, trimmed
- CountryCode required, uppercased ISO 3166-1 alpha-2
- Slug required, lowercased
- Throw DomainException on violation

Add WaylaAI.Domain.Tests:
- City_Create_WithValidData_SetsProperties
- City_Create_WithEmptyName_ThrowsDomainException

Acceptance:
- Domain project still has zero external package refs
- `dotnet test` passes Domain tests
```

---



## Phase 3 — Application layer



### B-005 — MediatR pipeline & DI registration

```
B-005 — Application: MediatR + ValidationBehavior + DI

Wire the Application layer cross-cutting pipeline using MediatR 12 (already added in B-003).

Create in WaylaAI.Application:
- Common/Behaviors/ValidationBehavior.cs — IPipelineBehavior that runs FluentValidation before handler
- DependencyInjection.cs — AddApplication():
  - RegisterMediatR from assembly
  - AddValidatorsFromAssembly
  - Register ValidationBehavior as transient IPipelineBehavior<,>

Create stub folder structure (empty handlers OK):
- Catalog/Cities/GetCities/ (placeholder files allowed)

Acceptance:
- Application builds
- ValidationBehavior throws FluentValidation.ValidationException on failures
- No references to Infrastructure, Api, or Conduit
- `dotnet build` green
```



### B-006 — GetCities use case (query, handler, validator, interface)

```
B-006 — GetCities query end-to-end in Application

Implement the first use case matching D-010.

Create under Application/Catalog/Cities/GetCities/:
- GetCitiesQuery.cs — record with CountryCode?, Page, PageSize; IRequest<GetCitiesResult>
- GetCitiesResult.cs — Items (IReadOnlyList<CityDto>), TotalCount
- CityDto.cs — Id, Name, CountryCode, Slug
- GetCitiesQueryHandler.cs — inject ICityReadRepository, map to DTOs
- GetCitiesQueryValidator.cs — Page >= 1, PageSize 1–100, CountryCode length 2 when set

Create Common/Interfaces/ICityReadRepository.cs:
- Task<(IReadOnlyList<City> Items, int TotalCount)> ListAsync(string? countryCode, int page, int pageSize, CancellationToken ct)

Acceptance:
- Handler unit-testable with mocked ICityReadRepository
- Validator rejects pageSize=500
```



### B-007 — Conduit library + in-memory repository

Two-part prompt. Run **Part 1** first (build the Conduit library — understand how MediatR works under the hood), then **Part 2** (in-memory repo so Ep 4 runs without PostgreSQL). Wayla **Api and handlers use MediatR 12** — `Conduit` is a standalone learning library, not referenced by `WaylaAI.Application`.

#### B-007 Part 1 — Conduit library

```
B-007 Part 1 — Conduit: lightweight async-first request dispatcher

Goal: Build a small mediator-style library named Conduit that replicates MediatR's core
request/handler/pipeline model — async-first, auto-registration, pipeline behaviors,
DI integration. After B-005/B-006 you already use MediatR in Application; Conduit is
the "build it yourself" companion for the episode, not a production dependency.

Rules:
1. Async-first: All handlers and pipelines return Task<TResponse>.
2. Auto-registration: Scan assemblies for IRequestHandler<,> and IValidator<>.
3. Pipeline behaviors: Logging and validation layered around handler execution.
4. DI integration: Resolve handlers and behaviors via IServiceProvider per Send.

Constraints:
- Target .NET 10 (net10.0), C# latest.
- Library packages: Microsoft.Extensions.DependencyInjection,
  Microsoft.Extensions.Logging.Abstractions
- Test packages: xunit, Microsoft.NET.Test.Sdk, coverlet.collector,
  Microsoft.Extensions.Logging (for services.AddLogging() in tests)
- Thread-safe handler registry (ConcurrentDictionary); frozen after AddConduit completes.
- Fail fast:
  - At registration: duplicate handler for the same request type → throw
  - At Send: no handler for request type → InvalidOperationException

Non-goals (do not implement):
- INotification / Publish
- IStreamRequest / streaming
- Caching pipeline

Core API (namespace Conduit — implement exactly):

- IRequest<TResponse>
- IRequestHandler<TRequest, TResponse> where TRequest : IRequest<TResponse>
  → Task<TResponse> Handle(TRequest request, CancellationToken cancellationToken)
- RequestHandlerDelegate<TResponse> — delegate Task<TResponse> Next()
- IPipelineBehavior<TRequest, TResponse>
  → Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next,
    CancellationToken cancellationToken)
- IConduit (NOT IMediator — avoids clash with MediatR in the same solution)
  → Task<TResponse> SendAsync<TResponse>(IRequest<TResponse> request,
    CancellationToken cancellationToken = default)
- ConduitDispatcher : IConduit — default implementation (class name avoids namespace/type clash)

Validation (no FluentValidation — Conduit owns its own types):
- IValidator<TRequest> → ValidationResult Validate(TRequest request)
- ValidationResult — IsValid, IReadOnlyList<string> Errors
- ConduitValidationException — thrown when validation fails
- ValidationBehavior: resolve IValidator<TRequest> from DI; if none registered, call next()
  without error; if invalid, throw ConduitValidationException (do not call handler)

Optional diagnostics hook (for tests and future observability):
- IPipelineTracker — void Record(string step); optional in DI (GetService, skip if null)
- LoggingBehavior: Record("Logging:Before") → await next() → Record("Logging:After")
- ValidationBehavior: Record("Validation") after validation passes, then await next()
- PingHandler: Record("Handler") after successful handle

Pipeline order (must match MediatR semantics):
- AddOpenBehavior registers services.AddTransient(typeof(IPipelineBehavior<,>), typeof(TBehavior<,>))
- At Send, resolve IEnumerable<IPipelineBehavior<TRequest,TResponse>> via GetServices
- Reverse the list, then fold into a delegate chain (innermost = handler, outermost = last registered)
- Registration order in tests: ValidationBehavior first, LoggingBehavior second
  → execution order: Logging:Before → Validation → Handler → Logging:After

SendAsync implementation (required — ensures pipeline works at runtime):

1. requestType = request.GetType()
2. handlerType = typeof(IRequestHandler<,>).MakeGenericType(requestType, typeof(TResponse))
3. If GetService(handlerType) is null → throw InvalidOperationException
4. Inner delegate: invoke handler.Handle(request, ct) via reflection (or compiled helper)
5. behaviorInterface = typeof(IPipelineBehavior<,>).MakeGenericType(requestType, typeof(TResponse))
6. behaviors = GetServices(behaviorInterface).Reverse().ToList()
7. Fold behaviors into delegate chain (each wraps previous next)
8. return await outermost()

Note: TRequest is only known at runtime (request.GetType()), so handler and behavior
resolution MUST use MakeGenericType — do not assume compile-time TRequest.

DI registration:
- services.AddConduit(cfg => { ... })
- cfg.RegisterHandlersFromAssembly(Assembly) — register each handler as transient for its
  IRequestHandler<TRequest,TResponse> interface AND as concrete type
- cfg.RegisterValidatorsFromAssembly(Assembly) — register each IValidator<T> as transient
- cfg.AddOpenBehavior(typeof(ValidationBehavior<,>))
- cfg.AddOpenBehavior(typeof(LoggingBehavior<,>))
- services.AddTransient<IConduit, ConduitDispatcher>()
- Handlers and IConduit: transient

Create projects (add to solution if not present):
- src/Conduit/ (class library)
- tests/Conduit.Tests/ (xUnit)

File layout under src/Conduit/:
- Abstractions/ — IRequest, IRequestHandler, IConduit, IPipelineBehavior,
  RequestHandlerDelegate, IPipelineTracker
- ConduitDispatcher.cs — IConduit implementation + pipeline builder
- ConduitOptions.cs + ServiceCollectionExtensions.cs — AddConduit
- Behaviors/LoggingBehavior.cs — ILogger<LoggingBehavior<,>> + optional IPipelineTracker
- Behaviors/ValidationBehavior.cs
- Validation/ — IValidator, ValidationResult, ConduitValidationException
- Internal/HandlerRegistry.cs

Test types (tests/Conduit.Tests/):
- Ping.cs — record Ping(string Message) : IRequest<string>
- PingHandler.cs — inject IPipelineTracker; return $"Pong: {request.Message}"; Record("Handler")
- PingValidator.cs — Message must not be null/whitespace
- PipelineTracker.cs — IPipelineTracker singleton; List<string> Steps
- ConduitPipelineTests.cs — full DI setup (see below)

Test DI setup (copy this shape):

  var services = new ServiceCollection();
  services.AddLogging();
  services.AddSingleton<IPipelineTracker, PipelineTracker>();
  services.AddConduit(cfg =>
  {
      cfg.RegisterHandlersFromAssembly(typeof(PingHandler).Assembly);
      cfg.RegisterValidatorsFromAssembly(typeof(PingValidator).Assembly);
      cfg.AddOpenBehavior(typeof(ValidationBehavior<,>));  // inner
      cfg.AddOpenBehavior(typeof(LoggingBehavior<,>));   // outer
  });
  var provider = services.BuildServiceProvider();
  var conduit = provider.GetRequiredService<IConduit>();

Acceptance (Conduit.Tests — all must pass):
- await conduit.SendAsync(new Ping("Hello Async"), ct) → "Pong: Hello Async"
- await conduit.SendAsync(new Ping(""), ct) → throws ConduitValidationException
- PipelineTracker.Steps equals ["Logging:Before", "Validation", "Handler", "Logging:After"]
- SendAsync for unregistered request type → InvalidOperationException
- `dotnet test` green
- Conduit project has zero references to WaylaAI.* or MediatR
```



#### B-007 Part 2 — In-memory repository

```
B-007 Part 2 — In-memory ICityReadRepository for local dev

Implement a temporary in-memory repository so Ep 4 can run without PostgreSQL.

Create in WaylaAI.Infrastructure:
- Persistence/InMemory/InMemoryCityReadRepository.cs
- Seed 3 cities: Lisbon (PT), Porto (PT), Barcelona (ES)
- Register in AddInfrastructure when ConnectionStrings:Default is empty OR env USE_IN_MEMORY_DB=true

Keep interface in Application; implementation in Infrastructure.

Acceptance:
- Handler integration test (Application.Tests) returns 3 cities with no DB
- Clear comment: "Replace with EF repository in B-013"
- `dotnet test` green (Conduit.Tests + Application.Tests)
- No reference from WaylaAI.* projects to Conduit
```

---



## Phase 4 — API layer



### B-008 — API bootstrap: Program.cs & CitiesEndpoints (Minimal API)

```
B-008 — WaylaAI.Api: Program.cs + CitiesEndpoints (Minimal API)

Wire the API presentation layer using Minimal API endpoint modules — NO controllers.

Create:
- Endpoints/Catalog/CitiesEndpoints.cs
  - static extension MapCitiesEndpoints(this IEndpointRouteBuilder app)
  - MapGroup("/api/cities").WithTags("Catalog")
  - MapGet("/", ...) — inject IMediator + [AsParameters] GetCitiesQuery, return Results.Ok(result)
  - .WithName("GetCities").Produces<GetCitiesResult>(200)
- Endpoints/EndpointRouteBuilderExtensions.cs — MapWaylaEndpoints() calls all resource modules
- Program.cs:
  - AddApplication(), AddInfrastructure(configuration)
  - NO AddControllers() / NO MapControllers()
  - Development: MapOpenApi() + optional Swagger UI middleware
  - app.MapWaylaEndpoints()
- appsettings.json — logging defaults, placeholder ConnectionStrings
- appsettings.Development.json — UseInMemory or empty connection string

Each route delegate must be thin (~3 lines): build/send MediatR request, return Results. No EF or validation in the route.

Acceptance:
- No Controllers/ folder exists in the solution
- `dotnet run --project src/WaylaAI.Api` starts
- curl GET /api/cities returns JSON with seeded cities
```



### B-009 — Health check & OpenAPI

```
B-009 — Health endpoint + OpenAPI documentation

Add operational endpoints and API docs.

Create:
- GET /health — MapGet("/health", ...) with ASP.NET Core health checks; add "self" check
- Optional: GET /health/ready when DB wired (skip DB check for in-memory mode)
- OpenAPI: .WithTags(), .WithName(), .Produces<T>() on catalog routes; document query params
- MapOpenApi() in Development; optional Swagger UI at /swagger

Update README: curl examples for /health and /api/cities

Acceptance:
- /health returns 200 Healthy
- Swagger UI loads at /swagger in Development
```



### B-010 — Application unit tests

```
B-010 — Unit tests for GetCitiesQueryHandler and validator

Add WaylaAI.Application.Tests coverage.

Tests:
- GetCitiesQueryHandler_ReturnsMappedDtos
- GetCitiesQueryHandler_PassesPaginationToRepository
- GetCitiesQueryValidator_RejectsPageSizeOver100
- GetCitiesQueryValidator_RejectsInvalidCountryCode
- ValidationBehavior_ThrowsWhenValidatorFails

Use Moq/NSubstitute for ICityReadRepository.

Acceptance:
- `dotnet test` all green
- No database required for test run
```

---



## Phase 5 — PostgreSQL & EF Core



### B-011 — Docker Compose for local PostgreSQL

```
B-011 — docker-compose.yml for local PostgreSQL

Add local database infrastructure.

Create at repo root:
- docker-compose.yml — postgres:16-alpine, port 5432, db wayla, user/password from env
- .env.example — POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB
- README section: docker compose up -d, connection string format

Connection string key: ConnectionStrings:Default

Acceptance:
- `docker compose up -d` starts Postgres
- psql or docker exec can connect
- .env in .gitignore
```



### B-012 — EF Core DbContext, configuration, initial migration

```
B-012 — EF Core: WaylaDbContext + City configuration + migration

Replace in-memory persistence path with EF Core (keep in-memory as fallback via config flag).

Create in Infrastructure:
- Persistence/WaylaDbContext.cs — DbSet<City>
- Persistence/Configurations/CityConfiguration.cs — table catalog.cities, indexes on CountryCode, Slug unique
- Repositories/ — placeholder for B-013

Install dotnet-ef if needed. Add migration: InitialCatalog.

Update AddInfrastructure:
- Register DbContext with Npgsql when connection string present
- Apply migrations on startup in Development only (document prod strategy)

Acceptance:
- `dotnet ef database update` creates catalog.cities
- Api still runs against in-memory when no connection string
```



### B-013 — EF CityReadRepository

```
B-013 — EF implementation of ICityReadRepository

Implement production repository.

Create Infrastructure/Persistence/Repositories/CityReadRepository.cs:
- ListAsync with optional country filter, Skip/Take pagination, CountAsync for total
- AsNoTracking for reads
- Map from City entity (Domain) — no anonymous types leaking to Application

Register in DI; remove or demote in-memory as dev-only fallback.

Acceptance:
- GET /api/cities?country=PT returns Lisbon + Porto from PostgreSQL
- Pagination: page=2&pageSize=1 works correctly
```



### B-014 — Seed data migration or hosted seeder

```
B-014 — Seed catalog data

Seed minimum viable catalog for demos and Ep 5 filming.

Add 3 cities (Lisbon, Porto, Barcelona) via:
- EF migration with InsertData, OR
- IHostedService / extension that seeds only when table empty (Development + Staging)

Also prepare stub entities for Location and Category (domain + EF config) — tables only, no endpoints yet.

Acceptance:
- Fresh `dotnet ef database update` + run → 3 cities in DB
- Idempotent — re-run does not duplicate
```

---



## Phase 6 — Complete catalog API (Episode 5)



### B-015 — GET /locations and GET /categories

```
B-015 — Locations and Categories read endpoints

Complete D-010 first API surface.

Domain:
- Catalog/Location.cs, Catalog/Category.cs (minimal aggregates)

Application (same pattern as GetCities):
- GetLocationsQuery + Handler + Validator + Dto + ILocationReadRepository
- GetCategoriesQuery + Handler + Validator + Dto + ICategoryReadRepository

Infrastructure:
- EF configs, repositories, seed 5 locations + 3 categories linked to seeded cities

Api (Minimal API endpoint modules):
- Endpoints/Catalog/LocationsEndpoints.cs — MapGet with cityId, page, pageSize query params
- Endpoints/Catalog/CategoriesEndpoints.cs — MapGet for list
- Register both in MapWaylaEndpoints()

Acceptance:
- curl examples for all three endpoints in README
- All handlers have validators + unit tests
- OpenAPI documents all query params on minimal routes
- No controller classes anywhere in WaylaAI.Api
```



### B-016 — Exception handling middleware (Problem Details)

```
B-016 — Global exception handling middleware

Implement consistent API error responses.

Create Api/Middleware/ExceptionHandlingMiddleware.cs:
- ValidationException → 400 Problem Details, errors grouped by property name
- DomainException → 400 or 422 with title + detail
- Unhandled → 500, no stack trace in Production

Register early in pipeline. Route delegates must not contain try/catch — middleware handles all errors.

Film moment: GET /api/cities?pageSize=500 → 400 with field errors.

Acceptance:
- Invalid pageSize returns application/problem+json
- Development still logs full exception
```

---



## Phase 7 — Container & configuration



### B-017 — Dockerfile for API

```
B-017 — Multi-stage Dockerfile for WaylaAI.Api

Create production-ready container image.

Create Dockerfile at repo root:
- Stage 1: sdk — restore, build, publish WaylaAI.Api
- Stage 2: aspnet runtime — non-root user, port 8080, ENTRYPOINT WaylaAI.Api.dll
- .dockerignore — bin, obj, .git

Optional: docker compose service `api` depending on postgres for full local stack.

Acceptance:
- `docker build -t wayla-api .` succeeds
- Container responds to GET /health on 8080
```



### B-018 — Configuration & user secrets

```
B-018 — appsettings hierarchy + user secrets

Configure environment-specific settings the Tech Lead way.

Structure:
- appsettings.json — non-secret defaults
- appsettings.Development.json — local overrides
- appsettings.Staging.json — placeholder structure for ECS
- User secrets ID on WaylaAI.Api for local dev

Sections:
- ConnectionStrings:Default
- Supabase:JwtIssuer, Supabase:JwtAudience (placeholders)
- Serilog or built-in logging levels per environment

Document in README: `dotnet user-secrets set` examples.

Acceptance:
- Api reads config without hardcoded connection strings
- appsettings.Development.json not committed if it contains secrets (use user secrets)
```

---



## Phase 8 — Authentication (Episode 7)



### B-019 — Supabase JWT authentication

```
B-019 — Configure Supabase Auth JWT validation

Wire authentication without polluting Domain.

Infrastructure:
- Auth/SupabaseJwtOptions.cs — Issuer, Audience, JwtSecret or JWKS URL
- Extension AddSupabaseAuthentication(IServiceCollection, IConfiguration)

Api:
- Program.cs: AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
- Validate issuer, audience, signature per Supabase project docs
- AddAuthorization — default policy optional for now

Endpoints:
- Keep catalog GET route groups anonymous for Ep 5–6 (no .RequireAuthorization())
- Add protected route: MapGet("/api/me", ...).RequireAuthorization() for Ep 7 demo

Do NOT add ASP.NET Identity, Keycloak, or MVC controllers.

Acceptance:
- Valid Supabase JWT → 200 on protected route
- Missing/invalid token → 401
- Document required Supabase dashboard settings + appsettings keys
- Note: confirm D-018 in decisions.md when Supabase is locked
```

---



## Phase 9 — CI/CD & environments



### B-020 — GitHub Actions api-ci.yml

```
B-020 — GitHub Actions CI for .NET API

Create PR pipeline per infrastructure/cicd.md.

Create .github/workflows/api-ci.yml:
- Trigger: pull_request to main, paths src/** tests/**
- Runs on ubuntu-latest
- Setup .NET 10.x
- dotnet restore, build, test
- docker build (no push) to verify Dockerfile
- Upload test results optional

No AWS credentials on PR workflow. No deploy.

Acceptance:
- Workflow YAML valid
- Matches cicd.md naming and goals
- Badge snippet for README
```



### B-021 — Staging configuration & CD placeholder

```
B-021 — Staging appsettings + api-cd-staging.yml skeleton

Prepare for Episode 16 deploy — skeleton only if AWS not provisioned.

Deliver:
1. appsettings.Staging.json — ECS/RDS placeholder keys, no secrets
2. .github/workflows/api-cd-staging.yml skeleton:
   - Trigger: push to main
   - OIDC to AWS (commented TODO for role ARN)
   - Steps: test → docker push ECR → ecs update-service (commented)
3. README: environment promotion table dev → staging → prod [D-016]

Acceptance:
- CI workflow from B-020 unchanged and green
- CD file documents exact secrets/vars needed in GitHub
- No hardcoded AWS account IDs
```

---



## Prompt index (quick reference)


| ID    | Title                            | Depends on   |
| ----- | -------------------------------- | ------------ |
| B-001 | Verify dev environment           | —            |
| B-002 | Solution scaffold                | B-001        |
| B-003 | NuGet packages                   | B-002        |
| B-004 | Domain / City                    | B-003        |
| B-005 | MediatR pipeline                 | B-004        |
| B-006 | GetCities use case               | B-005        |
| B-007 | Conduit library + in-memory repo | B-006        |
| B-008 | Minimal API + CitiesEndpoints    | B-007        |
| B-009 | Health + OpenAPI                 | B-008        |
| B-010 | Unit tests                       | B-006        |
| B-011 | Docker PostgreSQL                | B-001        |
| B-012 | EF DbContext + migration         | B-011, B-004 |
| B-013 | EF repository                    | B-012        |
| B-014 | Seed data                        | B-013        |
| B-015 | Locations + Categories API       | B-014        |
| B-016 | Exception middleware             | B-008        |
| B-017 | Dockerfile                       | B-008        |
| B-018 | Configuration / secrets          | B-008        |
| B-019 | Supabase JWT                     | B-018        |
| B-020 | api-ci.yml                       | B-010        |
| B-021 | Staging CD skeleton              | B-020        |


---



## Episode filming order


| Time  | Show on screen | Content                                                   |
| ----- | -------------- | --------------------------------------------------------- |
| 0:00  | Hook           | "Thin endpoints, fat domain — Minimal API + Onion + DDD." |
| 0:45  | B-001          | Tooling check — `dotnet --version`                        |
| 2:00  | Diagram        | Onion rings — dependency rule                             |
| 3:00  | B-002–B-003    | `dotnet new sln`, four projects live                      |
| 5:00  | B-004–B-006    | City aggregate + MediatR pipeline + GetCities handler     |
| 6:30  | B-007 Part 1   | Conduit library — Ping/Pong + pipeline demo               |
| 7:00  | B-008          | CitiesEndpoints — 3 lines, MediatR does the work          |
| 8:00  | B-016          | Break API on purpose — `pageSize=500` → 400               |
| 9:00  | B-011–B-013    | Docker Postgres + first migration                         |
| 10:00 | CTA            | Ep 5: B-015 — locations & categories                      |


**Short clips:** B-007 Part 1 (Conduit Ping/Pong pipeline) · B-008 (thin Minimal API route) · B-016 (validation) · B-002 (onion in 60s with diagram)

---



## Tech Lead sign-off checklist

Before merging Ep 4–5 PR:

- [ ] `dotnet build` — zero warnings (Directory.Build.props)
- [ ] `dotnet test` — all green, no DB for unit tests
- [ ] Onion dependency rule — no forbidden project refs
- [ ] Domain has zero infrastructure references
- [ ] No `Controllers/` folder — Minimal API endpoint modules only
- [ ] Route delegates only call `mediator.Send` — no EF or validation in endpoints
- [ ] Validators live next to queries in Application
- [ ] DTOs at API boundary — entities never serialized
- [ ] README: run locally + curl examples
- [ ] No secrets in git
- [ ] api-ci.yml green on PR

**Open decisions during build:**

- Lock **Minimal API (no controllers)** → add **D-017** to [decisions.md](../decisions.md)
- Lock **Supabase Auth** → add **D-018** to [decisions.md](../decisions.md)
- Confirm **.NET 10** in CI ([cicd.md](../../infrastructure/cicd.md))

---



## Related

- [context.md](../context.md) — current phase
- [roadmap.md](../roadmap.md) — Ep 4–7, 16
- [infrastructure/cicd.md](../../infrastructure/cicd.md)
- [infrastructure/terraform.md](../../infrastructure/terraform.md)
- [content/creator-assistant-prompt.md](../content/creator-assistant-prompt.md) — Q-005, Q-006 for episode-level tasks
- Interactive canvas: onion-ddd-architecture (Cursor Canvases)

