# Hangout Graph: Product & Technical Specification

*Working title · Draft v1.0 · October 24, 2026*

## 1. Vision and Principles

Hangout Graph is a social app where **you can only be reached through people you already trust**. There are no direct messages, no cold calls and no open “message anyone” button. The connection graph is the product: users see how they are linked, directly and indirectly, to everyone around them, and every private interaction happens inside a *temporary hangout* that someone invited them to through the graph.

Design principles:

1. **Graph first.** The connection graph is the main screen and the main feature.
2. **No unsolicited contact.** Nobody can message or call anyone. The only things you can send are a connection request and a hangout invitation, each with strict eligibility rules.
3. **Ephemeral by default.** Chats, voice and video rooms disappear when the hangout ends. Only *stories* are kept.
4. **Trust travels through people.** An invitation can be forwarded to the invitee’s own direct connections, so friends-of-friends can meet without a cold introduction.
5. **Public talk is separate.** The discussion feed is open and public; it never becomes a private messaging channel.

## 2. Glossary

| Term | Meaning |
| --- | --- |
| Node | A user account. |
| Connection | A mutual, accepted link between two nodes (an edge). Degree 1 = direct, degree 2 = friend of a friend, and so on. |
| Private ID | A short personal code a user shares outside the app (in person, on another platform). Whoever holds it may send that user a connection request. |
| Hangout | A time-boxed gathering, online (text, voice, video) or offline (in person), created by a host. |
| Invitation | A hangout invite sent to a direct connection, or forwarded by an invitee to one of their own direct connections. |
| Room | The temporary chat/voice/video space attached to a hangout. Closed when the hangout ends. |
| Invite chain | The recorded path host → forwarder → forwarded guest. |
| Hangout graph | The separate mini-graph formed by the people who accepted an invitation. |
| Story | A post-hangout memory (photos, short clips, text) shared by participants. |
| Feed | The public Twitter/Reddit-style discussion area. |

## 3. Functional Requirements

### 3.1 Accounts, profiles and settings

- Sign up with email + password (Argon2id hashing) or passkeys; OAuth can come later. Email or phone verification is required to curb fake accounts.
- Profile: display name, avatar, short bio, optional city. A unique @handle is used only on the feed.
- Each user gets a **Private ID** at sign-up (for example 10 characters of Crockford base32 with a check digit). It can be regenerated at any time, which invalidates the old one.
- Privacy settings: who may send me connection requests (Private ID only, or Private ID + indirect connections), how far I am visible in other people’s graphs (see 3.2), and who may invite me to hangouts (all connections or selected ones).
- Blocking, muting, reporting, and account deletion with real data erasure.

### 3.2 Connection graph (main feature)

What the user sees:

- **Ego view (default):** me at the center, direct connections in the first ring, second-degree nodes beyond.
- **Path view:** select any visible node and the graph highlights the shortest path(s) and the mutual connections that link us.
- **Explore mode:** click a node to expand its neighbors, within visibility limits.
- Search by name inside my visible network. A hover card shows name, degree and mutual count; clicking opens a profile card with a **Request connection** button when the sender is eligible (see 3.3).

**Scale rule.** Graphs explode quickly. With 150 connections each, degree 2 can hold about 22,000 nodes and degree 3 more than 3 million. So:

- Degree 1 is always shown in full.
- Degree 2 is ranked by number of mutual connections and capped (for example the top 500); the rest collapses into cluster nodes such as “+3,200 more via Sara”.
- Degree 3 and beyond are never drawn as individuals by default. They are reachable through path search and cluster expansion.

**Visibility privacy.** Each user chooses how far they appear: visible to people within 1, 2 or 3 degrees of them, or to the whole network. Past a viewer’s allowed depth, the node renders as an anonymous dot (the structure stays visible, the identity does not). Suggested default: 3 degrees.

**Look and feel.** Dark canvas, nodes colored by degree, communities subtly tinted, avatars appearing as you zoom in, edges fading with distance, animated camera moves, labels only on zoom or hover.

### 3.3 Connection requests

| Sender | May send a request to |
| --- | --- |
| New user with 0 connections | Only a person whose Private ID they hold |
| User with 1 or more connections | Anyone whose Private ID they hold, or any node visible in their graph at degree 2 or beyond |
| Anyone | Never to blocked users, or to users set to “Private ID only” when the sender lacks the ID |

Rules:

