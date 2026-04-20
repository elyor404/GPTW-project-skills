---
name: azure-functions
description: Azure Functions patterns for background jobs, event-driven processing, scheduled tasks. Use when implementing Emprising data ingestion, email triggers, and scheduled reminders.
---

# Azure Functions Patterns

## When to Use This Skill

Use this skill when:
- Processing Emprising data imports from OneDrive
- Handling scheduled background jobs
- Implementing event-driven workflows
- Processing queued tasks asynchronously

## Function Types

### Blob Trigger - Emprising Data Ingestion

```csharp
public class EmprisingIngestionFunction
{
    private readonly IEmprisingDataProcessor _processor;
    private readonly ILogger<EmprisingIngestionFunction> _logger;

    public EmprisingIngestionFunction(
        IEmprisingDataProcessor processor,
        ILogger<EmprisingIngestionFunction> logger)
    {
        _processor = processor;
        _logger = logger;
    }

    [Function("ProcessEmprisingFile")]
    public async Task Run(
        [BlobTrigger("emprising-imports/{name}", Connection = "StorageConnection")] Stream blobStream,
        string name,
        FunctionContext context)
    {
        _logger.LogInformation("Processing Emprising file: {FileName}", name);

        try
        {
            var result = await _processor.ProcessAsync(blobStream, name, context.CancellationToken);

            if (result.IsSuccess)
            {
                _logger.LogInformation(
                    "Successfully processed {FileName}: {RecordCount} records",
                    name, result.RecordCount);
            }
            else
            {
                _logger.LogError(
                    "Failed to process {FileName}: {Errors}",
                    name, string.Join(", ", result.Errors));
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing Emprising file: {FileName}", name);
            throw;
        }
    }
}
```

### Timer Trigger - Scheduled Tasks

```csharp
public class ScheduledReminderFunction
{
    private readonly INavigatorService _navigatorService;
    private readonly IEmailService _emailService;
    private readonly ILogger<ScheduledReminderFunction> _logger;

    [Function("MonthlyActivityCheck")]
    public async Task Run(
        [TimerTrigger("0 0 9 1 * *")] TimerInfo timerInfo, // 9 AM on 1st of each month
        FunctionContext context)
    {
        _logger.LogInformation("Running monthly activity check at {Time}", DateTime.UtcNow);

        var inactiveCompanies = await _navigatorService.GetInactiveCompaniesAsync(
            TimeSpan.FromDays(30),
            context.CancellationToken);

        foreach (var company in inactiveCompanies)
        {
            await _emailService.SendActivityReminderAsync(
                company.AdminEmails,
                company.Name,
                context.CancellationToken);
        }

        _logger.LogInformation("Sent reminders to {Count} inactive companies", inactiveCompanies.Count);
    }
}
```

### Queue Trigger - Async Processing

```csharp
public class InsightGenerationFunction
{
    private readonly IAiOrchestrationService _aiService;
    private readonly IElevateInsightRepository _repository;

    [Function("GenerateElevateInsight")]
    public async Task Run(
        [QueueTrigger("insight-generation", Connection = "StorageConnection")] InsightGenerationMessage message,
        FunctionContext context)
    {
        var result = await _aiService.GenerateInsightAsync(
            message.CompanyId,
            message.PhaseId,
            message.SurveyData,
            context.CancellationToken);

        if (result.IsSuccess)
        {
            await _repository.SaveInsightAsync(new ElevateInsight
            {
                CompanyId = message.CompanyId,
                PhaseId = message.PhaseId,
                Content = result.Value,
                Label = AiOutputLabel.AiGenerated,
                Status = InsightStatus.PendingReview
            }, context.CancellationToken);
        }
    }
}

public record InsightGenerationMessage(
    Guid CompanyId,
    int PhaseId,
    string SurveyData);
```

## Dependency Injection Setup

### Program.cs Configuration

```csharp
var host = new HostBuilder()
    .ConfigureFunctionsWorkerDefaults()
    .ConfigureServices((context, services) =>
    {
        // Configuration
        services.AddOptions<AzureOpenAIOptions>()
            .Bind(context.Configuration.GetSection("AzureOpenAI"));

        // Database
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseNpgsql(context.Configuration.GetConnectionString("DefaultConnection")));

        // Services
        services.AddScoped<IEmprisingDataProcessor, EmprisingDataProcessor>();
        services.AddScoped<IAiOrchestrationService, AiOrchestrationService>();
        services.AddScoped<IEmailService, EmailService>();
        services.AddScoped<INavigatorService, NavigatorService>();

        // Repositories
        services.AddScoped<IElevateInsightRepository, ElevateInsightRepository>();

        // Azure clients
        services.AddSingleton(sp =>
        {
            var options = sp.GetRequiredService<IOptions<AzureOpenAIOptions>>().Value;
            return new OpenAIClient(new Uri(options.Endpoint), new DefaultAzureCredential());
        });
    })
    .Build();

host.Run();
```

