# Monitoring & Observability
## File 12: Metrics, Tracing & Alerting for Kafka in .NET

---

## What You'll Learn

- Key Kafka metrics to monitor
- Consumer lag monitoring (the most important metric)
- OpenTelemetry tracing with Kafka
- Serilog structured logging for Kafka events
- Health checks for Kafka
- Alerting rules and dashboards

**Time Required:** 45 minutes

---

## 1. Key Metrics to Monitor

### Producer Metrics
| Metric | Description | Alert If |
|--------|-------------|----------|
| `produce-throttle-time` | Time producer was throttled by broker | > 0ms consistently |
| `request-latency` | Time to get delivery ack from broker | > 500ms |
| `record-error-rate` | % of produces that failed | > 0% |
| `buffer-available-bytes` | Free space in producer buffer | < 20% of max |
| `batch-size-avg` | Average batch size | Too low = LingerMs too short |

### Consumer Metrics
| Metric | Description | Alert If |
|--------|-------------|----------|
| **Consumer Lag** | Messages behind the latest offset | > threshold (your SLA) |
| `fetch-latency-avg` | Time to fetch from broker | > 500ms |
| `records-consumed-rate` | Messages consumed per second | Drops significantly |
| `commit-latency-avg` | Time to commit offsets | > 1000ms |
| `rebalance-latency-avg` | Rebalance duration | > 30 seconds |
| `join-rate` | How often consumer joins group | Frequent = instability |

---

## 2. Consumer Lag Monitoring (.NET)

**Consumer lag is the most important metric** — it tells you how far behind your consumer is.

```csharp
// Services/KafkaLagMonitor.cs
using Confluent.Kafka;
using Confluent.Kafka.Admin;

public class KafkaLagMonitor : BackgroundService
{
    private readonly string _bootstrapServers;
    private readonly ILogger<KafkaLagMonitor> _logger;
    private readonly ConsumerGroupsToMonitor[] _groups;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await CheckLagAsync();
            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken); // Check every 30s
        }
    }

    private async Task CheckLagAsync()
    {
        var adminConfig = new AdminClientConfig { BootstrapServers = _bootstrapServers };
        using var adminClient = new AdminClientBuilder(adminConfig).Build();

        foreach (var group in _groups)
        {
            try
            {
                // Get the consumer group's committed offsets
                var groupDescriptions = adminClient.DescribeConsumerGroups(
                    new[] { group.GroupId },
                    new DescribeConsumerGroupsOptions { RequestTimeout = TimeSpan.FromSeconds(10) });

                var groupOffsets = adminClient.ListConsumerGroupOffsets(
                    new[] { new ConsumerGroupTopicPartitions(group.GroupId) });

                foreach (var topicPartitions in groupOffsets.First().Partitions)
                {
                    var tp = new TopicPartition(topicPartitions.Topic, topicPartitions.Partition);
                    var committedOffset = topicPartitions.Offset;

                    // Get end offset (latest message position)
                    var watermarks = adminClient.QueryWatermarkOffsets(tp, TimeSpan.FromSeconds(5));
                    var endOffset = watermarks.High.Value;
                    var lag = endOffset - committedOffset.Value;

                    // Log and alert
                    _logger.LogInformation(
                        "Group: {Group} | Topic: {Topic} | Partition: {Partition} | Lag: {Lag}",
                        group.GroupId, tp.Topic, tp.Partition.Value, lag);

                    // Alert if lag exceeds threshold
                    if (lag > group.AlertThreshold)
                    {
                        _logger.LogWarning(
                            "⚠️  HIGH LAG ALERT! Group: {Group} | Topic: {Topic} | P{Partition} | Lag: {Lag} (threshold: {Threshold})",
                            group.GroupId, tp.Topic, tp.Partition.Value, lag, group.AlertThreshold);

                        // Send alert (Teams, PagerDuty, etc.)
                        await SendAlertAsync(group.GroupId, tp.Topic, tp.Partition.Value, lag);
                    }

                    // Record as metric (for Prometheus/Grafana)
                    _consumerLagGauge
                        .WithLabels(group.GroupId, tp.Topic, tp.Partition.Value.ToString())
                        .Set(lag);
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to check lag for group {Group}", group.GroupId);
            }
        }
    }
}

public record ConsumerGroupsToMonitor(string GroupId, long AlertThreshold = 1000);
```

---

## 3. OpenTelemetry Tracing

