# Hangfire Job Scheduler Service

A centralized, standalone ASP.NET Core Web API service that uses Hangfire to receive and execute recurring jobs from different .NET applications.

## Features

- ✅ **Multiple Job Types**: Recurring (cron-based), Fire-and-forget (immediate), and Delayed jobs
- ✅ **HTTP-Based Execution**: Execute jobs by making HTTP requests (GET, POST, PUT, DELETE)
- ✅ **Priority Queues**: Support for multiple priority queues (critical, normal, low, default)
- ✅ **Automatic Retries**: Configurable retry mechanism with max retry attempts
- ✅ **Web Dashboard**: Built-in Hangfire dashboard for monitoring and management
- ✅ **API Security**: API key authentication for job submission
- ✅ **Persistent Storage**: SQL Server storage for job persistence and reliability
- ✅ **Structured Logging**: Serilog integration with console and file logging
- ✅ **Docker Support**: Ready-to-use Docker and Docker Compose configuration

## Prerequisites

- .NET 8.0 SDK or later
- SQL Server (2019 or later) or SQL Server Express
- Docker (optional, for containerized deployment)

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/jpbrugal/hangfire-job-scheduler.git
cd hangfire-job-scheduler
```

### 2. Configure Database Connection

Edit `appsettings.json` and update the connection string:

```json
"ConnectionStrings": {
  "HangfireConnection": "Server=localhost;Database=HangfireJobs;Trusted_Connection=True;TrustServerCertificate=True;"
}
```

### 3. Run the Application

```bash
dotnet restore
dotnet build
dotnet run
```

The service will start at:
- API: `https://localhost:7001` or `http://localhost:5000`
- Hangfire Dashboard: `https://localhost:7001/hangfire`

### 4. Using Docker

```bash
docker-compose up -d
```

This will start both SQL Server and the Job Scheduler service.

## API Endpoints

### Schedule a Job

**POST** `/api/jobs/schedule`

Headers:
```
X-Api-Key: your-secret-api-key-here
Content-Type: application/json
```

#### Recurring Job Example

```json
{
  "jobName": "DailyReportJob",
  "jobType": "recurring",
  "cronExpression": "0 2 * * *",
  "httpMethod": "POST",
  "targetUrl": "https://your-app.com/api/reports/generate",
  "headers": {
    "Authorization": "Bearer your-token"
  },
  "body": {
    "reportType": "daily"
  },
  "queue": "normal",
  "maxRetries": 3
}
```

#### Fire-and-Forget Job Example

```json
{
  "jobName": "SendWelcomeEmail",
  "jobType": "fire-and-forget",
  "httpMethod": "POST",
  "targetUrl": "https://your-app.com/api/emails/welcome",
  "headers": {
    "Authorization": "Bearer your-token"
  },
  "body": {
    "userId": "12345",
    "email": "user@example.com"
  },
  "queue": "critical",
  "maxRetries": 5
}
```

#### Delayed Job Example

```json
{
  "jobName": "SendReminderEmail",
  "jobType": "delayed",
  "delayInSeconds": 3600,
  "httpMethod": "POST",
  "targetUrl": "https://your-app.com/api/emails/reminder",
  "body": {
    "message": "Don't forget to complete your profile!"
  },
  "queue": "low",
  "maxRetries": 2
}
```

### Delete a Recurring Job

**DELETE** `/api/jobs/recurring/{jobName}`

Headers:
```
X-Api-Key: your-secret-api-key-here
```

### Trigger a Recurring Job Manually

**POST** `/api/jobs/trigger/{jobName}`

Headers:
```
X-Api-Key: your-secret-api-key-here
```

## Common Cron Expressions

```
* * * * *        Every minute
*/5 * * * *      Every 5 minutes
0 * * * *        Every hour
0 0 * * *        Every day at midnight
0 2 * * *        Every day at 2 AM
0 9 * * 1        Every Monday at 9 AM
0 9 * * 1-5      Every weekday at 9 AM
0 15 15 * *      Every 15th of month at 3 PM
0 0 1 * *        First day of every month
```

## Client Integration Example (C#)

```csharp
using System.Net.Http.Headers;
using System.Text;
using System.Text.Json;

public class JobSchedulerClient
{
    private readonly HttpClient _httpClient;
    private readonly string _baseUrl;

    public JobSchedulerClient(string baseUrl, string apiKey)
    {
        _baseUrl = baseUrl;
        _httpClient = new HttpClient();
        _httpClient.DefaultRequestHeaders.Add("X-Api-Key", apiKey);
    }

    public async Task<string> ScheduleRecurringJobAsync()
    {
        var jobRequest = new
        {
            jobName = "DailyReportJob",
            jobType = "recurring",
            cronExpression = "0 2 * * *",
            httpMethod = "POST",
            targetUrl = "https://your-app.com/api/reports/generate",
            headers = new Dictionary<string, string>
            {
                { "Authorization", "Bearer your-token" }
            },
            body = new { reportType = "daily" },
            queue = "normal",
            maxRetries = 3
        };

        var content = new StringContent(
            JsonSerializer.Serialize(jobRequest),
            Encoding.UTF8,
            "application/json"
        );

        var response = await _httpClient.PostAsync(
            $"{_baseUrl}/api/jobs/schedule", 
            content
        );
        
        return await response.Content.ReadAsStringAsync();
    }
}

// Usage
var client = new JobSchedulerClient(
    "https://localhost:7001", 
    "your-secret-api-key-here"
);
var result = await client.ScheduleRecurringJobAsync();
```

## Configuration

### appsettings.json

```json
{
  "ConnectionStrings": {
    "HangfireConnection": "Server=localhost;Database=HangfireJobs;Trusted_Connection=True;TrustServerCertificate=True;"
  },
  "HangfireSettings": {
    "DashboardPath": "/hangfire",
    "WorkerCount": 20,
    "Queues": ["default", "critical", "normal", "low"]
  },
  "ApiSettings": {
    "RequireApiKey": true,
    "ApiKey": "your-secret-api-key-here"
  }
}
```

## Security Considerations

1. **API Key**: Change the default API key in production
2. **Dashboard Access**: Configure proper authentication for the Hangfire dashboard
3. **HTTPS**: Always use HTTPS in production
4. **SQL Server**: Use proper SQL authentication and secure connection strings
5. **Network**: Restrict access to the service using firewall rules

## Troubleshooting

### Database Connection Issues

- Ensure SQL Server is running
- Verify connection string is correct
- Check SQL Server allows remote connections
- Verify firewall settings

### Jobs Not Executing

- Check Hangfire dashboard for failed jobs
- Review logs in the `logs/` directory
- Verify target URLs are accessible
- Check API keys and authentication

### Dashboard Not Loading

- Verify the dashboard path in configuration
- Check if running on localhost
- Review `HangfireDashboardAuthorizationFilter` settings

## Project Structure

```
/
├── Controllers/
│   └── JobsController.cs
├── Models/
│   ├── JobRequest.cs
│   ├── JobResponse.cs
│   └── JobExecutionContext.cs
├── Services/
│   ├── IJobExecutor.cs
│   └── JobExecutor.cs
├── Middleware/
│   └── ApiKeyAuthenticationMiddleware.cs
├── Filters/
│   └── HangfireDashboardAuthorizationFilter.cs
├── Examples/
│   └── ClientExample.cs
├── Program.cs
├── appsettings.json
├── JobSchedulerService.csproj
├── Dockerfile
└── docker-compose.yml
```

## License

MIT License

## Support

For issues and questions, please open an issue on GitHub.
