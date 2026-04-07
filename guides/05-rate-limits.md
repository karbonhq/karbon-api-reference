# Handling Rate Limits

The Karbon API enforces rate limits per account per API application. Exceeding the limit returns a `429 Too Many Requests` response.

## The Limit

**Recommended:** No more than 120 requests per minute (2 requests/second).

Rate limits are scoped per account and per API application, so different applications connecting to the same account have independent limits.

## What a Rate Limit Response Looks Like

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 10

{
  "statusCode": "429",
  "message": "Rate limit is exceeded. Try again in 10 seconds."
}
```

## Handling 429s in Your Code

Always check for `429` responses and back off before retrying. Use the `Retry-After` header value (in seconds) as your minimum wait time.

A simple exponential backoff approach:

```python
import time
import requests

def api_request_with_retry(url, headers, max_retries=5):
    for attempt in range(max_retries):
        response = requests.get(url, headers=headers)

        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", 10))
            wait = retry_after * (2 ** attempt)  # exponential backoff
            print(f"Rate limited. Waiting {wait}s before retry {attempt + 1}/{max_retries}")
            time.sleep(wait)
            continue

        response.raise_for_status()
        return response

    raise Exception("Max retries exceeded")
```

```javascript
async function apiRequest(url, headers, maxRetries = 5) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    const response = await fetch(url, { headers });

    if (response.status === 429) {
      const retryAfter = parseInt(response.headers.get("Retry-After") ?? "10");
      const wait = retryAfter * Math.pow(2, attempt); // exponential backoff
      console.log(`Rate limited. Waiting ${wait}s...`);
      await new Promise(resolve => setTimeout(resolve, wait * 1000));
      continue;
    }

    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return response;
  }
  throw new Error("Max retries exceeded");
}
```

## Staying Within the Limit

### Enable response compression

Setting `Accept-Encoding: gzip` significantly reduces response sizes, which means faster requests and less need to paginate aggressively:

```http
GET https://api.karbonhq.com/v3/Contacts
Authorization: Bearer {token}
AccessKey: {key}
Accept-Encoding: gzip
```

This can reduce payload sizes by over 90% (e.g., 768KB → 56KB) and cut response times roughly in half.

### Avoid unnecessary `$expand`

`$expand=BusinessCards` adds an extra data fetch on the server side. Only include it when you need the expanded data.

### Batch with `$top` and `$filter`

Fetch larger pages (up to 100 items) rather than many small requests, and filter server-side to reduce total page count:

```http
GET /v3/WorkItems?$filter=PrimaryStatus eq 'In Progress'&$top=100
```

### Use webhooks for change detection

Instead of polling for changes on a schedule, subscribe to webhooks for the resources you care about. This eliminates the need for frequent polling entirely. See the [Webhooks guide](./07-webhooks.md).

## Summary

| Situation | Action |
|---|---|
| Received `429` | Wait `Retry-After` seconds, then retry with backoff |
| Building a sync/import | Add delays between requests, target ≤ 2 req/sec |
| Pulling large datasets | Use `$top=100`, filter aggressively, enable gzip |
| Monitoring for changes | Use webhooks rather than polling |
