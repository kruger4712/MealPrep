# Monitoring Setup Guide

## Overview
Comprehensive monitoring and observability setup for the MealPrep application, covering application performance monitoring (APM), logging, metrics collection, alerting, and health checks across development, staging, and production environments.

## Monitoring Architecture

### Observability Stack
```
???????????????????????????????????????????????????????????????
?                     User Experience                        ?
?                   • Synthetic Tests                        ?
?                   • Real User Monitoring                   ?
???????????????????????????????????????????????????????????????
                      ?
???????????????????????????????????????????????????????????????
?                Application Layer                           ?
?  • APM (Application Performance Monitoring)                ?
?  • Error Tracking                                         ?
?  • Custom Business Metrics                                ?
???????????????????????????????????????????????????????????????
                      ?
???????????????????????????????????????????????????????????????
?                Infrastructure Layer                        ?
?  • Container Metrics (Docker)                             ?
?  • Database Performance                                   ?
?  • Redis Cache Metrics                                    ?
?  • Network and Security Metrics                           ?
???????????????????????????????????????????????????????????????
                      ?
???????????????????????????????????????????????????????????????
?                   Data Layer                              ?
?  • Prometheus (Metrics Storage)                           ?
?  • Grafana (Visualization)                                ?
?  • ELK Stack (Logging)                                    ?
?  • AlertManager (Alerting)                                ?
???????????????????????????????????????????????????????????????
```

## Application Performance Monitoring

### .NET Core APM Setup
```csharp
// Program.cs - APM Integration
public static void Main(string[] args)
{
    var builder = WebApplication.CreateBuilder(args);

    // Add APM services
    builder.Services.AddApplicationInsights(builder.Configuration);
    builder.Services.AddSingleton<ITelemetryProcessor, FilteringTelemetryProcessor>();
    
    // Add custom metrics
    builder.Services.AddSingleton<IMetricsService, MetricsService>();
    builder.Services.AddSingleton<IHealthCheckService, HealthCheckService>();

    // Configure health checks
    builder.Services.AddHealthChecks()
        .AddCheck<DatabaseHealthCheck>("database")
        .AddCheck<RedisHealthCheck>("redis")
        .AddCheck<ExternalApiHealthCheck>("gemini-ai")
        .AddCheck<DiskSpaceHealthCheck>("disk_space");

    var app = builder.Build();

    // Configure middleware
    app.UseMiddleware<RequestMetricsMiddleware>();
    app.UseMiddleware<ErrorTrackingMiddleware>();
    
    // Health check endpoints
    app.MapHealthChecks("/health", new HealthCheckOptions
    {
        ResponseWriter = WriteHealthCheckResponse
    });
    
    app.MapHealthChecks("/health/ready", new HealthCheckOptions
    {
        Predicate = check => check.Tags.Contains("ready"),
        ResponseWriter = WriteHealthCheckResponse
    });
    
    app.MapHealthChecks("/health/live", new HealthCheckOptions
    {
        Predicate = _ => false,
        ResponseWriter = WriteHealthCheckResponse
    });

    app.Run();
}

// Custom metrics service
public class MetricsService : IMetricsService
{
    private readonly IMetrics _metrics;
    private readonly ILogger<MetricsService> _logger;

    // Business metrics
    private readonly Counter<long> _recipesCreated;
    private readonly Counter<long> _aiSuggestionsRequested;
    private readonly Counter<long> _menuPlansGenerated;
    private readonly Histogram<double> _aiResponseTime;
    private readonly Gauge<int> _activeUsers;

    public MetricsService(IMeterFactory meterFactory, ILogger<MetricsService> logger)
    {
        _logger = logger;
        var meter = meterFactory.Create("MealPrep.API");

        // Initialize counters
        _recipesCreated = meter.CreateCounter<long>(
            "mealprep_recipes_created_total",
            description: "Total number of recipes created");

        _aiSuggestionsRequested = meter.CreateCounter<long>(
            "mealprep_ai_suggestions_requested_total",
            description: "Total number of AI suggestions requested");

        _menuPlansGenerated = meter.CreateCounter<long>(
            "mealprep_menu_plans_generated_total",
            description: "Total number of menu plans generated");

        // Initialize histograms
        _aiResponseTime = meter.CreateHistogram<double>(
            "mealprep_ai_response_duration_seconds",
            description: "AI service response time in seconds");

        // Initialize gauges
        _activeUsers = meter.CreateGauge<int>(
            "mealprep_active_users_current",
            description: "Current number of active users");
    }

    public void RecordRecipeCreated(string userId, string recipeType)
    {
        _recipesCreated.Add(1, new TagList
        {
            ["user_id"] = userId,
            ["recipe_type"] = recipeType
        });
    }

    public void RecordAiSuggestionRequested(string userId, string suggestionType, TimeSpan responseTime)
    {
        _aiSuggestionsRequested.Add(1, new TagList
        {
            ["user_id"] = userId,
            ["suggestion_type"] = suggestionType
        });

        _aiResponseTime.Record(responseTime.TotalSeconds, new TagList
        {
            ["suggestion_type"] = suggestionType
        });
    }

    public void UpdateActiveUsers(int count)
    {
        _activeUsers.Record(count);
    }
}
```

