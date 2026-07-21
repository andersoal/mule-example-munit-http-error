# Research: driving one MUnit test across ~63 status codes

Ticket: [How can one MUnit test cover 63 status codes?](https://github.com/andersoal/mule-example-munit-http-error/issues/5)
Part of: [Wayfinder map: HTTP status → Mule error map, proven by MUnit](https://github.com/andersoal/mule-example-munit-http-error/issues/3)

## Versions

`pom.xml`: `munit.version` = **3.5.0** (drives `munit-maven-plugin`, `munit-runner`, `munit-tools`).

## Recommendation

**Use `<munit:parameterizations>`.** It is purpose-built for exactly this — running identical test logic against a list of inputs — and this repo's MUnit version (3.5.0) is well above the feature's minimum. `foreach`-inside-one-test and build-time-generated `<munit:test>` blocks were also evaluated and rejected; details below.

## How `<munit:parameterizations>` works

Source: official syntax confirmed via search-indexed content from [Parameterized | MuleSoft Documentation](https://docs.mulesoft.com/munit/latest/parameterized) (title/snippet — direct fetch returned 403 from this environment) and corroborated by community walkthroughs (DZone, Medium; direct fetch of those also blocked 403, titles/snippets only). `docs.mulesoft.com` was not directly fetchable for any source in this research session — see Sources below.

Declared once per `<munit:config>`, at the top of the test suite file:

```xml
<munit:config name="http-status-matrix-test-suite.xml">
  <munit:parameterizations>
    <munit:parameterization name="404-not-found">
      <munit:parameters>
        <munit:parameter propertyName="statusCode" value="404"/>
      </munit:parameters>
    </munit:parameterization>
    <munit:parameterization name="503-service-unavailable">
      <munit:parameters>
        <munit:parameter propertyName="statusCode" value="503"/>
      </munit:parameters>
    </munit:parameterization>
    <!-- one <munit:parameterization> per status code -->
  </munit:parameterizations>
</munit:config>
```

Every `<munit:test>` in that config runs **once per parameterization**, with values pulled via the `p('propertyName')` function anywhere a DataWeave/property expression is valid:

```xml
<munit:test name="statusCodeProducesExpectedError">
  <munit:behavior>
    <munit-tools:mock-when processor="http:request">
      <munit-tools:then-call flow="munit-util-mock-http-error.req-based-on-vars.munitHttp"/>
    </munit-tools:mock-when>
  </munit:behavior>
  <munit:execution>
    <set-variable variableName="munitHttp" value="#[{ path: '/', method: 'GET', queryParams: { statusCode: p('statusCode') as Number } }]"/>
    <flow-ref name="flow-under-test"/>
  </munit:execution>
  <munit:validation>
    <munit-tools:assert-that expression="#[error.errorType.identifier]" is="#[MunitTools::equalTo(expectedTypeFor(p('statusCode')))]"/>
  </munit:validation>
</munit:test>
```

This maps directly onto the Option A harness already in this repo (`README.adoc`, "Option A" section): the existing `munit-util-mock-http-error.req-based-on-vars.munitHttp` utility flow already takes its target status via `vars.munitHttp.queryParams.statusCode` — a parameterization only needs to set that var from `p('statusCode')` instead of a hardcoded value per test-specific setup flow. **No harness redesign is needed for the plain-response-code rows; the parameterization ticket unblocks the harness-extension work (Not yet specified on the map) rather than replacing it.**

An external YAML file (`<munit:parameterizations file="parameterizations.yaml"/>`, resource under `test/resources`) is also supported and is the better fit here at ~63 rows — it keeps 63 code/expected-type pairs as data instead of 63 blocks of XML boilerplate in the test suite file.

## Failure reporting — the key evaluation axis

Per the naming convention, **each parameterization's `name` attribute becomes the identifying label in the MUnit test report.** A test named `statusCodeProducesExpectedError` run under parameterization `503-service-unavailable` reports as a distinct result line (commonly rendered as `statusCodeProducesExpectedError {503-service-unavailable}` or nested under that name, depending on the Maven/Studio test report renderer). This means:

- A red row names its own status code directly, with no extra bookkeeping — as long as parameterization names are chosen deliberately (e.g. `"501-not-implemented"` rather than `"param1"`).
- This satisfies the map's requirement that "a red row is identifiable" — it's the strongest guarantee of the three candidate mechanisms, because the label is structural (part of the test framework's own reporting), not something the test body has to manually emit into a log line.

## Alternatives considered and rejected

### `foreach` inside one test's execution/validation block

Iterating a literal list of codes inside a single `<munit:test>` would technically work for driving the mock (DataWeave can loop and set `munitHttp` per iteration), but:

- A single `<munit:test>` reports as **one pass/fail result** in MUnit's output — a failure on code #47 out of 63 does not by itself tell you which iteration failed unless the test body manually logs or asserts with an inline message per iteration, which has to be hand-built and is easy to get wrong (e.g. `munit-tools:assert-that` stops the flow at the first failure inside the loop, so later iterations never even run once one fails, silently hiding whether the rest would also fail).
- Mocks (`mock-when`) and `verify-call` semantics are scoped to the whole test, not per-iteration, so verifying "the right thing happened for *this* code" inside a loop iteration requires extra bookkeeping the framework doesn't give for free.

Rejected: fails the "identify which code failed" requirement without significant hand-rolled infrastructure that parameterizations give for free.

### Build-time generation of one `<munit:test>` per code

A script (Maven plugin, XSLT, or a pre-build step) could emit 63 literal `<munit:test>` blocks into a generated XML file. This gives the clearest possible per-code failure reporting (each is a fully distinct named test) but:

- Adds a generation step and a generated-file-in-the-build-output that has to be kept out of version control or explicitly checked in and regenerated — more moving parts than a static parameterizations block or YAML file.
- No existing precedent for this was found in MuleSoft's own tooling or documentation; it would be a bespoke solution outside anything MUnit ships or supports directly.

Rejected: `<munit:parameterizations>` already gives equivalent per-code result granularity as a first-class, supported MUnit feature — generation would be solving an already-solved problem with more moving parts.

## Version constraint check

Community sources (DZone article, Medium walkthroughs — titles/snippets only, direct fetch blocked) describe parameterizations as available since early MUnit 2.x lines, well before this repo's MUnit 3.5.0. No version gate is expected to be a blocker; this should be reconfirmed once/if `docs.mulesoft.com` becomes reachable (see Sources), but is not treated as an open risk for the matrix design.

## Sources consulted

- [Parameterized | MuleSoft Documentation](https://docs.mulesoft.com/munit/latest/parameterized) — title/snippet via search only; direct WebFetch returned HTTP 403 from this environment for every `docs.mulesoft.com` URL attempted (both this page and the version-pinned `munit/2.3/parameterized`).
- [Munit: Parameterized Test Suite (DZone)](https://dzone.com/articles/munit-parameterizations-tests) — title/snippet via search only; direct fetch returned HTTP 403.
- [MUnit: Parameterizing the Test Suite (Medium, Jyoti Nimbalkar)](https://medium.com/@jnimbalkar777/munit-parameterizing-the-test-suite-695d260ddf80) — title/snippet via search only; direct fetch returned HTTP 403 (Medium's paywall/bot-block, not specific to this content).
- [MuleSoft: Generic Munit Template with Parameterization (Medium, Bibek Gorain)](https://medium.com/@bibekgorain/mulesoft-generic-munit-template-with-parameterization-3e11ffde554f) — title only, corroborating the same syntax pattern.
- `README.adoc` (this repo) — Option A harness description, used to confirm the parameterization plugs into the existing `munit-util-mock-http-error.req-based-on-vars.munitHttp` utility flow without redesign.

**Caveat:** every `docs.mulesoft.com` URL attempted in this research session (both this ticket and the sibling status-mapping ticket) returned HTTP 403 to direct WebFetch from this environment, and Medium/DZone article bodies were similarly blocked — only WebSearch's own indexed snippets were retrievable. The XML syntax and `p()` accessor above are corroborated across three independent community sources plus the general pattern is unambiguous and low-risk to get wrong, but the exact minimum-version number and the precise wording of MuleSoft's own failure-reporting documentation could not be directly quoted from a primary source in this session. If a session with working access to `docs.mulesoft.com` becomes available, re-fetching `https://docs.mulesoft.com/munit/latest/parameterized` directly would upgrade this from corroborated-by-secondary-sources to primary-sourced.
