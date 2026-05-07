---
applyTo: ["src/Banking.Api/**", "src/Banking.Application/**"]
---     

# Banking.Api — Copilot Instructions

## Project Overview

`Banking.Api` is an **ASP.NET Core Web API** targeting **.NET 10**, serving as the backend for a Banking system. backed by a SQL Server database via Entity Framework Core.
The API is orchestrated by a **.NET Aspire** `AppHost` alongside a React frontend application (`banking-app`).

---

## Solution Structure

```
Banking/
├── Banking.AppHost/          # .NET Aspire orchestration host
├── Banking.Api/              # ASP.NET Core REST API (presentation layer only)
│   ├── Controllers/          # API controllers (HTTP handling, validation, logging)
│   ├── Registry.cs           # Dependency injection registrations (API-specific)
│   └── Program.cs            # App entry point and middleware pipeline
├── Banking.Api.Tests/        # Unit tests for Banking.Api (controller tests)
├── Banking.Application/      # Core business logic and data access library
│   ├── Data/                 # EF Core DbContext (BankingDbContext)
│   ├── DTO/                  # Data Transfer Objects for API input/output
│   ├── Entities/             # EF Core entity models mapped to DB tables
│   ├── Helpers/              # Mapper (entity↔DTO) and GitHubIssueHelper
│   ├── Repositories/         # Data access layer (interfaces + implementations)
│   ├── Services/             # Business logic layer (interfaces + implementations)
│   └── Registry.cs           # Dependency injection registrations (Application layer)
├── Banking.Application.Tests/ # Unit tests for Banking.Application (service and repository tests)
└── Banking.SqlDb/            # SQL database project
```

---

## Architecture & Layering

The project follows a strict **layered architecture** with clear separation between the presentation layer (`Banking.Api`) and the application layer (`Banking.Application`).

### Layer Call Chain

```
Controller (Banking.Api) 
    → Service (Banking.Application) 
        → Repository (Banking.Application) 
            → BankingDbContext (Banking.Application) 
                → SQL Server
```

### Layer Responsibilities

**Banking.Api (Presentation Layer)**
- **Controllers** handle HTTP concerns only:
  - Routing and HTTP method handling
  - Model validation (via `ValidateXxx` methods)
  - Structured logging
  - Exception handling and error responses
  - Returning DTOs from services
- **Controllers do NOT:**
  - Perform DTO↔Entity mapping (services do this)
  - Reference Entity classes or DbContext
  - Contain business logic
  - Call repositories directly

**Banking.Application (Business & Data Access Layer)**
- **Services** contain business logic:
  - Accept DTOs as input
  - Convert DTOs to entities using `.ToEntity<T>()`
  - Orchestrate repository calls
  - Convert entities to DTOs using `.ToDto<T>()`
  - Return DTOs to controllers
- **Repositories** own all data access:
  - Execute all EF Core queries
  - Manage database persistence (`SaveChangesAsync`)
  - Set system fields (`CreatedDate`, `LastModifiedDate`)
  - Work exclusively with entities
- **DbContext** provides EF Core data access
- **Entities** are internal implementation details that never leave this layer
- **DTOs** are the public contract exposed to Banking.Api

### Key Architectural Rules

1. **DTOs are the boundary** — Banking.Application services accept and return DTOs, never entities
2. **Entities never escape** — Entity classes never leave Banking.Application
3. **No EF Core in Banking.Api** — The API project has no Entity Framework references
4. **Controllers are thin** — Controllers delegate all work to services and only handle HTTP concerns
5. **Mapping in services** — Entity↔DTO conversion happens in the service layer, not controllers

---

## Adding a New Resource

When adding a new resource (e.g., `Contact`), follow this checklist in order:

1. **Entity** — Add `Entities/Contact.cs` with `[Table]`, `[Key]`, `[Required]`, and `[MaxLength]` attributes. Include `CreatedDate` and `LastModifiedDate` system fields.
2. **DTO** — Add `DTO/ContactDTO.cs` mirroring the entity's public fields (no EF attributes).
3. **DbContext** — Add `DbSet<Contact> Contacts` to `BankingDbContext` and configure the table mapping in `OnModelCreating`.
4. **Repository interface** — Add `Repositories/IContactRepository.cs` with async CRUD methods.
5. **Repository implementation** — Add `Repositories/ContactRepository.cs` implementing the interface. Set `CreatedDate` and `LastModifiedDate` in `AddAsync`; update `LastModifiedDate` in `UpdateAsync`.
6. **Service interface** — Add `Services/IContactService.cs`.
7. **Service implementation** — Add `Services/ContactService.cs` delegating all calls to the repository.
8. **Controller** — Add `Controllers/ContactController.cs` following the patterns described below.
9. **Registry** — Register the repository and service as `Scoped` in `Registry.cs`.
10. **Tests** — Add unit tests in the appropriate test project: controller tests belong in `Banking.Api.Tests`; service and repository tests belong in `Banking.Application.Tests`.

---

## Testing Conventions

- **Banking.Api.Tests** contains all unit tests for the API layer, including controller tests that verify HTTP handling, routing, validation, and error responses.
- **Banking.Application.Tests** contains all unit tests for the application layer, covering business logic in services and data access in repositories.
- Test class names should mirror the class under test with a `Tests` suffix (e.g., `ContactControllerTests`, `ContactServiceTests`).
- Follow the Arrange-Act-Assert pattern in all test methods.
- Use meaningful test method names that describe the scenario and expected outcome.

---

## Controller Conventions

- Inherit from `ControllerBase`; decorate with `[ApiController]` and `[Route("api/[controller]")]`.
- Inject `IXxxService` (from `Banking.Application.Services`) and `ILogger<XxxController>` via constructor.
- Instantiate `GitHubIssueHelper` directly in the controller constructor (it lives in `Banking.Application.Helpers`).
- All action methods are `async Task<ActionResult<T>>`.
- Use **structured logging** with named placeholders: `_logger.LogInformation("Getting contact by id: {Id}", id)`.
- Apply **server-side validation** in `POST` and `PUT` actions via a private `ValidateXxx(dto)` method that returns `List<string>` of error messages. Return `UnprocessableEntity` or `BadRequest` with `new { errors = validationErrors }`.
- Wrap `POST` actions in try/catch; on exception call `_githubIssueHelper.CreateIssueAsync(...)` and return `StatusCode(500, ...)`.
- **Do NOT map entities in controllers** — Services return DTOs directly. Controllers simply pass DTOs to services and return DTOs from services.
- Return `CreatedAtAction(nameof(GetById), new { id = ... }, dto)` from `POST`.
- Return `NotFound()` when `GetById` yields null.
- Return `BadRequest()` when route `id` does not match DTO id in `PUT`.

---

## DTO & Entity Rules

- **Entities** live in `Banking.Application.Entities` and carry EF Core / data annotations (`[Table]`, `[Key]`, `[Required]`, `[MaxLength]`).
- **DTOs** live in `Banking.Application.DTO` and carry no EF Core attributes.
- DTOs must mirror the entity's fields that are safe to expose publicly.
- System fields (`CreatedDate`, `LastModifiedDate`) are present in both entities and DTOs but are **set automatically in the repository**, not by the caller.
- All string properties default to `string.Empty` (not `null`) for required fields.
- **Entities never leave the Banking.Application layer** — they are internal implementation details.
- **DTOs are the public contract** — they are shared between Banking.Application and Banking.Api.

---

## Service Rules

- **Services** live in `Banking.Application.Services` with both interface (`IXxxService`) and implementation (`XxxService`).
- **Services always accept DTOs as input parameters** (e.g., `Task<AccountOpeningDto> CreateAsync(AccountOpeningDto dto)`).
- **Services always return DTOs** — never return entities directly.
- **Entity↔DTO conversion happens inside the service layer** using the `Mapper` helper:
  - **Input**: Convert DTO to entity using `.ToEntity<TEntity>()`
  - **Processing**: Call repository methods with entities
  - **Output**: Convert entity to DTO using `.ToDto<TDto>()` before returning
