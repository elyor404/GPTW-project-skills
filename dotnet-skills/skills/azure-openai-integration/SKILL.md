---
name: azure-openai-integration
description: Azure OpenAI integration patterns for AI content generation - client setup, prompt management, token tracking, cost control, guardrails. Use when implementing AI features.
---

# Azure OpenAI Integration Patterns

## When to Use This Skill

Use this skill when:
- Integrating Azure OpenAI for content generation
- Managing prompts and system messages
- Implementing token tracking and cost control
- Adding AI safety guardrails
- Building AI-powered features (Activate recommendations, Elevate insights)

## Client Setup

### Azure OpenAI Client Configuration

```csharp
// appsettings.json
{
  "AzureOpenAI": {
    "Endpoint": "https://your-resource.openai.azure.com/",
    "DeploymentName": "gpt-4",
    "EmbeddingDeploymentName": "text-embedding-3-large",
    "ApiVersion": "2024-02-15-preview"
  }
}

// Configuration class
public class AzureOpenAIOptions
{
    public const string SectionName = "AzureOpenAI";
    public required string Endpoint { get; init; }
    public required string DeploymentName { get; init; }
    public required string EmbeddingDeploymentName { get; init; }
    public string ApiVersion { get; init; } = "2024-02-15-preview";
}

// Registration with Managed Identity (recommended)
builder.Services.AddSingleton(sp =>
{
    var options = sp.GetRequiredService<IOptions<AzureOpenAIOptions>>().Value;
    return new OpenAIClient(
        new Uri(options.Endpoint),
        new DefaultAzureCredential());
});

builder.Services.Configure<AzureOpenAIOptions>(
    builder.Configuration.GetSection(AzureOpenAIOptions.SectionName));
```

## AI Orchestration Service

### Central AI Service Pattern

