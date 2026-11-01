# Document Management REST API

A small ASP.NET Core Web API for uploading, listing, retrieving, downloading and deleting documents.
Metadata is stored in a relational database; file contents are stored on disk.

## 1. Overview

| Concern | Choice |
|---|---|
| Language / framework | C# / ASP.NET Core 8 (controllers) |
| Database | SQLite by default (zero setup), SQL Server supported by configuration |
| Data access | Entity Framework Core 8 |
| File storage | Local disk behind an `IFileStorage` interface |
| Tests | xUnit + `WebApplicationFactory` (integration tests against the real pipeline) |
| API docs | Swagger UI (Development only) |

## 2. Prerequisites

- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)
- An IDE: Visual Studio 2022, JetBrains Rider, or VS Code with the C# Dev Kit
- SQL Server is **optional** (see section 4)

## 3. Run it (IDE)

1. Open `DocumentManagement.sln`.
2. Set `DocumentManagement.Api` as the startup project.
3. Press **F5** / Run. Swagger UI opens at `http://localhost:5080/swagger`.

CLI alternative:

```bash
dotnet run --project src/DocumentManagement.Api
```

On first start the app creates the database schema and the `Storage` folder automatically.
Both are git-ignored.

## 4. Database configuration

**Default (SQLite):** nothing to do. A `documents.db` file is created in the API project folder.

**SQL Server:** set the provider and connection string *without committing secrets*, e.g. with user-secrets:

```bash
cd src/DocumentManagement.Api
dotnet user-secrets init
dotnet user-secrets set "Database:Provider" "SqlServer"
dotnet user-secrets set "ConnectionStrings:Default" "Server=localhost;Database=DocumentManagement;Trusted_Connection=True;TrustServerCertificate=True"
```

(Environment variables work too: `Database__Provider`, `ConnectionStrings__Default`.)
The schema is created on startup. If you prefer to create it yourself, run `database/schema.sql`.

## 5. Run the tests

```bash
dotnet test
```

Tests use an isolated temporary SQLite database and temporary storage folder, so they never touch your real data.
In an IDE, use the Test Explorer.

## 6. Endpoints

| Method | Route | Description | Success | Errors |
|---|---|---|---|---|
| POST | `/api/documents` | Upload (`multipart/form-data`, field `file`) | 201 + metadata, `Location` header | 400, 413, 415 |
| GET | `/api/documents` | List metadata (no file contents) | 200 | |
| GET | `/api/documents/{id}` | Metadata of one document | 200 | 404 |
| GET | `/api/documents/{id}/download` | Download the file | 200 (file) | 404 (record or file missing) |
| DELETE | `/api/documents/{id}` | Delete record + stored file | 204 | 404, 500 |

Errors are returned as RFC 7807 `application/problem+json` with a useful `detail` message.

Example:

```bash
curl -F "file=@report.pdf" http://localhost:5080/api/documents
```

## 7. Architecture and design decisions

**Structure** (kept deliberately small for the scope):

```
src/DocumentManagement.Api
  Controllers/   thin HTTP layer (routing, status codes)
  Services/      DocumentService: validation + orchestration; FileTypeRules; FileNameSanitizer
  Storage/       IFileStorage + LocalFileStorage
  Data/          AppDbContext (EF Core)
  Entities/ Dtos/ Options/ Errors/
tests/DocumentManagement.Tests   integration tests
```

**Data access: EF Core.** The model is one table, and EF gives provider portability (SQLite <-> SQL Server)
for free. The schema is created with `EnsureCreated()` to keep setup to a single click.

**File storage: local disk behind `IFileStorage`.** The database holds only metadata and the storage identifier,
so the storage can be swapped for blob storage without touching the service logic.

**Error handling:** services throw `ApiException(status, message)` for expected failures; one global
`IExceptionHandler` converts exceptions to problem-details responses. Unexpected errors return a generic 500
(details are logged, not leaked).

### Security and validation

- **10 MB limit** per file (`Documents:MaxFileSizeBytes`), enforced in the service (413) and also by
  Kestrel/form limits (request body capped at limit + 1 MB of multipart overhead).
- **Safe storage names:** files are stored as `{guid}{ext}`. The client's filename is never used as a path;
  it is sanitized (directory parts and control characters removed) and kept only as display metadata.
- **Path traversal defence in depth:** `LocalFileStorage` accepts only names matching `^[0-9a-f]{32}\.[a-z0-9]{1,5}$`
  and verifies the resolved path stays inside the storage root.
- **Collisions:** random GUID names plus `FileMode.CreateNew` (never overwrites).
- **Content checks:** see assumptions below.

### Consistency on failure

- **Upload:** file is written first, then the DB row. If the DB write fails, the file is removed (compensation).
  A partially written file is removed by the storage layer.
- **Delete:** inside a DB transaction the row is removed, then the file is deleted, then the transaction commits.
  If file deletion fails, the transaction rolls back (record and file both remain; request returns 500 and can be retried).
  This is covered by a test.

## 8. Assumptions

- **Supported types (allow-list):** `.pdf .png .jpg .jpeg .docx .xlsx .pptx .doc .xls .txt .csv`.
- The file **extension** decides the type; the file's first bytes (magic numbers) must match it
  (e.g. a `.pdf` must start with `%PDF`; `.docx/.xlsx/.pptx` must be ZIP containers; `.txt/.csv` must not contain NUL bytes).
- The client-sent `Content-Type` header is **not trusted**; the stored content type is derived from the extension.
- Empty files are rejected.
- No authentication/authorization (as specified).

## 9. Known limitations / what I would do next

- `EnsureCreated()` instead of EF migrations; with real schema changes I would add migrations.
- Magic-number checks are a sanity check, not malware scanning. Production would add antivirus scanning.
- A crash between "file deleted" and "transaction committed" can leave a record whose file is gone. Download then returns a clear 404 and the delete can be retried. A background reconciliation job would clean these up.
- Upload is "file first, row second": a hard process crash between the two steps leaves an orphan file. A periodic orphan-file sweep would handle it.
- No pagination on `GET /api/documents`.
- Files are stored unencrypted on local disk; no per-user isolation.
- Duplicate-content detection (hashing) is not implemented.

## 10. AI-assisted development

> **Edit this section so it reflects what you actually did before submitting.** The text below is a draft.

- **Tools used:** Claude (Anthropic) was used to generate the initial solution structure, implementation, tests and this README.
- **How it was used:** I gave it the assessment PDF and asked for a complete runnable solution; I then ran the application and the tests locally.
- **Reviewed / modified by me:** _(list what you read, ran and changed, e.g. validation rules, delete/transaction flow, test cases)_
- **Least confident areas:** _(be honest; suggestions: multipart/form size-limit behaviour in `Program.cs`, the delete transaction ordering, EF Core provider differences between SQLite and SQL Server, SQL Server path was not exercised as heavily as SQLite)_
