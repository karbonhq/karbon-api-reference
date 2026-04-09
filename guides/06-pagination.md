---
slug: pagination
sortOrder: 6
seo:
  title: Pagination
  description: Retrieve full result sets from Karbon's list endpoints by requesting pages of up to 100 items and following the @odata.nextLink cursor in each response, combining pagination with $filter and $orderby for consistent, efficient data retrieval.
---

# Pagination

Most list endpoints return a maximum of 100 items per request. Use pagination to retrieve full result sets.

## How It Works

Request a page of results using `$top`:

```http
GET https://api.karbonhq.com/v3/Contacts?$top=100
Authorization: Bearer {token}
AccessKey: {key}
```

The response includes a `@odata.nextLink` when more results are available:

```json
{
  "@odata.count": 523,
  "@odata.nextLink": "https://api.karbonhq.com/v3/Contacts?$skip=100",
  "value": [
    { "ContactKey": "abc1", "FullName": "Jane Smith" },
    ...
  ]
}
```

## Following `nextLink`

Use `@odata.nextLink` directly as the URL for your next request — don't construct it manually. The server may include additional state in the URL beyond a simple `$skip` value:

```python
def fetch_all(initial_url, headers):
    results = []
    url = initial_url

    while url:
        response = requests.get(url, headers=headers).json()
        results.extend(response.get("value", []))
        url = response.get("@odata.nextLink")  # None when on the last page

    return results
```

```javascript
async function fetchAll(initialUrl, headers) {
  const results = [];
  let url = initialUrl;

  while (url) {
    const response = await fetch(url, { headers }).then(r => r.json());
    results.push(...(response.value ?? []));
    url = response["@odata.nextLink"] ?? null;
  }

  return results;
}
```

## Getting a Total Count

`@odata.count` is automatically included in all list responses — no additional parameter required. Use it for progress tracking or building pagination controls:

```json
{
  "@odata.count": 651,
  "@odata.nextLink": "https://api.karbonhq.com/v3/Contacts?$skip=100",
  "value": [ ... ]
}
```

## Page Size Limits

The maximum `$top` value is **100** for most endpoints. Requesting more than the maximum is silently capped — you won't receive an error, just 100 results.

## Combining Pagination with Filtering and Sorting

Pagination works alongside `$filter` and `$orderby`. Always include a stable `$orderby` when paginating to ensure consistent ordering across pages:

```http
GET /v3/Contacts?$filter=ContactType eq 'Individual'&$orderby=FullName&$top=100
```

Without a sort order, the page boundaries can shift between requests if records are added or modified mid-pagination.

## Incremental Sync with `LastModifiedDateTime`

When syncing records incrementally, sort by `LastModifiedDateTime desc` so the most recently changed records appear first. On each page, check whether the last record's `LastModifiedDateTime` is older than your previous sync timestamp — if so, you can stop paginating early rather than fetching the rest of the result set:

```http
GET /v3/Contacts?$orderby=LastModifiedDateTime desc&$top=100
```

You can also compare `LastModifiedDateTime` on individual records against what you have stored to skip writes for records that haven't actually changed.

## Practical Tips

- **Enable gzip compression** (`Accept-Encoding: gzip`) to reduce payload size and speed up each page fetch.
- **Filter first** — reducing the result set with `$filter` means fewer pages to fetch.
- **Don't construct `$skip` manually** — always follow `@odata.nextLink` directly to avoid subtle bugs when records are added or modified between pages.