### Request Metrics Middleware
```csharp
// Middleware/RequestMetricsMiddleware.cs
public class RequestMetricsMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IMetricsService _metrics;
    private readonly ILogger<RequestMetricsMiddleware> _logger;

    // Metrics
    private readonly Counter<long> _requestsTotal;
    private readonly Histogram<double> _requestDuration;
    private readonly Counter<long> _requestErrors;

    public RequestMetricsMiddleware(
        RequestDelegate next,
        IMetricsService metrics,
        ILogger<RequestMetricsMiddleware> logger,
        IMeterFactory meterFactory)
    {
        _next = next;
        _metrics = metrics;
        _logger = logger;

        var meter = meterFactory.Create("MealPrep.API.Requests");
        
        _requestsTotal = meter.CreateCounter<long>(
            "http_requests_total",
            description: "Total number of HTTP requests");

        _requestDuration = meter.CreateHistogram<double>(
            "http_request_duration_seconds",
            description: "HTTP request duration in seconds");

        _requestErrors = meter.CreateCounter<long>(
            "http_request_errors_total",
            description: "Total number of HTTP request errors");
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();
        var path = context.Request.Path.Value?.ToLowerInvariant() ?? "unknown";
        var method = context.Request.Method;

        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            // Record error
            _requestErrors.Add(1, new TagList
            {
                ["method"] = method,
                ["path"] = SanitizePath(path),
                ["exception_type"] = ex.GetType().Name
            });

            _logger.LogError(ex, "Request failed for {Method} {Path}", method, path);
            throw;
        }
        finally
        {
            stopwatch.Stop();
            var statusCode = context.Response.StatusCode;

            // Record request metrics
            _requestsTotal.Add(1, new TagList
            {
                ["method"] = method,
                ["path"] = SanitizePath(path),
                ["status_code"] = statusCode.ToString()
            });

            _requestDuration.Record(stopwatch.Elapsed.TotalSeconds, new TagList
            {
                ["method"] = method,
                ["path"] = SanitizePath(path),
                ["status_code"] = statusCode.ToString()
            });
        }
    }

    private string SanitizePath(string path)
    {
        // Replace dynamic segments with placeholders
        path = Regex.Replace(path, @"/api/recipes/[0-9a-f-]{36}", "/api/recipes/{id}");
        path = Regex.Replace(path, @"/api/users/[0-9a-f-]{36}", "/api/users/{id}");
        path = Regex.Replace(path, @"/api/families/[0-9a-f-]{36}", "/api/families/{id}");
        
        return path;
    }
}
```

## Health Checks Implementation

