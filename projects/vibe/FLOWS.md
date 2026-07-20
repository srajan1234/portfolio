# Vibe — Key Flows

Six representative end-to-end flows. Each shows the sequence and a short walkthrough of the important design points.

## 1. Signup + Onboarding (day-8 unlock gate)

```mermaid
sequenceDiagram
  participant U as User (PWA)
  participant BE as Backend
  participant R as Redis
  participant DB as Postgres core

  U->>BE: POST /auth/register (email, password)
  BE->>DB: insert user (BCrypt-12 hash)
  BE->>R: store SHA-256(refresh) TTL 30d
  BE-->>U: access (15m) + refresh (30d)
  U->>BE: POST /users/me/onboarding (intents)
  BE->>DB: onboarding_complete=true, unlock_date=today+7
  BE-->>U: profile
  U->>BE: GET /communities (before unlock_date)
  BE-->>U: 403 LOCKED (countdown shown)
```

**Walkthrough.** Registration hashes the password with BCrypt cost 12 and issues a short access token plus a rotating refresh token (stored only hashed in Redis). Onboarding is idempotent and sets `unlock_date` to seven days out. Community and matching endpoints return `403 LOCKED` until then, and the PWA renders a friendly unlock countdown — the product deliberately makes users build a solo self-reflection habit before opening social features.

## 2. Log Emotion (async event fan-out)

```mermaid
sequenceDiagram
  participant U as User (PWA)
  participant BE as Backend
  participant R as Redis bus
  participant ML as ML service
  participant DB as Postgres core

  U->>BE: POST /emotions (type, intensity, reason)
  BE->>BE: validate, rate-limit (30/day → 429)
  BE->>DB: insert emotion_log
  BE->>R: publish emotion.logged
  BE-->>U: 201 (optimistic UI already updated)
  R-->>BE: emotion.logged (subscribers)
  BE->>R: increment daily counters + streak
  BE->>ML: POST /ml/sentiment (reason text)
  ML-->>BE: score, topics, triggers
  BE->>DB: update sentiment_score + ml_tags
  BE->>BE: goal progress · badges · crisis log-check
```

**Walkthrough.** The write path is intentionally thin: validate, persist, publish, respond. Everything else happens asynchronously off the `emotion.logged` event — daily counters and IST streaks in Redis, ML sentiment enrichment writing back `sentiment_score` and `ml_tags`, goal-progress and badge evaluation, and a crisis log-pattern check. A failure in any subscriber never blocks the user's log.

## 3. Reports & Insights (nightly + live merge)

```mermaid
sequenceDiagram
  participant J as Scheduler (00:30 IST)
  participant BE as Backend
  participant DB as Postgres core
  participant U as User (PWA)
  participant R as Redis

  J->>DB: roll up yesterday's logs → report_daily
  J->>BE: run insight rules → core.insights (top 5)
  U->>BE: GET /reports/weekly
  BE->>DB: range scan report_daily
  BE->>R: read today's live counters
  BE-->>U: distributions + deltas (history + today)
  U->>BE: GET /insights
  BE-->>U: WEEKDAY_PATTERN, IMPROVEMENT, ...
```

**Walkthrough.** A nightly job at 00:30 IST materializes `report_daily` and regenerates the top-5 insight cards. Weekly/monthly/yearly reports are computed on read from `report_daily` merged with today's live Redis counters, so the current day is always current without a second rollup tier. Insights come from five deterministic rules, each producing a confidence score used to keep only the strongest cards.

## 4. Community Post + ML Moderation

```mermaid
sequenceDiagram
  participant U as Author (PWA)
  participant BE as Backend
  participant DB as Postgres core
  participant R as Redis bus
  participant ML as ML service

  U->>BE: POST /communities/{slug}/posts (is_anonymous?)
  BE->>DB: insert post (status ACTIVE)
  BE->>R: publish post.created
  BE-->>U: 201 (author_id omitted if anonymous)
  R-->>BE: post.created (moderation subscriber)
  BE->>ML: POST /ml/moderate (body)
  ML-->>BE: {safe, categories, severity}
  alt unsafe
    BE->>DB: status FLAGGED + hide + mod_queue row
    BE->>U: gentle author notice
  end
```

**Walkthrough.** Posts publish immediately (status `ACTIVE`) and are moderated asynchronously so the feed stays fast. The moderation engine flags only harm aimed at others; first-person distress is masked out and left for the crisis pipeline. Unsafe content is hidden from other users (404 to them), queued for a human moderator, and the author gets a kind notice rather than a punitive one. Approving from the mod queue restores it. For anonymous content the author's UUID never enters the response JSON.

## 5. Vibe Matching + STOMP Chat

```mermaid
sequenceDiagram
  participant BE as Backend
  participant ML as ML service
  participant DB as Postgres core
  participant A as User A (STOMP)
  participant B as User B (STOMP)

  Note over BE: nightly batch OR 5-log debounce
  BE->>DB: build candidate pool (region/age, active 14d)
  BE->>ML: /ml/match-score (chunks of 50)
  ML-->>BE: sameVibe, guide, reasons
  BE->>DB: persist SUGGESTED ≥ 0.65 (cap 10) + reasons
  A->>BE: POST /matches/{id}/request
  B->>BE: POST /matches/{id}/accept → CONNECTED
  A->>BE: CONNECT /ws (JWT interceptor)
  A->>BE: SEND /chat/{matchId}
  BE->>DB: persist message
  BE-->>B: deliver (or FCM if offline)
  B-->>A: read receipt / typing
```

**Walkthrough.** Matches refresh nightly and on a debounced 5-log trigger. The backend scores candidates through the ML formula in batches of 50, persists suggestions above 0.65 with their human-readable reasons ("You both log ANXIOUS most — often around work"), and caps active suggestions at 10. After request/accept moves the pair to `CONNECTED`, chat opens over STOMP: the CONNECT frame is JWT-authenticated by a channel interceptor, SENDs from non-participants are rejected, messages persist and deliver in real time (with presence, typing, and read receipts), and offline recipients fall back to push.

## 6. Crisis Safety Path

```mermaid
sequenceDiagram
  participant U as User
  participant BE as Backend
  participant ML as ML service
  participant R as Redis
  participant DB as Postgres core

  U->>BE: log emotion / message (distress signals)
  BE->>ML: POST /ml/crisis-check (recent_logs, text)
  ML-->>BE: risk NONE|LOW|MEDIUM|HIGH
  alt risk >= LOW
    BE->>R: SETNX crisis:cooldown:{user} TTL 72h
    alt not cooled down (or HIGH escalation bypass)
      BE->>DB: insert crisis_flag
      BE-->>U: supportive resource card (helplines + guide CTA)
    end
  end
  Note over BE,DB: crisis_flags never exposed to org/brand/other users
```

**Walkthrough.** Crisis detection combines a log-pattern layer (consecutive heavy-negative days) with a text layer of curated EN + Hinglish patterns, idioms masked out first. It is assist-only: the response surfaces warm, dismissible helpline resource cards (never alarm-red, no clinical language) and a guide CTA — it never locks accounts or notifies anyone else. A 72-hour Redis cooldown prevents repeat cards, with one deliberate exception: an escalation to HIGH bypasses a cooldown set by a gentler earlier nudge, then re-arms at HIGH. Crisis data lives only in the user's own notification feed and is provably absent from every business or other-user endpoint.
