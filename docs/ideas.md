# Ideas

Things worth building, not yet built. Nothing here is committed to — it's a
place to put an idea so it stops taking up room in someone's head.

Each entry says what it is, why it's wanted, and honestly what it depends on,
because the dependency is usually the reason it isn't done yet. Every idea is
measured against the tenets in the README: it has to earn the code it adds.

## Sync

Notes written on the phone appear on the laptop without an export and import.

The format is already shaped for it: every device shares the vault's master key,
blobs never need re-encrypting to move, and notes carry `updated` times. A
server would hold only the keyring and opaque blobs, and the synced passkey
means a new device can unlock as soon as the blobs arrive.

**Depends on:** a server, which this app deliberately doesn't have yet — and
with it, an account or vault-addressing scheme, a relaxed `connect-src`, a
conflict story better than "newer wins" (see [Version history](#version-history)),
and a way for a permanent deletion to reach other devices. The vault keeps no
record of deleted notes, by design; with a server, the deletion can happen
there, but a device that was offline at the time would still need telling
without a lasting trace being kept.
There is no browser API that syncs web storage through the browser's own
account; `chrome.storage.sync` is for extensions only.

**Smaller step:** on desktop Chromium, the File System Access API can write the
vault into a folder the user picks — a Dropbox or iCloud Drive folder gives
sync for free, on desktop only.

## Version history

Keep earlier versions of a note, so an autosave, a bad import, or a sync
conflict can never destroy text.

Autosave makes this more valuable than in most editors: there is no "I didn't
save" to fall back on. It also fixes the weakest rule in the format — "the newer
copy wins" on import — because with history, the losing copy is kept instead of
discarded.

**Depends on:** deciding what a version is (every save is far too many with
continuous autosave; a version per editing session is likely right), and how
they are stored without revealing edit frequency to someone watching storage
sizes.

## Advanced search

Search note bodies, not just titles — with stemming, so `running` finds `run`.

**Depends on:** a search index, which is a second copy of every note's words and
must be encrypted like everything else. Decrypting all notes on demand is a
simpler first step that would work for hundreds of notes; a stemmed, encrypted
inverted index is the version that scales.

## Tags

Filter the list by tag.

**Depends on:** a tag field and the UI to edit and filter by it. Tags belong
beside the title, as their own property — not inline `#tags` parsed out of the
body, which would give the body a structure it is promised not to have. They
would be cached in the index alongside titles.

## Attachments

Images and files inside notes.

**Depends on:** encrypted blob storage for binaries, size limits, and a story
for showing an image without handing decrypted bytes to anything but an
`<img>` via a blob URL. Large attachments also strain browser storage quotas and
make exports heavy.

## Rendered Markdown preview

Show a note rendered, not just as raw text.

**Depends on:** a Markdown renderer that cannot become a cross-site scripting
hole, since injected script in this app reads every note. That means either a
small renderer that builds DOM nodes directly (never HTML strings) or a
well-audited library plus a sanitizer — and Trusted Types enforced by CSP.

## Verifiable deploys

Let a user confirm that the code their browser runs is exactly what a given
commit builds.

The app already shows its commit and asks before updating. The next step is
reproducible builds: the build digest stamped into the service worker, published
with each release, and recomputable by anyone from the repo. A small "verify"
page or script could compare them.

**Depends on:** a deterministic build — same input, same bytes — which needs
checking on Bun's bundler and pinning its version.

## Compression

Compress notes before encrypting them, with the built-in `CompressionStream`.

Left out of the MVP because text notes are small and padding already dominates
their stored size; compression would add a step to the format and to every
decrypt script for little gain. Worth reconsidering alongside attachments.

## Emptying the Trash automatically

Permanently delete notes that have sat in the Trash for a set time, as Google
Keep does after seven days.

Left out of the MVP so that nothing is ever deleted without the user asking.
**Depends on:** a setting, defaulting to off, and deciding when the sweep runs
— only while unlocked, since it needs the index.

## Merging different vaults on import

Importing an export from an unrelated vault into an existing one.

**Depends on:** re-encrypting every imported note under the local master key,
and deciding what happens to the other vault's keyring. Refused in format
version 1, which keeps import simple: it is either a restore or a same-vault
merge.

## A passphrase generator

Offer a generated passphrase during setup instead of relying on the user's
password manager.

**Depends on:** a wordlist (the EFF large list is about 60 KB) and the judgment
that it's worth carrying. Password managers already do this well.

## A decrypt script for Windows without Node

`tuck-decrypt.mjs` needs Node.js. A PowerShell version would let Windows users
decrypt with nothing installed — PowerShell can reach .NET's `AesGcm` and
`Rfc2898DeriveBytes`.

**Depends on:** someone to keep a second implementation in lockstep with
`docs/format.md`, and test vectors that both must pass.

## Backup reminders

Note when the vault was last exported, and nudge gently when that was a while
ago.

Browser storage can be wiped by clearing site data, or by Safari's eviction of
sites that aren't installed, and in a local-only app an export is the only
backup.

**Depends on:** a `lastExported` time in the index, and a judgment about what
"a while" is that doesn't turn into nagging.

## Filtering the note list

A field above the list that narrows it to notes whose display title matches.

**Depends on:** nothing new — display titles come from the index — just
whether the list grows long enough to need it.

## Changing the passphrase

Set a new passphrase from settings.

Cheap by design: only the master key's passphrase-wrapped copy is re-made; no
note is re-encrypted.

**Depends on:** deciding what a change means for exports made earlier, which
still open with the old passphrase.

## Passkeys from more than one password manager

Let one vault be unlocked by passkeys from, say, both iCloud Keychain and
Google Password Manager, so a device outside the first passkey's ecosystem
gets screen-lock unlock too, instead of the passphrase every time.

**Depends on:** the keyring's `passkey` becoming a list, which is a format
version change; a settings action to add one after a passphrase unlock; and
import adding passkeys the local keyring lacks.

## Richer exported files

Make `tuck-decrypt.mjs`'s output carry more than the body:

- YAML frontmatter with the title, `created`, and `updated` — the title
  otherwise survives only in the file name, which loses characters to
  sanitizing;
- each file's modification time set to `updated`;
- trashed notes in a `Trash/` subfolder rather than mixed in with the rest.

**Depends on:** a decision that the output is meant to be re-imported into
other tools that read frontmatter, rather than read as plain files.

## A link to the diff in the update prompt

When an update is waiting, link the prompt to GitHub's compare view between
the running commit and the new one, so reviewing what changed is one tap.

**Depends on:** only the commit SHAs, which the update prompt already has.

## Keeping notes out of app-switcher screenshots

iOS and Android take a screenshot of an app as it goes to the background, for
the app switcher. If that happens before the app locks, the switcher shows an
open note.

**Depends on:** first checking whether it happens at all on each platform, and
then whether hiding content on `visibilitychange` or `pagehide` runs early
enough.

## A dark theme

Follow the system's dark mode.

**Depends on:** the visual design settling first, so there is one palette to
derive the other from.
