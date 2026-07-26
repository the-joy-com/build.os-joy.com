# session 13 of building The Joy in the open

## let's start with the basics: input

ok, let's start over from scratch, Telegram is having reachability issues right now and it's definitely bugging me that all my deepest personal data is used on their servers....

I've been thinking: "what should be my entry point to this project?" well... it should be the entry point itself: the input layer! since I'm no Android developer but I do know web apps and also that my application should be reachable cross devices. Let's decide real quick: input layer is a web app. Fuck the chat experience though, I want this to be driven in a terminal-like experience, I don't want my mind to be cluttered by design considerations when this thing is about allowing me to communicate with an always on machine symbiot for my psychological stability in this crazy World.

Specs are thus:
- should look and feel like a terminal
- should be a PWA (app' feel experience on my Android phone)
- should be responsive

... so let's focus on that.

## The Joy — input shell (spec)

The **input layer** should be an always-on, terminal-styled web app that is the front door to The Joy. It is deliberately the smallest honest slice — the surface I'll actually open every day on my laptop and my phone — and nothing more.

The naming law holds: the human is **the symbiot** (ND), never "the user". The machine side is **The Joy**.

---

### Build order (methodology)

Front-first, mock-first — but **not** front-*only*. This is a full-stack iteration whose backend is real, just built last: the `/login` OTP/session flow (§9) and the endpoint a delivered line POSTs to are genuine server-side code that lands in phases 3–4. What stays mocked is the **agent** behind that endpoint — the thing that would read and act on a received line (§2) — never the server itself. Four phases, each shippable before the next one exists:

1. **Build the frontend against a backend that does not exist.** All input goes through the outbox + service-worker delivery (§5), with a stubbed `handoff()` that immediately returns `COPY`. The full offline-queue and Background-Sync plumbing is real from day one — only the handoff *target* is fake. The goal is to get the flow and the feel right with no server in the loop. Two checks can't go green here, by design: the stub knows nothing about sessions, so the **authed-vs-unauthed reply difference** and anything requiring real **auth** stay red until phase 3 — the stub returns the same `COPY` regardless.
2. **Deploy the frontend to the bare-metal server.** With the stub still standing in for the backend, QA and adjust the experience on mobile (PWA install, virtual keyboard, offline indicator, offline→reconnect flush) on the real device over the real network.
3. **Wire the server locally.** Build the real backend — the **auth endpoints** (OTP issuance, verification, session; §9) and a **receipt endpoint** that accepts a delivered line and acks it — and swap the stub `handoff()` *inside the service worker* for a real POST, run on localhost. The first real handoff, and where the genuine server-side work lands; the UI does not change.
4. **Deploy the server too.** Stand the backend up on the bare-metal box alongside the already-deployed frontend.

The seams (§5) are what make this order possible: phases 1–2 run the frontend against stubs, and phases 3–4 swap them for the real endpoints without touching UI code. But the iteration is only **done** when those endpoints are real — the auth flow especially (§10) can't be faked away — so don't read the mock-first *sequence* as a backend-free *iteration*.

### 1. What it is

A single-screen, terminal-styled web app:

- An append-only **log** of what I've submitted; each submitted line gets a `COPY` under it when The Joy received it. Offline, the line waits in a local outbox and is delivered automatically when connectivity returns (§5). Once received, The Joy sends back a **canned, auth-aware stand-in reply** — a placeholder for the agent's eventual voice, not a real answer (§2).
- A single **input line** at the bottom with a prompt caret.
- Installable as a **PWA** on Android, runs standalone, opens offline as a shell, with a persistent **offline indicator**.
- Reachable from laptop and phone, behind single-symbiot auth, on my own metal.

Functionally it is an input box wearing a terminal's clothes: I type, it confirms capture, that's the loop. "fuck the chat experience" meant no elaborate chat UI — and The Joy doesn't really *answer* yet, it just reliably takes and acknowledges input and sends back a canned, auth-aware stand-in for an answer.

### 2. What it is NOT (scope guard)

These are named so they stay deferred, not forgotten:

- **No agent, no LLM, no *real* reply.** Enter acknowledges receipt, and The Joy sends back a hardcoded, auth-aware line standing in for an answer — but nothing reasons over the input. The Joy does not actually *answer* yet; capturing input reliably is the whole job.
- **A minimal backend — not the *agent* backend.** This iteration *does* build real server-side code: the `/login` auth endpoints (§9) and an endpoint that receives a delivered line and acks it (`COPY`), plus a hardcoded, auth-aware reply standing in for what an agent would later say. What it does *not* build is the **agent** behind that endpoint — nothing reads, filters, or reasons over the received line yet. Early phases stub the server's targets (the SW's `handoff()` returns `COPY` locally), but by the end of the iteration the server is real (see the build order). So "no backend" is the wrong read; "no agent yet" is the right one.
- **No context-aware filter / soul feature.** That's the point of The Joy, still undecided as a first *capability*; this is the surface it will eventually speak through.
- No background deep agents, no streaming, no multimodality (voice/video), no settings screens, no multi-symbiot, no themes beyond one.

