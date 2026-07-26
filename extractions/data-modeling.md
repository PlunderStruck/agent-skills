# Extraction: data-modeling

**Source:** Data and Reality — William Kent

> Operational rules distilled from the source, written in our own words. Denser than the published skill; the place detail survives for re-distillation.

---

# Operational Rules for Data Modeling, Extracted from Kent's *Data and Reality*

Kent's book (1978) is a philosophical treatment of why representing a domain in data is inherently a series of arbitrary, purpose-driven decisions rather than a discovery of pre-existing structure. Below are the rules that translate into concrete schema, API, and domain-model checks, organized by topic. Vocabulary has been modernized (tables/columns/foreign keys/enums instead of records/fields/catalogs).

## Entity Identity

**Existence tests must be explicit, not incidental.** Kent distinguishes "list tests" (an entity must be explicitly registered somewhere before it can be referenced) from "syntactic tests" (any string of the right shape is accepted, whether or not the entity is real).
- *Trigger*: a column that names another entity — `city`, `manager_name`, `department` — implemented as free text validated only by format (length, character set), not by a real foreign key.
- *What goes wrong*: the system happily stores references to things that were never verified to exist. A typo'd city, a manager who left the company, a department that was never created — all pass validation and only surface as broken joins or wrong reports downstream, far from where the bad data was entered.
- *What to do*: for every field naming another entity, decide deliberately whether its population is closed and enumerable (make it a real foreign key against a real table) or genuinely open-ended (leave it as text, and say so). Don't let "it's just a string field" be a default.
- *How to check*: "Is there a real, finite list of valid values for this field somewhere in the system? If yes, why isn't this a foreign key instead of a syntax check?"

**Separate the surrogate (identity) from the symbol (name).** A thing's identifier should not be a value that could legitimately change while the thing stays "the same thing."
- *Trigger*: choosing a primary/foreign key: a natural key (email, SSN, department code, product name) versus an opaque surrogate key (auto-increment ID, UUID).
- *What goes wrong*: when the natural key must change — an employee's number is reformatted, a department is renamed, an email is corrected — every referencing table either needs a cascading update, or teams fake a delete-and-reinsert that silently severs relationship history (Kent's example: changing an identifier this way can look, structurally, like firing and re-hiring the same employee).
- *What to do*: use an opaque surrogate for identity and foreign keys. Treat human-readable names, codes, and numbers as ordinary mutable attributes, versioned or history-tracked separately if needed.
- *How to check*: "Can the value in this key ever legitimately change for the same real-world thing? If yes, is anything else in the schema using it as a foreign key?"

