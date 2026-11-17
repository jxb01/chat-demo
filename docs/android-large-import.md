# Importing large history exports from an Android WebView wrapper

This note documents what happens when the static build is wrapped in an Android app
and a real DeepSeek export is imported through **Settings → Data → Import chat history**.
It is written from measurements, not assumptions; it may help wrappers and future
changes to the import path.

## Setup

- build: `pnpm install && NODE_ENV=production pnpm exec rspack build`
- wrapper: `dist/` served from app assets through `WebViewAssetLoader`
  (`https://appassets.androidplatform.net/assets/www/index.html`)
- device: Android 14 emulator, system WebView
- input: DeepSeek export ZIP (`manifest.json` with `export_format_version: 3`)

## Measurements

| Input | Result |
|---|---|
| 2 conversations, 2.4 MB | imports in seconds |
| 1,078 conversations, 18.5 MB | modal buttons stay disabled 3+ minutes, then no sessions are added |
| 4,499 conversations, 637 MB, 5,502 entries | same |

No progress is shown while importing, and the only error signal is a line of red text
inside the settings modal, which disappears when the page reloads. From the outside it
looks like the import silently did nothing.

## Why it happens

1. **The import replaces, it cannot append.**
   `services/chat/backup.ts` calls `replaceHistory(...)`, which stages sessions and then
   swaps the whole `sessions` store. Splitting a big archive into several imports is
   therefore not possible: every import would wipe the previous one.

2. **One unacceptable session aborts everything.**
   `services/chat/history/schema.ts` validates every message strictly (status enum,
   RFC3339 timestamps, non-negative integer counters). Any mismatch raises
   `backupIncomplete` and the whole import is discarded. Real exports contain
   `FILE`, `TOOL_SEARCH`, `READ_LINK` and `CONTENT_RISK` fragments that can fail
   this validation.

3. **The ZIP path is expensive on Android.**
   `services/chat/history/zipReader.ts` reads the central directory, every local
   header, every name and every extra field with `file.slice(...).arrayBuffer()`, and
   computes CRC32 byte by byte in JavaScript. On Android the picked file is a
   MediaProvider `content://` URI, so each slice may reopen the file:
   5,502 entries means tens of thousands of opens, and 741 MB uncompressed means
   roughly 740 million loop iterations inside the WebView.

## A wrapper-side workaround

A wrapper can avoid the web file path entirely:

1. parse the ZIP natively (`java.util.zip.ZipFile`) — one open, no repeated SAF reads;
2. normalise sessions into the local model (linear messages, text only, millisecond
   timestamps, `status: "done"`) so validation cannot fail;
3. write in batches of ~50 sessions through a small global that performs
   `sessions.put(...)` / `sessionSummaries.put(...)` — a merge, not a replace — and
   call back into the shell when the batch commits;
4. report progress and reload when finished.

With that path the full 637 MB export (4,499 conversations) imports in about 45 seconds
on the emulator. Attachments, branches, thinking blocks and citations are dropped in
this route.

If a lossless path is wanted upstream, the smallest changes would be:

- expose an append/merge entry point next to `replaceHistory`, and
- let the importer consume an abstract byte source (Blob **or** host-provided stream)
  so wrappers can feed a `content://` file without `File.slice()` per entry.

A reference implementation of the wrapper is available at
<https://github.com/jxb01/deepseek-chat-android> (MIT).
