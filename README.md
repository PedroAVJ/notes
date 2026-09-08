# Notes

Create, find, read, update, organize, and delete Apple Notes on macOS through
host-appropriate live UI automation; create true native interactive
checklists; and retain the existing phone/FaceTime call-recording and private
transcript-cache workflows.

Version 0.6.1 supports both Codex and Claude. Codex controls Notes.app through
OpenAI's bundled Computer Use capability. Claude uses the plugin-owned `notes`
CLI, whose Node argument parser launches a bounded System Events UI script. The
write path never uses Python or Notes' sync-blocked account scripting
dictionary. The pinned, MIT-licensed `apple-notes-mcp@2.7.5` server remains
available for bounded discovery and reads.

## Create Notes

Ask Codex to create the note. The Notes skill invokes
`computer-use:computer-use`, opens Notes.app, enters the title and body, and
reads the live accessibility tree back before reporting success. It uses the
currently selected/default Notes folder unless the request explicitly names a
different folder or account.

Claude uses the same live Notes UI through the CLI:

```bash
notes create --title "Project brief" --body "First draft" --json
notes create --title "Long note" --stdin --json < body.txt
```

The CLI deliberately uses the currently selected/default Notes folder and
rejects `--account` and `--folder`; it does not enumerate iCloud accounts.

## Create Native Checklists

Apple Notes stores checklist style in a private gzipped protobuf and exposes no
checklist creation property through AppleScript. This plugin does not write the
live Notes database. Codex enters clean item lines in Notes.app, selects only
those lines, invokes Notes' own Shift+Command+L checklist action, and verifies
that every item is exposed as a native unchecked control. Existing plain lists
can be converted in place without manufacturing a duplicate note.

Claude invokes Notes' native checklist Accessibility action through the CLI:

```bash
notes checklist create \
  --title "Packing" \
  --item "Passport" \
  --item "Charger" \
  --item "Medication" \
  --json
```

For a large checklist, `--items-stdin` accepts a JSON array of strings. The UI
script verifies the title, every item, and that Notes retained native checklist
formatting before reporting success.

## General Notes MCP

The bundled MCP definition exposes the upstream search, read, folder, account,
attachment, checklist-state, and diagnostic surface. Although the upstream
server also advertises mutation tools, this plugin keeps them out of its write
workflow because Notes account sync can block their AppleScript calls. It is
loaded only in a fresh agent task after installation.
The exact third-party source, version, integrity, and license are recorded in
[`THIRD_PARTY_NOTICES.md`](./THIRD_PARTY_NOTICES.md).

Do not bulk-read or export a Notes library unless the user asked for that scope.
Before rewriting an existing note, check for attachments and whether it is
shared; full-body AppleScript updates can remove embedded content and shared
edits are visible to collaborators.

## Call Recordings

Apple's phone and FaceTime call recording store remains strictly read-only.
The plugin maintains only a private, source-local transcript cache for those
recordings.

Transcription is on demand. The `notes:process-recorded-call` skill processes an
explicitly selected call, and the CLI can manually reconcile a bounded batch
when requested. Nothing polls Notes, uploads recordings, files, routes,
summarizes, or acts on a conversation in the background.

## Install

```bash
codex plugin add notes@package-manager
claude plugin install notes@package-manager
```

Version 0.6.1 replaces the former Python/account-AppleScript writer with the
direct UI path and restores the Claude manifest. Existing private cache rows
and transcript artifacts remain intact.

## Call Source CLI

```bash
notes calls list --json
notes calls list --since 2026-06-01
notes calls path A1B2C3D4
notes calls transcript A1B2C3D4
notes calls doctor --json
```

A call can be referenced by its attachment UUID, a unique UUID prefix, or a
contact name. An ambiguous name is an error listing the candidates.

## Transcript Cache

```bash
notes calls transcriptions reconcile --limit 10 --json
notes calls transcriptions retry A1B2C3D4 --json
notes calls transcriptions show A1B2C3D4 --json
notes calls transcriptions status --state completed --limit 100 --json
notes calls transcriptions status --state completed --limit 100 \
  --after-cursor "$CURSOR" --json
```

`status` is a bounded, read-only cache query; it never reads Notes or invokes a
transcriber. Its opaque cursor allows independent consumers to keep their own
checkpoints. Completed items expose the stable UUID, call time and title,
source path, provider/model/format, transcript artifact path and SHA-256, and
completion timestamps.

The cache lives under:

```text
~/Library/Application Support/notes/transcriptions/
```

Each UUID is protected by a process lock. Transcript and plist writes use an
fsynced temporary file plus atomic replacement. Failures retain exponential
retry state; successful items are idempotent across renames and iCloud
rematerialization.

## Provider Order

Apple's transcript is embedded in the call note body as a gzipped protobuf in
`ZICNOTEDATA.ZDATA`. The worker uses non-empty Apple text first. Only when it is
absent does it resolve `elevenlabs` from `PATH` and run Scribe v2 with automatic
call diarization:

```bash
elevenlabs transcribe AUDIO \
  --model scribe_v2 \
  --response-format diarized_text \
  --diarize \
  --out TEMP_PATH
```

It never passes a guessed speaker count and never reaches into an ElevenLabs
plugin cache or version directory.

## Why the Notes Store Is Authoritative

Apple represents one call with a titled parent attachment and a child carrying
the real duration. The attachment pair brackets the wall-clock call window.
Media folders outlive deleted notes, file mtimes are CloudKit sync artifacts,
and the live database is WAL-mode. The CLI therefore snapshots
`NoteStore.sqlite` with its WAL sidecars, opens that snapshot read-only, and
uses the attachment UUID as identity. Globbing `Call with *.m4a` is not safe.

## Requirements and Scope

- Codex desktop with the bundled Computer Use plugin, or Claude on macOS with
  persistent Accessibility and Automation access for the signed Claude host
- Notes.app with at least one configured Notes account
- Full Disk Access for database-backed reads, call recordings, checklist state,
  and note metadata
- Python 3 for the read-only call-recording and transcript-cache CLI
- Node.js 20 or newer for Claude's write argument parser and the pinned Notes
  MCP server
- `elevenlabs` on `PATH`; `ELEVENLABS_API_KEY` is needed only on cache misses

Ordinary note and checklist writes occur only when explicitly requested. The
plugin uses the live Notes UI and never writes `NoteStore.sqlite`, call-recording
notes, attachments, recordings, or Apple transcripts. Call infrastructure
writes only its private transcript cache, receipts, and locks.
Manual semantic processing never sends messages or creates domain records
without the authority of the invoked skill and current task.
