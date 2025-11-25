# Create Trace Sampling Strategies

Implement trace sampling to manage cost and performance in high-volume systems. Sampling determines which traces are recorded and exported, reducing observability overhead while maintaining visibility into errors and anomalies.

## Requirements

Your trace sampling implementation should include:

- **Static Sampling**: Simple percentage-based sampling (always on, fixed rate)
- **Adaptive Sampling**: Dynamic sampling based on traffic patterns and error rates
- **Tail-Based Sampling**: Sample based on trace properties (status, latency, errors)
- **Custom Sampling Logic**: Implement business-specific sampling rules
- **Error-Rate Sampling**: Always sample traces containing errors
- **Latency-Based Sampling**: Increase sampling for slow operations
- **Rate Limiting**: Ensure sampling doesn't exceed throughput limits
- **Sampling Configuration**: Externalize sampling strategy configuration
- **Metrics**: Report sampling decisions and sampled trace counts
- **Documentation**: Clear guidance on sampling strategy selection

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Spring Boot, OpenTelemetry SDK
- **Sampling**: Custom Sampler implementations
- **Configuration**: Environment variables + properties files

## Template Example

```java
import io.opentelemetry.sdk.trace.samplers.Sampler;
import io.opentelemetry.sdk.trace.samplers.SamplingDecision;
import io.opentelemetry.sdk.trace.SamplingDecision;
import io.opentelemetry.api.trace.SpanKind;
import io.opentelemetry.context.Context;
import io.opentelemetry.sdk.trace.samplers.ProbabilitySampler;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Bean;
import org.springframework.beans.factory.annotation.Value;
import lombok.extern.slf4j.Slf4j;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.List;
import java.util.ArrayList;
import java.util.Collections;

/**
 * Trace sampling strategies for managing observability overhead.
 * Implements various sampling approaches from static to adaptive.
 */
@Slf4j
@Configuration
public class TraceSamplingConfig {

    @Value("${otel.sampling.type:fixed}")
    private String samplingType;

    @Value("${otel.sampling.rate:0.1}")
    private double samplingRate;

    @Value("${otel.sampling.error.always-sample:true}")
    private boolean alwaysSampleErrors;

    @Value("${otel.sampling.latency.threshold-ms:1000}")
    private long latencyThresholdMs;

    /**
     * Static sampling: Fixed percentage of traces are sampled.
     * Use for: Low traffic systems, non-critical services.
     * Example: Sample 10% of all requests (0.1 rate).
     */
    @Bean
    public Sampler staticSampler() {
        return Sampler.traceIdRatioBased(samplingRate);
    }

    /**
     * Always-on sampling: Every trace is sampled.
     * Use for: Critical services, low traffic, debugging.
     */
    @Bean
    public Sampler alwaysOnSampler() {
        return Sampler.alwaysOn();
    }

    /**
     * Always-off sampling: No traces are sampled (cost optimization).
     * Use for: Non-critical services, high traffic, cost-sensitive.
     */
    @Bean
    public Sampler alwaysOffSampler() {
        return Sampler.alwaysOff();
    }

    /**
     * Parent-based sampling: Respect parent span's sampling decision.
     * Use for: Distributed systems, honor upstream sampling.
     * Ensures trace integrity across service boundaries.
     */
    @Bean
    public Sampler parentBasedSampler() {
        return Sampler.parentBased(Sampler.traceIdRatioBased(samplingRate));
    }

    /**
     * Custom composite sampler: Combines multiple sampling strategies.
     * Use for: Complex requirements with multiple conditions.
     */
    @Bean
    public Sampler compositeSampler() {
        return new CompositeSampler(
                List.of(
                        new ErrorAlwaysSampler(),
                        new LatencyBasedSampler(latencyThresholdMs),
                        new TraceIdRatioBased(samplingRate)
                )
        );
    }

    /**
     * Error-aware sampling: Always sample traces with errors.
     * Use for: Error tracking and debugging.
     * Ensures all errors are visible in tracing backend.
     */
    public static class ErrorAlwaysSampler implements Sampler {
        private static final String SAMPLE_FOR_ERROR_KEY = "sampleForError";

        @Override
        public SamplingResult shouldSample(
                Context parentContext,
                String traceId,
                String name,
                SpanKind spanKind,
                java.util.Map<String, Object> attributes,
                java.util.List<io.opentelemetry.sdk.trace.SpanContext> parentLinks) {

            // Check if parent has error flag
            if (parentContext != null) {
                Boolean hasError = parentContext.get(io.opentelemetry.context.ContextKey.named(SAMPLE_FOR_ERROR_KEY));
                if (hasError != null && hasError) {
                    return SamplingResult.create(
                            SamplingDecision.RECORD_AND_SAMPLE,
                            java.util.Collections.singletonList(
                                    io.opentelemetry.api.common.AttributeKey.booleanKey(SAMPLE_FOR_ERROR_KEY)
                            )
                    );
                }
            }

            // Default: don't sample, but allow override for errors
            return SamplingResult.create(
                    SamplingDecision.RECORD_AND_SAMPLE,
                    java.util.Collections.emptyList()
            );
        }

        @Override
        public String getDescription() {
            return "ErrorAlwaysSampler";
        }
    }

    /**
     * Latency-based sampling: Sample traces with high latency.
     * Use for: Performance monitoring, detecting slow operations.
     */
    public static class LatencyBasedSampler implements Sampler {
        private final long latencyThresholdMs;

        public LatencyBasedSampler(long latencyThresholdMs) {
            this.latencyThresholdMs = latencyThresholdMs;
        }

        @Override
        public SamplingResult shouldSample(
                Context parentContext,
                String traceId,
                String name,
                SpanKind spanKind,
                java.util.Map<String, Object> attributes,
                java.util.List<io.opentelemetry.sdk.trace.SpanContext> parentLinks) {

            // Note: Latency is unknown at span start time.
            // Use tail-based sampling for latency decisions.
            return SamplingResult.create(
                    SamplingDecision.RECORD_AND_SAMPLE,
                    java.util.Collections.emptyList()
            );
        }

        @Override
        public String getDescription() {
            return "LatencyBasedSampler(threshold=" + latencyThresholdMs + "ms)";
        }
    }

    /**
     * Composite sampler: Apply multiple sampling strategies.
     * Samples if ANY strategy decides to sample.
     */
    public static class CompositeSampler implements Sampler {
        private final List<Sampler> samplers;

        public CompositeSampler(List<Sampler> samplers) {
            this.samplers = samplers;
        }

        @Override
        public SamplingResult shouldSample(
                Context parentContext,
                String traceId,
                String name,
                SpanKind spanKind,
                java.util.Map<String, Object> attributes,
                java.util.List<io.opentelemetry.sdk.trace.SpanContext> parentLinks) {

            for (Sampler sampler : samplers) {
                SamplingResult result = sampler.shouldSample(
                        parentContext, traceId, name, spanKind, attributes, parentLinks);
                if (result.getDecision() == SamplingDecision.RECORD_AND_SAMPLE) {
                    return result;
                }
            }

            return SamplingResult.create(SamplingDecision.DROP);
        }

        @Override
        public String getDescription() {
            return "CompositeSampler(" + samplers.size() + " strategies)";
        }
    }

    /**
     * Adaptive sampler: Adjusts sampling rate based on traffic.
     * Use for: High-traffic systems with variable load.
     * Maintains target span count regardless of traffic volume.
     */
    @Slf4j
    public static class AdaptiveSampler implements Sampler {
        private final long targetTracesPerSecond;
        private final AtomicInteger tracesInWindow = new AtomicInteger(0);
        private long windowStartMs = System.currentTimeMillis();
        private double currentSamplingRate = 0.1;

        public AdaptiveSampler(long targetTracesPerSecond) {
            this.targetTracesPerSecond = targetTracesPerSecond;
        }

        @Override
        public SamplingResult shouldSample(
                Context parentContext,
                String traceId,
                String name,
                SpanKind spanKind,
                java.util.Map<String, Object> attributes,
                java.util.List<io.opentelemetry.sdk.trace.SpanContext> parentLinks) {

            long now = System.currentTimeMillis();
            long windowElapsed = now - windowStartMs;

            // Adjust sampling rate every 10 seconds
            if (windowElapsed > 10000) {
                int tracesObserved = tracesInWindow.getAndSet(0);
                windowStartMs = now;

                // Adjust rate: if we sampled too many/few traces, adjust the rate
                double actualTracesPerSecond = tracesObserved / (windowElapsed / 1000.0);
                if (actualTracesPerSecond > 0) {
                    currentSamplingRate = Math.min(1.0,
                            currentSamplingRate * (targetTracesPerSecond / actualTracesPerSecond));
                }

                log.info("Adaptive sampling rate adjusted to: {}", String.format("%.4f", currentSamplingRate));
            }

            tracesInWindow.incrementAndGet();

            // Sample based on current rate
            long traceIdLong = Long.parseUnsignedLong(traceId.substring(0, 16), 16);
            if (traceIdLong < Long.MAX_VALUE * currentSamplingRate) {
                return SamplingResult.create(SamplingDecision.RECORD_AND_SAMPLE);
            }

            return SamplingResult.create(SamplingDecision.DROP);
        }

        @Override
        public String getDescription() {
            return "AdaptiveSampler(target=" + targetTracesPerSecond + "/sec)";
        }
    }

    /**
     * Tail-based sampling: Sample based on trace completion properties.
     * Use for: Error detection, performance analysis.
     * Note: Requires custom export pipeline to implement tail-based decisions.
     */
    @Slf4j
    public static class TailBasedSamplingDecider {
        private final long errorSampleRate;
        private final long slowOperationThresholdMs;
        private final double slowOperationSampleRate;

        public TailBasedSamplingDecider(long errorSampleRate,
                                       long slowOperationThresholdMs,
                                       double slowOperationSampleRate) {
            this.errorSampleRate = errorSampleRate;
            this.slowOperationThresholdMs = slowOperationThresholdMs;
            this.slowOperationSampleRate = slowOperationSampleRate;
        }

        /**
         * Determine if trace should be sampled based on final span attributes.
         */
        public boolean shouldSampleTrace(TraceData traceData) {
            // Always sample errors
            if (traceData.hasErrors()) {
                return true;
            }

            // Sample slow operations
            if (traceData.durationMs() > slowOperationThresholdMs) {
                return Math.random() < slowOperationSampleRate;
            }

            // Sample high-resource operations
            if (traceData.resourceUsagePercent() > 80) {
                return true;
            }

            // Don't sample
            return false;
        }
    }

    /**
     * Rate-limited sampler: Ensure sampling doesn't exceed throughput limit.
     * Use for: Cost control, preventing observability backend overload.
     */
    @Slf4j
    public static class RateLimitedSampler implements Sampler {
        private final long maxSpansPerSecond;
        private final AtomicInteger spansInCurrentSecond = new AtomicInteger(0);
        private long secondStartMs = System.currentTimeMillis();

        public RateLimitedSampler(long maxSpansPerSecond) {
            this.maxSpansPerSecond = maxSpansPerSecond;
        }

        @Override
        public SamplingResult shouldSample(
                Context parentContext,
                String traceId,
                String name,
                SpanKind spanKind,
                java.util.Map<String, Object> attributes,
                java.util.List<io.opentelemetry.sdk.trace.SpanContext> parentLinks) {

            long now = System.currentTimeMillis();
            if (now - secondStartMs >= 1000) {
                spansInCurrentSecond.set(0);
                secondStartMs = now;
            }

            int currentCount = spansInCurrentSecond.incrementAndGet();
            if (currentCount <= maxSpansPerSecond) {
                return SamplingResult.create(SamplingDecision.RECORD_AND_SAMPLE);
            }

            return SamplingResult.create(SamplingDecision.DROP);
        }

        @Override
        public String getDescription() {
            return "RateLimitedSampler(max=" + maxSpansPerSecond + "/sec)";
        }
    }

    /**
     * Span kind-specific sampling: Different rates for different span types.
     * Use for: Focusing on critical operations.
     */
    @Slf4j
    public static class SpanKindSampler implements Sampler {
        private final double serverSampleRate;
        private final double clientSampleRate;
        private final double internalSampleRate;

        public SpanKindSampler(double serverRate, double clientRate, double internalRate) {
            this.serverSampleRate = serverRate;
            this.clientSampleRate = clientRate;
            this.internalSampleRate = internalRate;
        }

        @Override
        public SamplingResult shouldSample(
                Context parentContext,
                String traceId,
                String name,
                SpanKind spanKind,
                java.util.Map<String, Object> attributes,
                java.util.List<io.opentelemetry.sdk.trace.SpanContext> parentLinks) {

            double rate = switch (spanKind) {
                case SERVER -> serverSampleRate;
                case CLIENT -> clientSampleRate;
                default -> internalSampleRate;
            };

            long traceIdLong = Long.parseUnsignedLong(traceId.substring(0, 16), 16);
            if (traceIdLong < Long.MAX_VALUE * rate) {
                return SamplingResult.create(SamplingDecision.RECORD_AND_SAMPLE);
            }

            return SamplingResult.create(SamplingDecision.DROP);
        }

        @Override
        public String getDescription() {
            return "SpanKindSampler(server=" + serverSampleRate + ", client=" + clientSampleRate + ")";
        }
    }

    /**
     * Sampling strategy comparison.
     */
    public static class SamplingStrategies {
        public static final String STATIC = "static";
        public static final String ALWAYS_ON = "always_on";
        public static final String ALWAYS_OFF = "always_off";
        public static final String PARENT_BASED = "parent_based";
        public static final String ADAPTIVE = "adaptive";
        public static final String COMPOSITE = "composite";
        public static final String ERROR_AWARE = "error_aware";
        public static final String RATE_LIMITED = "rate_limited";
    }

    /**
     * Metrics for sampling decisions.
     */
    @Slf4j
    public static class SamplingMetrics {
        private final AtomicInteger sampledCount = new AtomicInteger(0);
        private final AtomicInteger droppedCount = new AtomicInteger(0);

        public void recordSamplingDecision(boolean sampled) {
            if (sampled) {
                sampledCount.incrementAndGet();
            } else {
                droppedCount.incrementAndGet();
            }
        }

        public void logMetrics() {
            int sampled = sampledCount.get();
            int dropped = droppedCount.get();
            int total = sampled + dropped;

            if (total > 0) {
                double samplingRate = (double) sampled / total * 100;
                log.info("Sampling Metrics - Sampled: {}, Dropped: {}, Rate: {:.2f}%",
                        sampled, dropped, samplingRate);
            }
        }
    }

    /**
     * Trace data holder for tail-based sampling decisions.
     */
    public record TraceData(
            String traceId,
            long startTimeMs,
            long endTimeMs,
            boolean hasErrors,
            long resourceUsagePercent) {

        public long durationMs() {
            return endTimeMs - startTimeMs;
        }
    }

    /**
     * Best practices for sampling configuration.
     */
    public static class BestPractices {
        /**
         * 1. Start with 10% static sampling
         *    - Captures trends without overhead
         *    - Adjust based on backend costs
         */

        /**
         * 2. Always sample error traces
         *    - 100% error trace capture is recommended
         *    - Errors are critical for debugging
         */

        /**
         * 3. Sample slow operations
         *    - Operations exceeding 1000ms threshold
         *    - Helps identify performance issues
         */

        /**
         * 4. Use parent-based sampling for consistency
         *    - Respects upstream sampling decisions
         *    - Maintains trace integrity
         */

        /**
         * 5. Monitor sampling rates
         *    - Track actual sampled span counts
         *    - Alert if sampling rate deviates from target
         */

        /**
         * 6. Adjust sampling by environment
         *    - Production: 1-5% static + 100% errors
         *    - Staging: 20-50% static
         *    - Development: 100% (always-on)
         */

        /**
         * 7. Use tail-based sampling for complex decisions
         *    - Better visibility into anomalies
         *    - Requires infrastructure support (e.g., collector)
         */

        /**
         * 8. Test sampling configuration
         *    - Verify sampling decisions are applied
         *    - Confirm error traces are captured
         */
    }
}
```