- The recipient sees the sender’s name and mutual connections (or “via Private ID”). **No free-text note is allowed on a request**, because it would be a back door for direct messages. Context is generated by the system (“3 mutual connections”, “Met at *Friday board games*”).
- Accepting creates a mutual edge. Declining or ignoring starts a 30-day cool-down for that pair.
- Limits, for example: 10 requests per day for accounts under 7 days old, 30 per day afterwards, 50 pending at most. Many declines shrink the sender’s allowance.
- Private ID lookups are rate-limited and return the same response for invalid and unavailable IDs, so IDs cannot be enumerated.

### 3.4 Hangout invitations

Formats: **text chat**, **voice**, **video** (optional) and **offline meetup** (time + place, with a coordination chat only).

Core rules:

1. A host can invite **direct connections only**.
2. **Forwarding (“bring a friend”).** Any invitee who has *received* an invitation may forward it to their own direct connections. The host does not need to be connected to the forwarded person.
3. A forwarded invitee sees the host, the forwarder, the format, the time, and who has already accepted. They accept or decline independently.
4. Host controls: forwarding on/off, maximum chain depth (default 1 hop; the host may raise it to 2), forwards per invitee (default 2), **host approval for forwarded guests** (default on), maximum participants, late join on/off.
5. Every forward is stored in the invite chain.
6. Invitations expire when RSVP closes and can be cancelled by the host.
7. Block-aware delivery: if the forwarded person has blocked the host (or the reverse), delivery fails silently with a generic “couldn’t be delivered” message, so blocks are never revealed.

**Group size.** The minimum group is two people. A one-to-one chat or call exists only as a two-person hangout that the recipient has accepted. There is never a “message” or “call” button on a profile.

### 3.5 Temporary rooms and hangout lifecycle

State machine: **Draft → Inviting → Locked → Live → Closed → Archived**.

- **Inviting:** invitations are out and people RSVP.
- **Locked:** RSVP closes (at start time or when the host locks it). Accepted people form the hangout graph.
- **Live:** the room is open. For offline hangouts the coordination chat is open instead.
- **Closed:** triggered by the host ending it, the scheduled end time, everyone leaving for 15 minutes, or a hard cap (for example 12 hours).
- **Archived:** only metadata, the participant list and stories remain.

On close the system ends the media session and revokes tokens, hard-deletes chat messages, disconnects remaining sockets, and writes a tombstone record. One exception for safety: content that was *reported* during the hangout is kept for moderators for a limited period (for example 30 days).

### 3.6 Hangout graph

When RSVP locks, the accepted participants form a separate graph:

- **Solid edges:** existing connections between participants.
- **Dashed edges:** invite-chain links where participants are not directly connected (for example host ⇢ forwarded guest, via the forwarder).
- Participants can view it live and afterwards. Participants who are not yet connected can send each other connection requests: a forwarded guest is by construction two degrees from the host, so rule 3.3 already allows it.

### 3.7 Stories

After a hangout closes, a **story window** opens (default 7 days) in which participants add photos, short clips, text and an optional place tag to the hangout’s shared story.

- **Public:** anyone can view it and it may appear in discovery.
- **Connected:** visible to nodes directly connected to the hangout group (precise definition in Open Question 1).
- Each contributor controls the visibility of their own contribution; the host sets the default. Participants can remove themselves from any tag.
- Stories stay until their author deletes them (optional auto-expiry). They are the only content that outlives a hangout.

### 3.8 Discussion feed (secondary)

A Twitter/Reddit hybrid:

- **Posts:** text, images, links and polls, with an edit window and soft delete.
- **Threads:** nested comments with collapse; upvotes (downvotes optional inside topics).
- **Topics:** Reddit-style communities created by users, with rules, moderators and pinned posts.
- **Feed modes:** *Network* (posts from people within 2 degrees, the graph advantage), *Topics* (communities you joined), *Global* (hot / new / top).
- Posts carry a small graph badge (“2nd degree · 3 mutual”), so public discussion reinforces the graph.
- **Mentions** notify the mentioned person only if they are within 3 degrees or have opted in to everyone; otherwise the mention is shown but silent. This keeps the no-cold-contact principle intact.
- Ranking: “hot” = log(score) plus time decay, as on Reddit.
- Moderation: reports, topic moderator queue, rate limits, lower quotas for new accounts.

### 3.9 Notifications

In-app, web push and email digests, grouped, with quiet hours. Types: connection request, invitation, forwarded invitation, RSVP change, room starting, story added, feed replies.

## 4. Key User Flows

**Flow A: Onboarding**

