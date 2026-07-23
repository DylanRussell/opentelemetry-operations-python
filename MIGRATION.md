# Migration Guide

This guide provides instructions on how to migrate from the custom exporters in this repository to the standard OpenTelemetry OTLP exporters.

## Overview

Google Cloud supports native OTLP (OpenTelemetry Protocol) ingestion for Cloud Trace, Cloud Monitoring, and Cloud Logging via the [Telemetry API](https://docs.cloud.google.com/stackdriver/docs/reference/telemetry/overview). This allows you to use standard OpenTelemetry OTLP exporters for sending telemetry data to Google Cloud.

## Deprecation Notice

All exporters in this repository (`opentelemetry-exporter-gcp-trace`, `opentelemetry-exporter-gcp-monitoring`, and `opentelemetry-exporter-gcp-logging`) are deprecated. Please migrate to standard OTLP exporters using standard OpenTelemetry libraries.

---

## Resource Detection (Recommended for All Signals)

When migrating to OTLP exporters, installing the GCP Resource Detector package (`opentelemetry-resourcedetector-gcp`) automatically populates Google Cloud resource attributes (such as `gcp.project_id`, `cloud.account.id`, `host.id`, `k8s.pod.name`, etc.) for OpenTelemetry SDK providers (`TracerProvider`, `MeterProvider`, `LoggerProvider`).

### Installation

```bash
pip install opentelemetry-resourcedetector-gcp
```

Once installed, the GCP resource detector entrypoint is automatically discovered and loaded by the OpenTelemetry SDK without requiring explicit manual resource setup code.

---

## Migrate from OpenTelemetry Google Cloud Trace Exporter (`CloudTraceSpanExporter`) to OTLP Exporter

To migrate from `opentelemetry-exporter-gcp-trace` (`CloudTraceSpanExporter`) to the standard OpenTelemetry OTLP exporter, follow these steps:

### 1. Add Dependencies

Install the standard OpenTelemetry OTLP exporter, GCP resource detector, and GCP authentication dependencies:

```bash
pip install opentelemetry-exporter-otlp-proto-grpc opentelemetry-resourcedetector-gcp google-auth grpcio requests
```

### 2. Configure the SDK

You can configure the SDK using environment variables:

```bash
# Environment Variables
export OTEL_EXPORTER_OTLP_ENDPOINT="https://telemetry.googleapis.com"
export OTEL_TRACES_EXPORTER="otlp"
export OTEL_EXPORTER_OTLP_PROTOCOL="http/protobuf"
```

Or programmatically in Python using GCP authentication:

```python
import google.auth
import google.auth.transport.grpc
import google.auth.transport.requests
import grpc
from google.auth.transport.grpc import AuthMetadataPlugin
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import (
    OTLPSpanExporter,
)
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

# Configure GCP Authentication
credentials, project_id = google.auth.default()
request = google.auth.transport.requests.Request()
auth_metadata_plugin = AuthMetadataPlugin(
    credentials=credentials, request=request
)
channel_creds = grpc.composite_channel_credentials(
    grpc.ssl_channel_credentials(),
    grpc.metadata_call_credentials(auth_metadata_plugin),
)

trace.set_tracer_provider(TracerProvider())

otlp_exporter = OTLPSpanExporter(
    endpoint="telemetry.googleapis.com",
    credentials=channel_creds,
)
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(otlp_exporter)
)
```

### 3. Follow the Migration Guide

For more details, follow the official Google Cloud guide: [Migrate from the Trace exporter to the OTLP endpoint](https://docs.cloud.google.com/trace/docs/migrate-to-otlp-endpoints).

### Mapping and Limitations

#### Configuration Mapping

| `CloudTraceSpanExporter` Parameter | OTLP Equivalent Property / Env Var | Notes |
| :--- | :--- | :--- |
| `project_id` / `OTEL_EXPORTER_GCP_TRACE_PROJECT_ID` | Resource attribute: `gcp.project_id` | Set via `OTEL_RESOURCE_ATTRIBUTES="gcp.project_id=your-project-id"` or detected automatically via `opentelemetry-resourcedetector-gcp`. |
| `client` | N/A | Pre-configured `TraceServiceClient` is replaced by OTLP gRPC/HTTP channel credentials. |
| `resource_regex` | Standard Resource Attributes | Standard OpenTelemetry exports resource attributes attached to the `TracerProvider`. Filtering/copying resource attributes via regex is not supported in OTLP. |

#### Unsupported Features

* **Attribute Mapping & Regex Filtering (`resource_regex`)**: `CloudTraceSpanExporter` allowed filtering resource attributes via regex to copy matching keys into span attributes. The standard OTLP exporter exports standard OpenTelemetry resource attributes attached to the `TracerProvider` directly.
* **Custom Trace Service Client (`client`)**: You cannot pass a pre-configured `TraceServiceClient` instance directly to `OTLPSpanExporter`. If custom gRPC channels or metadata credentials are required, configure gRPC channel credentials programmatically as shown above.

---

## Migrate from OpenTelemetry Google Cloud Monitoring Exporter (`CloudMonitoringMetricsExporter`) to OTLP Exporter

> [!WARNING]
> **Breaking Change Warning:** Migrating from the legacy Google Cloud Monitoring exporter to the standard OTLP exporter introduces breaking changes to your metric names.
>
> * **Legacy Exporter:** Ingests metrics under the `workload.googleapis.com/` domain (unless a custom prefix was configured).
> * **OTLP Exporter:** Ingests metrics under the `prometheus.googleapis.com/` domain by default.
>
> Because of this domain change, your metric names in Cloud Monitoring will change. **This will break any existing dashboards, alerting policies, and cause data discontinuity** between your historical and new metrics.

### Why Migrate?

While this migration introduces breaking changes, transitioning to the standard OTLP exporter is recommended for the following reasons:
* **Standardization:** Aligns your application with the industry-standard OpenTelemetry Protocol (OTLP), ensuring vendor neutrality and compatibility with the broader OpenTelemetry ecosystem.
* **Google Managed Prometheus (GMP) Cost Savings:** Standard OTLP metrics are ingested into Google Managed Prometheus. GMP offers a robust, scalable, and cost-effective monitoring solution (~20x cheaper ingestion cost than legacy Cloud Monitoring API ingestion).
* **Future-proofing:** The legacy Google Cloud Monitoring exporter is deprecated. Migrating now ensures your monitoring pipeline remains supported.

---

### Migration Strategies

We recommend three paths for migration, depending on your operational requirements:

1. **Direct Migration (Recommended):** Migrate fully to the OTLP exporter and update your dashboards and alerts to use the new metric names under the `prometheus.googleapis.com/` domain.
2. **Transition via Double-Writing (Alternative):** Run both the legacy exporter and the OTLP exporter in parallel. This allows you to validate the new OTLP pipeline and update dashboards/alerts without any monitoring downtime, at the cost of temporary double-ingestion charges.
3. **Custom Metric Renaming / View Configuration (Alternative):** Use OpenTelemetry SDK Views or metric processors (or OpenTelemetry Collector relabeling) to map metric names and attributes, allowing you to maintain compatibility with existing dashboards.

---

### Strategy 1: Direct Migration (Recommended)

Follow these steps to fully transition to the standard OTLP exporter.

#### 1. Add Dependencies

```bash
pip install opentelemetry-exporter-otlp-proto-grpc opentelemetry-resourcedetector-gcp google-auth grpcio
```

#### 2. Configure Environment Variables

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="https://telemetry.googleapis.com"
export OTEL_METRICS_EXPORTER="otlp"
export OTEL_EXPORTER_OTLP_PROTOCOL="http/protobuf"
export OTEL_RESOURCE_ATTRIBUTES="gcp.project_id=$PROJECT_ID,location=us-central1,service.name=otlp-sample,service.instance.id=1"
```

#### 3. Programmatic Configuration

```python
import google.auth
import google.auth.transport.grpc
import google.auth.transport.requests
import grpc
from google.auth.transport.grpc import AuthMetadataPlugin
from opentelemetry import metrics
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import (
    OTLPMetricExporter,
)
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader

# Configure GCP Authentication
credentials, project_id = google.auth.default()
request = google.auth.transport.requests.Request()
auth_metadata_plugin = AuthMetadataPlugin(
    credentials=credentials, request=request
)
channel_creds = grpc.composite_channel_credentials(
    grpc.ssl_channel_credentials(),
    grpc.metadata_call_credentials(auth_metadata_plugin),
)

exporter = OTLPMetricExporter(
    endpoint="telemetry.googleapis.com",
    credentials=channel_creds,
)
reader = PeriodicExportingMetricReader(exporter)
provider = MeterProvider(metric_readers=[reader])
metrics.set_meter_provider(provider)
```

### Mapping and Limitations

#### Configuration Mapping

| `CloudMonitoringMetricsExporter` Parameter | OTLP Equivalent Property / Env Var | Notes |
| :--- | :--- | :--- |
| `prefix` (default `workload.googleapis.com`) | N/A | Legacy exporter ingested under `workload.googleapis.com/` (or custom prefix). OTLP exporter ingests into Google Managed Prometheus under `prometheus.googleapis.com/` by default. |
| `add_unique_identifier` | Resource Attributes (e.g. `service.instance.id`, `host.id`) | Legacy exporter appended a random identifier to time series. OTLP relies on standard OpenTelemetry resource attributes to distinguish instances. |
| `project_id` | Resource attribute: `gcp.project_id` | Set via `OTEL_RESOURCE_ATTRIBUTES="gcp.project_id=your-project-id"` or detected automatically via `opentelemetry-resourcedetector-gcp`. |
| `client` | N/A | Pre-configured `MetricServiceClient` cannot be passed directly to `OTLPMetricExporter`. |

#### Limitations & Breaking Changes

* **Metric Domain & Prefix:** Metric names in Cloud Monitoring will change from `workload.googleapis.com/<metric>` to `prometheus.googleapis.com/<metric>/<type>`. Existing Cloud Monitoring dashboards and alerts relying on `workload.googleapis.com/` metrics must be updated to query `prometheus.googleapis.com/` metrics.
* **Unique Identifier Handling:** The `add_unique_identifier` parameter in `CloudMonitoringMetricsExporter` is not supported. Use standard resource attributes like `service.instance.id` or `host.id` to separate metric streams from distinct exporter instances.

---

## Migrate from OpenTelemetry Google Cloud Logging Exporter (`CloudLoggingExporter`) to OTLP Exporter

The OTLP `LogRecord` to Cloud Logging `LogEntry` conversion logic in standard OTLP endpoints is described in the [Google OTLP LogRecord to LogEntry specification](https://docs.cloud.google.com/stackdriver/docs/reference/telemetry/otlp-log-record-to-log-entry).

To migrate from `opentelemetry-exporter-gcp-logging` (`CloudLoggingExporter`) to the standard OpenTelemetry OTLP log exporter, follow these steps:

### 1. Add Dependencies

Install the standard OpenTelemetry OTLP log exporter package and GCP resource detector:

```bash
pip install opentelemetry-exporter-otlp-proto-grpc opentelemetry-resourcedetector-gcp google-auth grpcio
```

### 2. Configure Environment Variables

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="https://telemetry.googleapis.com"
export OTEL_LOGS_EXPORTER="otlp"
export OTEL_EXPORTER_OTLP_PROTOCOL="http/protobuf"
export OTEL_RESOURCE_ATTRIBUTES="gcp.project_id=$PROJECT_ID,service.name=otlp-sample,service.instance.id=1"
```

### 3. Programmatic Configuration

```python
import logging
import google.auth
import google.auth.transport.grpc
import google.auth.transport.requests
import grpc
from google.auth.transport.grpc import AuthMetadataPlugin
from opentelemetry import _logs
from opentelemetry.exporter.otlp.proto.grpc._log_exporter import (
    OTLPLogExporter,
)
from opentelemetry.sdk._logs import LoggerProvider
from opentelemetry.sdk._logs.export import BatchLogRecordProcessor

# Configure GCP Authentication
credentials, project_id = google.auth.default()
request = google.auth.transport.requests.Request()
auth_metadata_plugin = AuthMetadataPlugin(
    credentials=credentials, request=request
)
channel_creds = grpc.composite_channel_credentials(
    grpc.ssl_channel_credentials(),
    grpc.metadata_call_credentials(auth_metadata_plugin),
)

logger_provider = LoggerProvider()
_logs.set_logger_provider(logger_provider)

otlp_log_exporter = OTLPLogExporter(
    endpoint="telemetry.googleapis.com",
    credentials=channel_creds,
)
logger_provider.add_log_record_processor(
    BatchLogRecordProcessor(otlp_log_exporter)
)
```

### Mapping and Limitations

#### Conversion Logic & Log Entry Mapping

* **Log Entry Conversion:** `CloudLoggingExporter` translated `LogRecord` objects locally using the Python `google-cloud-logging` client library. The standard OTLP endpoint converts OTLP `LogRecord` payloads server-side according to the [OTLP LogRecord to LogEntry specification](https://docs.cloud.google.com/stackdriver/docs/reference/telemetry/otlp-log-record-to-log-entry).
* **Log Names & Resources:** The OTLP endpoint maps log names from resource attributes (e.g. `gcp.log_name` or defaults to `projects/<project>/logs/otel`).
* **Query Impact:** If your existing Cloud Logging log queries filter by specific `logName` values (such as python logger names mapped by `CloudLoggingExporter`), you may need to update your Cloud Logging query filters to match the OTLP log names and attributes.
* **GCP Monitored Resource Association:** Installing `opentelemetry-resourcedetector-gcp` ensures log records contain appropriate GCP resource attributes, allowing Cloud Logging to associate logs with standard monitored resources (GCE instances, GKE pods, Cloud Run services, etc.).
