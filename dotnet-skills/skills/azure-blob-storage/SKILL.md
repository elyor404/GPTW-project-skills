---
name: azure-blob-storage
description: Azure Blob Storage patterns for file uploads, downloads, SAS tokens, and asset management. Use when handling file storage for images, documents, and media.
---

# Azure Blob Storage Patterns

## When to Use This Skill

Use this skill when:
- Uploading files (images, documents, videos)
- Generating secure download URLs
- Managing asset libraries
- Handling brand logos and certification badges
- Storing AI-generated images

## Client Configuration

### Blob Service Setup

```csharp
// appsettings.json
{
  "AzureBlobStorage": {
    "ConnectionString": "", // Use Key Vault in production
    "AccountName": "gptwplusstorage",
    "ContainerNames": {
      "Assets": "assets",
      "BrandLogos": "brand-logos",
      "Certifications": "certifications",
      "GeneratedImages": "generated-images"
    }
  }
}

// Configuration class
public class BlobStorageOptions
{
    public const string SectionName = "AzureBlobStorage";
    public string AccountName { get; init; } = string.Empty;
    public ContainerNamesOptions ContainerNames { get; init; } = new();
}

public class ContainerNamesOptions
{
    public string Assets { get; init; } = "assets";
    public string BrandLogos { get; init; } = "brand-logos";
    public string Certifications { get; init; } = "certifications";
    public string GeneratedImages { get; init; } = "generated-images";
}

// Registration with Managed Identity (recommended)
builder.Services.AddSingleton(sp =>
{
    var options = sp.GetRequiredService<IOptions<BlobStorageOptions>>().Value;
    var serviceUri = new Uri($"https://{options.AccountName}.blob.core.windows.net");
    return new BlobServiceClient(serviceUri, new DefaultAzureCredential());
});

builder.Services.Configure<BlobStorageOptions>(
    builder.Configuration.GetSection(BlobStorageOptions.SectionName));
```

## Blob Storage Service

### Interface and Implementation

