# API Contract: Document Operations

**Feature**: 002-document-upload  
**Date**: 2026-01-28  
**Type**: Internal service contracts and HTTP endpoints

## Overview

This document defines the contracts for document operations including:
1. `IDocumentService` - Business logic interface
2. `IFileStorageService` - Storage abstraction interface  
3. `DocumentsController` - HTTP API endpoints for file operations

---

## IDocumentService Contract

Business logic interface for document operations with authorization.

```csharp
public interface IDocumentService
{
    // Upload & Create
    Task<DocumentResult> UploadDocumentAsync(
        Stream fileStream,
        string originalFileName,
        string contentType,
        long fileSize,
        string title,
        string category,
        string? description,
        string? tags,
        int? projectId,
        int userId);

    // Read Operations
    Task<IEnumerable<DocumentDto>> GetMyDocumentsAsync(int userId);
    Task<IEnumerable<DocumentDto>> GetSharedWithMeAsync(int userId);
    Task<IEnumerable<DocumentDto>> GetProjectDocumentsAsync(int projectId, int userId);
    Task<DocumentDto?> GetDocumentAsync(int documentId, int userId);
    Task<IEnumerable<DocumentDto>> SearchDocumentsAsync(
        int userId,
        string? searchTerm,
        string? category,
        int? projectId,
        DateTime? fromDate,
        DateTime? toDate);

    // Update Operations
    Task<DocumentResult> UpdateDocumentMetadataAsync(
        int documentId,
        int userId,
        string title,
        string? description,
        string category,
        string? tags);
    Task<DocumentResult> ReplaceDocumentFileAsync(
        int documentId,
        int userId,
        Stream fileStream,
        string originalFileName,
        string contentType,
        long fileSize);

    // Delete Operations
    Task<DocumentResult> DeleteDocumentAsync(int documentId, int userId);

    // Share Operations
    Task<DocumentResult> ShareDocumentAsync(int documentId, int ownerId, int recipientUserId);
    Task<DocumentResult> UnshareDocumentAsync(int documentId, int ownerId, int recipientUserId);
    Task<IEnumerable<UserDto>> GetDocumentSharesAsync(int documentId, int userId);

    // Access Check
    Task<bool> CanUserAccessDocumentAsync(int documentId, int userId);
    Task<bool> CanUserDeleteDocumentAsync(int documentId, int userId);
}
```

### DocumentResult

```csharp
public class DocumentResult
{
    public bool Success { get; set; }
    public int? DocumentId { get; set; }
    public string? ErrorMessage { get; set; }
    public DocumentResultCode Code { get; set; }
}

public enum DocumentResultCode
{
    Success,
    ValidationError,
    Unauthorized,
    NotFound,
    FileSizeExceeded,
    InvalidFileType,
    StorageError
}
```

### DocumentDto

```csharp
public class DocumentDto
{
    public int DocumentId { get; set; }
    public string Title { get; set; } = string.Empty;
    public string? Description { get; set; }
    public string Category { get; set; } = string.Empty;
    public string? Tags { get; set; }
    public string OriginalFileName { get; set; } = string.Empty;
    public long FileSize { get; set; }
    public string FileType { get; set; } = string.Empty;
    public DateTime UploadDate { get; set; }
    public string UploaderName { get; set; } = string.Empty;
    public int UploadedById { get; set; }
    public int? ProjectId { get; set; }
    public string? ProjectName { get; set; }
    public bool IsOwner { get; set; }
    public bool CanDelete { get; set; }
    public bool CanEdit { get; set; }
    public bool CanPreview { get; set; }
}
```

---

## IFileStorageService Contract

Storage abstraction for file operations. Enables swapping between local and cloud storage.

