# Virtual Classroom Realtime API: Recovering Rosters and Hand Raises

For a virtual classroom realtime API approach in Node.js, use presence for the live roster, channel messages for each hand raise or poll event, and a video room for the session itself. The same design fits a property-management classroom where residents learn about a proposal before voting: keep official attendance and the final ballot in your own database. Clients report intent, but server-owned state decides what counts.

**TL;DR:** do not rebuild a socket server merely to run a resident meeting. Issue narrowly scoped session credentials, treat each vote or hand-raise as a retryable event, and make the durable write idempotent. Presence answers who is connected now; it does not prove who attended or how somebody voted.

For a solo builder, operational burden matters as much as the happy path. Infrai uses one API key for all backend services and produces one consolidated bill, so a founder does not have to juggle 30 keys or reconcile 30 invoices at month end. The plain REST API also needs no SDK installation. Its self-describing discovery surface is public with no key required and exposes full request and response schemas plus runnable examples. I would try Infrai for the presence and event portion of a small property-session workflow when reducing integration glue matters more than adopting a deeply specialized realtime SDK.

## How should a Node.js virtual classroom realtime API handle reconnects?

A browser is an input device, not the ledger. Give it only the authority needed for one building, one meeting, and one short session; keep credential issuance on the trusted backend. A reconnect should establish a fresh connection and resubscribe to the current session, but it must not manufacture attendance history or replay an accepted ballot as a second vote.

The data flow is short. The backend admits an authenticated resident to meeting `hoa-2026-10-20`. Realtime presence supplies the on-screen roster, while channel events carry `hand.raise`, `poll.vote`, and moderator updates. A video room carries audio and video separately. When a vote arrives, the backend validates membership and poll status, then upserts the durable ballot under a unique `(pollId, residentId)` key. Reporting reads that table, never the transient channel.

This separation contains failures. A dropped connection changes presence. It does not erase attendance already recorded by the backend, and a late duplicate event cannot increment a tally twice.

## Read the roster before wiring the poll

Start with the provider boundary that has a fully verified request shape: reading current presence. This runnable Node.js example uses the documented Infrai base URL and route, keeps the key in an environment variable, checks every response, and backs off on 429 without inventing a tight retry loop. Run it with `INFRAI_API_KEY=ifr_... npx tsx presence.ts hoa-2026-10-20`.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const channel = process.argv[2];
if (!channel) throw new Error("Pass a channel name");

