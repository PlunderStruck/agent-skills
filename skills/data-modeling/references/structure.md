# Structure: Columns, Tables, and Relationships

## The boundary between attribute and entity is arbitrary

The demonstration worth internalising: *Henry Jones weighs 175 pounds* and *Henry Jones works in Accounting* have identical logical structure. One nearly always becomes a scalar column and the other a foreign key — by convention, not reasoning.

**Trigger.** Deciding whether something is a plain column, an enum, or its own table.

**What goes wrong.** Teams argue past each other — "is `department` really an attribute or an entity?" — with no decision rule, then discover the "just a column" choice can't support what's needed: history, multiplicity, or querying from the other direction. The fix is a migration.

**The test, applied before modelling:** will this value ever need its own attributes — when it started, who asserted it, its source, whether there can be more than one? Or need to be queried from the other side: *who has this value?* Either yes means it's a relationship, and modelling it as a column is borrowing against a migration.

**Check.** Will you ever ask "who has this value" as a first-class query, or attach metadata to this fact itself?

## A fact with attributes needs its own row

**Trigger.** Adding *since when*, *by whom*, *confidence*, or *quantity* to an association — putting `date_of_assignment` on the employee table rather than on an assignment table.

**What goes wrong.** Bolting relationship metadata onto an endpoint corrupts that endpoint's semantics. An employee row models current state, so it silently overwrites history every time the relationship changes. The row was never built to hold one instance of a repeating fact.

**What to do.** A relationship needing its own attributes gets its own table with its own key — even when it looks like a simple one-to-many.

**Check.** Does this fact have a lifetime, source, or quantity independent of either endpoint? If so, does it live in its own row?

## One scalar foreign key assumes one type and one value

**Trigger.** A column like `owner_id` or `assigned_to_id` meant to reference more than one kind of entity — an item owned by either an employee or a department. Or a fact that's genuinely multi-valued: an employee with several skills, a product in several categories.

**What goes wrong.** Forcing a single FK across heterogeneous types produces either an unenforced XOR across two nullable columns, or an invented shared ID space no business logic actually uses. Forcing a multi-valued fact into a scalar produces comma-separated strings that can't be queried, indexed, or constrained.

**What to do.** Polymorphic relationships get an explicit type discriminator plus ID, or a table per type joined through a common relationship. Multi-valued facts get a child table — never plurality encoded into a scalar.

**Check.** Can the other side of this fact legitimately be more than one type, or hold more than one value at once?

## Co-locating two facts asserts their populations match

**Trigger.** Adding a new fact as a column on an existing table rather than as a related table.

**What goes wrong.** Putting `salary` beside `department_id` on the employee row quietly requires that everyone with a salary has a department and vice versa. Nobody decided that. The table structure did, and now nulls become a workaround for a rule no one intended to state.

**What to do.** When adding a column, ask whether its population genuinely matches the row's other columns. If not — or if that was never a rule anyone meant to impose — it belongs in its own table.

**Check.** If you deleted this column's value and kept the rest of the row, would that be a valid state? Or does the structure simply not allow expressing it?

## What the model makes cheap to ask

A schema built for the questions asked at launch can make entire classes of later questions expensive or impossible — because the relationships, and the reasons for them, were never represented as data.

**Trigger.** Choosing between a rigid normalised schema, a schemaless blob, and explicit relationship structures early in a system's life.

**What goes wrong.** Significant relationships get encoded only in application code and convention — a matching column here, a lookup there — so no query can enumerate *what kinds of relationship exist between these two entities*. That knowledge lives in someone's head, and leaves when they do.

**What to do.** For relationships you expect stakeholders to ask about — not every relationship, which is its own over-engineering — represent them explicitly as named, queryable structures rather than leaving them inferable only from code.

**Check.** Could someone who has never read the application code list, from the schema alone, every kind of relationship between two given records?
