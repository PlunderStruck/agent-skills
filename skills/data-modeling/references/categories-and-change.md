# Categories and Change

## Category boundaries are fuzzy and purpose-relative

**Trigger.** Defining a `status` or `type` enum whose value drives more than one downstream rule — eligibility, access, reporting.

**What goes wrong.** *Employee* looks obvious until you ask whether it includes part-timers, contractors, people on leave, or someone who accepted an offer but hasn't started. Payroll, benefits, and access control may each legitimately answer differently.

Production data eventually contains a case the enum doesn't cover, and different consumers of the same field silently disagree. Someone is an employee for payroll but not for benefits, the schema has no way to say so, and the difference gets hacked into application code only the original author understands.

**What to do.** For each enum, identify every downstream purpose it drives. If two purposes could disagree at the edges, don't collapse them into one shared column — make each purpose's defining predicate independently queryable.

**Check.** Could two legitimate stakeholders disagree about whether a record belongs in this category? If so, whose definition does the field currently encode, and where does the other one live?

### The code-level counterpart

The same decision recurs in the type system, and there it can be enforced rather than documented. A status field beside fields only meaningful for some of its values is the schema version of what `design-by-contract`'s [types-as-contracts](../../design-by-contract/references/types-as-contracts.md) calls representing each state as its own type — give each state a shape carrying only its own fields, so a function that only makes sense in one state cannot be handed another.

Worth deciding both at once: if the schema keeps a nullable column per status, the code will keep a null check per reader.

## Mutual exclusivity is a default, not a requirement

**Trigger.** Single-table inheritance, or a mandatory single-valued `type` discriminator, for something that could belong to several categories at once — a person who is simultaneously employee, customer, and stockholder.

**What goes wrong.** Forcing exclusivity means either duplicating the entity across barely-linked tables — with "is this the same person" recorded as a fragile secondary relationship, if at all — or picking one category as primary and bolting the rest on as ad hoc flags that don't scale.

**What to do.** Where more than one category can plausibly apply at once, model membership as its own many-to-many relationship rather than an inherent single-valued property of the row.

**Check.** Is there any real member of this population who could belong to two supposedly exclusive categories? Would the schema survive them showing up?

## A category exists independently of its members

**Trigger.** A reference table — departments, statuses, product categories — whose rows are derived from current usage rather than stored, e.g. `SELECT DISTINCT status FROM orders`.

**What goes wrong.** A category with zero current members vanishes from the system's knowledge. It can't be assigned to, validated against, or referenced, and code listing valid states breaks silently when the last row using one is deleted.

The concept and its current membership are different things. *Employee* doesn't stop existing because nobody is currently employed.

**What to do.** Model the category as a first-class row in a real table, never cascade-deleted and never implicitly derived from membership.

**Check.** If every current member were removed, would the category still exist as a queryable row?

---

## Identity across change needs a chosen invariant

**Trigger.** Deciding what a primary key tracks through substantial change — a company after a merger, a product after a spec revision, a session after a token refresh.

**What goes wrong.** Without an explicit invariant, different developers make inconsistent implicit choices about what counts as *the same thing, updated* versus *a new thing*. Audit trails break where a rename looks like a delete-and-recreate, or two genuinely different things get treated as continuous because they shared a mutable key.

The useful framing: a vehicle's identity can be defined, by policy, as its engine block — every other part replaceable without it becoming a different vehicle. That's a decision somebody made, not a fact anyone discovered.

**What to do.** For anything undergoing substantial change, name the specific field or rule defining continuity, and make it immutable. Everything else is a mutable attribute.

**Check.** What's the one thing that, if it changed, would make you say this became a *different* entity rather than an updated one? Is that what the key is built on?

## Current and history are two readings of one fact

**Trigger.** A "latest wins" column — `current_price`, `current_address` — coexisting with a separate audit table for the same fact.

**What goes wrong.** The two get built independently and drift. Updating current doesn't reliably append to history, or a point-in-time query disagrees with what current reports right now. Nobody notices until an audit.

**What to do.** Pick one representation as primitive — usually the full version history — and derive the other mechanically, via a trigger, a view, or an enforced invariant. Not two code paths maintaining two representations.

**Check.** If you query current and history-as-of-now through different paths, are they *guaranteed* to agree, or just usually in sync?

## Deletion is a claim about the world

**Trigger.** Implementing hard delete, soft delete, or archival for anything other rows may still reference.

**What goes wrong.** Hard-deleting a row still validly referenced elsewhere — an employee terminated decades ago, still cited in a historical contract — either orphans those references or forces a cascade that destroys legitimately needed history. In the other direction, treating *we stopped tracking it* as *it ceased to exist* produces false negatives: "we have no record of X" gets read as "X never existed."

**What to do.** For every delete path, state which real-world event it represents and choose the mechanism to match. Don't let the mechanical operation stand in for a domain decision nobody made.

**Check.** When this row is deleted, does the real thing cease to exist, or does the organisation just stop caring? Does every foreign key pointing at it agree?

## The threshold between edit and new version must be stated

**Trigger.** Documents, specifications, contracts, or listings that undergo revision, where an edit could update in place or spawn a new version.

**What goes wrong.** Without a stated threshold, minor edits get modelled as new entities — breaking every permalink and reference — while major edits get modelled as in-place updates, silently erasing what was originally agreed. For anything with legal or audit weight, the second is a real liability.

**What to do.** Define structurally which changes trigger a new version versus an in-place update — a content-hash change forces a version, a metadata-only edit doesn't — and enforce it rather than leaving it to whoever writes the update code.

**Check.** Is there a written rule for what counts as a new version, or does it depend on the author of the day?
