# Changelog

Every published version, newest first.  This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.0.2 — 2026-09-16

README wording: the project vocabulary is gone; no change to the
interface.

## 0.0.1 — 2026-09-11

The **interface**, before anyone implements it.  Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`.  Adding this
package works and calling it panics.

- Six modules.  `fake` is the seed and the locale, and `fakename`,
  `fakeplace`, `fakelorem`, `fakeid` and `faketime` are people, places,
  text, identifiers and dates.
- **A fake IS a proptest-core-nv strategy**, not a wrapper around one
  and not a kind of its own.  The layer decided it — rand-nv is `host`
  and a `core` package may not reach it, while proptest-core-nv's
  `Tape` is `core` and seeded — and the consequence is that every
  combinator in that package works on a fixture, and a property test
  and a fixture draw from one source.  This package therefore ships no
  `choice`: `strategy.sample_of` is already that.
- **A fake does shrink**, which is the answer to the question the grid's
  row asked.  Shrinking there is arithmetic on the choice sequence and
  not on the value, so a name drawn as a table index shrinks towards the
  first name in the table.  What it does not do is shrink meaningfully
  in its domain, and the README says so.
- **The locale is a value on every generator**, never a global and never
  a default — and it changes the SHAPE, not only the words: a four-digit
  postcode against a five, a street number after the name against
  before, no area code against one, no region against a state.  `en` and
  `da`, because two is the smallest number that proves the seam.
- **A record's fields agree.**  `fakename.person` and
  `fakeplace.address` draw together, because three independent draws
  produce data that looks wrong to a reader and hides the class of bug
  where a system mismatches a name against an email.
- **Nothing generated can reach anybody**: RFC 2606's reserved domains,
  the 555 telephone range, RFC 5737's documentation addresses, and
  locally administered MAC addresses.  `fakename.email_at` is the
  explicit escape, at the call site where a reviewer can see it.
- **A UUID comes back as text**, because uuid-nv is `host` and a `core`
  package may not name its `Uuid`.  The canonical thirty-six characters
  with the version and variant nibbles set, which `uuid.parse` accepts.
  They are also NOT unique, deliberately: the same seed gives the same
  identifier, which is the point for a fixture and a disaster if
  mistaken for `uuid.v4`.
- **There is no "now".**  `faketime.before` and `faketime.after` take
  the date they are relative to, so a fixture does not stop being
  reproducible when the clock moves — which is what happens to every
  fake library whose `past_date()` means "relative to this instant".
- **Filler text is not Latin** unless it is asked to be: the locale's
  own words, so a Danish fixture contains `æ ø å` and a field that
  truncates by byte breaks in a test rather than in front of somebody.
- `tests/` holds thirteen API tests, every one red.  They assert the
  PROMISE rather than the data — determinism, independence, shape, and
  the locale seam — because a fixture library's output is arbitrary by
  construction and pinning a literal would pin a table nobody agreed to
  keep.
