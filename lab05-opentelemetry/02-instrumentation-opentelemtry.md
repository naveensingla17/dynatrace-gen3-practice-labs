# Lab 5 – Instrument PetClinic with OpenTelemetry

Instrument the **Spring PetClinic** app with the **OpenTelemetry Java agent** and send its traces, metrics and logs through an **OpenTelemetry Collector** to Dynatrace. No OneAgent is used in this lab.

For the concepts (API, SDK, Collector, OTLP, signals), see [01-concepts.md](01-concepts.md).

## Architecture

```text
┌──────────────────────────────┐        ┌──────────────────────────────┐        ┌────────────────────┐
│ EC2 VM-2: petclinic host     │  OTLP  │ EC2 VM-1: collector host     │  OTLP  │ Dynatrace SaaS     │
│                              │  HTTP  │                              │  HTTP  │                    │
│  Spring PetClinic :8080      │──────▶ │  Dynatrace OTel Collector    │──────▶ │  /api/v2/otlp      │
│  OTel Java agent             │  :4318 │  receivers → batch → export  │  :443  │                    │
└──────────────────────────────┘        └──────────────────────────────┘        └────────────────────┘
```

| # | Setting | Value |
|---|---------|-------|
| 1 | VMs | 2 × `t2.large`, Amazon Linux |
| 2 | Collector | Dynatrace OTel Collector `0.60.0` |
| 3 | Java agent | OpenTelemetry Java agent `v2.33.1` |
| 4 | Service name | `petclinic-otel` |
| 5 | Ports | `4317` (OTLP gRPC), `4318` (OTLP HTTP) on the Collector VM |

## Checklist

