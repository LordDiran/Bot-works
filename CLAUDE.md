# Bot Crossing

A 3D colony sim where every coding-agent thread on your machine becomes a bot that walks out of a ship, claims a plot for its repo and builds something. Vite, three.js, plain JavaScript, Node 22.13 or newer. Reads each harness's own session files locally; nothing is uploaded and there is no account.

## Read this first: this is not originally Timmy's codebase

The package is `bot-crossing` by Jarren Rocks, upstream at `jarrenrocks/bot-crossing`, live at botcrossing.com. This repo is `LordDiran/Bot-works`, a working copy.

Unlike most imported code, forking here is explicitly welcomed. `CONTRIBUTING.md` says so directly: it is MIT, the author cannot promise to maintain it, and he would rather see a healthy fork than a stale PR.

Two things that permission does not cover, both in `TRADEMARKS.md`:

- **The name "Bot Crossing" is reserved.** MIT is a copyright licence and grants no trademark rights. A fork may not ship under that name or a name likely to be confused with it. Descriptive reference is fine: "a fork of Bot Crossing".
- **Crew character design created after 16 September 2026 is reserved** and is not under MIT. The code that renders the character stays MIT. Design published before that date stays under the licence it shipped under.

So if this copy becomes something Timmy publishes, it needs its own name and its own character design. Confirm with him what this copy is for before significant work.

`README.md`, `CONTRIBUTING.md` and `DECISIONS.md` are all long, current and genuinely good. `DECISIONS.md` records what is settled and why, and is worth a skim before proposing anything. This file carries only what you get wrong without it.

## Commands

All verified on a clean `npm ci` as of 2026-11-13.

```
npm ci            install, exit 0
npm test          node --test, 204 tests, all pass
npm run build     assets then vite build, exit 0
npm run dev       vite with the API inside it, one process, port 5274
npm start         build then serve
npm run serve     serve an existing dist/
npm run assets    re-pack source art into .glb
```

There is no linter and no typecheck. Do not invent `npm run lint`. Keeping the existing tests green is the bar; the suite is not exhaustive and new tests are welcome rather than demanded.

`npm run assets` is a **no-op on a fresh clone**. The built `.glb` files are committed and the raw art packs are not. You only need it if you are changing the art pipeline.

## Definition of done

`npm test` exits 0 and `npm run build` exits 0.

Anything interesting on screen needs a real harness installed with real threads. There is nothing to see from a bare clone.

## Known and accepted

The build warns that a chunk is over 500 kB. It is 1,131 kB, almost all three.js. Pre-existing, not your regression. Do not open an unrequested code-splitting effort because you saw it.

## Two hard rules, both of which have already cost someone's machine something

From `CONTRIBUTING.md`, and the author says he will not bend on either.

**Nothing is ever written to a harness.** Not a transcript, not a session record, not one flag. `data/colony.json` is the only file this project writes, anywhere. It used to write `isArchived` onto Claude Code's own session record and that was removed deliberately: the desktop app serves from its in-memory copy, so the write looked broken and holding it together needed a re-assert on every scan plus a `ps` sweep. Archiving is the colony's own bookkeeping now. `setArchived` is not part of the adapter interface and adding one back is a bug.

**Nothing is read from or executed inside another application's bundle.** Only files under the user's own home directory. An adapter that fell back to running a binary inside `/Applications` set off a Gatekeeper alert and moved two unrelated apps to the Trash on the author's machine. Opening a thread goes through a URL the OS resolves, or a command already on `PATH`, never a path guessed inside an app bundle.

## The harness seam

`server/harnesses/` holds one adapter per harness with a registry in `index.mjs`. Nine supported: claude-code, codex, opencode, antigravity, cursor, hermes and others. Adding one should be one new file and one registry line. `server/harnesses/README.md` is the interface contract, read it first.

Three things the adapters settle, all in `DECISIONS.md`:

