using System.Diagnostics;

namespace CleanArchitecture.Api.Middleware;

/// <summary>
/// Ensures every request has a correlation ID: reuses an inbound
/// X-Correlation-Id header if present, otherwise generates one.
/// Makes the ID available via ICorrelationIdProvider, attaches it
/// to the logging scope, and echoes it back on the response.
/// </summary>
public sealed class CorrelationIdMiddleware
{
    private const string HeaderName = "X-Correlation-Id";

    private readonly RequestDelegate _next;
    private readonly ILogger<CorrelationIdMiddleware> _logger;

    public CorrelationIdMiddleware(RequestDelegate next, ILogger<CorrelationIdMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context, ICorrelationIdProvider correlationIdProvider)
    {
        var correlationId = ResolveCorrelationId(context);

        // Make it available to everything downstream (services, repositories,
        // event handlers) without them needing to touch HttpContext.
        correlationIdProvider.Set(correlationId);
        context.Items[HeaderName] = correlationId;

        // Tie it into the current Activity so it flows through any
        // System.Diagnostics-based tracing already in place.
        Activity.Current?.SetTag("correlation_id", correlationId);

        // Echo it back before the response starts, so callers can
        // correlate their request with server-side logs.
        context.Response.OnStarting(() =>
        {
            context.Response.Headers[HeaderName] = correlationId;
            return Task.CompletedTask;
        });

        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["CorrelationId"] = correlationId
        }))
        {
            await _next(context);
        }
    }

    private static string ResolveCorrelationId(HttpContext context)
    {
        if (context.Request.Headers.TryGetValue(HeaderName, out var existing) &&
            !string.IsNullOrWhiteSpace(existing))
        {
            return existing.ToString();
        }

        return Guid.NewGuid().ToString();
    }
}

/// <summary>
/// Scoped accessor so services outside the HTTP pipeline (application
/// services, background dispatch, outbox writers) can read the current
/// request's correlation ID without a dependency on HttpContext.
/// </summary>
public interface ICorrelationIdProvider
{
    string CorrelationId { get; }
    void Set(string correlationId);
}

public sealed class CorrelationIdProvider : ICorrelationIdProvider
{
    public string CorrelationId { get; private set; } = string.Empty;

    public void Set(string correlationId) => CorrelationId = correlationId;
}

// Program.cs registration:
//
// builder.Services.AddScoped<ICorrelationIdProvider, CorrelationIdProvider>();
// ...
// app.UseMiddleware<CorrelationIdMiddleware>(); // before exception handling & logging
// app.UseMiddleware<RequestContextLoggingMiddleware>();