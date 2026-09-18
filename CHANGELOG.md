# Changelog

All notable changes to junit-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `junitsuite` — the report, the suite, the case and the outcome as
  values. A case holds exactly one `JunitOutcome`, so it is never both
  failed and skipped. The `tests`, `failures`, `errors` and `skipped`
  attributes are not fields: `totals_of` computes them from the cases,
  so a document this package writes agrees with itself. A duration is
  an integer count of microseconds, because the `time` attribute is a
  float in the file and a float does not round-trip. A `timestamp` is
  the text it will carry, because a package with no effects can read no
  clock.
- `junitread` — both root shapes into one value, a bare `testsuite`
  included. Unknown attributes and unknown elements are tolerated, and
  four things are refused by name: a root that is neither element, a
  `testcase` with no `name`, a `time` attribute that is not a decimal
  number, and a case with two outcome elements.
- `junitwrite` — the document appended to a buffer the caller owns.
  Text holding a character XML 1.0 section 2.2 does not admit is
  refused as `JunitUnwritable` with its offset, rather than emitted as
  a document no reader will accept. `sanitize_text` is the explicit way
  to drop one, kept a separate call so the loss appears in the caller's
  source.
- `juniterror` — seven faults, with `is_writer_fault` dividing a bug in
  the program producing a report from a fault in a file somebody else
  wrote. The two have different responses, and a merge tool that cannot
  tell them apart stops on the wrong one.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  junit-nv.<module>.<fn>`.
- **xml-nv is itself an interface at 0.0.2.** This package cannot be
  implemented before xml-nv's parser and writer have bodies. Taking it
  rather than carrying a minimal writer is deliberate: escaping is
  where a minimal writer goes wrong, and `xmlwrite.write_text` already
  refuses what it cannot escape.
