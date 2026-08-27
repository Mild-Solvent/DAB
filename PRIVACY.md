# What DAB actually leaks

A plain-language account of what this app keeps private, what it exposes, and to
whom. Written from reading the code, not from the design intent.

The short version: **what you say is private, and so is who you say it to.**
Content was always encrypted; the metadata around it — your pubkey, your
friend's, and a plaintext label saying what kind of thing you had just done —
used to be readable by anyone who opened a WebSocket to a relay. That is now
gift-wrapped. What remains exposed is network position: your IP address, and the
timing of your traffic.

---

## How the app moves data

Two paths, and they leak differently.

**Direct (WebRTC).** When both phones are online and NAT traversal works, they
talk peer-to-peer. Fast, and no third party sees the content.

**Encrypted mailbox (Nostr relays).** When the direct path fails — the friend is
offline, or a firewall blocks it — messages go to public relays as encrypted
events, and the friend collects them later. This is what makes offline delivery
work without running a server, and it is where the metadata exposure lives.
There are six relays in the pool, and each conversation uses a different three
of them each day, so no single operator sees a whole history.

Both paths encrypt content with NIP-44 (XChaCha20-Poly1305), keyed from your
secret key and your friend's public key. Nobody but the recipient can read it.

---

## What a relay event actually contains

A Nostr event has seven fields. Here is what each one holds now, and what it
used to hold.

| Field | Visible? | Now | Before |
|---|---|---|---|
| `pubkey` | **cleartext** | a throwaway key, one per conversation per day | **your identity key** |
| `created_at` | **cleartext** | fuzzed backwards by up to 10 minutes | timestamp, to the second |
| `kind` | **cleartext** | `30078` — generic "app data", harmless | same |
| `tags[p]` | **cleartext** | 32 opaque bytes, rotating daily | **your friend's pubkey** |
| `tags[d]` | **cleartext** | 32 opaque bytes | `mybump:chat:<friend's pubkey>:<message id>` |
| `sig` | **cleartext** | meaningless — the key is throwaway | proves you wrote it |
| `content` | **encrypted** | the message, wrapped twice | the message |

So an observer scanning relays now sees a pile of generic events from unrelated
keys, addressed to opaque tags that change every day, holding ciphertext. No
pubkeys, no graph, no labels.

One honest caveat about the "unrelated keys": the throwaway key is shared by
everything one conversation sends in one day. It has to be, or messages could
not be deleted afterwards — see below. So events *within* a day are groupable,
which the rotating address tag already made true anyway. Across days, and across
conversations, nothing connects them.

### What that replaced

Your friend's pubkey used to appear **twice** — in `p`, and again inside `d`.
And the `d` tag prefixes described themselves:

```
mybump:chat:      mybump:loc:        mybump:call:
mybump:profile:   mybump:map-chunk:  mybump:attach-chunk:
mybump:pair:      mybump:call-signal: mybump:chat-settings:
```

So an observer didn't just learn that two keys interacted. They read a labelled
timeline:

> `14:32` sent a **location** to B · `14:35` **called** C ·
> `14:41` sent a **photo** to B · `14:52` **paired** with D

The photo's *size* used to be in there too — the chunk count gave it away to
within a few kilobytes. Relayed attachments are padded to fixed chunk buckets,
with every chunk the same width, so the count says "small, medium or large" and
nothing finer. Direct WebRTC transfers skip the padding, since only the friend
sees them.

Worst of all, because every tag started with `mybump:`, **the whole DAB user
base was enumerable.** Anyone could scan relays for that prefix and harvest
every user's pubkey and social graph without knowing a single key beforehand.
Nothing in the wire format says "DAB" any more, and there is no prefix to scan
for — an address tag is 32 bytes you cannot guess.

### How it works

This is **NIP-59 / NIP-17 gift wrapping**, which already exists and has been
reviewed. Three layers: the real event (the *rumor*), sealed with your real key
so the recipient can prove it was you, and that seal wrapped again with a
throwaway key so the relay sees nobody it recognises.

Two things are done differently from the specification, and both are forced:

