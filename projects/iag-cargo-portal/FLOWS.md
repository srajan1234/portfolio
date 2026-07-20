# Flows — IAG Cargo Portal

> **Sanitized architecture case study — internal identifiers, account IDs and schema omitted.**

Six representative end-to-end flows, each with a sequence diagram and short explanation.

---

## 1. SSO Bridge Login → JWT Issue

The user authenticates through the legacy portal login. Rather than continue with a container cookie session, the platform exchanges that trusted signal for a first-class JWT minted by the Token Service.

```mermaid
sequenceDiagram
    participant U as Browser (Angular)
    participant L as Legacy Portal Login
    participant T as Token Service
    participant D as DynamoDB (sessions)
    participant P as Portal Database API
    participant O as Legacy Oracle

    U->>L: Authenticate (existing credentials)
    L-->>U: Trusted login signal
    U->>T: POST /token/issue (trusted signal)
    T->>P: Resolve user / role (private API)
    P->>O: Read user + role (Transit Gateway)
    O-->>P: User + role
    P-->>T: Identity details
    T->>D: Create session (enforce max 5, set TTL)
    T-->>U: JWT access cookie (15 min) + refresh token
```

The Token Service resolves user and role from Oracle via the private abstraction service, creates a session in DynamoDB (evicting the oldest if the 5-session cap is exceeded), and returns a 15-minute access JWT plus a refresh token.

---

## 2. Per-Request Auth via Lambda Authorizer

Every protected request is authorized statelessly before it reaches a business Lambda.

```mermaid
sequenceDiagram
    participant U as Browser
    participant G as API Gateway (Public REST)
    participant A as Lambda Authorizer (Token Service)
    participant S as Target Service (e.g. Dashboard BFF)

    U->>G: GET /api/v1/dashboard/widgets (JWT cookie)
    G->>A: Authorize (token)
    A->>A: Verify signature, issuer, expiry, claims
    alt Valid
        A-->>G: Allow policy + context (sub, role, scope)
        G->>S: Forward request + identity context
        S-->>U: 200 Response
    else Invalid / expired
        A-->>G: Deny policy
        G-->>U: 401 Unauthorized
    end
```

The authorizer verifies the JWT independently and returns an **Allow** or **Deny** IAM policy. On Allow it passes an identity context (subject, role, scope) to the target service; on Deny the request is rejected at the gateway.

---

## 3. Token Refresh (~14 min)

The SPA refreshes silently before the 15-minute access token expires, keeping the session alive without a re-login.

```mermaid
sequenceDiagram
    participant U as Browser (refresh timer ~14 min)
    participant T as Token Service
    participant D as DynamoDB (refresh tokens)

    U->>T: POST /token/refresh (refresh token)
    T->>D: Validate refresh token + session
    alt Valid
        T->>D: Rotate refresh token, extend TTL
        T-->>U: New JWT access cookie (15 min) + new refresh token
    else Invalid / revoked / over session cap
        T->>D: Invalidate session
        T-->>U: 401 -> force re-login
    end
```

Refresh **rotates** the refresh token and issues a fresh access JWT. If the refresh token is invalid, revoked, or the session is no longer valid, the user is forced back to login.

---

## 4. Asynchronous Flight Search

Flight search fans out to several slow backends and routinely exceeds the API Gateway ~29s HTTP timeout, so it runs out-of-band. A WebSocket carries only a small "complete + search_id" signal; the full result is fetched over REST from a DynamoDB cache.

```mermaid
sequenceDiagram
    participant U as Browser
    participant W as WebSocket API
    participant G as Public REST API
    participant Q as SQS FIFO
    participant S as eBooking Search BFF (Java)
    participant X as SOAP / Rules / Spot / Portal DB API
    participant C as DynamoDB (search cache)

    U->>W: Connect (establish channel)
    U->>G: POST /api/v1/flights/search
    G->>Q: Enqueue search request (search_id)
    G-->>U: 202 Accepted (search_id)
    Q->>S: Deliver search job
    par Parallel fan-out
        S->>X: Reservation availability (SOAP)
        S->>X: Rules engine
        S->>X: Spot pricing
        S->>X: Legacy data (Portal DB API)
    end
    X-->>S: Partial results
    S->>C: Cache assembled results (search_id)
    S->>W: Push "search complete + search_id"
    W-->>U: Completion signal
    U->>G: GET /api/v1/flights/search/{search_id}
    G->>S: Fetch cached results
    S->>C: Read by search_id
    C-->>S: Full results
    S-->>U: 200 Full search results
```

The REST call returns immediately (202 + `search_id`) after enqueuing to SQS FIFO. The Search BFF fans out in parallel, assembles and caches results in DynamoDB, then signals completion over the WebSocket. The browser fetches the full payload via REST — no HTTP timeout, and the socket stays lightweight.

---

## 5. Booking Submit → EventBridge Event Publish

A booking submit responds as soon as the reservation is placed; all side effects are published as an event and handled asynchronously.

```mermaid
sequenceDiagram
    participant U as Browser
    participant G as Public REST API
    participant S as eBooking Search BFF (Java)
    participant R as Reservation system (SOAP)
    participant B as EventBridge Bus
    participant A as S3 analytics archive

    U->>G: POST /api/v1/bookings/submit
    G->>S: Submit booking
    S->>R: Place reservation (SOAP)
    R-->>S: Reservation confirmed
    S->>B: Publish BookingEvent (eventType inside payload)
    S->>A: Archive request/response
    S-->>U: 200 Booking confirmed (fast)
```

Once the reservation is confirmed, the Search BFF publishes a single `BookingEvent` (with the routing key inside the payload) to EventBridge and returns immediately. Confirmation email, audit and pre-alert evaluation happen downstream — the response never waits on them.

---

## 6. Event-Driven Notification with Retry → DLQ

The Notification Service consumes booking events and performs the deferred side effects, with automatic retry and a dead-letter queue for poison messages.

```mermaid
sequenceDiagram
    participant B as EventBridge Bus
    participant N as Notification Service (Java)
    participant M as SMTP relay
    participant O as Legacy Oracle (audit)
    participant Q as SQS DLQ

    B->>N: Deliver BookingEvent
    N->>N: Branch on eventType (confirm / cancel / pre-alert)
    N->>M: Send transactional email
    N->>O: Write audit record
    N->>N: Evaluate pre-alert business rules
    alt Success
        N-->>B: Ack (complete)
    else Failure
        N->>N: Retry (bounded attempts)
        N->>Q: Route to dead-letter queue after retries
    end
```

The consumer branches on the payload's `eventType`, sends the transactional email over SMTP, writes an audit record to Oracle, and evaluates pre-alert rules. Transient failures are retried a bounded number of times; anything still failing is routed to the SQS DLQ for inspection and replay, so no booking event is silently lost.