If a feature isn't "type a line, see it acknowledged, on both devices, always reachable," it's out of scope for now.

### 3. Stack

- **TypeScript + Vite.** No UI framework. One screen doesn't need one, and a no-framework codebase is the easiest to resume after idle days.
- **xterm.js** for the terminal surface itself. It is the best way to get a real terminal feel without burning time hand-building one — it owns the rendering, the caret, scrollback, input history, and resize handling, so I don't reinvent any of it.
- **vite-plugin-pwa** for manifest + service worker (in `injectManifest` mode, since the SW carries real logic — outbox delivery, Background Sync, the inbound message channel — not just generated precache).
- **The service worker owns delivery.** A local IndexedDB **outbox** is drained by the **Background Sync API** when connectivity returns, even if the app is closed; one SW→page message channel carries every inbound thing — `COPY`s and the canned auth-aware reply now, real agent replies and push notifications later. Background Sync is Chromium-only, which is fine (Android is the target), and a flush-on-load fallback covers any browser without it.
- Minimal dependencies on purpose. Every dep is a thing to re-understand on resume — xterm.js earns its place by removing far more terminal-plumbing work than it adds.

### 4. Interaction model

- **Terminal surface:** xterm.js renders the whole thing — monospace, dark background, prompt string + blinking caret, scrollback. I write the prompt, read the input line, and print back to the terminal; xterm owns the rest. On init the terminal prints **The Joy ASCII banner** — a small boot masthead — then drops to the prompt. It's the fixed opening of every fresh session: since the terminal does not resume prior scrollback (§8), the banner is the one thing a freshly-opened, otherwise-blank terminal always shows.
- **Submit → COPY:** Enter echoes my line immediately and writes it to the outbox (§5). The line under it is the **delivery-status slot**: while the line is `pending` that slot shows a **dim** marker, and when delivery succeeds the SW posts back and the slot is **replaced in place** by a full-brightness `COPY`. The dim→bright flip *is* the "received" signal — no color, no glyph vocabulary beyond the one theme (§2). The dim text says *why* it's waiting, so the per-line state never contradicts the §6 global indicator:
  - offline → `⋯ queued` (the global indicator agrees)
  - online, in flight / retrying → `⋯ sending…`
  This closes the stuck-vs-in-flight gap: a slow or silently-failed delivery reads as `sending…`, visibly distinct from simply being offline (`queued`). `COPY` lands usually instantly when online, or later (automatically) if I was offline when I typed it; until then the line just sits captured with its dim marker. No "failed/giving up" state for now — the SW retries indefinitely via Background Sync (§5), so there is no dead state yet; that belongs with real error handling later. The only machine reply beyond `COPY` is the canned, auth-aware stand-in (§2) — no real answer yet.