- Services delegate all data access to repositories; no direct DbContext usage in services.
- Services contain business logic and orchestration; repositories handle only data access.

---

## Dependency Injection

The solution uses **two separate Registry.cs files** for dependency injection:

### Banking.Application/Registry.cs

All **application layer components** are registered here with **Scoped** lifetime.

**Responsibilities:**
- Register all repositories (`IXxxRepository`, `XxxRepository`)
- Register all services (`IXxxService`, `XxxService`)
- Register `BankingDbContext` with SQL Server connection string from `"DefaultConnection"`
- Exposes `RegisterBankingApplication` extension method

### Banking.Api/Registry.cs

Contains **API-specific registrations** and delegates to Banking.Application

**Responsibilities:**
- Call `RegisterBankingApplication` to register all application layer dependencies
- Register any API-specific services (filters, middleware, API-only utilities)
- Exposes `RegisterBankingApi` extension method called from `Program.cs`

### Registration Flow

```
Program.cs
    ↓
builder.Services.RegisterBankingApi(config)  ← Banking.Api/Registry.cs
    ↓
services.RegisterBankingApplication(config)  ← Banking.Application/Registry.cs
    ↓
Repositories, Services, DbContext registered
```

**Key Points:**
- Banking.Api never directly registers application layer services
- Banking.Api only calls `RegisterBankingApplication` and adds API-specific registrations
- All business logic dependencies (services, repositories, DbContext) are owned by Banking.Application

---

## Database & EF Core

- **DbContext**: `BankingDbContext` in `Banking.Application.Data`
- **Current Tables**: `Customer`, `Contact` — one entity per table, configured in `OnModelCreating`
- **Connection string key**: `"DefaultConnection"` (provided by .NET Aspire or `appsettings.json`)
- Always use async EF Core methods (`ToListAsync`, `FirstOrDefaultAsync`, `SaveChangesAsync`)
- **Entity Configuration**: Use Fluent API in `OnModelCreating` for table mappings, relationships, and indexes
- **Banking.Api has NO EF Core references** — all database operations happen in Banking.Application

---

## API Documentation

- OpenAPI is enabled via `Microsoft.AspNetCore.OpenApi` and served at `/openapi`.
- The **Scalar** UI (`Scalar.AspNetCore`) is mounted at `/scalar` and is the default landing page (`/` redirects to `/scalar`).
- OpenAPI and Scalar are enabled in all environments (the `IsDevelopment` guard is intentionally commented out).

---

## CORS

A single `"AllowAll"` policy is configured, permitting any origin, header, and method. This is intentional for the current development stage. Do not restrict origins without updating the Aspire frontend reference as well.

---

## Error Handling & GitHub Issue Integration

`GitHubIssueHelper` (in `Banking.Application.Helpers/`) posts uncaught exceptions from controller `POST` actions as GitHub issues to the configured repository. It is instantiated directly in each controller constructor (not DI-injected). Exceptions in `PUT` and `DELETE` are not currently wired to this helper — follow the existing pattern when adding new endpoints.

---

## Aspire Orchestration

The `Banking.AppHost` project orchestrates the solution:

- The API is registered as `"Banking-API"` and exposed with a Scalar link.
- The React frontend (`banking-app`) is an npm app that references and waits for the API.
- When running locally, start via the AppHost (`Banking.AppHost`) — not the API project directly.

---

## Coding Conventions

- Nullable reference types are **enabled** (`<Nullable>enable`).
- Implicit usings are **enabled**.
- Use `string.Empty` instead of `""` for default string values.
- Prefer `async`/`await` throughout; never use `.Result` or `.Wait()`.
- Use `ILogger<T>` structured logging in controllers; do not use `Console.Write`.
- Follow existing file/folder naming: `PascalCase` for all C# files and types.
