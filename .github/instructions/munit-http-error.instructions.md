---
description: "Use when creating or updating Option A MUnit tests. Enforce dynamic-port listener based HTTP error mocking with mock-when plus then-call and helper flows."
name: "MUnit Option A Test Conventions"
applyTo: "src/test/munit/option-a/**/*.xml"
---

# MUnit Option A Test Conventions

## Scope
- Apply only to Option A test suites and helpers.

## Source References



## Required Pattern
- Use dynamic port reservation for local test listener.
- Keep listener and request helper flows in the suite.
- Intercept outbound runtime requests with mock-when.
- Use then-call to invoke helper flows that prepare response context and trigger local listener.
- Enable required helper flow sources via munit:enable-flow-sources.

## Validation Expectations
- Assert business behavior first:
  - expected branch entered
  - expected log/process outcome
- Use verify-call for branch confirmation when meaningful.
- Keep selectors precise:
  - method
  - url or path
  - config-ref or doc:name where needed

## Pitfalls To Prevent
- Missing enable-flow-source for helper/listener flows.
- Attribute mismatch between runtime request and mock-when selectors.
- Duplicate global config collisions from importing external test utility files instead of colocated flows.
- A try/on-error-continue in munit:execution with no munit:validation block: the test swallows every error and can never fail.

## The Status Map (examples/parameterized/)
- The map's data lives in `examples/parameterized/http-status-parameterizations.yaml`, one row per status code. Edit mappings there, never in test XML.
- `http-status-error-map-rows.adoc` is generated from that YAML and must be regenerated whenever the YAML changes. There is no script and no build binding: ask Claude Code to regenerate the map rows. A stale rows file is the only way the published table can lie.
- Status-less error types (`HTTP:CONNECTIVITY`, `HTTP:TIMEOUT`, `HTTP:RETRY_EXHAUSTED`) belong in `impl-http-error-families-test-suite.xml`, never in the parameterized suite: parameterizations are scoped to munit:config, so any test added there runs once per row.
- When a row is red, the connector is right and the YAML is wrong. Correct the YAML to the observed error type, then regenerate.