- **Input history:** Up/Down arrows walk previously submitted lines.
- **Meta-commands** (slash-prefixed, never sent to the outbox as input). Keep this set tiny:
  - *Client-side only:* `/clear` clears the terminal, `/help` lists commands.
  - *Auth/session (talk to the auth endpoints, §9):* `/login` starts the email-OTP flow (prompts for email, then for the code); `/logout` drops the session token/cookie and returns to the unauthed prompt; `/status` prints the current state — connection (online/offline, matching the §6 indicator) and session (authed as which email, or not).
- **Submit:** Enter sends the current line; multi-line input is captured verbatim.

### 5. The I/O seams (outbound, delivery, inbound)

Input does **not** round-trip through one request/response function, because delivery is owned by the service worker (§6) and can finish after the page is closed. So there are two seams and one channel — and that split is deliberate: a `COPY`, a real reply from The Joy, and a push notification are all the same thing, an inbound message, and they all arrive the same way.

**Outbound — the page writes to a local outbox. Never blocks, never loses input:**

```ts
type Outgoing = { id: string; text: string; createdAt: number; state: "pending" | "acked" };

async function enqueue(input: string): Promise<{ id: string }> {
  const id = crypto.randomUUID();                    // client-side handle for outbox bookkeeping — never sent to the server
  await outbox.put({ id, text: input, createdAt: Date.now(), state: "pending" }); // persist first (§8)
  await registration.sync.register("flush-outbox");  // wake the SW when connectivity returns
  return { id };                                     // UI echoes the line as pending — no COPY yet
}
```

**Delivery — the service worker drains the outbox when online, even if the app is closed. Everything queued goes up as one formatted transmission:**

```ts
self.addEventListener("sync", (e) => {
  if (e.tag === "flush-outbox") e.waitUntil(flush());
});

async function flush() {
  const lines = await outbox.pending();                // oldest first
  if (!lines.length) return;
  const res = await handoff(formatBatch(lines));       // one transmission, all queued lines at once
  if (!res.ok) return;                                 // still unreachable — leave them pending, sync retries
  for (const line of lines) await outbox.markAcked(line.id); // delivery is the SW's job — so is flipping the status
  await notifyClients({ kind: "ack", at: res.at }); // nudge only: the outbox changed, re-derive — COPY appears under the now-acked lines
}
```

```ts
// One reconnect can carry several lines typed across a span of offline time.
// Send them as a single, legibly-formatted burst — with each line's original
// timestamp — so the agent downstream reads them as one coherent batch that
// arrived together on reconnect, not N context-free pings. No ids on the wire.
function formatBatch(lines: Outgoing[]): string {
  const body = lines
    .map((l) => `[${new Date(l.createdAt).toISOString()}]\n${l.text}`)
    .join("\n\n— — —\n\n");
  const header =
    lines.length > 1
      ? `# ${lines.length} messages queued offline, sent together on reconnect\n\n`
      : "";
  return header + body;
}
```

**Inbound — the page renders whatever the SW sends up:**

```ts
// One channel, a tagged union of what the SW sends up — switch on `kind`:
type FromSW =
  | { kind: "ack"; at: number }                   // SW already flipped delivered lines to acked; re-derive — COPY appears
  | { kind: "message"; at: number; text: string }; // The Joy spoke — a reply or an unprompted nudge