```csharp
public interface IBlobStorageService
{
    Task<BlobUploadResult> UploadAsync(
        string containerName,
        string blobName,
        Stream content,
        string contentType,
        IDictionary<string, string>? metadata = null,
        CancellationToken cancellationToken = default);

    Task<Stream?> DownloadAsync(
        string containerName,
        string blobName,
        CancellationToken cancellationToken = default);

    Task<bool> DeleteAsync(
        string containerName,
        string blobName,
        CancellationToken cancellationToken = default);

    Task<string> GetSasUrlAsync(
        string containerName,
        string blobName,
        TimeSpan expiresIn,
        BlobSasPermissions permissions = BlobSasPermissions.Read);

    Task<IReadOnlyList<BlobInfo>> ListBlobsAsync(
        string containerName,
        string? prefix = null,
        CancellationToken cancellationToken = default);
}

public class BlobStorageService : IBlobStorageService
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly ILogger<BlobStorageService> _logger;

    public BlobStorageService(
        BlobServiceClient blobServiceClient,
        ILogger<BlobStorageService> logger)
    {
        _blobServiceClient = blobServiceClient;
        _logger = logger;
    }

    public async Task<BlobUploadResult> UploadAsync(
        string containerName,
        string blobName,
        Stream content,
        string contentType,
        IDictionary<string, string>? metadata = null,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
            await containerClient.CreateIfNotExistsAsync(cancellationToken: cancellationToken);

            var blobClient = containerClient.GetBlobClient(blobName);

            var uploadOptions = new BlobUploadOptions
            {
                HttpHeaders = new BlobHttpHeaders
                {
                    ContentType = contentType,
                    CacheControl = "public, max-age=31536000" // 1 year cache
                },
                Metadata = metadata
            };

            await blobClient.UploadAsync(content, uploadOptions, cancellationToken);

            _logger.LogInformation(
                "Uploaded blob {BlobName} to container {Container}",
                blobName, containerName);

            return new BlobUploadResult
            {
                Success = true,
                BlobName = blobName,
                Uri = blobClient.Uri.ToString(),
                ContentType = contentType
            };
        }
        catch (RequestFailedException ex)
        {
            _logger.LogError(ex, "Failed to upload blob {BlobName}", blobName);
            return new BlobUploadResult
            {
                Success = false,
                Error = ex.Message
            };
        }
    }

    public async Task<Stream?> DownloadAsync(
        string containerName,
        string blobName,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
            var blobClient = containerClient.GetBlobClient(blobName);

            if (!await blobClient.ExistsAsync(cancellationToken))
                return null;

            var response = await blobClient.DownloadStreamingAsync(cancellationToken: cancellationToken);
            return response.Value.Content;
        }
        catch (RequestFailedException ex)
        {
            _logger.LogError(ex, "Failed to download blob {BlobName}", blobName);
            return null;
        }
    }

    public async Task<bool> DeleteAsync(
        string containerName,
        string blobName,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
            var blobClient = containerClient.GetBlobClient(blobName);

            var response = await blobClient.DeleteIfExistsAsync(cancellationToken: cancellationToken);
            return response.Value;
        }
        catch (RequestFailedException ex)
        {
            _logger.LogError(ex, "Failed to delete blob {BlobName}", blobName);
            return false;
        }
    }

    public Task<string> GetSasUrlAsync(
        string containerName,
        string blobName,
        TimeSpan expiresIn,
        BlobSasPermissions permissions = BlobSasPermissions.Read)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        var blobClient = containerClient.GetBlobClient(blobName);

        var sasBuilder = new BlobSasBuilder
        {
            BlobContainerName = containerName,
            BlobName = blobName,
            Resource = "b",
            StartsOn = DateTimeOffset.UtcNow.AddMinutes(-5), // Clock skew tolerance
            ExpiresOn = DateTimeOffset.UtcNow.Add(expiresIn)
        };

        sasBuilder.SetPermissions(permissions);

        var sasUri = blobClient.GenerateSasUri(sasBuilder);
        return Task.FromResult(sasUri.ToString());
    }

    public async Task<IReadOnlyList<BlobInfo>> ListBlobsAsync(
        string containerName,
        string? prefix = null,
        CancellationToken cancellationToken = default)
    {
        var containerClient = _blobServiceClient.GetBlobContainerClient(containerName);
        var blobs = new List<BlobInfo>();

        await foreach (var blobItem in containerClient.GetBlobsAsync(
            prefix: prefix,
            cancellationToken: cancellationToken))
        {
            blobs.Add(new BlobInfo
            {
                Name = blobItem.Name,
                ContentType = blobItem.Properties.ContentType,
                Size = blobItem.Properties.ContentLength ?? 0,
                CreatedOn = blobItem.Properties.CreatedOn,
                Metadata = blobItem.Metadata
            });
        }

        return blobs;
    }
}
```

## Asset Management Service

### Company Asset Service

