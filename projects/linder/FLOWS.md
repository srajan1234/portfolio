# Linder — Key Flows

Six end-to-end flows spanning the mobile app, the gateway, and the backend services. Each shows the sequence and a short walkthrough of what happens and why.

## 1. Authentication & Onboarding

```mermaid
sequenceDiagram
    participant App as Mobile App
    participant GW as API Gateway
    participant US as user-service
    participant SS as SecureStore

    App->>GW: POST /api/auth/register (email, password, role)
    GW->>US: forward (public path, no JWT)
    US->>US: hash password (BCrypt), create user, generate OTP
    US-->>App: 201 Created (verification required)
    App->>GW: POST /api/auth/verify-otp (email, code)
    GW->>US: forward
    US->>US: validate OTP, mark is_verified
    US-->>App: JWT access + refresh token
    App->>SS: persist tokens (expo-secure-store)
    App->>GW: PUT /api/profiles/{role}/me (Bearer JWT)
    GW->>US: validate JWT, inject X-User-Id/Role
    US-->>App: profile saved, completion %
```

**Walkthrough.** Registration is a public route, so the gateway forwards it without a token. user-service BCrypt-hashes the password, stores the account, and issues an email OTP. After the user submits the OTP, the account is verified and a JWT access token plus refresh token are returned; the app persists them in `expo-secure-store` (via Redux Persist). The user then selects a role (seeker or recruiter) and completes the corresponding profile — the first authenticated call, now carrying the bearer token that the gateway validates and enriches with identity headers.

## 2. Swipe → Mutual Match

```mermaid
sequenceDiagram
    participant App as Mobile App
    participant GW as API Gateway
    participant MS as match-service
    participant MG as MongoDB
    participant RMQ as RabbitMQ

    App->>GW: POST /api/swipes (targetId, type, direction=RIGHT)
    GW->>MS: forward (+ X-User-Id, X-User-Role)
    MS->>MG: upsert Swipe (unique swiperId+swipedId)
    MS->>MG: look for complementary swipe (same job)
    alt both sides swiped RIGHT
        MS->>MG: create Match (seeker, recruiter, job, company)
        MS->>MG: flag both swipes matched + matchId
        MS->>RMQ: publish match.created -> match.exchange
        MS-->>App: { matched: true, match }
    else no reciprocal swipe yet
        MS-->>App: { matched: false }
    end
```

**Walkthrough.** Every swipe is recorded idempotently (unique index on swiper + swiped). On a right/up swipe, match-service checks for the complementary swipe — a seeker's interest in a job paired with a recruiter's interest in that seeker *for the same job*. When both exist, a `Match` is created, both swipes are back-filled with the `matchId`, and a `match.created` event is published to the RabbitMQ topic exchange. Publishing (rather than calling downstream services directly) keeps the swipe response fast and lets chat and notification services react independently.

## 3. Real-Time Chat over STOMP/WebSocket

```mermaid
sequenceDiagram
    participant A as Sender App
    participant GW as API Gateway
    participant CS as chat-service
    participant MG as MongoDB
    participant RMQ as RabbitMQ
    participant B as Recipient App

    Note over CS: On match.created, a Conversation was pre-created
    A->>GW: STOMP CONNECT /ws (SockJS)
    GW->>CS: proxy lb:ws://chat-service
    A->>CS: SEND /app/chat.send (tempId, content)
    CS->>MG: persist Message, update lastMessage + unread
    CS-->>A: confirmation (tempId -> saved message)
    CS-->>B: /topic/chat.{id} new message
    CS->>RMQ: publish message.sent
    A->>CS: SEND /app/chat.typing / chat.read
    CS-->>B: typing indicator / read receipt
```

**Walkthrough.** Because a conversation is created the moment a match forms, chat is ready as soon as the match appears. The app opens a STOMP session over SockJS; the gateway proxies the upgrade to chat-service (`lb:ws://`). Outbound messages carry a client `tempId` so the optimistic UI can reconcile the persisted message when the confirmation returns. chat-service stores the message, updates the conversation's last-message summary and unread counters, broadcasts to the conversation topic and per-user destinations, and publishes `message.sent` for the notification pipeline. Typing indicators and read receipts travel over the same socket.

