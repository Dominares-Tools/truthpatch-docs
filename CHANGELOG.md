# Changelog

## 1.0.2 (2026-10-04)

README only: it opens with the sample reply and its repaired output, and leaves the install steps to the page you bought it from. No code change.

## 1.0.1 (2026-10-04)

Fixes from review.

- **No clock, no day repairs.** `Facts.timezone` now defaults to nothing instead of UTC. With neither `hours` nor
  `timezone`, the weekday, date and open/closed repairs are off, so a right "Happy Friday!" on a Friday evening in a US
  zone is no longer rewritten to Saturday. `Facts.clock_timezone` can be `None`.
- A dead host now matches only that host: `oldhost.test.other.example` and `oldhost.testing.example` are left alone, and
  punctuation after a rewritten link (`).`, `,`) stays. A bare `https://oldhost.test` with no path is now rewritten too.
- A `dead_hosts` entry with no live link (`null`, empty, not a string) is an error in words, not the text "None".
- The README states the real dump rule and its limits, and a Streaming section: guard the whole reply so far, after every chunk.
- Error text for a bad time no longer mentions a time picker; the hours prompt block no longer mentions notes.
- The brand-free test scans only files git tracks or would track, and skips binary files, so a local `.coverage` file cannot fail it.
- Removed a stray greeting word and a misaligned line.

## 1.0.0 (2026-10-04)

First release.

- `Guard` runs eight repairs over a reply, in order, and reports which fired: raw retrieval dump, HTML anchors and tags,
  dead or invented links, `**bold**` markers, invented emails, retired phone numbers, a wrong weekday for today, and wrong
  day or open/closed claims.
- `Facts` (or a JSON file) holds what the repairs check against; a mistake in it is an error in words.
- `truthpatch.hours`: business hours as facts. Status, next opening, dated week, a one-line "today", a prompt block, an
  expiring follow-up promise, and reply-window arithmetic.
- `guard_draft_text` for drafting assistants, and a greeting check that removes a name that is not in the email being answered.
- Command line: `fix`, `dump`, `salutation`, `hours`.
