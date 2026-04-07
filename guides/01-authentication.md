# Making Your First API Request

The Karbon API uses two required headers on every request. Once you have them, you're ready to go.

## Prerequisites

- A Karbon account with API access

## Required Headers

Every request must include both of the following headers:

| Header | Format | Where to find it |
|---|---|---|
| `Authorization` | `Bearer {token}` | Issued when you register for API access |
| `AccessKey` | JWT (starts with `eyJ...`) | Karbon UI → Settings → Connected Apps → your application |

The `Authorization` token is a 36 character long GUID. The `AccessKey` is a JWT with three Base64URL-encoded segments separated by dots.

This guide contains example values for Authorization and AccessKey, you'll need to substitute in your own values in order to make a successful API request.

## Base URL

```
https://api.karbonhq.com
```

All endpoints are versioned under `/v3/`.

## Making a Request

The following examples all call `GET /v3/TenantSettings` — a good smoke test that returns your account configuration with no side effects.

### Postman / Bruno

[Postman](https://www.postman.com) and [Bruno](https://www.usebruno.com) are desktop API clients that let you make HTTP requests without writing any code — useful for exploring the API and testing requests interactively. Bruno is open-source and stores collections as local files; Postman is cloud-based with a generous free tier.

1. Create a new **GET** request to `https://api.karbonhq.com/v3/TenantSettings`
2. Open the **Headers** tab and add:
   - Key: `Authorization` — Value: `Bearer 550e8400-e29b-41d4-a716-446655440000`
   - Key: `AccessKey` — Value: `eyJhbGciOiJIUzI1NiJ9...`
3. Send the request

**Tip:** Store your credentials in an environment and reference them as variables (e.g. `{{authToken}}` and `{{accessKey}}`) so you don't need to repeat them on every request.

### curl

```bash
curl https://api.karbonhq.com/v3/TenantSettings \
  -H "Authorization: Bearer 550e8400-e29b-41d4-a716-446655440000" \
  -H "AccessKey: eyJhbGciOiJIUzI1NiJ9.eyJrZXkiOiJ2YWx1ZSJ9.signature"
```

### JavaScript (fetch)

```javascript
const TOKEN = process.env.KARBON_TOKEN;
const ACCESS_KEY = process.env.KARBON_ACCESS_KEY;

const response = await fetch("https://api.karbonhq.com/v3/TenantSettings", {
  headers: {
    "Authorization": `Bearer ${TOKEN}`,
    "AccessKey": ACCESS_KEY,
  },
});

if (!response.ok) {
  throw new Error(`HTTP ${response.status}: ${response.statusText}`);
}

const data = await response.json();
console.log(data);
```

### JavaScript (axios)

```javascript
import axios from "axios";

const client = axios.create({
  baseURL: "https://api.karbonhq.com/v3",
  headers: {
    "Authorization": `Bearer ${process.env.KARBON_TOKEN}`,
    "AccessKey": process.env.KARBON_ACCESS_KEY,
  },
});

const { data } = await client.get("/TenantSettings");
console.log(data);
```

### Python (requests)

```python
import os
import requests

headers = {
    "Authorization": f"Bearer {os.environ['KARBON_TOKEN']}",
    "AccessKey": os.environ["KARBON_ACCESS_KEY"],
}

response = requests.get(
    "https://api.karbonhq.com/v3/TenantSettings",
    headers=headers,
)
response.raise_for_status()
print(response.json())
```

**Tip:** Create a reusable session so headers are sent on every request automatically:

```python
session = requests.Session()
session.headers.update(headers)

tenant = session.get("https://api.karbonhq.com/v3/TenantSettings").json()
contacts = session.get("https://api.karbonhq.com/v3/Contacts").json()
```

### C# (.NET)

```csharp
using System.Net.Http;
using System.Net.Http.Headers;

var token = Environment.GetEnvironmentVariable("KARBON_TOKEN");
var accessKey = Environment.GetEnvironmentVariable("KARBON_ACCESS_KEY");

using var client = new HttpClient { BaseAddress = new Uri("https://api.karbonhq.com/v3/") };
client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Bearer", token);
client.DefaultRequestHeaders.Add("AccessKey", accessKey);

var response = await client.GetAsync("TenantSettings");
response.EnsureSuccessStatusCode();

var json = await response.Content.ReadAsStringAsync();
Console.WriteLine(json);
```

**Tip:** Register `HttpClient` as a singleton (e.g., via `IHttpClientFactory` in ASP.NET Core) so headers are configured once and reused across requests.

### Ruby (net/http)

```ruby
require 'net/http'
require 'uri'
require 'json'

uri = URI('https://api.karbonhq.com/v3/TenantSettings')

response = Net::HTTP.start(uri.host, uri.port, use_ssl: true) do |http|
  request = Net::HTTP::Get.new(uri)
  request['Authorization'] = "Bearer #{ENV['KARBON_TOKEN']}"
  request['AccessKey'] = ENV['KARBON_ACCESS_KEY']
  http.request(request)
end

puts JSON.parse(response.body)
```

Or with the `faraday` gem for a cleaner interface:

```ruby
require 'faraday'
require 'json'

conn = Faraday.new(url: 'https://api.karbonhq.com/v3') do |f|
  f.headers['Authorization'] = "Bearer #{ENV['KARBON_TOKEN']}"
  f.headers['AccessKey'] = ENV['KARBON_ACCESS_KEY']
  f.response :raise_error
end

response = conn.get('/TenantSettings')
puts JSON.parse(response.body)
```

## Verifying Your Setup

A successful `GET /v3/TenantSettings` returns your tenant's configuration:

```json
{
  "ContactTypes": ["Individual", "Business"],
  "WorkTypes": [...],
  "WorkStatuses": [...],
  "ClientAccessActivated": false
}
```

Cache this response — you'll need `WorkTypes`, `ContactTypes`, and `WorkStatuses` throughout your integration.

## Account-Level Access

The Karbon API operates at the **account level**, not the user level. When you authenticate, you are acting on behalf of the entire Karbon account — not as any individual user within it.

This means:
- Actions taken via the API (creating work, updating contacts, recording payments) are not attributed to a specific Karbon user
- The `AccessKey` is tied to your API application and the account it's connected to, not to any individual's login
- All data visible within that account is accessible regardless of which user would normally see it in the Karbon UI

If your integration needs to attribute actions to a specific person (e.g., setting an assignee on a work item), you pass that user's email address explicitly in the request body — it is not inferred from the credentials.

## Common Authentication Errors

| Status | Meaning |
|---|---|
| `401 Unauthorized` | Missing or invalid `Authorization` header or `AccessKey` |

## Security Note

Never hard-code credentials in source code. Use environment variables or a secrets manager.