**Identity resolution is a policy you must state when integrating sources, not something the system infers.** A single "representative" per real-world thing is an intent, not a guarantee — nothing prevents two data-entry paths from creating two rows for one entity.
- *Trigger*: merging or integrating two systems that each maintain their own notion of the "same" entity type (Kent's example: a "part" in an inventory system versus a "part" in a quality-control system, one meaning a SKU and the other a physical unit).
- *What goes wrong*: silent duplicate representatives for the same real thing — the systems never actually agreed on what makes two records "the same," so merges either double-count or produce spurious matches (a variant of the fraud pattern Kent cites: the same person under two identities, undetected because no rule was ever asserted).
- *What to do*: before merging schemas, write down the identity-resolution rule explicitly — what field(s) or process establishes that two incoming records refer to one entity — and pick a single canonical surrogate key for the merged entity.
- *How to check*: "If the same real-world thing enters through two different paths, does anything actually detect and reconcile the duplicate, or does the schema just assume it won't happen?"

## The Oneness Problem

**State, per entity type, which level of "one thing" a row represents.** Kent's recurring example: is a "part" one kind of part (a catalog entry) or one physical instance? Is a "book" the authored work, an edition, or a physical copy?
- *Trigger*: defining any table for a noun that could plausibly be counted multiple ways — parts, books, locations, wells, sessions, accounts.
- *What goes wrong*: two teams (or two features) build against the same table assuming different units. Counts disagree, joins produce nonsense (multiplying instead of matching), and the mismatch is invisible until someone compares a report against reality.
- *What to do*: write, in the schema documentation, exactly which level a row represents (e.g., "a `parts` row is one catalog SKU; physical units live in `part_instances`") and keep the levels in separate tables connected by an explicit relationship, rather than collapsing them into one ambiguous table.
- *How to check*: "If I asked a domain expert 'how many X are there', would my query on this table return the same number they'd give?"

**When one physical or legal thing plays multiple tracked roles, decide explicitly whether the roles are one entity or several.** Kent's example: an employee who is also a stockholder and a dependent — is that one row or three linked rows?
- *Trigger*: modeling a person, organization, or asset that can simultaneously occupy multiple business roles with independent lifecycles (hired/fired, enrolled/withdrawn, active/inactive).
- *What goes wrong*: conflating roles into a single row means deleting or updating one role's data accidentally destroys or corrupts another role's data (firing someone deletes their stockholder record); splitting roles into unlinked tables loses the fact that they're the same underlying person, breaking any "is this the same individual" query.
- *What to do*: model each role as its own entity, connected to a shared underlying identity by an explicit relationship (e.g., `person_id` shared by `employee`, `stockholder`, `dependent`), and define deletion/update semantics per role independently.
- *How to check*: "If this thing loses one of its roles, does removing that role's data accidentally touch data that belongs to a different role of the same underlying thing?"

## Naming and Reference

**A qualified/composite key bakes a relationship into identity, and that relationship then leaks into every unrelated fact about the entity.** Kent's example: identifying an employee's dependent by `(employee_id, first_name)` works as a key, but now every fact about that dependent — a benefits claim, a medical record — carries a redundant, gratuitous reference to "their" employee, even in contexts where that employee is irrelevant.
- *Trigger*: a table's primary key is a composite of a "parent" foreign key plus a locally-unique attribute (e.g., `(department_id, employee_short_code)`, `(account_id, contact_name)`).
- *What goes wrong*: if the qualifying relationship ever needs to change (the dependent is reassigned to a different employee, a contact moves accounts), every table referencing the entity must cascade-update. Worse, tables that have nothing to do with the qualifying relationship still carry it, and it becomes ambiguous whether a given fact is "about the entity" or "about the entity's relationship to its qualifier" (Kent calls this the reducibility ambiguity: the same composite-key shape is indistinguishable between "attribute of the child" and "fact about the parent-child relationship").
- *What to do*: give the child entity its own surrogate key, independent of the qualifying relationship. Represent the qualifying relationship as an ordinary, independently updatable foreign key or join row — not as part of the primary key.
- *How to check*: "If the qualifying value changes, do I have to touch every table referencing this entity, or just one relationship row?"

**Treat "one name, one entity" as a simplification you're choosing, not a default that holds.** Names are non-unique (two entities can share one) and multi-valued (one entity can have several legitimate names — maiden/married, alias, DBA, nickname).
- *Trigger*: dedup logic, "find-or-create" upserts keyed on a display name, unique constraints on human-entered labels, mailing-list-style matching.
- *What goes wrong*: matching on a shared name produces false merges (Kent's assassination-plot example: two unrelated people, same name); failing to account for a real alias produces false negatives — duplicate rows that a system can never recognize as the same thing, the everyday reason "sophisticated computers can't deduplicate a mailing list."
- *What to do*: for any field being used (even informally) as an identifier, ask explicitly whether it's unique and whether it's singular, and if either answer is "no," build a proper synonym/alias table and match on a surrogate key, not the string.
- *How to check*: "Can two different real things ever legitimately share this value? Can one real thing legitimately have two different values for it?" A "yes" to either disqualifies it as a key.

**A naming scheme's scope is local, and forcing a single global namespace has a real integration cost.** Kent's oil-well example: some wells have an industry-standard code, some don't; owning companies each have their own numbering conventions, and jointly-owned wells may have several valid codes at once.
- *Trigger*: designing an identifier meant to eventually unify records from multiple source systems, each with its own numbering scheme.
- *What goes wrong*: inventing one universal ID that nobody's source system natively uses forces a translation layer everyone must maintain forever; alternatively, assuming existing codes are globally unique risks silent collisions when two sources' schemes overlap.
- *What to do*: model identifiers as `(source_system, local_code)` pairs, with a crosswalk table mapping to a canonical surrogate, rather than forcing one universal code prematurely.
- *How to check*: "Is this code assigned by exactly one authority, or could the same code format be independently issued by multiple sources I'll eventually need to merge?"

## Attributes vs. Entities vs. Relationships

**The boundary between a scalar column and a related table is a query-driven choice, not an objective fact about the data.** Kent's central demonstration: "Henry Jones weighs 175 pounds" and "Henry Jones works in Accounting" have identical logical structure, yet one nearly always ends up a scalar column and the other a foreign key, by convention rather than reasoning.
- *Trigger*: deciding whether something is a plain column, an enum, or its own table with a foreign key back to the subject.
- *What goes wrong*: teams argue past each other ("is `department` really an attribute or an entity?") without a decision rule, then discover too late that the "just a column" choice can't support what's actually needed — history, multiplicity, or being queried from the other direction — forcing an expensive schema migration.
- *What to do*: apply a concrete test before modeling: will this value ever need its own attributes (when it started, who asserted it, its source, whether there can be more than one) or need to be queried from the other side ("who/what has this value")? If yes to either, model it as its own table with a relationship, from the start.
- *How to check*: "Will I ever need to ask 'who has this value' as a first-class query, or attach metadata to this fact itself? If yes, it's a relationship, not a column."

**When a fact has attributes of its own, it needs its own row — don't smear relationship metadata onto one of the endpoints.** Kent's examples: date of assignment, percentage of surface covered by a color, age of a dependent — all "facts about facts."
- *Trigger*: adding "since when," "by whom," "confidence," or "quantity" to a many-to-many association (e.g., adding `date_of_assignment` to the `employee` table instead of to an `employee_department_assignment` join table).
- *What goes wrong*: bolting relationship metadata onto one endpoint corrupts that endpoint's own semantics — an employee row that models "current state" silently overwrites history every time the relationship changes, because the row was never built to hold one instance of a repeating fact.
- *What to do*: whenever a relationship needs its own attributes, give it its own table with its own primary key (a proper join/junction table), even for what looks like a simple one-to-many relationship.
- *How to check*: "Does this fact have a lifetime, source, or quantity independent of either endpoint? If so, does it live in its own row, or did it get merged into one side?"

**A single scalar foreign key silently assumes the "other side" is one type and one value — state explicitly when that's false.** Kent's "owns" example: an item can be owned by an employee or a department, two structurally different entity types sharing one relationship.
- *Trigger*: a column like `owner_id` or `assigned_to_id` meant to reference more than one possible entity type, or a fact that can genuinely be multi-valued (an employee with several skills, a car with multiple colors).
- *What goes wrong*: forcing a single FK column across heterogeneous types leads to either an unenforced XOR across two nullable columns, or an invented shared ID space nobody's business logic actually uses; forcing a multi-valued fact into a scalar column produces comma-separated-string hacks that can't be queried or constrained.
- *What to do*: model genuinely polymorphic relationships with an explicit type-discriminator plus ID pair (or a proper table per type joined through a common relationship table); model genuinely multi-valued facts as their own child table, never as plurality encoded into a scalar.
- *How to check*: "Can the other side of this fact legitimately be more than one type of thing, or hold more than one value at once? If yes, a single scalar column can't represent it honestly."

**Grouping two facts in the same row silently asserts a 1:1 correspondence between their populations — notice when that's a hidden constraint, not a foregone conclusion.** Kent's example: storing salary and department assignment in the same employee row implicitly requires every employee with a salary to also have a department, and vice versa.
- *Trigger*: adding a new fact as a new column on an existing table rather than a new related table.
- *What goes wrong*: the co-location becomes an accidental, unstated business rule enforced only by table structure — no one decided that "has a salary" and "has a department" must coincide, but the schema now silently requires it, and nulls become a workaround rather than a decision.
- *What to do*: when adding a column, ask whether its population is truly guaranteed to match the row's other columns; if not, or if that's not actually a rule anyone intended, put it in its own table.
- *How to check*: "If I deleted this column's value while keeping the rest of the row, would that represent a real, valid state — or does the table's structure just not allow expressing it?"

## Categories and Types

**Category boundaries are fuzzy and purpose-relative — don't let one enum serve two purposes that could disagree at the edges.** Kent's example: "employee" seems obvious until you ask whether it includes part-timers, contractors, people on leave, or people who've accepted an offer but not started — and payroll, benefits, and access control may legitimately answer differently.
- *Trigger*: defining a `status` or `type` enum, or a discriminator column, whose value drives more than one downstream business rule (eligibility, access, reporting).
- *What goes wrong*: production data eventually contains a case the enum doesn't cleanly cover, and different consumers of the same field silently disagree about what it means — someone is "an employee" for payroll but not for benefits, and the schema has no way to say so, forcing hacks in application code that only the original author understands.
- *What to do*: for each category/enum, identify every downstream purpose it drives; if two purposes might disagree at the edges, don't collapse them into one shared column — make each purpose's defining predicate independently queryable.
- *How to check*: "Could two legitimate stakeholders disagree about whether a given record belongs in this category? If yes, whose definition does the field currently encode, and where does the other one live?"

**Real entities belong to overlapping categories simultaneously — forced mutual exclusivity (single-table inheritance, one non-nullable `type` column) is a modeling default, not a domain requirement.**
- *Trigger*: choosing single-table-per-type inheritance or a mandatory single-valued `type` discriminator for an entity that could plausibly belong to more than one category over its lifetime (a person who is employee, customer, and stockholder at once).
- *What goes wrong*: forcing mutual exclusivity means either duplicating the entity across separate, barely-linked tables (with "is this the same person" recorded as a fragile secondary relationship, if at all) or picking one category as primary and bolting the others on as ad hoc flags that don't scale.
- *What to do*: model category membership as its own many-to-many relationship (entity ↔ category) rather than an inherent, single-valued property of the row, whenever more than one category could plausibly apply at once.
- *How to check*: "Is there any real member of this population who could simultaneously belong to two of my supposedly mutually-exclusive categories? Would the schema survive that member showing up?"

**A category is a distinct, persistent concept from its current membership — don't let it disappear when it's empty.** Kent's intension/extension distinction: "employee" as a concept doesn't cease to exist just because no one is currently employed.
- *Trigger*: a lookup/reference table (departments, statuses, product categories) whose rows are only implicitly derived from current usage (`SELECT DISTINCT status FROM orders`) rather than stored independently.
- *What goes wrong*: a category with zero current members vanishes from the system's knowledge — it can't be assigned to, validated against, or referenced, and code that lists "valid states" silently breaks when the last row using a given state is deleted.
- *What to do*: model the category/type as its own first-class row in a real table, never cascade-deleted or implicitly derived from current membership.
- *How to check*: "If every current member of this category were removed, would the category itself still exist as a queryable, referenceable row?"

## Change Over Time

**Identity across change requires a chosen invariant ("essence"), and it must be picked deliberately, not left implicit.** Kent's DMV example: a car's identity is defined, by policy, as the engine block — every other part can be swapped without creating "a new car," precisely because someone decided so.
- *Trigger*: designing what a primary key tracks across substantial change — a company after a merger, a product after a spec revision, a user session after a token refresh.
- *What goes wrong*: without an explicit invariant, different developers make inconsistent implicit choices about what counts as "the same thing, updated" versus "a new thing" — audit trails silently break when a renamed entity looks like a delete-and-recreate, or two genuinely different things get treated as continuous because they happened to share a mutable key.
- *What to do*: for any entity that undergoes substantial change, name the specific field or rule that defines continuity, and make that field immutable; treat everything else as a mutable attribute.
- *How to check*: "What's the one thing about this entity that, if it changed, I'd say 'this became a different entity, not an updated one'? Is that actually what the primary key is built on?"

**"Current" and "history" are two readings of the same underlying fact, and the relationship between them must be structurally enforced, not maintained by convention.** Kent's compiler-version example: "the current version" is simultaneously singular (what you invoke) and plural (an explicit list you can roll back through).
- *Trigger*: any "latest wins" column (`current_price`, `current_address`) that coexists with a separate audit/history table for the same fact.
- *What goes wrong*: the two are built independently and drift — updating "current" doesn't reliably append to history, or a point-in-time history query disagrees with what "current" reports right now, and no one notices until an audit.
- *What to do*: pick one representation as primitive (usually the full version history) and derive the other from it mechanically — via a database trigger, a materialized view, or an enforced application-layer invariant — rather than maintaining both by separate code paths.
- *How to check*: "If I query 'current' and 'history as of now' through two different code paths, are they guaranteed to agree, or just usually in sync?"

**Deletion is a "notice/forget" instruction to the system, not necessarily the real-world end of the entity — decide which one it is for every delete path.**
- *Trigger*: implementing hard-delete, soft-delete, or archival for any entity that other rows may still reference.
- *What goes wrong*: hard-deleting a row still validly referenced elsewhere (an employee terminated decades ago, still cited in a historical contract) either orphans those references or forces a cascade that destroys legitimately needed history; conversely, treating "stopped needing to track it" as "ceased to exist" produces false negatives — "we have no record of X" gets misread as "X never existed."
- *What to do*: for every delete/archive path, state explicitly which real-world event it represents (the thing ceased to exist vs. we simply stopped tracking it) and choose hard-delete, soft-delete, or immutable-archive to match — don't let the mechanical operation stand in for a domain decision no one actually made.
- *How to check*: "When this row is deleted, does the real-world thing cease to exist, or does the organization just stop caring? Does every foreign key pointing at it agree with that answer?"

**The threshold between "same thing, edited" and "new thing" must be a stated rule, not whoever's writing the update code that day.** Kent's book/edition/printing spectrum: small changes are absorbed as edits, large changes get a new identity, and the cutoff is arbitrary but must be fixed.
- *Trigger*: schemas for documents, specifications, contracts, or listings that undergo revisions, where an edit could either update a row in place or spawn a new versioned row.
- *What goes wrong*: without a stated threshold, minor edits (typo fixes) get modeled as brand-new entities, breaking every permalink and reference to the old one, while major edits (renegotiated terms) get modeled as in-place updates, silently erasing what was originally agreed — a real liability for anything with legal or audit weight.
- *What to do*: define, structurally, which field(s) or magnitude of change trigger a new version row versus an in-place update (e.g., a content-hash change forces a new version; a metadata-only edit doesn't), and enforce it rather than leaving it to developer judgment.
- *How to check*: "Is there a written rule for what counts as a new version here, or does it depend on who wrote the update code?"

## Viewpoint and Purpose

**A schema encodes one group's viewpoint and purpose — never assume a shared vocabulary exists between teams until you've checked.** Kent's central late argument: there is no single objective reality to model, only agreement reconciled well enough for a given purpose and a given scope of stakeholders — and the reconciliation gets harder as both purpose and stakeholder count grow.
- *Trigger*: any schema or API meant to serve more than one consuming team, or any moment two teams say "obviously X means Y" without having compared definitions.
- *What goes wrong*: the assumed shared vocabulary doesn't actually exist — the same field name (Kent's "well," "part," "account") means structurally different things to different consumers, and this surfaces only after integration, as silent mis-joins or as a dispute over what a report "really" shows.
- *What to do*: for any schema serving multiple consumers, document per entity/field whose definition and purpose it encodes, rather than assuming self-evidence; when reconciling multiple viewpoints, expect the cost to scale with the number of stakeholders and the breadth of purpose — budget for it rather than treating reconciliation as a one-time modeling exercise.
- *How to check*: "If I handed this schema's field list to a second team working with the same subject matter, would they define every field the same way — and does it matter for their purposes if not?"

**A model's structural bias determines which future questions are cheap to ask — make the relationships you expect to matter into first-class, queryable structures.** A schema built for the questions asked at launch can make entire classes of legitimate later questions ("what relationships exist between X and Y, and why?") expensive or impossible, because relationships and their reasons were never represented as data, only as implicit application logic.
- *Trigger*: choosing between a rigid normalized schema, a schema-less blob, or an explicit relationship/graph structure at the start of a system's life.
- *What goes wrong*: significant relationships end up encoded only in application code or convention (a matching column here, a lookup there), so no query can enumerate "every kind of relationship that exists between these two entities" — that knowledge lives only in someone's head.
- *What to do*: for relationships you expect stakeholders to ask about later (not every relationship — that's over-engineering), represent them explicitly as named, queryable structures (a typed join table, not an implied foreign key convention) rather than leaving them to be inferred from code.
- *How to check*: "Could someone unfamiliar with the application code list, from the schema alone, every kind of relationship that exists between two given records?"

## Passages Worth Knowing, Without a Corresponding Check

- Kent's unresolved "murderer and butler" problem — two entities modeled independently and later discovered to be the same one — has no clean schema solution; expect a lossy, ad hoc merge whenever it happens, not a general fix.
- Language habitually supplies nouns for some relationships and not others (a "schedule" connects a train and a time; no equivalent noun connects a person and a salary), which quietly steers which facts modelers notice as "entities" worth naming in the first place — worth being aware of, but not independently checkable in a schema review.
- Kent's closing claim that no absolute shared reality exists, only purpose-bounded reconciliation, reframes what "correct" even means for a data model — it motivates the viewpoint/purpose rules above but adds no further concrete check beyond them.