- **Addressed to a rotating tag, not to your friend's pubkey.** NIP-59 accepts
  revealing the recipient. That would leave the social graph intact, which is
  the entire finding. The tag is derived from the shared secret the two phones
  already compute for encryption, so both sides agree without transmitting
  anything, and it changes every day.
- **The throwaway key is per conversation-day, not per event.** NIP-59 wants a
  fresh random key each time. But relays replace an event only when the kind,
  the pubkey and the `d` tag all match — so a tombstone signed by a fresh key
  replaces nothing, and messages would stop disappearing. Since every event of a
  conversation-day already shares one address tag, a shared signing key tells an
  observer nothing they could not already group.

Which relays carry a conversation also rotates, derived the same way from the
same secret. That stops any single relay from accumulating a complete history.
It does **not** help with your IP address — it makes that slightly worse, since
six operators now see it instead of three.

---

## Who can see it

| Who | What they get |
|---|---|
| **The six relay operators** | that *someone* published to a tag, when, and roughly how large — **plus your IP and connection times**. Any one of them sees only the conversation-days its rotation routed to it |
| **Anyone on the internet** | the same minus your IP. Relay reads are public and need no authentication — open a WebSocket, send a `REQ`, receive it. But you now have to know an address tag to ask for anything, and you cannot guess one |
| **Nostr indexers / scrapers** | the same, crawled continuously and archived by design |
| **Your ISP** | that you connect to `relay.damus.io` and friends (visible in TLS SNI), plus sizes and timing — not content |
| **Your friends** | the content meant for them, **plus your IP** (see below) |

The relay operators are also the only ones who see *you* connect, which is why
the timing correlation below is now the strongest attack left rather than an
afterthought.

## Who keeps it, and for how long

Relays store events — that is their function. Retention is entirely the
operator's choice; you have no agreement with any of them.

**Every** event now carries an `expiration` tag, so nothing lingers past **7
days** wherever NIP-40 is honoured — and the sender replaces a message with an
empty tombstone as soon as the recipient confirms receipt, which is usually
seconds. Attachment chunks and their tombstones work the same way.

Location and map snapshots used to be the exception: they were published under
one unchanging address and simply overwritten, so they carried no expiry at all.
Rotating the address meant a new copy no longer replaces the old one, so they
had to gain the same 7-day expiry as everything else.

Two caveats. The tombstone is a NIP-01 replacement, which every relay
implements; the `expiration` backstop is NIP-40, which not every relay honours,
so a relay that ignores both is still free to keep what it was given. And when
disappearing messages are on, the relay copy now uses that same countdown rather
than the 7-day backstop — the promise is true of the relay copy, not just of the
two phones.

And because public Nostr data is routinely archived by third-party indexers,
deleting from the relays cannot recall what has already been harvested. Worse,
what was harvested before this change is in the *old* format, with pubkeys and
readable activity labels intact. Retention and gift wrapping cap what
accumulates from here; neither undoes history.

---

## Your IP address

This is separate from the relay problem and is not solved by any encryption.

### Who learns it

| Who | When |
|---|---|
| **Every friend** | continuously, whenever your app is open — not only during calls |
| **Google STUN** (`stun.l.google.com`, `stun1`) | constantly — every NAT discovery |
| **Cloudflare STUN** | during calls only |
| **The relays** | whenever the app is connected — and there are six of them now, not three, because relay rotation widened this |
| **The relays — content?** | no. ICE candidates travel inside the encrypted payload |

The "every friend, continuously" part surprises people. The live-sync layer opens
a WebRTC session with *all* friends whenever the app runs, so your address is
shared with each of them constantly — calls are not the only trigger.

One small mercy: ICE gathering only starts once a call is **accepted**, so
declining or missing a call leaks nothing extra.

### How identifying is it?

Depends entirely on the connection:

- **Mobile data (IPv4/CGNAT):** hundreds or thousands of subscribers share one
  address. It identifies your carrier and region, not you. Weak — though the
  carrier's logs can unmask it on legal request.
- **Home WiFi (IPv4):** one address for your whole household, typically stable
  for weeks. Combined with a permanent pubkey, that is effectively a personal
  identifier.
