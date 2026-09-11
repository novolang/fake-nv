# fake-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

**Fixture data that is the same every run.**  Names, addresses, emails,
telephone numbers, sentences, dates, numbers in a range and identifiers
— in English and Danish, from a seed a test can print and a person can
paste back.

```novo
use fake
use fakename

fn main() [io]
    println(fake.seed_line(42, FakeEn))     // fake seed: 42 (en)
    for p in fake.many(fakename.person(FakeEn), 42, 3)
        println("${p.full_name} <${p.email}>")
```

Six modules.

| surface | module | reach for it when |
| --- | --- | --- |
| the **seed and the locale** | `fake` | you are turning a generator into values |
| **people** | `fakename` | names, usernames, emails, a whole person |
| **places** | `fakeplace` | streets, cities, postcodes, telephone numbers |
| **text** | `fakelorem` | words, sentences, paragraphs, filler of a size |
| **identifiers and numbers** | `fakeid` | UUIDs, digits, hex, amounts, probabilities |
| **dates** | `faketime` | dates, times and durations over calendar-nv |

## Adding it, and checking it

```bash
novo pkg add fake-nv            # into your novo.toml
novo pkg build                  # type- and effect-check the package
novo test --isolate tests/fake_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: fake-nv.<module>.<fn>`.  They
turn green one at a time as bodies land.

## Why a fake is a strategy

`fakename.first_name(FakeEn)` answers a **`PropStrategy<Str>`** —
proptest-core-nv's own type, not a wrapper around it and not something
that converts into one.  Every combinator in that package works on it:

```novo norun:pseudo
let team = strategy.list_of(fakename.person(FakeDa), 3, 8)
let maybe_nickname = strategy.option_of(fakename.first_name(FakeDa))
```

Three reasons, and the first is not a preference.

**1. The layer decided it.**  This package is `core`, and a `core`
package may depend only on `core` packages.  rand-nv is `host` — it
declares `[fs, rand]`, because half of it is the operating system's
CSPRNG — so the plan's other option, "a generator that is a function of
a rand-nv state", is not reachable from this layer at all.
proptest-core-nv is `core`, and its `Tape` is a seeded splitmix64 with
no effect anywhere in it.

**2. A second generator type would be a second ecosystem.**
`strategy.list_of`, `strategy.option_of`, `strategy.map`,
`strategy.one_of`, `strategy.pair_of` are exactly the combinators a
fixture wants — twenty of these, optionally absent, mapped into a
caller's own struct.  Reimplementing them here would be a hundred
functions doing the same thing under different names, and a reader
wondering which of the two `choice` functions they should use.  This
package therefore has **no** `choice` and no `weighted_choice`:
`strategy.sample_of` and `strategy.weighted_of` are those, already.

**3. One transcript.**  A property test that used a fake reproduces from
the tape, and the tape does not know or care which of its draws were
"fake" and which were "property".  A suite that wants both — a hundred
plausible customers, *and* the property that the importer never loses
one — writes one generator and uses it twice.

### Does a fake shrink?

The question the grid's row asks, and the answer is **yes**, which is
not the obvious one.

Shrinking in proptest-core-nv is arithmetic on the **choice sequence**
and not on the value, so anything that draws through the tape shrinks
for free: a name drawn as an index into a table shrinks towards index 0,
which is the first name in the table.

What a fake does not do is shrink *meaningfully in its domain* —
"Zdenka Sørensen" shrinking to "Aaron" is arithmetic, not simplification
a person would have chosen.  In practice that is fine and occasionally
better than fine: a counterexample reported with the first name in the
table is a counterexample whose name is visibly not the point, which is
usually exactly true.  A test that wants a specific value uses
`strategy.just`.

## The seed is the whole contract

Every generator is a pure function of the tape it is handed, so
`fake.one(g, 42)` is the same value on every machine, every run, and
every toolchain that agrees on `tape.split_step`.

`fake.seed_line(seed, locale)` is the line a failing test should print
and `fake.parse_seed_line` is what reads it back.  It exists rather than
being left to each caller because the value of a reproducible fixture is
entirely in somebody actually printing the seed — and a library that
only made it *possible* would find that most suites did not.

`fake.many(g, seed, n)` draws from **one** tape in sequence, so the
values differ from each other.  `n` calls to `fake.one` with one seed
would give `n` copies of one value, which is the mistake this signature
exists to prevent.

## The locale is a value, and it changes the shape

Two locales, `en` and `da`, and the pair is deliberate rather than a
start: one locale is a library that cannot be tested for
locale-dependence at all, and two is the smallest number that proves the
seam.

It is a **parameter on every generator**, never a global and never a
default.  A global locale is a library where one test changes another's
data; a default locale is a library that makes English the unmarked
case.

And what it changes is the *shape*, not only the words:

```
en   1234 Elm Street          da   Elmegade 12
     Springfield, IL 62704         2200 København N
     +1 555 014 0198               +45 32 14 01 98
```

The number goes before the street name in one and after it in the other.
The postcode goes after the city in one and before it in the other.  One
has a state and the other does not; one postcode is five digits and the
other four; one telephone number has an area code and the other does not
— because Denmark has none.  A package that produced the English shape
with Danish words would produce data a Danish reader can see is wrong,
and the locale would be decoration.

So `fakeplace.address` answers a **struct of parts** and
`fakeplace.formatted` renders it the locale's way, and `FakeDa` is where
`æ`, `ø` and `å` get into a fixture — which is how a field that
truncates by byte rather than by character breaks in a test rather than
in front of somebody.

## Records whose fields agree

`fakename.person` and `fakeplace.address` draw their fields **together**.

Three independent draws give `Astrid`, `Hansen` and
`mwilliams@example.net` — data that looks wrong to anybody who reads it
and, worse, is wrong in a way that hides bugs: an importer that silently
mismatched a name against an email would pass a suite whose fixtures
never matched either.

The single-field generators stay, because most tests want one field and
should not pay for six.  What they do not do is pretend to be consistent
with each other.

## Nothing generated here can reach anybody

- **Emails** are at RFC 2606's reserved domains — `example.com`,
  `example.net`, `example.org`.  `fakename.email_at` is the explicit
  escape for a caller that has decided it needs its own domain, and it
  is explicit *at the call site* where a reviewer can see it.
- **Telephone numbers** in `FakeEn` use the 555 range that North
  American numbering reserves for fiction.
- **IPv4 addresses** are in RFC 5737's three documentation ranges.
- **MAC addresses** are locally administered, so none was assigned to a
  real card.

A fixture generator that produced a real domain is one that will one day
be in a test that actually sends, and the person on the other end did
not ask for it.

## Two things the layer costs, named rather than hidden

**A UUID comes back as text.**  uuid-nv owns the `Uuid` type and it is
`host`, because it depends on rand-nv.  A `core` package cannot name
that type, so `fakeid.uuid_v4` answers the canonical lowercase
thirty-six characters — with the version and variant nibbles set —
which is what a fixture writes into a column or a JSON document anyway.
A caller that wants the type calls `uuid.parse`, which is one call and
no dependency here.

The alternative was fake-nv at `host`, which would put every fixture
generator out of reach of every `core` package that wanted one.  That is
the worse trade, and this is the cost of the better one.

**And these UUIDs are not unique.**  Every value is a pure function of
the tape, so the same seed gives the same UUID — which is the whole
point for a fixture and a disaster if anybody mistook it for `uuid.v4`.
`fakeid` fills a column; uuid-nv mints an identifier.

## There is no "now" in this package

Every other fake library has `past_date()` and `future_date()` meaning
"relative to this instant", and every one of them produces fixtures that
stop being reproducible the moment the clock moves: a test that passed
in December because its fixture was "a date in the last year" fails in
January for a reason that has nothing to do with the code.

A `core` package has no clock to consult even if it wanted one, so the
question never arises: `faketime.before` and `faketime.after` take the
date they are relative to, and a caller that means "before today" passes
today's date from the host that has one.  The fixture is then
reproducible from a seed **and** a date, both of which a failure report
can carry.

The type is calendar-nv's `CivilDate`, not a string and not an epoch
day, because that is the type this grid already means when it says "a
date".

## Filler text is not Latin

`lorem ipsum` is the wrong default for what filler text is actually
*for* in a test — exercising the widths, the wrapping, the truncation
and the encoding of a text field.  Latin is pure ASCII, its word lengths
are unlike either shipped locale's, and it never contains the characters
that break things.

So `fakelorem` draws from the locale's own word list, and
`fakelorem.lorem` is still there for the one case Latin is right for:
filler a reviewer should immediately recognise as placeholder.

## The tables are small, and the size is public

A few hundred names per locale, not a census.  `fakename.first_name_count`
and `fakename.last_name_count` are public because a fixture of a
thousand rows with a unique-name constraint **will** collide — and
finding that out from a function is better than finding it out from a
flaky test.  A caller that needs a long tail composes a name with
`fakeid.digits`.

## Naming

Every public type in this package starts `Fake`, and every module file
starts `fake`.  Struct and enum identity is keyed by **name** across a
whole assembly, dependencies included, so two packages that both declare
`Person` or `Address` cannot be used by one program — and those are
names a great many packages will want.

## The reference implementation

Rust's [fake](https://github.com/cksac/fake-rs) and Python's
[Faker](https://faker.readthedocs.io), whose vocabulary this borrows.
Where it differs it differs on purpose, and the README says where: no
clock, no global locale, a UUID as text, and a generator that is a
property-testing strategy rather than a kind of its own.

Apache-2.0.