Trace Kafka messages through your entire distributed system:

```powershell
dotnet add package OpenTelemetry
dotnet add package OpenTelemetry.Instrumentation.Http
dotnet add package OpenTelemetry.Exporter.OpenTelemetryProtocol
dotnet add package OpenTelemetry.Extensions.Hosting
dotnet add package Confluent.Kafka  # Already have this
```

```csharp
// Program.cs — OpenTelemetry configuration
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing =>
    {
        tracing
            .SetResourceBuilder(ResourceBuilder.CreateDefault()
                .AddService("order-service", serviceVersion: "1.0.0"))
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddSource("Kafka.Producer")   // Custom activity sources
            .AddSource("Kafka.Consumer")
            .AddOtlpExporter(opts =>
            {
                opts.Endpoint = new Uri("http://localhost:4317"); // Jaeger / OTLP collector
            });
    })
    .WithMetrics(metrics =>
    {
        metrics
            .AddAspNetCoreInstrumentation()
            .AddRuntimeInstrumentation()
            .AddMeter("Kafka.Consumer.Lag")
            .AddPrometheusExporter(); // Expose /metrics endpoint
    });

// Add Prometheus scrape endpoint
app.MapPrometheusScrapingEndpoint("/metrics");
```

### Instrumented Producer

```csharp
// TracingKafkaProducer.cs — wraps producer with distributed tracing
public class TracingKafkaProducer<TKey, TValue> : IKafkaProducer<TKey, TValue>
{
    private static readonly ActivitySource ActivitySource = new("Kafka.Producer");
    private readonly IProducer<TKey, TValue> _inner;

    public async Task<DeliveryResult<TKey, TValue>> ProduceAsync(
        string topic, TKey key, TValue value, CancellationToken ct = default)
    {
        using var activity = ActivitySource.StartActivity(
            $"{topic} publish",
            ActivityKind.Producer);

        activity?.SetTag("messaging.system", "kafka");
        activity?.SetTag("messaging.destination", topic);
        activity?.SetTag("messaging.destination_kind", "topic");
        activity?.SetTag("messaging.kafka.message_key", key?.ToString());

        // Propagate trace context in headers (W3C TraceContext standard)
        var headers = new Headers();
        if (activity != null)
        {
            var propagationContext = new PropagationContext(activity.Context, Baggage.Current);
            Propagators.DefaultTextMapPropagator.Inject(
                propagationContext,
                headers,
                (h, key, value) => h.Add(key, Encoding.UTF8.GetBytes(value)));
        }

        var message = new Message<TKey, TValue>
        {
            Key = key,
            Value = value,
            Headers = headers
        };

        try
        {
            var result = await _inner.ProduceAsync(topic, message, ct);
            activity?.SetTag("messaging.kafka.partition", result.Partition.Value);
            activity?.SetTag("messaging.kafka.offset", result.Offset.Value);
            return result;
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.RecordException(ex);
            throw;
        }
    }
}
```

### Instrumented Consumer

```csharp
// In your consumer loop:
private async Task ProcessWithTracingAsync(ConsumeResult<string, string> result)
{
    // Extract trace context from message headers
    var parentContext = Propagators.DefaultTextMapPropagator.Extract(
        default,
        result.Message.Headers,
        (headers, key) =>
        {
            if (headers.TryGetLastBytes(key, out var bytes))
                return new[] { Encoding.UTF8.GetString(bytes) };
            return Enumerable.Empty<string>();
        });

    using var activity = ActivitySource.StartActivity(
        $"{result.Topic} process",
        ActivityKind.Consumer,
        parentContext.ActivityContext); // Link to producer's trace!

    activity?.SetTag("messaging.system", "kafka");
    activity?.SetTag("messaging.destination", result.Topic);
    activity?.SetTag("messaging.kafka.partition", result.Partition.Value);
    activity?.SetTag("messaging.kafka.offset", result.Offset.Value);
    activity?.SetTag("messaging.kafka.consumer_group", _config.GroupId);

    try
    {
        await DoProcessingAsync(result.Message.Value);
        activity?.SetStatus(ActivityStatusCode.Ok);
    }
    catch (Exception ex)
    {
        activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        activity?.RecordException(ex);
        throw;
    }
}
```

---

## 4. Structured Logging with Serilog

