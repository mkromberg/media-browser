# Dyalog APL media file server

## Context

The user wants a standalone Dyalog APL media file server that exposes an HTTP/JSON API for listing image and video files in a configured folder, so that a client application can browse and display media from that folder without needing filesystem access itself.

Decisions already made with the user:

- HTTP framework: **Jarvis** (Dyalog's official REST microservice framework, built on the bundled Conga library), rather than raw Conga or HttpCommand-as-server.
- The **root** folder is fixed via a **config file**, not a CLI argument or per-request parameter. Requests may name a **subfolder relative to that root** to browse into it (see "Frontend" below); the root itself is never overridable, and every subfolder request is resolved and validated to stay inside it.
- Any single scan (root or a requested subfolder) is **top-level only** (no recursion below the folder being scanned).
- A scan's response includes both the matching files and a list of that folder's immediate **subfolder names**, so a client can see what's available without the service ever recursing automatically.
- Per-file metadata is **full**: path, filename, extension, type, size, last-modified, plus **EXIF tags for images**, extracted by a **native APL parser** (no `exiftool`/`ffprobe`, neither of which is installed, and shelling out was explicitly rejected to keep the service dependency-free).
- Video files get basic metadata only; no duration/codec extraction.
- A minimal **browser frontend**, served by the same Jarvis instance, lets a user browse the configured folder and its subfolders, view image thumbnails, and play videos, without the client needing filesystem access itself. This requires the API to grow a second endpoint that streams raw file bytes (see "API additions for the frontend" below).

Both the service architecture and the EXIF parser were designed by fetching and reading the actual upstream Jarvis source and by running live `dyalogscript` experiments to confirm APL primitive behaviour (`⎕NREAD`, `⎕NINFO`, `⎕NPARTS`, `⎕JSON`, endian-decode idioms) rather than relying on assumptions. Remaining open questions are called out explicitly below, to be confirmed empirically during the RED phase rather than guessed at.

## Repository layout

First Dyalog code in the repo gets its own top-level directory, parallel to the existing `app/`/`static/`/`tests/` (Python, untouched). Existing `docs/plans`, `docs/prs`, `docs/bugs`, `docs/reviews` stay at the repo root per `CLAUDE.md` and are not duplicated.

```
apl-service/
  vendor/jarvis/
    Jarvis.dyalog            # vendored verbatim from github.com/Dyalog/Jarvis Source/Jarvis.dyalog
    VENDORED_FROM.md         # source URL, commit SHA, fetch date
  config/
    jarvis.config.json       # Jarvis framework config (native keys only)
    app.config.json          # our own config: just { "rootFolder": ... }
  APLSource/
    Initialize.aplf          # AppInitFn: loads app.config.json into AppState
    LoadAppConfig.aplf       # path -> (rc cfg), pure, no Jarvis dependency
    Get.aplf                 # REST GET handler: routes /files, /file, and static frontend assets
    CheckFolder.aplf         # root -> 'ok' | 'not-found' | 'not-a-directory' | 'not-readable'
    ResolveSubfolder.aplf    # (root relFolder) -> (rc path); rejects anything that resolves outside root
    ScanFolder.aplf          # folder -> file paths/sizes/mtimes + subfolder names (one ⎕NINFO call)
    ClassifyExtension.aplf   # extension -> 'image' | 'video' | ''
    ContentTypeFor.aplf      # extension -> MIME type string, or '' if unrecognised
    FormatTimestamp.aplf     # ⎕TS vector -> ISO-8601-ish string
    BuildFileMeta.aplf       # (size mtime) BuildFileMeta path -> per-file metadata namespace
    BuildListingResponse.aplf # folder -> full response namespace (orchestrator)
    ParseEXIF.aplf           # path -> namespace of EXIF tags (see EXIF design below)
    ServeFile.aplf           # (root path) -> raw bytes + content-type, for GET /file
    ServeStaticAsset.aplf    # (frontendRoot urlPath) -> file bytes + content-type, for the frontend
  frontend/
    index.html                # single page: folder view + breadcrumb + media grid
    app.js                    # fetches /files, renders grid/breadcrumbs, drives navigation state
    style.css
  scripts/
    start-server.apls        # bootstrap: fix vendor, ⎕NEW Jarvis, set JarvisConfig, .Start, keep-alive
    run-dev.sh                # cd to repo root, exec start-server.apls
  tests/
    unit/
      ClassifyExtensionTests.aplf
      CheckFolderTests.aplf
      ResolveSubfolderTests.aplf
      ScanFolderTests.aplf
      FormatTimestampTests.aplf
      BuildFileMetaTests.aplf
      LoadAppConfigTests.aplf
      BuildListingResponseTests.aplf
      ParseEXIFTests.aplf
      ServeFileTests.aplf
    e2e/
      SmokeTest.aplf           # HttpCommand-driven, real server against a fixture folder
    fixtures/
      app.config.sample.json
      with_exif.jpg / no_exif.jpg / not_an_image.png / corrupt_truncated.jpg
      generate_fixtures.apls   # hand-crafts the binary fixtures byte-by-byte
    run-tests.apls              # fixes src+tests, runs Test_* functions, exit code = pass/fail
```

Test files mirror `APLSource/` one-to-one inside `apl-service/tests/`, the closest honest analogue to the repo's existing (Python-specific) `tests/` mirroring `app/`. `frontend/` has no tests of its own beyond the e2e smoke test driving it indirectly through the API; it is static markup and vanilla JS with no build step.

`.aplf` names a file `⎕FIX`ed into the workspace, whether it holds a single bare function (`APLSource/*.aplf`) or a named namespace script bundling several functions (`tests/unit/*.aplf`). `.apls` names a file dyalogscript executes directly as top-level statements and never fixes: `run-tests.apls`, `start-server.apls`, `generate_fixtures.apls`.

## Configuration schemas

`apl-service/config/jarvis.config.json` (native Jarvis keys only):

```json
{
  "CodeLocation": "../APLSource",
  "Port": 8080,
  "Paradigm": "REST",
  "RESTMethods": "Get",
  "AppInitFn": "Initialize",
  "Debug": 0
}
```

`CodeLocation` resolves relative to this file's own location, so it works regardless of process cwd. `RESTMethods: "Get"` means Jarvis itself rejects PUT/POST/DELETE with 405, no app code needed.

`apl-service/config/app.config.json` (ours; the only app-specific setting):

```json
{
  "rootFolder": "/srv/photos"
}
```

`Initialize` reads this from an absolute config-file path: `$APL_SERVICE_APP_CONFIG` if set (which must itself be absolute), else an absolute path derived from the service's own install location, never resolved against the process working directory. The env var only ever names _which file_ to read (used by the e2e test to point at a fixture folder); the root folder value itself always comes from JSON content, never from the environment or a request parameter. `LoadAppConfig` likewise takes an absolute path. The exact default-anchor mechanism is settled in the epic-1 issue that ships `Initialize` and its end-to-end coverage, not in the `LoadAppConfig`-only issue.

## Request flow and function responsibilities

`Get req` splits `req.Endpoint` and dispatches on the leading path segment:

| Path          | Handling                                                                                       |
| ------------- | ------------------------------------------------------------------------------------------------ |
| `/files`      | listing endpoint, described below                                                                |
| `/file`       | raw byte-serving endpoint, described in "API additions for the frontend"                         |
| anything else | served as a static frontend asset from `apl-service/frontend/`, or `req.Fail 404` if no match     |

For `/files`: read the optional `folder` query parameter; `ResolveSubfolder AppState.RootFolder folder` maps it (absent or empty means the root itself) to an absolute path guaranteed to be inside the root, or fails. Then `CheckFolder` that resolved path, and on `'ok'` call `BuildListingResponse` and return it (auto-JSON-marshalled by Jarvis); otherwise map the status to an HTTP error:

| `CheckFolder` / `ResolveSubfolder` result | HTTP response                                                                                      |
| ------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `'not-found'`       | 404 - folder doesn't exist (ordinary client-facing condition)                                        |
| `'not-a-directory'` | 500 for the root, 404 for a requested subfolder (client asked for something that isn't a folder)     |
| `'not-readable'`    | 500 - misconfiguration/permissions                                                                    |
| `'outside-root'`    | 400 - the `folder` parameter resolved (via `..` or an absolute path) to somewhere outside `AppState.RootFolder` |

`Get` wraps its calls to `BuildListingResponse` and `ServeFile` in `:Trap 0` to turn any unexpected mid-scan error into a clean `req.Fail 500` rather than a stack trace.

Pure, unit-testable functions (no server, no network):

- **`ClassifyExtension ext`** - lower-cases, strips leading `.`, looks up two local extension lists (images: `jpg jpeg png gif bmp tiff webp heic`; videos: `mp4 mov avi mkv webm m4v`), returns `'image'`/`'video'`/`''`.
- **`ContentTypeFor ext`** - lower-cases, strips leading `.`, maps to a concrete MIME type via a per-extension table (`jpg`/`jpeg`->`image/jpeg`, `png`->`image/png`, `gif`->`image/gif`, `bmp`->`image/bmp`, `tiff`->`image/tiff`, `webp`->`image/webp`, `heic`->`image/heic`, `mp4`->`video/mp4`, `mov`->`video/quicktime`, `avi`->`video/x-msvideo`, `mkv`->`video/x-matroska`, `webm`->`video/webm`, `m4v`->`video/x-m4v`), returns `''` for anything `ClassifyExtension` would also reject. Kept separate from `ClassifyExtension`: routing and EXIF-gating only need the broad `'image'`/`'video'` category, but `Content-Type` needs the exact subtype.
- **`CheckFolder folder`** - `⎕NEXISTS` / `1 ⎕NINFO` (type) / `11 ⎕NINFO` (readability, needs a Linux spot-check since the manual allows `¯1`="unknown" on some platforms). Takes whichever absolute folder path it's given, root or a resolved subfolder; it has no notion of "the" root.
- **`ResolveSubfolder root relFolder`** - the path-traversal gate. Empty `relFolder` returns `root` unchanged. Otherwise joins `root` and `relFolder`, then canonicalises (resolving `.`/`..` and symlinks) and checks the result is either exactly `root` or starts with `root` followed by the path separator; anything else (including a sibling directory such as `root,'-evil'`, which a bare string-prefix match would wrongly admit) returns `'outside-root'` rather than a path. This is the only function permitted to turn client-supplied path text into a filesystem path passed to `CheckFolder`/`ScanFolder`/`ServeFile`.
- **`ScanFolder folder`** - one call, `(0 1 2 3)(⎕NINFO⍠1)folder,'/*'`, splitting by type into file paths/sizes/mtimes and bare subfolder names (via `⎕NPARTS`). Never recurses below `folder`, whether `folder` is the root or a resolved subfolder.
- **`FormatTimestamp ts`** - `⎕TS`-format vector to `YYYY-MM-DDTHH:MM:SS` (confirm during RED phase whether `⎕NINFO` mtimes are local or UTC before deciding on a trailing `Z`).
- **`BuildFileMeta stat path`** - `stat` is `size mtime` (already known from `ScanFolder`, no redundant stat). Computes extension/filename via `⎕NPARTS`, classifies; returns `⍬` for unrecognised extensions (dropped from the listing). Builds `.fullPath .fileName .extension .type .size .lastModified .url`, where `.url` is `/file?path=` plus the URL-encoded `.fullPath`, for the frontend to use directly as an `<img src>`/`<video src>`; for `type='image'`, calls `ParseEXIF path` and, only if `.Found`, nests the non-empty tags under `.exif` (omitted entirely otherwise). Video files never call `ParseEXIF`.
- **`LoadAppConfig path`** - reads/parses JSON, validates non-empty `rootFolder`, returns `(0 cfg)` or `(1 message)`, never signals.
- **`BuildListingResponse folder`** - orchestrates `ScanFolder` + `BuildFileMeta` per file, assembles `.folder .files .subfolders`. This is the one function whose tests must specifically cover the `⎕JSON` array-shape invariant below.
- **`ServeFile root path`** - re-validates `path` is inside `root` (defence in depth: `Get` already resolved it via `ResolveSubfolder`, but `ServeFile` never trusts a caller-supplied absolute path on its own) and that `ClassifyExtension` recognises its extension, rejecting anything else rather than serving an arbitrary readable file from inside `root`; then `⎕NREAD`s the raw bytes and returns them alongside a `Content-Type` from `ContentTypeFor` for Jarvis to set on the response.

Jarvis-facing glue:

- **`Initialize`** (`AppInitFn`) - resolves app-config path, calls `LoadAppConfig`, assigns to global `AppState` (visible to `Get` since both execute inside Jarvis's `CodeLocation` namespace). Non-zero return means the server refuses to start on a malformed `app.config.json` - fail fast, distinct from a missing _folder_, which is a per-request 404.

**Verified `⎕JSON` gotcha to guard against**: a bare namespace serialises as a JSON object, not a one-element array (`1 ⎕JSON⍠'HighRank' 'Split'⊢ns` gives `{...}`, but `⊢,⊂ns` gives `[{...}]`). `files` and `subfolders` must always be built as enclosed vectors so that 0, 1, and N matches all serialise as JSON arrays - this needs an explicit test for each of those three cases, not just the N-case.

## API additions for the frontend

Two changes to the API surface support the browser frontend; both are additive, and existing `GET /files` behaviour against the root (no `folder` parameter) is unchanged.

- **`GET /files?folder=<relative-path>`** - re-scans a subfolder instead of the root, using the same `ScanFolder`/`BuildFileMeta` pipeline and the same top-level-only, no-implicit-recursion rule, just retargeted at `ResolveSubfolder AppState.RootFolder folder`. `folder` is always interpreted relative to the root; there is no way to make it name an absolute filesystem location. The response's `.folder` field echoes back the folder actually listed (as an absolute path), and `.subfolders` remains bare child names for the frontend to append to whatever relative path it is currently browsing.
- **`GET /file?path=<url-encoded-absolute-path>`** - streams the raw bytes of a single file, with `Content-Type` set from `ContentTypeFor`. `path` must already be one of the `.fullPath` values a prior `/files` response returned; `ServeFile` re-validates it against both `AppState.RootFolder` and `ClassifyExtension` regardless, and any path outside the root, naming something other than a plain readable file, or naming a file whose extension isn't recognised, is a 400 rather than a filesystem error - or an arbitrary-file read - leaking through. There is no directory listing or `..` support in `path` - only exact file paths already surfaced by `/files`, and the extension check makes that a server-enforced guarantee rather than only a client convention.

## Example response

```json
{
  "folder": "/srv/photos",
  "files": [
    {
      "fullPath": "/srv/photos/IMG_0001.JPG",
      "fileName": "IMG_0001.JPG",
      "extension": "jpg",
      "type": "image",
      "size": 4821932,
      "lastModified": "2026-06-30T14:22:05",
      "url": "/file?path=%2Fsrv%2Fphotos%2FIMG_0001.JPG",
      "exif": {
        "Make": "Canon",
        "Model": "Canon EOS 5D",
        "DateTimeOriginal": "2026:06:30 14:22:05",
        "Orientation": 1,
        "ImageWidth": 4000,
        "ImageHeight": 3000,
        "GPSLatitude": 51.5074,
        "GPSLongitude": -0.1278
      }
    },
    {
      "fullPath": "/srv/photos/no_exif.png",
      "fileName": "no_exif.png",
      "extension": "png",
      "type": "image",
      "size": 102400,
      "lastModified": "2026-06-29T08:00:00",
      "url": "/file?path=%2Fsrv%2Fphotos%2Fno_exif.png"
    },
    {
      "fullPath": "/srv/photos/clip.mp4",
      "fileName": "clip.mp4",
      "extension": "mp4",
      "type": "video",
      "size": 20481932,
      "lastModified": "2026-06-28T09:10:00",
      "url": "/file?path=%2Fsrv%2Fphotos%2Fclip.mp4"
    }
  ],
  "subfolders": ["2024-summer", "2025-winter"]
}
```

`GET /files?folder=2024-summer` returns the same shape for `/srv/photos/2024-summer`, with `.folder` set to that absolute path and `.subfolders`/`.files.*.fullPath` naming that folder's own children.

A file with an unrecognised extension (e.g. `document.txt`) is silently excluded. `.exif` appears only when EXIF was actually found, and only with the fields that resolved.

## EXIF parser design (`ParseEXIF`)

Native binary parser, no external tools. Verified live via `dyalogscript` against hand-built JPEG/TIFF header bytes: `⎕NTIE`/`⎕NREAD` with conversion code `83` reads raw bytes as signed 8-bit integers (mask with `256|` for unsigned); `⎕NREAD` degrades to short reads rather than erroring, which simplifies truncated-file handling to length checks; big-endian decode is `256⊥bytes`, little-endian is `256⊥⌽bytes`.

**Algorithm**:

1. `FindAPP1`: read a small bounded prefix of the file (JPEG's 16-bit segment-length field caps a single APP1 segment, and therefore the whole EXIF TIFF blob, at 65535 bytes - one bounded read is provably sufficient). Confirm SOI (`FF D8`); walk markers until an APP1 (`FF E1`) segment whose payload starts `Exif\0\0` is found, or until SOS/EOF is reached with none found. Any non-JPEG file (first two bytes not `FF D8`) is rejected in this one check. The JPEG segment-length field is always big-endian regardless of the TIFF byte order declared inside APP1 - a documented, easy-to-miss invariant worth a comment in the code.
2. `ParseTIFFHeader`: validate `II`/`MM` byte-order marker + 42 magic, decode the IFD0 offset.
3. `ReadIFDEntries` / `ResolveEntryValue`: walk IFD0's 12-byte entries (bounds-checked, entry count capped defensively), resolving inline values (≤4 bytes) vs offset-referenced values. Follow tag `0x8769` to the Exif sub-IFD and `0x8825` to the GPS sub-IFD when present.
4. Tag set in scope: `Make`, `Model`, `Orientation` (IFD0); `DateTimeOriginal`, `ExifImageWidth`, `ExifImageHeight` (Exif sub-IFD); `GPSLatitudeRef/Latitude`, `GPSLongitudeRef/Longitude` (GPS sub-IFD, RATIONAL degree/minute/second triples converted to signed decimal degrees, populated only if both axes resolve).
5. Result contract: `meta←ParseEXIF path` always returns a namespace with a `Found` boolean plus one field per tag, "absent" uniformly represented as `''`/`⍬` - never `⎕NULL`, never a raised error. `BuildFileMeta` checks `.Found` before nesting anything under `.exif`.

**Error handling**: every failure mode (missing path, non-JPEG, no APP1, truncated segment, malformed TIFF header, out-of-bounds IFD entries, zero-denominator RATIONAL) degrades to `Found←0` or an individual empty field, detected at a specific, named step - never relying solely on a catch-all trap, though an outer `:Trap 0` exists as a last line of defence.

**Known gap**: PNG files carrying a real `eXIf` chunk (rare) are not parsed; `Found` will be `0` for them. A fixture/test documents this as current, intentional behaviour rather than silently missing coverage.

**Fixtures** (hand-crafted via a checked-in `generate_fixtures.apls`, avoiding a Pillow/piexif dependency purely for test data): `with_exif.jpg` (full tag set, `N`/`W` GPS refs to exercise the sign flip), `no_exif.jpg`, `not_an_image.png`, `corrupt_truncated.jpg` (byte-truncated mid-IFD0, must return `Found=0` with no signalled error, not just avoid crashing the process).

## Frontend

`apl-service/frontend/` is plain HTML/CSS/JS with no build step and no framework, served by the same Jarvis instance and port as the API (via `Get`'s static-asset fallback), so the page and the API it calls are always same-origin and there is no CORS configuration to get wrong.

- **`index.html`** - a single page: a breadcrumb bar, a media grid, and (on selecting a file) a detail panel showing full metadata and, for images, the EXIF block.
- **`app.js`** - all client logic. State is just "current relative folder", initialised from a `?folder=` query parameter on the page URL itself (so a browsed-into folder is bookmarkable/shareable and survives a reload) and updated via `history.pushState` on navigation, never a page reload. On any folder change it calls `GET /files?folder=<current>`, then renders:
  - the breadcrumb bar by splitting the current relative folder path into clickable segments (root always present as the first, un-removable segment);
  - one grid tile per entry in `.files`: an `<img>` for `type='image'` and a `<video muted preload="metadata">` for `type='video'`, both pointed at `.url`, giving a native browser thumbnail/first-frame without any server-side thumbnailing; a plain file-icon tile for anything `ScanFolder` skipped is impossible by construction, since unrecognised extensions never reach the response;
  - one entry per `.subfolders` name, rendered as a folder tile that on click appends that name to the current relative folder and re-fetches;
  - clicking a media tile opens the detail panel (filename, size, formatted timestamp, and the `.exif` block when present) and, for video, switches that tile's `<video>` to `controls` so it can be played in place rather than only auto-loading its first frame.
- **`style.css`** - a responsive grid layout only; no visual design system is in scope.

Deliberately out of scope: authentication, pagination/lazy-loading for very large folders, drag-drop upload or any write operation (the API is GET-only end to end), and search/filter across folders. All of these are extensions to layer on later if needed, not gaps in this plan.

## Running the service

```bash
cd /workspace
dyalogscript apl-service/scripts/start-server.apls
# separately:
curl -s http://localhost:8080/files | python3 -m json.tool
# or, to browse interactively:
open http://localhost:8080/          # serves apl-service/frontend/index.html
```

`start-server.apls` fixes the vendored `Jarvis.dyalog`, `⎕NEW`s an instance, points `.JarvisConfig` at `apl-service/config/jarvis.config.json`, calls `.Start`, then loops on `.Running`. The exact necessity of the keep-alive loop and the semantics of `.Running` are inferred from Jarvis's own `AutoStart.dyalog` pattern but get a real `curl`-against-a-running-instance confirmation during the RED phase, before the e2e smoke test is written to depend on it.

## Test strategy

**Unit tests** (`apl-service/tests/unit/*.aplf`, no network): `tests/run-tests.apls` fixes `APLSource/*.aplf` and the test namespaces, discovers niladic `Test_*` functions via `⎕NL`, runs each under `:Trap 0`, reports pass/fail, exits 0/1. This is the "full suite" run referred to by `CLAUDE.md`'s workflow.

Concrete coverage required: extension classification (all listed extensions, case-insensitivity, unknown -> `''`); folder-check outcomes including a `chmod 000` fixture for `'not-readable'`; folder scan split (files vs subfolders, no recursion); timestamp formatting edge cases (zero-padding); file-meta building (unrecognised extension dropped, EXIF nested only when present, video never calls `ParseEXIF`, `.url` correctly URL-encodes a path containing spaces/unicode); app-config loading (valid, missing, malformed, missing/empty key); listing-response JSON array shape at 0/1/N files; the full `ParseEXIF` tag set plus all documented degradation cases; `ResolveSubfolder` (empty input returns root unchanged, a valid child subfolder resolves, `../../etc`-style and absolute-path inputs both return `'outside-root'`, a symlink inside root pointing outside it is rejected, a sibling directory sharing root's name as a string prefix - e.g. `<root>-evil` next to `root` - is rejected despite the prefix match); `ServeFile` (valid image/video paths return the right bytes and content-type, a path outside root is rejected even if well-formed, a path naming a directory or nonexistent file is rejected, a path naming a readable file inside root with an unrecognised extension - e.g. a stray `.txt` - is rejected rather than served); `ContentTypeFor` (every extension in both `ClassifyExtension` lists maps to its exact MIME type, an unrecognised extension returns `''`).

**End-to-end smoke test** (`apl-service/tests/e2e/SmokeTest.aplf`, modelled on Jarvis's own `Samples/REST/Test.aplf`, using the bundled `HttpCommand.dyalog`): builds a temp fixture folder (including a nested subfolder with its own files, to exercise `folder` navigation), points a temp `app.config.json` at it via `$APL_SERVICE_APP_CONFIG`, launches `start-server.apls` as a background process, polls for the port, then asserts `GET /files` (200, expected shape), `GET /files?folder=<subfolder>` (200, scoped to that subfolder), `GET /files?folder=../outside` (400), `GET /file?path=<one of the returned fullPaths>` (200, correct content-type and byte length), `GET /file?path=/etc/passwd` (400, rejected despite being a real readable file, because it's outside root), `GET /file?path=<a readable non-media file inside the fixture folder, e.g. a stray .txt>` (400, rejected despite being inside root, because its extension isn't recognised), `GET /bogus` (404), `PUT /files` (405, free from Jarvis config), a nonexistent root folder (404), and `GET /` (200, serves `index.html`). Run manually per PR cycle (not on every micro red/green loop) and record the result in `docs/prs/<id>.md`.

## Verification

1. `dyalogscript apl-service/tests/run-tests.apls` - full unit suite passes, no regressions.
2. `dyalogscript apl-service/scripts/start-server.apls` against a real folder containing a mix of images (with and without EXIF), videos, an unrecognised file type, and a subfolder; `curl http://localhost:8080/files` and inspect the JSON matches the documented shape, including the 0/1/N-files array-shape cases; repeat with `?folder=<subfolder>` and confirm the response scopes to it.
3. `dyalogscript apl-service/tests/e2e/SmokeTest.aplf` for the full request/response/error-code cycle against a real running server.
4. Manually confirm the two flagged open questions before relying on them in tests: whether `⎕NINFO` mtimes are local or UTC, and whether `11 ⎕NINFO` reliably reports 0 (not `¯1`) for a permission-denied directory on this platform.
5. Open `http://localhost:8080/` in a browser against that same real folder: confirm the root grid renders with working image thumbnails and playable videos, clicking a subfolder tile navigates into it and updates the URL's `?folder=`, the browser back button returns to the parent folder, clicking a file shows its metadata (and EXIF, for images that have it) in the detail panel, and a reload with `?folder=<subfolder>` in the address bar lands directly in that subfolder.

## Implementation epics

The build is sequenced into five epics. Each one ends with a running, shippable server, not a partial slice: every epic's acceptance criteria is a `curl` (or browser) session against a real server process, not just a passing unit test. Each epic is one `{issue-id}-{slug}` branch under the per-issue workflow, with its own `docs/prs/<id>.md`.

Ordering rationale: epics 1-2 grow breadth (the whole folder tree is browsable) before depth; epic 3 unlocks raw byte access before epic 4 enriches metadata, so the frontend in epic 5 never blocks on an incomplete API.

1. **Bootstrap and root listing.** Vendor Jarvis; `jarvis.config.json` and `app.config.json`; `Initialize`, `LoadAppConfig`, `CheckFolder`, `ScanFolder`, `ClassifyExtension`, `FormatTimestamp`, `BuildFileMeta` (no EXIF yet), `BuildListingResponse`; `Get` wired to `/files` for the root folder only, no `folder` parameter. This epic also stands up the e2e smoke-test harness itself (temp fixture folder, background server launch, port poll), which later epics extend with assertions rather than rebuild. Shippable: `dyalogscript apl-service/scripts/start-server.apls` runs; `curl localhost:8080/files` against a real folder returns correct JSON (path/name/extension/type/size/mtime, `files`/`subfolders` arrays, 0/1/N shape-correct).
2. **Subfolder navigation.** `ResolveSubfolder`; the `folder` query parameter on `/files`; the `outside-root`/`not-a-directory` error-code mapping in `Get`. Shippable: same API, now browsable into the whole tree via `?folder=`, with traversal escapes rejected (400).
3. **Raw file streaming.** `ServeFile`; `ContentTypeFor`; the `GET /file?path=` endpoint; the `.url` field on each file entry. Shippable: a client (curl, or a hand-written `<img src>`) can fetch actual bytes for any file the API surfaced; `path` is validated against both the root and the recognised-extension list, independently of `ResolveSubfolder`/`ClassifyExtension`'s earlier use on the listing path.
4. **EXIF metadata.** `ParseEXIF` and its integration into `BuildFileMeta` for `type='image'`. Shippable: same API, image entries now carry `.exif` when present; all documented degradation cases (non-JPEG, no APP1, truncated, malformed TIFF) return `Found=0` rather than erroring.
5. **Browser frontend.** `index.html`, `app.js`, `style.css`; the static-asset fallback in `Get`. Shippable: the full end-user product as scoped above, a grid/breadcrumb/detail-panel browsing a real folder in a browser.