1. Sign up and receive a Private ID.
2. Enter a friend’s Private ID, send a request, friend accepts, and the first edge appears.
3. Once the user has one connection, the graph shows second-degree nodes and requests through the graph unlock.

**Flow B: Hangout with forwarding**

1. Alex (host) creates “Friday board games” (voice) and invites Bea and Chen, his direct connections, with 1-hop forwarding on.
2. Bea receives it and forwards it to Dev, her direct connection. Alex does not know Dev.
3. Dev sees “Alex invited you, via Bea” and accepts. Host approval is on, so Alex sees “Dev, friend of Bea” and approves.
4. At start time RSVP locks. Alex, Bea, Chen and Dev form the hangout graph: solid edges Alex–Bea, Alex–Chen, Bea–Dev, and a dashed Alex ⇢ Dev link via Bea.
5. The room opens and they talk. Alex ends it.
6. The room and its messages are deleted. The story window opens, Dev posts a photo with “Connected” visibility, and Alex and Dev can request a connection with each other.

## 5. System Architecture

Start as a **modular monolith** plus two specialized services (realtime and media). Split further only when metrics demand it.

```
Web client (React / Next.js, Sigma.js)
   │ REST/JSON          │ WebSocket          │ WebRTC
   ▼                    ▼                    ▼
API service  ◄──►  Realtime gateway      LiveKit SFU (+ TURN)
(Go)                (Go)
   │                    │
   ├─ PostgreSQL (source of truth, graph adjacency)
   ├─ Redis (cache, pub/sub, rate limits, room TTLs)
   ├─ Object storage + CDN (story and feed media)
   └─ Job queue / scheduler (room closing, expiry, notifications)
```

- **Frontend:** TypeScript, Next.js (or SvelteKit), TanStack Query, Sigma.js + graphology. Ship as a PWA first; native apps later.
- **API service:** REST/JSON with an OpenAPI contract; auth, graph, invitations, hangouts, stories, feed.
- **Realtime gateway:** WebSockets for notifications, room chat, presence and graph updates.
- **Media:** self-hosted LiveKit (WebRTC SFU) for voice and video, plus TURN for restrictive networks.
- **Data:** PostgreSQL as the system of record; Redis for caching, pub/sub, rate limiting and presence; S3-compatible storage for media.
- **Observability:** structured logs, OpenTelemetry traces, Prometheus/Grafana metrics, Sentry for errors.
- **Deployment:** containers on a managed platform; adopt Kubernetes only when you actually need it.

## 6. Graph Engine

**Storage and queries.** Begin with a PostgreSQL adjacency table (section 7). With depth capped at 3 and per-node limits, indexed joins are fast. Cache each user’s ego snapshot in Redis, keyed by a per-user graph version that is bumped whenever an edge is added or removed. For shortest path between two nodes, use bidirectional BFS capped at depth 6 over neighbor lists fetched in batches.

*Escalation path:* if traversal becomes the bottleneck, add an in-memory graph service fed by change events from Postgres, or evaluate Apache AGE or Neo4j. Do not start there: a dedicated graph database adds operational cost before you have proof you need it.

**API shape.** `GET /graph/ego?depth=2&limit=500` returns compact arrays of nodes and edges (columnar JSON or MessagePack), already filtered by the viewer’s visibility rights. The server never sends data the viewer may not see; the client performs no privacy filtering.

**Rendering with Sigma.js**

- Sigma v3 on a graphology graph. WebGL rendering handles thousands of nodes smoothly.
- Layout: ForceAtlas2 in a web worker. For ego views, seed positions in concentric rings by degree, then relax with ForceAtlas2 for a stable, fast first paint.
- Keep the rendered graph under roughly 5,000 nodes and 20,000 edges; larger sets are aggregated into cluster nodes on the server.
- Level of detail: label density thresholds, avatars (the node-image program) only above a zoom level, thin or hidden edges when zoomed out.
- Community detection (Louvain via graphology) for cluster colors.
- Node and edge reducers for hover and path highlighting without rebuilding the graph; animated camera on focus.
- Apply live deltas (edge added or removed) from the WebSocket instead of refetching.

## 7. Data Model

Core entities: `users`, `private_ids`, `connection_requests`, `connections`, `blocks`, `hangouts`, `invitations`, `participants`, `room_messages` (ephemeral), `stories`, `story_items`, `topics`, `posts`, `comments`, `votes`, `reports`, `notifications`.

The invite chain lives in `invitations` through `parent_invitation_id` and `depth`.

