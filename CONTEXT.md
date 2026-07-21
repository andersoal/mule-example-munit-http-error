# MUnit HTTP Error Handling

A reference repository for testing MuleSoft HTTP error handling with MUnit. It compares four mocking techniques and publishes an empirically proven map of HTTP status code to Mule error type.

## Language

### Harnesses

**Option A**:
The recommended mocking technique: a real HTTP listener on a dynamic port answers a real request, so the connector itself produces the error.
_Avoid_: listener mock, realistic mock

**Mock Listener**:
The Option A flow that echoes back whatever status code, body and headers the caller asked for.
_Avoid_: stub server, fake server

**Setup Flow**:
A per-test flow, invoked by `then-call`, that prepares the mock's variables before delegating to the shared requester flow.
_Avoid_: mock flow, helper flow

### The Map

**Status Map**:
The published table of what the HTTP connector actually throws for each HTTP status code, every row backed by a green MUnit test.
_Avoid_: matrix, error table, mapping doc

**Map Row**:
One status code and the error type it produces, expressed as a single parameterization of the matrix suite.
_Avoid_: test case, entry

**Named Type**:
An error type the HTTP connector documents by name, such as `HTTP:NOT_FOUND`.
_Avoid_: known error, official type

**Fallback Rule**:
What the connector produces for a status code with no named type. Proven to be `MULE:UNKNOWN`, for both 4xx and 5xx.
_Avoid_: default error, catch-all

**Status-less Error**:
An error type that no status code can produce because it arises from the connection rather than the response: `HTTP:CONNECTIVITY`, `HTTP:TIMEOUT`, `HTTP:RETRY_EXHAUSTED`.
_Avoid_: transport error, infrastructure error

**Seed-and-Correct**:
The workflow for filling the map: write the expected error type as a hypothesis, run the suite, and let red rows correct the data.
_Avoid_: guess-and-check, discovery run