### Custom Health Checks
```csharp
// HealthChecks/DatabaseHealthCheck.cs
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly MealPrepDbContext _context;
    private readonly ILogger<DatabaseHealthCheck> _logger;

    public DatabaseHealthCheck(MealPrepDbContext context, ILogger<DatabaseHealthCheck> logger)
    {
        _context = context;
        _logger = logger;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            // Test database connectivity
            var canConnect = await _context.Database.CanConnectAsync(cancellationToken);
            if (!canConnect)
            {
                return HealthCheckResult.Unhealthy("Cannot connect to database");
            }

            // Test query performance
            var stopwatch = Stopwatch.StartNew();
            var userCount = await _context.Users.CountAsync(cancellationToken);
            stopwatch.Stop();

            var responseTime = stopwatch.ElapsedMilliseconds;
            var data = new Dictionary<string, object>
            {
                ["user_count"] = userCount,
                ["response_time_ms"] = responseTime,
                ["connection_string"] = _context.Database.GetConnectionString()?.Substring(0, 20) + "..."
            };

            if (responseTime > 5000) // 5 seconds
            {
                return HealthCheckResult.Degraded("Database response time is slow", data: data);
            }

            return HealthCheckResult.Healthy("Database is healthy", data);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Database health check failed");
            return HealthCheckResult.Unhealthy("Database health check failed", ex);
        }
    }
}

// HealthChecks/RedisHealthCheck.cs
public class RedisHealthCheck : IHealthCheck
{
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<RedisHealthCheck> _logger;

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var database = _redis.GetDatabase();
            var stopwatch = Stopwatch.StartNew();
            
            // Test Redis connectivity and performance
            var testKey = $"health_check_{Guid.NewGuid()}";
            await database.StringSetAsync(testKey, "test", TimeSpan.FromSeconds(30));
            var value = await database.StringGetAsync(testKey);
            await database.KeyDeleteAsync(testKey);
            
            stopwatch.Stop();

            var responseTime = stopwatch.ElapsedMilliseconds;
            var data = new Dictionary<string, object>
            {
                ["response_time_ms"] = responseTime,
                ["connected_endpoints"] = _redis.GetEndPoints().Length
            };

            if (value != "test")
            {
                return HealthCheckResult.Unhealthy("Redis read/write test failed", data: data);
            }

            if (responseTime > 1000) // 1 second
            {
                return HealthCheckResult.Degraded("Redis response time is slow", data: data);
            }

            return HealthCheckResult.Healthy("Redis is healthy", data);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Redis health check failed");
            return HealthCheckResult.Unhealthy("Redis health check failed", ex);
        }
    }
}

// HealthChecks/ExternalApiHealthCheck.cs
public class ExternalApiHealthCheck : IHealthCheck
{
    private readonly HttpClient _httpClient;
    private readonly IConfiguration _configuration;
    private readonly ILogger<ExternalApiHealthCheck> _logger;

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var geminiApiKey = _configuration["Gemini:ApiKey"];
            if (string.IsNullOrEmpty(geminiApiKey))
            {
                return HealthCheckResult.Degraded("Gemini API key not configured");
            }

            var stopwatch = Stopwatch.StartNew();
            
            // Test Gemini AI connectivity
            var response = await _httpClient.GetAsync(
                "https://generativelanguage.googleapis.com/v1/models?key=" + geminiApiKey,
                cancellationToken);
            
            stopwatch.Stop();

            var responseTime = stopwatch.ElapsedMilliseconds;
            var data = new Dictionary<string, object>
            {
                ["response_time_ms"] = responseTime,
                ["status_code"] = (int)response.StatusCode
            };

            if (!response.IsSuccessStatusCode)
            {
                return HealthCheckResult.Unhealthy("Gemini AI API is not accessible", data: data);
            }

            if (responseTime > 10000) // 10 seconds
            {
                return HealthCheckResult.Degraded("Gemini AI response time is slow", data: data);
            }

            return HealthCheckResult.Healthy("External APIs are healthy", data);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "External API health check failed");
            return HealthCheckResult.Unhealthy("External API health check failed", ex);
        }
    }
}
```

## Prometheus Configuration

### Prometheus Setup
```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

scrape_configs:
  # MealPrep API metrics
  - job_name: 'mealprep-api'
    static_configs:
      - targets: ['mealprep-api:5000']
    metrics_path: '/metrics'
    scrape_interval: 10s
    
  # Database metrics
  - job_name: 'sql-server'
    static_configs:
      - targets: ['sql-exporter:9399']
    scrape_interval: 30s
    
  # Redis metrics
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']
    scrape_interval: 15s
    
  # Container metrics
  - job_name: 'docker-containers'
    static_configs:
      - targets: ['cadvisor:8080']
    scrape_interval: 15s
    
  # Node metrics
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
    scrape_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

# Alert rules
# prometheus/alert_rules.yml
groups:
  - name: mealprep_alerts
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: rate(http_request_errors_total[5m]) > 0.1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} requests/second"

      # High response time
      - alert: HighResponseTime
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High response time detected"
          description: "95th percentile response time is {{ $value }}s"

      # Database connection issues
      - alert: DatabaseDown
        expr: up{job="sql-server"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Database is down"
          description: "SQL Server is not responding"

      # Redis connection issues
      - alert: RedisDown
        expr: up{job="redis"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis is down"
          description: "Redis cache is not responding"

      # AI service issues
      - alert: AiServiceSlow
        expr: histogram_quantile(0.95, rate(mealprep_ai_response_duration_seconds_bucket[10m])) > 30
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "AI service is slow"
          description: "AI response time is {{ $value }}s (95th percentile)"

      # High memory usage
      - alert: HighMemoryUsage
        expr: (container_memory_usage_bytes / container_spec_memory_limit_bytes) > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"
          description: "Container memory usage is {{ $value | humanizePercentage }}"

      # Low disk space
      - alert: LowDiskSpace
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Low disk space"
          description: "Disk space is {{ $value | humanizePercentage }} full"
```

## Grafana Dashboards

