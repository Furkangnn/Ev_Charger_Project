# EV Charging Operations Platform

**Portfolio case study.** This repository documents the backend of an electric-vehicle charging platform used to operate chargers in Singapore and Malaysia. It is a simplified public write-up for recruiters and backend engineering roles.

This is not a source-code repository. It does not contain production code, infrastructure configuration, secrets, customer data, charger identifiers, or internal business rules.

## Role

Backend development for the operations platform, with ownership of:

- The real-time command path between the portal and charge points
- Charging-session lifecycle, from authorization through metering to settlement
- HTTP APIs used by the driver portal and the operator console
- Payment and prepaid-balance integration around a charging session

## Platform in brief

Drivers find a charger, authorize a session, and follow energy and cost while the vehicle charges. Operators see which charge points are online, which connectors are busy, and how sessions settle.

Charge points speak **OCPP 1.6J** over a persistent **WebSocket**. The central system is a **.NET** backend with **SQL Server** as the system of record. The portals call a **REST** API. Live session state is pushed to connected browsers with **SignalR**, so the UI does not poll for meter updates. Payments are handled through a prepaid balance: the driver tops up through a hosted payment page, a session reserves balance up front, and unused balance is released when the session ends.

The same backend model supports many charge points, more than one connector per charge point, and more than one market, with tariffs and payment options that differ by country.

## Documents

| Document | What it covers |
| --- | --- |
| [Architecture](docs/architecture.md) | Components, trust boundaries, and how live state reaches the UI |
| [OCPP command flow](docs/ocpp-flow.md) | Remote start and remote stop, including the sequence diagrams |
| [Payment flow](docs/payment-flow.md) | Top-up, reservation, settlement, and refund |
| [API overview](docs/api-overview.md) | Simplified resource model for drivers and operators |

## System shape

```mermaid
flowchart LR
  subgraph clients [Clients]
    Driver[Driver portal]
    Operator[Operator console]
  end

  subgraph backend [.NET backend]
    API[REST API]
    Sessions[Session service]
    Billing[Billing and wallet]
    Hub[SignalR hub]
    Ocpp[OCPP 1.6J gateway]
  end

  CP[Charge points]
  DB[(SQL Server)]
  Pay[Payment provider]

  Driver --> API
  Operator --> API
  API --> Sessions
  API --> Billing
  Sessions --> DB
  Billing --> DB
  Billing --> Pay
  CP <-->|WebSocket| Ocpp
  Ocpp --> Sessions
  Sessions --> Hub
  Ocpp --> Hub
  Hub --> Driver
  Hub --> Operator
```

Remote start and remote stop are specified in [docs/ocpp-flow.md](docs/ocpp-flow.md).

## What a driver sees

The driver portal is deliberately small. A session starts only after the charger is identified, typically by scanning the code on the unit. Balance is held as tokens. Top-up sends the driver to a hosted payment page and credits the balance after the provider confirms payment.

The gallery below is the original set of product photographs, plus a live socket test. A small OCPP 1.6J simulator stood in for a home charger: it accepted remote start, reported meter values, and accepted remote stop. The session settled at 0.11 kWh against a 0.10 token hold. The simulator was disconnected after the stop, so it did not stay online.

These photographs are the originals. They show the product as it was captured, including account and site details visible on screen.

<img alt="AC charger used for the socket test" src="docs/images/field-charger.jpeg" width="280">

<img alt="Driver home" src="docs/images/01-driver-home.jpeg" width="280">

<img alt="Station list" src="docs/images/02-driver-stations.jpeg" width="280">

<img alt="Station detail with start" src="docs/images/03-driver-station-detail.jpeg" width="280">

<img alt="Session history" src="docs/images/04-driver-history.jpeg" width="280">

<img alt="Wallet and payment methods" src="docs/images/05-driver-wallet.jpeg" width="280">

<img alt="Operator charge-point list" src="docs/images/06-operator-chargers.jpeg" width="280">

<img alt="Test session form after the simulator reported the cable connected" src="docs/images/07-test-start-form.png" width="320">

<img alt="Live session while meter values were arriving" src="docs/images/08-test-charging.png" width="320">

<img alt="Settled session in the operator console" src="docs/images/09-test-session-settled.png" width="640">

Operator console, station list. One charger was online during the socket test; the other stayed offline.

<img alt="Admin station list with one charger online" src="docs/images/10-admin-stations.png" width="640">

Operator map, zoomed out so both chargers sit in frame. The blue pin is the charger that was online for the test. The gray pin is an offline charger. The test charger has no saved location of its own, so a temporary map pin was set for this screenshot and removed afterward.

<img alt="Admin map with the online charger and an offline charger" src="docs/images/11-admin-map.png" width="640">

Desktop views of the same driver portal, with the account name removed:

<img alt="Driver portal home" src="docs/images/driver-portal-home.png" width="640">

<img alt="Scan required before a session" src="docs/images/driver-scan.png" width="640">

<img alt="Prepaid top-up" src="docs/images/driver-topup.png" width="640">

## What was hard about the backend

The interesting problems were concurrency and partial failure, not the happy-path message list.

- The charge point, not the browser, is the source of truth for energy. The API can request a start, but the session is not "charging" until the charge point confirms it.
- Meter values arrive on the WebSocket while the driver is watching a SignalR stream. Those two channels have to agree on one session record.
- A connector can accept only one active transaction. Remote start has to be rejected when the connector is offline, faulted, or already busy.
- Payment cannot wait for the final meter reading to take money, and it also cannot keep a hold forever if the charger never delivers energy. Reservation, capture, and refund are tied to OCPP outcomes.
- Many charge points stay connected at once. A dropped WebSocket has to age the charge point out of "available" without dropping in-flight session history.

## What this repository intentionally leaves out

- Application source, project files, and database scripts
- API keys, connection strings, and payment credentials
- Host names, IP addresses, and deployment topology
- Customer records, charger serial numbers, and tariff tables
- The production API contract and internal validation rules

The diagrams and API sketch in `docs/` are a teaching view of the design. They are not the production schema and not a client you can call.

## Stack

- .NET backend
- SQL Server
- OCPP 1.6J over WebSocket
- SignalR for live portal updates
- REST API for driver and operator actions
- Hosted payment page for prepaid top-up
- Multi-charge-point operations across Singapore and Malaysia