```sql
CREATE TABLE connections (
  user_id    BIGINT NOT NULL REFERENCES users(id),
  friend_id  BIGINT NOT NULL REFERENCES users(id),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, friend_id),
  CHECK (user_id <> friend_id)
);
-- each friendship is stored as two rows, (a,b) and (b,a), for fast neighbor lookups
CREATE INDEX ON connections (friend_id);

CREATE TABLE invitations (
  id                   BIGSERIAL PRIMARY KEY,
  hangout_id           BIGINT NOT NULL REFERENCES hangouts(id),
  from_user_id         BIGINT NOT NULL REFERENCES users(id),
  to_user_id           BIGINT NOT NULL REFERENCES users(id),
  parent_invitation_id BIGINT REFERENCES invitations(id),
  depth                SMALLINT NOT NULL DEFAULT 0,
  status               TEXT NOT NULL,  -- pending, accepted, declined, expired, awaiting_host
  UNIQUE (hangout_id, to_user_id)
);
```

Second-degree nodes ranked by mutual connections:

```sql
SELECT c2.friend_id, COUNT(*) AS mutuals
FROM connections c1
JOIN connections c2 ON c2.user_id = c1.friend_id
WHERE c1.user_id = $1
  AND c2.friend_id <> $1
  AND NOT EXISTS (
    SELECT 1 FROM connections d
    WHERE d.user_id = $1 AND d.friend_id = c2.friend_id)
GROUP BY c2.friend_id
ORDER BY mutuals DESC
LIMIT 500;
```

## 8. Real-Time and Temporary Room Design

- The WebSocket gateway authenticates with a short-lived token and subscribes each client to `user:{id}` (notifications) and `room:{id}` (chat, presence). Instances fan out through Redis pub/sub; move to NATS if you outgrow it.
- Voice and video: one LiveKit room per hangout. The API mints a join token only for accepted participants of a Live room.
- Chat messages are stored in Postgres keyed by `room_id` for reliability during the hangout, then hard-deleted at close. Message bodies are never written to logs.
- **Closing:** a scheduled job per room (River on Postgres, or Asynq on Redis) runs an idempotent close routine at the end time. A sweeper that runs every minute catches anything missed. A hard maximum duration backs it up.
- Presence uses Redis keys with heartbeat TTLs; it drives the “everyone left for 15 minutes” rule.
- The room sends a `room.closing` event 60 seconds before it ends.
- Offline hangouts: the coordination chat opens at lock and closes a few hours after the scheduled end. The exact location is revealed only to accepted participants, and only after lock.

## 9. Backend Language: Go vs Rust

| Criterion | Go | Rust |
| --- | --- | --- |
| Time to a working product | Fast: simple language, quick compiles | Slower: ownership and lifetimes, longer compiles |
| WebSockets and many concurrent connections | Natural with goroutines | Excellent with Tokio, but more boilerplate |
| Raw performance and memory | Very good; the garbage collector adds small overhead | Best: no GC, lowest memory use |
| Web and database ecosystem | Mature: chi/Echo, pgx, sqlc, goose | Mature: axum, sqlx, tower, diesel |
| WebRTC and media | LiveKit and Pion are written in Go, so first-class | Possible (webrtc-rs), less battle-tested |
| Modeling state machines | Good with discipline | Enums with exhaustive matching make invalid states hard to represent |
| Learning curve and hiring | Gentle | Steep |
| In-memory graph algorithms | Fine up to millions of edges | Best for very large, latency-critical graphs |

**Recommendation: use Go for the core backend.**

1. The workload is I/O-bound (database, sockets, media signaling), where Go’s performance is more than enough and the garbage collector will not be your bottleneck.
2. Concurrent connections are Go’s sweet spot, and your realtime gateway is a large part of this product.
3. The media stack is a natural fit: LiveKit’s server and Pion are Go, with first-party Go server SDKs for token minting and room control.
4. The rules in this spec (forwarding limits, approval, story visibility) will change as you learn from users. Fast iteration matters more than the last 20% of performance right now.
5. Operations are simple: one static binary per service.

**When Rust is the better call:** you already know it well; you need strict tail latency for an in-memory graph serving millions of nodes; or you want the compiler to enforce the invitation and room lifecycle as types.

**A sensible hybrid:** build everything in Go now. If graph traversal becomes the measured bottleneck, extract a small Rust `graphd` service behind the same API. Do not pay Rust’s development cost across the whole product to optimize one component.

Suggested Go stack: chi (router), pgx + sqlc (database), goose (migrations), coder/websocket, River (jobs), slog + OpenTelemetry, oapi-codegen (OpenAPI), testcontainers (integration tests).

