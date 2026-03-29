# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Test Commands

```bash
# Build the full template
dotnet build Clean.Architecture.slnx

# Build in Release mode (as CI does)
dotnet build Clean.Architecture.slnx -c Release

# Run all tests
dotnet test Clean.Architecture.slnx

# Run a single test project
dotnet test tests/Clean.Architecture.UnitTests
dotnet test tests/Clean.Architecture.IntegrationTests
dotnet test tests/Clean.Architecture.FunctionalTests

# Run a single test by name
dotnet test tests/Clean.Architecture.UnitTests --filter "FullyQualifiedName~ContributorConstructor"

# Run the web app
dotnet run --project src/Clean.Architecture.Web
```

### EF Core Migrations (run from `src/Clean.Architecture.Web/`)

```bash
dotnet ef migrations add MigrationName -c AppDbContext -p ../Clean.Architecture.Infrastructure/Clean.Architecture.Infrastructure.csproj -s Clean.Architecture.Web.csproj -o Data/Migrations

dotnet ef database update -c AppDbContext -p ../Clean.Architecture.Infrastructure/Clean.Architecture.Infrastructure.csproj -s Clean.Architecture.Web.csproj
```

## Architecture

This is the **Ardalis Clean Architecture** template for .NET 10. The repo contains two template variants (`/src` = full, `/MinimalClean` = minimal) plus a sample app in `/sample`. The primary solution is `Clean.Architecture.slnx`.

### Dependency Flow (Full Template)

```
Core ← UseCases ← Infrastructure
Core ← UseCases ← Web
                      Web → Infrastructure (directly, for DI wiring)
```

**Core** has no outer dependencies. It is never allowed to reference UseCases, Infrastructure, or Web.

### Project Responsibilities

| Project | Role |
|---------|------|
| **Core** | Domain model: entities, aggregates, value objects (Vogen), domain events, specifications, repository interfaces, domain services |
| **UseCases** | CQRS handlers: commands (mutations via repositories) and queries (reads, can bypass repository pattern), DTOs, query service interfaces |
| **Infrastructure** | EF Core (`AppDbContext`), repository implementations, email sender, external services. References Core + UseCases |
| **Web** | FastEndpoints API (REPR pattern), DI configuration, middleware, Serilog, Swagger/Scalar. Entry point |
| **ServiceDefaults** | Shared Aspire/OTel setup |
| **AspireHost** | .NET Aspire orchestration (optional) |

### Key Patterns

- **FastEndpoints + REPR**: Each API endpoint is a class inheriting `Endpoint<TRequest, TResponse>`. One endpoint per file. Request/response/validator in separate files (`Create.CreateRequest.cs`, `Create.CreateValidator.cs`)
- **CQRS via Mediator** (source-generated): Commands → `*Command` + `*Handler`. Queries → `*Query` + `*Handler`
- **Value Objects**: Uses [Vogen](https://github.com/SteveDunn/Vogen) source generators (e.g., `ContributorName`, `ContributorId`, `PhoneNumber`)
- **Specifications**: `Ardalis.Specification` for encapsulating query logic (e.g., `ContributorByIdSpec`)
- **Domain Events**: Entities call `RegisterDomainEvent()`. Handlers in Core; infrastructure dispatch lives in Infrastructure
- **Results**: `Ardalis.Result<T>` for returning success/error from use cases. Web layer maps via `Ardalis.Result.AspNetCore`

### Validation Layers

1. **Web/API level**: FluentValidation on request DTOs (FastEndpoints `Validator<T>`)
2. **Use Case level**: Defensive validation in command handlers
3. **Domain level**: Business invariants throw exceptions (assumes pre-validated input); Vogen validates value object construction

## Code Conventions

- **.NET 10** / `net10.0` target framework, C# `latest` lang version
- File-scoped namespaces (`csharp_style_namespace_declarations = file_scoped:warning`)
- Central package management in `Directory.Packages.props` — use `<PackageReference Include="..." />` without Version attribute
- `Directory.Build.props` sets: `TreatWarningsAsErrors`, `Nullable=enable`, `ImplicitUsings=enable`
- Primary constructors for required dependencies, but **always assign to private fields** — never use primary constructor parameters directly
- Private fields prefixed with `_`
- All braces required except single-line `return`/`throw`
- 2-space indentation, UTF-8 BOM, CRLF line endings

### Web Endpoint File Organization

```
Contributors/
  Create.cs                      # Endpoint class (all types in one file is ok for simple cases)
  Delete.cs                      # Endpoint only
  Delete.DeleteContributorRequest.cs   # Request DTO
  Delete.DeleteContributorValidator.cs # FluentValidation
  GetById.cs                     # Endpoint only
  GetById.GetContributorByIdRequest.cs
  GetById.GetContributorValidator.cs
```

### Test Projects

| Project | Tests | Framework | Notes |
|---------|-------|-----------|-------|
| **UnitTests** | Core logic, use case handlers | xUnit + NSubstitute + Shouldly | References Core + UseCases |
| **IntegrationTests** | EF Core repository with InMemory DB | xUnit + NSubstitute + Shouldly | References Infrastructure |
| **FunctionalTests** | API endpoints via `WebApplicationFactory` | xUnit + Shouldly + Testcontainers | Subcutaneous HTTP tests |
| **AspireTests** | Aspire host integration | xUnit | References AspireHost |

## Template Structure Note

This repo serves dual purpose: it's both a working template and a NuGet template source. The `/sample` folder contains a complete working app (`NimblePros.SampleToDo`). When using as a template, replace `Ardalis.SharedKernel` with your own shared kernel. The sample is for reference — delete the template's sample code (Contributor aggregate) once you understand the patterns.
