# Changelog

All notable changes to junit-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-10-12

**The implementation.** Every body the 0.0.1 interface published is
written — the 54 `todo()`s under `src/` are gone — and the four API
suites that were waiting on them are green, as are the doc examples in
all four modules (`novo test src/junitread.nv` and its three
neighbours). The published signatures, effect rows and refusals are
unchanged; the version moves because nothing was implemented before.

### Added

- `junitsuite.with_id` — the same suite with an `id`. `JunitSuite.id`
  was published as a field with no builder, so the reader could not
  record the index a `testsuite` element has in the document it came
  from, and `-1` — the value `suite` starts with — is what the writer
  reads as "the file did not say". The one addition to the surface.

### Changed

- The `novo` floor moves from `>= 0.9.0` to `>= 0.20.2`, the toolchain
  the implementation was built, tested and measured on.
- `stability` moves from `draft` to `experimental`: the bodies are in,
  and no program outside this package has used the surface yet.
- `tests/junitwrite_edge_tests.nv` is new: the counts on a written
  document against the cases and the root's counts against the sum of
  its suites, the four case shapes read back as the four outcomes, a
  failure's text through the escaping and back, and the exact text of a
  `timestamp` and a `time`. It also carries the half of each module the
  four API suites leave alone: the element the writer names when it
  refuses a character XML cannot carry, every field of a report written
  one refusal at a time, the attributes the reader fills from reports
  this writer never emits (`id`, `file`, `line`, the suite's
  `properties`, `system-out` and `system-err`), the reader's refusals
  for a case with two outcome elements and for a `time` that is not a
  number of seconds, and how many bytes a character takes, including
  sequences that encode no character at all.

### Known

- **A `??` with a literal fallback is avoided in `src/`.** The miss
  path of `<call returning ?T> ?? <literal>` hands its caller a
  `.rodata`-backed `Str` that `novo_dup_rc_v` then writes a refcount
  into, and the program dies with SIGSEGV — but only when the entry
  call sits in another package, which is why the reader's own tests
  never saw it and the doc-example harness did. Every site is written
  with `match` instead (`attr_text`, `child_text_or` and `line_of` in
  `junitread.nv`, `time_text` in `junitwrite.nv`), each naming
  `bugs/memory-perceus/the-miss-path-of-a-null-coalesce-over-a-package-call-with-a-literal-fallback-hands-dup-rc-v-a-rodata-str-segv.md`
  in the novo repository. The workaround comes out when that filing is
  fixed and the toolchain carrying the fix is verified here.
- **The buffer parameter copies.** `write_report`, `write_suite`,
  `write_open` and `write_close` take `out: [u8]` — a plain parameter,
  as the interface published it — and append with `list.concat`, so
  each call copies the bytes already in the buffer. A run appending
  ten thousand suites to one buffer is doing quadratic copying. A
  `var out: [u8]` would append in place, and the change is a signature
  change, so it is recorded here rather than made in a minor release.
- **`rerunFailure` and `flakyFailure` are still not read.** See the
  README's what-is-not-included list; the refusals are unchanged.
- **One line under `src/` is excused, not covered.** The `None` arm of
  `junitwrite.time_text` carries `// cov: skip — every caller reaches
  here through a `duration_fault` that refused a duration this cannot
  render, before any attribute is written`. The three call sites — the
  `testsuites` open tag, a suite's attributes and a case's — each ask
  `duration_fault` first, and that guard asks `format_time`, so the arm
  is reached only if the guard is removed. Every other line under
  `src/` is covered by the five suites.

This release requires novo 0.23.0 or later.

## 0.0.2 — 2026-09-28

The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- xml-nv: `^0.0.2` to `^0.1.0`.

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