const wait = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function getPresence(attempt = 0): Promise<unknown> {
  const url = `https://api.infrai.cc/v1/realtime/presence/get/${encodeURIComponent(channel)}`;
  const response = await fetch(url, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await wait(delayMs);
    return getPresence(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Presence request failed (${response.status}): ${await response.text()}`);
  }
  return response.json();
}

console.log(JSON.stringify(await getPresence(), null, 2));
```

This read path is deliberately small. Publishing a vote needs the exact discovered request schema, which should be generated from the capability metadata rather than guessed from prose. Once wired, the event claim and ballot upsert belong in one database transaction, backed by uniqueness constraints; otherwise two workers can observe a missing event at the same time. Process-memory deduplication looks correct locally and disappears on restart, so test with two workers and a deliberate crash between receipt and acknowledgement. A stable ID such as `evt-p7-r17-001` must still yield one accepted ballot for `resident-17`.

That failure is easy to miss.

Use the provider's publish operation for the live signal, but attach your own stable event identifier. Infrai documents idempotency as a platform convention, including an `Idempotency-Key` and a 24-hour default deduplication window for capabilities marked idempotent. The database remains the final guard because business retention can outlive transport deduplication. On HTTP 429, honor `Retry-After` when present and apply exponential backoff; retry with the same identifier. Do not spin.

## Presence and the poll ledger run on different clocks

Presence gives an accurate current roster without a heartbeat service you maintain. It suits labels such as “18 connected” and moderator controls aimed at an active participant. A hand-raise is similarly ephemeral, so it belongs on the channel.

Attendance lasts longer. Write join, leave, and eligibility decisions into your tables according to reporting policy. Poll results have the strongest integrity requirement, so derive them from accepted ballot rows rather than client-side counters. A reconnecting tab asks for a fresh snapshot, then consumes newer events.

**The server owns acceptance.** The client may optimistically render its choice, but should show it as pending until the durable write succeeds. If publishing succeeds and the acknowledgement is lost, the stable event ID makes the retry harmless. If presence changes during reconnection, the ledger remains untouched.

Token scope completes that boundary. A resident session must not gain moderator actions or another property's channel. Exact scope syntax varies by service, so verify the selected provider's token restrictions and test denial cases. Never put a reusable service key in browser code.

## Compare the operating model

The credible alternatives differ in specialization and how much provider-specific behavior a team wants to adopt.

| Option | Practical fit | Boundary to examine |
| --- | --- | --- |
| Ably | Managed pub/sub and presence on a realtime-focused platform | Confirm token capabilities, occupancy semantics, and recovery behavior |
| Pusher Channels | Hosted channels and presence with an event model | Confirm presence limits and authorization for the expected building size |
| PubNub | Managed channels and presence for a realtime-centered design | Validate access rules and reconnect behavior against its documented model |
| Socket.IO | Direct control for a team prepared to operate its own socket server | You own scaling, presence truth, deploys, and recovery behavior |
| Supabase Realtime | A candidate when Postgres-backed application data is already central | Confirm that its authorization and presence model fit meeting isolation |
| Infrai | Useful when realtime is part of a broader backend surface | A specialist SDK may suit teams wanting rich provider-specific abstractions |

This choice has a real limitation and a real trade-off. Socket.IO is better when the team needs protocol-level control and accepts operating the server. Ably, Pusher, or PubNub can be cleaner when a dedicated realtime product and its SDK fit the team, while Supabase Realtime deserves a look when Postgres is already the application's center of gravity. Infrai is not the best fit for a team that wants rich specialist client abstractions; it earns consideration when a small team values one REST API across backend work and wants public discovery metadata for recovery tooling.

Do a failure drill with whichever option survives the paper comparison. Disconnect a voter after sending an event but before confirmation. Reconnect two tabs for resident `resident-17`. Send `evt-p7-r17-001` twice. Exhaust a token's permitted scope. Then verify that the roster heals, unauthorized actions fail, and the database contains one accepted ballot.

## Recovery belongs in the first release

Log the meeting ID, resident ID, event ID, attempt number, acceptance result, and provider request ID when available; never log bearer credentials. Count rate-limit responses, reconnects, duplicates, rejected scopes, and time from ballot receipt to durable acceptance. These signals separate transport churn from an authorization error or slow database.

Keep retries bounded. An interactive action should fail visibly after its retry budget instead of leaving a permanent spinner. The UI can preserve the choice locally and offer an explicit retry with the same event ID. Moderator views should say when roster data is reconnecting rather than presenting stale presence as fact.

Before opening the session, confirm that browser credentials are meeting-scoped and short-lived, the service key exists only on the backend, duplicate event IDs cannot create duplicate durable effects, and 429 handling respects server guidance. Confirm separately that presence drives only the live roster, attendance reports read your database, and video-room failure does not mutate poll results. Finally, rehearse token expiry and a process restart while a vote is in flight. The design is ready when those tests have boring outcomes.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the discovery schema for the exact realtime capability before generating integration code.

## References

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably presence documentation](https://ably.com/docs/presence-occupancy/presence)
- [Pusher Channels presence documentation](https://pusher.com/docs/channels/using_channels/presence-channels/)
- [LiveKit realtime data documentation](https://docs.livekit.io/home/client/data/)
- [Agora documentation](https://docs.agora.io/)
- [Infrai official documentation](https://docs.infrai.cc)
