# 153 - Format tool details only when expanded

## Background

Commit `3812271` made complete-round opening expose more history work. The
one-round policy in [152](152-open-single-round.md) reduces the range but does
not make long individual rounds cheap. Tool call/result renderers still built
complete terminal, numbered-read, diff and generic bodies before handing them
to the fragment UI, which then omitted those bodies when folded.

Inspection of installed dsh Web 0.2.1-rc.2 showed the useful boundary: detailed
tool components are mounted when expanded, although summaries and some data
models are prepared earlier. Its ordinary opening window is also bounded; it
does not repeatedly fetch pages just to complete the requested turn count.
This change addresses the rendering boundary, not the separate loading policy.

## Decision

Allow a fragment body to be a zero-argument text formatter as well as a string.
The renderer captures the card facts with `apply-partially`; it does not retain
an event handler's whole lexical environment or read later mutable tool state.
Pending tool cards and ordinary settled result cards use this boundary.
Settled question/goal cards and immediate state transitions remain unchanged.

The UI formats a minimal card's body on expansion. It retains one cached
string-or-nil result per fragment, keyed by formatter identity and body width.
A new result supplies a new formatter. Width changes invalidate cached layout;
repeated folds and unrelated buffer edits do not. Bordered cards still need a
body for their hidden-line count, and non-foldable cards render immediately.

## Why

This avoids doing invisible work rather than moving it to an idle timer or
raising GC thresholds. The fragment owner already knows fold state, layout,
replacement and rollback, so a separate history renderer or timer queue would
duplicate those responsibilities. Only the selected tool body is delayed;
protocol decoding, summaries and live execution state stay immediate.

Cached results are published in the new fragment snapshot only after a
successful atomic write. An exception must leave the old card and cache intact
so expansion can be retried. Function identity is preserved rather than
copying a lexical closure. Snapshot updates strip internal fold/cache fields
before carrying the existing cache forward, avoiding duplicate retained bodies
when a caller passes a previously published state back to the updater.

## Consequence

Users see the same summaries and expanded bodies. Already-expanded tools still
refresh as results arrive, including pending-to-settled transitions. Visible
transcript copying is unchanged. Tests cover deferred calls, reuse, nil results,
source/width invalidation, state round-trips, layout styles and rollback after
both formatter and insertion failures. Existing tool-output and history/live
isolation tests run through the real expansion path.

Synthetic batch measurement on Emacs 31.1.50, source-loaded with `-Q`, default
800000-byte GC threshold, three samples and an explicit GC before each: 100
folded Bash tools, each receiving a 1000-line result. Fixture construction is
outside timing; per-sample GC during rendering is included.

| Measurement | Before | After |
| --- | ---: | ---: |
| Median call/result rendering | 144.2 ms | 36.0 ms |
| Terminal-body formatter calls | 200 | 0 |
| GC cycles during each sample | 2 | 0 |
| Final visible buffer size | 2624 chars | 2624 chars |

These are synthetic rendering measurements, not full session-open or GUI
latency numbers. This work is uncommitted; the earlier commit identifies the
loading regression, not the implementation of this optimization.

## Known limitations

Raw results are still downloaded, decoded, retained and used for summaries.
A first expansion pays its formatting cost synchronously. An expanded card
retains both source data and one formatted result until replacement or buffer
cleanup. Ordinary assistant Markdown and complete-round publication remain
synchronous; this is not viewport virtualization. Width is reevaluated on
update/fold, not by a new resize hook. Formatter inputs must be immutable and
formatters must not mutate the transcript or session state.
