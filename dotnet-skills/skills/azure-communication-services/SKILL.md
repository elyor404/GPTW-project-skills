---
name: azure-communication-services
description: Azure Communication Services for email sending - transactional emails, templates, invitations. Use when implementing email functionality.
---

# Azure Communication Services Email

## When to Use This Skill

Use this skill when:
- Sending user invitation emails
- Sending password reset emails
- Implementing transactional email notifications

## Email Service Setup

### Configuration

```csharp
// appsettings.json
{
  "AzureCommunicationServices": {
    "ConnectionString": "", // Use Key Vault
    "SenderAddress": "noreply@gptw-plus.com"
  }
}

public class EmailOptions
{
    public const string SectionName = "AzureCommunicationServices";
    public string ConnectionString { get; init; } = string.Empty;
    public string SenderAddress { get; init; } = string.Empty;
}

// Registration
builder.Services.Configure<EmailOptions>(
    builder.Configuration.GetSection(EmailOptions.SectionName));

builder.Services.AddSingleton(sp =>
{
    var options = sp.GetRequiredService<IOptions<EmailOptions>>().Value;
    return new EmailClient(options.ConnectionString);
});

builder.Services.AddScoped<IEmailService, EmailService>();
```

### Email Service Implementation

```csharp
public interface IEmailService
{
    Task<bool> SendInvitationEmailAsync(
        string recipientEmail,
        string recipientName,
        string setPasswordUrl,
        CancellationToken cancellationToken = default);

    Task<bool> SendPasswordResetEmailAsync(
        string recipientEmail,
        string resetUrl,
        CancellationToken cancellationToken = default);

    Task<bool> SendActivityReminderAsync(
        IEnumerable<string> recipientEmails,
        string companyName,
        CancellationToken cancellationToken = default);
}

public class EmailService : IEmailService
{
    private readonly EmailClient _emailClient;
    private readonly EmailOptions _options;
    private readonly ILogger<EmailService> _logger;

    public EmailService(
        EmailClient emailClient,
        IOptions<EmailOptions> options,
        ILogger<EmailService> logger)
    {
        _emailClient = emailClient;
        _options = options.Value;
        _logger = logger;
    }

    public async Task<bool> SendInvitationEmailAsync(
        string recipientEmail,
        string recipientName,
        string setPasswordUrl,
        CancellationToken cancellationToken = default)
    {
        var subject = "Welcome to GPTW PLUS+ - Set Your Password";
        var htmlContent = BuildInvitationEmailHtml(recipientName, setPasswordUrl);
        var plainTextContent = BuildInvitationEmailPlainText(recipientName, setPasswordUrl);

        return await SendEmailAsync(
            recipientEmail,
            subject,
            htmlContent,
            plainTextContent,
            cancellationToken);
    }

    public async Task<bool> SendPasswordResetEmailAsync(
        string recipientEmail,
        string resetUrl,
        CancellationToken cancellationToken = default)
    {
        var subject = "GPTW PLUS+ - Password Reset Request";
        var htmlContent = BuildPasswordResetEmailHtml(resetUrl);
        var plainTextContent = BuildPasswordResetEmailPlainText(resetUrl);

        return await SendEmailAsync(
            recipientEmail,
            subject,
            htmlContent,
            plainTextContent,
            cancellationToken);
    }

    public async Task<bool> SendActivityReminderAsync(
        IEnumerable<string> recipientEmails,
        string companyName,
        CancellationToken cancellationToken = default)
    {
        var subject = "GPTW PLUS+ - Monthly Activity Reminder";
        var htmlContent = BuildActivityReminderHtml(companyName);
        var plainTextContent = BuildActivityReminderPlainText(companyName);

        var tasks = recipientEmails.Select(email =>
            SendEmailAsync(email, subject, htmlContent, plainTextContent, cancellationToken));

        var results = await Task.WhenAll(tasks);
        return results.All(r => r);
    }

    private async Task<bool> SendEmailAsync(
        string recipientEmail,
        string subject,
        string htmlContent,
        string plainTextContent,
        CancellationToken cancellationToken)
    {
        try
        {
            var emailMessage = new EmailMessage(
                senderAddress: _options.SenderAddress,
                recipients: new EmailRecipients(new List<EmailAddress>
                {
                    new EmailAddress(recipientEmail)
                }),
                content: new EmailContent(subject)
                {
                    Html = htmlContent,
                    PlainText = plainTextContent
                });

            var operation = await _emailClient.SendAsync(
                WaitUntil.Completed,
                emailMessage,
                cancellationToken);

            _logger.LogInformation(
                "Email sent to {Recipient}, OperationId: {OperationId}",
                recipientEmail, operation.Id);

            return operation.HasCompleted && operation.Value.Status == EmailSendStatus.Succeeded;
        }
        catch (RequestFailedException ex)
        {
            _logger.LogError(ex,
                "Failed to send email to {Recipient}: {Error}",
                recipientEmail, ex.Message);
            return false;
        }
    }

    private static string BuildInvitationEmailHtml(string name, string url) => $"""
        <!DOCTYPE html>
        <html>
        <head>
            <style>
                body {{ font-family: Arial, sans-serif; line-height: 1.6; color: #333; }}
                .container {{ max-width: 600px; margin: 0 auto; padding: 20px; }}
                .button {{ 
                    display: inline-block; 
                    padding: 12px 24px; 
                    background-color: #0066cc; 
                    color: white; 
                    text-decoration: none; 
                    border-radius: 4px; 
                }}
                .footer {{ margin-top: 30px; font-size: 12px; color: #666; }}
            </style>
        </head>
        <body>
            <div class="container">
                <h1>Welcome to GPTW PLUS+</h1>
                <p>Dear {name},</p>
                <p>You have been invited to join GPTW PLUS+, the platform that helps Great Place to Work 
                   certified organisations maximise the value of their certification.</p>
                <p>Please click the button below to set your password and access the platform:</p>
                <p><a href="{url}" class="button">Set Your Password</a></p>
                <p>This link will expire in 24 hours.</p>
                <p>If you did not expect this invitation, please ignore this email.</p>
                <div class="footer">
                    <p>Great Place to Work UK<br/>
                    This is an automated message. Please do not reply to this email.</p>
                </div>
            </div>
        </body>
        </html>
        """;

    private static string BuildInvitationEmailPlainText(string name, string url) => $"""
        Welcome to GPTW PLUS+

        Dear {name},

        You have been invited to join GPTW PLUS+, the platform that helps Great Place to Work 
        certified organisations maximise the value of their certification.

        Please visit the following link to set your password and access the platform:
        {url}

        This link will expire in 24 hours.

        If you did not expect this invitation, please ignore this email.

        Great Place to Work UK
        This is an automated message. Please do not reply to this email.
        """;

    private static string BuildPasswordResetEmailHtml(string url) => $"""
        <!DOCTYPE html>
        <html>
        <head>
            <style>
                body {{ font-family: Arial, sans-serif; line-height: 1.6; color: #333; }}
                .container {{ max-width: 600px; margin: 0 auto; padding: 20px; }}
                .button {{ 
                    display: inline-block; 
                    padding: 12px 24px; 
                    background-color: #0066cc; 
                    color: white; 
                    text-decoration: none; 
                    border-radius: 4px; 
                }}
            </style>
        </head>
        <body>
            <div class="container">
                <h1>Password Reset Request</h1>
                <p>We received a request to reset your GPTW PLUS+ password.</p>
                <p>Click the button below to reset your password:</p>
                <p><a href="{url}" class="button">Reset Password</a></p>
                <p>This link will expire in 1 hour.</p>
                <p>If you did not request a password reset, please ignore this email.</p>
            </div>
        </body>
        </html>
        """;

    private static string BuildPasswordResetEmailPlainText(string url) => $"""
        Password Reset Request

        We received a request to reset your GPTW PLUS+ password.

        Please visit the following link to reset your password:
        {url}

        This link will expire in 1 hour.

        If you did not request a password reset, please ignore this email.
        """;

    private static string BuildActivityReminderHtml(string companyName) => $"""
        <!DOCTYPE html>
        <html>
        <body>
            <h1>Monthly Activity Reminder</h1>
            <p>This is a reminder that {companyName} has not logged any activities 
               on GPTW PLUS+ in the past month.</p>
            <p>Log in to explore Activate, Elevate, and Empower features to maximise 
               your Great Place to Work certification.</p>
        </body>
        </html>
        """;

    private static string BuildActivityReminderPlainText(string companyName) => $"""
        Monthly Activity Reminder

        This is a reminder that {companyName} has not logged any activities 
        on GPTW PLUS+ in the past month.

        Log in to explore Activate, Elevate, and Empower features to maximise 
        your Great Place to Work certification.
        """;
}
```

## References

- [Azure Communication Services Email](https://learn.microsoft.com/en-us/azure/communication-services/concepts/email/email-overview)
- [Email SDK for .NET](https://learn.microsoft.com/en-us/azure/communication-services/quickstarts/email/send-email)