## 10. Security, Privacy and Abuse Prevention

| Threat | Mitigation |
| --- | --- |
| Spam and fake accounts | Email/phone verification, rate limits, progressive trust (new accounts get small quotas), device and IP heuristics |
| Private ID enumeration | Long random codes with a check digit, rate limits, uniform error responses, easy rotation |
| Graph scraping and stalking | Server-side visibility filtering, depth caps, per-user query budgets, anonymized nodes past the allowed depth |
| Forwarding abuse (strangers pulled into hangouts) | Host approval on by default, 1-hop limit, per-invitee quota, forwarded guests see host and chain before accepting, report button on invitations |
| Offline-meetup risk | Location hidden until accepted and locked, option to share details with a trusted contact, first-meetup safety tips, report link |
| Harassment through requests | No free-text notes, cool-downs, blocking cascades (removes the edge, cancels pending invitations, hides both users from each other’s graphs) |
| Room content leaks | TLS everywhere, authorization on every room subscription, deletion at close, no content in logs |
| Data protection | Account deletion and data export, minimal retention, a minimum-age policy (13+ or higher by region) with offline hangouts disabled for minors |
| Feed abuse | Rate limits, report queue, topic moderators, link scanning |

## 11. API Surface (v1 sketch)

**REST**

- Auth and profile: `POST /auth/signup`, `POST /auth/login`, `GET/PATCH /me`, `POST /me/private-id/rotate`
- Graph: `GET /graph/ego`, `GET /graph/path?to={userId}`
- Connections: `POST /connection-requests`, `POST /connection-requests/{id}/accept`, `POST /connection-requests/{id}/decline`, `DELETE /connections/{userId}`, `POST /blocks`
- Hangouts: `POST /hangouts`, `POST /hangouts/{id}/invitations`, `POST /invitations/{id}/forward`, `POST /invitations/{id}/respond`, `POST /hangouts/{id}/guests/{userId}/approve`, `POST /hangouts/{id}/end`, `GET /hangouts/{id}/graph`
- Stories: `POST /hangouts/{id}/stories`, `GET /stories?scope=connected|public`
- Feed: `POST /posts`, `GET /feed?mode=network|topics|global`, `POST /posts/{id}/comments`, `POST /posts/{id}/vote`, `POST /topics`

**WebSocket events:** `notification.new`, `invitation.received`, `room.message`, `room.presence`, `room.closing`, `graph.delta`.

## 12. Non-Functional Targets

- API latency: p95 under 200 ms for ordinary requests.
- Ego graph (500 nodes): under 300 ms server time and under 150 KB gzipped payload.
- Room join in under 2 seconds; first load test target of 10,000 concurrent sockets per gateway instance.
- Availability goal: 99.9%; the room-closing sweeper must never miss a close.
- Privacy invariants covered by automated tests: no data returned beyond the viewer’s visibility, no room data after close, no delivery that reveals a block.

## 13. Roadmap

| Phase | Scope |
| --- | --- |
| 0. Foundations | Auth, profiles, Private IDs, schema, CI/CD, observability |
| 1. Graph MVP | Connection requests, ego view with Sigma.js, path view, blocking, visibility settings |
| 2. Hangouts | Invitations, forwarding and approval, text and voice rooms, lifecycle and auto-close, hangout graph |
| 3. Stories and notifications | Story window, public and connected visibility, push and email |
| 4. Feed | Posts, comments, voting, topics, moderation tools |
| 5. Hardening and scale | Load tests, abuse tooling, video, mobile apps, graph service if metrics demand it |

**MVP = phases 0 to 2.** The graph plus invite-only temporary hangouts is what makes this product different, so ship that before the feed.

## 14. Open Questions

1. **Who sees “Connected” stories?** Nodes connected to *at least one* participant, or to *all* of them? “All” almost never matches anyone in a larger group. Recommended default: at least one participant.
2. **Two-person rule.** This document reads “the group can contain two people” as a minimum of two (one-to-one hangouts are allowed, always by invitation). If you meant a maximum of two for chats, say so. Per-format maximum sizes are still to decide.
3. **Forwarding defaults.** Chain depth, forwards per invitee, and whether host approval should be on by default.
4. **Default graph visibility depth** (3 degrees is proposed) and how much a stranger-at-degree-3 node reveals.
5. **Offline hangouts.** Only a coordination chat, or built-in maps and live location sharing?
6. **Age policy and moderation capacity.** Who reviews reports, and how fast?
7. **Name and business model.** Both affect trust and safety design choices.
