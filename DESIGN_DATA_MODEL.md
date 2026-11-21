# Teach developers to design a data model

## Purpose

Build a short, practical course that helps developers make data modeling decisions before writing application code or asking AI to generate it. The course takes its starting point from [Mani Doraisamy's article, “We promoted developers into architects without informing them”](https://manidoraisamy.com/developer-to-architect.html): a developer must question the requirement, choose what information the product actually needs, define the model, and review implementation against those decisions. The article supplies the motivation; the course supplies a repeatable method.

**Audience:** developers who can write SQL and build CRUD features but have had little practice designing a schema from ambiguous requirements. Use a relational database for the exercises, while keeping the early modeling steps independent of a particular database.

**Outcome:** given a product request, a learner can produce a small, defensible data model; explain its assumptions and tradeoffs; check it against real operations and sample records; and guide a schema change without accumulating redundant tables or losing important data rules.

## Anchor case: tracking addon installations

Use a fictionalized version of the article's situation throughout the course: “Count new installations each week and let marketing follow up with new customers.” The welcome screen must load promptly, and an installer’s email may be unavailable. Release new facts in stages so learners have to revisit their first model:

1. **Count installs:** only an installation timestamp is reliably available. Ask what counts as an installation or a new customer, and whether those are the same measure.
2. **Contact people:** an email becomes available only in some consented flows, after the welcome screen. Decide how to represent unknown identity, consent, and the link between a person and an installation. Do not infer a contactable person from a timestamp.
3. **Handle change:** an account may install more than once; a person may revoke consent or request deletion; marketing needs a historical weekly count. Decide which facts and history must survive each event, subject to the agreed retention policy.

Give learners stakeholder answers, a few example records, and the main reads and writes for each stage. Several models may be valid if their assumptions, constraints, and costs are clear.

## Learning sequence

Run five 60–90 minute workshops. Each starts with a short example, spends most of its time on a modeling task in pairs, and ends with a design review. Keep one evolving model across all five sessions.

| Workshop | Decision to practice | Learner output |
| --- | --- | --- |
| 1. Clarify the product request | Identify the decisions hidden in “new customer”: event versus person, available identifiers, consent, latency, reporting window, and what the team will give up if a field is unavailable. Agree on the questions and assumptions before choosing tables. | A one-page requirements brief with open questions, three concrete operations, and explicit exclusions. |
| 2. Draw the conceptual model | Name the domain facts and their lifecycles. Define the grain of each record; distinguish installation events, accounts, people, and consent. Mark relationships, cardinalities, and unknown values without choosing SQL types yet. | A glossary and small entity-relationship diagram with example records. |
| 3. Design the logical schema | Choose stable identifiers, keys, foreign keys, uniqueness rules, optional fields, and history. Separate facts that change at different rates; justify any duplicated data. Translate business rules into constraints where possible. | A table sketch with columns, relationships, invariants, and two rejected alternatives. |
| 4. Test the model against use | Walk through install, consent, revoke, reinstall, and deletion scenarios. Write representative queries for weekly counts and contact eligibility. Check whether the model answers them correctly, then choose types and indexes from observed access patterns. Discuss retention and sensitive data. | Sample rows, query sketches, an initial physical schema, and a short list of performance/privacy tradeoffs. |
| 5. Evolve and review | Introduce a changed requirement. Plan a safe data migration and decide whether compatibility is needed and for how long. Review an AI-generated schema and migration that adds a redundant table, assumes every install has an email, or weakens a constraint. | A revised model, migration outline, and review comments tied to the agreed model. |

## The modeling method to reinforce

Use the same checklist in every workshop so learners can apply it later without the case study:

1. **Ask what decision the data supports.** Identify the user action, report, or rule; push back when the requested data cannot be obtained reliably or does not justify its cost.
2. **Define the grain and lifecycle.** State what one record means, when it is created, what can change, and when it is removed or retained.
3. **Name entities and relationships.** Record cardinality, optionality, and the difference between a missing fact and a negative fact.
4. **Write invariants and examples.** Test duplicates, retries, partial identity, changing consent, and historical reports before writing DDL.
5. **Choose storage details from operations.** Pick keys, constraints, types, indexes, and any denormalization to serve the agreed reads and writes; record the tradeoff.
6. **Control change.** Treat the model and its invariants as the implementation contract. Require a design review when generated code or a new feature would change them.

## Assessment and teaching materials

The final exercise uses a **new domain**, such as equipment loans, so learners cannot merely repeat the installation schema. Give each learner an ambiguous request, two later change requests, a few sample records, and three operations to support. Ask for a requirements brief, conceptual diagram, logical schema, example queries, and a short review of a flawed proposed migration.

Assess the reasoning rather than a single answer key. A successful submission:

- identifies material unanswered questions and states its assumptions;
- defines record grain, relationships, optionality, keys, and enforceable invariants;
- handles the given operations and edge cases without inventing unavailable data;
- explains indexing, sensitive data, retention, and migration decisions at the level the case requires; and
- catches model changes in generated code and proposes a justified correction.

Prepare an instructor pack with staged requirement cards, stakeholder answers, sample data, a few defensible model variants, common failure examples, and a review rubric. Pilot the first workshop with a small group of developers; note which questions they miss and where the scenario is ambiguous, then adjust the cards before running the full sequence. The first publishable package is the case study, five workshop outlines, and the final exercise with its rubric.
