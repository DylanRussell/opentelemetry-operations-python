# Migration Guide

This guide provides instructions on how to migrate from the custom exporters in this repository to the standard OpenTelemetry OTLP exporters.

## Overview

Google Cloud supports native OTLP (OpenTelemetry Protocol) ingestion for Cloud Trace and Cloud Monitoring via the [Telemetry API](https://docs.cloud.google.com/stackdriver/docs/reference/telemetry/overview). This allows you to use standard OpenTelemetry OTLP exporters for sending telemetry data to Google Cloud.

## Deprecation Notice

All exporters in this repository are deprecated. Please migrate to the standard OTLP exporters using standard OpenTelemetry libraries. This repository will be
archived soon.

---

## Migrate from OpenTelemetry Google Cloud Trace Exporter (`CloudTraceSpanExporter`) to OTLP Exporter

To migrate from `opentelemetry-exporter-gcp-trace` (`CloudTraceSpanExporter`) to the standard OpenTelemetry OTLP exporter, follow these steps:

### 1. Add Dependencies

Install the standard OpenTelemetry OTLP exporter and GCP authentication dependencies:

```bash
pip install opentelemetry-exporter-otlp-proto-grpc google-auth grpcio requests
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
| `project_id` | Use resource attribute: `gcp.project_id` | Set via `OTEL_RESOURCE_ATTRIBUTES="gcp.project_id=your-project-id"`. |
| `client` | N/A | Pre-configured `TraceServiceClient` is replaced by HTTP/gRPC OTLP exporter configuration. |
| `resource_regex` | Resource Attributes / Processors | Use standard OpenTelemetry Resource attributes instead. |

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

Transitioning to the standard OTLP exporter is recommended for the following reasons:
* **Standardization:** Aligns your application with the industry-standard OpenTelemetry Protocol (OTLP).
* **Google Managed Prometheus (GMP):** Standard OTLP metrics are ingested into Google Managed Prometheus, offering robust, scalable, and cost-effective monitoring.
* **Future-proofing:** The legacy Google Cloud exporters in this repository are deprecated.

### Strategy & Steps

#### 1. Add Dependencies

```bash
pip install opentelemetry-exporter-otlp-proto-grpc google-auth grpcio
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

---

## Migrate from OpenTelemetry Google Cloud Logging Exporter (`CloudLoggingExporter`) to OTLP Exporter

The OTLP `LogRecord` to Cloud Logging `LogEntry` conversion logic in the `CloudLoggingExporter` is similar but not identical to the Google OTLP endpoint conversion logic. 

The Google OTLP endpoint `LogRecord` to Cloud Logging `LogEntry` conversion logic is described here: https://docs.cloud.google.com/stackdriver/docs/reference/telemetry/otlp-log-record-to-log-entry.

The conversion logic in the `CloudLoggingExporter` can be found here: https://github.com/GoogleCloudPlatform/opentelemetry-operations-python/blob/main/opentelemetry-exporter-gcp-logging/src/opentelemetry/exporter/cloud_logging/__init__.py.

You can send the same OTLP `LogRecord` to the Google OTLP endpoint and to Cloud Logging via the `CloudLoggingExporter` to see exactly how the conversion logic diverges for a particular log if at all.

To migrate from `opentelemetry-exporter-gcp-logging` (`CloudLoggingExporter`) to the standard OpenTelemetry OTLP log exporter, follow these steps:

### 1. Add Dependencies

Install the standard OpenTelemetry OTLP exporter package:

```bash
pip install opentelemetry-exporter-otlp-proto-grpc google-auth grpcio
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
