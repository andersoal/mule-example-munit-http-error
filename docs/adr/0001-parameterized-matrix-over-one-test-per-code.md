# The status map is driven by MUnit parameterizations, not one test per code

A parallel session built a green suite with one `<munit:test>` per status code (19 codes, 951 lines), which made the parameterization decision look contested by working code. We are still going with `<munit:parameterizations>` for the full ~63-code map, because at 63 codes the per-test shape costs roughly 3,000 lines of near-identical XML and puts the map's data inside test bodies, where it cannot be read as data or generated from.

## Considered options

- **One `<munit:test>` per code** — already proven green, no new mechanism to learn. Rejected at 63 codes: unmaintainable duplication, and correcting a mapping means editing XML rather than a data row.
- **`foreach` inside one test** — rejected earlier (see `research/munit-parameterization.md`): a failure names the test, not the code that failed.

## Consequences

- Parameterizations are scoped to `<munit:config>`, i.e. to the whole file, so every test in the matrix suite runs once per row. The status-less error tests (`HTTP:CONNECTIVITY`, `HTTP:TIMEOUT`, `HTTP:RETRY_EXHAUSTED`) must therefore live in a separate, non-parameterized suite.
- The built 19-test suite is kept as-is rather than deleted; it remains the longhand teaching example under `src/test/munit/option-a/examples/`, with the parameterized suite alongside it in `examples/parameterized/`.
- 1xx and 3xx codes are seeded as ordinary rows rather than assumed special. Any code the mock listener cannot actually produce is demoted out of the data and documented as a prose footnote — "not producible through a real listener" is itself a publishable finding.
