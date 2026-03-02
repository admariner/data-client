# Data Client Protocol

> Reference documentation for constructing valid HTTP requests to the Stape Data Client — a Server-side Google Tag Manager (sGTM) Client template that claims incoming HTTP requests and maps them to [Common Event Data](https://developers.google.com/tag-platform/tag-manager/server-side/common-event-data).

## Overview

The Data Client acts as a universal HTTP-to-Event-Data mapper. It claims requests on a configurable path and parses the incoming data—from query parameters, request body, and HTTP headers—into a unified sGTM event model that downstream tags, variables, and triggers can consume.

Typically, requests are sent by the **Data Tag** (a Web GTM tag). However, any HTTP client (backend service, mobile app, IoT device, etc.) can send requests directly to the Data Client by following this protocol.

---

## Endpoint

### Default Path

```
/data
```

The Data Client claims requests to `/data` by default. Additional paths can be configured within the Client template settings (e.g., `/callback`, `/custom-endpoint`).

### Base URL

The full endpoint URL follows the pattern:

```
https://<your-sgtm-domain>/data
```

---

## Supported HTTP Methods

| Method    | Description                                                                                                 |
|-----------|-------------------------------------------------------------------------------------------------------------|
| `GET`     | Sends event data entirely via **query string parameters**. Response is a 1×1 tracking pixel by default.     |
| `POST`    | Sends event data via **request body** (JSON or form-urlencoded). Query parameters are merged as base data.  |
| `OPTIONS` | CORS preflight. Returns `200` with CORS headers. No event processing occurs.                                |

---

## Request Headers

### Automatically Consumed Headers

These HTTP headers are read by the Data Client and mapped into event data if the corresponding field is not already provided in the request payload.


| Header             | Maps To Event Data            | Description                                                                                    |
|--------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| `User-Agent`       | `user_agent`                  | Browser/client user agent string.                                                              |
| `Accept-Language`  | `language`                    | Parsed to extract the first two-character language code (e.g., `en` from `en-US,en;q=0.9`).    |
| `Content-Type`     | N/A / *(parsing behavior)*    | Determines body parsing: `application/json` (default) or `application/x-www-form-urlencoded`.  |
| `Origin`           | N/A / *(CORS response)*       | Echoed back in `Access-Control-Allow-Origin` response header.                                  |

### CORS Response Headers

Every response includes the following CORS headers:

| Response Header                      | Value                                                                                        |
|--------------------------------------|----------------------------------------------------------------------------------------------|
| `Access-Control-Max-Age`             | `600`                                                                                        |
| `Access-Control-Allow-Origin`        | *(echoed from request `Origin` header)*                                                      |
| `Access-Control-Allow-Methods`       | `GET,POST,PUT,DELETE,OPTIONS`                                                                |
| `Access-Control-Allow-Headers`       | `content-type,set-cookie,x-robots-tag,x-gtm-server-preview,x-stape-preview,x-stape-app-version` |
| `Access-Control-Allow-Credentials`   | `true`                                                                                       |

---

## Query String Parameters

All query parameters are merged into the base event model. Two special parameters provide compact payloads for `GET` requests:

| Parameter | Method   | Format         | Description                                                                                          |
|-----------|----------|----------------|------------------------------------------------------------------------------------------------------|
| `v`       | Any      | Number         | Protocol version. Used by the Data Tag for versioning (default: `2`). Passed through as event data.  |
| `event`   | Any      | String         | The event name (URL-encoded). Passed as a regular query parameter.                                   |
| `dtcd`    | `GET` only | JSON string  | A JSON-encoded object whose properties are expanded into the base event model.                       |
| `dtdc`    | `GET` only | Base64 string | A **Base64-encoded** JSON string whose properties are expanded into the base event model.            |
| *(other)* | Any      | String         | Any other query parameter is added directly to the event model as `key=value`.                       |

> [!IMPORTANT]
> The `dtcd` and `dtdc` parameters are **only processed on `GET` requests**. On `POST` requests, they are treated as regular query parameters.

### How `dtcd` / `dtdc` Work

When the Data Tag uses the `auto` request type and sends a `GET` request, it encodes all event data into a single `dtdc` query parameter:

```
GET /data?v=2&event=page_view&dtdc=eyJwYWdlX2xvY2F0aW9uIjoiaHR0cHM6Ly9leGFtcGxlLmNvbSJ9
```

The Data Client Base64-decodes `dtdc`, parses the result as JSON, and spreads the properties into the event model. Alternatively, `dtcd` works the same way but expects raw JSON (not Base64-encoded):

```
GET /data?v=2&event=page_view&dtcd={"page_location":"https://example.com"}
```

---

## Request Body Schema

The request body is processed on `POST` requests (and any method that produces a body). Two content types are supported:

### `application/json` *(recommended)*

The body must be **either**:

- **A single JSON object** — processed as one event.
- **A JSON array of objects** — each object is processed as a separate event (requires the *"Accept Multiple Events"* option enabled in the Data Client configuration).

#### Single Event

```json
{
  "event_name": "purchase",
  "page_location": "https://example.com/checkout",
  "page_referrer": "https://example.com/cart",
  "currency": "USD",
  "value": 99.99,
  "transaction_id": "TXN-12345",
  "client_id": "dcid.1.1706000000000.123456789",
  "user_id": "user-abc-123",
  "items": [
    {
      "item_id": "SKU-001",
      "item_name": "Widget",
      "price": 49.99,
      "quantity": 2
    }
  ]
}
```

#### Multiple Events (array)

```json
[
  { "event_name": "page_view", "page_location": "https://example.com" },
  { "event_name": "view_item", "page_location": "https://example.com/product/123" }
]
```
> [!NOTE]
> When a single object body is sent, the response body returns a single JSON object. When an array is sent (with *Accept Multiple Events* enabled), the response returns a JSON array. This behavior can be changed in the template UI under _Response Settings > Response Body_.

### `application/x-www-form-urlencoded`

The body is parsed using dot-notation expansion for nested objects and arrays:

```
event_name=purchase&page_location=https%3A%2F%2Fexample.com&items.0.item_id=SKU-001&items.0.price=49.99
```

This produces the equivalent of:

```json
{
  "event_name": "purchase",
  "page_location": "https://example.com",
  "items": [{ "item_id": "SKU-001", "price": 49.99 }]
}
```

---

## Auto-Generated Event Data

The Data Client automatically generates and attaches the following fields to **every** event model:

| Field              | Type     | Description                                                                        |
|--------------------|----------|------------------------------------------------------------------------------------|
| `timestamp`        | `number` | Unix timestamp in **seconds** (not milliseconds), generated at the time of request.|
| `unique_event_id`  | `string` | A unique identifier in the format `{timestamp_ms}_{random_9_digits}`.              |
| `client_id`        | `string` | Client identifier, resolved from multiple sources (see [Client ID Resolution](#client-id-resolution)). |

These fields can be overriden by the incoming request payload. Client ID is generated automatically only if the user checks the box "Always generate client_id parameter" in the template UI.

---

## Event Data Reference

The following table lists all Common Event Data fields that the Data Client recognizes and normalizes. You can send data using **any** of the listed property aliases; the Data Client will map them to the canonical sGTM event data key.

### Core Event Properties

| Event Data Key     | Type     | Accepted Aliases                           | Default Behavior                                                  |
|--------------------|----------|--------------------------------------------|-------------------------------------------------------------------|
| `event_name`       | `string` | `eventName`, `event`, `e_n`                | Defaults to `"Data"` if none of the aliases are provided.         |
| `client_id`        | `string` | `data_client_id`, `_dcid`                  | See [Client ID Resolution](#client-id-resolution).                |
| `user_id`          | `string` | `userId`                                   | —                                                                 |

### Page & Navigation

| Event Data Key     | Type     | Accepted Aliases                           | Default Behavior                                                  |
|--------------------|----------|--------------------------------------------|-------------------------------------------------------------------|
| `page_location`    | `string` | `pageLocation`, `url`, `href`              | —                                                                 |
| `page_referrer`    | `string` | `pageReferrer`, `referrer`, `urlref`       | —                                                                 |
| `page_hostname`    | `string` | `pageHostname`, `hostname`                 | —                                                                 |
| `page_path`        | `string` | `pagePath`                                 | —                                                                 |
| `page_title`       | `string` | `pageTitle`                                | —                                                                 |
| `page_encoding`    | `string` | `pageEncoding`                             | —                                                                 |

### Client & Device

| Event Data Key       | Type     | Accepted Aliases                         | Default Behavior                                                  |
|----------------------|----------|------------------------------------------|-------------------------------------------------------------------|
| `ip_override`        | `string` | `ip`, `ipOverride`                       | Falls back to the requester's remote IP address.                  |
| `user_agent`         | `string` | `userAgent`                              | Falls back to the `User-Agent` request header.                    |
| `language`           | `string` | —                                        | Falls back to the first 2 chars of the `Accept-Language` header.  |
| `screen_resolution`  | `string` | `screenResolution`                       | —                                                                 |
| `viewport_size`      | `string` | `viewportSize`                           | —                                                                 |

### Ecommerce

| Event Data Key   | Type       | Description                                                                                    |
|------------------|------------|------------------------------------------------------------------------------------------------|
| `value`          | `number`   | Transaction/event value. Alias: `e_v`. Auto-calculated from `items` if not provided.           |
| `currency`       | `string`   | Currency code. Auto-populated from `items[0].currency` if not set.                             |
| `transaction_id` | `string`   | Transaction identifier.                                                                        |
| `items`          | `array`    | Array of item objects. See [Items Schema](#items-schema).                                      |

When a single item exists in the `items` array, the following are auto-promoted to top-level event data:

| Event Data Key   | Source from `items[0]`  |
|------------------|-------------------------|
| `item_id`        | `item_id`               |
| `item_name`      | `item_name`             |
| `item_brand`     | `item_brand`            |
| `item_quantity`   | `quantity`              |
| `item_category`  | `item_category`         |
| `item_price`     | `price`                 |

#### Items Schema

Each object in the `items` array supports:

| Field           | Type     | Description                |
|-----------------|----------|----------------------------|
| `item_id`       | `string` | Item/product identifier.   |
| `item_name`     | `string` | Item/product name.         |
| `item_brand`    | `string` | Brand name.                |
| `item_category` | `string` | Product category.          |
| `price`         | `number` | Unit price.                |
| `quantity`       | `number` | Quantity (defaults to `1`).|
| `currency`      | `string` | Currency code.             |

### User Data

If the `user_data` object is **not** explicitly provided in the request body, the Data Client will attempt to construct it from individual flat fields. Below are the recognized aliases:

| `user_data` Field                | Accepted Aliases (flat request fields)                                             |
|----------------------------------|------------------------------------------------------------------------------------|
| `user_data.email_address`        | `userEmail`, `email_address`, `email`, `mail`                                      |
| `user_data.phone_number`         | `userPhoneNumber`, `phone_number`, `phoneNumber`, `phone`                          |
| `user_data.address.first_name`   | `userFirstName`, `first_name`, `firstName`, `name`                                 |
| `user_data.address.last_name`    | `userLastName`, `last_name`, `lastName`, `surname`, `family_name`, `familyName`    |
| `user_data.address.street`       | `street`                                                                           |
| `user_data.address.city`         | `city`                                                                             |
| `user_data.address.region`       | `region`, `state`                                                                  |
| `user_data.address.postal_code`  | `postal_code`, `postalCode`, `zip`                                                 |
| `user_data.address.country`      | `country`                                                                          |

> [!TIP]
> You can send a pre-structured `user_data` object directly in the body to skip all alias resolution:
> ```json
> {
>   "user_data": {
>     "email_address": "user@example.com",
>     "phone_number": "+15551234567",
>     "address": {
>       "first_name": "Jane",
>       "last_name": "Doe",
>       "city": "San Francisco",
>       "region": "CA",
>       "postal_code": "94103",
>       "country": "US"
>     }
>   }
> }
> ```

### UA Enhanced Ecommerce (Legacy)

The Data Client also supports the legacy `ecommerce` object format (Universal Analytics Enhanced Ecommerce). If an `ecommerce` key is present, the recognized actions are:

`detail`, `click`, `add`, `remove`, `checkout`, `checkout_option`, `purchase`, `refund`

For `purchase` actions with an `actionField`, the following are extracted:

| Event Data Key   | Source                                    |
|------------------|-------------------------------------------|
| `revenue`        | `ecommerce.purchase.actionField.revenue`  |
| `transaction_id` | `ecommerce.purchase.actionField.id`       |
| `affiliation`    | `ecommerce.purchase.actionField.affiliation` |
| `tax`            | `ecommerce.purchase.actionField.tax`      |
| `shipping`       | `ecommerce.purchase.actionField.shipping` |
| `coupon`         | `ecommerce.purchase.actionField.coupon`   |

---

## Client ID Resolution

The `client_id` is resolved using the following priority order:

1. **Request payload:** `client_id`, `data_client_id`, or `_dcid` from the event data.
2. **Cookie:** `_dcid` cookie value (set by the Data Client on previous requests).
3. **Temporary ID:** `_dcid_temp` field from the event data (used internally by the Data Tag to maintain identity before the server-set cookie is available). This field is **removed** from the final event model after use. It is only used when the *"Always generate client_id parameter"* option is enabled (default: **enabled**).
4. **Auto-generated:** `dcid.1.{timestamp_ms}.{random_9_digits}` — only when the *"Always generate client_id parameter"* option is enabled (default: **enabled**).

If none of the above produces a value and auto-generation is disabled, `client_id` is set to an empty string.

> [!IMPORTANT]
> For backend integrations, always pass your own `client_id` to maintain consistent user identity across requests.

---

## Consent State

The Data Tag can optionally send a `consent_state` object. The Data Client passes this through as-is in the event model:

```json
{
  "consent_state": {
    "ad_storage": true,
    "ad_user_data": false,
    "ad_personalization": false,
    "analytics_storage": true,
    "functionality_storage": true,
    "personalization_storage": true,
    "security_storage": true
  }
}
```

---

## Common Cookies

The Data Tag sends common third-party cookies in a `common_cookie` object. Backend integrations can replicate this behavior:

```json
{
  "common_cookie": {
    "_fbc": "fb.1.1706000000000.click_id",
    "_fbp": "fb.1.1706000000000.1234567890",
    "_gcl_aw": "GCL.1706000000000.gclid_value"
  }
}
```

Supported cookie names include: `_fbc`, `_fbp`, `_gtmeec`, `ttclid`, `_ttp`, `_epik`, `_scid`, `_scclid`, `taboola_cid`, `awin_awc`, `awin_sn_awc`, `awin_source`, `rakuten_site_id`, `rakuten_time_entered`, `rakuten_ran_mid`, `rakuten_ran_eaid`, `rakuten_ran_site_id`, `stape_klaviyo_email`, `stape_klaviyo_kx`, `stape_klaviyo_viewed_items`, `outbrain_cid`, `wg_cid`, `ps_id`, `uet_msclkid`, `_uetmsclkid`, `uet_vid`, `_uetvid`, `_dm_session_attributes`, `FPGCLAW`, `_gcl_aw`, `FPGCLGB`, `_gcl_gb`, `FPGCLAG`, `_gcl_ag`.

---

## Custom / Arbitrary Data

Any property sent in the request body or as a query parameter that is **not** listed above will still be included in the event model as-is. This makes the protocol fully extensible — you can send any custom key-value pairs and retrieve them in sGTM using the `getEventData` API.

```json
{
  "event_name": "form_submit",
  "form_id": "contact-form",
  "form_destination": "https://example.com/thank-you",
  "custom_dimension_1": "premium_user"
}
```

---

## Versioning

The Data Client uses the `v` query parameter as an informational protocol version marker. Currently:

- The value `v=2` is the canonical version sent by the Data Tag.
- The Data Client **does not** gate behavior on the `v` value — all versions are accepted and processed identically.
- The `v` parameter is passed through to the event model as-is.

There is **no** server-side version negotiation. The Data Client MUST accept requests with any `v` value (or no `v` parameter at all).

---

## Response Format

### `POST` Requests (status `200` / `201`)

The response body depends on the Data Client's *"Response Body"* configuration:

| Configuration                         | Response Body                                              |
|---------------------------------------|------------------------------------------------------------|
| `timestamp` *(default, recommended)*  | `{ "timestamp": 1706000000, "unique_event_id": "..." }`   |
| `eventData`                           | The full event model as JSON.                              |
| `empty`                               | No response body.                                          |

Response header: `Content-Type: application/json`.

### `GET` Requests (status `200` / `201`)

By default, returns a **1×1 transparent pixel** (via sGTM's `setPixelResponse` API). This can be overridden in the Data Client configuration to return a JSON body instead.

---

## Examples

### Example 1: Minimal `POST` Request

A simple page view event sent from a backend:

```bash
curl -X POST 'https://gtm.example.com/data?v=2&event=page_view' \
  -H 'Content-Type: application/json' \
  -H 'User-Agent: MyApp/1.0' \
  -d '{
    "event_name": "page_view",
    "client_id": "backend.1.1706000000.987654321",
    "page_location": "https://www.example.com/products",
    "page_title": "Our Products",
    "page_referrer": "https://www.google.com",
    "user_id": "user-12345"
  }'
```

### Example 2: Purchase Event with User Data

```bash
curl -X POST 'https://gtm.example.com/data?v=2&event=purchase' \
  -H 'Content-Type: application/json' \
  -d '{
    "event_name": "purchase",
    "client_id": "backend.1.1706000000.987654321",
    "page_location": "https://www.example.com/checkout/confirmation",
    "transaction_id": "TXN-2024-001",
    "value": 149.97,
    "currency": "USD",
    "items": [
      {
        "item_id": "SKU-A1",
        "item_name": "Running Shoes",
        "item_brand": "SportCo",
        "item_category": "Footwear",
        "price": 79.99,
        "quantity": 1
      },
      {
        "item_id": "SKU-B2",
        "item_name": "Athletic Socks (3-Pack)",
        "price": 34.99,
        "quantity": 2
      }
    ],
    "user_data": {
      "email_address": "jane.doe@example.com",
      "phone_number": "+15551234567",
      "address": {
        "first_name": "Jane",
        "last_name": "Doe",
        "city": "San Francisco",
        "region": "CA",
        "postal_code": "94103",
        "country": "US"
      }
    }
  }'
```

### Example 3: Minimal `GET` Request

A simple tracking pixel request with Base64-encoded data:

```bash
curl 'https://gtm.example.com/data?v=2&event=page_view&dtdc=eyJwYWdlX2xvY2F0aW9uIjoiaHR0cHM6Ly9leGFtcGxlLmNvbSIsInBhZ2VfdGl0bGUiOiJIb21lIn0='
```

Where the Base64 value `dtdc` decodes to:

```json
{ "page_location": "https://example.com", "page_title": "Home" }
```

### Example 4: Multiple Events in a Single Request

> Requires *"Accept Multiple Events"* enabled in the Data Client configuration.

```bash
curl -X POST 'https://gtm.example.com/data?v=2' \
  -H 'Content-Type: application/json' \
  -d '[
    {
      "event_name": "page_view",
      "client_id": "backend.1.1706000000.987654321",
      "page_location": "https://www.example.com"
    },
    {
      "event_name": "view_item",
      "client_id": "backend.1.1706000000.987654321",
      "page_location": "https://www.example.com/product/42",
      "items": [{ "item_id": "42", "item_name": "Widget", "price": 19.99 }]
    }
  ]'
```

### Example 5: Raw HTTP Request

```http
POST /data?v=2&event=page_view HTTP/1.1
Host: gtm.example.com
Content-Type: application/json
User-Agent: MyBackend/2.0
Accept-Language: en-US,en;q=0.9

{
  "event_name": "page_view",
  "client_id": "backend.1.1706000000.987654321",
  "page_location": "https://www.example.com",
  "page_title": "Home",
  "ip_override": "203.0.113.42"
}
```

---

## Data Merge Priority

When a request includes both query parameters and a body, data is merged in the following order (later sources take precedence):

1. **Auto-generated fields** (`timestamp`, `unique_event_id`).
2. **Query string parameters** (including expanded `dtcd`/`dtdc`).
3. **Request body fields** (body values **override** query parameters with the same key).

---

## Event Data Processing Pipeline

The following diagram illustrates the full processing pipeline from HTTP request to sGTM event model:

```
HTTP Request
    │
    ├──► Parse Query Parameters ──────────────────► Base Event Model
    │                                                      │
    ├──► Parse Request Body (JSON / form-urlencoded) ──────┤ (body overrides query params)
    │                                                      │
    │                                               ┌──────▼──────┐
    │                                               │ Event Model  │
    │                                               └──────┬──────┘
    │                                                      │
    ├──► addRequiredParameters ─── Set event_name ─────────┤
    │                              (default: "Data")       │
    │                                                      │
    ├──► addCommonParameters ──── Map aliases, ────────────┤
    │                              set ip_override,        │
    │                              user_agent, language,   │
    │                              build user_data, etc.   │
    │                                                      │
    ├──► addClientId ──────────── Resolve client_id ───────┤
    │                                                      │
    │                                               ┌──────▼──────┐
    │                                               │ Final Event  │
    │                                               │    Model     │
    │                                               └──────┬──────┘
    │                                                      │
    └──────────────────────────────────────────────► runContainer()
```
