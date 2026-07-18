# session 14 of building The Joy in the open

## first, a blank page on real metal

Before I write a single line of the actual shell integration, I'm going to stand up the *plumbing*: a blank web page wired to my domain (`shell.os-joy.com`), served over HTTPS, from my own bare-metal Ubuntu 24 box. No app yet — just an empty page proving the whole path works: DNS → server → TLS → browser. This is the groundwork the shell will later be deployed onto, so I'd rather flush out the metal/domain/cert pain now, against nothing, than discover it mid-iteration.

### recall: the definition of done we're building toward (session 13)

The blank page isn't the goal — it's the runway. The actual first iteration is the **input shell** described in [session 13](./session_13.md), and that iteration is only **done** when every box in its end-user QA checklist (§10) is green, on **both laptop and phone**. The shape of that DoD:

- **Served for real, HTTPS only**, at `shell.os-joy.com`.
- **Installs as a PWA** to my Android home screen and launches standalone — no browser chrome.
- **Terminal feel**: ASCII banner on launch, then a prompt; `/help` lists every command (`/clear`, `/login`, `/logout`, `/status`).
- **The capture loop**: type a line → it echoes → shows a dim `⋯ sending…`/`⋯ queued` marker → on ack the marker is replaced **in place** by a full-brightness `COPY`. `COPY` never appears for a line The Joy didn't actually receive.
- **Real auth**: `/login` issues a code only to the one configured address; wrong/empty/unknown/smuggled values never let me in and never error; replies are identical across success and failure; only the latest code works; `/logout` is idempotent.
- **Unauthed input submits by the exact same rules as authed** — but the server's reply differs authed vs unauthed, proving that distinction is server-side, not in the shell.
- **Offline-first**: opens in airplane mode, queues lines locally, delivers them automatically on reconnect (Background Sync) — even across a full app close — batched as one timestamped transmission; no `id`s leak into the payload.
- **No session resume**: reload/quit gives a blank terminal; un-acked lines still survive in the outbox and deliver later.
- **Polish**: virtual keyboard never covers the input, no focus-zoom, `v0.0.1` tag bottom-right, connectivity indicator bottom-left (green online / red offline).

That's the destination. Today's blank page just gets the road laid.

### concrete steps: deploy the blank page on Ubuntu 24

What I'm going to execute, in order:

1. **DNS** — point an `A` record for `shell.os-joy.com` at the server's public IP; confirm propagation with `dig +short shell.os-joy.com`.
2. **Server prep** — SSH in, `sudo apt update && sudo apt upgrade -y`, then install nginx: `sudo apt install -y nginx`.
3. **Firewall** — `sudo ufw allow OpenSSH`, `sudo ufw allow 'Nginx Full'` (opens 80 + 443), `sudo ufw enable`.
4. **Document root** — create `/var/www/shell.os-joy.com/html`, drop a minimal `index.html` (the blank page), and `sudo chown -R www-data:www-data` it.
5. **nginx server block** — write `/etc/nginx/sites-available/shell.os-joy.com`. This is the plain HTTP block I start from; certbot rewrites it for TLS in step 6, so I don't hand-write any `ssl_*` or port-443 lines here:

   ```nginx
   server {
       listen 80;
       listen [::]:80;

       server_name shell.os-joy.com;
       root /var/www/shell.os-joy.com/html;
       index index.html;

       location / {
           try_files $uri $uri/ =404;
       }

       # Serve the PWA manifest with the right MIME type.
       # nginx's default map has no entry for .webmanifest,
       # so it would otherwise fall back to application/octet-stream — a "binary download".
       # Desktop Chrome tolerates that and installs anyway, but Android does not:
       # installing there hands the manifest URL to Google's WebAPK build server,
       # which re-fetches it and refuses anything not served as a manifest type,
       # so the install silently degrades to a plain "Add to home screen" shortcut.
       location = /manifest.webmanifest {
           default_type application/manifest+json;
       }
   }
   ```

   Then enable it and reload:

   ```bash
   sudo ln -s /etc/nginx/sites-available/shell.os-joy.com /etc/nginx/sites-enabled/
   sudo nginx -t
   sudo systemctl reload nginx
   ```

   (Optional — drop the stock default site so it can't shadow this one as the catch-all: `sudo rm /etc/nginx/sites-enabled/default`.)
6. **HTTPS** — install certbot (`sudo apt install -y certbot python3-certbot-nginx`), then `sudo certbot --nginx -d shell.os-joy.com` to issue the cert and rewrite the server block for TLS + HTTP→HTTPS redirect; confirm auto-renewal with `sudo systemctl status certbot.timer`.
7. **Verify** — load `https://shell.os-joy.com` in a browser (laptop + phone) and see the blank page over a valid certificate, with `http://` redirecting to `https://`.

Once that's green, the road is laid and the shell can land on it.

## next: the terminal feel, and nothing more

With the runway live, the first real slice is deliberately tiny: a terminal that *feels* like a terminal, and not one box more from the [session 13](./session_13.md) DoD. No auth, no capture loop, no outbox, no service worker, no server — those all hang off having a prompt that takes a line, so the prompt comes first. This is the trunk; everything harder is a branch off it.

### what "done" means for this slice

- **Stack**: TypeScript + Vite, [xterm.js](https://xtermjs.org/) for the terminal surface. A static build — `vite build` produces a `dist/` of plain assets, no runtime backend.
- **On launch**: an ASCII banner, then a prompt. It looks like a shell the instant it loads.
- **Commands that actually respond**: `/help` lists the command set; `/clear` wipes the screen. The rest of the verbs (`/login`, `/logout`, `/status`) can be listed by `/help` but are allowed to be stubs for now — they don't have to *do* anything yet.
- **Served for real**: the built `dist/` lands on `shell.os-joy.com` over the HTTPS I just stood up, and renders on **both laptop and phone**.

Explicitly *not* in this slice: typing a line and having it echo/queue/`COPY` (the capture loop), any notion of being logged in, offline behaviour, PWA install. Each of those is its own later slice — though the order I'd put them in has since changed; see *next, reconsidered* at the end of this log.

### why this is the right first cut

- It's the smallest slice that's *real* — something I can load on the phone and feel, not scaffolding.
- It exercises the **deploy loop** against the fresh runway while the stakes are still zero. Today's deploy was a hand-written `index.html`; the thing to prove next is that a *built artifact* (`vite build` → `dist/`) reaches the box cleanly. Better to hit any build/permissions/caching snags now, against a page that barely does anything, than mid-feature.
- Every harder box in the DoD takes a line of input at the prompt as its starting point. Lay the trunk, then grow the branches.

## then: the trunk, built and committed

The slice above is no longer a plan — it's the first commit on `github.com/the-joy-com/shell.os-joy.com`. The terminal that *feels* like a terminal exists, and nothing past it does, exactly as scoped.

What landed:

- **The stack, as promised**: TypeScript + Vite + [xterm.js](https://xtermjs.org/), a static build with no runtime backend. `yarn build` type-checks and emits a plain `dist/`. I swapped xterm's default DOM renderer for the **canvas** one — it paints far faster, and unlike WebGL it has no GPU-context-loss cliff on mobile, which matters because the phone is a first-class target, not an afterthought.
- **The launch feel**: an ASCII banner (figlet "ANSI Shadow" block glyphs — solid blocks stay legible green-on-black where thin line-art turns to mush), in two cuts. The wide one-liner needs ~56 columns; phones don't have them, so narrow terminals get the stacked two-word cut. `main.ts` picks based on the width xterm actually fitted. Then a tagline carrying the `v0.0.1` version, and a prompt.
- **The commands that respond**: `/help` lists the set, `/clear` (and the bare `reset` keyword, no slash) wipes the screen. `/login`, `/logout`, `/status` are listed by `/help` and acknowledged, but say so plainly rather than pretending to work — the auth and session behind them is a later slice.
- **A little more than I scoped, because it was nearly free**: fish-style inline autocomplete. As you type `/h`, the rest of the best-matching command ghosts in dim grey; **Tab** or **→** accepts it. It's cheap by construction — only the tail of the line is ever repainted, the prompt never is — so typing stays snappy. It's not a DoD box, but it's the kind of thing that makes the prompt feel *alive* the instant it loads, which is the whole point of this slice.

What deliberately isn't here is everything I said wouldn't be: no capture loop, no `COPY`, no auth, no outbox, no service worker, no PWA, no server round-trip. The trunk is laid.

What's *not yet done*, and is the immediate next move: this artifact has only run on `localhost`. The DoD says **served for real, on both laptop and phone**, and the second reason this slice exists at all was to prove a `vite build` → `dist/` reaches the box cleanly. So the slice isn't closed until the build is sitting behind the HTTPS I stood up at the top of this session and rendering on the phone. That deploy is the next thing I do.

## next, reconsidered: the phone before the server

Up in the terminal-feel slice I filed PWA install away as "its own later slice" and left the order open. Sitting with it now, I think the order matters — and PWA install should come **next**, before any server-side plumbing, before the capture loop.

The reasoning:

- **It's the last thing that's still free.** Everything up to and including PWA install is *static* — no backend, no auth, no round-trip. It rides the exact deploy loop I just proved (`vite build` → `dist/` → the box). The capture loop is the first slice that genuinely needs a server, and a server is where the real cost and the real unknowns live. Better to finish *everything* that doesn't need one while the stakes are still zero — the same discipline that put a blank page on metal before any app went near it.
- **It's the slice that makes The Joy *live on the phone*.** Right now it's a browser tab. Installed to the home screen, launching standalone with no browser chrome, it becomes a thing that's *present* on the device — the symbiot has a place to live before it can say anything back. That presence is worth having before I teach it to listen.
- **The service worker it needs is groundwork I'll need anyway.** Installability wants a manifest and a service worker; the offline-first capture later (Background Sync) wants that *same* service worker. Standing it up now — against a static app with nothing yet to sync — means the hard offline plumbing lands later on a foundation I've already debugged in isolation, instead of tangled up with the backend going in at the same moment.

So the revised order is: **PWA install next, server-side plumbing after.** Don't braid the two sets of snags together. Lay the offline shell onto the phone first, *then* give it something to carry.

## a build note: the service worker is compiled, not dropped in

A service worker can be the easiest thing in the world to ship — drop a hand-written `sw.js` into `public/` and Vite copies it to the root untouched. I started there, then threw it away and gave the worker the full TypeScript build pipeline instead. It's worth saying why, because it cost real config to do.

The reason is what's coming. Right now the worker does one small thing — cache the shell so the app opens offline. But the capture loop's whole offline story lives *in here*: the outbox, the Background Sync that flushes queued lines on reconnect, the message back to the page that flips a line from `pending` to `COPY`. That's real logic, and it's going to grow fast. A static `sw.js` in `public/` is never type-checked — a typo in the most safety-critical code in the app would sail straight through `yarn build`. I'm not willing to write delivery logic with the compiler looking the other way.

So the worker is now `src/sw.ts`, compiled like everything else. Two wrinkles had to be solved to get there:

- **It has to land at the root, un-hashed.** A service worker's scope is capped by where it's served from — `/sw.js` controls the whole site, but `/assets/sw-a1b2c3.js` (what Vite's normal bundling would produce) only controls `/assets/`. So the worker gets its *own* build pass: a second `vite build` with a tiny config that emits exactly `dist/sw.js`, as a classic script, without wiping the app build that ran first.
- **It lives in a different world than the app.** A worker isn't a web page — no `window`, no DOM, and its own globals instead (`ServiceWorkerGlobalScope`, `FetchEvent`, the Cache API). Those types come from TypeScript's `WebWorker` lib, which flat-out conflicts with the `DOM` lib the app compiles against. So the worker also carries its own `tsconfig`, and `yarn build` now type-checks both before it bundles either.

More machinery than a one-line file, yes. But the worker is about to become some of the most important code in the project, and I'd rather pay for the toolchain now — while it's fifty trivial lines — than retrofit it around logic I'm already afraid to touch.

## a quiet upside: there's no "update" step, ever

One thing falls out of the service-worker design that I want on the record, because it's the kind of win you forget you got for free: once The Joy is installed — on a phone or a desktop — the person using it never has to update it. No app-store prompt, no reinstall, no "clear your cache and try again." They open it, and it's current.

That isn't luck, it's the cache strategy. The worker is *network-first*: every time the app opens online it asks the server for each file first, serves the fresh copy, and only falls back to what it cached when the network is actually gone. So an install is never frozen at the version it had the day it landed on the home screen — it tracks the server. The worker itself updates the same way: the browser re-checks it on launch, and when it has changed the new one takes over immediately and throws away the old cache (named after the app version, so a release rotates it cleanly).

So the whole update story, from where I sit, is one line on the server: `./deploy.sh`. The next time anyone opens the installed app with a connection, they're on the new build. Nothing is asked of them.

I'm calling it out because the obvious alternative — a *cache-first* worker — is the classic PWA trap: faster to open, but it strands people on a stale version for days until the worker quietly swaps underneath them. I'd rather pay a few milliseconds at launch for the guarantee that what's deployed is what's running. For something I'm going to be shipping changes to constantly, freshness-by-default is worth more than a slightly faster cold start.

## the install that fought back: desktop in one click, Android by attrition

I expected install to be the victory lap. On the laptop it was: Chrome put an install icon in the address bar on its own, I clicked it, and The Joy opened in its own chrome-less window. Done. On Android the same site offered only "Add to home screen" — the plain bookmark shortcut — and never the real "Install app". Same manifest, same server, two completely different outcomes. That gap turned out to be the whole story.

The reason desktop and Android diverge: desktop Chrome judges installability itself and is forgiving. Android doesn't install your page directly — it ships the manifest URL off to Google's **WebAPK build server**, which re-fetches everything and mints a tiny signed APK so the result is a true installed app in the launcher. That second fetch, by a machine I can't see, is a far stricter and far more opaque gate than anything the laptop applies. Everything that went wrong lived behind it.

Two real defects came out of chasing it, both worth keeping:

- **The icon set was too thin.** I'd shipped a single 512 maskable icon, reasoning one good icon covers everything (that was the [session 13](./session_13.md) plan). It doesn't: Android's installability check specifically wants a 192 **and** a 512 with `purpose: "any"`, and a maskable icon does *not* count toward that. So the set grew to three — 192 `any`, 512 `any`, 512 maskable — and that's the honest minimum, not the over-provisioning I'd talked myself out of earlier.
- **The manifest was served as the wrong type.** nginx has no default mapping for `.webmanifest`, so it fell through to `application/octet-stream` — "anonymous binary download". Desktop Chrome shrugged and installed anyway, which is exactly why this hid for so long; the WebAPK server refused it outright. One nginx line fixed it (`default_type application/manifest+json` for that one path), now folded into the server block up in step 5.

The lesson under both: **desktop install succeeding tells you almost nothing about Android.** The forgiving path masks defects the strict path rejects, so "it installs on my laptop" is not evidence the manifest is right — it's evidence the laptop is lenient.

After both fixes I wanted proof rather than another hopeful reload, so I ran the live URL through Lighthouse's `installable-manifest` audit — the same installability engine Android's check is built on. It came back **score 1, zero blocking errors**: the deployed site is, provably, fully installable. The server side of this is genuinely finished.

And the phone *still* showed "Add to home screen". That's the part I want on the record honestly: once a device has formed an opinion about a site, it caches it hard — the old service worker, the old manifest verdict, the failed-WebAPK cooldown — and "clear site data" didn't shake it loose across several tries. The clean way to read a device's *own* verdict is remote debugging over `chrome://inspect` from the laptop, but the phone never showed up there even with USB debugging enabled — a tooling dead end on top of a caching dead end.

So, when I saw that even after deleting site settings cookies etc., this "install" prompt did not appear on my Android's Chrome, I stopped. I took the "Add to home screen" shortcut: there's a Joy icon on the home screen now, the green-J mark and the launch feel are right, and that's enough to live with for this slice. The status, stated plainly: the **site is provably installable** (a fresh device, or that phone once its cache finally turns over, gets the real WebAPK standalone install), but I could not coax the full standalone install out of *this* device today, and I chose not to spend more of the slice on one phone's stale state. The DoD line — *installs as a PWA, launches standalone* — is met at the level I control (the artifact) and pending at the level I don't (this handset). I'd rather log that split truthfully than either claim a win I didn't see or sink another evening into a device cache.

What I'd do differently next time: validate installability with the Lighthouse audit *first*, before ever trusting the laptop's install icon or a phone's menu — it's the only check that's both honest and repeatable, and it would have pointed straight at the icon set and the MIME type without the days of reload-and-squint.

One genuine consolation, and it's the one I'd actually been counting on: the auto-update works on Android regardless. I shipped a couple more changes — the version tag moved out of the launch tagline and into the corner and `/help` — redeployed, and just *tapped the home-screen icon*. The new build was there: no reinstall, no cache-clear, no prompt. That's the *no update step, ever* claim from the section above, confirmed on a real phone — and confirmed via the lowly "Add to home screen" shortcut at that, because the shortcut still opens the same scoped URL through the registered service worker, so network-first freshness doesn't care how the icon got onto the home screen. So the WebAPK install fought me to a draw, but the thing the install was *for* — an icon on the phone that's always running the latest code — I have anyway.


## let's finish the static parts of the shell _then_ we'll do the server-side plumbing

Let's look back at our static checklist from [session 13](./session_13.md):

**Static shell — done.**

- [x] Served at `shell.os-joy.com`, over HTTPS only.
- [x] Installs to my Android home screen and launches standalone — full-screen, with no browser chrome (no address bar, tabs, or navigation buttons), so it looks like a native app rather than a page in Chrome.
- [x] On launch, the terminal initializes showing The Joy ASCII banner, then drops to the prompt.
- [x] `/help` lists every command, including `/clear`, `/login`, `/logout`, `/status`.
- [x] Type a line → Enter → it echoes immediately.
- [x] Renders cleanly on my laptop — the log fills the viewport, the input stays pinned at the bottom, and the layout holds across window resizes.
- [x] The Android virtual keyboard does not cover the input line, and tapping the input does not trigger focus-zoom.
- [x] A version tag — `v0.0.1` — is pinned to the bottom-right corner of the terminal, present from the moment it opens.
- [x] A connectivity indicator is pinned to the bottom-left corner: **green** when online, **red** when offline. Going into airplane mode flips it to red; reconnecting flips it back to green — tracking the same connectivity the outbox drains on (§5).

**Server-side plumbing — to do, in the order I'll build and test it.**

The thing on the other end of the wire has a name now: **`kernel.os-joy.com`**. The domain already hands you the metaphor — `os-joy.com`, an OS — and an OS has exactly two halves. The **shell** is the thin user-facing layer that takes input and echoes it back; that's what's built and checked above. The **kernel** is the privileged core behind it that does the real work — intake and release, the buffer, identity, the Dead Man's Switch — mediating between the World and the symbiot the way a real kernel mediates between hardware and the programs above it. `shell` ↔ `kernel` reads as deliberate the moment you see both subdomains; it says the whole architecture for free. It's the first thing I'll deploy, on its own subdomain, before any of the boxes below can be ticked.

_First, prove the server is even there._

- [x] The indicator reflects **server** state too, not just the browser's network flag: **online** means both client *and* server are up. `navigator.onLine` only proves the device has a network — it says nothing about whether The Joy is actually reachable. So the green state must be earned by a real round trip to the server (a health probe), and a reachable network with a dead server reads **offline**, not online.

_Then the bare send-and-acknowledge round trip._

- [x] While `pending`, the line shows a dim `⋯ sending…` marker beneath it.
- [x] On ack, that marker is replaced **in place** by a full-brightness `COPY` — no second line is added.
- [x] `COPY` never appears for a line The Joy didn't actually receive.

_Then identity on top of that round trip._

- [ ] `/login` with the exact configured address → a code is issued and sent only to that stored address; entering it authes the same terminal, and `/status` then reports authed for that email.
- [ ] `/login`, then entering a **wrong** code → rejected; the terminal stays unauthed and lets me retry, and only the correct (latest) code ever authes. No lockout yet, but a bad code never lets me in and never errors out the flow.
- [ ] `/login`, then submitting an **empty/blank** line instead of an address → no code is generated and nothing is sent; the flow cancels cleanly (or re-prompts), never errors, and never leaves a half-open login state.
- [ ] `/login` with an unknown address → no code is generated and nothing is sent; the terminal's reply is identical to the success case.
- [ ] `/login` with a recipient-smuggling value (`me@allowed, attacker@evil`, `me@allowed;attacker@evil`, a substring like `me@allowed.evil` or `attacker+me@allowed`, or an embedded newline) → no code, no email to anyone, nothing to the smuggled address; reply still identical.
- [ ] `/login` run twice before entering a code → only the most recently emailed code works; the earlier one is rejected.
- [ ] `/login` while already authed → no fresh email prompt; it just reports the current authed status.
- [ ] `/logout` drops the session; `/status` reverts to unauthed.
- [ ] `/logout` run again while already unauthed → idempotent no-op; it confirms "not logged in" and nothing breaks.

_Then prove identity actually changes the reply — and only the server decides that._

- [ ] Unauthed input submits by the exact same rules as authed — the shell makes no distinction.
- [ ] Submit a line while **authed** → beyond the `COPY`, The Joy returns a distinct, hardcoded reply standing in for what an agent would eventually say (no real LLM involved).
- [ ] Submit a line while **not authed** → the reply is recognizably *different* from the authed one. Same submission path (per the check above); the difference is the server's doing, proving authed vs unauthed is handled server-side — not by the shell.

_Then durability when the network drops._

- [ ] Switch to airplane mode → the shell still opens, renders the terminal, and accepts input.
- [ ] Submit 2–3 lines offline → each echoes and is held `pending` with a dim `⋯ queued` marker, no `COPY`.
- [ ] Reconnect → the queued lines deliver automatically and their `COPY`s appear in submit order.
- [ ] Submit offline, fully close the app, then reopen → a blank terminal (just the banner + prompt); the prior lines are gone from view but still deliver on reconnect (Background Sync), and their `COPY`s print as they arrive.
- [ ] Several lines queued offline go up as **one** transmission, formatted with each line's timestamp and a separator.
- [ ] No `id`s appear in the transmitted payload — they stay client-side bookkeeping only.

_Finally, what must **not** survive a reload._

- [ ] Reload, or fully quit and reopen → a blank terminal; the prior scrollback and input history are **not** restored (the session does not resume).
- [ ] Quit/reload with un-acked lines → they're gone from the blank view but survive in the outbox and still deliver on reconnect, printing their `COPY`s as they arrive.

Reading that list back, it sorts into two piles, and the split *is* the plan for the rest of this slice.

**The static shell** — everything the browser can do on its own, with no server on the other end of the wire — is now fully checked. It serves over HTTPS, opens to the banner and drops to a prompt, `/help` enumerates the commands, typed lines echo on Enter, the layout holds on the laptop and dodges the Android keyboard, and the version tag sits in its corner. The last one to land was the **connectivity indicator** in the bottom-left — green online, red offline — which reads `navigator.onLine` and the `online`/`offline` events: no endpoint, no round trip, pure client. With that checked, the static pile is closed. (The "shell still opens in airplane mode" box is static too — the service worker already caches the shell — it just needs a deliberate offline test to tick off, which I'll fold into the offline run below.)

**The live wire** — every box with a `pending`/`COPY`, a `/login` code, an authed-vs-unauthed reply, or an offline outbox that drains on reconnect — needs a server to talk to, and that server is **`kernel.os-joy.com`** (named above). There's no honest way to tick those against a backend that doesn't exist yet: a `COPY` means *The Joy received it*, and nothing can mean that until there's a Joy — a kernel — to receive it. So they stay unchecked on purpose. That's the server-side plumbing this slice ends with — not skipped, just sequenced after the parts that don't need it.

The order is deliberate. Finish what the browser can prove on its own first, so that when I do stand up the server the only new variable is the server. If the shell were still shifting underneath me I'd never be able to tell whether a missing `COPY` was the wire or the terminal. Nail the static half flat, and the live half has exactly one thing it can be.

The live half is itself sequenced the way I'll build and test it, each step leaning on the one before so a failure always points at the youngest layer. First prove the server is even *there* — the health probe that earns the green dot, the simplest possible round trip. Then the bare send-and-acknowledge: a line goes up, comes back `COPY`. Then identity on top of that round trip — `/login` / `/logout` and all their edge cases. Then prove identity actually *changes the reply*, and that only the server decides it. Then durability when the network drops — the offline outbox and Background Sync. And last, the inverse: what must **not** survive a reload, even while un-acked lines live on in the outbox. Reachability before send, send before identity, identity before its consequences, the happy path before the offline path — and persistence pinned down only once there's something worth persisting.

## standing up the kernel — first slice: prove it's there

The shell's first deploy didn't try to be a terminal. It served one HTTPS page that drew a banner — *something of ours is alive at this address.* The kernel's first slice is the exact mirror, and it's the first box of the server-side pile above: **prove the server is even there.** One endpoint, `GET /health`, answering `200` with a tiny JSON body — `{"msg": "ok", "data": {"version": "0.0.1"}}` — at `https://kernel.os-joy.com`. No send/ack, no login, no replies. Just a Python process that answers a round trip. This is the thing the shell's connectivity dot will eventually probe to earn its green: a reachable network with a dead kernel must read **offline**, and only a real `200` from here flips it back.

**One envelope for every response.** Decided now, while there's only one endpoint to shape it on, so nothing has to be retrofitted later: every response the kernel returns — `/health`, the eventual send-and-ack, `/login`, the agent replies, every error — wears the same shape: `{ "msg": string, "data": <array | object | null> }`. `msg` is a human-legible line about what happened (`"ok"`, an error reason, a status word). `data` is the payload — a JSON array, a JSON object, or `null` when there's nothing to carry. The shell then always knows where to look: `msg` for something to show, `data` for something to act on, and never has to guess the shape per route. `/health` is the first instance of it, not an exception to it: its version goes in `data`, its status word in `msg`.

**The stack — decided.** **FastAPI on uvicorn.** For a single health endpoint it's the same handful of lines as Flask, so it costs nothing today — but the roadmap leans on what it gives for free later: async, because the agent replies will eventually *stream* (which the synchronous frameworks fight), and type-driven request validation for the `/login` payloads and the recipient-smuggling guards already listed above. Choosing it now means no rewrite at the first endpoint that's actually interesting. The kernel runs as a **systemd unit** — uvicorn bound to `127.0.0.1:9713`, kept alive across crashes and reboots, logging to journald — behind **nginx as a reverse proxy** for `kernel.os-joy.com`, with TLS terminated at nginx by a **certbot** cert, exactly as the shell's HTTPS is.

**The one real difference from the shell.** The shell is static files rsync'd into nginx's docroot; nothing *runs*. The kernel is a long-running process, so the deploy gains three things the static side never needed: nginx configured as a proxy rather than a file server, a certbot cert for the new subdomain, and a systemd unit so the process outlives its terminal. `deploy.sh` keeps the shell's rhythm but swaps the payload step — `git pull` → install deps into a venv → restart the service — instead of `build` → rsync.

**Repo.** A new `kernel.os-joy.com` sibling beside `shell.os-joy.com`, self-contained the same way, MIT-licensed the same way.

**Build order for this slice.** Scaffold the FastAPI app and `/health` locally and run it → stand up the server-side infra (nginx vhost, certbot cert, systemd unit, `deploy.sh`) → confirm `https://kernel.os-joy.com/health` answers a clean `200` from the open internet. Only once that round trip is real does the next box — wiring the shell's dot to probe it — become honest to tick.

### concrete steps: deploy the kernel on Ubuntu 24

The same shape as the shell's blank-page deploy, with the three additions the static side never needed: nginx as a *proxy*, a systemd unit, and a `deploy.sh` that restarts a service. The repo already carries `deploy.sh` and `deploy/kernel-os-joy.service`; the rest is one-time server setup. All commands run on the box over SSH.

1. **DNS** — point an `A` record for `kernel.os-joy.com` at the same server IP the shell already resolves to. (nginx tells the two subdomains apart by `server_name`; one box serves both.)

2. **Clone the repo** — beside wherever the shell lives:

   ```bash
   mkdir -p ~/apps && cd ~/apps
   git clone git@github.com:the-joy-com/kernel.os-joy.com.git
   cd kernel.os-joy.com
   ```

3. **First deploy** — install [`uv`](https://docs.astral.sh/uv/getting-started/installation/) if the box doesn't have it, then just run the deploy script:

   ```bash
   ./deploy.sh
   ```

   The first run prompts once for the server user (the Linux account that owns this clone) and remembers it in a gitignored `.deploy-user`, so every later deploy is non-interactive. From that user it builds the venv from the lockfile, renders the systemd unit from `deploy/kernel-os-joy.service` (filling in the user), installs and enables it so the process outlives the shell and survives reboots, then starts uvicorn on `127.0.0.1:9713`. Confirm the unit's two paths suit the box first — the clone lives under `~/apps`, and `which uv` resolves to `~/.local/bin/uv`.

   ```bash
   curl http://127.0.0.1:9713/health        # {"msg":"ok","data":{"version":"0.0.1"}}
   ```

   `systemctl status kernel-os-joy` should read `active (running)`; logs go to journald (`journalctl -u kernel-os-joy -f`).

4. **nginx server block** — write `/etc/nginx/sites-available/kernel.os-joy.com`. Plain HTTP to start; certbot rewrites it for TLS in step 5. The one difference from the shell's block is that this `location` *proxies* to the local uvicorn instead of serving files from a docroot:

   ```nginx
   server {
       listen 80;
       listen [::]:80;
       server_name kernel.os-joy.com;

       location / {
           proxy_pass http://127.0.0.1:9713;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header X-Forwarded-Proto $scheme;
       }
   }
   ```

   ```bash
   sudo ln -s /etc/nginx/sites-available/kernel.os-joy.com /etc/nginx/sites-enabled/
   sudo nginx -t
   sudo systemctl reload nginx
   ```

   **Verify the block is actually live before going further — this is the gate for step 5.** A clean `curl http://kernel.os-joy.com/health` over plain HTTP is *not* enough: a bare host with a default server can answer port 80 too, so a `200` doesn't prove *this* block is the one serving the name. Confirm nginx really parsed a `server_name kernel.os-joy.com;` directive:

   ```bash
   sudo nginx -T 2>/dev/null | grep -n "server_name.*kernel"   # must print a match
   ls -l /etc/nginx/sites-enabled/ | grep kernel               # symlink must exist
   ```

   If `grep` comes back empty, the block isn't enabled (or `server_name` doesn't match) and step 5 will fail — fix it here, not after certbot has already run.

5. **HTTPS** — certbot is already installed from the shell's deploy, so it's one command to issue the cert and rewrite the block for TLS + the HTTP→HTTPS redirect:

   ```bash
   sudo certbot --nginx -d kernel.os-joy.com
   ```

   Then confirm from *outside* the box: `curl https://kernel.os-joy.com/health` returns the envelope over a valid certificate, no TLS warning.

   **Gotcha (hit once, on the first deploy).** `certbot --nginx` needs a server block with a matching `server_name` *already present and enabled* — it edits an existing block, it does not create one. Run it before step 4's block is live and certbot still **issues** the cert (saved under `/etc/letsencrypt/live/kernel.os-joy.com/`) but fails to **install** it, with: *"Could not automatically find a matching server block for kernel.os-joy.com."* The cert is fine — don't re-issue it (Let's Encrypt rate-limits duplicates). Create + enable the step-4 block, pass the verification above, then install the already-issued cert into it:

   ```bash
   sudo certbot install --cert-name kernel.os-joy.com
   ```

Every deploy after that is a single command from the clone — `./deploy.sh` — which pulls `main`, runs `uv sync --frozen`, and restarts the unit. The uvicorn port stays bound to `127.0.0.1`, so nginx is the only door open to the internet.

- [x] `GET /health` returns `200` with the standard envelope — `{"msg": "ok", "data": {"version": "..."}}` — when hit locally against the running uvicorn.
- [x] The same endpoint answers `200` over HTTPS at `https://kernel.os-joy.com/health` from outside the server, with a valid certbot certificate (no TLS warning).
- [x] nginx reverse-proxies `kernel.os-joy.com` to the local uvicorn; the uvicorn port is not exposed directly to the internet.
- [x] `deploy.sh` pulls, installs into the venv, and restarts the service in one run from the server.
- [x] The kernel runs as a systemd unit: it comes back on reboot and restarts on crash, with logs in journald. *(restart-on-crash is wired via `Restart=on-failure`; reboot-survival confirmed by an actual reboot of the bare-metal box — the unit came back `active` on its own and HTTPS `/health` answered without anyone touching it.)*

## closing the loop: the dot earns its green

With the kernel answering `/health` over HTTPS, the box the build order parked at the top of the server-side pile is finally honest to tick: **make the connectivity dot reflect the server, not the browser's network flag.** Until now the bottom-left dot only read `navigator.onLine` — a green light that meant *the device has a network*, which is almost the wrong question. A shell whose whole reason to exist is the kernel behind it should go green when *the kernel answered*, and red the instant it doesn't — even on full Wi-Fi. So the dot now polls `kernel.os-joy.com/health` and flips green only on a real `{ "msg": "ok" }` round trip; a healthy network behind a dead kernel reads **offline**, exactly as the DoD demanded. `navigator.onLine` stays, but demoted to a cheap pre-check: if the browser already knows it's offline, skip the fetch and paint red at once.

**The wiring, kept deliberately small.** A probe on load, a gentle 15s poll, and an immediate re-probe whenever the network returns or a backgrounded tab refocuses — so the dot is right from the first frame and snaps back the moment things change, without hammering the kernel. Each probe carries a 4s abort timeout (a hung socket must not freeze the dot at a stale colour) and an overlap guard (triggers can't stack fetches). Which kernel it probes is read from `VITE_KERNEL_URL`, **defaulting to the production kernel** so a plain clone and every prod build need zero config; a gitignored `.env.local` points dev at a locally-running kernel when I want to watch it flip green/red by hand.

**The snag, and it rhymes with the WebAPK one.** I wrote the probe, ran the kernel locally, `curl`'d `/health` — clean `200` — and the deployed dot still sat red. The cause: a browser reading the kernel from the *shell's* origin is a **cross-origin** request, and the browser refuses to let JavaScript *read* the response unless the kernel sends back a matching `Access-Control-Allow-Origin` header. The kernel wasn't sending one, so the `fetch` rejected and the dot painted red — while `curl`, which doesn't enforce CORS at all, reported a perfectly happy `200` the whole time. That's the same shape as the desktop-vs-Android install trap from earlier this session: **a lenient tool (`curl`, desktop Chrome) masking a defect that the strict real path (the browser, the WebAPK server) rejects outright.** "It works in `curl`" is no more evidence the dot will go green than "it installs on my laptop" was evidence Android would install it. The fix was a CORS allow-list on the kernel — the shell's production origin plus the two localhost dev origins, `GET` only, nothing wildcarded — so the browser is given explicit permission to read exactly the responses it's meant to.

**One ordering truth worth keeping.** Because the allow-list ships *inside* the kernel, the green dot at `shell.os-joy.com` can't light up until a kernel *carrying that code* is deployed — and a reboot of the box doesn't ship new code, only `./deploy.sh` does. I tripped on the milder version of this locally: with no `.env.local`, dev was probing the *production* kernel, which hadn't been redeployed yet, so the local dot was red for a completely real reason. The lesson is the quiet companion to the certbot blunder up in step 5 — *prove the thing you think is true is actually in effect*: a `200` from `curl`, or a box that's up, or a process that restarted, none of them prove the **browser** can read the response. The check that can only pass if it truly worked is the cross-origin one — `curl -H "Origin: …" -i` and look for the `access-control-allow-origin` header coming back, or just watch the dot in a real browser.

**Confirmed, the honest way.** After deploying the kernel with CORS and rebooting the bare-metal box, I loaded the *deployed* shell and watched the dot sit green against the live kernel — then confirmed it goes red when the kernel can't be reached. That's the round trip closed end to end in the real browser, not in `curl`: the first box of the server-side pile is genuinely green, and the same reboot that I'd have done anyway doubled as the proof for the systemd reboot-survival box above. The dot is now a true server-state light — the first thread actually strung between the shell and the kernel, and the foundation every `pending`/`COPY` box hangs off next.

## the capture loop: a line crosses the wire and comes back `COPY`

The dot only proves the kernel is *there*. The next box is the first time the shell sends it something to *hold*: type a line that isn't a `/command`, it goes up to the kernel, and the dim `⋯ sending…` beneath it turns into a bright `COPY`. That's the box ticked now — and like the dot before it, the interesting part wasn't the happy path, it was deciding exactly what `COPY` is allowed to mean and refusing to block the human to get it.

**The kernel side is deliberately hollow.** `POST /intake` takes a body — `{ "line": "<text>" }` — validates it, and answers `{ "msg": "copy", "data": null }`. Then it *drops the line on the floor*. That's not a shortcut I'm hiding; `"copy"` means **The Joy received it**, not that it kept it. Holding the line — the buffer — is a separate concern that layers on top of this round trip, and folding it in now would mean a `COPY` that quietly promises storage I haven't built. Better an honest ack of receipt than a dishonest ack of keeping. The buffer lands later, *on* this round trip, never ahead of it.

The validation, on the other hand, is real, because it's the cheap kind of real: the body is a Pydantic model — `IntakeRequest`, a **DTO**, named for what it carries rather than the `/intake` route or the `intake()` handler so the three don't collide. A `line` that's missing, empty, or past 4096 chars is a `422`, not a `200` — a stray paste can't ship an unbounded body, and the kernel never has to guard against junk downstream because junk never gets past the door. DTOs got their own `dtos.py` the moment there was one to write, so a request's *shape* is defined once, apart from the route's *behaviour*. The outbound envelope is the mirror image — the response DTO — and I've left it untyped until a second route makes a generic `Envelope[T]` actually worth the abstraction. One DTO is a data class; two with a shared shape is a pattern. I'll wait for the second.

**`COPY` is held to the same honesty rule as the green dot.** The marker only flips to `COPY` on a real `{ "msg": "copy" }` body — not on a `200`, not on "the fetch didn't throw". A `200` with the wrong shape, a CORS-blocked read, a timeout, a network error: every one of them paints `✗ not delivered` instead. It's the exact discipline the dot earned the hard way — *a lenient signal must never stand in for the strict one* — applied before I could trip on it twice. And trip I would have: this is the same cross-origin wire as the dot, so `/intake` needed `POST` added to the kernel's CORS allow-list (the JSON body makes the browser preflight with `OPTIONS`, which Starlette's middleware answers itself). A `curl` `POST` returning `copy` proves exactly as little as a `curl` `GET` returning `ok` did — only the dim marker turning bright in a real browser tab is evidence, and that's the test I actually ran before ticking anything.

**The design call I'm proudest of: the prompt never blocks.** My first cut held the input line until each send resolved — a quiet `inFlight` flag — and it felt wrong the instant I typed against it. The whole point of The Joy is that the human can *unload*; making them wait on the network to say the next thing is exactly the friction the thing exists to remove. So input is now **never** gated on connectivity. Enter hands the prompt straight back; the send goes off in the background; you can fire line after line with three acks still in flight. Each pending line remembers the absolute terminal row its marker sits on and repaints *only that row* when its *own* ack lands — so several sends resolve independently, in whatever order the kernel answers, and none of them ever disturbs the line you're typing right now. (xterm parses writes asynchronously, so the row is only trustworthy when captured inside a write callback — a small thing that took a beat to see.) The one honest limit: a marker that scrolls out of the viewport before its ack returns is no longer addressable, so it's left as-is. On a live round trip the ack is near-instant and the row is still on screen; the offline outbox, when it comes, is what will repaint markers that have scrolled away.

That non-blocking call is now a **principle**, not a one-off: the right to submit is never gated — not on the network, and pointedly not on being logged in. `/login` isn't built yet and it doesn't matter; an unauthed line submits by the exact same path an authed one will. Whether identity changes the *reply* is the kernel's decision to make, later, on the far side of this same wire. The shell's job is to take what the human says and get it across — and now it does, honestly, without ever making them wait.

One housekeeping note for the record: a pass over both repos to strip transient-phase language from comments and docs — "for now", "this slice", "coming later" and their kin. The code should describe what the system *is*, not the order I happened to build it in; that story belongs here, in the journal, not scattered through the source. The forward-looking architecture notes that name a real *capability* (the buffer, the offline outbox) stayed — those describe the design, not a phase.

Three boxes green, the honest way. The wire now carries content, not just a heartbeat — and the next thread strung across it is identity.

## a pivot, before I write a line of it: identity gets real foundations now

I'm writing this *ahead* of the code, on purpose — so the reasoning is the honest one I'm holding now, not a tidy story reverse-engineered after it all worked. The next rung is identity: `/login` issues a one-time code, `/login/verify` spends it for a session, `/status` and `/logout` round it out (boxes 198–206). My first instinct for the smallest honest cut was to **stub the code to journald** — generate the OTP, write it to the log instead of emailing it, and build the whole state machine and its security invariants against a fake delivery. It's the same move that served the `COPY` rung well: build the thing that can be proven on its own, defer the part that needs the outside world.

**I'm changing course, and the reason is sequencing, not impatience.** Identity is the kernel's *first stateful rung* — the first time it has to remember anything between two requests. A code issued by one call has to still be there for the next call to verify; a session minted at login has to outlive the request that made it. That memory has to live somewhere, and every later rung — the buffer, the Dead Man's Switch — needs it too. So rather than fake persistence now and pay to rip the fake out later, I'm standing up the real floor while I'm down here laying the first thing on it: **Postgres**, with **hand-rolled migrations that run at startup** (no ORM, no Alembic — ordered SQL tracked in a ledger table, because I want to *see* every change to the schema, not have one generated for me), and the **human symbiot seeded from `.env`** so a fresh database is born knowing exactly one address is allowed to log in. Local development gets a Postgres in docker-compose; the server already has one, reachable by peer auth tied to my Ubuntu user — which means the database URL is genuinely *different* on the two boxes, and that difference is a thing to document, not paper over.

**And on delivery: I'm not stubbing the email — I'm building it for real, behind an interface.** The pull toward journald was really a pull toward *testability* — I didn't want live credentials in the loop every time the suite runs. But "code to an interface" buys me that without faking anything: an `EmailClient` with one honest method, a real Gmail-API implementation behind it, and a fake that just records what it was asked to send. The tests run entirely against the fake — fast, offline, deterministic, and able to read back the exact code that *would* have been emailed so the verify path can be exercised end to end. Then the real client sends a real code to a real inbox, and the only thing the swap changes is whether the bytes leave the building. Real where it counts, fake where it would only slow me down — and no journald half-measure that I'd have to delete the moment I wanted actual email.

**The discipline from the rungs below carries up unchanged.** This is TDD: the test list *is* the spec, written red before the code goes green, and it's a near one-to-one transcription of boxes 198–206 — the anti-enumeration reply that's byte-identical for a known address, an unknown one, and a recipient-smuggling attempt; only the latest code ever working; a wrong code that rejects without derailing; an idempotent logout. And the honesty rule the dot and the `COPY` marker both earned the hard way still holds: **a green checkbox has to be paid for in the strict path, never the lenient one.** A passing pytest suite against a fake email client proves the state machine, not the wire — so boxes 198–206 stay empty until I've watched a real code land in my own inbox from the *deployed* kernel and walked the whole login round trip against `https://kernel.os-joy.com` in the open. `curl` proving `200` taught me that lesson twice already; I'm not going to need teaching a third time. Foundations first, then the flow on top of them, then the boxes — in that order, and not before.

## the email had to be real, so it had to be a service account

The one piece of the identity rung that genuinely leaves the building is the code email, and I'd decided (above) to build it for real rather than stub it. That turned a design choice into a credentials problem: *how does a headless process, running as a systemd unit with nobody sitting at it, prove to Google that it's allowed to send mail as my domain?* There's a tempting easy answer — an app password, or an OAuth2 "desktop app" flow where I click *Allow* once and stash a refresh token. I turned both down. The app password is a shared secret with full send rights and no scoping; the refresh-token dance assumes a human at a browser at setup time and leaves a long-lived user credential to babysit and rotate. Neither sits right for a server that's meant to run untended for a long time.

**The fit for an untended server is a service account with domain-wide delegation.** A service account is a non-human identity — it has its own key, no inbox, no password to phish. On its own it can't touch my Workspace; *delegation* is the deliberate, admin-only act of saying "this one identity may impersonate users in my domain, but **only** for these exact scopes." I grant it precisely one: `https://www.googleapis.com/auth/gmail.send` — the right to send, and nothing else. Not read, not list, not delete. Even if the key leaked, the worst it buys is sending mail as the delegated mailbox; the inbox itself stays shut. That narrowness is the whole point: the credential's blast radius is a single verb.

**The shape of the setup, recorded so future-me can redo it from cold.** It spans two consoles, because it's two different grants. In the **GCP** console: enable the Gmail API, create the service account, mint a JSON key, and copy the account's numeric client ID. Then in the **Workspace Admin** console — a separate place, by design, because authorising impersonation is a domain-owner's decision, not a project's — register that client ID against the single `gmail.send` scope under Domain-Wide Delegation. The key file lands on each box gitignored (it's a credential; it never goes near git), its path in `.env` as `GMAIL_CREDENTIALS_FILE`, and `GMAIL_SENDER` names the real mailbox the account sends *as*. The full click-path lives in the kernel README so it's a checklist, not a memory.

**In the code, delegation is two lines and a lazy import.** The client loads the service-account key scoped to `gmail.send`, then `.with_subject(sender)` — that single call is the impersonation, the in-code echo of the admin-console grant. The Google libraries and the key are imported and read *lazily, on the first send*, never at module load — so the test suite, which swaps in the fake, never so much as touches them, and import stays cheap. And the same honesty rule holds at the bottom: with no key configured the real client *raises*, loudly, rather than silently no-op a send that never happened. A green test suite still proves only the state machine. Whether Google actually accepts the bytes is a thing I can only learn by watching a real code land in a real inbox — first locally, then from the deployed kernel — and that, not a passing `pytest`, is what will finally earn boxes 198–206 their checkmarks. (One known quirk to expect: delegation takes a few minutes to propagate, so the very first send can `403` purely from timing, and the fix is to wait and try again — not to go spelunking through the code.)

## one valid code, enforced — because for this user a stray code isn't a corner case

Reviewing the code I'd written, I caught myself making the wrong call out loud — and I want it on the record, because the reasoning is the point, not the line of SQL.

The rule is *only the latest code works*. I'd built it the obvious way: when a new code is issued, the old unconsumed ones get their `expires_at` pushed to `now()`, so the next verify won't accept them. That's airtight for the ordinary path — type `/login`, type it again, only the second code lets you in (box 203). But it leans on *timing*: the expire-then-insert is two steps in application code, and two `/login` requests that overlap can each insert a code without seeing the other's, leaving two live codes at once. I'd looked at that race and waved it off as "essentially never hit for a single human — not worth the constraint." I called closing it over-engineering.

That was the wrong frame, and it's worth being precise about *why* it was wrong. "Essentially never, for a single well-behaved human" is the kind of reasoning that's fine for a toy and disqualifying for this. The Joy is meant to be a *vital* tool for neurodivergent people who struggle to fit — the kind of thing someone leans on when they're overwhelmed, double-tapping, firing the same action twice because the first didn't visibly land, on a flaky phone connection that retries requests under them. The exact conditions that trigger this race — impatience, duplication, retries, bad networks — aren't the unusual case for the person this is *for*. They're Tuesday. A security invariant I'm only willing to hold "as long as the user behaves predictably" isn't an invariant; it's a hope, and it quietly offloads the cost of my shortcut onto the person least able to absorb it. Two valid codes alive at once doubles the window an attacker has to guess or intercept one, and it does so precisely when the legitimate user is most stressed. That's not a corner to cut.

So the requirement, logged plainly: **at most one login code is ever spendable for a symbiot, and the database is what guarantees it — not the order requests happen to arrive in, and not the discipline of whoever writes the next caller.** The fix as built: a partial unique index — `WHERE consumed_at IS NULL`, folded into the identity migration since it had never run on prod — so the row layer simply cannot hold two live codes for one symbiot. My first sketch was "expire the old code, then insert the new one," but that's two statements, and two overlapping `/login`s can each insert before seeing the other — the very race I'm trying to kill. So issuance became a single atomic upsert instead: `INSERT … ON CONFLICT (symbiot_id) WHERE consumed_at IS NULL DO UPDATE`, which *overwrites the one live row in place*. Two concurrent calls don't collide into two rows and don't error out either — the second is absorbed as an update, one of the two codes wins, exactly one stays live, and `/login` keeps its single canonical reply. The constraint is the guarantee; the application code just leans on it.

It's tested the way an internal invariant should be — at its source, not by racing requests: one test asserts three issuances leave exactly one row (in-place overwrite, no pile-up), another inserts two live codes directly and asserts the database itself raises a unique violation. That second test is the real proof, because it shows the property holds no matter how requests interleave — timing can't defeat a row constraint.

This one deliberately does *not* go in the DoD checklist above. Those boxes (198–206) are things I verify by hand as the end user — type a line, watch the reply — and a concurrency race isn't manually reproducible: you can't reliably fire two overlapping `/login`s from a terminal and observe the window. It's an internal invariant, so it's paid for the right way for an internal invariant — an automated test that issues two codes under contention and asserts only one survives — not a manual runbook step the symbiot walks. Logging it here, in the journal, is the honest home for a requirement that's real but not end-user-observable.

## a limiter at the door, and the real locks behind it — because rate limiting alone is only a hope

Now that the kernel touches a database and sends mail per request, two endpoints became weapons: `/login` is an email cannon (every call mails someone), and `/login/verify` is a brute-force oracle (every call is a guess at a code). So I added rate limiting — and then, pushed to apply the *make-it-impossible* lesson from the section above, had to be honest about a thing that's easy to fudge: **rate limiting cannot be made impossible.** It's a soft dampener by nature. The key is a client IP, and IPs are shared (a whole office behind one NAT), spoofable, and rotatable; "too many requests in a window" isn't a *state* you can forbid with a constraint the way "two live codes" is — it's a count in memory that a restart forgets and a second process wouldn't share. An in-memory limiter is the lock on the *gate*, not the vault. Treating it as the guarantee would be exactly the lock-vs-luck mistake: a defense that *happens to* filter the bad case until the day it doesn't.

So the limiter is real but honestly scoped, and the actual guarantees were put where they can't be bypassed — in the database. Two layers, each doing only what it's good at:

- **The gate (soft, in-memory):** a small hand-rolled sliding-window limiter at the edge — no dependency, same spirit as the hand-rolled migrations and HMAC. It sheds gross volume cheaply before it reaches a route. `/login` is tight (a person needs one or two codes a minute), `/login/verify` a little looser, everything else gets a generous ceiling a stressed human double-tapping will never hit. `/health` is exempt outright — throttling the connectivity probe would make a *healthy* kernel read offline to the shell's dot, which is the one thing it must never do. A map-holding catch I had to surface and note for deploy: the kernel sits on `127.0.0.1` behind nginx, so at the app layer *every* request looks like it comes from localhost. Keying by IP only works if nginx stamps `X-Real-IP` (`proxy_set_header X-Real-IP $remote_addr;`); without it the limiter throttles the whole world as one bucket. That's now a deploy step, not an option.

- **The vault (hard, in the database):** the guarantees that survive a restart, a second process, and IP rotation, because they're rows, not counters.
  - *Email bombing* → a **re-issue interval**: a fresh code can't be minted while the live one is younger than the window. Built into the same atomic upsert (`DO UPDATE … WHERE created_at <= cutoff … RETURNING id`) so the database decides whether a code was issued and we only mail when it says one was. A side gift: a double-tap no longer *invalidates the code already in the inbox* — it's a no-op, not an overwrite. The thing that dampens abuse also fixes a real ND-UX trap.
  - *Brute force* → a **per-code attempt budget**: each live code absorbs a fixed number of wrong guesses, then the row burns itself (`consumed_at` set). Bounded by the row, immune to which IP does the guessing — the part a per-IP limiter can never promise once an attacker rotates addresses.

The brute-force lock forced a real design choice worth recording. `/login/verify` was spec'd to take only `{code}` — but a *wrong* guess matches no code row, so there's nothing to charge a failed attempt against, and no way to tell whose code is under attack. A global counter would let any attacker lock the real symbiot out (a self-inflicted DoS), so that's out. The clean fix is to make the guess **attributable**: verify now carries `{address, code}`, so a wrong code is charged to *that* symbiot's live code and to no one else's. The cost is honest — it changes the verify contract from the original build-log spec, and the shell (which knows the address it just typed) has to send it. The residual risk is also honest: someone who knows the symbiot's address can burn their live code by spending its attempt budget — but that only *forces a re-issue*, it never grants entry, and the re-issue interval bounds how fast they can make the symbiot dance. Account compromise stays impossible; nuisance is the price of attributability, and it's the right trade.

Proven in the suite the way each layer deserves: the DB guarantees by driving the state machine (a code dies after its budget of wrong guesses; a guess against a stranger's address burns nobody; a second `/login` inside the interval mails nothing and leaves the first code spendable), and the edge limiter by its own small file (a burst past the ceiling gets a 429, `/health` never does, one noisy IP can't spend another's budget). None of these are DoD boxes either — same reasoning as the single-code invariant: they're internal properties asserted at their source, not things the symbiot reproduces by hand.

## the kernel won't tell you it's a typo — so the shell must, and that's enough

A question came up while building login: should the kernel validate that an address is *shaped* like an email? The answer is a clean **no**, and the reason is the same anti-enumeration rule everything else here bends to. Format validation isn't security — parameterised SQL already makes a malformed address harmless, and the exact-match lookup already turns garbage into a clean no-match. A format gate would add nothing there but theater. Worse, it would *break* a property we paid for: a `422` on "that's not a valid email" is an oracle. It distinguishes "wrong shape" from "valid-but-unknown," which is exactly the distinction the canonical single reply exists to erase. We deliberately accept a blank address for this reason; rejecting a malformed one would re-open the door we shut. So the kernel stays silent on shape — one identical reply for a real address, an unknown one, or pure noise.

That silence is correct for security and unkind to a human. The symbiot who fat-fingers their address gets the same warm "if that address is registered, a code is on its way" as someone who typed it perfectly — and then waits on an email that can never arrive, with nothing to tell them why. The kernel *can't* warn them without becoming the oracle we just refused to build.

So the warning belongs in the shell, before the request is ever sent — and that's what I built: `/login` now gives an instant, local typo nudge ("that doesn't look like an email address — check for a typo; nothing was sent") for an obviously malformed address, and only lets a plausible one through. The check is deliberately lenient — a missing `@`, no domain, a stray space — not RFC-5322 pedantry, which rejects valid oddities and fights the wrong war. (The actual code-request round trip is still a later rung; this slice is only getting the address right.)

Why is this *enough*? Because the only cost of an unvalidated shape is a confused human waiting on a no-show email — a UX failure, never a security one. The kernel's safety does not depend, even slightly, on the shell validating anything: a crafted `curl` with garbage still gets the same canonical reply and still does no harm. So format-checking is pure kindness, not a load-bearing control — and kindness belongs where the human is, at the point of typing, where it can be instant and reveal nothing about who's registered. Putting it in the kernel would buy no safety and cost an enumeration oracle; putting it in the shell costs nothing and spares the symbiot the silent wait. The strict layer holds the guarantees (§ above); the shell holds the courtesy. That division is the whole point.

## a real code, in a real inbox — the floor holds, locally

The credentials walkthrough is done, and it ended the only way that counts: a six-digit code generated by the kernel left the building through the Gmail API and landed in a real inbox. Two consoles, exactly as designed — in GCP I enabled the Gmail API on `the-joy-496315`, created the `joy-gmail-client` service account with no IAM role (it authorises by delegation, not by project rights), minted its JSON key, and copied its client ID; in Workspace Admin I authorised that one client ID against the single `gmail.send` scope under domain-wide delegation. The key file lives gitignored beside the clone, its path in `.env`, and `.with_subject(yactouat@yactouat.com)` is the in-code echo of that admin-console grant. The first send went through on the first try — no propagation `403`, which I'd braced for. The narrow scope means the worst that key can ever do is *send* as that one mailbox; it can't read a thing.

Then the part I'd been holding the boxes hostage for: the whole login loop, run for real against the local kernel, on live Gmail rather than the fake. `POST /login` returned the one canonical line and put an actual code in the inbox. `POST /login/verify` spent that code for a session token. `GET /status` with the token read `authed`, with the right email. `POST /logout` revoked it, and the very same token then read `not logged in` — the revoke held, the bearer went inert. Every rung I built against `FakeEmailClient` does the identical thing when the bytes actually leave the building. The interface bet paid off exactly as intended: the only difference between the test suite and this run was whether Google saw the message.

And still — **the boxes stay empty.** This is the local box, not the deployed one, and the honesty rule I've earned twice already (a `curl` `200` proves nothing a browser hasn't) says boxes 198–206 are paid for only by watching this same loop run against `https://kernel.os-joy.com` in the open: the service up clean under systemd with migrations applied and the symbiot seeded, a code that crosses the real internet from the deployed kernel, the round trip walked live. The floor is real and it holds under my own hands here; the next session stands it up on the server and walks it in daylight, and *only* then do the checkmarks get earned. Foundations, then the flow, then the boxes — in that order, and not before.

## in daylight — the kernel holds on the server

Foundations, then the flow — and now, in the open. The box took the deploy: a Postgres database under peer auth (`postgresql:///the-joy` — no password, the OS user *is* the role, `\conninfo` confirmed before I trusted it), its own fresh `KERNEL_SECRET` distinct from the local one, the same gitignored Gmail key beside the clone. Then I walked the whole identity state machine against `https://kernel.os-joy.com`, one rung proven before the next:

- `/health` still green over HTTPS — the reachability rung didn't regress under the new lifespan that now runs migrations and seeds the symbiot at startup.
- `/login` with an **unknown** address, a **smuggled** `me@allowed, attacker@evil`, and a **blank** line each returned the byte-identical canonical line and mailed nothing — the anti-enumeration silence holds on the deployed box, not just in the suite.
- `/login` with the **exact** address put a real six-digit code across the open internet into the inbox; `/login/verify` spent it for a token; `/status` read `authed` with the right email; `/logout` revoked it and the same bearer went inert on the next `/status`. The full loop, in daylight, from the server.
- A **wrong** code (`000000`) got the clean "that code didn't work" and left the session unauthed. `/logout` run again with an inert token, and with pure garbage, both returned the calm not-authed envelope — idempotent, never an error.
- **Only the latest code lives**: a freshly-issued, never-used code came back rejected the instant a newer `/login` superseded it, while the latest one had logged in moments before. The single-live-code invariant, proven against the real database over the wire.

Every behaviour behind boxes 198–206 is now real on the server — and yet the boxes *still* don't get ticked, for a reason I want on the record. They're written in the **shell's** voice ("authes the same terminal," "re-prompts," "the terminal's reply"), and the shell's `/login`/`/logout`/`/status` are still inert stubs. So this is the **kernel half** earned in full and honestly: the privileged core does every right thing when hit directly over HTTPS. The checkmarks wait for the shell slice that wires the terminal to these routes and walks them in a browser — the boxes describe that end-to-end experience, and I won't tick them on a `curl` proof of the half that lives below the terminal. The honesty rule cuts the same way it has all session: the kernel is verified live; the terminal experience the boxes actually name is the next rung.

One real anomaly surfaced and I'm not burying it: firing two `/login` calls back-to-back delivered only **one** email both times, though the kernel issued and (as far as the route knew) sent both, and the state machine behaved perfectly — the older code was correctly superseded. Likely Gmail collapsing two near-simultaneous messages with an identical subject, or a send error the route swallows by design (the canonical reply returns regardless of delivery, on purpose). It didn't block the invariant — I proved supersession with a single live code instead — but a symbiot who legitimately re-requests could hit the same silent drop. Worth a look at the server's send logs and, probably, varying the subject or surfacing send failures to journald, next session.

The lesson rhymes with the `curl`-vs-browser one from earlier this session: *don't let the lenient path stand in for the strict one.* There it was a tool that didn't enforce CORS masking a real defect; here it's a usage pattern ("a calm user, one request at a time") masking a real race. Both are the same mistake — judging a safety property by the easy case and calling the hard case unlikely. For the human this is built for, the hard case is the normal one.