```csharp
public interface IAssetService
{
    Task<AssetUploadResult> UploadAssetAsync(
        Guid companyId,
        IFormFile file,
        string description,
        CancellationToken cancellationToken = default);

    Task<string> GetAssetUrlAsync(
        Guid companyId,
        Guid assetId,
        CancellationToken cancellationToken = default);

    Task<IReadOnlyList<AssetDto>> GetCompanyAssetsAsync(
        Guid companyId,
        CancellationToken cancellationToken = default);

    Task<bool> DeleteAssetAsync(
        Guid companyId,
        Guid assetId,
        CancellationToken cancellationToken = default);
}

public class AssetService : IAssetService
{
    private readonly IBlobStorageService _blobStorage;
    private readonly ApplicationDbContext _context;
    private readonly BlobStorageOptions _options;
    private readonly ILogger<AssetService> _logger;

    private static readonly HashSet<string> AllowedContentTypes = new(StringComparer.OrdinalIgnoreCase)
    {
        "image/jpeg", "image/png", "image/gif", "image/webp",
        "video/mp4", "video/webm"
    };

    private const long MaxFileSizeBytes = 50 * 1024 * 1024; // 50MB

    public async Task<AssetUploadResult> UploadAssetAsync(
        Guid companyId,
        IFormFile file,
        string description,
        CancellationToken cancellationToken = default)
    {
        // Validate file
        if (file.Length == 0)
            return AssetUploadResult.Failed("File is empty");

        if (file.Length > MaxFileSizeBytes)
            return AssetUploadResult.Failed($"File exceeds maximum size of {MaxFileSizeBytes / 1024 / 1024}MB");

        if (!AllowedContentTypes.Contains(file.ContentType))
            return AssetUploadResult.Failed($"Content type '{file.ContentType}' is not allowed");

        // Generate unique blob name with company isolation
        var assetId = Guid.NewGuid();
        var extension = Path.GetExtension(file.FileName);
        var blobName = $"{companyId}/{assetId}{extension}";

        // Upload to blob storage
        await using var stream = file.OpenReadStream();
        var uploadResult = await _blobStorage.UploadAsync(
            _options.ContainerNames.Assets,
            blobName,
            stream,
            file.ContentType,
            new Dictionary<string, string>
            {
                ["companyId"] = companyId.ToString(),
                ["originalFileName"] = file.FileName
            },
            cancellationToken);

        if (!uploadResult.Success)
            return AssetUploadResult.Failed(uploadResult.Error!);

        // Save metadata to database
        var asset = new Asset
        {
            Id = assetId,
            CompanyId = companyId,
            BlobName = blobName,
            OriginalFileName = file.FileName,
            ContentType = file.ContentType,
            Size = file.Length,
            Description = description
        };

        _context.Assets.Add(asset);
        await _context.SaveChangesAsync(cancellationToken);

        return AssetUploadResult.Succeeded(new AssetDto
        {
            Id = assetId,
            FileName = file.FileName,
            ContentType = file.ContentType,
            Size = file.Length,
            Description = description,
            Url = uploadResult.Uri
        });
    }

    public async Task<string> GetAssetUrlAsync(
        Guid companyId,
        Guid assetId,
        CancellationToken cancellationToken = default)
    {
        var asset = await _context.Assets
            .FirstOrDefaultAsync(a => a.Id == assetId && a.CompanyId == companyId, cancellationToken);

        if (asset is null)
            throw new NotFoundException("Asset", assetId);

        // Generate SAS URL valid for 1 hour
        return await _blobStorage.GetSasUrlAsync(
            _options.ContainerNames.Assets,
            asset.BlobName,
            TimeSpan.FromHours(1));
    }
}
```

## Brand Logo Upload

### Brand Kit Service

```csharp
public class BrandKitService
{
    private readonly IBlobStorageService _blobStorage;
    private readonly BlobStorageOptions _options;

    public async Task<string> UploadLogoAsync(
        Guid companyId,
        IFormFile file,
        LogoType logoType,
        CancellationToken cancellationToken = default)
    {
        // Validate image
        if (!file.ContentType.StartsWith("image/"))
            throw new ValidationException("Logo must be an image file");

        // Use predictable naming for brand logos
        var extension = Path.GetExtension(file.FileName);
        var blobName = $"{companyId}/{logoType.ToString().ToLowerInvariant()}{extension}";

        await using var stream = file.OpenReadStream();
        var result = await _blobStorage.UploadAsync(
            _options.ContainerNames.BrandLogos,
            blobName,
            stream,
            file.ContentType,
            cancellationToken: cancellationToken);

        return result.Uri;
    }

    public async Task<string> GetLogoUrlAsync(
        Guid companyId,
        LogoType logoType,
        CancellationToken cancellationToken = default)
    {
        // Find the logo (extension might vary)
        var blobs = await _blobStorage.ListBlobsAsync(
            _options.ContainerNames.BrandLogos,
            $"{companyId}/{logoType.ToString().ToLowerInvariant()}",
            cancellationToken);

        var logo = blobs.FirstOrDefault();
        if (logo is null)
            return string.Empty;

        return await _blobStorage.GetSasUrlAsync(
            _options.ContainerNames.BrandLogos,
            logo.Name,
            TimeSpan.FromHours(24));
    }
}

public enum LogoType
{
    Primary,
    Secondary,
    Icon,
    Monochrome
}
```