```csharp
public interface IAiOrchestrationService
{
    Task<AiGenerationResult<string>> GenerateTextAsync(
        TextGenerationRequest request,
        CancellationToken cancellationToken = default);

    Task<AiGenerationResult<T>> GenerateStructuredAsync<T>(
        StructuredGenerationRequest request,
        CancellationToken cancellationToken = default) where T : class;

    Task<float[]> GenerateEmbeddingAsync(
        string text,
        CancellationToken cancellationToken = default);
}

public class AiOrchestrationService : IAiOrchestrationService
{
    private readonly OpenAIClient _client;
    private readonly AzureOpenAIOptions _options;
    private readonly IAiUsageTracker _usageTracker;
    private readonly IAiGuardrails _guardrails;
    private readonly ILogger<AiOrchestrationService> _logger;

    public AiOrchestrationService(
        OpenAIClient client,
        IOptions<AzureOpenAIOptions> options,
        IAiUsageTracker usageTracker,
        IAiGuardrails guardrails,
        ILogger<AiOrchestrationService> logger)
    {
        _client = client;
        _options = options.Value;
        _usageTracker = usageTracker;
        _guardrails = guardrails;
        _logger = logger;
    }

    public async Task<AiGenerationResult<string>> GenerateTextAsync(
        TextGenerationRequest request,
        CancellationToken cancellationToken = default)
    {
        // Check quota before calling
        var quotaCheck = await _usageTracker.CheckQuotaAsync(request.CompanyId, cancellationToken);
        if (!quotaCheck.HasQuota)
        {
            return AiGenerationResult<string>.QuotaExceeded(quotaCheck.Message);
        }

        // Apply input guardrails
        var sanitizedPrompt = await _guardrails.SanitizeInputAsync(request.UserPrompt);

        var chatOptions = new ChatCompletionsOptions
        {
            DeploymentName = _options.DeploymentName,
            Temperature = request.Temperature ?? 0.7f,
            MaxTokens = request.MaxTokens ?? 1000,
            Messages =
            {
                new ChatRequestSystemMessage(request.SystemPrompt),
                new ChatRequestUserMessage(sanitizedPrompt)
            }
        };

        try
        {
            var response = await _client.GetChatCompletionsAsync(chatOptions, cancellationToken);
            var completion = response.Value;

            // Track usage
            await _usageTracker.TrackUsageAsync(new AiUsageRecord
            {
                CompanyId = request.CompanyId,
                UserId = request.UserId,
                Model = _options.DeploymentName,
                PromptType = request.PromptType,
                PromptTokens = completion.Usage.PromptTokens,
                CompletionTokens = completion.Usage.CompletionTokens,
                TotalTokens = completion.Usage.TotalTokens,
                EstimatedCost = CalculateCost(completion.Usage),
                Timestamp = DateTimeOffset.UtcNow
            }, cancellationToken);

            var content = completion.Choices[0].Message.Content;

            // Apply output guardrails
            var sanitizedOutput = await _guardrails.SanitizeOutputAsync(content);

            return AiGenerationResult<string>.Success(sanitizedOutput, new AiMetadata
            {
                TokensUsed = completion.Usage.TotalTokens,
                Model = _options.DeploymentName,
                GenerationType = AiOutputLabel.AiGenerated
            });
        }
        catch (RequestFailedException ex) when (ex.Status == 429)
        {
            _logger.LogWarning("Azure OpenAI rate limit hit for company {CompanyId}", request.CompanyId);
            return AiGenerationResult<string>.RateLimited("Rate limit exceeded. Please try again later.");
        }
    }

    public async Task<AiGenerationResult<T>> GenerateStructuredAsync<T>(
        StructuredGenerationRequest request,
        CancellationToken cancellationToken = default) where T : class
    {
        var textResult = await GenerateTextAsync(new TextGenerationRequest
        {
            CompanyId = request.CompanyId,
            UserId = request.UserId,
            SystemPrompt = request.SystemPrompt + "\n\nRespond with valid JSON only.",
            UserPrompt = request.UserPrompt,
            PromptType = request.PromptType,
            Temperature = 0.3f // Lower for structured output
        }, cancellationToken);

        if (!textResult.IsSuccess)
        {
            return AiGenerationResult<T>.FromError(textResult.Error!);
        }

        try
        {
            var parsed = JsonSerializer.Deserialize<T>(textResult.Value!);
            return AiGenerationResult<T>.Success(parsed!, textResult.Metadata);
        }
        catch (JsonException ex)
        {
            _logger.LogWarning(ex, "Failed to parse AI response as {Type}", typeof(T).Name);
            return AiGenerationResult<T>.ParseError("AI response was not valid JSON");
        }
    }

    private decimal CalculateCost(CompletionsUsage usage)
    {
        // GPT-4 pricing (adjust based on actual pricing)
        const decimal promptCostPer1K = 0.03m;
        const decimal completionCostPer1K = 0.06m;

        return (usage.PromptTokens / 1000m * promptCostPer1K) +
               (usage.CompletionTokens / 1000m * completionCostPer1K);
    }
}
```

## Prompt Management

### Prompt Templates

```csharp
public static class PromptTemplates
{
    public static class Activate
    {
        public const string SocialPostRecommendations = """
            You are an employer branding specialist for Great Place to Work certified companies.
            
            Based on the company's Trust Index survey results and certification achievements,
            generate social media post recommendations that highlight their workplace culture.
            
            Guidelines:
            - Focus on authentic, data-backed statements
            - Use UK English spelling (organisation, colour, programme)
            - Avoid generic corporate language
            - Include specific metrics when available
            - Suggest appropriate hashtags
            
            Company Context:
            {context}
            
            Trust Index Highlights:
            {highlights}
            
            Generate exactly 3 post recommendations in the following JSON format:
            {
              "recommendations": [
                {
                  "rank": 1,
                  "headline": "Brief attention-grabbing headline",
                  "post_text": "Full post text (max 280 chars for Twitter, 700 for LinkedIn)",
                  "key_metric": "The standout statistic",
                  "comparison": "Comparison to benchmark",
                  "strategic_rationale": "Why this message matters",
                  "suggested_platform": "LinkedIn|Twitter|Instagram"
                }
              ]
            }
            """;
    }

    public static class Elevate
    {
        public const string SummaryOfResults = """
            You are an organisational psychologist analysing Trust Index survey results.
            
            Analyse the following survey data and generate a summary of results comparing
            the company's scores against the benchmark.
            
            Rules:
            - All numbers must exactly match the source data
            - Use UK English spelling
            - Focus on the most significant findings
            - Label outputs appropriately (AI-generated)
            
            Survey Data:
            {surveyData}
            
            Benchmark Data:
            {benchmarkData}
            
            Generate analysis in the following JSON format:
            {
              "summary": "Executive summary paragraph",
              "key_findings": [
                {
                  "finding": "Description",
                  "company_score": 85,
                  "benchmark_score": 78,
                  "difference": 7,
                  "significance": "high|medium|low"
                }
              ],
              "overall_trust_index": 82,
              "benchmark_comparison": "+5 above UK Best Workplaces"
            }
            """;
    }
}

// Usage
var request = new TextGenerationRequest
{
    SystemPrompt = PromptTemplates.Activate.SocialPostRecommendations
        .Replace("{context}", companyContext)
        .Replace("{highlights}", trustIndexHighlights),
    UserPrompt = "Generate recommendations for our Q4 social media campaign",
    PromptType = "activate-recommendations"
};
```