- **IPv6 (common on mobile):** no address scarcity means **no NAT**, so devices
  often get their own globally routable address. Privacy extensions rotate half
  of it, but the `/64` prefix still identifies your line. The "I'm hidden in a
  crowd" assumption does not hold here.

### What can be done

| Goal | How | Cost |
|---|---|---|
| Full privacy, calls still work | **VPN** (on the phone, or on the router to cover the house) | trust moves to the VPN provider |
| Full privacy, no calls | **Orbot** + relay-only mode | Tor carries TCP only, so WebRTC dies |
| Calls, accept household exposure | current behaviour | — |

Bundling Tor into the app was considered and rejected: it means shipping a
daemon, a 10–30 second bootstrap before sync, a much larger binary, and native
work to push WebSockets through SOCKS — all to reproduce what Orbot already does
on both Android and iOS.

**Public proxy lists are not an option.** Traffic is already encrypted, so a
hostile proxy cannot read messages — but it would see your IP, timing, sizes and
which relay you are reaching, which is roughly what the relay sees. You would be
adding an anonymous, unaccountable observer rather than removing one. Free VPNs
have the same problem for the same reason: exit bandwidth costs money, so a free
provider monetises somehow. Proton's free tier and Cloudflare WARP are the
defensible exceptions.

---

## What is genuinely safe

The cryptography holds up. Nobody but the recipient ever sees:

- message text
- GPS coordinates
- photos and files
- profile name and picture

Signatures cannot be forged, so messages cannot be spoofed. Attachment bodies
are chunked and encrypted the same way.

---

## What is being fixed, and how

**1. Retention — done.** Chat events carry an expiration tag, and the sender
replaces the event with an empty tombstone once the recipient confirms receipt.
Uses NIP-01 addressable-event replacement, which every relay implements — not
NIP-09 deletion, which is optional and widely ignored. This also makes
disappearing messages mean what they say.

**1b. Attachment size — done.** Relayed chunk counts are padded to fixed
buckets and every chunk is published at the same width.

**2. The metadata itself — done.** Gift-wrapped transport, opaque per-day
address tags, opaque slot tags, throwaway signing keys, fuzzed timestamps and
deterministic relay rotation. See "What a relay event actually contains" above.

Two things worth knowing about the change:

- **Both phones must be on the new build.** There is no compatibility path. An
  old client and a new client do not crash — they simply never see each other's
  events. Anything still sitting undelivered on a relay at the moment of the
  upgrade is lost; everything already delivered is on the phone and untouched.
- **The background listener goes quiet after a few days** if the app is never
  opened. It cannot derive the rotating tags itself, so the app hands it a few
  days' worth in advance. Opening the app refreshes them.

**3. Network position.** Now the largest remaining exposure. A relay-only mode
so Orbot is coherent rather than half-broken, a choice of when to use direct
connections, and more varied STUN providers.

---

## What cannot be fixed

Worth stating so the app never overclaims:

- **Your IP to whatever you connect to.** Only a VPN or Tor moves it, and Tor
  costs you calls.
- **Timing correlation.** Someone watching a relay sees an event published to a
  tag, then a subscriber to that tag connecting moments later from some address.
  That links two parties without reading anything. Only cover traffic or a
  mixnet addresses it. **This is now the strongest attack on the relay path** —
  gift wrapping removed everything that was easier than it.
- **Total volume.** How much you use the app.
- **Already-scraped history.** Public data that has been archived stays
  archived, and it is archived under the old scheme, with pubkeys and readable
  labels intact. Gift wrapping caps what accumulates from here; it cannot
  retract what alpha testers have already published.

---

## What the app should tell users

The metadata work has landed, so the first sentence can now stand on its own:

> Your messages, photos and location are end-to-end encrypted, and so is who you
> talk to — the relays that carry them see opaque, unlinkable events. What they
> do see is your network address and when you are online.

And for the network side:

> Friends learn your network address, and so do the servers that make direct
> connections possible. On home WiFi that identifies your household. Use a VPN if
> that matters — or Orbot with relay-only mode, which hides it completely but
> disables calls.

A flat "fully private" claim would be an overclaim today, and alpha testers are
exactly the people who would check.
