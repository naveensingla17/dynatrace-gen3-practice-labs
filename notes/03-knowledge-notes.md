# Dynatrace Knowledge Notes

Short notes on the core Dynatrace concepts used across the labs.

## Contents

1. [OneAgent monitoring modes](#1-oneagent-monitoring-modes)
2. [Prerequisites for OneAgent](#2-prerequisites-for-oneagent)
   - 2.1 [Environment ActiveGate](#21-environment-activegate)
   - 2.2 [Network Zone](#22-network-zone)
   - 2.3 [Platform Token](#23-platform-token)
3. [Host group](#3-host-group)
4. [Real User Monitoring (RUM)](#4-real-user-monitoring-rum)
5. [References](#references)

---

## 1. OneAgent monitoring modes

| # | Mode | What it monitors | What it leaves out |
|---|------|------------------|--------------------|
| 1 | **Full-Stack** | Everything: infrastructure plus application monitoring, distributed tracing, code-level profiling, automatic injection into all supported processes | – |
| 2 | **Infrastructure** | Host health, processes, detailed disk, network and memory analysis, topology; runtime metrics for supported languages | Distributed tracing and code-level (application) monitoring |
| 3 | **Discovery** | Basic host health only: CPU, memory, filesystem, OS services, topology discovery | Process details, detailed disk, network and memory analysis, tracing. Needs the Dynatrace Platform Subscription (DPS) |

> Lab 1 uses **Full-Stack**, because APM and RUM need tracing and injection into the app process.

## 2. Prerequisites for OneAgent

### 2.1 Environment ActiveGate

A secure proxy between OneAgents and the Dynatrace tenant. It is optional, but most real setups use one.

| # | Purpose | What it does |
|---|---------|--------------|
| 1 | **Route OneAgent traffic** | Collects, compresses and encrypts OneAgent traffic before sending it to Dynatrace, so hosts don't need direct internet access. Several ActiveGates give failover and spread the load |
| 2 | **Extensions** | Runs remote extensions, for example for databases, SNMP, WMI or Prometheus |
| 3 | **Cloud and remote monitoring** | Polls APIs of AWS, Azure, GCP, Kubernetes, VMware and others |
| 4 | **Synthetic monitoring** | Runs monitors from **private Synthetic locations**, to test internal and public URLs. This uses a separate **Synthetic-enabled** ActiveGate (see Lab 2) |

### 2.2 Network Zone

- A **configuration setting**, not a component you install
- Tells each OneAgent which ActiveGates to send its traffic to, so traffic stays in the right region or network (for example, `petclinic` in Lab 1)
- Without network zones, any OneAgent can connect to any ActiveGate, which can cause cross-region traffic
- **Alternative zones:** used when no ActiveGate in the primary zone is available
- **Fallback mode:** what happens when neither the primary nor the alternative zones have an ActiveGate:

| # | Fallback mode | Behavior |
|---|---------------|----------|
| 1 | **Any ActiveGate** (default) | Use an ActiveGate from any zone |
| 2 | **Only default zone** | Use only ActiveGates in the default zone |
| 3 | **None** | Don't send traffic to any other zone |

### 2.3 Platform Token

- A long-lived token that gives scripts, tools and installers **programmatic access** to Dynatrace, for example to install an ActiveGate or OneAgent or to ingest data
- Access is limited by the **scopes** you select when you create it
- It never gets more than its **user's own permissions**: a scope only works if the user (or service user) it belongs to has the matching permission
- Can have an expiry date or never expire; expired tokens return `403`
- Sent as `Authorization: Bearer <token>`
- Create it from your **profile → Platform tokens → Generate token**

> Treat a token like a password. Never commit it; use placeholders such as `<YOUR_DYNATRACE_PLATFORM_TOKEN>`.

## 3. Host group

- Groups hosts that share the same role or environment, for example `petclinic01` in Lab 1
- A host belongs to **only one** host group
- Used to apply settings to a group of hosts at once, such as monitoring settings, OneAgent updates and process group detection
- Set it during installation (`--set-host-group=<name>`) or later with `oneagentctl --set-host-group=<name>`, then restart the monitored apps. See [Lab 1 – 3.2 Update the host group](../lab01-petclinic-ec2-apm-rum/03-install-oneagent.md#32-update-the-host-group)

## 4. Real User Monitoring (RUM)

RUM records what real users do in the browser or mobile app. In Gen3 it is explored in **Experience Vitals** and **Users & Sessions**.

| # | Concept | What it is |
|---|---------|------------|
| 1 | **User session** | One visit by one user, from the first action until the user leaves or is inactive |
| 2 | **Session timeline** | The user actions and events of a session in order, with timings and errors |
| 3 | **Session Replay** | A video-like playback of the session, rebuilt from recorded page (DOM) changes |

> **Session Replay costs more.** Sessions with Session Replay are billed at a higher rate than plain RUM sessions. See [License calculation](07-license-calculation.md).

---

## References

- [OneAgent monitoring modes](https://docs.dynatrace.com/docs/platform/oneagent/monitoring-modes/monitoring-modes)
- [Dynatrace ActiveGate documentation](https://docs.dynatrace.com/docs/ingest-from/dynatrace-activegate)
- [Network zones](https://docs.dynatrace.com/docs/manage/network-zones-classic/network-zones-basic-info)
- [Platform tokens](https://docs.dynatrace.com/docs/manage/identity-access-management/access-tokens-and-oauth-clients/platform-tokens)
