---
name: data-modeling
description: Make the arbitrary decisions in a schema explicit — what makes two records the same thing, whether a row is one thing or several, when a value should be a column versus its own table, what a category means at its edges, and what identity survives change. Use when designing or reviewing a schema, choosing a primary key, adding a status enum, modelling history or versioning, or integrating data from two systems that name things differently.
---

# Data Modeling

## Purpose

A schema is not a discovery of structure that was already there. It is a series of arbitrary, purpose-driven decisions about how to represent a domain — and almost all of them get made implicitly, by whoever happened to write the migration.

The failures that follow are recognisable: two teams reading the same column differently, a count that disagrees with reality, a rename that looks like a delete-and-recreate, a category with no slot for the record that just arrived. In each case the decision was never wrong exactly — it was never *made*.

This skill is about surfacing those decisions while they're still cheap.

## Rules that apply without loading anything

**1. Separate identity from name.** A primary key must not be something that can legitimately change while the thing stays the same thing. Use an opaque surrogate; treat codes, emails, and names as ordinary mutable attributes. Otherwise a rename cascades through every referencing table — or gets faked as a delete-and-reinsert, which silently severs history.

**2. If there's a finite list of valid values, it's a foreign key, not a string.** A column naming another entity, validated only by format, will happily store references to things that don't exist. The typo surfaces later as a broken join.

**3. State which level of "one thing" a row represents.** Is a `parts` row a catalogue entry or a physical unit? A `book` row a work, an edition, or a copy? Two features will assume different answers, counts will disagree, and joins will multiply instead of match.

**4. A value gets its own table if it will ever need its own attributes or be queried from the other side.** "When did this start," "who asserted it," "can there be more than one" — any yes means it's a relationship, not a column. Decide before the migration, not after.

**5. A fact with attributes of its own needs its own row.** Bolting *date of assignment* onto the employee table corrupts that table's semantics — it models current state, so every change silently overwrites history.

**6. Co-locating two facts in one row asserts their populations match.** Adding a `salary` column beside `department_id` quietly requires that everyone with one has the other. Nobody decided that; the table structure did.

**7. Name the invariant that survives change.** For anything that undergoes substantial revision, state what makes it the same entity afterward. Without that, some developers treat a change as an update and others as a new row, and the audit trail breaks at the seam.

**8. Deletion is a claim about the world — pick which one.** "This ceased to exist" and "we stopped tracking it" are different events. Hard-deleting a row still validly referenced by history destroys evidence; treating "stopped caring" as "never existed" produces false negatives.

## Triage

| What you're doing | Reference |
|---|---|
| Choosing keys, resolving duplicate records, integrating two systems | [identity](references/identity.md) |
| Deciding column vs table, modelling relationships and their attributes | [structure](references/structure.md) |
| Status enums, type discriminators, versioning, history, soft deletes | [categories-and-change](references/categories-and-change.md) |

## The question underneath all of them

**Whose viewpoint and whose purpose does this schema encode?**

There is no single objective reality to model — only agreement reconciled well enough for a given purpose among a given set of stakeholders. That reconciliation gets harder as both grow, and the cost is routinely treated as a one-time modelling exercise rather than an ongoing one.

The practical consequence: the same field name means structurally different things to different consumers, and this surfaces only after integration, as a silent mis-join or a dispute over what a report actually shows.

**The check:** hand your field list to a second team working the same subject matter. Would they define every field the same way? Where they wouldn't, does it matter for their purposes — and does the schema have any way to say so?

## Two things worth knowing that produce no check

Included because they change how you read the rest, not because you can act on them directly.

**Some problems have no clean fix.** Two entities modelled independently and later discovered to be the same one is a genuine unsolved case. Expect a lossy, ad hoc merge when it happens rather than a general solution — and design identity resolution so it happens less often.

**Language biases which facts you notice.** A "schedule" is a noun connecting a train and a time; there's no equivalent noun connecting a person and a salary. Relationships that happen to have names in English get modelled as entities; relationships that don't get flattened into columns. That asymmetry is in your vocabulary, not in the domain.

## Boundary with neighbouring skills

- **`sql-performance`** — making a query fast against a schema that exists. This skill is about whether the schema represents the domain honestly. A well-modelled schema can be slow; a fast one can be lying.
- **`api-evolution`** — changing a shape other systems already read. Related on identity and versioning; that skill covers compatibility, this one covers whether the model was right to begin with.
- **`distributed-data`** — correctness when the same data lives in more than one place.
