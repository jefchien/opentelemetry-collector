# OpenTelemetry Collector Observability

## Goal

The goal of this document is to have a comprehensive description of observability of the Collector and changes needed to achieve observability part of our [vision](vision.md).

## What Needs Observation

The following elements of the Collector need to be observable.

### Current Values

- Resource consumption: CPU, RAM (in the future also IO - if we implement persistent queues) and any other metrics that may be available to Go apps (e.g. garbage size, etc).

- Receiving data rate, broken down by receivers and by data type (traces/metrics).

- Exporting data rate, broken down by exporters and by data type (traces/metrics).

- Data drop rate due to throttling, broken down by data type.

- Data drop rate due to invalid data received, broken down by data type.

- Current throttling state: Not Throttled/Throttled by Downstream/Internally Saturated.

- Incoming connection count, broken down by receiver.

- Incoming connection rate (new connections per second), broken down by receiver.

- In-memory queue size (in bytes and in units). Note: measurements in bytes may be difficult / expensive to obtain and should be used cautiously.

- Persistent queue size (when supported).

- End-to-end latency (from receiver input to exporter output). Note that with multiple receivers/exporters we potentially have NxM data paths, each with different latency (plus different pipelines in the future), so realistically we should likely expose the average of all data paths (perhaps broken down by pipeline).

- Latency broken down by pipeline elements (including exporter network roundtrip latency for request/response protocols).

“Rate” values must reflect the average rate of the last 10 seconds. Rates must exposed in bytes/sec and units/sec (e.g. spans/sec).

Note: some of the current values and rates may be calculated as derivatives of cumulative values in the backend, so it is an open question if we want to expose them separately or no.

### Cumulative Values

- Total received data, broken down by receivers and by data type (traces/metrics).

- Total exported data, broken down by exporters and by data type (traces/metrics).

- Total dropped data due to throttling, broken down by data type.

- Total dropped data due to invalid data received, broken down by data type.

- Total incoming connection count, broken down by receiver.

- Uptime since start.

### Trace or Log on Events

We want to generate the following events (log and/or send as a trace with additional data):

- Collector started/stopped.

- Collector reconfigured (if we support on-the-fly reconfiguration).

- Begin dropping due to throttling (include throttling reason, e.g. local saturation, downstream saturation, downstream unavailable, etc).

- Stop dropping due to throttling.

- Begin dropping due to invalid data (include sample/first invalid data).

- Stop dropping due to invalid data.

- Crash detected (differentiate clean stopping and crash, possibly include crash data if available).

For begin/stop events we need to define an appropriate hysteresis to avoid generating too many events. Note that begin/stop events cannot be detected in the backend simply as derivatives of current rates, the events include additional data that is not present in the current value.

### Host Metrics

The service should collect host resource metrics in addition to service's own process metrics. This may help to understand that the problem that we observe in the service is induced by a different process on the same host.

## How We Expose Telemetry

By default, the Collector exposes service telemetry in two ways currently:

- internal metrics are exposed via a Prometheus interface which defaults to port `8888`
- logs are emitted to stdout

Traces are not exposed by default. There is an effort underway to [change this][issue7532]. The work includes supporting
configuration of the OpenTelemetry SDK used to produce the Collector's internal telemetry. This feature is
currently behind two feature gates:

```bash
  --feature-gates=telemetry.useOtelForInternalMetrics
  --feature-gates=telemetry.useOtelWithSDKConfigurationForInternalTelemetry
```

The `useOtelForInternalMetrics` feature gate changes the internal telemetry to use OpenTelemetry rather
than OpenCensus. This will become the default at some point [in the future][issue7454]. The second gate,
`useOtelWithSDKConfigurationForInternalTelemetry` enables the Collector to parse configuration
that aligns with the [OpenTelemetry Configuration] schema. The support for this schema is still
experimental, but it does allow telemetry to be exported via OTLP.

The following configuration can be used in combination with the feature gates aforementioned
to emit internal metrics and traces from the Collector to an OTLP backend:

```yaml
service:
 telemetry:
   metrics:
     readers:
       - periodic:
           interval: 5000
           exporter:
             otlp:
               protocol: grpc/protobuf
               endpoint: https://backend:4317
   traces:
     processors:
       - batch:
           exporter:
             otlp:
               protocol: grpc/protobuf
               endpoint: https://backend2:4317
```

See the configuration's [example][kitchen-sink] for additional configuration options.

Note that this configuration does not support emitting logs as there is no support for [logs] in
OpenTelemetry Go SDK at this time.

### Impact

We need to be able to assess the impact of these observability improvements on the core performance of the Collector.

### Configurable Level of Observability

Some of the metrics/traces can be high volume and may not be desirable to always observe. We should consider adding an observability verboseness “level” that allows configuring the Collector to send more or less observability data (or even finer granularity to allow turning on/off specific metrics).

The default level of observability must be defined in a way that has insignificant performance impact on the service.

[issue7532]: https://github.com/open-telemetry/opentelemetry-collector/issues/7532
[issue7454]: https://github.com/open-telemetry/opentelemetry-collector/issues/7454
[logs]: https://github.com/open-telemetry/opentelemetry-go/issues/3827
[OpenTelemetry Configuration]: https://github.com/open-telemetry/opentelemetry-configuration
[kitchen-sink]: https://github.com/open-telemetry/opentelemetry-configuration/blob/main/examples/kitchen-sink.yaml