## 4. Push Notification via RabbitMQ → FCM

```mermaid
sequenceDiagram
    participant RMQ as RabbitMQ
    participant NS as notification-service
    participant US as user-service
    participant MG as MongoDB
    participant FCM as Firebase FCM
    participant App as Recipient Device

    RMQ->>NS: deliver message.sent / match.created
    NS->>NS: check presence (viewing this chat?)
    alt recipient actively viewing chat
        NS-->>NS: suppress push (in-app is enough)
    else recipient away
        NS->>US: resolve sender display name / job title
        NS->>MG: persist Notification (isRead=false)
        NS->>MG: look up active DeviceTokens for user
        NS->>FCM: send push (title, preview, data payload)
        FCM-->>App: deliver push (deep-link screen + ids)
        NS->>MG: record pushSent / pushError
    end
```

**Walkthrough.** notification-service consumes match/message/interview events from RabbitMQ. For messages it first checks presence — if the recipient is currently viewing that conversation (tracked via the presence endpoints), the push is suppressed since the in-app socket already delivered the message. Otherwise it enriches the event with human-readable names, writes an in-app `Notification`, fetches the user's active FCM device tokens, and sends a push carrying a data payload (target screen + ids) for deep-linking. Dead-letter queues protect against poison messages, and each attempt records `pushSent`/`pushError`.

## 5. Recruiter Job Creation

```mermaid
sequenceDiagram
    participant App as Recruiter App
    participant GW as API Gateway
    participant JS as job-service
    participant PG as PostgreSQL
    participant RD as Redis

    App->>GW: POST /api/jobs (title, skills, salary, ...) + Bearer JWT
    GW->>GW: validate JWT, inject X-User-Id + X-User-Role=RECRUITER
    GW->>JS: forward
    JS->>JS: authorize recruiter role, validate payload
    JS->>PG: insert Job (company_id, recruiter_id, skills[], expires_at)
    JS->>RD: invalidate/refresh cached listings
    JS-->>App: 201 Created (job detail)
    Note over JS: Job now surfaces in seekers' scored feed
```

**Walkthrough.** A recruiter posts a role from the app. The gateway validates the JWT and forwards the `RECRUITER` role header; job-service authorizes on that header, validates the payload, and persists the job in PostgreSQL with its required-skills array (GIN-indexed) and expiry. Cached listings are refreshed so the posting immediately enters the seeker feed, where it will be ranked by the compatibility scorer against each seeker's profile.

## 6. Offline Queue Sync

```mermaid
sequenceDiagram
    participant UI as App UI
    participant Q as offlineQueueService
    participant Net as Network Listener
    participant GW as API Gateway
    participant Svc as Backend Service

    UI->>Q: enqueue action (SWIPE / MESSAGE / PROFILE_UPDATE)
    Q->>Q: persist to AsyncStorage (priority, retryCount)
    UI-->>UI: optimistic update (feels instant)
    Note over Net: device offline...
    Net->>Q: connectivity restored
    loop drain by priority
        Q->>GW: replay action (Bearer JWT)
        GW->>Svc: forward
        alt success
            Svc-->>Q: 2xx -> remove from queue
        else retriable failure
            Svc-->>Q: error -> increment retryCount, keep
        end
    end
```

**Walkthrough.** When the device is offline, user actions are applied optimistically in the UI and appended to a durable, prioritized queue in `AsyncStorage` (capped at 100 actions, 24-hour TTL, per-action retry limits). A network listener wakes the queue when connectivity returns and replays actions highest-priority-first through the gateway. Successful actions are removed; retriable failures increment a retry counter and stay queued, while expired or exhausted actions are dropped. This makes swiping and messaging resilient on unreliable mobile networks without blocking the user.
