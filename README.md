# truthpatch — Fix the bot's reply before the visitor reads it

**Paid tool: $99 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/truthpatch**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

An output guard for customer-facing chatbots: it snaps invented emails and phone numbers back to the ones you published, rewrites dead links, replaces raw search dumps, and corrects wrong weekday, date and open/closed claims from your real hours. Pure Python, no model call.

## Get it

Buy truthpatch at https://dominares.org/tools/truthpatch: one payment of $99, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python package and the truthpatch command, with the business-hours module, an example facts file and the full test suite.

Requirements: Python 3.10 or newer; standard library only (the tz extra installs tzdata for systems without a time zone database).

This page documents version 1.0.2. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

`examples/reply.txt` is a model reply with six problems in it. `truthpatch fix` with `examples/facts.json` as the facts and a clock of Saturday 2026-09-26 10:44 (Pacific) prints it repaired:

```text
We're closed right now. We open again on Monday, September 28 at 9:00 AM. Email jordan@acme.example or call (555) 010-1000.
Sign up at https://acme.example/signup and get the form at https://acme.example/downloads
```

The reply said "Yes, we're open today until 6pm!" on a Saturday morning, mailed `seniorjordan@acme.example`
(the real handle is `jordan`), gave a retired phone number, linked a dead tracking host and an invented
download URL, and left `**bold**` markers in plain text. Add `--json` to see which repairs fired.

### What it does

You give it a `Facts` object (or a JSON file): what you have published, and your hours. It runs these
repairs, in this order, on every reply. Each one needs only the facts it uses, and is off without them.

| Repair | What it fixes | Facts it needs |
|---|---|---|
| `dump` | A reply that is a raw retrieval dump is replaced by a safe line, and nothing else runs on it. It is a dump when it opens with a retrieval header (`Knowledge results for`, `Notes results for`, `Website results for`, `Indexed website pages`) or a `[relevance: 3/9]` chunk; carries a marker inside a ```` ```json ```` fence; holds six or more `[relevance: ...]` chunks beside the word Knowledge, Notes or EMAIL THREAD; or carries any marker (those, `"search_knowledge`, `"search_notes`, `EMAIL THREAD:`, `<html><head`) and is over 4000 characters long. | none (`main_email` is named in the line when set) |
| `html` | `<a href>` anchors become text plus URL; stray tags are removed. | none |
| `links` | Links on a dead host (and its numbered tracking siblings) become the live link; each `url_rules` pattern is rewritten. | `dead_hosts`, `url_rules` |
| `markdown` | `**bold**` markers are dropped, the words stay. | none |
| `emails` | An address on one of your domains that you did not publish snaps to the single published handle it ends with (`seniorjordan` to `jordan`), else to `main_email`, else to the words "our team". Published addresses are left exactly as written. | `emails` |
| `phones` | A number you registered but retired becomes your one number. A number you never registered, or state on purpose, is never touched. | `phone`, `retired_phones` |
| `weekday` | "Today is Friday" on a Saturday is corrected. A sentence that drew open/closed from the wrong day is replaced by the real status line. | a clock: `hours` or `timezone` (and `hours` for the status line) |
| `day-and-hours` | A wrong weekday for tomorrow, the day after tomorrow or a written date ("Sunday, September 28"); a false "we're open right now" while the hours say closed; "I don't have our hours" while they are set. | a clock: `hours` or `timezone`; `hours` for the open/closed parts |

It also ships, as plain functions you can call on their own:

- **The hours module** (`truthpatch.hours`): turns a business-hours record into facts a bot can quote. Open now, next opening,
  today's and tomorrow's hours, a dated 7-day list, a one-line "today" for a prompt, and a follow-up promise that
  expires ("today, before 6:00 PM"). It handles special days, hours past midnight, split days and daylight-saving changes.
- **A drafting-assistant guard** (`guard_draft_text`): a dump becomes an explicit error, any other draft is returned unchanged.
- **A greeting check** (`strip_bleeding_salutation_name`): removes a first name in a drafted greeting that is not in the email
  being answered ("Hi Dana," to someone who signed "Priya").

No model call, no network, no dependencies beyond the Python standard library.

### Use it in code

```python
from datetime import datetime, timezone
from truthpatch import Facts, Guard

