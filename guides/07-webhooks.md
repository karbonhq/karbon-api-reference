---
slug: webhooks
sortOrder: 7
seo:
  title: Webhooks
  description: Subscribe to Karbon webhook events to receive real-time push notifications when contacts, work items, invoices, and other entities change—eliminating the need to poll—then fetch the full updated record using the ResourcePermaKey in the payload.
---

# Webhooks

Webhooks let Karbon push notifications to your server when data changes, eliminating the need to poll for updates.

## Overview

When a subscribed event occurs, Karbon sends a `POST` request to your endpoint with a lightweight payload identifying what changed. You then fetch the full record if needed.

**Delivery behaviour:**

- One notification is sent per entity change (create, update, delete)
- Notifications are coalesced — there is a 60-second window before a notification is sent, so rapid changes to the same entity result in a single notification rather than many
- Bulk operations in Karbon (e.g. updating many clients at once) can generate a large number of webhook notifications in a short period — ensure your endpoint can handle bursts and processes payloads asynchronously

**Payload format:**

```json
{
  "ResourcePermaKey": "{EntityKey}",
  "ResourceType": "Contact",
  "ActionType": "Updated",
  "TimeStamp": "2026-03-31T14:22:00Z"
}
```

## Available Webhook Types

| Type              | Triggers on                               |
| ----------------- | ----------------------------------------- |
| `Contact`         | Contacts, Organizations, and ClientGroups |
| `Work`            | Work items                                |
| `Note`            | Notes                                     |
| `User`            | Users                                     |
| `Invoice`         | Invoices                                  |
| `EstimateSummary` | Estimate summaries                        |
| `CustomField`     | Custom field definitions                  |
| `IntegrationTask` | Integration tasks (partners only)         |

## Creating a Subscription

```http
POST https://api.karbonhq.com/v3/WebhookSubscriptions
Authorization: Bearer {token}
AccessKey: {key}
Content-Type: application/json

{
  "WebhookType": "Work",
  "TargetUrl": "https://your-app.example.com/webhooks/karbon",
  "SigningKey": "your-secret-signing-key-min-16-chars"
}
```

`SigningKey` is optional but recommended — it's used to sign webhook payloads so you can verify they genuinely came from Karbon. It must be at least 16 characters and contain only letters, numbers, dashes, or underscores.

**Important:** You can only have one active subscription per webhook type per API application.

## Checking a Subscription

```http
GET https://api.karbonhq.com/v3/WebhookSubscriptions/Work
Authorization: Bearer {token}
AccessKey: {key}
```

A `404` response means either no subscription exists or it was auto-cancelled due to delivery failures.

## Deleting a Subscription

```http
DELETE https://api.karbonhq.com/v3/WebhookSubscriptions/Work
Authorization: Bearer {token}
AccessKey: {key}
```

## Auto-Cancellation

If your endpoint fails to respond with an HTTP `2xx` status **10 consecutive times**, Karbon automatically cancels the subscription. To resume receiving events:

1. Fix whatever caused your endpoint to fail
2. Re-create the subscription with a new `POST`

Build monitoring into your webhook endpoint to detect and alert on delivery failures before you hit the 10-failure threshold.

## Handling Incoming Webhooks

Your endpoint must:

1. Respond with HTTP `2xx` quickly (before any timeout)
2. Process the notification asynchronously if needed

A minimal example:

```python
from flask import Flask, request, jsonify
import requests

app = Flask(__name__)

KARBON_HEADERS = {
    "Authorization": "Bearer {token}",
    "AccessKey": "{key}"
}

@app.route("/webhooks/karbon", methods=["POST"])
def handle_webhook():
    payload = request.get_json()

    resource_key = payload["ResourcePermaKey"]
    resource_type = payload["ResourceType"]
    action_type = payload["ActionType"]

    # Acknowledge immediately
    # Process asynchronously (queue, background task, etc.)
    enqueue_task(resource_type, resource_key, action_type)

    return jsonify({"ok": True}), 200

def enqueue_task(resource_type, resource_key, action_type):
    # Your async processing logic here
    pass
```

## Fetching the Changed Resource

The webhook payload only tells you what changed — it doesn't include the full record. Fetch it using the `ResourcePermaKey`:

```python
def fetch_changed_resource(resource_type, resource_key):
    endpoint_map = {
        "Contact": f"/v3/Contacts/{resource_key}",
        "Work": f"/v3/WorkItems/{resource_key}",
        "Invoice": f"/v3/Invoices/{resource_key}",
    }
    url = f"https://api.karbonhq.com{endpoint_map[resource_type]}"
    return requests.get(url, headers=KARBON_HEADERS).json()
```

## Webhook Security

Validate that incoming requests are genuinely from Karbon before processing them. Consider:

- Setting a `SigningKey` when creating the subscription — Karbon uses it to sign payloads so you can verify authenticity
- Restricting your webhook endpoint to Karbon's IP ranges (check Developer Center documentation for current ranges)
- Checking the `Content-Type` header is `application/json`