## Controller Integration

### File Upload Endpoint

```csharp
[ApiController]
[Route("api/companies/{companyId}/assets")]
[Authorize]
public class AssetsController : ControllerBase
{
    private readonly IAssetService _assetService;

    [HttpPost]
    [RequestSizeLimit(50 * 1024 * 1024)] // 50MB
    [ProducesResponseType(typeof(AssetDto), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Upload(
        Guid companyId,
        [FromForm] IFormFile file,
        [FromForm] string description,
        CancellationToken cancellationToken)
    {
        var result = await _assetService.UploadAssetAsync(
            companyId, file, description, cancellationToken);

        if (!result.Success)
            return BadRequest(result.Error);

        return CreatedAtAction(
            nameof(GetById),
            new { companyId, id = result.Asset!.Id },
            result.Asset);
    }

    [HttpGet("{id}")]
    [ProducesResponseType(StatusCodes.Status302Found)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> GetById(
        Guid companyId,
        Guid id,
        CancellationToken cancellationToken)
    {
        try
        {
            var url = await _assetService.GetAssetUrlAsync(companyId, id, cancellationToken);
            return Redirect(url);
        }
        catch (NotFoundException)
        {
            return NotFound();
        }
    }

    [HttpDelete("{id}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(
        Guid companyId,
        Guid id,
        CancellationToken cancellationToken)
    {
        var success = await _assetService.DeleteAssetAsync(companyId, id, cancellationToken);
        return success ? NoContent() : NotFound();
    }
}
```

## Result Types

```csharp
public record BlobUploadResult
{
    public bool Success { get; init; }
    public string? BlobName { get; init; }
    public string? Uri { get; init; }
    public string? ContentType { get; init; }
    public string? Error { get; init; }
}

public record BlobInfo
{
    public string Name { get; init; } = string.Empty;
    public string? ContentType { get; init; }
    public long Size { get; init; }
    public DateTimeOffset? CreatedOn { get; init; }
    public IDictionary<string, string>? Metadata { get; init; }
}

public record AssetUploadResult
{
    public bool Success { get; init; }
    public AssetDto? Asset { get; init; }
    public string? Error { get; init; }

    public static AssetUploadResult Succeeded(AssetDto asset) =>
        new() { Success = true, Asset = asset };

    public static AssetUploadResult Failed(string error) =>
        new() { Success = false, Error = error };
}
```

## Anti-Patterns to Avoid

### DON'T: Store files without tenant isolation

```csharp
// BAD - no tenant isolation
var blobName = $"{file.FileName}";

// GOOD - include company ID in path
var blobName = $"{companyId}/{assetId}{extension}";
```

### DON'T: Return blob URLs without SAS tokens for private containers

```csharp
// BAD - won't work for private containers
return blobClient.Uri.ToString();

// GOOD - generate SAS URL
return await GetSasUrlAsync(container, blobName, TimeSpan.FromHours(1));
```

## References

- [Azure Blob Storage Documentation](https://learn.microsoft.com/en-us/azure/storage/blobs/)
- [Azure SDK for .NET](https://learn.microsoft.com/en-us/dotnet/azure/sdk/azure-sdk-for-dotnet)
- [SAS Tokens](https://learn.microsoft.com/en-us/azure/storage/common/storage-sas-overview)