## Sampling Strategy Comparison

| Strategy | Cost | Completeness | Error Visibility | Use Case |
|----------|------|--------------|------------------|----------|
| **Always-On** | High | 100% | Full | Critical systems, debugging |
| **Static 10%** | Low | 10% | 10% | Typical production |
| **Error + Static** | Medium | 10% + all errors | Full | Most systems |
| **Adaptive** | Variable | ~Target | Full | High-traffic systems |
| **Tail-Based** | Medium | Anomalies | Full | Performance analysis |
| **Rate-Limited** | Very Low | Limited | Limited | Cost-sensitive |

## Configuration Examples

```properties
# Static 10% sampling
otel.sampling.type=fixed
otel.sampling.rate=0.1

# Error-aware sampling
otel.sampling.type=composite
otel.sampling.error.always-sample=true
otel.sampling.latency.threshold-ms=1000

# Adaptive sampling for 100 traces/second
otel.sampling.type=adaptive
otel.sampling.target-traces-per-second=100

# Rate-limited to 1000 spans/second
otel.sampling.type=rate_limited
otel.sampling.max-spans-per-second=1000
```

## Related Documentation

- [OpenTelemetry Sampling Documentation](https://opentelemetry.io/docs/concepts/sampling/)
- [Jaeger Sampling Strategies](https://www.jaegertracing.io/docs/latest/sampling/)
- [Collector Tail Sampling](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor)
```
