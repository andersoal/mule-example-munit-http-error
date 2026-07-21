# The published map rows are generated from the parameterization data, by hand-off to an agent

The parameterization YAML holds exactly the map's data — status code and expected error type — so the published table is generated from that file into an included rows document, wrapped in a hand-written shell that carries the prose and footnotes. Duplicating 63 rows by hand in AsciiDoc would drift silently, because the tests validate the YAML and nothing validates the document.

The generation is performed by an agent on request and documented as a step in `.github/instructions/munit-http-error.instructions.md` — there is no script and no Maven binding. This repo has no CI, so a build-phase generator would enforce nothing that the agent step does not, and the pom is deliberately being kept free of extra machinery.

## Consequences

- Freshness of the published rows is a social guarantee, not a mechanical one. The document must say so rather than implying otherwise.
- A green suite is what makes the table true: a YAML row that disagrees with the connector is a red test named after its own status code, so "suite green" and "table correct" are the same statement.
