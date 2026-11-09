# Decision record template by EdgeX

This is a template for EdgeX Foundry ADR.
Source: https://docs.edgexfoundry.org/2.3/design/adr/template/

> Note: the section headings below follow the EdgeX ADR structure (Submitters, Change Log,
> Referenced Use Case(s), Context, Proposed Design, Considerations, Decision, Other Related ADRs,
> References). The mirror used to build this file had the heading lines stripped, so the headings
> were restored from the EdgeX documentation. Compare with the original on GitHub for exact wording.

# <Title>

## Submitters

List ADR submitters.

Format:

- Name (Organization)

## Change Log

List the changes to the document, incl. state, date, and PR URL.
State is one of: pending, approved, amended, deprecated.
Date is an ISO 8601 (YYYY-MM-DD) string.
PR is the pull request that submitted the change, including information such as the diff, contributors, and reviewers.

Format:

- [Status of ADR e.g. approved, amended, etc.](URL of pull request) YYYY-MM-DD

## Referenced Use Case(s)

List all relevant use case / requirements documents.
ADR requires at least one relevant, approved use case.

Format:

- [Use Case Name](URL)

Add explanations if the ADR is not addressing all the requirements of a use case.

## Context

Describe:

- how the design is architecturally significant - warranting an ADR (versus simple issue and PR to fix a problem)
- the high level design approach (details described in the proposed design below)

## Proposed Design

Details of the design (without getting into implementation where possible).

Outline:

- services/modules to be impacted (changed)
- new services/modules to be added
- model and DTO impact (changes/additions/removals)
- API impact (changes/additions/removals)
- devops impact

## Considerations

Document alternatives, concerns, ancillary or related issues, questions that arose in debate of the ADR.
Indicate if/how they were resolved or mollified.

## Decision

<The decision reached, and the reasoning behind it.>

## Other Related ADRs

List any relevant ADRs - such as a design decision for a sub-component of a feature, a design deprecated as a result of this design, etc..

- [ADR Title](URL) - Relevance

## References

<Links to supporting material.>