```csharp
public interface IFileStorageService
{
    /// <summary>
    /// Upload a file to storage.
    /// </summary>
    /// <param name="fileStream">File content stream</param>
    /// <param name="storagePath">Relative storage path (e.g., "1/personal/abc123.pdf")</param>
    /// <param name="contentType">MIME type of the file</param>
    /// <returns>The storage path used</returns>
    Task<string> UploadAsync(Stream fileStream, string storagePath, string contentType);

    /// <summary>
    /// Delete a file from storage.
    /// </summary>
    /// <param name="storagePath">Relative storage path</param>
    Task DeleteAsync(string storagePath);

    /// <summary>
    /// Download a file from storage.
    /// </summary>
    /// <param name="storagePath">Relative storage path</param>
    /// <returns>File content stream</returns>
    Task<Stream> DownloadAsync(string storagePath);

    /// <summary>
    /// Check if a file exists in storage.
    /// </summary>
    /// <param name="storagePath">Relative storage path</param>
    /// <returns>True if file exists</returns>
    Task<bool> ExistsAsync(string storagePath);

    /// <summary>
    /// Generate a unique storage path for a new file.
    /// </summary>
    /// <param name="userId">Owner user ID</param>
    /// <param name="projectId">Optional project ID</param>
    /// <param name="fileExtension">File extension (e.g., ".pdf")</param>
    /// <returns>Unique storage path</returns>
    string GenerateStoragePath(int userId, int? projectId, string fileExtension);
}
```

### Storage Path Convention

```
{userId}/{projectId|personal}/{guid}.{extension}

Examples:
- 1/personal/a1b2c3d4-e5f6-7890-abcd-ef1234567890.pdf
- 1/5/b2c3d4e5-f6a7-8901-bcde-f12345678901.docx
```

---

## DocumentsController HTTP Endpoints

REST API endpoints for file download and preview operations.

### GET /api/documents/{documentId}/download

Download a document file.

**Authorization**: User must have access to document (owner, shared, or project member)

**Response**:
- `200 OK`: File stream with `Content-Disposition: attachment`
- `401 Unauthorized`: User not authenticated
- `403 Forbidden`: User doesn't have access
- `404 Not Found`: Document doesn't exist

**Headers**:
```
Content-Type: {document MIME type}
Content-Disposition: attachment; filename="{original filename}"
Content-Length: {file size}
```

### GET /api/documents/{documentId}/preview

Preview a document in browser (PDF/images only).

**Authorization**: User must have access to document

**Response**:
- `200 OK`: File stream with `Content-Disposition: inline`
- `400 Bad Request`: File type doesn't support preview
- `401 Unauthorized`: User not authenticated
- `403 Forbidden`: User doesn't have access
- `404 Not Found`: Document doesn't exist

**Headers**:
```
Content-Type: {document MIME type}
Content-Disposition: inline; filename="{original filename}"
Content-Length: {file size}
```

**Previewable Types**:
- `application/pdf`
- `image/jpeg`
- `image/png`

---

## Notification Events

Document operations trigger notifications via existing `INotificationService`.

### Document Shared

```csharp
await _notificationService.CreateNotificationAsync(new Notification
{
    UserId = recipientUserId,
    Title = "Document Shared",
    Message = $"{sharerName} shared '{documentTitle}' with you",
    Type = NotificationType.DocumentShared,
    Priority = NotificationPriority.Normal,
    CreatedDate = DateTime.UtcNow,
    IsRead = false
});
```

### Project Document Added

```csharp
// For each project member (except uploader)
await _notificationService.CreateNotificationAsync(new Notification
{
    UserId = memberId,
    Title = "New Project Document",
    Message = $"'{documentTitle}' added to {projectName}",
    Type = NotificationType.ProjectDocumentAdded,
    Priority = NotificationPriority.Normal,
    CreatedDate = DateTime.UtcNow,
    IsRead = false
});
```

---

## Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `ValidationError` | 400 | Invalid input (missing title, invalid category) |
| `FileSizeExceeded` | 400 | File exceeds 25 MB limit |
| `InvalidFileType` | 400 | File type not in allowed list |
| `Unauthorized` | 403 | User doesn't have permission |
| `NotFound` | 404 | Document doesn't exist |
| `StorageError` | 500 | File system operation failed |
