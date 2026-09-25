# API overview

The portals talk to a REST API. Charge points do not. They use the OCPP WebSocket described in [OCPP command flow](ocpp-flow.md).

The resources below are a **simplified public sketch** of the API surface. Paths, fields, and status codes are illustrative. They are not the production contract, and they are not enough to call a real environment.

Authentication is required on every route. Drivers see only their own profile, balance, and sessions. Operators see charge points and sessions in their scope. The sketches ignore paging, error bodies, and versioning.

## Driver

### Identify a charger

A session is not started from a free-text search in the happy path. The driver scans the charger. The API resolves that scan to a connector the caller is allowed to use.

```http
POST /api/chargers/resolve
```

Illustrative response: connector identity from the caller's point of view, availability, power type, and whether a start is currently allowed. No serial number is returned to the public case study, and none belongs in a client log.

### Start a session

```http
POST /api/sessions
```

The body names the connector and acknowledges the reserve. The API returns a session in `starting` when the charger accepts the remote start, or a failure when the connector is offline, busy, or the balance cannot cover the reserve. Transition to `charging` happens only after the charger sends its start transaction. Clients learn that from the session resource or from the live stream, not from this response alone.

### Read a session

```http
GET /api/sessions/{sessionId}
```

Returns lifecycle state, energy so far, estimated or final cost, and timestamps. Intermediate meter samples are not the whole history on this resource. The latest sample is enough for the screen. The final cost appears after stop.

### Stop a session

```http
POST /api/sessions/{sessionId}/stop
```

Asks the charger to stop. The response means the stop command was accepted or rejected. Final cost is available once the stop transaction has been settled.

### Live updates

Session and connector updates are pushed over SignalR to the authenticated browser. The push carries the same view as `GET /api/sessions/{sessionId}`: state, energy, and cost estimate. It is not a second source of billing truth.

### Wallet

```http
GET /api/wallet
POST /api/wallet/top-ups
GET /api/wallet/entries
```

`GET /api/wallet` returns posted balance and any active reserve. `POST /api/wallet/top-ups` creates a pending credit and returns a redirect to the hosted payment page. `GET /api/wallet/entries` is the driver's ledger: top-ups, reserves, captures, and refunds.

## Operator

Operators use the same session and wallet concepts, with a wider scope.

```http
GET /api/charge-points
GET /api/charge-points/{id}
GET /api/sessions?chargePointId={id}
POST /api/sessions/{sessionId}/stop
```

The charge-point list is the fleet view: online or offline, connector status, and whether a session is open. It is a projection of gateway state, not a live query to each device.

Operator stop uses the same command path as driver stop. The only difference is authorization scope.

Reporting and invoices are read models over settled sessions. They are not drawn in this sketch because they do not change the command flow.

## Resource states

Session state, as the API exposes it:

| State | Meaning |
| --- | --- |
| `starting` | Remote start was accepted. Waiting for the charger to open a transaction |
| `charging` | Start transaction received. Meter values may follow |
| `stopping` | Remote stop was accepted. Waiting for the stop transaction |
| `finished` | Stop transaction settled, or the session was closed with a refund |
| `failed` | Start never opened a transaction, or the charger rejected the command |

Connector availability is separate. A connector can be `available`, `occupied`, `faulted`, or `offline`. An occupied connector has an open session. Offline means the WebSocket is down or heartbeats have been missed.

## Design choices worth calling out in an interview

- Commands return as soon as the charger acknowledges them. Completion is a later state change, because OCPP splits "accepted" from the transaction message.
- Clients do not send meter readings. If a client could, the ledger would be writable by the browser.
- Top-up returns a redirect, not a card form. Payment details stay on the provider page.
- Stop is idempotent from the client's point of view. Repeating it after `finished` does not create a second charge.
- List endpoints for operators are projections. The API process that serves them is not the process that holds the charger sockets, so a slow report cannot stall heartbeats.

## Not included

Production paths, authentication scheme, claim names, rate limits, validation messages, and any identifier format are omitted.
