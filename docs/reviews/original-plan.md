# Review: docs/plans/original-plan.md - Dyalog APL media file server

## Summary

The plan is thorough and, unusually, its trickiest APL semantics claims are checked against a live `dyalogscript` interpreter rather than asserted from memory. Every claim tested during this review (`⎕JSON` array-shape behaviour, `⎕NREAD` conversion code 83 plus `256|` masking, short-read behaviour, the `256⊥`/`256⊥⌽` endian idioms, the single-call `(0 1 2 3)(⎕NINFO⍠1)folder,'/*'` scan, `⎕NPARTS`'s three-part return, and `⎕NINFO` type codes 1/2) holds exactly as described. The plan also explicitly flags two open questions for empirical confirmation during the RED phase; both were resolved directly in this review (see Findings) and can be removed from the open-questions list.

Three design gaps remain in the request-handling and file-serving surface that should be closed in the plan text before the corresponding epics (2 and 3) are implemented: the subfolder path-traversal check as described is vulnerable to a sibling-directory bypass, `ServeFile`'s validation does not actually restrict requests to media files despite the plan's stated intent, and `Content-Type` derivation from `ClassifyExtension` is under-specified for anything beyond the broad image/video split.

No implementation exists yet on this branch (`apl-service/` is untracked, empty scaffolding), so this is a pure design review; there is no code to check for architectural conformity or comment hygiene.

## Findings

### Major: `ResolveSubfolder`'s prefix check as described admits a sibling-directory bypass

The plan specifies the traversal gate as: canonicalise the joined path, "checks the result's prefix is still `root`". A plain string-prefix comparison treats `/srv/photos-private` as having prefix `/srv/photos`, so a symlink or directory named `photos-evil` next to `photos` would pass the check even though it is not inside the root. The plan's own test list (`../../etc`-style, absolute-path, symlink-escape) does not include this sibling case, so it would not be caught by the planned test suite either.

The fix is to compare against `root` followed by the platform path separator (or require exact equality with `root` itself), not a bare prefix match. Worth adding as an explicit test case alongside the other `ResolveSubfolder` cases already listed.

Location: "Pure, unit-testable functions" section, `ResolveSubfolder` bullet; also referenced in "Concrete coverage required" under Test strategy.

### Major: `ServeFile` does not enforce the "media files only" contract it claims

The "API additions for the frontend" section states `path` "must already be one of the `.fullPath` values a prior `/files` response returned" and that there is "no directory listing or `..` support in `path` - only exact file paths already surfaced by `/files`". But `ServeFile`'s actual validation, per its own bullet, is: re-validate inside root, then `⎕NREAD` the bytes; there is no check that the extension is one `ClassifyExtension` recognises. Any client can call `GET /file?path=<any readable file under root>`, including config files, dotfiles, or source files with unrecognised extensions that were never surfaced by `/files`. That contradicts the stated contract and quietly turns a media-file server into a general read-only file server for anything reachable under the root.

If this is intentional (simpler code, no real security boundary since read-only and root-confined), the plan should say so and drop the "only exact file paths already surfaced" language. If not, `ServeFile` needs to reject paths whose extension is unrecognised (empty `ClassifyExtension` result), and that case needs a test alongside the other `ServeFile` cases already listed.

Location: "API additions for the frontend" section, second bullet; `ServeFile` bullet in "Pure, unit-testable functions".

### Major: `Content-Type` derivation from `ClassifyExtension` is under-specified

`ClassifyExtension` is documented to return only the coarse category `'image'`/`'video'`/`''`. `ServeFile`'s bullet says it derives "a `Content-Type` ... from `ClassifyExtension` (`image/jpeg`, `video/mp4`, etc.)", which requires a full extension-to-MIME-type table (`png`->`image/png`, `mp4`->`video/mp4`, `mov`->`video/quicktime`, etc.), not the two-way image/video split `ClassifyExtension` actually returns. No function in the repository layout owns this mapping. Before epic 3 is implemented, the plan should either extend `ClassifyExtension`'s contract or name a separate function (e.g. `ContentTypeFor`) responsible for the extension->MIME mapping, and the unit-test coverage list should cover it per extension, not just per broad category.

Location: `ServeFile` bullet in "Pure, unit-testable functions"; epic 3 in "Implementation epics".

### Minor: not all classified video extensions are guaranteed browser-playable

The video extension list includes `avi` and `mkv`. Neither is reliably playable through a native HTML `<video>` element across current browsers (container/codec support varies), unlike `mp4`/`webm`/`mov`(Safari)/`m4v`. Epic 5's shippable criterion ("playable videos") may not hold uniformly for every extension `ClassifyExtension` accepts. Worth a note in the Frontend section that playback is best-effort and codec-dependent for less common containers, rather than a guarantee, or narrowing the video list to browser-safe formats.

Location: `ClassifyExtension` bullet (extension lists); Frontend section, `app.js` bullet.

### Note: both explicitly flagged open questions are resolved by this review

The plan calls out two items to "confirm empirically during the RED phase". Both were tested directly against this environment's `dyalogscript` and can be treated as settled rather than open:

- **`⎕NINFO` mtime is local time, not UTC.** `3 ⎕NINFO <file>` (the `⎕TS`-format mtime `FormatTimestamp` consumes) returned a value consistent with the container's local `date`/`⎕TS` output at time of creation, not shifted by the local UTC offset. Separately, `⎕NINFO`'s own manual entry documents property `13` as "Last modification time, as a **UTC** Dyalog Date Number" specifically, implying property `3` (no such qualifier) is local. `FormatTimestamp` should not append a trailing `Z`; if UTC timestamps are wanted instead, use property `13` (already confirmed present and usable) rather than converting property `3`.
- **`11 ⎕NINFO` reports `0`, not `¯1`, for a permission-denied directory on Linux.** Verified directly: a `chmod 000` directory owned by the running (non-root) user returns `11 ⎕NINFO` = `0`. The manual's `¯1`="unknown" case did not trigger on this platform. The planned `chmod 000` fixture for the `'not-readable'` `CheckFolder` outcome is realistic and needs no fallback handling for an ambiguous `¯1` result on Linux.

No plan change is required beyond removing the "confirm during RED phase" hedge on these two points and stating them as fact, since they no longer need re-confirmation once implementation starts.

Location: `FormatTimestamp` bullet; "Concrete coverage required" (`chmod 000` fixture); "Verification" section item 4.

### Note: verified APL semantics claims (no issues)

The following claims were independently checked against `dyalogscript` in this environment and hold exactly as the plan states, with no corrections needed:

- Bare namespace `⎕JSON` gives a JSON object; an enclosed one-element vector gives a one-element array; an enclosed empty vector gives `[]`; an enclosed multi-element vector gives an N-element array. Confirms the "`files`/`subfolders` must always be built as enclosed vectors" invariant.
- `⎕NREAD` conversion code `83` returns signed 8-bit values (e.g. byte `0xFF` reads back as `¯1`); `256|` recovers the unsigned value.
- `⎕NREAD` requesting more bytes than remain in the file returns the bytes actually available with no signalled error (a 5-byte file, asked for 100, returns exactly 5 bytes).
- `256⊥bytes` and `256⊥⌽bytes` correctly decode big-endian and little-endian byte sequences respectively.
- `(0 1 2 3)(⎕NINFO⍠1)folder,'/*'` in one call returns per-entry name/type/size/mtime for every child of `folder`, splittable by the type code (`1`=directory, `2`=regular file) into the files/subfolders split `ScanFolder` needs.
- `⎕NPARTS` on an absolute path returns the claimed three-element (directory, base name, extension-with-leading-dot) shape that `BuildFileMeta` relies on.

## Verification

- Tests: not run - no test files exist on this branch yet (design-only review, `apl-service/` is untracked, empty directory scaffolding).
- APL semantics: verified live via `dyalogscript`, see Findings above.

## Recommendation

Approve with minor changes. The overall architecture, request flow, configuration split, and EXIF parser design are sound, and the plan's more unusual APL semantics claims all check out against a live interpreter. The three Major findings (traversal boundary check, `ServeFile` scope enforcement, content-type mapping) should be resolved in the plan text before epics 2 and 3 are implemented, but none require rearchitecting the service.
