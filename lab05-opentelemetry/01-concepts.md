# Lab 5 – OpenTelemetry Concepts

## What it is

**OpenTelemetry (OTel)** is an open-source, vendor-neutral standard for generating, collecting and exporting telemetry data. It is a CNCF project. You instrument your app once with OpenTelemetry and can then send the data to any backend that supports it, such as Dynatrace, without changing the code.

> OpenTelemetry only **produces and transports** telemetry. It does not store or analyze it. That is the job of a backend such as Dynatrace.

## Contents

1. [Components](#1-components)
   - 1.1 [API and SDK](#11-api-and-sdk)
   - 1.2 [Instrumentation](#12-instrumentation)
   - 1.3 [Collector](#13-collector)
   - 1.4 [OTLP](#14-otlp)
2. [Signals](#2-signals)
3. [OpenTelemetry in Dynatrace](#3-opentelemetry-in-dynatrace)
   - 3.1 [OTLP endpoint](#31-otlp-endpoint)
   - 3.2 [Token scopes](#32-token-scopes)
   - 3.3 [Collector configuration](#33-collector-configuration)
   - 3.4 [OneAgent vs OpenTelemetry](#34-oneagent-vs-opentelemetry)
4. [References](#references)

---

## 1. Components

```text
┌───────────────────────────┐        ┌───────────────────────────────┐        ┌──────────────────┐
│ Application               │  OTLP  │ OpenTelemetry Collector       │  OTLP  │ Backend          │
│                           │──────▶ │                               │──────▶ │ (e.g. Dynatrace) │
│  OTel API  (what to send) │        │  receivers → processors →     │        │                  │
│  OTel SDK  (how to send)  │        │  exporters                    │        │  store, analyze  │
└───────────────────────────┘        └───────────────────────────────┘        └──────────────────┘
```

| # | Component | What it does |
|---|-----------|--------------|
| 1 | **API** | Interfaces the code calls to create spans, metrics and logs |
| 2 | **SDK** | The implementation of the API: sampling, batching and exporting the data |
| 3 | **Instrumentation** | Code that uses the API, either added automatically (zero-code) or written by you |
| 4 | **Collector** | A separate process that receives, processes and exports telemetry |
| 5 | **OTLP** | The OpenTelemetry Protocol used to send data between the parts |

### 1.1 API and SDK

- **API:** What the application and libraries code against. Without an SDK, API calls do nothing, so a library can use the API safely without forcing a setup on its users.
- **SDK:** Plugged in by the application owner. It decides sampling, adds resource attributes (such as `service.name`) and exports the data, for example over OTLP.

### 1.2 Instrumentation

| # | Type | How it works | Example |
|---|------|--------------|---------|
| 1 | **Zero-code** (automatic) | An agent or library instruments common frameworks without code changes | Java agent: `java -javaagent:opentelemetry-javaagent.jar -jar app.jar` |
| 2 | **Code-based** (manual) | You call the API in your code to add custom spans, metrics and attributes | Create a span around a business transaction |

### 1.3 Collector

The Collector is optional but recommended. It decouples the app from the backend, so you can change where data goes without touching the app.

| # | Part | What it does | Example |
|---|------|--------------|---------|
| 1 | **Receivers** | Get data in | `otlp`, `prometheus`, `filelog`, `hostmetrics` |
| 2 | **Processors** | Change data in flight | `batch`, `memory_limiter`, `filter`, `transform`, `attributes` |
| 3 | **Exporters** | Send data out | `otlp_http` (to Dynatrace), `debug` |
| 4 | **Pipelines** | Connect receivers, processors and exporters for each signal | `traces`, `metrics`, `logs` |

Deployment patterns:

| # | Pattern | Where it runs |
|---|---------|---------------|
| 1 | **Agent** | Next to the app: on each host, or as a sidecar or DaemonSet in Kubernetes |
| 2 | **Gateway** | A central Collector cluster that receives data from many agents or apps |

Distributions:

| # | Distribution | Contents |
|---|--------------|----------|
| 1 | **Core** | A minimal set of components maintained by the OpenTelemetry project |
| 2 | **Contrib** | Core plus many community components |
| 3 | **Dynatrace Collector** | A Dynatrace-supported build with the components tested for Dynatrace |

### 1.4 OTLP

The **OpenTelemetry Protocol (OTLP)** defines how telemetry is encoded and sent. It runs over **gRPC** (port `4317`) or **HTTP** (port `4318`, protobuf or JSON). OTLP is stable for traces, metrics and logs.

---

## 2. Signals

A **signal** is a type of telemetry. The common observability acronym is **MELT**: Metrics, Events, Logs and Traces. In OpenTelemetry, events are a kind of log record, so OTel has three main signals:

| # | Signal | Answers | Example | Status |
|---|--------|---------|---------|--------|
| 1 | **Traces** | Where did the request go, and where was time spent? | A request across frontend → API → database, as **spans** | Stable |
| 2 | **Metrics** | How much, how often, how fast, over time? | Request count, latency histogram, CPU usage | Stable |
| 3 | **Logs** | What happened at a point in time? | An error message with the trace ID attached | Stable |
| 4 | **Profiles** | Which code uses the CPU or memory? | A CPU flame graph | Alpha (since March 2026) |

Related concepts:

- **Span:** One operation in a trace, with a start time, duration, attributes and status
- **Context propagation:** Passes the trace ID between services (W3C `traceparent` header), so spans join into one trace
- **Baggage:** Key-value pairs that travel with the context across services
- **Resource:** Attributes that describe the source of the data, such as `service.name`, `host.name` or `k8s.pod.name`
- **Semantic conventions:** Standard attribute names, such as `http.request.method`, so all tools read the data the same way

---

## 3. OpenTelemetry in Dynatrace

Dynatrace ingests OpenTelemetry data natively through its **OTLP API**. You can send data directly from the SDK or through a Collector.

### 3.1 OTLP endpoint

| # | Target | Base URL |
|---|--------|----------|
| 1 | SaaS | `https://<environment-id>.live.dynatrace.com/api/v2/otlp` |
| 2 | ActiveGate | `https://<activegate-host>:9999/e/<environment-id>/api/v2/otlp` |

The exporter adds `/v1/traces`, `/v1/metrics` or `/v1/logs` for each signal.

> - Use the `.live.dynatrace.com` domain, **not** `.apps.dynatrace.com`.
> - The Dynatrace OTLP API accepts **OTLP over HTTP** (protobuf). To send gRPC data, put a Collector in between to convert it.

### 3.2 Token scopes

Send the token in the `Authorization` header. One token is set per exporter, so give it the scopes for every signal you export.

| # | Signal | Platform token scope (recommended) | Classic access token scope |
|---|--------|------------------------------------|----------------------------|
| 1 | Traces | `openpipeline:traces:ingest` | `openTelemetryTrace.ingest` |
| 2 | Metrics | `openpipeline:metrics:ingest` | `metrics.ingest` |
| 3 | Logs | `openpipeline:logs:ingest` | `logs.ingest` |

| Token type | Header |
|------------|--------|
| Platform token | `Authorization: Bearer <token>` |
| Classic access token | `Authorization: Api-Token <token>` |

### 3.3 Collector configuration

A minimal Collector pipeline that receives OTLP from apps and exports to Dynatrace. Set the token as an environment variable, never in the file:

```bash
export DT_ENDPOINT='https://<environment-id>.live.dynatrace.com/api/v2/otlp'
export DT_PLATFORM_TOKEN='<YOUR_DYNATRACE_PLATFORM_TOKEN>'
```

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch: {}

exporters:
  otlp_http:
    endpoint: ${env:DT_ENDPOINT}
    headers:
      Authorization: "Bearer ${env:DT_PLATFORM_TOKEN}"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp_http]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp_http]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp_http]
```

> The Collector receives gRPC or HTTP from the apps and always exports OTLP over HTTP to Dynatrace.

### 3.4 OneAgent vs OpenTelemetry

| # | | OneAgent | OpenTelemetry |
|---|---|----------|---------------|
| 1 | Setup | Install once per host; instruments everything automatically | Add the SDK or agent to each app, plus a Collector |
| 2 | Coverage | Full stack: hosts, processes, services, RUM, topology (Smartscape) | What you instrument: traces, metrics, logs |
| 3 | Vendor | Dynatrace only | Vendor-neutral, works with any backend |
| 4 | Best for | Hosts and apps where you can install OneAgent | Serverless, managed services, unsupported languages, or teams that standardize on OTel |

> They work together: OneAgent can also capture OpenTelemetry spans that an app creates, and add them to its own traces.

---

## References

- [OpenTelemetry documentation](https://opentelemetry.io/docs/)
- [OpenTelemetry signals](https://opentelemetry.io/docs/concepts/signals/)
- [OpenTelemetry specification status](https://opentelemetry.io/docs/specs/status/)
- [OTLP specification](https://opentelemetry.io/docs/specs/otlp/)
- [OpenTelemetry Profiles enters public alpha](https://opentelemetry.io/blog/2026/profiles-alpha/)
- [Dynatrace OTLP API endpoints](https://docs.dynatrace.com/docs/ingest-from/opentelemetry/otlp-api)
- [Transform OTLP gRPC to HTTP with the OTel Collector (Dynatrace)](https://docs.dynatrace.com/docs/ingest-from/opentelemetry/collector/use-cases/grpc)
- [Dynatrace API – Tokens and authentication](https://docs.dynatrace.com/docs/dynatrace-api/basics/dynatrace-api-authentication)