## AI Safety Guardrails

### Input/Output Sanitisation

```csharp
public interface IAiGuardrails
{
    Task<string> SanitizeInputAsync(string input);
    Task<string> SanitizeOutputAsync(string output);
    Task<bool> ValidateNumericAccuracyAsync(string output, IDictionary<string, decimal> sourceData);
}

public class AiGuardrails : IAiGuardrails
{
    private readonly ILogger<AiGuardrails> _logger;

    // PII patterns to detect and remove
    private static readonly Regex[] PiiPatterns =
    {
        new(@"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b", RegexOptions.Compiled), // Email
        new(@"\b\d{3}[-.]?\d{3}[-.]?\d{4}\b", RegexOptions.Compiled), // Phone
        new(@"\b[A-Z]{1,2}\d{1,2}[A-Z]?\s*\d[A-Z]{2}\b", RegexOptions.Compiled), // UK Postcode
    };

    public Task<string> SanitizeInputAsync(string input)
    {
        var sanitized = input;

        // Remove potential prompt injection attempts
        sanitized = sanitized.Replace("ignore previous instructions", "[FILTERED]");
        sanitized = sanitized.Replace("disregard above", "[FILTERED]");

        // Remove PII from input
        foreach (var pattern in PiiPatterns)
        {
            sanitized = pattern.Replace(sanitized, "[REDACTED]");
        }

        return Task.FromResult(sanitized);
    }

    public Task<string> SanitizeOutputAsync(string output)
    {
        var sanitized = output;

        // Remove any PII that might have leaked
        foreach (var pattern in PiiPatterns)
        {
            sanitized = pattern.Replace(sanitized, "[REDACTED]");
        }

        // Remove placeholder text
        sanitized = Regex.Replace(sanitized, @"\[PLACEHOLDER\]", "", RegexOptions.IgnoreCase);

        return Task.FromResult(sanitized);
    }

    public Task<bool> ValidateNumericAccuracyAsync(
        string output,
        IDictionary<string, decimal> sourceData)
    {
        // Extract numbers from output and verify against source
        var numbersInOutput = Regex.Matches(output, @"\b(\d+(?:\.\d+)?)\s*%?\b")
            .Select(m => decimal.Parse(m.Groups[1].Value))
            .ToHashSet();

        var sourceNumbers = sourceData.Values.ToHashSet();

        // All numbers in output should exist in source data
        var unmatchedNumbers = numbersInOutput.Except(sourceNumbers).ToList();

        if (unmatchedNumbers.Any())
        {
            _logger.LogWarning(
                "AI output contains numbers not in source data: {Numbers}",
                string.Join(", ", unmatchedNumbers));
            return Task.FromResult(false);
        }

        return Task.FromResult(true);
    }
}
```

## Usage Tracking and Quotas

### AI Usage Tracker