1. [Basic preparation and setup](#1-basic-preparation-and-setup)
   - [ ] [1.1 Create the VMs](#11-create-the-vms)
   - [ ] [1.2 Get the Dynatrace endpoint](#12-get-the-dynatrace-endpoint)
   - [x] [1.3 Create a platform token](#13-create-a-platform-token)
2. [Install the OpenTelemetry Collector (VM-1)](#2-install-the-opentelemetry-collector-vm-1)
   - [ ] [2.1 Install the package](#21-install-the-package)
   - [ ] [2.2 Configure the Collector](#22-configure-the-collector)
   - [ ] [2.3 Set the endpoint and token](#23-set-the-endpoint-and-token)
   - [ ] [2.4 Start and check the Collector](#24-start-and-check-the-collector)
3. [Instrument PetClinic (VM-2)](#3-instrument-petclinic-vm-2)
   - [x] [3.1 Build PetClinic](#31-build-petclinic)
   - [x] [3.2 Download the Java agent](#32-download-the-java-agent)
   - [ ] [3.3 Run PetClinic with the agent](#33-run-petclinic-with-the-agent)
4. [Verify in Dynatrace](#4-verify-in-dynatrace)
5. [Alternatives](#5-alternatives)
   - [x] [5.1 Send directly to Dynatrace (no Collector)](#51-send-directly-to-dynatrace-no-collector)
     - [x] [5.1.1 Test from the shell](#511-test-from-the-shell)
     - [x] [5.1.2 Run as a service](#512-run-as-a-service)
   - [5.2 Manual instrumentation with the SDK](#52-manual-instrumentation-with-the-sdk)
6. [Cleanup](#6-cleanup)
7. [References](#references)

---

## 1. Basic preparation and setup

### 1.1 Create the VMs

Create two `t2.large` Amazon Linux VMs:

| # | VM | Purpose |
|---|----|---------|
| 1 | **VM-1** | OpenTelemetry Collector |
| 2 | **VM-2** | Spring PetClinic with the OpenTelemetry Java agent |

In the security group, allow:

| # | Port | From | Why |
|---|------|------|-----|
| 1 | `4317`, `4318` | VM-2 | The Java agent sends OTLP to the Collector |
| 2 | `8080` | Your IP | Open PetClinic in a browser |
| 3 | `22` | Your IP | SSH |

### 1.2 Get the Dynatrace endpoint

The OTLP base URL must end in `/api/v2/otlp` and use the `.live` domain:

```text
https://<environment-id>.live.dynatrace.com/api/v2/otlp
```

> Don't use the `.apps.dynatrace.com` domain here. The exporter adds `/v1/traces`, `/v1/metrics` and `/v1/logs` itself.

### 1.3 Create a platform token

1. Go to **My platform tokens**: click your **profile** (top right) → **Platform tokens** → **Generate token**, or open <https://myaccount.dynatrace.com/platformTokens>
2. Create a token with a scope for each signal you send:

   | # | Signal | Scope |
   |---|--------|-------|
   | 1 | Traces | `openpipeline:traces:ingest` |
   | 2 | Metrics | `openpipeline:metrics:ingest` |
   | 3 | Logs | `openpipeline:logs:ingest` |

3. Copy the token. It's sent as the header `Authorization: Bearer <your-platform-token>`.

> Never commit the token. Keep it in an environment variable or a root-only file on the host.

---

## 2. Install the OpenTelemetry Collector (VM-1)

### 2.1 Install the package

```bash
sudo su -

VERSION=0.60.0
wget https://github.com/Dynatrace/dynatrace-otel-collector/releases/download/v${VERSION}/dynatrace-otel-collector_${VERSION}_Linux_x86_64.rpm
rpm -ivh dynatrace-otel-collector_${VERSION}_Linux_x86_64.rpm
```

The package installs a `systemd` service named `dynatrace-otel-collector`.

### 2.2 Configure the Collector

Write the config to the default path, `/etc/dynatrace-otel-collector/config.yaml`. The service may fail to start if this file is missing.

```bash
cat > /etc/dynatrace-otel-collector/config.yaml <<'EOF'
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
    endpoint: "${env:DT_ENDPOINT}"
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
EOF
```

### 2.3 Set the endpoint and token

Pass them to the service as environment variables with a `systemd` override, so they're not in the config file:

```bash
systemctl edit dynatrace-otel-collector
```

Add these lines, then save and exit:

```ini
[Service]
Environment="DT_ENDPOINT=https://<environment-id>.live.dynatrace.com/api/v2/otlp"
Environment="DT_PLATFORM_TOKEN=<YOUR_DYNATRACE_PLATFORM_TOKEN>"
```

### 2.4 Start and check the Collector

```bash
systemctl daemon-reload
systemctl enable dynatrace-otel-collector
systemctl restart dynatrace-otel-collector
systemctl status dynatrace-otel-collector

journalctl -u dynatrace-otel-collector -f     # watch the logs
ss -tlnp | grep -E '4317|4318'                # check the ports are listening
```

---

## 3. Instrument PetClinic (VM-2)

### 3.1 Build PetClinic

Install Java 17, Git and Maven, then clone and build PetClinic. Follow lab 1: [2. Build and test PetClinic](../lab01-petclinic-ec2-apm-rum/02-install-petclinic.md#2-build-and-test-petclinic).

### 3.2 Download the Java agent

The agent instruments Spring, Tomcat, JDBC and many other libraries automatically, with no code changes (zero-code instrumentation).

```bash
mkdir -p /opt/otel && cd /opt/otel
wget https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/download/v2.33.1/opentelemetry-javaagent.jar
```

### 3.3 Run PetClinic with the agent

Point the agent at the Collector on VM-1. Use VM-1's **private IP**.

```bash
export OTEL_SERVICE_NAME=petclinic-otel
export OTEL_RESOURCE_ATTRIBUTES="service.version=1.0.0,deployment.environment=demo"
export OTEL_EXPORTER_OTLP_ENDPOINT=http://<VM-1_PRIVATE_IP>:4318
export OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
export OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE=delta

cd ~/spring-petclinic
java -javaagent:/opt/otel/opentelemetry-javaagent.jar -jar target/*.jar
```

| # | Variable | Why |
|---|----------|-----|
| 1 | `OTEL_SERVICE_NAME` | The service name shown in Dynatrace |
| 2 | `OTEL_RESOURCE_ATTRIBUTES` | Extra attributes on all data, such as version and environment |
| 3 | `OTEL_EXPORTER_OTLP_ENDPOINT` | Where to send the data: the Collector |
| 4 | `OTEL_EXPORTER_OTLP_PROTOCOL` | `http/protobuf`, to match the Collector's port `4318` |
| 5 | `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` | `delta`, which Dynatrace requires for metrics |

> To run it as a service, see [5.1.2 Run as a service](#512-run-as-a-service). Use the Collector endpoint there instead of the Dynatrace one.

Open `http://<VM-2_PUBLIC_IP>:8080`, then add owners and pets and browse a few pages to generate traffic.

---

## 4. Verify in Dynatrace

| # | What | Where |
|---|------|-------|
| 1 | Traces | **Distributed Tracing** → filter by `service.name = petclinic-otel` |
| 2 | Service | **Services** → `petclinic-otel` |
| 3 | Logs | **Logs** → filter by `service.name = petclinic-otel` |
| 4 | Metrics | **Notebooks** → query the metrics from `petclinic-otel` |

If nothing appears:

1. Check the Collector logs on VM-1: `journalctl -u dynatrace-otel-collector`. A `401` or `403` from the exporter means the token or its scopes are wrong.
2. Check that VM-2 can reach the Collector: `curl -v http://<VM-1_PRIVATE_IP>:4318`. A timeout means the security group is blocking the port.

---

## 5. Alternatives

### 5.1 Send directly to Dynatrace (no Collector)

For a quick test, the agent can export straight to Dynatrace, without VM-1. The agent then needs the Dynatrace endpoint and the token itself.

#### 5.1.1 Test from the shell

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=https://<environment-id>.live.dynatrace.com/api/v2/otlp
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer <YOUR_DYNATRACE_PLATFORM_TOKEN>"
export OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
export OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE=delta
export OTEL_RESOURCE_ATTRIBUTES="service.name=petclinic,service.version=1.0.1"

cd /root/spring-petclinic
java -javaagent:/opt/otel/opentelemetry-javaagent.jar -jar target/spring-petclinic-4.0.0-SNAPSHOT.jar
```

> - `http/protobuf` is already the Java agent's default, but setting it makes the config explicit.
> - A **platform token** uses `Authorization=Bearer <token>`. Only a **classic access token** uses `Authorization=Api-Token <token>`.

#### 5.1.2 Run as a service

Variables exported in your shell are **not** passed to a `systemd` service. Put them in the unit as `Environment=` lines instead.

1. Edit the PetClinic service from lab 1:

   ```bash
   vi /etc/systemd/system/petclinic.service
   ```

2. Replace its contents with:

   ```ini
   [Unit]
   Description=Spring PetClinic Application (OpenTelemetry)
   After=network.target

   [Service]
   Type=simple
   User=root
   Group=root
   WorkingDirectory=/root/spring-petclinic

   Environment="OTEL_EXPORTER_OTLP_ENDPOINT=https://<environment-id>.live.dynatrace.com/api/v2/otlp"
   Environment="OTEL_EXPORTER_OTLP_HEADERS=Authorization=Bearer <YOUR_DYNATRACE_PLATFORM_TOKEN>"
   Environment="OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf"
   Environment="OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE=delta"
   Environment="OTEL_RESOURCE_ATTRIBUTES=service.name=petclinic,service.version=1.0.1"

   ExecStart=/usr/bin/java -javaagent:/opt/otel/opentelemetry-javaagent.jar -jar /root/spring-petclinic/target/spring-petclinic-4.0.0-SNAPSHOT.jar
   StandardOutput=append:/opt/petclinic.log
   StandardError=append:/opt/petclinic.log

   SuccessExitStatus=143
   Restart=on-failure
   RestartSec=10

   [Install]
   WantedBy=multi-user.target
   ```

3. Reload and restart the service:

   ```bash
   chmod 600 /etc/systemd/system/petclinic.service    # the file contains the token
   systemctl daemon-reload
   systemctl restart petclinic.service
   systemctl status petclinic.service
   tail -f /opt/petclinic.log
   ```

> - Don't add `> /opt/petclinic.log 2>&1 &` to `ExecStart`. `systemd` doesn't run it through a shell, so the redirect and `&` are passed to Java as arguments. Use `StandardOutput=` and `StandardError=` instead, and let `systemd` run the process in the background.
> - To send through the Collector instead, set `OTEL_EXPORTER_OTLP_ENDPOINT=http://<VM-1_PRIVATE_IP>:4318` and remove the `OTEL_EXPORTER_OTLP_HEADERS` line.

> The Collector is still recommended: the token stays on one host, and you can change the backend, filter or transform data without touching the app.

### 5.2 Manual instrumentation with the SDK

Instead of the agent, you can add the OpenTelemetry SDK to the code, as in the Dynatrace [Java manual instrumentation walkthrough](https://docs.dynatrace.com/docs/ingest-from/opentelemetry/walkthroughs/java-manual):

| # | Approach | Dependencies | Endpoint and token |
|---|----------|--------------|--------------------|
| 1 | **With SDK auto-configuration** | `opentelemetry-sdk-extension-autoconfigure`, `opentelemetry-exporter-otlp`, `opentelemetry-semconv` | The same `OTEL_*` environment variables as above |
| 2 | **Without auto-configuration** | `opentelemetry-api`, `opentelemetry-sdk`, `opentelemetry-exporter-otlp`, `opentelemetry-semconv`, `opentelemetry-log4j-appender-2.17` | Set in code, with an exporter for each signal |

Manual instrumentation lets you create your own spans, metrics and log records for business logic that the agent can't see.

---

## 6. Cleanup

On VM-1:

```bash
systemctl stop dynatrace-otel-collector
rpm -e dynatrace-otel-collector
```

On VM-2, stop PetClinic. Then delete the platform token if you no longer need it, and terminate both EC2 instances in AWS.

---

## References

- [Manually instrument your Java application with OpenTelemetry (Dynatrace)](https://docs.dynatrace.com/docs/ingest-from/opentelemetry/walkthroughs/java-manual)
- [Deploy the Dynatrace OTel Collector](https://docs.dynatrace.com/docs/ingest-from/opentelemetry/collector/deployment)
- [Dynatrace OTLP API endpoints](https://docs.dynatrace.com/docs/ingest-from/opentelemetry/otlp-api)
- [Dynatrace OTel Collector releases](https://github.com/Dynatrace/dynatrace-otel-collector/releases)
- [OpenTelemetry Java instrumentation releases](https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases)
- [OpenTelemetry Java agent configuration](https://opentelemetry.io/docs/zero-code/java/agent/configuration/)
