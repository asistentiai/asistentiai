# AsistentiAI Connector Specification (v1)

This page explains how to connect your ERP, CRM or custom platform to AsistentiAI. The connection is a thin HTTP/JSON bridge for order status, stock and, in a later version, order creation.

> Source of truth: [asistentiai.com/en/integrations](https://asistentiai.com/en/integrations). If this page and the website ever differ, the website wins.

## Overview

AsistentiAI answers your business's customers 24/7. To give them live facts such as order status and stock, the agent needs to read them from your system.

You don't need a separate integration for every ERP. Instead, you implement **one thin HTTP/JSON bridge** that follows this specification. We can also build it for you as a service. Well-known platforms (WooCommerce, WordPress) already have ready-made connectors.

There are two directions:

1. **Connector (we call your system).** AsistentiAI calls the bridge you implement, for example to look up an order or a SKU while it talks to a customer. See [Endpoints you implement](#endpoints-you-implement).
2. **AsistentiAI API (your system calls us).** Your ERP or shop pushes data, such as stock, to AsistentiAI with an API key. See [AsistentiAI API](#asistentiai-api-your-system-sends-us-data).

## Principles

- **Read-only in v1.** AsistentiAI changes nothing in your system. Order creation (`POST /orders`) is reserved for v1.1 and is enabled only with your explicit approval.
- **Customer privacy.** Your bridge returns an order's status only when the order number matches the phone or email the customer gives. You do this check on your side.
- **Every call comes from AsistentiAI's servers.** Each call is authenticated with a token or an HMAC signature and has a **12-second timeout**.
- **Approval before activation.** A connection goes live only after the AsistentiAI platform approves it. A connection the platform sets up itself goes live immediately.

## Authentication

There are two methods. You choose one when you request the connection in the dashboard.

### 1. Bearer token

```http
Authorization: Bearer <token>
```

### 2. HMAC-SHA256 signature (recommended)

Each request carries three headers:

```text
X-AAI-Key:       <key id>
X-AAI-Timestamp: <unix seconds>
X-AAI-Signature: hex( HMAC-SHA256( secret, timestamp + "." + METHOD + "." + pathWithQuery + "." + sha256hex(body) ) )
```

The signed message joins four parts with `.`:

1. the timestamp (the same value as `X-AAI-Timestamp`)
2. the HTTP method in upper case
3. the request path with its query string
4. `sha256hex(body)`, the hex SHA-256 of the request body (for an empty body, `sha256hex("")`)

Here is an example for `GET /orders/1234?phone=0691234567`, which has an empty body:

```text
message = "1727100000.GET./orders/1234?phone=0691234567." + sha256hex("")
```

**Reject requests whose timestamp is older than 5 minutes.**

## Endpoints you implement

All paths are relative to the base URL you give us, for example `https://erp.company.com/asistentiai`. Responses are always JSON.

### `GET /health`

The connection test. AsistentiAI calls it when you add the connection.

```jsonc
200  { "ok": true, "capabilities": ["orders.read", "stock.read"] }
// "capabilities" (optional): what your bridge implements
```

### `GET /orders/{number}?phone=…&email=…`

This returns the order **only** if the phone (last 8 digits) or the email matches the order.

```jsonc
200  { "found": true, "verified": true,
       "order": {
         "number": "1234",
         "status": "processing",            // pending | processing | on-hold | completed | cancelled | refunded | failed | shipped | delivered | returned
         "statusLabel": "Being prepared",   // optional, in the customer's language; if missing, we translate the status code
         "date": "2026-09-20",
         "total": "1700", "currency": "ALL",
         "items": [ { "name": "Electric blower 48V", "qty": 1 } ],
         "shipping": "Courier", "tracking": "AL123456789", "city": "Fier"
       } }

200  { "found": true, "verified": false }   // the order exists but the contact doesn't match (or respond 403)

404  { "found": false }
```

`status` takes one of these values: `pending`, `processing`, `on-hold`, `completed`, `cancelled`, `refunded`, `failed`, `shipped`, `delivered`, `returned`.

You can add `statusLabel` to control the exact wording shown to the customer. If you leave it out, AsistentiAI translates the status code.

### `GET /stock/{sku}`

This endpoint is optional.

```jsonc
200  { "sku": "A2105", "name": "Electric blower 48V", "inStock": true, "quantity": 12, "price": "1400 L",
       "locations": [ { "name": "Tiranë", "qty": 7 }, { "name": "Durrës", "qty": 3 }, { "name": "Fier", "qty": 2 } ] }
// "locations" (optional): stock per store / warehouse

404  { "found": false }
```

### `POST /orders` (v1.1, coming soon)

This is reserved for order creation. It is enabled only with your explicit approval.

```jsonc
// request
{ "customer": { "name": "…", "phone": "…", "email": "…", "address": "…", "city": "…" },
  "items": [ { "sku": "A2105", "qty": 1 } ],
  "note": "…", "source": "asistentiai", "conversationId": "…" }

// response
201  { "number": "1235", "status": "pending" }
```

## Error codes

| Code | Meaning | What AsistentiAI does |
|---|---|---|
| `401` | Invalid authentication | The agent tells the customer that the team will check. |
| `404` | The order or product does not exist | |
| `429` / `5xx` | Temporarily unavailable | AsistentiAI does not retry within the same conversation, and it records the error in the connection's events. |

## AsistentiAI API: your system sends us data

This is the reverse direction. Instead of AsistentiAI querying your system, your system (an ERP or a shop) sends the data to us itself.

The AsistentiAI team creates an **API key** for your business and gives it to you. The key belongs to that business only and has only the capabilities granted to it. It can be revoked at any time.

```http
Authorization: Bearer <your API key>
```

### `GET /api/v1/ping`

The connection test. It shows which business the key belongs to and which capabilities it has.

```jsonc
GET https://asistentiai.com/api/v1/ping

200  { "ok": true, "api": "asistentiai/v1", "business": "Example Store", "key": "Store ERP", "scopes": ["stock.write"] }
```

### `POST /api/v1/stock` (capability `stock.write`)

This sends stock per SKU, optionally per store or warehouse. You can send only the changes or the whole list, as often as you like. AsistentiAI keeps the latest state, and it shows up immediately in the conversations dashboard when your team creates an order.

```jsonc
POST https://asistentiai.com/api/v1/stock
{ "items": [
    { "sku": "A4543", "quantity": 221,
      "locations": [ { "name": "Tiranë", "qty": 221 }, { "name": "Durrës", "qty": 45 } ] },
    { "sku": "A0099", "inStock": false }
] }

200  { "ok": true, "saved": 2, "rejected": [] }   // up to 1000 items per request
```

To read back what was stored:

```http
GET https://asistentiai.com/api/v1/stock?sku=A4543
```

Errors:

| Code | Meaning |
|---|---|
| `401` | Missing or revoked key |
| `403` | The key lacks this capability |
| `429` | More than 120 requests per minute |

## Sandbox

AsistentiAI runs a reference implementation of this specification with fictional data. You can use it to compare your bridge and to try the connection in the dashboard without a real system.

```text
Base URL: https://asistentiai.com/api/connector-demo

Orders:  1234 (phone 0691234567 / email demo@asistentiai.com) → shipped, tracking AL123456789
         1235 (phone 0682223344 / email klienti@shembull.al)  → being prepared
Stock:   A2105 → in stock · B2201 → out of stock
```

The sandbox's demo bearer token and its HMAC key id and secret are published in the **Sandbox** section of [asistentiai.com/en/integrations](https://asistentiai.com/en/integrations). Here is an example call, with the token from that page in place of the placeholder:

```bash
curl -H "Authorization: Bearer <sandbox token>" \
  "https://asistentiai.com/api/connector-demo/orders/1234?phone=0691234567"
```

## How a connection is activated

1. **Implement** `/health`, `/orders/{number}` and, optionally, `/stock/{sku}` in your system, using one of the authentication methods.
2. **Request the connection.** The business Administrator goes to **Data connections** → **Internal system / ERP** in the dashboard and enters the base URL and the token or HMAC key. AsistentiAI calls `/health` to test it.
3. **Approval.** The AsistentiAI platform approves the connection and enables the capabilities (order status, stock). From then on, the agent uses them automatically in conversations.

## Help

If you have questions or need help with the implementation, write to the AsistentiAI team from the dashboard. We can also build the bridge for you as a service.

---

Part of the public documentation of [AsistentiAI](https://asistentiai.com/en). © AsistentiAI
