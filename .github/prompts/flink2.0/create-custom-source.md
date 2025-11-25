# Create Custom Source

Create custom source connectors in Apache Flink for reading from non-standard data sources.

## Requirements

- **SourceFunction Interface**: Implement for single-parallel sources
- **ParallelSourceFunction**: Implement for distributed sources
- **Checkpointing**: Support state for recovery
- **Timestamp/Watermark**: Emit timestamps and watermarks
- **Source Splitting**: Divide data for parallel processing
- **Backpressure**: Handle backpressure correctly
- **Error Handling**: Graceful failure recovery
- **Configuration**: Externalize source parameters
- **Examples**: Common source patterns

## Code Style

- Language: Java 17 LTS
- Framework: Apache Flink 2.0
- API: Modern source API (Source interface)

## Template

```java
import org.apache.flink.api.connector.source.*;
import org.apache.flink.api.connector.source.util.SerializableSupplier;
import org.apache.flink.core.io.SimpleVersionedSerializer;
import org.apache.flink.connector.base.source.reader.*;
import lombok.extern.slf4j.Slf4j;
import java.io.*;
import java.util.*;

/**
 * Custom source for reading from external system.
 */
@Slf4j
public class CustomSource {

    /**
     * Simple sequential source: Read from external API sequentially.
     */
    public static class SequentialApiSource implements SourceFunction<Data> {
        @Override
        public void run(SourceContext<Data> ctx) throws Exception {
            for (int i = 0; i < 1000; i++) {
                Data data = fetchFromApi(i);
                ctx.collect(data);
                Thread.sleep(100);  // Rate limit
            }
        }

        @Override
        public void cancel() {}

        private Data fetchFromApi(int id) {
            return new Data(id, "data-" + id);
        }
    }

    /**
     * Parallel source: Distribute reading across multiple tasks.
     */
    public static class ParallelFileSource implements ParallelSourceFunction<Line> {
        private int taskIndex;
        private int taskCount;

        @Override
        public void run(SourceContext<Line> ctx) throws Exception {
            for (int i = taskIndex; i < 10000; i += taskCount) {
                ctx.collect(new Line(i, "line-" + i));
            }
        }

        @Override
        public void cancel() {}

        @Override
        public void setRuntimeContext(RuntimeContext t) {
            taskIndex = t.getIndexOfThisSubtask();
            taskCount = t.getNumberOfParallelSubtasks();
        }
    }

    /**
     * Checkpointed source: Support recovery from checkpoint.
     */
    @Slf4j
    public static class CheckpointedSource implements SourceFunction<Message>, 
            Checkpointed<Integer> {
        private volatile int messageId = 0;
        private volatile boolean isRunning = true;

        @Override
        public void run(SourceContext<Message> ctx) throws Exception {
            while (isRunning && messageId < 100000) {
                synchronized (ctx.getCheckpointLock()) {
                    ctx.collect(new Message(messageId, "msg-" + messageId));
                    messageId++;
                }
                Thread.sleep(10);
            }
        }

        @Override
        public void cancel() { isRunning = false; }

        @Override
        public Integer snapshotState(long checkpointId, long timestamp) {
            return messageId;
        }

        @Override
        public void restoreState(Integer state) {
            messageId = state;
            log.info("Restored to message ID: {}", messageId);
        }
    }

    /**
     * Source with timestamps and watermarks.
     */
    @Slf4j
    public static class TimestampedSource implements SourceFunction<TimedData> {
        private volatile boolean isRunning = true;

        @Override
        public void run(SourceContext<TimedData> ctx) throws Exception {
            long startTime = System.currentTimeMillis();
            int messageCount = 0;

            while (isRunning && messageCount < 10000) {
                long eventTime = startTime + (messageCount * 100);
                TimedData data = new TimedData(messageCount, "data-" + messageCount, eventTime);
                
                synchronized (ctx.getCheckpointLock()) {
                    ctx.collectWithTimestamp(data, eventTime);
                    if (messageCount % 100 == 0) {
                        ctx.emitWatermark(new Watermark(eventTime - 1000));
                    }
                }
                messageCount++;
                Thread.sleep(10);
            }
        }

        @Override
        public void cancel() { isRunning = false; }
    }

    // Helper classes
    public record Data(int id, String value) {}
    public record Line(int lineNum, String content) {}
    public record Message(int id, String payload) {}
    public record TimedData(int id, String data, long timestamp) {}
}
```

## Related Documentation

- [Custom Sources](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/connectors/datastream/overview/)
