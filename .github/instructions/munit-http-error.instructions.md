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