```powershell
dotnet add package Serilog.AspNetCore
dotnet add package Serilog.Sinks.Console
dotnet add package Serilog.Sinks.ApplicationInsights  # For Azure
dotnet add package Serilog.Enrichers.Environment
dotnet add package Serilog.Enrichers.Thread
```

```csharp
// Program.cs
Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
    .MinimumLevel.Override("Confluent.Kafka", LogEventLevel.Warning)
    .Enrich.FromLogContext()
    .Enrich.WithEnvironmentName()
    .Enrich.WithMachineName()
    .Enrich.WithProperty("Application", "OrderService")
    .WriteTo.Console(new JsonFormatter())         // Structured JSON to console
    .WriteTo.ApplicationInsights(               // Azure Application Insights
        TelemetryConfiguration.CreateDefault(),
        TelemetryConverter.Traces)
    .CreateLogger();

builder.Host.UseSerilog();
```

### Consistent Kafka Logging Pattern

```csharp
// Use structured logging with Kafka context properties
_logger.LogInformation("Processing Kafka message {@KafkaContext}",
    new
    {
        Topic = result.Topic,
        Partition = result.Partition.Value,
        Offset = result.Offset.Value,
        MessageKey = result.Message.Key,
        ConsumerGroup = config.GroupId,
        Timestamp = result.Message.Timestamp.UtcDateTime,
    });

// Log processing outcome
_logger.LogInformation(
    "Order {OrderId} processed successfully | P[{Partition}] O[{Offset}] | Duration: {DurationMs}ms",
    order.OrderId,
    result.Partition.Value,
    result.Offset.Value,
    stopwatch.ElapsedMilliseconds);

// Log errors with full context
_logger.LogError(ex,
    "Failed to process order {OrderId} | P[{Partition}] O[{Offset}] | Attempt: {Attempt}/{MaxAttempts}",
    order.OrderId,
    result.Partition.Value,
    result.Offset.Value,
    attemptCount,
    maxRetries);
```

---

## 5. Health Checks for Kafka

```powershell
dotnet add package AspNetCore.HealthChecks.Kafka
```

```csharp
// Health check setup
builder.Services.AddHealthChecks()
    .AddKafka(new ProducerConfig
    {
        BootstrapServers = "localhost:9092",
        MessageTimeoutMs = 5000,
    },
    topic: "health-check",        // Kafka must have this topic
    name: "kafka",
    failureStatus: HealthStatus.Unhealthy,
    tags: new[] { "kafka", "messaging" });

// Map health check endpoints
app.MapHealthChecks("/health", new HealthCheckOptions
{
    Predicate = _ => true,
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse,
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("kafka"),
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false, // Always healthy if the app is running
});
```

### Custom Kafka Health Check

```csharp
// CustomKafkaHealthCheck.cs
public class KafkaHealthCheck : IHealthCheck
{
    private readonly string _bootstrapServers;

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            using var adminClient = new AdminClientBuilder(
                new AdminClientConfig { BootstrapServers = _bootstrapServers })
                .Build();

            // Try to get cluster metadata — fast check
            var metadata = adminClient.GetMetadata(TimeSpan.FromSeconds(5));

            var data = new Dictionary<string, object>
            {
                { "brokers", metadata.Brokers.Count },
                { "topics", metadata.Topics.Count },
            };

            return metadata.Brokers.Any()
                ? HealthCheckResult.Healthy("Kafka connected", data)
                : HealthCheckResult.Unhealthy("No brokers found");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Kafka unreachable", ex);
        }
    }
}

// Register
builder.Services.AddHealthChecks()
    .AddCheck<KafkaHealthCheck>("kafka-cluster");
```

---

## 6. Custom Metrics (Prometheus)