## Emprising Data Processor

```csharp
public interface IEmprisingDataProcessor
{
    Task<ProcessingResult> ProcessAsync(Stream fileStream, string fileName, CancellationToken ct);
}

public class EmprisingDataProcessor : IEmprisingDataProcessor
{
    private readonly ApplicationDbContext _context;
    private readonly ILogger<EmprisingDataProcessor> _logger;

    public async Task<ProcessingResult> ProcessAsync(
        Stream fileStream,
        string fileName,
        CancellationToken ct)
    {
        var errors = new List<string>();
        var recordCount = 0;

        // Extract client ID from filename (e.g., "UK-1234_TrustIndex_2024.xlsx")
        var clientId = ExtractClientId(fileName);
        if (clientId is null)
        {
            return ProcessingResult.Failed("Could not extract client ID from filename");
        }

        var company = await _context.Companies
            .FirstOrDefaultAsync(c => c.ClientId == clientId, ct);

        if (company is null)
        {
            return ProcessingResult.Failed($"Company not found for client ID: {clientId}");
        }

        using var workbook = new XLWorkbook(fileStream);

        // Validate required sheets
        var requiredSheets = new[] { "Trust Index", "Demographics", "Benchmarks" };
        foreach (var sheetName in requiredSheets)
        {
            if (!workbook.Worksheets.Contains(sheetName))
            {
                errors.Add($"Missing required sheet: {sheetName}");
            }
        }

        if (errors.Any())
        {
            return ProcessingResult.Failed(errors);
        }

        // Process Trust Index data
        var trustIndexSheet = workbook.Worksheet("Trust Index");
        var statements = ProcessTrustIndexSheet(trustIndexSheet, company.Id);
        recordCount += statements.Count;

        // Create import record
        var import = new EmprisingImport
        {
            Id = Guid.NewGuid(),
            CompanyId = company.Id,
            FileName = fileName,
            ImportDate = DateTimeOffset.UtcNow,
            RecordCount = recordCount
        };

        _context.EmprisingImports.Add(import);
        _context.TrustIndexStatements.AddRange(statements);
        await _context.SaveChangesAsync(ct);

        return ProcessingResult.Succeeded(recordCount);
    }

    private List<TrustIndexStatement> ProcessTrustIndexSheet(
        IXLWorksheet sheet,
        Guid companyId)
    {
        var statements = new List<TrustIndexStatement>();
        var rows = sheet.RowsUsed().Skip(1); // Skip header

        foreach (var row in rows)
        {
            statements.Add(new TrustIndexStatement
            {
                Id = Guid.NewGuid(),
                CompanyId = companyId,
                StatementId = row.Cell(1).GetValue<int>(),
                StatementText = ConvertToUkEnglish(row.Cell(2).GetString()),
                Dimension = row.Cell(3).GetString(),
                CompanyScore = row.Cell(4).GetValue<decimal>(),
                BenchmarkScore = row.Cell(5).GetValue<decimal>()
            });
        }

        return statements;
    }

    private string ConvertToUkEnglish(string text)
    {
        // US to UK spelling conversion
        var conversions = new Dictionary<string, string>
        {
            { "organization", "organisation" },
            { "Organization", "Organisation" },
            { "behavior", "behaviour" },
            { "favor", "favour" },
            { "color", "colour" },
            { "recognize", "recognise" }
        };

        foreach (var (us, uk) in conversions)
        {
            text = text.Replace(us, uk);
        }

        return text;
    }

    private string? ExtractClientId(string fileName)
    {
        var match = Regex.Match(fileName, @"^([A-Z]{2,4}-\d{4,8})_");
        return match.Success ? match.Groups[1].Value : null;
    }
}

public record ProcessingResult
{
    public bool IsSuccess { get; init; }
    public int RecordCount { get; init; }
    public IReadOnlyList<string> Errors { get; init; } = Array.Empty<string>();

    public static ProcessingResult Succeeded(int recordCount) =>
        new() { IsSuccess = true, RecordCount = recordCount };

    public static ProcessingResult Failed(string error) =>
        new() { IsSuccess = false, Errors = new[] { error } };

    public static ProcessingResult Failed(IEnumerable<string> errors) =>
        new() { IsSuccess = false, Errors = errors.ToList() };
}
```

## References

- [Azure Functions .NET Guide](https://learn.microsoft.com/en-us/azure/azure-functions/dotnet-isolated-process-guide)
- [Blob Trigger](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-storage-blob-trigger)
- [Timer Trigger](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-timer)
