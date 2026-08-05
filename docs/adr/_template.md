<!--
MADR 4.0 (https://github.com/adr/madr), vendored verbatim except where marked LOCAL.
Upstream sections are unmodified so the template can be re-synced against a future MADR release.
House conventions for this repo are in CLAUDE.md.
-->

---
# These are optional metadata elements. Feel free to remove any of them.
status: "{proposed | rejected | accepted | deprecated | … | superseded by ADR-0123"
date: {YYYY-MM-DD when the decision was last updated}
decision-makers: {list everyone involved in the decision}
consulted: {list everyone whose opinions are sought (typically subject-matter experts); and with whom there is a two-way communication}
informed: {list everyone who is kept up-to-date on progress; and with whom there is a one-way communication}
---

# {short title, representative of solved problem and found solution}

## Context and Problem Statement

{Describe the context and problem statement, e.g., in free form using two to three sentences or in the form of an illustrative story. You may want to articulate the problem in form of a question and add links to collaboration boards or issue management systems.}

<!-- LOCAL — Proof-of-concept evidence belongs here, restated as a finding rather than as a diff
     against a system the reader is not assumed to know. Keep the citation for provenance; lead
     with the finding.
       Good — "Range-partitioning a 90-day-retention alerts table costs ~1,600 index relations per
               tenant and delivers neither bulk expiry nor pruning unless queries carry a
               timestamp predicate." (`database/init/01-schema.sql:300`)
       Bad  — "The PoC partitioned alerts at 01-schema.sql:300 and never used DROP PARTITION." -->

<!-- This is an optional element. Feel free to remove. -->
## Decision Drivers

* {decision driver 1, e.g., a force, facing concern, …}
* {decision driver 2, e.g., a force, facing concern, …}
* … <!-- numbers of drivers can vary -->

## Considered Options

* {title of option 1}
* {title of option 2}
* {title of option 3}
* … <!-- numbers of options can vary -->

<!-- LOCAL — An option with no honest case for it was never a real alternative; leave it out. -->

## Decision Outcome

Chosen option: "{title of option 1}", because {justification. e.g., only option, which meets k.o. criterion decision driver | which resolves force {force} | … | comes out best (see below)}.

<!-- This is an optional element. Feel free to remove. -->
### Consequences

* Good, because {positive consequence, e.g., improvement of one or more desired qualities, …}
* Bad, because {negative consequence, e.g., compromising one or more desired qualities, …}
* … <!-- numbers of consequences can vary -->

<!-- This is an optional element, but it is included in many ADRs. -->
### Confirmation

<!-- LOCAL — upstream MADR wording replaced. -->
{How this decision is confirmed, and kept from eroding afterwards. Prefer a numbered list of
assertions, each one a check that can fail — a test, not a convention. Where there is an
enforceable surface, name the test. Where there isn't, say so plainly rather than falling back on
"code review will catch it".}

<!-- LOCAL — optional element. Obligations this decision places on *future* work, as opposed
     to Consequences, which is what follows from the decision itself. Write it for whoever hits the
     constraint without having read the ADR. Remove if the decision imposes none. -->
### Constraints Imposed

{constraint 1}
{constraint 2}

<!-- This is an optional element. Feel free to remove. -->
## Pros and Cons of the Options

### {title of option 1}

<!-- This is an optional element. Feel free to remove. -->
{example | description | pointer to more information | …}

* Good, because {argument a}
* Good, because {argument b}
<!-- use "neutral" if the given argument weights neither for good nor bad -->
* Neutral, because {argument c}
* Bad, because {argument d}
* … <!-- numbers of pros and cons can vary -->

### {title of other option}

{example | description | pointer to more information | …}

* Good, because {argument a}
* Good, because {argument b}
* Neutral, because {argument c}
* Bad, because {argument d}
* …

<!-- This is an optional element. Feel free to remove. -->
## More Information

{You might want to provide additional evidence/confidence for the decision outcome here and/or document the team agreement on the decision and/or define when/how this decision the decision should be realized and if/when it should be re-visited. Links to other decisions and resources might appear here as well.}
