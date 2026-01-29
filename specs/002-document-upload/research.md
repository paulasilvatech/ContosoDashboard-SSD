# Research: Document Upload and Management

**Feature**: 002-document-upload  
**Date**: 2026-01-28  
**Purpose**: Resolve technical unknowns and establish best practices before design

## Research Summary

All technical questions have been resolved. The implementation will use Blazor Server's `InputFile` component for uploads, local filesystem storage with `IFileStorageService` abstraction, and follow existing service patterns in ContosoDashboard.

---

## 1. File Upload in Blazor Server

**Question**: How to handle large file uploads (up to 25MB) in Blazor Server?

**Decision**: Use `InputFile` component with `IBrowserFile` and MemoryStream pattern

**Rationale**: 
- Blazor Server's `InputFile` component handles file selection and streaming
- SignalR connection requires careful stream handling to avoid disposal issues
- MemoryStream pattern copies file data immediately, preventing stream reuse errors

**Pattern**:
```csharp
// Extract metadata BEFORE opening stream
var fileName = browserFile.Name;
var fileSize = browserFile.Size;
var contentType = browserFile.ContentType;

// Copy to MemoryStream immediately
using var memoryStream = new MemoryStream();
await using (var fileStream = browserFile.OpenReadStream(maxAllowedSize: 25 * 1024 * 1024))
{
    await fileStream.CopyToAsync(memoryStream);
}
memoryStream.Position = 0;

// Pass memoryStream to storage service
await _fileStorageService.UploadAsync(memoryStream, storagePath, contentType);
```

**Alternatives Considered**:
- Direct stream passing: Rejected due to Blazor stream disposal issues
- Chunked upload: Rejected as unnecessary complexity for 25MB limit

---

## 2. File Storage Architecture

**Question**: How to store files securely with Azure migration path?

**Decision**: `IFileStorageService` interface with `LocalFileStorageService` implementation

**Rationale**:
- Interface abstraction enables DI-based swapping for Azure Blob Storage
- Files stored outside `wwwroot` in `AppData/uploads/` for security
- GUID-based filenames prevent path traversal attacks
- Controller endpoints serve files with authorization checks

**Interface Design**:
```csharp
public interface IFileStorageService
{
    Task<string> UploadAsync(Stream fileStream, string storagePath, string contentType);
    Task DeleteAsync(string storagePath);
    Task<Stream> DownloadAsync(string storagePath);
    Task<bool> ExistsAsync(string storagePath);
}
```

**Storage Path Pattern**: `{userId}/{projectId|personal}/{guid}.{extension}`

**Alternatives Considered**:
- Store in wwwroot: Rejected (security - files would be publicly accessible)
- Store in database as BLOB: Rejected (performance, database bloat)
- Azure Blob Storage direct: Rejected (violates offline-first requirement)

---

## 3. File Serving and Preview

**Question**: How to serve files for download and preview with authorization?

**Decision**: ASP.NET Core Controller with authorization checks

**Rationale**:
- Files outside wwwroot require programmatic serving
- Controller enables authorization verification before serving
- PDF preview uses browser's native PDF viewer via `Content-Type: application/pdf`
- Image preview uses `<img>` tag with controller endpoint URL

**Controller Pattern**:
```csharp
[Authorize]
[ApiController]
[Route("api/[controller]")]
public class DocumentsController : ControllerBase
{
    [HttpGet("{documentId}/download")]
    public async Task<IActionResult> Download(int documentId)
    {
        // 1. Verify user has access to document
        // 2. Get document metadata from database
        // 3. Stream file from storage service
        // 4. Return FileStreamResult with original filename
    }
    
    [HttpGet("{documentId}/preview")]
    public async Task<IActionResult> Preview(int documentId)
    {
        // Same as download but inline disposition for browser display
    }
}
```

