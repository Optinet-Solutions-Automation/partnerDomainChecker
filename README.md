# partnerDomainCheckerAPI

A Node.js Express API that traces HTTP redirects hop-by-hop and checks whether the final destination domain is a known partner domain.

## Features

- Follows all HTTP redirects (301, 302, 303, 307, 308) manually using Node.js built-in `http`/`https` modules
- Stops early once a partner domain is detected — no unnecessary hops
- Subdomain-aware matching: `casino.spinjo.com` matches `spinjo.com`
- Retry logic: retries the full trace up to `MAX_RETRIES` times if no partner domain is found
- Optional proxy support with authentication (HTTP proxy + HTTPS CONNECT tunnel)
- Realistic browser `User-Agent` and headers on every request
- Configurable via `.env`

## Setup

```bash
# 1. Install dependencies
npm install

# 2. Copy the example env file and configure it
cp .env.example .env

# 3. Start the server
npm start
```

Server listens on `http://localhost:3000` by default (or `PORT` from `.env`).

## API

### `GET /trace`

Traces redirects from the given URL and checks if the final domain is a partner domain.

**Query Parameters**

| Parameter | Required | Description |
|-----------|----------|-------------|
| `url` | Yes | The URL to trace (URL-encoded) |
| `proxy` | No | Proxy URL to route requests through (URL-encoded) |

**Success Response**

```json
{
  "input_url": "https://short.example.com/abc123",
  "final_url": "https://casino.spinjo.com/landing?btag=xyz",
  "status": 200,
  "hops": 3,
  "attempts": 1,
  "is_partner": true
}
```

| Field | Description |
|-------|-------------|
| `input_url` | The original URL passed in |
| `final_url` | The last URL after all redirects (or the partner URL if stopped early) |
| `status` | Final HTTP status code (`null` if partner was detected on the starting URL) |
| `hops` | Total number of redirects followed |
| `attempts` | Number of trace attempts made (due to retry logic) |
| `is_partner` | `true` if the final domain matches a configured partner domain |
| `js_rendered` | `true` if the ScrapingBee fallback was used to resolve the URL |
| `proxy` | The proxy used (only present if a proxy was supplied) |

**Error Response**

```json
{
  "error": "Descriptive error message",
  "input_url": "https://short.example.com/abc123"
}
```

## Example curl Commands

### Direct trace (no proxy)

```bash
curl "http://localhost:3000/trace?url=https%3A%2F%2Fshort.example.com%2Fabc123"
```

### Trace with a proxy (no auth)

```bash
curl "http://localhost:3000/trace?url=https%3A%2F%2Fshort.example.com%2Fabc123&proxy=http%3A%2F%2F192.168.1.100%3A8080"
```

### Trace with an authenticated proxy

```bash
curl "http://localhost:3000/trace?url=https%3A%2F%2Fshort.example.com%2Fabc123&proxy=http%3A%2F%2Fuser%3Apassword%40proxy.example.com%3A8080"
```

### Geographic proxy example (e.g. residential proxy)

```bash
curl "http://localhost:3000/trace?url=https%3A%2F%2Fshort.example.com%2Fabc123&proxy=http%3A%2F%2Fuser%3Apass%40geo.proxy.example.com%3A10000"
```

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `3000` | Server port |
| `PARTNER_DOMAINS` | _(empty)_ | Comma-separated list of partner domains |
| `MAX_HOPS` | `20` | Max redirect hops per attempt |
| `MAX_RETRIES` | `3` | Max retry attempts if partner domain not found |
| `RETRY_DELAY_MS` | `1000` | Delay between retries (ms) |
| `TIMEOUT_MS` | `10000` | Per-request timeout (ms) |
| `SCRAPINGBEE_API_KEY` | _(empty)_ | ScrapingBee API key for JS rendering fallback |
| `ENABLE_JS_FALLBACK` | `false` | Set to `true` to enable ScrapingBee fallback |

## ScrapingBee JS Rendering Fallback

When the normal hop-by-hop trace is blocked (status `403`, `429`, `407`, `503`, or a connection-level failure), the API can fall back to ScrapingBee, which renders the URL in a real Chromium browser — bypassing Cloudflare and JS-driven redirects.

**How it works:**
1. Normal trace runs first (fast, no API credits used)
2. If all `MAX_RETRIES` attempts result in a blocked status → ScrapingBee is called once
3. ScrapingBee returns the final resolved URL via the `spb-resolved-url` response header
4. That URL is checked against partner domains as usual

**To enable:**
1. Sign up at [scrapingbee.com](https://www.scrapingbee.com) and get your API key
2. Set `SCRAPINGBEE_API_KEY=your_key` and `ENABLE_JS_FALLBACK=true` in your `.env`

**Note:** ScrapingBee uses API credits per request. The fallback only fires when the direct trace is blocked, keeping credit usage minimal.

## Proxy Routing

- **HTTP targets**: the full URL is forwarded directly to the proxy as the request path
- **HTTPS targets**: a `CONNECT` tunnel is opened to the proxy, then TLS is negotiated inside that tunnel with the real target

## Limits

- Maximum **20 hops** per attempt before stopping (configurable via `MAX_HOPS`)
- **10 second timeout** per individual request (configurable via `TIMEOUT_MS`)
- Maximum **3 retry attempts** per trace (configurable via `MAX_RETRIES`)
