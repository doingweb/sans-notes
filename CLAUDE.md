# CLAUDE.md

Guidance for Claude Code when working in this repo.

## What this is

**Tuck Notes**: a minimalist, end-to-end encrypted, local-only Markdown notes
app. Web-only, an installable PWA, no backend. Read these before changing
anything substantial:

- `README.md` — the tenets, the threat model, the architecture.
- `docs/format.md` — **the contract.** Every byte the app stores or exports.
- `docs/plan.md` — milestones to the MVP. Work happens in that order.
- `docs/ideas.md` — the backlog. Ideas that come up mid-task go there, not into
  the code.

The tenets decide every trade-off: **simple, verifiable, yours.** Less code
beats clever code, because every line is one someone has to audit. When a
feature and the tenets disagree, the tenets win; raise it rather than quietly
compromising.

**A note's body is free-form.** Its title is a separate, optional field, and
nothing may parse, require, or assign meaning to the body's structure — not its
first line, not Markdown headings, not inline syntax. An untitled note is named
by when it was created, never by its contents. Nothing taken from a body is
stored outside the note itself, and **a body is decrypted only when the user
opens that note** (or exports or imports the vault) — never to build the list.

## Security rules

These are not style preferences. Breaking one is a bug even if the tests pass.

- **No runtime dependencies.** The shipped app uses browser APIs only. Dev
  dependencies are limited to `typescript`, `@types/bun`, and Biome; ask before
  adding any other, and justify it against "write it ourselves."
- **Crypto is Web Crypto only.** `crypto.subtle`, `crypto.getRandomValues`,
  `crypto.randomUUID`. Never `Math.random`, never a hand-rolled primitive, never
  a crypto library.
- **IVs are generated inside `seal` and nowhere else.** Callers cannot pass
  one. Reusing an IV under the same key breaks AES-GCM completely. The only
  exception is the known-answer test vectors.
- **Nothing readable is stored unencrypted.** No note content, titles, counts,
  timestamps, or settings in IndexedDB outside sealed blobs, and nothing at all
  in `localStorage`, `sessionStorage`, cookies, or the Cache API. The keyring is
  the only plaintext record, and it holds no secrets.
- **The master key is non-extractable once unwrapped**, is held only in memory,
  and is never logged, serialized, or stored. Locking drops every reference to
  it and to decrypted notes.
- **Never build HTML from strings.** No `innerHTML`, `outerHTML`,
  `insertAdjacentHTML`, or `document.write`. Use `textContent` and DOM APIs.
  Markdown is never rendered to HTML in the MVP. Script injection in this app
  means every note is readable.
- **The app sends nothing.** Pages are served with `connect-src 'none'`. Never
  add analytics, error reporting, remote fonts, or any third-party origin.
- **The passkey relying party ID is `tuck.apps.theantes.family`.** Changing the
  domain orphans every passkey and every browser's stored notes. Don't.

## Changing the format

`docs/format.md` is a promise to anyone holding an export, including people
decrypting it without our code. So:

- Any change to stored or exported bytes **bumps the format version**, updates
  `docs/format.md` in the same commit, adds test vectors, and **keeps reading
  every older version**.
- `tuck-decrypt.mjs` and `docs/format.md` move in lockstep. If they disagree,
  the spec is right and the code is the bug — or the spec needs fixing first.
- Never change associated-data strings, KDF parameters, or padding rules
  without a version bump.
- Keep the product's name out of every label inside the format, so a rename
  only touches the domain and user-facing text.

## Toolchain

**Bun** for running, bundling, and testing. Not Node, not npm, not Deno. Node
is used only to run the built `tuck-decrypt.mjs` in its end-to-end test,
because that is what users will run it with.

```bash
bun install          # dev tooling
bun run dev          # build + local server
bun test             # unit tests
bun run check        # tsc --noEmit, Biome lint + format check
bun run build        # → dist/
```

Bun does not typecheck; `bun run check` does. Run it, and `bun test`, before
finishing any task.

The bundle is **not minified** and ships linked source maps, so the deployed
code reads like this repo. Don't turn minification on.

## Code conventions

- Follow `~/.claude/skills/writing-code`. Notably: named `function`
  declarations for helpers, enums for closed sets, `@module`-tagged file-level
  JSDoc, `@param` on non-trivial functions, **`is`-prefixed booleans**, and
  throwing on exceptional conditions rather than widening return types.
- **Co-locate by feature** under `src/` (`crypto/`, `vault/`, `passkey/`,
  `export/`, `app/`, `pwa/`), with tests beside the code as `*.test.ts`.
- **Keep logic pure and plumbing thin.** Anything with a right answer — crypto,
  padding, display titles, merging, the passphrase policy, file naming — is a
  pure function tested under Bun. Browser-only code (IndexedDB, WebAuthn, the
  DOM, the service worker) is a thin layer over it. Storage sits behind an interface
  so the vault is testable with an in-memory implementation.
- Write tests first (`test-driven-development` skill). Crypto changes need
  known-answer vectors and tamper tests, not just round trips.
- Plain DOM, no framework. Small modules over big ones.

## Planning

Each milestone in `docs/plan.md` gets a detailed implementation plan written
with the `writing-plans` skill into `docs/superpowers/plans/` before code is
written. Update `docs/plan.md` when a milestone's scope or findings change.

## Testing in a browser

- **The service worker serves stale code while you work.** Once registered it
  is cache-first, so a rebuild can look like a change that didn't work.
  Unregister it and clear site data before believing what a browser shows.
- **Passkeys are bound to the domain they were made on.** Test passkeys on
  `localhost` or `antes-tuck-notes.web.app`, never the production domain.
- Check storage in dev tools after any change that writes: nothing readable
  before unlock.

## Deploy

Firebase Hosting, project `antes-tuck-notes`, served at
<https://tuck.apps.theantes.family/>. GitHub Actions checks, builds, and deploys
on push to `main`, in the same shape as `~/Code/songbook`'s workflow with Bun in
place of Deno. Security headers, including the CSP, live in `firebase.json`;
treat changes there as security changes.
