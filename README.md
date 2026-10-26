# Pitobi Search API

Go API for **KLM**, **Air France**, and **Air Cairo** search. Akamai / Incapsula is handled with Hyper Solutions. Airline traffic must go through **sticky residential proxies** (Oxylabs or Bright Data).

Run **one** API process. Each process has its own session pool. Extra replicas burn Hyper and proxy sessions and can break KLM / Air France round trips (offer IDs are bound to the session that created them).

## Environment

Copy `.env.example`. Do not commit real values.

| Variable | Required | Purpose |
| --- | --- | --- |
| `HYPER_API_KEY` | yes | Hyper Solutions API key |
| `HTTP_PROXY` | yes in production | Sticky proxy URL. Keep the literal `SESSIONID` in the username — the API replaces it per browser session |
| `PITOBI_API_TOKEN` | recommended when hosted | Protects every search POST. `GET /health` stays public |
| `PORT` | no (`80`) | HTTP listen port |
| `PITOBI_LOG_LEVEL` | no (`info`) | `debug` logs anti-bot steps |
| `PITOBI_POOL_MAX` | no (`16`) | Max bootstrapped sessions per site. Use `4` in production |
| `PITOBI_CHROME_MAJOR` | no (`152`) | Chrome TLS / UA major |

### Proxy URL format

Oxylabs residential:

```text
http://customer-USERNAME-cc-nl-sessid-SESSIONID:PASSWORD@pr.oxylabs.io:7777
```

Bright Data residential:

```text
http://brd-customer-CUSTOMERID-zone-ZONE-session-SESSIONID:PASSWORD@brd.superproxy.io:33335
```

`SESSIONID` is a placeholder. Each bootstrapped session gets its own value so Akamai and Incapsula see a stable IP. A rotating IP invalidates the cookies. Sticky session length should be **at least 30 minutes**.

## Run

### Local

```bash
cp .env.example .env
# fill HYPER_API_KEY, HTTP_PROXY, PITOBI_API_TOKEN
set -a && source .env && set +a
go run ./cmd
```

### Docker

```bash
docker build -t pitobi .
docker run -p 80:80 \
  -e HYPER_API_KEY \
  -e HTTP_PROXY \
  -e PITOBI_API_TOKEN \
  pitobi
```

### Tests

Live tests hit the real sites and are skipped unless `HYPER_API_KEY` is set.

```bash
go vet ./cmd ./klm ./airfrance ./aircairo ./internal
go test ./cmd ./klm ./airfrance ./aircairo
```

## Endpoints

Send `Authorization: Bearer <PITOBI_API_TOKEN>` or `X-API-Key: <PITOBI_API_TOKEN>` when the token is set.

For a KLM or Air France **round trip**, call **outbound → intermediate → inbound** in that order against the **same API process**. Offer IDs from outbound only work on that session. If a date has no direct flight, skip intermediate and inbound.

### GET /health

Load-balancer check. No token. Returns `{"status":"ok"}`.

### POST /klm  (aliases: `/outbound`, `/klm/outbound`)

KLM outbound (1 adult, economy). Omit `returnDate` for one-way. Returns `data.flightOffers.offers[]`. `resources.upsellOffers` is the input for `/klm/intermediate`.

```bash
curl -X POST http://localhost/klm \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $PITOBI_API_TOKEN" \
  -d '{
    "origin": "AMS",
    "destination": "CPT",
    "departureDate": "2026-12-19",
    "returnDate": "2026-12-26"
  }'
```

### POST /klm/intermediate  (alias: `/intermediate`)

KLM branded fares for one outbound. `resourceId` is `flightProducts[].resources.upsellOffers`.

```bash
curl -X POST http://localhost/klm/intermediate \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $PITOBI_API_TOKEN" \
  -d '{"resourceId": "/a/v3/flight-products/q0Zqw.../upsell-offers"}'
```

### POST /klm/inbound  (alias: `/inbound`)

KLM return flights. `availableOfferResourceID` comes from `/klm/intermediate`. `origin`, `destination`, and `outboundDate` can be omitted when inbound runs on the same process as the outbound that produced the offer.

```bash
curl -X POST http://localhost/klm/inbound \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $PITOBI_API_TOKEN" \
  -d '{
    "availableOfferResourceID": "/a/v3/available-offers?d=q0aZ...",
    "departureDate": "2026-12-26"
  }'
```

### POST /airfrance

Air France (`wwws.airfrance.fr`). Same body as `/klm`. Omit `returnDate` for one-way. Then `/airfrance/intermediate` and `/airfrance/inbound`, same as KLM.

```bash
curl -X POST http://localhost/airfrance \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $PITOBI_API_TOKEN" \
  -d '{
    "origin": "CDG",
    "destination": "AMS",
    "departureDate": "2026-12-19",
    "returnDate": "2026-12-26"
  }'
```

### POST /aircairo

Air Cairo search body is forwarded as-is. Anti-bot and session handling stay in the API.

```bash
curl -X POST http://localhost/aircairo \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $PITOBI_API_TOKEN" \
  -d '{
    "commercialFareFamilies": ["SMEUROPE", "SMMEAST"],
    "itineraries": [{
        "originLocationCode": "CGN",
        "destinationLocationCode": "HRG",
        "departureDateTime": "2026-12-19T00:00:00.000",
        "isRequestedBound": true
    }],
    "travelers": [{"passengerTypeCode": "ADT"}],
    "searchPreferences": {"showMilesPrice": false}
}'
```

## Session handling

- KLM / Air France: reuse a session for up to 8 searches or 24 minutes. A round trip (outbound + intermediate + inbound) stays on the same session. A blocked session is dropped; the next **outbound** starts clean. Do not retry intermediate/inbound on a new process with an old offer ID.
- Air Cairo: Reese84 is renewed on the same session; OAuth is cached per route/date. Sessions rotate before Incapsula `edet=15`.
- Collector concurrency should stay at **1** per airline API.
