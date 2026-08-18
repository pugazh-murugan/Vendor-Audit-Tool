# Vendor Audit Tool — Backend File Summary

Quick reference for every controller, repository, and supporting file in the
`backend/` project. Useful for interview prep / explaining "what does this
file do" on demand.

---

## Controllers (`VendorAudit.Api/Controllers/`)

### `VendorsController.cs`
| Endpoint | Description |
|---|---|
| `GET /api/vendors` | Lists all vendors with open finding count + computed risk rating badge |
| `GET /api/vendors/{id}` | Vendor detail with all its findings |
| `GET /api/vendors/{id}/risk` | Risk assessment only (rating, reasoning, contributing finding IDs) |

### `FindingsController.cs`
| Endpoint | Description |
|---|---|
| `GET /api/vendors/{vendorId}/findings` | Findings for one vendor |
| `POST /api/vendors/{vendorId}/findings` | Add a new finding (title, notes, severity, status, date) |

### `AiAssistController.cs`
| Endpoint | Description |
|---|---|
| `POST /api/vendors/{vendorId}/ai-summary` | Generates the AI summary + risk rating for a vendor via `IAiAssistant` |

---

## Repositories (`VendorAudit.Infrastructure/Repositories/`)

**`VendorRepository.cs`**
Implements `IVendorRepository`: `GetAllAsync` (findings included), `GetByIdAsync`,
`GetWithFindingsAsync`, `AddAsync`. All reads use `AsNoTracking()` for performance.

**`FindingRepository.cs`**
Implements `IFindingRepository`: `GetByVendorIdAsync`, `GetByIdAsync`, `AddAsync`.

---

## Core interfaces (`VendorAudit.Core/Interfaces/`)

- **`IVendorRepository.cs`** / **`IFindingRepository.cs`** — contracts the
  repositories above implement; controllers depend on these, not EF Core directly.
- **`IRiskScoringService.cs`** — defines the `RiskAssessment` record
  (rating + reasoning + contributing finding IDs) and `Score(Vendor)`.
- **`IAiAssistant.cs`** — defines the `AiVendorSummary` record and
  `SummarizeVendorAsync(Vendor)`.

---

## Core entities (`VendorAudit.Core/Entities/`)

- **`Vendor.cs`** — Id, Name, Category, ContactEmail, OnboardedOn, collection of Findings.
- **`AuditFinding.cs`** — Id, VendorId, Title, Notes, Severity, Status, Date.
- **`Enums.cs`** — `Severity` (Low/Medium/High), `FindingStatus` (Open/InRemediation/Closed),
  `RiskRating` (Low/Medium/High).

---

## AI / scoring logic (`VendorAudit.Infrastructure/AI/`)

**`RiskScoringService.cs`**
Deterministic rule engine: any open High finding → High; any open Medium
finding, or 2+ open Low findings → Medium; otherwise → Low. Closed findings
never count toward the rating.

**`StubAiAssistant.cs`**
Default `IAiAssistant` implementation. No external API calls — builds a
narrative summary from the vendor's actual findings and reuses
`RiskScoringService` for the rating, so the output is always grounded and
can never contradict the recorded audit data.

**`AzureOpenAiAssistant.cs`**
Not active by default; a commented-out wiring sketch showing how a real
Semantic Kernel + Azure OpenAI call would plug into the same `IAiAssistant`
interface for production.

---

## Data layer (`VendorAudit.Infrastructure/Data/`)

**`AuditDbContext.cs`**
EF Core DbContext — defines `Vendors`/`Findings` DbSets and the cascade-delete
relationship between them.

**`DbSeeder.cs`**
Inserts 5 sample vendors with findings on first run only
(`if (db.Vendors.Any()) return;` guards against duplicate seeding).

---

## DTOs (`VendorAudit.Api/DTOs/`)

- **`VendorDtos.cs`** — `VendorListItemDto`, `VendorDetailDto`, plus mapping
  extensions from `Vendor` entity → DTO.
- **`FindingDtos.cs`** — `FindingDto`, `CreateFindingDto`, plus mapping extensions.
- **`AiSummaryDto.cs`** — `AiSummaryDto`, plus mapping extension from
  `AiVendorSummary` → DTO.

---

## Other

**`Program.cs`**
Wires up DI (repositories, `RiskScoringService`, stub-vs-real `IAiAssistant`
based on config), sets up EF Core with SQLite, seeds data on startup,
configures CORS/Swagger/API-key middleware.

**`Middleware/ApiKeyMiddleware.cs`**
Checks the `X-Api-Key` header against `Auth:ApiKey` config on every request
except `/swagger` and `/health`.

---

## Request flow (end to end)

```
Angular component
  -> VendorService (HTTP client)
    -> Controller (VendorsController / FindingsController / AiAssistController)
      -> Repository (IVendorRepository / IFindingRepository)  [data]
      -> RiskScoringService                                    [rating rule]
      -> IAiAssistant (StubAiAssistant)                        [AI summary]
        -> AuditDbContext (EF Core) -> SQLite
```

Controllers never talk to `AuditDbContext` directly — they only depend on the
interfaces, which is what makes `RiskScoringService` and `StubAiAssistant`
unit-testable without a database (see `backend/tests/VendorAudit.Tests/`).