```csharp
// KafkaMetrics.cs — expose custom metrics for Prometheus/Grafana
using System.Diagnostics.Metrics;

public class KafkaConsumerMetrics : IDisposable
{
    private static readonly Meter Meter = new("Kafka.Consumer", "1.0.0");

    // Counters
    private static readonly Counter<long> MessagesProcessed =
        Meter.CreateCounter<long>("kafka_messages_processed_total",
            description: "Total messages successfully processed");

    private static readonly Counter<long> MessagesFailed =
        Meter.CreateCounter<long>("kafka_messages_failed_total",
            description: "Total messages that failed processing");

    private static readonly Counter<long> MessagesRetried =
        Meter.CreateCounter<long>("kafka_messages_retried_total",
            description: "Total message processing retries");

    // Histograms
    private static readonly Histogram<double> ProcessingDuration =
        Meter.CreateHistogram<double>("kafka_message_processing_duration_seconds",
            description: "Message processing duration in seconds");

    // Gauges (for lag)
    private static readonly ObservableGauge<long> ConsumerLag =
        Meter.CreateObservableGauge<long>("kafka_consumer_lag",
            observeValue: GetCurrentLag,
            description: "Current consumer lag per partition");

    // Usage in consumer:
    public void RecordProcessed(string topic, string consumerGroup, double durationSeconds)
    {
        MessagesProcessed.Add(1, new TagList
        {
            { "topic", topic },
            { "consumer_group", consumerGroup }
        });

        ProcessingDuration.Record(durationSeconds, new TagList
        {
            { "topic", topic }
        });
    }

    public void RecordFailed(string topic, string consumerGroup, string errorType)
    {
        MessagesFailed.Add(1, new TagList
        {
            { "topic", topic },
            { "consumer_group", consumerGroup },
            { "error_type", errorType }
        });
    }

    private static long GetCurrentLag() => _currentLag; // Updated by lag monitor

    public void Dispose() => Meter.Dispose();
}
```

---

## 7. Grafana Dashboard Queries (PromQL)

```promql
# Consumer lag by group and topic
kafka_consumer_lag{consumer_group="order-service"}

# Messages processed per second
rate(kafka_messages_processed_total[5m])

# Error rate percentage
rate(kafka_messages_failed_total[5m]) / rate(kafka_messages_processed_total[5m]) * 100

# 95th percentile processing time
histogram_quantile(0.95, rate(kafka_message_processing_duration_seconds_bucket[5m]))

# Consumer lag trend (alert if increasing for 5 minutes)
increase(kafka_consumer_lag[5m]) > 1000
```

---

## 8. Alerting Rules

```yaml
# alerting-rules.yaml (for Prometheus Alertmanager)
groups:
  - name: kafka-alerts
    rules:
      # High consumer lag
      - alert: KafkaConsumerHighLag
        expr: kafka_consumer_lag > 10000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High Kafka consumer lag for {{ $labels.consumer_group }}"
          description: "Consumer group {{ $labels.consumer_group }} is {{ $value }} messages behind on topic {{ $labels.topic }}"

      # Consumer group not consuming
      - alert: KafkaConsumerNotConsuming
        expr: rate(kafka_messages_processed_total[10m]) == 0
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Kafka consumer stopped consuming"
          description: "Consumer {{ $labels.consumer_group }} has not consumed any messages in 10 minutes"

      # High error rate
      - alert: KafkaHighErrorRate
        expr: >
          rate(kafka_messages_failed_total[5m]) /
          rate(kafka_messages_processed_total[5m]) > 0.05
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Kafka consumer error rate > 5%"

      # DLQ growing
      - alert: KafkaDLQGrowing
        expr: increase(kafka_consumer_lag{topic=~".*\\.dlq"}[30m]) > 10
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Dead Letter Queue is growing — manual intervention needed"
```

---

## 9. Kafka Monitoring Checklist

```
DAILY MONITORING:
✅ Consumer lag for all groups (trend, not just current value)
✅ DLQ message count (should be 0)
✅ Producer error rates
✅ Broker disk usage

WEEKLY REVIEW:
✅ Slow consumer groups (consistently high lag)
✅ Topic partition balance (one broker overloaded?)
✅ Log retention settings (are topics growing too large?)
✅ Consumer group health (any inactive groups?)

PRODUCTION ALERTS:
✅ Consumer lag > SLA threshold (e.g., > 10K messages = > 10 minutes behind)
✅ DLQ > 0 for more than 15 minutes
✅ Broker down (any broker offline)
✅ Producer delivery error rate > 0.1%
✅ Consumer not consuming for > 5 minutes
```

---

## Summary

You now know:
- ✅ Key Kafka metrics (especially consumer lag)
- ✅ Consumer lag monitoring with AdminClient
- ✅ OpenTelemetry tracing with context propagation across producer → consumer
- ✅ Serilog structured logging for Kafka events
- ✅ Health checks for Kafka connectivity
- ✅ Custom Prometheus metrics
- ✅ Grafana queries and alerting rules

**Next:** [13-REAL-WORLD-PROJECT.md](./13-REAL-WORLD-PROJECT.md) — Full project: Order Processing System
