# N8N Webhook → Jobber Integration

## Overview

This workflow receives data via a webhook in n8n and creates a client (and optionally a job/request) in Jobber using their GraphQL API.

## Architecture

```
[External Source] → [Webhook (n8n)] → [HTTP Request Node (GraphQL)] → [Jobber API]
```

## Jobber API Reference

- **Endpoint:** `https://api.getjobber.com/api/graphql`
- **Auth:** OAuth 2.0 Bearer Token
- **Format:** GraphQL (POST requests with JSON body)
- **Required Header:** `X-JOBBER-GRAPHQL-VERSION: 2025-01-13` (date-based API version)
- **Content-Type:** `application/json` (Jobber no longer accepts form-urlencoded or multipart)
- **Rate Limits:** 2,500 requests per 5 minutes per app/account + GraphQL query cost budget
- **Lead Source:** Clients created via API automatically have your app name as the lead source

## Prerequisites

1. A Jobber account with Developer Center access
2. A Jobber App created in the Developer Center (to get OAuth credentials)
3. An access token obtained via the OAuth 2.0 flow
4. Self-hosted n8n instance (e.g., https://n8n.calibrecleaning.com.au/)

---

## Step-by-Step: Building the Workflow in n8n

### Step 1: Create the Webhook Trigger Node

1. Open your n8n instance
2. Click **"Add first step"** → search for **"Webhook"**
3. Configure the Webhook node:
   - **HTTP Method:** `POST`
   - **Path:** `jobber-intake` (or any name you choose)
   - Copy the **Production URL** — this is what external systems will POST data to
   - Example: `https://n8n.calibrecleaning.com.au/webhook/jobber-intake`

### Step 2: Set Up Jobber Credentials in n8n

1. Go to **Settings** → **Credentials** → **Add Credential**
2. Search for **"Header Auth"** and create one:
   - **Name:** `Jobber API Token`
   - **Header Name:** `Authorization`
   - **Header Value:** `Bearer YOUR_ACCESS_TOKEN_HERE`

> **Important:** Jobber access tokens expire after ~60 minutes. See the
> "Token Refresh" section below for handling this automatically.

### Step 3: Add an HTTP Request Node (Create Client)

1. Add a new node after the Webhook → search for **"HTTP Request"**
2. Configure it:
   - **Method:** `POST`
   - **URL:** `https://api.getjobber.com/api/graphql`
   - **Authentication:** `Predefined Credential Type` → `Header Auth` → select `Jobber API Token`
   - **Send Headers:** ON
     - Add header: `Content-Type` = `application/json`
     - Add header: `X-JOBBER-GRAPHQL-VERSION` = `2025-01-13`
   - **Send Body:** ON
   - **Body Content Type:** `JSON`
   - **Specify Body:** `Using JSON`
   - **JSON Body:**

```json
{
  "query": "mutation ClientCreate($input: ClientCreateInput!) { clientCreate(input: $input) { client { id firstName lastName } userErrors { message path } } }",
  "variables": {
    "input": {
      "firstName": "{{ $json.firstName }}",
      "lastName": "{{ $json.lastName }}",
      "companyName": "{{ $json.companyName }}",
      "emails": [
        {
          "description": "MAIN",
          "primary": true,
          "address": "{{ $json.email }}"
        }
      ],
      "phones": [
        {
          "description": "MAIN",
          "primary": true,
          "number": "{{ $json.phone }}"
        }
      ],
      "billingAddress": {
        "street1": "{{ $json.address }}",
        "city": "{{ $json.city }}",
        "province": "{{ $json.state }}",
        "postalCode": "{{ $json.postalCode }}",
        "country": "{{ $json.country }}"
      }
    }
  }
}
```

### Step 4 (Optional): Add a Request/Job Creation Node

After the client is created, you can chain another HTTP Request node to create a job:

1. Add another **HTTP Request** node after the Create Client node
2. Same URL, Auth, and Headers as Step 3
3. **JSON Body:**

```json
{
  "query": "mutation JobCreate($input: JobCreateInput!) { jobCreate(input: $input) { job { id title } userErrors { message path } } }",
  "variables": {
    "input": {
      "clientId": "{{ $json.data.clientCreate.client.id }}",
      "title": "{{ $('Webhook').item.json.serviceType }}",
      "instructions": "{{ $('Webhook').item.json.notes }}"
    }
  }
}
```

### Step 5: Add Error Handling

The importable workflow (`workflow-webhook-to-jobber.json`) already includes
robust error handling. If you're building the workflow by hand, the key
rules (learned from real failures — see the "Failure Modes" section below)
are:

1. **Never let the HTTP node throw.** Set, under the HTTP Request node's
   **Options**:
   - `Response → Response → Never Error` = ON
   - `Response → Response → Full Response` = ON (so you can read `statusCode`)
   - `Timeout` = `30000` ms
   - `Retry On Fail` = ON, `Max Tries` = 3, `Wait Between Tries` = 2000 ms
2. **Classify the response in a Code node**, not an IF node, because
   `$json.data.clientCreate.userErrors.length` throws when `data` is
   undefined (which happens on auth errors, rate limits, and schema
   errors — Jobber returns `{"errors":[…]}` with no `data`).
3. Branch after the classifier: a boolean `success` flag routed to
   **Success Response** / **Error Response** `respondToWebhook` nodes.

### Step 6: Activate the Workflow

1. Click **"Save"** in the top right
2. Toggle the workflow to **Active**
3. Your Production webhook URL is now live and listening

---

## Testing the Webhook

Send a test POST request using curl:

```bash
curl -X POST https://n8n.calibrecleaning.com.au/webhook/jobber-intake \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "John",
    "lastName": "Smith",
    "companyName": "Smith Residence",
    "email": "[email protected]",
    "phone": "0412345678",
    "address": "123 Main Street",
    "city": "Melbourne",
    "state": "VIC",
    "postalCode": "3000",
    "country": "AU",
    "serviceType": "Standard Clean",
    "notes": "3 bedroom house, weekly clean"
  }'
```

Or use the **"Test workflow"** button in n8n and click **"Listen for test event"** on the Webhook node, then send the curl request to the **Test URL** shown.

---

## Handling Token Refresh (Important!)

Jobber OAuth tokens expire after ~60 minutes. You have two options:

### Option A: Manual Token (Quick Start)

Use the Developer Center's built-in tool to generate a token, paste it into n8n credentials. You'll need to update it manually when it expires. **Good for testing only.**

### Option B: Automatic Token Refresh (Recommended for Production)

1. Create a separate n8n workflow for token refresh:
   - Use a **Schedule Trigger** (every 50 minutes)
   - **HTTP Request** node to POST to `https://api.getjobber.com/api/oauth/token` with:
     ```json
     {
       "client_id": "YOUR_CLIENT_ID",
       "client_secret": "YOUR_CLIENT_SECRET",
       "grant_type": "refresh_token",
       "refresh_token": "YOUR_REFRESH_TOKEN"
     }
     ```
   - Store the new access token (e.g., in a static data variable or n8n credentials API)

2. Alternatively, use n8n's built-in **OAuth2 API** credential type:
   - **Grant Type:** Authorization Code
   - **Authorization URL:** `https://api.getjobber.com/api/oauth/authorize`
   - **Access Token URL:** `https://api.getjobber.com/api/oauth/token`
   - **Client ID:** Your Jobber app's Client ID
   - **Client Secret:** Your Jobber app's Client Secret
   - **Scope:** `read_clients write_clients read_jobs write_jobs` (adjust as needed)
   - n8n will handle token refresh automatically!

---

## Example Webhook Payload Schema

The webhook expects a JSON body. Here's the recommended schema:

| Field         | Type   | Required | Description                    |
|---------------|--------|----------|--------------------------------|
| firstName     | string | Yes      | Client first name              |
| lastName      | string | Yes      | Client last name               |
| companyName   | string | No       | Company or property name       |
| email         | string | Yes      | Client email address           |
| phone         | string | No       | Client phone number            |
| address       | string | No       | Street address                 |
| city          | string | No       | City                           |
| state         | string | No       | State/Province                 |
| postalCode    | string | No       | Postal/ZIP code                |
| country       | string | No       | Country code (e.g., AU)        |
| serviceType   | string | No       | Type of service requested      |
| notes         | string | No       | Additional notes/instructions  |

---

## Useful Jobber GraphQL Queries

### List Clients
```graphql
query {
  clients(first: 10) {
    nodes {
      id
      firstName
      lastName
      emails { address }
    }
  }
}
```

### Get Client by ID
```graphql
query {
  client(id: "CLIENT_ID") {
    firstName
    lastName
    jobs { nodes { id title } }
  }
}
```

---

## Failure Modes & Prevention

The first version of this workflow had several ways to hard-fail an
execution (making it appear as a red "failed" run in n8n instead of
returning a structured error to the caller). The current workflow
guards against each of these:

| # | Failure | Symptom | Guard |
|---|---------|---------|-------|
| 1 | Expired Jobber token (HTTP 401) | Node throws "401 Unauthorized" and the whole execution is marked failed; webhook caller gets no JSON | `neverError: true` on HTTP node + response classifier returns `error: "auth_error"` with HTTP 502 |
| 2 | Rate limit (HTTP 429) | Execution fails on a busy hour | `neverError: true` + node retry (3 tries, 2 s backoff) + classifier returns `rate_limited` |
| 3 | Upstream 5xx / network timeout | Execution fails with a stack trace | `timeout: 30000`, node retry, `neverError` |
| 4 | GraphQL top-level `errors` (no `data`) | IF node throws `Cannot read properties of undefined (reading 'clientCreate')` | Classifier Code node checks `body.errors` before touching `body.data` |
| 5 | Missing required name fields | Empty `firstName`/`lastName`/`companyName` reach Jobber and come back as `userErrors` | Normalize & Validate Code node rejects the request with HTTP 400 before the API call |
| 6 | Payload shape drift (`first_name` vs `firstName`, `{body: {...}}` wrapper) | Fields silently empty | Normalize node accepts snake_case/camelCase aliases and unwraps `body` |
| 7 | Empty `billingAddress`/`emails`/`phones` sent as empty strings/arrays | Occasional Jobber `userErrors` | Normalize node omits empty optional fields entirely |
| 8 | Malformed JSON body from a stringified template | Silent bad-request loops | Body is built as a structured object in the Code node and stringified once at the HTTP node boundary |

### Monitoring recommendations

- In n8n **Settings → Log streaming** (or **Error Workflow**), wire a
  catch-all error workflow that posts to Slack/email when a run still
  somehow fails. With the guards above, this should be rare — useful as
  a canary.
- Set the workflow's **Settings → Error Workflow** to a dedicated
  workflow so unexpected crashes surface quickly instead of piling up
  silently in the Executions tab.
- If you aren't using OAuth2 (Option B under "Handling Token Refresh"),
  build a scheduled token-refresh workflow — cause #1 above will come
  back every 60 minutes without it.

---

## Resources

- [Jobber API Docs](https://developer.getjobber.com/docs/)
- [Jobber API Queries & Mutations](https://developer.getjobber.com/docs/using_jobbers_api/api_queries_and_mutations/)
- [Jobber OAuth 2.0 Guide](https://developer.getjobber.com/docs/building_your_app/app_authorization/)
- [n8n Webhook Node Docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/)
- [Jobber Developer Center](https://help.getjobber.com/hc/en-us/articles/25924078048151-Developer-Center)