facts = Facts(
    emails=["jordan@acme.example"], main_email="desk@acme.example",
    phone="(555) 010-1000", retired_phones=["(555) 010-1099"],
    hours={"timezone": "America/Los_Angeles",
           "weekly": {"mon": [["09:00", "18:00"]], "tue": [["09:00", "18:00"]]}},
)
guard = Guard(facts)

result = guard.fix("Email seniorjordan@acme.example", now=datetime.now(timezone.utc))
result.text      # "Email jordan@acme.example"
result.repairs   # ("emails",)
result.changed   # True
```

`guard.sanitize(text)` returns just the text. Running it again on its own output changes nothing, and the clock and hours
are read only for a reply that names a day or an open/closed claim.

With no `hours` and no `timezone` the guard has no clock, so it judges no day or date at all: a guessed zone would
"correct" a right answer on an evening in another zone. Give it one to turn the day repairs on.

#### Streaming

Guard **the whole reply received so far, after every chunk**, never a chunk by itself. A claim, an address or a link
that spans two chunks is only visible in the joined text (`"Yes, we're "` and `"open right now!"` are both fine alone).
The safe text of a longer prefix can also differ from what is already on screen, so send the new tail when it extends
what was shown, and replace the shown text when it does not:

```python
buffer, shown = "", ""
for chunk in model_stream:
    buffer += chunk
    safe = guard.sanitize(buffer)
    if safe.startswith(shown):
        send(safe[len(shown):])          # the new tail
    else:
        replace_shown_text(safe)         # the guard rewrote something already shown
    shown = safe
```

A reply cut off right after "we're open" is not judged until the next words say whether it is "open to new clients" or
"open right now".

### The facts file

```json
{
  "emails": ["jordan@acme.example"],
  "main_email": "desk@acme.example",
  "phone": "(555) 010-1000",
  "retired_phones": ["(555) 010-1099"],
  "stated_phones": [],
  "dead_hosts": {"ml.oldhost.test": "https://acme.example/signup"},
  "url_rules": [{"pattern": "https?://cdn\\.files\\.test/\\S+", "replacement": "https://acme.example/downloads"}],
  "hours": {"timezone": "America/Los_Angeles", "weekly": {"mon": [["09:00", "18:00"]]}, "exceptions": []},
  "timezone": "UTC"
}
```

| Key | Meaning |
|---|---|
| `emails` | Every address you have published. Replies may carry these verbatim. |
| `main_email` | Where an unfixable address is sent. Counts as published. |
| `email_domains` | Optional: guard these domains instead of the ones your `emails` are on. |
| `phone`, `retired_phones` | The one number to give, and the others that must read as it. |
| `stated_phones`, `stated_texts` | Numbers (or text containing them) that you state on purpose and must not be snapped. |
| `dead_hosts` | `{dead host: live link}`. A tracking-style host (`ml.example.test`) also covers `ml2.example.test`, `url123.c.example.test`. Any other host is an exact match: `www.` of it, and hosts that merely start with it (`oldhost.test.other.example`), survive. Punctuation right after a rewritten link (`).` or `,`) stays. |
| `url_rules` | `{pattern, replacement}` pairs: a case-insensitive regular expression and the literal link to put in its place. |
| `hours` | `timezone`, `weekly` (`mon`..`sun`, each a list of `["09:00", "18:00"]`; `"24:00"` is allowed as a close) and `exceptions` (`{"date": "2026-11-26", "intervals": [], "note": "Holiday"}`). |
| `timezone` | The clock when no hours are set. With hours, their own zone is the clock. With neither, no day or date is judged. |
| `stand_in` | The words used when an address cannot be fixed and there is no `main_email`. Default "our team". |

A mistake is an error in plain words, not a silent no-op: an unknown key, a bad time zone, a time that is not `HH:MM`,
two overlapping sets of hours, a regular expression that does not compile.

### Command line

```text
truthpatch fix [FILE | - | --text TEXT] [--facts F] [--now ISO] [--json] [--check] [--keep-html] [--keep-markdown]
truthpatch dump [FILE | - | --text TEXT] [--json]
truthpatch salutation --draft FILE --source FILE [--own-name NAME ...] [--check] [--json]
truthpatch hours {check|status|today|week|prompt|followup} HOURS_FILE [--now ISO] [--label L] [--json]
```

- `fix --check` prints nothing but a verdict and exits **1** if the reply would change, **0** if it is clean. Use it to grade
  a transcript in CI.
- `dump` exits 1 when the reply is a raw retrieval dump.
- `hours` takes an hours file, or a facts file (it reads the `hours` key).
- `--now` is an ISO time such as `2026-09-26T10:44:00-07:00`; with no offset it means UTC. The default is the real clock.
- Exit 2 is a usage or file error, with the reason on stderr.

### Business hours

```python
from datetime import datetime, timezone
from truthpatch import hours

