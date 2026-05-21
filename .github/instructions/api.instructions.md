---
description: 'Guidelines for building C# REST API applications'
applyTo: ["src/*.Api/**", "src/*.Application/**"]
---     

# \<ProjectName\>.Api — Copilot Instructions

## Project Overview

`<ProjectName>.Api` is an **ASP.NET Core Web API** targeting **.NET 10**, backed by a SQL Server database via Entity Framework Core.
The API is orchestrated by a **.NET Aspire** `AppHost` alongside a frontend application.

---

## Solution Structure

```
<ProjectName>/
├── <ProjectName>.AppHost/          # .NET Aspire orchestration host
├── <ProjectName>.Api/              # ASP.NET Core REST API (presentation layer only)
│   ├── Controllers/                # API controllers (HTTP handling, validation, logging)
│   ├── Registry.cs                 # Dependency injection registrations (API-specific)
│   └── Program.cs                  # App entry point and middleware pipeline
├── <ProjectName>.Api.Tests/        # Unit tests for <ProjectName>.Api (controller tests)
├── <ProjectName>.Application/      # Core business logic and data access library
│   ├── Data/                       # EF Core DbContext (<ProjectName>DbContext)
│   ├── DTO/                        # Data Transfer Objects for API input/output
│   ├── Entities/                   # EF Core entity models mapped to DB tables
│   ├── Helpers/                    # Mapper (entity↔DTO) and GitHubIssueHelper
│   ├── Repositories/               # Data access layer (interfaces + implementations)
│   ├── Services/                   # Business logic layer (interfaces + implementations)
│   └── Registry.cs                 # Dependency injection registrations (Application layer)
├── <ProjectName>.Application.Tests/ # Unit tests for <ProjectName>.Application
└── <ProjectName>.SqlDb/            # SQL database project
```

---

## Architecture & Layering

The project follows a strict **layered architecture** with clear separation between the presentation layer (`<ProjectName>.Api`) and the application layer (`<ProjectName>.Application`).

### Layer Call Chain

```
Controller (<ProjectName>.Api) 
    → Service (<ProjectName>.Application) 
        → Repository (<ProjectName>.Application) 
            → <ProjectName>DbContext (<ProjectName>.Application) 
                → SQL Server
```

### Layer Responsibilities

**\<ProjectName\>.Api (Presentation Layer)**
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

**\<ProjectName\>.Application (Business & Data Access Layer)**
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
- **DTOs** are the public contract exposed to `<ProjectName>.Api`

### Key Architectural Rules

1. **DTOs are the boundary** — `<ProjectName>.Application` services accept and return DTOs, never entities
2. **Entities never escape** — Entity classes never leave `<ProjectName>.Application`
3. **No EF Core in `<ProjectName>.Api`** — The API project has no Entity Framework references
4. **Controllers are thin** — Controllers delegate all work to services and only handle HTTP concerns
5. **Mapping in services** — Entity↔DTO conversion happens in the service layer, not controllers

---

## Adding a New Resource

When adding a new resource (e.g., `Contact`), follow this checklist in order:

1. **Entity** — Add `Entities/Contact.cs` with `[Table]`, `[Key]`, `[Required]`, and `[MaxLength]` attributes. Include `CreatedDate` and `LastModifiedDate` system fields.
2. **DTO** — Add `DTO/ContactDTO.cs` mirroring the entity's public fields (no EF attributes).
3. **DbContext** — Add `DbSet<Contact> Contacts` to `<ProjectName>DbContext` and configure the table mapping in `OnModelCreating`.
4. **Repository interface** — Add `Repositories/IContactRepository.cs` with async CRUD methods.
5. **Repository implementation** — Add `Repositories/ContactRepository.cs` implementing the interface. Set `CreatedDate` and `LastModifiedDate` in `AddAsync`; update `LastModifiedDate` in `UpdateAsync`.
6. **Service interface** — Add `Services/IContactService.cs`.
7. **Service implementation** — Add `Services/ContactService.cs` delegating all calls to the repository.
8. **Controller** — Add `Controllers/ContactController.cs` following the patterns described below.
9. **Registry** — Register the repository and service as `Scoped` in `Registry.cs`.
10. **Tests** — Add unit tests in the appropriate test project: controller tests belong in `<ProjectName>.Api.Tests`; service and repository tests belong in `<ProjectName>.Application.Tests`.

---

## Testing Conventions

- **`<ProjectName>.Api.Tests`** contains all unit tests for the API layer, including controller tests that verify HTTP handling, routing, validation, and error responses.
- **`<ProjectName>.Application.Tests`** contains all unit tests for the application layer, covering business logic in services and data access in repositories.
- Test class names should mirror the class under test with a `Tests` suffix (e.g., `ContactControllerTests`, `ContactServiceTests`).
- Follow the Arrange-Act-Assert pattern in all test methods.
- Use meaningful test method names that describe the scenario and expected outcome.

