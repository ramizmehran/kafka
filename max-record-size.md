# New Producer Configuration: max.record.size

## Introduction

This document describes a new Kafka producer configuration parameter `max.record.size` that was added to improve handling of large records that are significantly compressed.

## Problem Statement

Currently, Kafka producers have a `max.request.size` configuration that limits the size of the request sent to Kafka brokers. This configuration serves two purposes:

1. It limits the maximum size of an individual record before compression
2. It limits the total size of a compressed batch of records in a request

This dual purpose can lead to inefficiencies and unexpected behaviors, particularly when records are significantly large before compression but compress to a much smaller size. During spikes in data transmission, even when compressed records fit within the `max.request.size` limit, large batches formed by highly compressed records can cause increased latency and processing backlog.

## Solution: max.record.size Configuration

The new `max.record.size` configuration parameter allows administrators to define the maximum size of a record before compression, separate from the compressed request size limit.

### Benefits

- **Predictability**: Producers can reject records that exceed the `max.record.size` before spending resources on compression.
- **Efficiency**: Helps maintain efficient batch sizes and system throughput, especially under high load conditions.
- **System Stability**: Avoids the potential for large batch processing which can affect latency and throughput negatively.

### Configuration Details

```properties
# Maximum size of a record in bytes before compression
max.record.size=1048576 # Default is 1MB (same as max.request.size default)
```

### Example Use Case

Consider a scenario where the producer sends records up to 20 MB in size which, when compressed with an efficient algorithm like zstd, fit into a batch under the 25 MB `max.request.size` multiple times. These large batches can be problematic to process efficiently.

With `max.record.size`, we can separate the concerns:
- `max.request.size` continues to limit the compressed request size
- `max.record.size` controls the maximum uncompressed record size

By setting `max.record.size` to, for example, 5 MB, we can prevent very large uncompressed records from being sent, thus avoiding latency spikes even if they would compress to fit within the `max.request.size`.

## Implementation

The `max.record.size` check is performed before compression, as part of the `ensureValidRecordSize` method in `KafkaProducer`. This ensures that oversized records are rejected early, before resources are spent on serialization and compression.

## Default Value

The default value for `max.record.size` is set to 1MB, matching the default of `max.request.size`. This maintains backward compatibility with existing systems.

## API Impact

This new configuration does not impact the public API. Existing applications will continue to work as before, with the `max.request.size` limiting both compressed requests and uncompressed record sizes.

## Compatibility

This change is fully backward compatible. If `max.record.size` is not specified, the behavior is identical to previous versions, using `max.request.size` for both limits. 