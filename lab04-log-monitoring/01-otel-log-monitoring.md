# Lab 4 – Log Monitoring with the OpenTelemetry Collector

Send the **Spring PetClinic** log file to Dynatrace with the **OpenTelemetry Collector** `filelog` receiver, then search and analyze it with DQL.

## Ways to get logs into Dynatrace

| # | Method | How it works | Best for |
|---|--------|--------------|----------|
| 1 | **OneAgent** | Auto-discovers log files of monitored processes; extra files are added with log ingest rules | Hosts that already run OneAgent |
| 2 | **OpenTelemetry Collector** | `filelog` receiver tails files, `otlp` receiver accepts logs from SDKs/agents, exports over OTLP | Hosts without OneAgent, vendor-neutral pipelines |
| 3 | **OpenTelemetry SDK / Java agent** | The app sends its log records over OTLP (see [lab 5](../lab05-opentelemetry/02-instrumentation-opentelemtry.md)) | Logs linked to traces, no file needed |
| 4 | **AWS (Lambda, CloudWatch)** | CloudWatch Logs are streamed to Dynatrace with Amazon Data Firehose or the Dynatrace log forwarder | Serverless and managed AWS services |
| 5 | **Log ingest API** | `POST /api/v2/logs/ingest` with JSON or plain text | Scripts, quick tests, custom integrations |

