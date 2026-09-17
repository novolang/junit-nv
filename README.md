# junit-nv

JUnit XML is the file a test runner writes and a continuous integration
service reads. It has no normative specification: the format is the
schema Apache Ant emitted in 2002, extended by Maven Surefire and
copied by every runner since, and what a reader accepts is what those
writers produce. The closest thing to a reference is the
[Jenkins JUnit plugin's schema](https://github.com/jenkinsci/xunit-plugin/blob/master/src/main/resources/org/jenkinsci/plugins/xunit/types/model/xsd/junit-10.xsd),
which this package follows. It reads such a file into typed values and
writes one back.

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What a JUnit report is

A report is an XML document. Its root element is either `testsuites`,
holding one `testsuite` element per group of tests, or a bare
`testsuite` when a run produced only one. Both shapes are in the field
and both are read here.

A `testsuite` element holds one `testcase` element per test, and may
hold a `properties` element recording the environment, a `system-out`
element holding what the suite printed on standard output, and a
`system-err` element holding what it printed on standard error.

A `testcase` element is one test. A case with no child element passed.
A case holding a `failure` element made an assertion that did not hold.
A case holding an `error` element did not finish — it raised, crashed
or timed out. A case holding a `skipped` element did not run. Those
three elements each carry a `message` attribute, a `type` attribute
naming the assertion or exception class, and character data, which is
where a stack trace goes.

The attributes a reader uses are these.

| Element | Attribute | Meaning |
| --- | --- | --- |
| `testsuites` | `name`, `time` | The run's name, and how long all of it took |
| `testsuite` | `name` | What the group is called |
| `testsuite` | `package` | The module or directory it came from |
| `testsuite` | `timestamp` | When it started, ISO 8601 local time with no time zone |
| `testsuite` | `hostname` | The machine it ran on |
| `testsuite` | `id` | Its index within the document |
| `testsuite` | `time` | How long the group took, including setup |
| `testcase` | `name` | What the test is called |
| `testcase` | `classname` | The group the test belongs to |
| `testcase` | `time` | How long the test took |
| `testcase` | `file`, `line` | Where the test is written |

Both `testsuites` and `testsuite` also carry `tests`, `failures`,
`errors` and `skipped` attributes. They are counts of the cases below
them.

A `time` attribute is a decimal number of seconds. Three decimal places
is what most writers emit and six is the finest any writer uses.

## Install

```
novo pkg add junit-nv
```

## Example

```novo
use junitsuite
use junitwrite

fn main() [io]
    // One test that passed, taking 125 milliseconds.
    let ok = junitsuite.with_time(junitsuite.case("parses an empty document",
                                                  "xml_tests"), 125000)

    // One that failed, carrying the assertion's message and its text.
    let bad = junitsuite.with_outcome(junitsuite.case("rejects a stray tag",
                                                      "xml_tests"),
                                      JunitFailure("expected 2, got 3",
                                                   "assert_eq", "at line 9"))

    // The suite, and the report around it. The counts are not set
    // here: they are computed from the cases when the file is written.
    let suite = junitsuite.with_cases(junitsuite.suite("xml_tests"), [ok, bad])
    let report = junitsuite.with_run(junitsuite.report([suite]), "novo test", 1250000)

    // Write it into a buffer. The answer is an error when any text in
    // the report holds a character XML cannot carry.
    match junitwrite.to_str(report, junitwrite.options())
        Ok(document) => println(document)
        Err(fault)   => println("cannot write the report: ${fault.message()}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: junit-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `juniterror` | Every way a report is refused, and whether the fault is in the report being written or in one being read. |
| `junitsuite` | The report, the suite, the case and the outcome as values, the builders over them, and the counts computed from the cases. |
| `junitread` | Reading a document, or an already-parsed tree, into those values. |
| `junitwrite` | Writing them back into a buffer the caller owns, and the two ways of handling text XML cannot carry. |

## How to choose an entry point

**`junitread.read` takes the text of a document.** It parses and reads
in one call, which is what a tool opening a file wants.

**`junitread.read_document` takes an already-parsed tree.** Use it when
the document was parsed for another reason, such as reading an
attribute this package does not model. The parse is the expensive half.

**`junitwrite.to_str` produces the whole document as a string.** Use it
when the document is going to be held anyway.

**`junitwrite.write_report` appends to a buffer the caller owns.** Use
it for a large run, and reuse the buffer.

**`junitwrite.write_open`, `write_suite` and `write_close` write a
document a suite at a time.** Use them when a run is long enough that
holding every suite in memory is not wanted. The caller supplies the
counts on the open tag, because a streaming writer has not got them
until the run ends.

## The rules a user needs

1. **A case has exactly one outcome.** `JunitOutcome` has four arms and
   a `JunitCase` holds one of them, so a case is never both failed and
   skipped. A document whose `testcase` carries two outcome elements is
   refused as `JunitTwoOutcomes` rather than read by taking the first,
   which would report a failure as a skip whenever a writer emitted the
   skip first.
2. **A failure is not an error.** A `failure` element is an assertion
   that did not hold. An `error` element is a test that did not finish.
   The distinction is the one thing this format gets right, and a
   report that merges them cannot tell a broken test from a broken
   build.
3. **The counts are computed, never stored.** There is no `tests` field
   on a suite. `junitsuite.totals_of` counts the cases, and that is
   what the writer emits, so a document this package writes agrees with
   itself. A document that is read keeps its cases and discards its
   counts.
4. **A duration is an integer count of microseconds.** The `time`
   attribute is a decimal number of seconds, which is a float in the
   file, and a float does not round-trip. `junitread.parse_time`
   truncates digits beyond a microsecond rather than rounding, so a
   report read and written again holds the same numbers.
5. **A duration that does not fit the decimal places asked for is
   refused.** `junitwrite.format_time(1500, 3)` is `None`, and the
   writer answers `JunitTimeTooPrecise`. Rounding it silently would
   make a report say a test took no time.
6. **Text XML 1.0 cannot carry is refused, not escaped.** Section 2.2
   of XML 1.0 admits tab, newline, carriage return and U+0020 upwards,
   and admits no other control character. There is no escape for one:
   `&#1;` is as ill-formed as the byte. A `system-out` element carries
   whatever a test printed, and a test that printed a progress bar
   printed a carriage return or an escape byte.
7. **`junitwrite.sanitize_text` is the explicit way to lose a byte.**
   It drops what section 2.2 excludes and substitutes nothing. It is a
   separate call rather than an option on the writer, so that the
   difference between what a test printed and what the report says it
   printed appears in the caller's own source.
8. **A timestamp is text.** The `timestamp` attribute is an ISO 8601
   local time with no time zone. This package performs no input or
   output, so it can neither read a clock nor decide what local means.
   `junitsuite.is_timestamp_shaped` checks the shape and nothing else:
   `2026-02-30T00:00:00` is shaped and is not a day.
9. **A suite's own duration is not the sum of its cases.** The
   difference is setup and teardown. `junitsuite.suite_totals` gives
   the sum, and the suite's `time_us` field gives what the file said.
10. **A skip is not a failure.** `junitsuite.is_green` is true when
    there are no failures and no errors. A run of nothing but skips is
    green, which is what every CI service reports.
11. **Both root shapes read into one value.** A bare `testsuite` root
    becomes a report holding one suite. `junitwrite.bare_suite` writes
    that shape back, for a reader that accepts only it.
12. **A `testcase` with no `name` is refused.** It is the one attribute
    every reader in the field needs, and a case that cannot be named
    cannot be reported.
13. **Unknown attributes and unknown child elements are kept in the
    document and dropped from the values.** Reports in the field carry
    attributes no schema mentions. Refusing them would refuse most real
    files.

## What `novo test` should be able to emit

One `testsuite` per test source, named after the file. One `testcase`
per `@test` function, with `classname` set to the source file's stem
and `name` set to the function's name. A test that panicked is an
`error` whose `message` is the panic text; a failed assertion is a
`failure` whose `message` is the assertion and whose character data is
the `test.case` label it was under. Captured output goes in
`system-out`. The suite's `time` is the wall time of the file and each
case's `time` is the wall time of the function.

## What is not included

- **Reading or writing a file.** Every function here works on a string
  or on a buffer the caller owns. This package performs no input or
  output, so it never opens a file and never prints.
- **A clock.** A `timestamp` is the text the caller supplies. See rule
  8.
- **More than one outcome element on a case.** See rule 1.
- **The `rerunFailure` and `flakyFailure` elements.** They are Maven
  Surefire's extension for a retried test, they change what a count
  means, and no other writer emits them.
- **The xUnit.net, NUnit and TestNG schemas.** They are different
  formats that are also XML, and a reader that guessed between them
  would guess wrong on a file that uses one element name from each.
- **Validation against an XSD.** There is no normative schema to
  validate against. The refusals in rule 1, 4 and 12 are the checks
  this package makes.

## Related packages

- [xml-nv](https://novo-lang.org/packages/xml-nv) is the XML underneath,
  in both directions. This package depends on it.
- [tap-nv](https://novo-lang.org/packages/tap-nv) is the Test Anything
  Protocol, the other report format a CI reads. It is a line-oriented
  text format with no nesting and no durations.

## Tests

```bash
novo test tests/junitsuite_tests.nv   # the values, the counts and the outcomes
novo test tests/junitread_tests.nv    # what the reader tolerates and refuses
novo test tests/junitwrite_tests.nv   # what the writer refuses, and the durations
novo test tests/juniterror_tests.nv   # which side of the pipe a fault is on
```

The reference documents are the Jenkins JUnit plugin's schema for the
elements and their attributes, and XML 1.0 section 2.2 for the
characters a document may carry. The reference implementations are the
Rust crate `junit-report` and the Python package `junitparser`.

The suite asserts that a case has one outcome, that the counts come
from the cases, that a duration truncates rather than rounds, that a
control character is refused and named with its offset, that a bare
`testsuite` root reads into a report with one suite, and that a
`testcase` with no `name` is refused.

The tests compile today and fail at run, each on the
`not implemented: junit-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at a
time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `junitsuite.JunitReport`, `.JunitSuite`, `.JunitCase`, `.JunitOutcome`, `.JunitProperty`, `.JunitTotals` | the types are declared |
| `junitsuite.property`, `.property_value` | no |
| `junitsuite.case`, `.with_time`, `.with_outcome`, `.with_system_out`, `.with_system_err`, `.at_source`, `.with_case_properties` | no |
| `junitsuite.suite`, `.with_cases`, `.with_properties`, `.with_suite_time`, `.with_timestamp`, `.with_origin`, `.with_suite_output` | no |
| `junitsuite.report`, `.with_run` | no |
| `junitsuite.suite_totals`, `.totals_of`, `.is_green` | no |
| `junitsuite.is_passed`, `.is_failure`, `.is_errored`, `.is_skipped`, `.outcome_tag`, `.outcome_message`, `.is_timestamp_shaped` | no |
| `junitread.read`, `.read_document`, `.read_suite`, `.read_case` | no |
| `junitread.root_kind`, `.is_junit`, `.parse_time`, `.case_nodes`, `.suite_nodes` | no |
| `junitwrite.options`, `.compact`, `.bare_suite`, `.with_time_decimals`, `.without_output` | no |
| `junitwrite.write_report`, `.write_suite`, `.write_open`, `.write_close`, `.to_str` | no |
| `junitwrite.sanitize_text`, `.first_unwritable`, `.format_time` | no |
| `juniterror.is_writer_fault`, `.code`, `.offset`, `JunitFault.message` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