---

## Controller Conventions

- Inherit from `ControllerBase`; decorate with `[ApiController]` and `[Route("api/[controller]")]`.
- Inject `IXxxService` (from `<ProjectName>.Application.Services`) and `ILogger<XxxController>` via constructor.
- Instantiate `GitHubIssueHelper` directly in the controller constructor (it lives in `<ProjectName>.Application.Helpers`).
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

- **Entities** live in `<ProjectName>.Application.Entities` and carry EF Core / data annotations (`[Table]`, `[Key]`, `[Required]`, `[MaxLength]`).
- **DTOs** live in `<ProjectName>.Application.DTO` and carry no EF Core attributes.
- DTOs must mirror the entity's fields that are safe to expose publicly.
- System fields (`CreatedDate`, `LastModifiedDate`) are present in both entities and DTOs but are **set automatically in the repository**, not by the caller.
- All string properties default to `string.Empty` (not `null`) for required fields.
- **Entities never leave the `<ProjectName>.Application` layer** — they are internal implementation details.
- **DTOs are the public contract** — they are shared between `<ProjectName>.Application` and `<ProjectName>.Api`.

---

## Service Rules

- **Services** live in `<ProjectName>.Application.Services` with both interface (`IXxxService`) and implementation (`XxxService`).
- **Services always accept DTOs as input parameters** (e.g., `Task<ResourceDto> CreateAsync(ResourceDto dto)`).
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

### \<ProjectName\>.Application/Registry.cs

All **application layer components** are registered here with **Scoped** lifetime.

**Responsibilities:**
- Register all repositories (`IXxxRepository`, `XxxRepository`)
- Register all services (`IXxxService`, `XxxService`)
- Register `<ProjectName>DbContext` with SQL Server connection string from `"DefaultConnection"`
- Exposes `Register<ProjectName>Application` extension method

### \<ProjectName\>.Api/Registry.cs

Contains **API-specific registrations** and delegates to `<ProjectName>.Application`.

**Responsibilities:**
- Call `Register<ProjectName>Application` to register all application layer dependencies
- Register any API-specific services (filters, middleware, API-only utilities)
- Exposes `Register<ProjectName>Api` extension method called from `Program.cs`

### Registration Flow

```
Program.cs
    ↓
builder.Services.Register<ProjectName>Api(config)  ← <ProjectName>.Api/Registry.cs
    ↓
services.Register<ProjectName>Application(config)  ← <ProjectName>.Application/Registry.cs
    ↓
Repositories, Services, DbContext registered
```

**Key Points:**
- `<ProjectName>.Api` never directly registers application layer services
- `<ProjectName>.Api` only calls `Register<ProjectName>Application` and adds API-specific registrations
- All business logic dependencies (services, repositories, DbContext) are owned by `<ProjectName>.Application`

---

## Database & EF Core

- **DbContext**: `<ProjectName>DbContext` in `<ProjectName>.Application.Data`
- **Connection string key**: `"DefaultConnection"` (provided by .NET Aspire or `appsettings.json`)
- Always use async EF Core methods (`ToListAsync`, `FirstOrDefaultAsync`, `SaveChangesAsync`)
- **Entity Configuration**: Use Fluent API in `OnModelCreating` for table mappings, relationships, and indexes
- **`<ProjectName>.Api` has NO EF Core references** — all database operations happen in `<ProjectName>.Application`

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

`GitHubIssueHelper` (in `<ProjectName>.Application.Helpers/`) posts uncaught exceptions from controller `POST` actions as GitHub issues to the configured repository. It is instantiated directly in each controller constructor (not DI-injected). Exceptions in `PUT` and `DELETE` are not currently wired to this helper — follow the existing pattern when adding new endpoints.

---

## Aspire Orchestration

The `<ProjectName>.AppHost` project orchestrates the solution:

- The API is registered as `"<ProjectName>-API"` and exposed with a Scalar link.
- The frontend app references and waits for the API.
- When running locally, start via the AppHost (`<ProjectName>.AppHost`) — not the API project directly.

---

## Coding Conventions

- Nullable reference types are **enabled** (`<Nullable>enable`).
- Implicit usings are **enabled**.
- Use `string.Empty` instead of `""` for default string values.
- Prefer `async`/`await` throughout; never use `.Result` or `.Wait()`.
- Use `ILogger<T>` structured logging in controllers; do not use `Console.Write`.
- Follow existing file/folder naming: `PascalCase` for all C# files and types.