record = hours.normalize({"timezone": "America/Los_Angeles",
                          "weekly": {"mon": [["09:00", "18:00"]], "tue": [["09:00", "18:00"]]}})
now = datetime(2026, 9, 26, 17, 44, tzinfo=timezone.utc)      # Saturday 10:44 AM in Los Angeles
hours.status_sentence(record, now)      # "We're closed right now. We open again on Monday, September 28 at 9:00 AM."
hours.today_line(record, now, tz_name="UTC")
hours.render_prompt_block(record, now, label="Business hours")   # the dated 7-day block for a system prompt
hours.followup_when(record, now)        # "on Monday, September 28"
```

Computing the days in code and handing the bot a finished line removes the failure at the source: a model asked
"will you be open the day after tomorrow?" counts weekdays badly, and the usual fix is to stop letting it count.

### Limits

- **A reply is a dump only by the rule in the table.** A short email thread, a short page of HTML, or a JSON blob with no
  ```` ```json ```` fence and under 4000 characters passes through. Change the markers with `Guard(markers=...)`.
- **No clock, no day repairs.** Without `hours` or `timezone` the weekday, date and open/closed repairs do not run; the
  command line with no `--facts` makes no day claims either.
- **English only.** The day and open/closed patterns, the greeting check and the dump markers are English.
- **Phone numbers are 10-digit US/Canada shapes** (an optional leading 1).
- **It only judges what it can verify.** It knows the weekday of any date and what your hours record says. It does not know
  your prices, your policies or whether a staff member exists; a wrong claim about those passes through.
- **Open/closed is one-sided on purpose.** A false "we're open right now" is corrected; "we're closed today" while the weekly
  hours say open is left as written, because a note (a storm, a staff day) can close a day that your hours record does not know about.
  Only claims made by "we" or the office/desk/store/team are judged; "online ordering is open 24/7" is not.
- **The status line comes from the weekly hours and special days you give it.** A record with every day closed and no special days
  counts as "no hours set", and then no open/closed claim is judged (the day arithmetic still is, in that record's time zone).
- **A written date with no year** is checked only within two weeks of today. Farther out, the year is a guess, so the sentence is left alone.
- **Guarded domains are matched whole.** If you publish an address on a shared mail domain (a free webmail address), every address on that domain
  in a reply is checked against your list. Set `email_domains` to guard only your own.
- **Links are matched by host or by your patterns.** It does not browse: a dead link on a host you did not list is not found.
- **Markup cleanup is blunt.** Anything that looks like an HTML tag (`<b>`, `</p>`, `<!-- -->`) is removed.
- **Hours:** up to 3 sets of hours a day and 60 special days; the next opening is looked for 14 days ahead.
- **Needs a time zone database.** Python's `zoneinfo` uses the system's; on Windows or a slim container, `pip install '.[tz]'`.

Python 3.10 or newer, standard library only.

---

Docs mirror of the private repository Dominares-Tools/truthpatch; updated with each release.