This lab uses method 2. Methods 1 and 4 are summarized in [5. Alternatives](#5-alternatives).

## Architecture

```text
┌──────────────────────────────────────────────┐        ┌────────────────────┐
│ EC2 VM: petclinic host                       │  OTLP  │ Dynatrace SaaS     │
│                                              │  HTTP  │                    │
│  Spring PetClinic ──▶ /opt/petclinic.log     │        │  /api/v2/otlp      │
│                            │                 │──────▶ │  → OpenPipeline    │
│  Dynatrace OTel Collector  ▼                 │  :443  │  → Grail (logs)    │
│  filelog → resourcedetection → batch → export│        │                    │
└──────────────────────────────────────────────┘        └────────────────────┘
```

| # | Setting | Value |
|---|---------|-------|
| 1 | VM | 1 × `t2.large`, Amazon Linux (the PetClinic host) |
| 2 | Collector | Dynatrace OTel Collector `0.60.0` |
| 3 | Log file | `/opt/petclinic.log` |
| 4 | Service name | `petclinic` |
| 5 | Token scope | `openpipeline:logs:ingest` |

> The `filelog` receiver reads local files, so the Collector must run **on the same host** as the log file.

## Checklist

1. [Prerequisites](#1-prerequisites)
   - [ ] [1.1 Run PetClinic with a log file](#11-run-petclinic-with-a-log-file)
   - [ ] [1.2 Create a platform token](#12-create-a-platform-token)
2. [Install and configure the Collector](#2-install-and-configure-the-collector)
   - [ ] [2.1 Install the package](#21-install-the-package)
   - [ ] [2.2 Configure the filelog receiver](#22-configure-the-filelog-receiver)
   - [ ] [2.3 Set the endpoint and token](#23-set-the-endpoint-and-token)
   - [ ] [2.4 Give the Collector read access to the log](#24-give-the-collector-read-access-to-the-log)
   - [ ] [2.5 Start and check the Collector](#25-start-and-check-the-collector)
3. [Generate log data](#3-generate-log-data)
4. [Verify in Dynatrace](#4-verify-in-dynatrace)
   - [ ] [4.1 Logs app](#41-logs-app)
   - [ ] [4.2 DQL queries](#42-dql-queries)
5. [Alternatives](#5-alternatives)
   - [ ] [5.1 OneAgent log monitoring](#51-oneagent-log-monitoring)
   - [ ] [5.2 AWS Lambda and CloudWatch logs](#52-aws-lambda-and-cloudwatch-logs)
   - [ ] [5.3 Log ingest API (quick test)](#53-log-ingest-api-quick-test)
6. [Troubleshooting](#6-troubleshooting)
7. [Cleanup](#7-cleanup)
8. [References](#references)

---

## 1. Prerequisites

### 1.1 Run PetClinic with a log file

Use the PetClinic `systemd` service from [lab 1](../lab01-petclinic-ec2-apm-rum/02-install-petclinic.md). Make sure its output goes to a file:

```ini
StandardOutput=append:/opt/petclinic.log
StandardError=append:/opt/petclinic.log
```

```bash
systemctl daemon-reload
systemctl restart petclinic.service
tail -f /opt/petclinic.log
```

> If OneAgent is also installed on this host, it may already send this file. Either disable OneAgent log monitoring for the host or filter by `log.file.name` in [4.2](#42-dql-queries) so you can see which source sent each record.

### 1.2 Create a platform token

Create a platform token with the scope `openpipeline:logs:ingest`. Follow lab 5: [1.3 Create a platform token](../lab05-opentelemetry/02-instrumentation-opentelemtry.md#13-create-a-platform-token). You can reuse the lab 5 token if it has this scope.

---

## 2. Install and configure the Collector

### 2.1 Install the package

```bash
sudo su -

VERSION=0.60.0
wget https://github.com/Dynatrace/dynatrace-otel-collector/releases/download/v${VERSION}/dynatrace-otel-collector_${VERSION}_Linux_x86_64.rpm
rpm -ivh dynatrace-otel-collector_${VERSION}_Linux_x86_64.rpm
```

### 2.2 Configure the filelog receiver

```bash
cat > /etc/dynatrace-otel-collector/config.yaml <<'EOF'
receivers:
  filelog/petclinic:
    include: [/opt/petclinic.log]
    start_at: end                      # only new lines; use "beginning" to backfill
    include_file_name: true            # adds log.file.name
    include_file_path: true            # adds log.file.path
    multiline:
      # A new record starts with a timestamp; Java stack trace lines are appended to it
      line_start_pattern: '^\d{4}-\d{2}-\d{2}'
    operators:
      - type: regex_parser
        regex: '^(?P<timestamp>\S+)\s+(?P<severity>TRACE|DEBUG|INFO|WARN|ERROR|FATAL)\s'
        on_error: send                 # keep lines that don't match (banner, stack traces)
        timestamp:
          parse_from: attributes.timestamp
          layout_type: gotime
          layout: '2006-01-02T15:04:05.000Z07:00'
        severity:
          parse_from: attributes.severity

processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 256
    spike_limit_mib: 64

  resourcedetection:
    detectors: [system, ec2]           # adds host.name, cloud.* attributes
    timeout: 5s

  resource/petclinic:
    attributes:
      - key: service.name
        value: petclinic
        action: upsert
      - key: deployment.environment
        value: demo
        action: upsert

  batch: {}

exporters:
  otlphttp/dynatrace:
    endpoint: "${env:DT_ENDPOINT}"
    headers:
      Authorization: "Bearer ${env:DT_PLATFORM_TOKEN}"

service:
  pipelines:
    logs:
      receivers: [filelog/petclinic]
      processors: [memory_limiter, resourcedetection, resource/petclinic, batch]
      exporters: [otlphttp/dynatrace]
EOF
```

| # | Setting | Why |
|---|---------|-----|
| 1 | `start_at: end` | Avoids re-sending the whole file on first start. The Collector keeps no position across restarts without a `storage` extension, so `beginning` re-sends everything after each restart |
| 2 | `multiline.line_start_pattern` | Keeps a Java stack trace in **one** log record instead of one record per line |
| 3 | `regex_parser` | Sets the real event time and `loglevel` from the Spring Boot line, instead of the read time and `NONE` |
| 4 | `resourcedetection` | Adds `host.name` so the logs can be linked to the host |
| 5 | `resource/petclinic` | Adds `service.name` for filtering, the same as the OTel SDK would |

> Spring Boot's default line looks like `2026-11-20T10:15:30.123Z  INFO 1234 --- [main] o.s.s.petclinic...`. If your timestamp format differs, adjust `layout` or remove the `timestamp` block – Dynatrace then uses the read time.

### 2.3 Set the endpoint and token

```bash
systemctl edit dynatrace-otel-collector
```

Add these lines, then save and exit:

```ini
[Service]
Environment="DT_ENDPOINT=https://<environment-id>.live.dynatrace.com/api/v2/otlp"
Environment="DT_PLATFORM_TOKEN=<YOUR_DYNATRACE_PLATFORM_TOKEN>"
```

```bash
chmod 600 /etc/systemd/system/dynatrace-otel-collector.service.d/override.conf
```

> Use the `.live.dynatrace.com` domain, not `.apps.dynatrace.com`. The exporter adds `/v1/logs` itself.

### 2.4 Give the Collector read access to the log

The Collector service doesn't run as `root`, so it may not be able to read `/opt/petclinic.log`. Check the user and the file permissions:

```bash
systemctl show -p User --value dynatrace-otel-collector   # the service user
ls -l /opt/petclinic.log
```

If the file isn't readable by that user, make it world-readable:

```bash
chmod 644 /opt/petclinic.log
```

### 2.5 Start and check the Collector

```bash
systemctl daemon-reload
systemctl enable dynatrace-otel-collector
systemctl restart dynatrace-otel-collector
systemctl status dynatrace-otel-collector

journalctl -u dynatrace-otel-collector -f
```

Look for a line like `Started watching file  {"path": "/opt/petclinic.log"}`. A `permission denied` here means step [2.4](#24-give-the-collector-read-access-to-the-log) is needed.

---

## 3. Generate log data

1. Open `http://<VM_PUBLIC_IP>:8080` and browse **Find owners**, **Veterinarians** and add an owner and a pet.
2. Trigger an error to get a stack trace: click **Error** in the PetClinic menu (`/oups`).
3. Watch the file grow:

   ```bash
   tail -f /opt/petclinic.log
   ```

---

## 4. Verify in Dynatrace

### 4.1 Logs app

1. Open the **Logs** app.
2. Filter by `service.name = petclinic` or `log.file.name = petclinic.log`.
3. Open the `/oups` error record and check that the full stack trace is in **one** record and its status is `ERROR`.

### 4.2 DQL queries

Run these in a **Notebook** or the **Logs** app (DQL mode).

All PetClinic logs, newest first:

```sql
fetch logs, from: now() - 30m
| filter log.file.name == "petclinic.log"
| sort timestamp desc
| fields timestamp, loglevel, host.name, service.name, content
```

Count by log level:

```sql
fetch logs, from: now() - 1h
| filter service.name == "petclinic"
| summarize count(), by: { loglevel }
```

Errors over time:

```sql
fetch logs, from: now() - 2h
| filter service.name == "petclinic" and loglevel == "ERROR"
| makeTimeseries errors = count(), interval: 5m
```

Find the stack trace from the `/oups` page:

```sql
fetch logs, from: now() - 1h
| filter service.name == "petclinic"
| filter contains(content, "Expected: controller used to showcase what happens when an exception is thrown")
```

---

## 5. Alternatives

### 5.1 OneAgent log monitoring

With OneAgent on the host (see [lab 1](../lab01-petclinic-ec2-apm-rum/03-install-oneagent.md)), log monitoring is on by default and no Collector is needed.

1. OneAgent auto-discovers log files that monitored processes write to.
2. A file outside the defaults, such as `/opt/petclinic.log`, needs a **custom log source**: **Settings** → **Log Monitoring** → **Custom log sources** → add the path for the host or host group.
3. Choose what to store with **Log ingest rules**: **Settings** → **Log Monitoring** → **Log ingest rules** → add a rule that matches `log.source` = `/opt/petclinic.log`.
4. To test the "disable" practice task from [notes/04-practice-tasks.md](../notes/04-practice-tasks.md): install OneAgent with `--set-app-log-content-access=false`, or turn off **Log monitoring** for the host in its settings, and confirm no new records arrive.

| # | | OneAgent | OTel Collector |
|---|-|----------|----------------|
| 1 | Setup | Automatic discovery | Configure each file in YAML |
| 2 | Topology | Linked to host, process and service automatically | Linked through resource attributes (`host.name`, `service.name`) |
| 3 | Parsing | OpenPipeline processing in Dynatrace | Collector operators and/or OpenPipeline |
| 4 | Vendor lock-in | Dynatrace only | Any OTLP backend |

### 5.2 AWS Lambda and CloudWatch logs

| # | Option | How |
|---|--------|-----|
| 1 | **Amazon Data Firehose** (recommended) | Create a Firehose stream with Dynatrace as the destination, then add a CloudWatch Logs subscription filter on the Lambda log group (`/aws/lambda/<function>`) |
| 2 | **Dynatrace AWS log forwarder** | A CloudFormation-deployed Lambda that reads CloudWatch log groups and sends them to the log ingest API |
| 3 | **OneAgent Lambda layer / OTel Lambda layer** | Instruments the function itself; logs are sent with traces |

### 5.3 Log ingest API (quick test)

Send a test record without any agent. The token needs the `logs.ingest` scope (classic access token) or `openpipeline:logs:ingest` (platform token):

```bash
curl -X POST "https://<environment-id>.live.dynatrace.com/api/v2/logs/ingest" \
  -H "Authorization: Bearer <YOUR_DYNATRACE_PLATFORM_TOKEN>" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d '[{"content":"Hello from lab 4","log.source":"lab4-curl","loglevel":"INFO","service.name":"petclinic"}]'
```

A `204` response means the record was accepted. Find it with `fetch logs | filter log.source == "lab4-curl"`.

---

## 6. Troubleshooting

| # | Symptom | Likely cause |
|---|---------|--------------|
| 1 | `permission denied` in the Collector logs | The service user can't read the file – see [2.4](#24-give-the-collector-read-access-to-the-log) |
| 2 | No logs, no errors | `start_at: end` and nothing new was written – generate traffic, or temporarily use `beginning` |
| 3 | `401 Unauthorized` | Wrong token, or `Api-Token` used with a platform token (use `Bearer`) |
| 4 | `403 Forbidden` | Token missing `openpipeline:logs:ingest` |
| 5 | `404 Not Found` | `.apps.` domain instead of `.live.`, or missing `/api/v2/otlp` |
| 6 | Each stack trace line is a separate record | `multiline.line_start_pattern` doesn't match your timestamp format |
| 7 | `loglevel` is `NONE` | `regex_parser` didn't match – check the line format |
| 8 | Duplicate records | OneAgent and the Collector both send the same file |

---

## 7. Cleanup

```bash
systemctl stop dynatrace-otel-collector
rpm -e dynatrace-otel-collector
```

Delete the platform token if you no longer need it, and terminate the EC2 instance in AWS.

---

## References

| # | Link |
|---|------|
| 1 | [Dynatrace – Log Monitoring](https://docs.dynatrace.com/docs/analyze-explore-automate/logs) |
| 2 | [Dynatrace – Ingest logs with the OpenTelemetry Collector](https://docs.dynatrace.com/docs/ingest-from/opentelemetry/collector/use-cases/filelog) |
| 3 | [Dynatrace OTel Collector releases](https://github.com/Dynatrace/dynatrace-otel-collector/releases) |
| 4 | [OpenTelemetry `filelog` receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/filelogreceiver) |
| 5 | [Dynatrace – Log ingestion API](https://docs.dynatrace.com/docs/dynatrace-api/environment-api/log-monitoring-v2/post-ingest-logs) |
| 6 | [Dynatrace – Stream logs with Amazon Data Firehose](https://docs.dynatrace.com/docs/analyze-explore-automate/logs/lma-log-ingestion/lma-log-ingestion-via-api/lma-aws-firehose) |
| 7 | [DQL – `fetch logs`](https://docs.dynatrace.com/docs/platform/grail/dynatrace-query-language/commands/data-source-commands) |