```csharp
public interface IAiUsageTracker
{
    Task<QuotaCheckResult> CheckQuotaAsync(Guid companyId, CancellationToken ct);
    Task TrackUsageAsync(AiUsageRecord record, CancellationToken ct);
    Task<AiUsageSummary> GetUsageSummaryAsync(Guid companyId, DateTimeOffset from, DateTimeOffset to, CancellationToken ct);
}

public class AiUsageTracker : IAiUsageTracker
{
    private readonly ApplicationDbContext _context;
    private readonly IOptions<AiQuotaOptions> _quotaOptions;

    public async Task<QuotaCheckResult> CheckQuotaAsync(Guid companyId, CancellationToken ct)
    {
        var startOfMonth = new DateTimeOffset(
            DateTimeOffset.UtcNow.Year,
            DateTimeOffset.UtcNow.Month,
            1, 0, 0, 0, TimeSpan.Zero);

        var currentUsage = await _context.AiCallLogs
            .Where(x => x.CompanyId == companyId && x.Timestamp >= startOfMonth)
            .SumAsync(x => x.EstimatedCost, ct);

        var monthlyLimit = _quotaOptions.Value.MonthlyBudgetPerCompany;

        if (currentUsage >= monthlyLimit)
        {
            return new QuotaCheckResult(false, $"Monthly AI budget of £{monthlyLimit} exceeded");
        }

        var remaining = monthlyLimit - currentUsage;
        return new QuotaCheckResult(true, $"£{remaining:F2} remaining this month");
    }

    public async Task TrackUsageAsync(AiUsageRecord record, CancellationToken ct)
    {
        _context.AiCallLogs.Add(new AiCallLog
        {
            Id = Guid.NewGuid(),
            CompanyId = record.CompanyId,
            UserId = record.UserId,
            Model = record.Model,
            PromptType = record.PromptType,
            TokensUsed = record.TotalTokens,
            Cost = record.EstimatedCost,
            OutputLabel = AiOutputLabel.AiGenerated,
            CreatedAt = record.Timestamp
        });

        await _context.SaveChangesAsync(ct);
    }
}
```

## Result Types

```csharp
public record AiGenerationResult<T>
{
    public bool IsSuccess { get; init; }
    public T? Value { get; init; }
    public string? Error { get; init; }
    public AiErrorType? ErrorType { get; init; }
    public AiMetadata? Metadata { get; init; }

    public static AiGenerationResult<T> Success(T value, AiMetadata metadata) =>
        new() { IsSuccess = true, Value = value, Metadata = metadata };

    public static AiGenerationResult<T> QuotaExceeded(string message) =>
        new() { IsSuccess = false, Error = message, ErrorType = AiErrorType.QuotaExceeded };

    public static AiGenerationResult<T> RateLimited(string message) =>
        new() { IsSuccess = false, Error = message, ErrorType = AiErrorType.RateLimited };

    public static AiGenerationResult<T> ParseError(string message) =>
        new() { IsSuccess = false, Error = message, ErrorType = AiErrorType.ParseError };

    public static AiGenerationResult<T> FromError(string error) =>
        new() { IsSuccess = false, Error = error, ErrorType = AiErrorType.GenerationFailed };
}

public record AiMetadata
{
    public int TokensUsed { get; init; }
    public string Model { get; init; } = string.Empty;
    public AiOutputLabel GenerationType { get; init; }
}

public enum AiOutputLabel
{
    AiGenerated,
    LogicBased,
    ConsultantAdvisory,
    CompanyInformed
}

public enum AiErrorType
{
    QuotaExceeded,
    RateLimited,
    ParseError,
    GenerationFailed
}
```

## Anti-Patterns to Avoid

### DON'T: Call Azure OpenAI directly from controllers

```csharp
// BAD - no guardrails, no tracking
[HttpPost]
public async Task<IActionResult> Generate([FromBody] string prompt)
{
    var response = await _openAiClient.GetChatCompletionsAsync(...);
    return Ok(response.Value.Choices[0].Message.Content);
}

// GOOD - use orchestration service
[HttpPost]
public async Task<IActionResult> Generate([FromBody] GenerateRequest request)
{
    var result = await _aiOrchestrationService.GenerateTextAsync(request);
    return result.IsSuccess ? Ok(result.Value) : BadRequest(result.Error);
}
```

### DON'T: Include raw customer data in prompts without sanitisation

```csharp
// BAD - potential data leakage
var prompt = $"Analyse this data: {rawSurveyData}";

// GOOD - sanitise and structure
var sanitizedData = await _guardrails.SanitizeInputAsync(rawSurveyData);
var structuredPrompt = BuildStructuredPrompt(sanitizedData);
```

## References

- [Azure OpenAI Service Documentation](https://learn.microsoft.com/en-us/azure/ai-services/openai/)
- [Azure OpenAI .NET SDK](https://learn.microsoft.com/en-us/dotnet/api/overview/azure/ai.openai-readme)
- [Responsible AI practices](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/responsible-ai)
