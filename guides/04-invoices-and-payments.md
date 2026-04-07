# Retrieving Invoices and Recording Payments

This guide covers listing and retrieving invoices, then recording a manual payment against one.

## Listing Invoices

```http
GET https://api.karbonhq.com/v3/Invoices
Authorization: Bearer {token}
AccessKey: {key}
```

### Sorting Invoices

```http
# Most recent invoices first
GET /v3/Invoices?$orderby=InvoiceDate desc

# By invoice number
GET /v3/Invoices?$orderby=InvoiceNumber

# Recently created or updated
GET /v3/Invoices?$orderby=CreatedAt desc
GET /v3/Invoices?$orderby=UpdatedAt desc
```

### Paginating Through Results

Use `$top` and `$skip` or follow the `@odata.nextLink` in the response:

```http
GET /v3/Invoices?$top=50
```

Response:

```json
{
  "@odata.count": 214,
  "@odata.nextLink": "https://api.karbonhq.com/v3/Invoices?$skip=50",
  "value": [ ... ]
}
```

`@odata.count` is returned automatically. Always use the `@odata.nextLink` URL directly rather than manually incrementing `$skip`. See the [Pagination guide](./06-pagination.md) for details.

## Retrieving a Specific Invoice

```http
GET https://api.karbonhq.com/v3/Invoices/{InvoiceKey}
Authorization: Bearer {token}
AccessKey: {key}
```

The response includes the invoice total, subtotal, tax total, status, currency, payment due date, client details, and a `TaxLineItems` array breaking down individual tax components.

## Listing Existing Payments

To see payments already recorded:

```http
GET https://api.karbonhq.com/v3/Payments
Authorization: Bearer {token}
AccessKey: {key}
```

## Recording a Manual Payment

Once you have the `InvoiceKey`, record a payment against it:

```http
POST https://api.karbonhq.com/v3/ManualPayments
Authorization: Bearer {token}
AccessKey: {key}
Content-Type: application/json

{
  "InvoiceKey": "8fRnKqT3mXvP",
  "PaymentMethod": "EFT",
  "PaymentDate": "2026-03-31",
  "TotalAmount": 1500.00
}
```

**Required fields:**

| Field | Description |
|---|---|
| `InvoiceKey` | The key of the invoice being paid |
| `PaymentMethod` | One of: `Check`, `Cash`, `EFT`, `Credit Card`, `BankTransfer`, `DirectDebit`, `Other` |
| `PaymentDate` | ISO 8601 date (e.g., `2026-03-31`) |
| `TotalAmount` | The amount being paid (decimal) |

A successful `201 Created` response returns the new payment record including its `PaymentKey`.

## Reversing a Manual Payment

If a payment was recorded in error, reverse it using the `ReverseManualPayment` endpoint. You'll need the `PaymentKey` from the payment record and the date the reversal should be recorded:

```http
POST https://api.karbonhq.com/v3/ReverseManualPayment
Authorization: Bearer {token}
AccessKey: {key}
Content-Type: application/json

{
  "PaymentKey": "5hWjNbL2cYqR",
  "ReversalDate": "2026-04-01T00:00:00Z"
}
```

## Deleting a Manual Payment

Alternatively, delete it directly:

```http
DELETE https://api.karbonhq.com/v3/ManualPayments/{PaymentKey}
Authorization: Bearer {token}
AccessKey: {key}
```

## Subscribing to Invoice Webhooks

To be notified when invoices are created or updated without polling, subscribe to invoice webhooks:

```http
POST https://api.karbonhq.com/v3/WebhookSubscriptions
Authorization: Bearer {token}
AccessKey: {key}
Content-Type: application/json

{
  "WebhookType": "Invoice",
  "TargetUrl": "https://your-app.example.com/webhooks/karbon",
  "SigningKey": "your-secret-signing-key-min-16-chars"
}
```

Your endpoint will receive payloads in this format:

```json
{
  "ResourcePermaKey": "{InvoiceKey}",
  "ResourceType": "Invoice",
  "ActionType": "Updated",
  "TimeStamp": "2026-03-31T14:22:00Z"
}
```

Fetch the full invoice using the `ResourcePermaKey` when you receive a notification.

**Important:** You can only have one webhook subscription per type. If your endpoint fails to respond with HTTP 2xx ten times in a row, the subscription is automatically cancelled. Check for a `404` on your subscription to detect this:

```http
GET /v3/WebhookSubscriptions/Invoice
```

A `404` means the subscription was auto-cancelled and needs to be re-created.