### MealPrep Application Dashboard
```json
{
  "dashboard": {
    "id": null,
    "title": "MealPrep Application Dashboard",
    "tags": ["mealprep", "application"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "Request Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "Requests/sec"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "reqps"
          }
        }
      },
      {
        "id": 2,
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "50th percentile"
          },
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "95th percentile"
          },
          {
            "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "99th percentile"
          }
        ],
        "yAxes": [
          {
            "unit": "s"
          }
        ]
      },
      {
        "id": 3,
        "title": "Error Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_request_errors_total[5m])",
            "legendFormat": "Errors/sec"
          }
        ]
      },
      {
        "id": 4,
        "title": "AI Suggestions",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(mealprep_ai_suggestions_requested_total[5m])",
            "legendFormat": "AI Requests/sec"
          }
        ]
      },
      {
        "id": 5,
        "title": "Database Performance",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(sql_server_database_io_read_bytes[5m])",
            "legendFormat": "Read bytes/sec"
          },
          {
            "expr": "rate(sql_server_database_io_write_bytes[5m])",
            "legendFormat": "Write bytes/sec"
          }
        ]
      }
    ],
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "refresh": "30s"
  }
}
```

## Logging Configuration

### Structured Logging Setup
```csharp
// Program.cs - Logging configuration
builder.Logging.ClearProviders();
builder.Logging.AddConsole();
builder.Logging.AddSerilog(new LoggerConfiguration()
    .ReadFrom.Configuration(builder.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithProperty("Application", "MealPrep.API")
    .Enrich.WithProperty("Environment", builder.Environment.EnvironmentName)
    .WriteTo.Console(outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}")
    .WriteTo.File("logs/mealprep-.log", 
        rollingInterval: RollingInterval.Day,
        retainedFileCountLimit: 30,
        outputTemplate: "{Timestamp:yyyy-MM-dd HH:mm:ss.fff} [{Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}")
    .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri(builder.Configuration.GetConnectionString("Elasticsearch")))
    {
        IndexFormat = "mealprep-logs-{0:yyyy.MM.dd}",
        AutoRegisterTemplate = true,
        AutoRegisterTemplateVersion = AutoRegisterTemplateVersion.ESv7
    })
    .CreateLogger());

// Custom logging extensions
public static class LoggingExtensions
{
    public static void LogUserAction(this ILogger logger, string userId, string action, object? details = null)
    {
        logger.LogInformation("User action: {UserId} performed {Action} with details {@Details}",
            userId, action, details);
    }

    public static void LogApiCall(this ILogger logger, string endpoint, TimeSpan duration, bool success)
    {
        logger.LogInformation("API call to {Endpoint} took {Duration}ms and {Status}",
            endpoint, duration.TotalMilliseconds, success ? "succeeded" : "failed");
    }

    public static void LogBusinessEvent(this ILogger logger, string eventType, object eventData)
    {
        logger.LogInformation("Business event: {EventType} with data {@EventData}",
            eventType, eventData);
    }
}
```

## AlertManager Configuration

### Alert Routing and Notification
```yaml
# alertmanager/alertmanager.yml
global:
  smtp_smarthost: 'localhost:587'
  smtp_from: 'alerts@mealprep.com'

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'
  routes:
  - match:
      severity: critical
    receiver: critical-alerts
  - match:
      severity: warning
    receiver: warning-alerts

receivers:
- name: 'web.hook'
  webhook_configs:
  - url: 'http://127.0.0.1:5001/'

- name: 'critical-alerts'
  email_configs:
  - to: 'devops@mealprep.com'
    subject: 'CRITICAL: {{ .GroupLabels.alertname }}'
    body: |
      {{ range .Alerts }}
      Alert: {{ .Annotations.summary }}
      Description: {{ .Annotations.description }}
      {{ end }}
  slack_configs:
  - api_url: 'YOUR_SLACK_WEBHOOK_URL'
    channel: '#alerts'
    title: 'CRITICAL ALERT'
    text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'

- name: 'warning-alerts'
  email_configs:
  - to: 'team@mealprep.com'
    subject: 'WARNING: {{ .GroupLabels.alertname }}'
    body: |
      {{ range .Alerts }}
      Alert: {{ .Annotations.summary }}
      Description: {{ .Annotations.description }}
      {{ end }}

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
```

## Docker Compose Monitoring Stack

### Complete Monitoring Setup
```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=200h'
      - '--web.enable-lifecycle'
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager:/etc/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
      - '--web.external-url=http://localhost:9093'
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3001:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    restart: unless-stopped

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.6.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    restart: unless-stopped

  kibana:
    image: docker.elastic.co/kibana/kibana:8.6.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped

volumes:
  prometheus-data:
  grafana-data:
  elasticsearch-data:
```

This comprehensive monitoring setup provides complete observability for the MealPrep application with metrics collection, logging, alerting, and visualization across all application components.

*This monitoring guide should be updated as the application scales and new monitoring requirements emerge.*