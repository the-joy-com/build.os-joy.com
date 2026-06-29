# session 15 of building The Joy in the open

## first, get the vision out of my head and into a tracker

I'm starting this session not in code but in [Linear](https://linear.app/): a project for The Joy. The reason is honest and a little embarrassing — holding the whole vision in my own head has started to cost more than it gives. `ROADMAP.md` and [`TODOS.md`](../TODOS.md) were supposed to be the place the big map lives so I don't have to carry it, but they've become the opposite: two long flat lists I re-read top to bottom every time I want to find the one next thing, and every read drags the entire backlog back into working memory. The agentic OS, the sovereignty stack, the integrations, the anti-pacification engine, the bootstrap TODOs — all of it surfaces at once, and the clutter is exactly the kind of digital noise The Joy itself is meant to filter. Building the thing badly in the very way the thing exists to fix.

So before any more kernel or shell work, I'm moving the vision into Linear — somewhere I can park the whole map *outside* my head and pull up only the slice I'm working on, leaving the rest collapsed and quiet until I ask for it. The markdown files stay as the open-source record of the thinking; Linear becomes the working surface that decides what's in front of me right now.

That's today's first move: zoom all the way out, get the map onto a wall I can step back from, then zoom back in to the one slice that's actually next.

## the loop: one item at a time, draft → shoot → drain

The way I worked the move matters as much as the move, because doing it badly would have re-created the exact pile I was escaping. The rule was: **one roadmap bullet at a time.** For each line I'd get a keep-or-drop read, a proposed home (which of the six projects below) and a proposed priority with a one-line justification — then *stop* and wait for an explicit "shoot" before anything was written to Linear. Only on approval did the issue get created; and the instant it did, the source line in `ROADMAP.md` was **deleted**. Items already done, or that I decided to drop, had their line drained the same way with no ticket behind them.

The draining is the part I'd defend hardest. If the roadmap and Linear both held the same item, I'd be back to two maps to reconcile — so every bullet ends up in exactly one place: live in Linear, or gone. By the end of the pass `ROADMAP.md` held nothing but its empty section headers, so I deleted the file outright — the wall is in Linear now, and a hollow markdown husk pointing nowhere is just more of the noise this whole move existed to clear. (No Linear write — create, edit, re-parent, re-prioritise — ever happened without an explicit go for that specific change. A *question* about why I'd placed something somewhere is not permission to move it.)

## the wall has six rooms, sorted by *kind of work*

Before filing anything I had to decide what the columns even are, and the trap was obvious once I named it: if you bucket by the *resource a task consumes*, every networked task lands in "drivers" and every always-on task lands in "kernel," and both become meaningless catch-alls. So the projects are sorted by the **kind of work**, not the resource:

- **kernel** — the always-on cognitive core; self-monitoring runtime; the resilience layer (durability, local fallback, availability).
- **shell** — the human interface: UI/UX, the input layer.
- **drivers** — connectors to the outside world (messaging, platform/AI APIs, devices). Connector-*building* only; capabilities built *on* a connector live elsewhere.
- **promotion** — go-to-market for The Joy itself.
- **studio** — the creative application layer: the symbiot's own content and creative output (the book lives here).
- **cyberspace ICE** — *Intrusion Countermeasures Electronics*: the application layer that **defends the human symbiot**, on two fronts. Digital (identity, reputation, footprint) and psychological (psy-ops countermeasures against pacification and manipulation — inward, my own spirals; outward, gaslighting and history-rewriting by others). The anti-pacification cognitive engine lives here, not in kernel.

`studio` and `cyberspace ICE` are the two halves of an application layer sitting *above* kernel and drivers — creative output vs. defending my digital existence. New work that's an *application for my life* belongs in one of those rather than being forced down into the core.

## what came off the UI/UX shelf

The `### UI/UX` ideas drained into **shell**: inline autocompletion in input prompts (I nearly dropped this as already-done, then caught that the *prompt-level* autocompletion wasn't built and filed it); copy-paste / drag-and-drop into the input; a kill / reboot control for the running shell; and multimodality, filed as an **umbrella** issue with a child per modality. That last one is where a pattern I leaned on all session first showed up — split a fat capability into an umbrella plus pluggable children, and keep the *capability* (what the shell can accept) separate from the *driver* (the specific connector that carries each modality).

## the anti-pacification engine: one spec block, twenty-odd rungs

The whole `### anti-pacification cognitive engine` spec — about two dozen bullets — became a single cluster under one parent issue in cyberspace ICE, cross-wired so the substrate pieces (memory, the CRM, highlights) visibly feed the higher-order ones (reality-checking, relationship-tending, the compass, tactical "what next"). A few calls inside it are worth keeping on the record:

- **The reactive/proactive split.** "What should I do next?" on demand is a different rung from the system *unprompted* surfacing the next best thing with full context in mind. Same distinction I'd honoured in the shell; same distinction here — being asked is not the same capability as choosing to speak.
- **The book belongs in studio, not here.** The "gather material for the book" bullet sat in the engine spec, but gathering creative material is *creative output*, not psychological defense — so it went to **studio**, related to the engine rather than nested under it. It's a High, because writing the book actually matters to me, not a someday-maybe.
- **Priorities skew High on purpose.** Most of the engine children are High — the crisis anchor, daily routines, time-tracking, the kakebō for day-to-day visibility on my pecuniary power, the reward-hacking guard that's a lot of what keeps me going. I deliberately set *zero* Urgents inline.

## the one thing I deliberately deferred: the urgency pass

That last point is its own decision. Urgency only means something *comparatively* — assigning "Urgent" ticket-by-ticket during intake just inflates the label until it's noise. So I held every Urgent call for a single dedicated pass *after* the roadmap is fully drained, judging the whole High-priority field at once and promoting only the genuinely highest-stakes few. The roadmap is drained now, so that pass — plus a small title-casing retrofit on the earliest tickets, and a decision on whether `TODOS.md` gets the same treatment — is what's queued next. Nothing gets promoted without the same draft-then-shoot approval every other write got.

Two principles held the whole way through and are worth restating because they're load-bearing, not decoration: the principal is **the human symbiot**, never "the user"; and the **input layer never gates submission on identity** — auth decides what the *reply* is, never the right to speak. Both showed up as constraints on individual tickets, and both are really the same stance the shell already takes, carried into the backlog.