- `openThread(ref)` and `newSession(dir)` return `{ ok, url, command }`. The server decides what to do with it per platform. No harness knowledge reaches `launch()`, and that seam is why the directory is swappable at all.
- `sizeBytes` is bytes, not tokens. Not every harness reports tokens and CLI sessions often report none. It feeds a shared log scale across the whole map, so mixing units would make one harness's buildings taller for the same work.
- Thread ids are prefixed, `claude-code:<uuid>`. `colony.json` v1 files are migrated on read.
- Optional `diagnostic()` returns a sentence or `''`. Without it a harness that cannot read its own store reports `detected: true`, throws on every poll, and looks healthy in the HUD while contributing nothing.

## The Host and Origin checks are the security model, not friction

The server reads your transcripts and can ask the OS to open things, so four things hold it in: it binds `127.0.0.1`, it checks `Host` (DNS rebinding reaches loopback as a same-origin page), it checks `Origin` (a cross-site `fetch` with a `text/plain` body is not preflighted), and the built page ships a CSP.

A bare `curl` POST is refused by design, because browsers always send `Origin` and its absence means the caller is not the page. Script against the API with `-H 'Origin: http://localhost:5274'`. Do not relax a check to make a request work. `npm run dev` sends no CSP because Vite injects styles for hot reload; that is deliberate.

`BOT_CROSSING_HOST` and `BOT_CROSSING_ALLOWED_HOSTS` widen it on purpose, documented in the README.

## Two rendering traps

**Every hand-written shader calls `withCurve`.** The world curve is patched into three's own `project_vertex`, so built-in materials bend without knowing about it. A material that installs its own `onBeforeCompile` replaces that and has to call `withCurve(shader)` itself, or its uniforms stay zero, it stays flat, and it floats above ground that bent away under it. Same for a custom depth material, or its shadow stays flat.

**No sound samples are committed.** `public/audio/` is gitignored apart from its README, because sample libraries license sounds for use in a work rather than for redistribution, and a public MIT repo is redistribution. The registry in `src/audio/sounds.js` refuses to load if any name lacks a procedural generator, so the game is fully audible from a fresh clone. Real recordings are a local override by name through `manifest.json`. CC0 sounds may be committed with a line in `CREDITS.md`.

## Pull requests upstream are treated as feature requests

If work here is going back to `jarrenrocks/bot-crossing`, know the model before investing in a branch. The author batches open PRs, tests them on his own machine and lands the intent of them together, crediting the contributor by name. Your PR may be closed unmerged and still be the reason something shipped. Five PRs once widened the same interface five incompatible ways, which is the reason.

What makes a PR useful there: one thing at a time, and what you verified and on what machine. He cannot test Windows, cannot test Linux, and cannot test a harness he does not have. "Ran it against 40 real Codex sessions on Fedora" is worth more than a clean diff.

## House style of the upstream code

No semicolons, single quotes, 2-space indent, around 110 columns. Comments explain **why**, particularly why an obvious approach was rejected. Match it where you touch things.

The runtime has two dependencies, `three` and `@mdi/js`, and the author wants it to stay small. Do not add one casually.

## This repo ships its own Claude Code skill

`.claude/skills/agent-session-world/` with references on the asset pipeline, making it feel alive, rendering traps and harness adapters. That is the authority on building this kind of world. Read it before doing rendering or scene work rather than working it out from the source.

## Do not

- Do not assume this is Timmy's code to rename, rebrand or publish as his own.
- Do not ship a fork under the name "Bot Crossing".
- Do not write to a harness. `data/colony.json` is the only file this project writes.
- Do not read from or execute anything inside another application's bundle.
- Do not relax the Host, Origin or CSP checks to make a request work.
- Do not commit a sound sample, or add art that is not CC0.
- Do not add a dependency, a linter config or house conventions unasked.
- Do not treat the chunk-size warning as a regression.
