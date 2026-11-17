# The road to the MVP

The MVP is the app the maintainer can trust with real notes: a local-only,
installable PWA that encrypts everything, unlocks with a passkey, autosaves,
locks itself, and exports to a file that can be decrypted without it.

Each milestone ends with something working and deployed. Before starting one,
write its detailed implementation plan with the `writing-plans` skill (into
`docs/superpowers/plans/`), test-first.

## MVP scope

**In:** create and edit notes · a Trash (move to Trash, restore, delete
permanently, empty Trash) · raw Markdown editor with continuous autosave · an
optional title, separate from the body · note list sorted by last edit, showing
each note's display title · passkey unlock with passphrase fallback · auto-lock
when idle (a setting, default 5 minutes) and when hidden · export · import
(restore and same-vault merge) · `sans-decrypt.mjs` · installable, offline,
asks before updating.

**Out:** everything in [`ideas.md`](ideas.md).

## M0 — Spike: prove the platform

Throwaway code, deployed to `https://sans-notes.web.app` — deliberately
**not** the production domain, so spike passkeys never bind to it.

- Create a passkey with PRF and read the PRF output on each target: iPhone
  Safari (in the browser and installed to the home screen), macOS Safari and
  Chrome, Android Chrome.
- Confirm that the same synced passkey gives **the same PRF output** on two
  devices. The whole design leans on this.
- Confirm when `visibilitychange` fires: tab switch, app switch, screen off,
  desktop screen lock.
- Record what `navigator.storage.persist()` returns on each, installed and not.

**Done when:** findings are written into `docs/format.md` and this plan, and
anything that changes the design has been decided. If PRF is unavailable on a
must-have target, stop and rethink unlock before building more.

## M1 — Toolchain and deploy

- `.gitignore`, `.editorconfig`.
- Bun, pinned. `tsconfig.json` in strict mode. Dev dependencies only:
  `typescript`, `@types/bun`, and Biome for formatting and linting.
- `scripts/build.ts` using `Bun.build`: the app bundle, the service worker, and
  `sans-decrypt.mjs` (targeting Node), unminified, with linked source maps.
  Package scripts: `dev`, `build`, `check`, `test`.
- Firebase project `sans-notes`, `firebase.json` with the security
  headers (strict CSP with `connect-src 'none'` for pages, `'self'` for
  `/sw.js`; `frame-ancestors 'none'`; `no-referrer`; `nosniff`; `no-cache` on
  `/sw.js`), and the GitHub Actions deploy workflow modelled on songbook's.
- Custom domain `sans.apps.theantes.family` (see the `setup-firebase-app`
  skill).

**Done when:** a placeholder page is live on the domain, CI checks and deploys
on push, and the browser reports the CSP as enforced.

## M2 — Crypto core

Pure functions over Web Crypto, fully unit-tested under `bun test`.

- `seal` / `open` for sealed blobs, with associated data and padding. Callers
  never supply IVs.
- Master key generation; wrap and unwrap with a KEK.
- PBKDF2 and HKDF key derivation.
- The keyring: create, serialize, unwrap by passphrase.
- The passphrase policy.
- Tests: known-answer vectors (fixed key, IV, and plaintext → fixed bytes),
  committed to the repo as the format's reference vectors; every tampered byte
  fails to open; associated-data mismatch fails; padding sizes.

**Done when:** a vault's worth of blobs can be created and reopened in tests
from nothing but a passphrase.

## M3 — The vault

- IndexedDB storage behind a small interface, so vault logic is testable in
  Bun with an in-memory implementation.
- Notes and the index, written together in one transaction; the Trash
  lifecycle (trash, restore, delete permanently, empty), where permanent
  deletion leaves no trace; rebuilding a lost index from the notes.
- The display title (`docs/format.md`) as a pure function shared with
  `sans-decrypt.mjs`.
- Same-vault merge (newer `updated` wins; notes only in the export are added)
  as a pure function.
- `navigator.storage.persist()` on setup.

**Done when:** notes can be created, listed, edited, trashed, restored, and
permanently deleted through the vault API in tests and in a bare browser
harness.

## M4 — Passkey unlock

- Create a PRF-capable passkey during setup; add its wrapped key to the
  keyring.
- Unlock: passkey first, passphrase as fallback, with clear feature detection
  when PRF is unavailable.
- Relying party ID fixed to `sans.apps.theantes.family`.

**Done when:** on the maintainer's phone and laptop, the vault unlocks with the
screen lock.

## M5 — The app

- Setup flow: *new vault* (passphrase with live policy feedback, then passkey)
  or *restore from an export*.
- Unlock screen.
- Note list: each entry shows the display title — an untitled note appears as
  `Note from <created>`, so listing never decrypts a note.
- Editor, in the spirit of Google Keep: a single-line title field, with a
  placeholder rather than a required value, above a plain `<textarea>` for the
  body.
- Continuous autosave: debounced while typing, and immediately on blur,
  `visibilitychange`, and `pagehide`.
- Move to Trash from the editor and the list, with no confirmation since it is
  undoable. A Trash view listing trashed notes, with Restore, Delete
  Permanently, and Empty Trash; only the permanent actions ask for
  confirmation.
- Minimal, elegant styling.

**Done when:** the maintainer can use it for a day without reaching for a
missing feature that's in scope.

## M6 — Locking

- Lock on idle and when hidden. Save first, then drop the master key and every
  decrypted note, and clear the editor.
- A setting for the idle timeout, default 5 minutes.

**Done when:** locking is impossible to miss and never loses a keystroke.

## M7 — Export, import, and the decrypt script

- Export `sans-export.json`.
- Import: restore into an empty browser, merge into the same vault, refuse a
  different vault. Unlock the export with its passkey if this device has it,
  or the passphrase. Verify every blob before writing anything.
- `sans-decrypt.mjs`: passphrase from stdin, one Markdown file per note, safe
  file names. Served from the site and linked from the export screen.
- An end-to-end test: export from the vault in Bun, run the built script under
  Node, compare the files.

**Done when:** a phone's notes move to the laptop by export and import, and the
same export decrypts on the command line.

## M8 — PWA

- Web app manifest and icons; installable on iOS and Android.
- Service worker: precache the build, serve it offline, cache named by build
  digest (the songbook pattern).
- Commit SHA stamped into `sw.js`.
- Update prompt when a new worker is waiting, showing the running and new
  commits; reload on accept.

**Done when:** installed, the app works in airplane mode, and a deploy shows
the update prompt with both commits.

## M9 — Hardening and dogfooding

- A security review of the whole codebase (`security-review` skill), against
  the threat model in the README.
- Re-read `docs/format.md` against the code; regenerate test vectors.
- Check the CSP, headers, and storage contents in each browser's dev tools:
  nothing readable before unlock.
- A week of real use; fix what hurts.

**Done when:** the maintainer moves real notes in.
