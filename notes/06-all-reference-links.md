# All Reference Links

Every external link used in the labs and notes, grouped by topic. Each lab guide also lists its own references.

## Contents

1. [Dynatrace platform](#1-dynatrace-platform)
2. [OneAgent and ActiveGate](#2-oneagent-and-activegate)
3. [Synthetic monitoring](#3-synthetic-monitoring)
4. [Real User Monitoring](#4-real-user-monitoring)
5. [OpenTelemetry](#5-opentelemetry)
6. [Sample applications](#6-sample-applications)
7. [Dummy APIs for testing](#7-dummy-apis-for-testing)
8. [General web and security](#8-general-web-and-security)

---

## 1. Dynatrace platform

| # | Link | Used in |
|---|------|---------|
| 1 | [Dynatrace API reference (Swagger)](https://api.dynatrace.com/spec/#/) | Lab 2 |
| 2 | [Dynatrace API – Tokens and authentication](https://docs.dynatrace.com/docs/dynatrace-api/basics/dynatrace-api-authentication) | Lab 5 |
| 3 | [Network zones](https://docs.dynatrace.com/docs/manage/network-zones-classic/network-zones-basic-info) | Knowledge notes |
| 4 | [Dynatrace pricing](https://www.dynatrace.com/pricing/) | Lab 2, license calculation |
| 5 | [Platform tokens](https://docs.dynatrace.com/docs/manage/identity-access-management/access-tokens-and-oauth-clients/platform-tokens) | Knowledge notes |

## 2. OneAgent and ActiveGate

| # | Link | Used in |
|---|------|---------|
| 1 | [Dynatrace OneAgent documentation](https://docs.dynatrace.com/docs/ingest-from/dynatrace-oneagent) | Lab 1 |
| 2 | [OneAgent monitoring modes](https://docs.dynatrace.com/docs/platform/oneagent/monitoring-modes/monitoring-modes) | Knowledge notes |
| 3 | [Set up Dynatrace OneAgent as a Docker container](https://docs.dynatrace.com/docs/ingest-from/setup-on-container-platforms/docker/set-up-dynatrace-oneagent-as-docker-container) | Lab 2 |
| 4 | [Dynatrace ActiveGate documentation](https://docs.dynatrace.com/docs/ingest-from/dynatrace-activegate) | Lab 1 |

## 3. Synthetic monitoring

| # | Link | Used in |
|---|------|---------|
| 1 | [Create a private Synthetic location](https://docs.dynatrace.com/docs/observe/digital-experience-classic/synthetic-monitoring-classic/private-synthetic-locations/create-a-private-synthetic-location) | Lab 2 |
| 2 | [Manage private Synthetic locations – Browser autoupdate](https://docs.dynatrace.com/docs/observe/digital-experience-classic/synthetic-monitoring-classic/private-synthetic-locations/manage-private-synthetic-locations) | Lab 2 |
| 3 | [Configure HTTP monitors in Classic](https://docs.dynatrace.com/docs/observe/digital-experience-classic/synthetic-monitoring-classic/http-monitors-classic/configure-http-monitors-classic) | Lab 2 |
| 4 | [Pre- and post-execution scripts for HTTP monitors](https://docs.dynatrace.com/docs/observe/digital-experience/synthetic-monitoring/http-monitors-classic/pre-and-post-scripting-for-http-monitors-classic) | Lab 2 |
| 5 | [Synthetic monitor authentication](https://docs.dynatrace.com/docs/observe/digital-experience-classic/synthetic-monitoring-classic/general-information/synthetic-authentication) | Lab 2 |
| 6 | [Network availability monitoring](https://docs.dynatrace.com/docs/observe/digital-experience/synthetic-monitoring/network-availability-monitors/network-availability-monitoring) | Lab 2 |
| 7 | [Configure a NAM monitor](https://docs.dynatrace.com/docs/observe/digital-experience/synthetic-monitoring/network-availability-monitors/create-a-nam-monitor) | Lab 2 |

## 4. Real User Monitoring

| # | Link | Used in |
|---|------|---------|
| 1 | [Manual instrumentation (Android, OneAgent SDK)](https://docs.dynatrace.com/docs/observe/digital-experience-classic/rum-classic/mobile-applications/instrument-android-app/instrumentation-via-oneagent-sdk/manual-instrumentation) | Lab 1 |

## 5. OpenTelemetry

| # | Link | Used in |
|---|------|---------|
| 1 | [OpenTelemetry documentation](https://opentelemetry.io/docs/) | Lab 5 |
| 2 | [OpenTelemetry signals](https://opentelemetry.io/docs/concepts/signals/) | Lab 5 |
| 3 | [OpenTelemetry specification status](https://opentelemetry.io/docs/specs/status/) | Lab 5 |
| 4 | [OTLP specification](https://opentelemetry.io/docs/specs/otlp/) | Lab 5 |
| 5 | [OpenTelemetry Profiles enters public alpha](https://opentelemetry.io/blog/2026/profiles-alpha/) | Lab 5 |
| 6 | [Dynatrace OTLP API endpoints](https://docs.dynatrace.com/docs/ingest-from/opentelemetry/otlp-api) | Lab 5 |
| 7 | [Transform OTLP gRPC to HTTP with the OTel Collector](https://docs.dynatrace.com/docs/ingest-from/opentelemetry/collector/use-cases/grpc) | Lab 5 |

## 6. Sample applications

| # | Link | Used in |
|---|------|---------|
| 1 | [Spring PetClinic](https://github.com/spring-projects/spring-petclinic) | Lab 1 |
| 2 | [easyTravel-Docker](https://github.com/Dynatrace/easyTravel-Docker) | Lab 2 |
| 3 | [EasyTrade](https://github.com/Dynatrace/easytrade) | Practice tasks |
| 4 | [Grafana OSS on Docker Hub](https://hub.docker.com/r/grafana/grafana-oss) | Lab 2 |

## 7. Dummy APIs for testing

Free public APIs for practicing HTTP monitors and API monitoring without your own backend.

| # | API | What it offers | Link |
|---|-----|----------------|------|
| 1 | **DummyJSON** | Fake REST data plus a login flow with access tokens (demo user `emilys` / `emilyspass`) | [Website](https://dummyjson.com/) · [Docs](https://dummyjson.com/docs) · [Auth docs](https://dummyjson.com/docs/auth) · [GitHub](https://github.com/Ovi/DummyJSON) |
| 2 | **JSONPlaceholder** | Fake REST data for `GET`, `POST`, `PUT`, `PATCH` and `DELETE`; no login | [Website](https://jsonplaceholder.typicode.com/) |

## 8. General web and security

| # | Link | Used in |
|---|------|---------|
| 1 | [HTTP response status codes (MDN)](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status) | Lab 2 |
| 2 | [Kerberos authentication overview (Microsoft)](https://learn.microsoft.com/en-us/windows-server/security/kerberos/kerberos-authentication-overview) | Lab 2 |
