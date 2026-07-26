# Identity

## Separate the surrogate from the symbol

**Trigger.** Choosing a primary or foreign key: a natural key (email, employee number, department code, SKU) versus an opaque surrogate.

**What goes wrong.** When the natural key must change — a code gets reformatted, a department renamed, an email corrected — every referencing table needs a cascading update. Teams that don't want to do that fake a delete-and-reinsert instead, which structurally looks identical to the entity ceasing to exist and a new one appearing. Relationship history is severed and nothing reports an error.

**What to do.** Opaque surrogate for identity and every foreign key. Human-readable names and codes are ordinary mutable attributes, versioned separately if the history matters.

**Check.** Can this value ever legitimately change for the same real-world thing? If yes, is anything using it as a foreign key?

## Existence tests, not syntax tests

**Trigger.** A column naming another entity — `city`, `manager_name`, `department` — stored as free text and validated only by shape: length, character set, maybe a regex.

**What goes wrong.** The system stores references to things that were never confirmed to exist. A typo'd city, a manager who left, a department nobody created — all pass validation, all surface much later as a broken join or a report that's quietly wrong.

**What to do.** For every field naming another entity, decide deliberately whether the population is closed and enumerable — in which case it's a foreign key against a real table — or genuinely open-ended, in which case say so explicitly. "It's just a string field" should not be a default.

**Check.** Is there a real, finite list of valid values somewhere in the system? If so, why is this a syntax check rather than a foreign key?

## Identity resolution is a policy, not an inference

**Trigger.** Merging or integrating two systems that each maintain their own notion of the "same" entity — one system's *part* meaning a catalogue SKU, the other's meaning a physical unit.

**What goes wrong.** A single representative per real thing is an *intent*, not a guarantee. Nothing prevents two entry paths creating two rows for one entity. Because the systems never actually agreed what makes two records the same, merges either double-count or match spuriously — and the same person under two identities goes undetected precisely because no rule was ever asserted.

**What to do.** Before merging schemas, write down the identity-resolution rule: which field or process establishes that two incoming records refer to one entity. Then pick a single canonical surrogate for the merged entity.

**Check.** If the same real thing enters through two paths, does anything actually detect and reconcile the duplicate — or does the schema assume it won't happen?

## Names are neither unique nor singular

Two entities can share a name. One entity can legitimately have several — maiden and married, alias, trading name, nickname.

**Trigger.** Deduplication logic, find-or-create upserts keyed on a display name, unique constraints on human-entered labels.

**What goes wrong.** Matching on a shared name produces false merges: two unrelated people with the same name become one record. Failing to account for a real alias produces the opposite — duplicate rows the system can never recognise as the same thing. This is the everyday reason deduplicating a mailing list is harder than it looks.

**What to do.** For any field used even informally as an identifier, ask both questions explicitly. If either answer is no, build a proper alias table and match on the surrogate.

**Check.** Can two different real things share this value? Can one real thing hold two values for it? Either yes disqualifies it as a key.

## Identifier scope is local

**Trigger.** Designing an identifier meant to eventually unify records across multiple source systems, each with its own numbering convention.

**What goes wrong.** Two failure modes in opposite directions. Inventing one universal ID nobody's source system natively uses forces a translation layer everyone maintains forever. Assuming existing codes are globally unique invites silent collisions when two sources' schemes overlap — and some entities legitimately carry several valid codes at once, issued by different authorities.

**What to do.** Model identifiers as `(source_system, local_code)` pairs with a crosswalk to a canonical surrogate, rather than forcing one universal code prematurely.

**Check.** Is this code assigned by exactly one authority, or could the same format be independently issued by multiple sources you'll eventually merge?

## Qualified keys leak a relationship into everything

**Trigger.** A primary key composed of a parent foreign key plus a locally-unique attribute — `(employee_id, dependent_name)`, `(account_id, contact_name)`.

**What goes wrong.** Two costs. If the qualifying relationship ever changes — the dependent is reassigned, the contact moves accounts — every referencing table must cascade. And every fact about the entity now carries a redundant reference to its qualifier, even in contexts where that qualifier is irrelevant: a medical record for a dependent drags along which employee they're attached to.

It also creates a genuine ambiguity. The same composite-key shape can't distinguish "an attribute of the child" from "a fact about the parent-child relationship."

**What to do.** Give the child its own surrogate. Represent the qualifying relationship as an ordinary, independently updatable foreign key.

**Check.** If the qualifying value changes, do you touch every table referencing this entity, or one relationship row?