**Alternatives Considered**:
- Blazor JSInterop file serving: Rejected (doesn't support streaming large files)
- Signed URLs: Not applicable for local storage

---

## 4. Database Schema Design

**Question**: How to model documents with relationships to users, projects, and sharing?

**Decision**: Three entities - Document, DocumentShare, DocumentAuditLog

**Rationale**:
- Document entity stores metadata with FK to User (uploader) and optional FK to Project
- DocumentShare models many-to-many sharing relationship
- DocumentAuditLog captures all operations for compliance
- Integer PKs match existing User/Project pattern
- Category stored as string (not enum) per stakeholder requirements

**Key Design Decisions**:
- `DocumentId` as `int` (matches User/Project pattern)
- `Category` as `string` (flexibility, stakeholder requirement)
- `FileType` as `string[255]` (accommodates long MIME types)
- `FilePath` as `string[500]` (GUID-based paths with directory structure)
- Tags stored as comma-separated string (simplicity over junction table)

**Alternatives Considered**:
- GUID primary keys: Rejected (inconsistent with existing schema)
- Enum for Category: Rejected (stakeholder specified string storage)
- Tags junction table: Rejected (adds complexity, simple search sufficient)

---

## 5. Search Implementation

**Question**: How to implement document search with <2s performance?

**Decision**: EF Core LINQ queries with database indexes

**Rationale**:
- SQLite supports LIKE queries for text search
- Indexes on Category, UploadedById, ProjectId enable fast filtering
- Client-side filtering acceptable for MVP scale (~500 docs per user)
- Full-text search can be added later if needed

**Index Strategy**:
```csharp
modelBuilder.Entity<Document>()
    .HasIndex(d => d.UploadedById);
modelBuilder.Entity<Document>()
    .HasIndex(d => d.ProjectId);
modelBuilder.Entity<Document>()
    .HasIndex(d => d.Category);
modelBuilder.Entity<Document>()
    .HasIndex(d => d.UploadDate);
```

**Alternatives Considered**:
- Full-text search: Deferred (SQLite FTS adds complexity, not needed at MVP scale)
- Elasticsearch: Rejected (violates offline-first, over-engineering)

---

## 6. Authorization Model

**Question**: How to enforce document access control?

**Decision**: Service-level authorization with role hierarchy

**Rationale**:
- Users can access: own documents, shared documents, project documents (if member)
- Project Managers can delete any document in their projects
- Administrators have full access for audit purposes
- Authorization checked in DocumentService before any data access (IDOR protection)

**Access Rules**:
| Actor | Own Docs | Shared Docs | Project Docs | All Docs |
|-------|----------|-------------|--------------|----------|
| Employee | CRUD | Read | Read (if member) | ❌ |
| TeamLead | CRUD | Read | Read (if member) | ❌ |
| ProjectManager | CRUD | Read | CRUD (their projects) | ❌ |
| Administrator | CRUD | Read | CRUD | Read/Delete |

**Alternatives Considered**:
- Page-level only authorization: Rejected (doesn't prevent IDOR)
- Policy-based authorization: Considered but service checks simpler for this scope

---

## 7. Blazor Component State Management

**Question**: How to handle upload component state after successful upload?

**Decision**: Use `@key` directive and explicit state reset

**Rationale**:
- `InputFile` component doesn't automatically clear after upload
- `@key` with changing value forces component re-render
- Explicit null assignment of `IBrowserFile` reference prevents reuse errors

**Pattern**:
```razor
<InputFile @key="@_inputFileKey" OnChange="HandleFileSelected" />

@code {
    private Guid _inputFileKey = Guid.NewGuid();
    private IBrowserFile? _selectedFile;
    
    private async Task HandleUpload()
    {
        // ... upload logic ...
        
        _selectedFile = null;
        _inputFileKey = Guid.NewGuid(); // Force re-render
        StateHasChanged();
    }
}
```

---

## 8. Notification Integration

**Question**: How to notify users when documents are shared or added to projects?

**Decision**: Use existing NotificationService pattern

**Rationale**:
- ContosoDashboard already has `INotificationService` with `CreateNotificationAsync`
- Document events create notifications with appropriate type and message
- Consistent with existing notification patterns

**Notification Types to Add**:
- `DocumentShared`: "User X shared 'Document Title' with you"
- `ProjectDocumentAdded`: "New document 'Title' added to Project X"

---

## Resolved Clarifications

All items from Technical Context that marked "NEEDS CLARIFICATION" have been resolved:

| Item | Resolution |
|------|------------|
| File upload component | Blazor `InputFile` with MemoryStream pattern |
| Storage location | `AppData/uploads/` outside wwwroot |
| File serving | ASP.NET Core Controller with auth checks |
| Search performance | EF Core + indexes (sufficient for MVP scale) |
| Preview implementation | Browser-native PDF viewer, img tag for images |
