# Lab 2 – Configure Synthetic Monitoring

Set up a **private Synthetic location** on a Synthetic-enabled ActiveGate, then create an **HTTP monitor** (including a `POST` API check), a single-URL **browser monitor**, a **clickpath** monitor and a **network availability monitor** that check easyTravel from it.

For what the monitors cost, see [License calculation](../notes/07-license-calculation.md).

For the concepts (monitor types, locations, authentication), see [02-synthetic-monitoring-concepts.md](02-synthetic-monitoring-concepts.md).

## Checklist

1. [Install a Synthetic-enabled ActiveGate](#1-install-a-synthetic-enabled-activegate)
   - [x] [1.1 Create a VM](#11-create-a-vm)
   - [x] [1.2 In the Dynatrace console](#12-in-the-dynatrace-console)
   - [x] [1.3 On the ActiveGate server](#13-on-the-activegate-server)
2. [Configure the private location](#2-configure-the-private-location)
   - [x] [2.1 Create the private location](#21-create-the-private-location)
   - [x] [2.2 Why leave Chromium auto-update off](#22-why-leave-chromium-auto-update-off)
3. [Configure an HTTP monitor](#3-configure-an-http-monitor)
   - [x] [3.1 Check that the app is reachable](#31-check-that-the-app-is-reachable)
   - [x] [3.2 Create the monitor](#32-create-the-monitor)
   - [x] [3.3 Test the monitor](#33-test-the-monitor)
   - [x] [3.4 Monitor a POST API](#34-monitor-a-post-api)
   - [ ] [3.5 Monitor a POST API with authentication](#35-monitor-a-post-api-with-authentication)
     - [ ] [3.5.1 Test the API first](#351-test-the-api-first)
     - [ ] [3.5.2 Store the credentials](#352-store-the-credentials)
     - [ ] [3.5.3 Create the monitor](#353-create-the-monitor)
     - [ ] [3.5.4 Other authentication options](#354-other-authentication-options)
4. [Configure a browser monitor](#4-configure-a-browser-monitor)
   - [x] [4.1 Single-URL browser monitor](#41-single-url-browser-monitor)
   - [x] [4.2 Clickpath with the Synthetic Recorder](#42-clickpath-with-the-synthetic-recorder)
   - [x] [4.3 Browser monitor settings](#43-browser-monitor-settings)
5. [Configure a network availability monitor](#5-configure-a-network-availability-monitor)
   - [ ] [5.1 Enable ICMP on the private location](#51-enable-icmp-on-the-private-location)
   - [ ] [5.2 Create the monitor](#52-create-the-monitor)
6. [Cleanup](#6-cleanup)
7. [References](#references)

---

## 1. Install a Synthetic-enabled ActiveGate

A private location runs synthetic monitors from inside your own network, on a Synthetic-enabled Environment ActiveGate.

### 1.1 Create a VM

| # | Setting | Value |
|---|---------|-------|
| 1 | Instance type | `t2.medium` (2 vCPU, 4 GB RAM) |
| 2 | Storage | 60 GB |
| 3 | Image | Amazon Linux |

### 1.2 In the Dynatrace console

1. Go to **Discovery & Coverage**
2. Click **Install** (top right)
3. Select **Install ActiveGate**
4. Select **Synthetic** as the ActiveGate purpose
5. Select the Network Zone `petclinic`
6. Create an ActiveGate group named `pvt01`

### 1.3 On the ActiveGate server

1. Create `connection.properties` with the values shown in the console. Never paste real tokens into this file:

   ```properties
   SERVER=https://nes80740.live.dynatrace.com/communication
   TENANT=nes80740
   TENANT_TOKEN=<YOUR_TENANT_TOKEN>
   ACTIVEGATE_AUTH_TOKEN=<YOUR_ACTIVEGATE_AUTH_TOKEN>
   ```

2. Download the installer:

   ```bash
   wget -O Dynatrace-ActiveGate-Linux-x86-1.349.80.20261105-202347-petclinic.sh \
     "https://nes80740.apps.dynatrace.com/platform/fleet-management/v1/activegate/installer/unix/amd64/1.349.80.20261105-202347:download" \
     --header="Authorization: Bearer <YOUR_DYNATRACE_PLATFORM_TOKEN>" \
     --header="accept: application/octet-stream"
   ```

3. Verify that the script is authentic:

   ```bash
   wget https://ca.dynatrace.com/dt-root.cert.pem ; ( echo 'Content-Type: multipart/signed; protocol="application/x-pkcs7-signature"; micalg="sha-256"; boundary="--SIGNED-INSTALLER"'; echo ; echo ; echo '----SIGNED-INSTALLER' ; cat Dynatrace-ActiveGate-Linux-x86-1.349.80.20261105-202347-petclinic.sh ) | openssl cms -verify -CAfile dt-root.cert.pem > /dev/null
   ```

4. Run the installer with Synthetic enabled:

   ```bash
   /bin/bash Dynatrace-ActiveGate-Linux-x86-1.349.80.20261105-202347-petclinic.sh \
     --connection-properties=connection.properties \
     --enable-synthetic \
     --set-network-zone=petclinic \
     --set-group=pvt01
   ```

5. Check the status, then delete `connection.properties`:

   ```bash
   systemctl status dynatracegateway.service
   shred -u connection.properties
   ```

---

## 2. Configure the private location

> **Prerequisite:** Create the private location before you configure any synthetic monitor that runs from it.

### 2.1 Create the private location

1. Go to **Synthetic** and click **New private location** (top right)
2. Select **Classic**
3. Fill in the details:

   | # | Field | Value |
   |---|-------|-------|
   | 1 | Location name | `SG-Punggol` |
   | 2 | Geographic location | Singapore, Singapore |
   | 3 | ActiveGate | The Synthetic-enabled ActiveGate in group `pvt01` |
   | 4 | Chromium auto-update | **Leave unselected** |
   | 5 | Location outage handling | Select the options to be notified when the location has an outage |

4. Click **Save**

### 2.2 Why leave Chromium auto-update off

The private location uses Chromium, the open-source browser behind Chrome, to run browser monitors. With auto-update on, Dynatrace updates it whenever a new version is released.

Keep it off because:

1. **A failed update can break the location.** If an update fails, for example because of a network issue, synthetic monitoring on the location can stop working and you have to re-create it.
2. **Updates run without notice.** This causes unplanned monitoring downtime and unnecessary downtime alerts.

Instead, update Chromium manually in a planned maintenance window.

---

## 3. Configure an HTTP monitor

> **Prerequisite:** [2. Configure the private location](#2-configure-the-private-location)

### 3.1 Check that the app is reachable

The monitor runs on the Synthetic ActiveGate, so run this on the ActiveGate server:

```bash
curl -I http://<EC2_PUBLIC_IP>:32770/
```

You should get `HTTP/1.1 200 OK`. If the request times out, allow the port in the easyTravel VM's security group.

### 3.2 Create the monitor

1. Go to **Synthetic → Create a synthetic monitor → HTTP monitor**
2. Enter the monitor name and tags, and select the existing frontend
3. Add the HTTP request:

   | # | Setting | Value |
   |---|---------|-------|
   | 1 | Request name | A name for the request |
   | 2 | URL | The URL to check, for example the easyTravel URL |
   | 3 | Authentication | Only if the URL needs a login |
   | 4 | SSL | Certificate options, for example accept any SSL certificate |
   | 5 | Response status code verification | Default: **Fail** on `4xx` and `5xx` |

4. Set the frequency to **5 minutes** and select the private location `SG-Punggol`
5. Choose the outage alert options and save the monitor

> For the status code rules you can enter, see [Response status code verification](02-synthetic-monitoring-concepts.md#113-response-status-code-verification). For what each code means, see [HTTP status codes](02-synthetic-monitoring-concepts.md#114-http-status-codes).

### 3.3 Test the monitor

Run the monitor on demand instead of waiting for the next scheduled run:

1. Go to **Synthetic** and select the monitor
2. Click **View details** (top right) → **On-demand execution**
3. Select the location `SG-Punggol` and click **Trigger now**

To see the result of the last run in JSON format:

1. Go to **Synthetic** and select the monitor
2. Click **Analyze data → Last successful**

### 3.4 Monitor a POST API

An HTTP monitor can also call an API with a request body. This example sends a `POST` to the free [JSONPlaceholder](https://jsonplaceholder.typicode.com/) test API.

> **Test the API first.** Run the request in Postman or with `curl` before you create the monitor, so you don't pay for failed monitor runs while you fix the request:
>
> ```bash
> curl -i -X POST https://jsonplaceholder.typicode.com/posts \
>   -H "Content-Type: application/json" \
>   -d '{"title": "Dynatrace Synthetic Training", "body": "Testing POST API using Dynatrace", "userId": 101}'
> ```
>
> You should get `HTTP/1.1 201 Created`.

1. Go to **Synthetic → Create a synthetic monitor → HTTP monitor**
2. Add the HTTP request:

   | # | Setting | Value |
   |---|---------|-------|
   | 1 | URL | `https://jsonplaceholder.typicode.com/posts` |
   | 2 | HTTP method | `POST` |
   | 3 | Additional HTTP headers | Header name `Content-Type`, header value `application/json` |
   | 4 | Request body | The JSON below |

   ```json
   {
     "title": "Dynatrace Synthetic Training",
     "body": "Testing POST API using Dynatrace",
     "userId": 101
   }
   ```

3. Click **Continue** and save the monitor
4. Trigger it on demand (see [3.3 Test the monitor](#33-test-the-monitor)). A successful run returns `201 Created`.

### 3.5 Monitor a POST API with authentication

Most real APIs need a login. This example uses the free [DummyJSON](https://dummyjson.com/docs/auth) test API, which works like a real login flow:

1. `POST /auth/login` with a username and password returns an **access token**
2. `GET /auth/me` returns the logged-in user, but only if you send the token as `Authorization: Bearer <token>`. Without it, you get `401`.

The monitor runs both requests in order and passes the token from the first to the second. The credentials are stored in the Dynatrace **credential vault**, so they never appear in the monitor configuration.

> DummyJSON's demo user `emilys` / `emilyspass` is published in its docs for testing. For a real API, use a dedicated service account (see [Authentication requirements](02-synthetic-monitoring-concepts.md#31-requirements)).

#### 3.5.1 Test the API first

Test in Postman or with `curl`:

```bash
# 1. Log in and get the token (expect HTTP 200)
TOKEN=$(curl -s -X POST https://dummyjson.com/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "emilys", "password": "emilyspass", "expiresInMins": 30}' \
  | python3 -c 'import sys, json; print(json.load(sys.stdin)["accessToken"])')

# 2. Call the protected endpoint with the token (expect HTTP 200; 401 without the token)
curl -i https://dummyjson.com/auth/me -H "Authorization: Bearer $TOKEN"
```

#### 3.5.2 Store the credentials

1. Open **Credential vault** (search for it in the Dynatrace menu) and click **Add credentials**
2. Select **User and password**, enter `emilys` / `emilyspass`, and save
3. Note the credential ID, for example `CREDENTIALS_VAULT-0000000000000000`

#### 3.5.3 Create the monitor

1. Go to **Synthetic → Create a synthetic monitor → HTTP monitor**
2. Add request 1, **Login**:

   | # | Setting | Value |
   |---|---------|-------|
   | 1 | URL | `https://dummyjson.com/auth/login` |
   | 2 | HTTP method | `POST` |
   | 3 | Additional HTTP headers | Header name `Content-Type`, header value `application/json` |
   | 4 | Request body | The JSON below. The placeholders are replaced with the stored credentials at run time. |
   | 5 | Post-execution script | The script below. It saves the token for the next request. |

   ```json
   {
     "username": "{CREDENTIALS_VAULT-0000000000000000|username}",
     "password": "{CREDENTIALS_VAULT-0000000000000000|password}",
     "expiresInMins": 30
   }
   ```

   ```javascript
   var body = JSON.parse(response.getResponseBody());
   api.setValue("accessToken", body.accessToken);
   ```

3. Add request 2, **Get current user**:

   | # | Setting | Value |
   |---|---------|-------|
   | 1 | URL | `https://dummyjson.com/auth/me` |
   | 2 | HTTP method | `GET` |
   | 3 | Pre-execution script | The script below. It adds the token from request 1. |

   ```javascript
   request.addHeader("Authorization", "Bearer " + api.getValue("accessToken"));
   ```

4. Select the private location `SG-Punggol`, set the frequency, and save the monitor
5. Trigger it on demand (see [3.3 Test the monitor](#33-test-the-monitor)). Both requests should return `200`. A `401` on request 2 means the token wasn't passed.

#### 3.5.4 Other authentication options

| # | Authentication | How to set it |
|---|----------------|---------------|
| 1 | **Basic** (username and password) | **Set authentication/authorization → Basic**, then select the credential. Dynatrace builds the `Authorization` header for you. |
| 2 | **Static token** (API key or long-lived token) | Add a header `Authorization` with value `Bearer {CREDENTIALS_VAULT-0000000000000000\|token}`, using a **Token** credential |
| 3 | **NTLM / Kerberos / OAuth 2.0 / certificate** | **Set authentication/authorization** and select the type. See [Supported authentication types](02-synthetic-monitoring-concepts.md#33-supported-authentication-types). |

> The `{<credential ID>|username}`, `{<credential ID>|password}` and `{<credential ID>|token}` placeholders work in the request URL, headers and body.

---

## 4. Configure a browser monitor

> **Prerequisite:** [2. Configure the private location](#2-configure-the-private-location)

Unlike an HTTP monitor, a browser monitor captures **screenshots** of the page. It runs either a **single URL** (one page load) or a **clickpath** (a recorded user journey). See [Monitor types](02-synthetic-monitoring-concepts.md#1-monitor-types).

### 4.1 Single-URL browser monitor

1. Go to **Synthetic → Create a synthetic monitor → Browser monitor**
2. Enter the monitor name and the URL to check
3. Add the conditions and a content validation rule, for example text that must appear on the page
4. Select the private location `SG-Punggol`
5. Keep the defaults for the other settings and save the monitor

### 4.2 Clickpath with the Synthetic Recorder

The **Synthetic Recorder** is a Chrome extension that records the transactions you perform in the app and turns them into a clickpath. You don't have to write any script.

1. Go to **Synthetic → Create a synthetic monitor → Browser monitor**
2. Enter the application name and the URL
3. Click **Install Synthetic Recorder** and add the extension to Chrome
4. Click **Record clickpath**. The URL opens in a private browser window.
5. Perform the transactions you want to monitor, for example search → book → pay
6. Test the clickpath. If a step isn't right, correct it or record it again.
7. Continue, keep the default values, and set the frequency to **5 minutes**
8. Save the monitor, then trigger it on demand to check that it works (see [3.3 Test the monitor](#33-test-the-monitor))

> A clickpath is billed for **every action** in the recording, so keep it to the steps you need. See [3. Browser monitor cost](../notes/07-license-calculation.md#3-browser-monitor-cost).

### 4.3 Browser monitor settings

| # | Setting | What it does |
|---|---------|--------------|
| 1 | **Credentials** | Store login details for the monitor in the Dynatrace **credential vault**, or sync them from an external vault: Azure Key Vault, HashiCorp Vault or CyberArk Vault |
| 2 | **Element locators** | How a clickpath event finds the element it clicks or types into: a **DOM** locator (document object model) or a **CSS selector** |

---

## 5. Configure a network availability monitor

> **Prerequisite:** [2. Configure the private location](#2-configure-the-private-location). Network availability monitors run **only** from private locations.

A network availability monitor checks basic reachability of a host, without HTTP:

| # | Protocol | What it checks | Lab example |
|---|----------|----------------|-------------|
| 1 | **ICMP** | The host answers a ping | The easyTravel VM's private IP |
| 2 | **TCP** | A port on the host is open and accepts connections | The easyTravel VM on port `32770` |
| 3 | **DNS** | A host name resolves to an IP address | A public host name, for example `www.dynatrace.com` |

### 5.1 Enable ICMP on the private location

ICMP is off by default on a private location. To use ICMP checks:

1. Go to **Settings → Web and mobile monitoring → Private Synthetic locations**
2. Open `SG-Punggol`
3. Turn on **Enable ICMP request type for Network Availability Monitors execution** and save

> Also allow ICMP (and the TCP port you check) from the ActiveGate to the target in the easyTravel VM's security group.

### 5.2 Create the monitor

1. Go to **Synthetic → Create a synthetic monitor → Network availability monitor**
2. Enter the monitor name and select the protocol: **ICMP**, **TCP** or **DNS**
3. Enter the target host (and the port, for TCP)
4. For ICMP, keep the defaults for the number of packets, timeout and success rate, or adjust them
5. Select the private location `SG-Punggol` and set the frequency
6. Save the monitor, then trigger it on demand (see [3.3 Test the monitor](#33-test-the-monitor))

---

## 6. Cleanup

When you finish the lab, remove the synthetic setup so it stops running and billing:

1. Delete the HTTP, browser, clickpath and network availability monitors in **Synthetic**
2. Delete the private location `SG-Punggol`
3. Uninstall the ActiveGate on the ActiveGate server:

   ```bash
   /opt/dynatrace/gateway/uninstall.sh
   ```

4. Terminate the ActiveGate EC2 instance in AWS

---

## References

- [DummyJSON auth API](https://dummyjson.com/docs/auth)
- [Pre- and post-execution scripts for HTTP monitors](https://docs.dynatrace.com/docs/observe/digital-experience/synthetic-monitoring/http-monitors-classic/pre-and-post-scripting-for-http-monitors-classic)
- [Supported authentication methods in Synthetic Monitoring](https://docs.dynatrace.com/docs/observe/digital-experience/synthetic-monitoring/general-information/synthetic-authentication)
- [Configure a NAM monitor](https://docs.dynatrace.com/docs/observe/digital-experience/synthetic-monitoring/network-availability-monitors/create-a-nam-monitor)
- [Network availability monitoring](https://docs.dynatrace.com/docs/observe/digital-experience/synthetic-monitoring/network-availability-monitors/network-availability-monitoring)
- [Create a private Synthetic location](https://docs.dynatrace.com/docs/observe/digital-experience-classic/synthetic-monitoring-classic/private-synthetic-locations/create-a-private-synthetic-location)
- [Manage private Synthetic locations – Browser autoupdate](https://docs.dynatrace.com/docs/observe/digital-experience-classic/synthetic-monitoring-classic/private-synthetic-locations/manage-private-synthetic-locations#:~:text=Data%20Explorer%20Classic-,Browser%20autoupdate,-You%20can%20Enable)
- [Configure HTTP monitors in Classic](https://docs.dynatrace.com/docs/observe/digital-experience-classic/synthetic-monitoring-classic/http-monitors-classic/configure-http-monitors-classic)
- [Dynatrace pricing](https://www.dynatrace.com/pricing/)
- [Dynatrace API reference (Swagger)](https://api.dynatrace.com/spec/#/): fetch information from Dynatrace through its APIs
