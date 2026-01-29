# Quickstart: Document Upload and Management

**Feature**: 002-document-upload  
**Date**: 2026-01-28  
**Prerequisites**: ContosoDashboard running locally

## Quick Setup

### 1. Create Storage Directory

```bash
# From repository root
mkdir -p ContosoDashboard/AppData/uploads
```

### 2. Register Services (Program.cs)

```csharp
// Add after existing service registrations
builder.Services.AddScoped<IFileStorageService, LocalFileStorageService>();
builder.Services.AddScoped<IDocumentService, DocumentService>();

// Add controllers for file serving
builder.Services.AddControllers();
```

### 3. Map Controller Routes (Program.cs)

```csharp
// Add before app.MapBlazorHub()
app.MapControllers();
```

### 4. Run the Application

```bash
cd ContosoDashboard
dotnet run
```

### 5. Test Document Upload

1. Navigate to `https://localhost:{port}/login`
2. Login as any user (e.g., "Camille Nicole")
3. Navigate to Documents page
4. Click "Upload Document"
5. Select a PDF file (under 25 MB)
6. Enter title "Test Document" and select category "Reports"
7. Click Upload
8. Verify document appears in list

---

## Key Files to Implement

| File | Purpose |
|------|---------|
| `Models/Document.cs` | Document entity |
| `Models/DocumentShare.cs` | Sharing relationship entity |
| `Models/DocumentAuditLog.cs` | Audit log entity |
| `Services/IFileStorageService.cs` | Storage interface |
| `Services/LocalFileStorageService.cs` | Local filesystem implementation |
| `Services/IDocumentService.cs` | Document service interface |
| `Services/DocumentService.cs` | Document business logic |
| `Controllers/DocumentsController.cs` | Download/preview API |
| `Pages/Documents.razor` | Documents list page |

---

## Common Issues

### "File not found" when downloading

- Verify `AppData/uploads/` directory exists
- Check file path in database matches actual file location
- Ensure `LocalFileStorageService` uses correct base path

### Upload fails with "Stream disposed" error

- Use MemoryStream pattern to copy file data immediately
- Don't reuse `IBrowserFile` reference after reading

### Authorization denied for project documents

- Verify user is a member of the project
- Check `ProjectMembers` table includes user

---

## Testing Checklist

- [ ] Upload PDF file (success)
- [ ] Upload file > 25 MB (rejection message)
- [ ] Upload .exe file (rejection message)
- [ ] Download document (file saves with original name)
- [ ] Preview PDF (displays in browser)
- [ ] Preview image (displays in modal)
- [ ] Search documents (results within 2s)
- [ ] Filter by category
- [ ] Sort by upload date
- [ ] Edit document metadata
- [ ] Delete document (with confirmation)
- [ ] Share document (recipient sees in "Shared with Me")
- [ ] Project document visibility (all members see it)
- [ ] IDOR test: Try accessing another user's document by URL