navigator.serviceWorker.addEventListener("message", (e) => {
  const m = e.data as FromSW;
  switch (m.kind) {
    case "ack":     /* re-derive the view from the outbox — the now-acked lines show COPY */ break;
    case "message": /* render m.text on arrival; it answers to nothing I typed */ break;
  }
});
```

The trigger and retry rules are unchanged by the batching — only the payload shape is: instead of one send per line, the whole `pending` set goes up as a single formatted transmission. There is deliberately **no server-side dedup**: if a success ack gets lost and the next `sync` re-sends, The Joy just receives the batch twice — which is exactly what it is, a connectivity hiccup, and the agent can read it as one rather than us engineering around it. The line `id`s stay a purely client-side bookkeeping handle — what's `pending`, what to flip to `acked` — and are never put on the wire. A line stays `pending` until a real success, so a failed or offline attempt just waits and the next `sync` retries it — `COPY` is never printed for something The Joy didn't receive. (`flush()` also runs on SW activate and on page load, so an open app drains immediately instead of waiting on the sync event, and browsers without Background Sync still flush.)

This is the no-debt shape, not an optimization bolted on later: outbound capture and inbound rendering are genuinely different concerns, and every inbound thing The Joy will ever do — `COPY` now, real replies and push notifications later — arrives through the one SW→page channel. Phase 3 swaps the stub `handoff()` for a real POST *inside the service worker* and touches no UI. Persist-before-anything mirrors the old arc's rule: never confirm receipt of something you haven't durably captured, and never print a `COPY` The Joy didn't give.

### 6. PWA requirements

- **Web app manifest:** name, short_name, `display: standalone`, `background_color` (the terminal's bg — kills the white cold-start splash flash), a single `512×512` maskable icon. No `theme_color` (fullscreen leaves no chrome to tint) and no 192px variant (512 down-scales); installability is the only thing the icon guards, and one maskable 512 satisfies every Chromium version without depending on the synthesized-icon fallback.
- **Service worker:** precaches the app shell so it opens offline, **and owns message delivery** — a local IndexedDB outbox drained by the **Background Sync API** when connectivity returns, even if the app was closed, with a flush-on-load/activate fallback for browsers without Background Sync. The same SW→page channel that delivers `COPY` and the canned auth-aware reply today carries real agent replies and push notifications later (§5). Offline still captures the line locally; it's delivered on reconnect, no `COPY` until then.
- **Installable** to the Android home screen and launchable full-screen with no browser chrome.
- **Offline indicator:** a persistent status element reflecting `navigator.onLine` (+ the `online`/`offline` events), present from the first build. Offline never blocks input — the line is captured, sits `pending` in the outbox, and gets no `COPY` until reconnect; the indicator is what tells me The Joy hasn't received it yet, so the connection state is always visible.
- HTTPS is mandatory (service workers require it) — see §8.

### 7. Responsive / cross-device

- Single fluid layout: the log fills the viewport, input pinned to the bottom.
- **Virtual-keyboard handling** via the `visualViewport` API so the input line stays above the Android keyboard instead of being covered.
- Respect `env(safe-area-inset-*)` for notches/home indicators.
- Font size ≥ 16px on the input to prevent iOS/Android focus zoom.
- Autofocus the input on laptop; tap-to-focus on mobile with a comfortable tap target.

### 8. Persistence

- Every submitted line is written to a local **outbox** in IndexedDB *before* anything is sent (§5): `{ id, text, createdAt, state }`, where `id` is a client-side bookkeeping handle (never sent to the server) and `state` goes `pending → acked`. Single-symbiot, single-device-local store — no cross-device sync for now.
- **The store exists for delivery, not for replay.** Its one job is that no line is ever lost: `pending` lines survive a reload, a crash, or a full quit and drain automatically on reconnect (§5) — including ones typed in a session I've since closed. Background Sync runs without the terminal needing to show anything.
- **Quitting the app does not resume the terminal session.** A fresh open is a **blank terminal** — just the boot banner (§3) and a prompt. The prior scrollback is *not* rebuilt from the store, and neither is the input history; the terminal is a live view of the current session only.
- This holds together precisely because delivery is decoupled from the view. A line queued in a previous session still goes up on reconnect — I just don't watch it leave on the way back in. And anything inbound — `COPY`s now, real replies later — is **not anchored to any client `id`**, so it **prints as it arrives** in whatever terminal happens to be open, even when the line that triggered it scrolled away with a closed session. The in-place marker→`COPY` swap (§5) is simply the same-session shortcut for when that line is still on screen.

### 9. Auth & hosting

- **Self-hosted** on my own bare metal (the same box that already runs Postgres). Sovereignty was the reason Telegram died; the shell honors it from line one.
- **Served at `shell.os-joy.com`**, with **HTTPS** terminated at the box (reverse proxy + cert). Required for PWA and for protecting personal data in transit.
- **Single-symbiot auth — email OTP, in-terminal, from line one:** there is **no login screen and no OTP screen**, and the shell does **not** treat unauthed differently. The terminal always renders, and **unauthed input follows the exact same rules as authed input** — same outbox → delivery seam (§5), same echo, same `COPY`, no nag to `/login`. The only difference is what rides the request: a valid session cookie or not. What that distinction *means* — how authed vs unauthed input is handled — is entirely **the server's responsibility**, not the shell's. This is deliberate and load-bearing for the future: it's how the World talks to me through my agent, so the shell accepts **visitors / alpha-testers from day one** without a second build. Auth gates *identity* (does the server know this is the symbiot?), not *the right to submit*.
- **`/login` flow:** runs the whole exchange *in the terminal* — type the email at the prompt, fetch the one-time code from the mailbox, type the code back in; a valid code mints a signed session token/cookie and the same terminal is now authed as the symbiot. Even the first mocked iteration ships this real (no fake/bypassed login) — but it stays minimal: no signup (the one allowed address is configured, not registered), no roles, no password, no recovery flows. The mailing can be mocked early (code logged/stubbed) while the flow itself is real end-to-end.
  - **OTP issuance is allow-listed to the configured symbiot address, and the input is only ever a yes/no gate — never the recipient.** The server **never emails the code to an address read from the input.** It compares the typed value to the one configured address by **exact, normalized, single-value equality**: trim, lowercase, and require exactly one address — reject anything containing a comma, semicolon, surrounding whitespace, or newline before comparing. On a match it sends the code to the **stored configured address it already holds**; on anything else **no OTP is generated and no email is sent at all** — nothing leaves the box. This is what closes the obvious tweaks: a multi-recipient string like `me@allowed, attacker@evil` fails the single-value check (no comma-splitting into multiple `To:`s, ever); a substring like `me@allowed.evil` or `attacker+me@allowed` fails equality; a newline can't smuggle a second recipient because the recipient is the server's own constant, not echoed input. The terminal's response looks the same either way ("if that address is allowed, a code is on its way"), so the prompt never enumerates who is or isn't the symbiot. For now there is exactly one allowed address; everyone else is simply a `pending`-only submitter who never receives a code.
  - **Re-running `/login` is safe and the latest code always wins.** Requesting a new code **invalidates any previously issued one** — only the most recently emailed code is ever valid, so two live codes can never float around at once (a half-entered flow that gets re-started just abandons cleanly). Running `/login` while **already authed** is a no-op that simply reports the current `/status` ("already authed as …") rather than starting a fresh email prompt — there's only one address, so re-authing buys nothing.
- **`/logout` / `/status`:** `/logout` drops the session token/cookie and returns the terminal to the unauthed prompt; running it again when **already unauthed is an idempotent no-op** — it just confirms "not logged in", never errors and never does anything destructive. `/status` prints the current connection state (online/offline, matching the §6 indicator) and session state (authed as which email, or unauthed). All three are listed by `/help` alongside `/clear`, so the auth surface is discoverable from inside the terminal without a screen.

### 10. end-user QA for the shell iteration

Run these in order on both laptop and phone; the shell iteration is done when every box checks.

- [ ] Served at `shell.os-joy.com`, over HTTPS only.
- [ ] Installs to my Android home screen and launches standalone — full-screen, with no browser chrome (no address bar, tabs, or navigation buttons), so it looks like a native app rather than a page in Chrome.
- [ ] On launch, the terminal initializes showing The Joy ASCII banner, then drops to the prompt.
- [ ] `/help` lists every command, including `/clear`, `/login`, `/logout`, `/status`.
- [ ] Type a line → Enter → it echoes immediately.
- [ ] While `pending`, the line shows a dim `⋯ sending…` marker beneath it.
- [ ] On ack, that marker is replaced **in place** by a full-brightness `COPY` — no second line is added.
- [ ] `COPY` never appears for a line The Joy didn't actually receive.
- [ ] `/login` with the exact configured address → a code is issued and sent only to that stored address; entering it authes the same terminal, and `/status` then reports authed for that email.
- [ ] `/login`, then entering a **wrong** code → rejected; the terminal stays unauthed and lets me retry, and only the correct (latest) code ever authes. No lockout yet, but a bad code never lets me in and never errors out the flow.
- [ ] `/login`, then submitting an **empty/blank** line instead of an address → no code is generated and nothing is sent; the flow cancels cleanly (or re-prompts), never errors, and never leaves a half-open login state.
- [ ] `/login` with an unknown address → no code is generated and nothing is sent; the terminal's reply is identical to the success case.
- [ ] `/login` with a recipient-smuggling value (`me@allowed, attacker@evil`, `me@allowed;attacker@evil`, a substring like `me@allowed.evil` or `attacker+me@allowed`, or an embedded newline) → no code, no email to anyone, nothing to the smuggled address; reply still identical.
- [ ] `/login` run twice before entering a code → only the most recently emailed code works; the earlier one is rejected.
- [ ] `/login` while already authed → no fresh email prompt; it just reports the current authed status.
- [ ] `/logout` drops the session; `/status` reverts to unauthed.
- [ ] `/logout` run again while already unauthed → idempotent no-op; it confirms "not logged in" and nothing breaks.
- [ ] Unauthed input submits by the exact same rules as authed — the shell makes no distinction.
- [ ] Submit a line while **authed** → beyond the `COPY`, The Joy returns a distinct, hardcoded reply standing in for what an agent would eventually say (no real LLM involved).
- [ ] Submit a line while **not authed** → the reply is recognizably *different* from the authed one. Same submission path (per the check above); the difference is the server's doing, proving authed vs unauthed is handled server-side — not by the shell.
- [ ] Switch to airplane mode → the shell still opens, renders the terminal, and accepts input.
- [ ] Submit 2–3 lines offline → each echoes and is held `pending` with a dim `⋯ queued` marker, no `COPY`.
- [ ] Reconnect → the queued lines deliver automatically and their `COPY`s appear in submit order.
- [ ] Submit offline, fully close the app, then reopen → a blank terminal (just the banner + prompt); the prior lines are gone from view but still deliver on reconnect (Background Sync), and their `COPY`s print as they arrive.
- [ ] Several lines queued offline go up as **one** transmission, formatted with each line's timestamp and a separator.
- [ ] No `id`s appear in the transmitted payload — they stay client-side bookkeeping only.
- [ ] Reload, or fully quit and reopen → a blank terminal; the prior scrollback and input history are **not** restored (the session does not resume).
- [ ] Quit/reload with un-acked lines → they're gone from the blank view but survive in the outbox and still deliver on reconnect, printing their `COPY`s as they arrive.
- [ ] Renders cleanly on my laptop — the log fills the viewport, the input stays pinned at the bottom, and the layout holds across window resizes.
- [ ] The Android virtual keyboard does not cover the input line, and tapping the input does not trigger focus-zoom.
- [ ] A version tag — `v0.0.1` — is pinned to the bottom-right corner of the terminal, present from the moment it opens.
- [ ] A connectivity indicator is pinned to the bottom-left corner: **green** when online, **red** when offline. Going into airplane mode flips it to red; reconnecting flips it back to green — tracking the same connectivity the outbox drains on (§5).

When every box checks, this iteration is done: a real shell on real metal, real auth, and lines reliably received and acked — a full-stack slice, not a front-end. The next iteration is the **agent**: what The Joy actually does with a received line once it has it (today it just replies with a hardcoded, auth-aware stand-in).

## the second iteration to come: give a ghost to the shell

before any of the features listed in our roadmap is implemented, we need to have a context-aware helpful receiving end wired, i.e., something that:

- writes inputs and persists them
- takes the history of _all_ these inputs over time (long-term memory from day 0) into consideration before answering
- answers as with the right persona: no pacification scripts, objective data, no performed empathetic fillers, no sycophancy, etc.

... only then we have the bedrock to implement any of the features listed on the roadmap, but first: the shell!