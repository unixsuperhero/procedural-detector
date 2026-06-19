# Smell Catalog

Seven smells of a procedural mentality. Each entry: **what it is**, **how to spot it**,
**why it costs**, and a **refactor sketch**. Examples are illustrative, not language-bound —
translate the principle to whatever language you're auditing.

The unifying diagnosis: **data and the behavior that operates on it have been separated.**
The unifying fix: **find the object that should own the data, and move the behavior onto it.**

---

## 1. verb_noun function names

**What:** Functions named `verb_noun` — `parse_file`, `generate_report`, `validate_user`,
`calculate_total`, `format_address`.

**Spot it:** A function whose name is an action performed *on* a noun, where that noun is
passed in as the first argument (`parse_file(file)`, `format_address(address)`).

**Why it costs:** The name announces that behavior lives outside the thing it acts on. The
noun is just inert data; every operation on it is a separate free function the reader must
hunt for. Behavior is scattered instead of discoverable on the object.

**Refactor:** Move the verb onto the noun as a method. `parse_file(file)` → `file.parse()`
or a `ParsedFile` object. `format_address(address)` → `address.formatted`. The noun becomes
an object that *owns* its operations.

> Not every `verb_noun` is a smell. Pure, stateless utilities (`clamp(x, lo, hi)`,
> `slugify(str)`) are fine. The smell is specifically when the noun *should* be an object
> with behavior and isn't.

---

## 2. Temporary-variable shuffling

**What:** Values pulled off an object's attributes into locals, mutated, and passed around —
often alongside *other* temporaries pulled from *the same* object.

**Spot it:**
```
name  = user.name
email = user.email
name  = name.strip.downcase
record = build_record(name, email, user.id)
notify(record, email)
```
Three locals derived from one `user`, transformed, threaded through several calls.

**Why it costs:** The object is shredded into loose pieces; the relationships between those
pieces (they all belong to one user) are lost. Logic that belongs to `user` lives in the
caller. Mutation of the temporaries makes data flow hard to follow.

**Refactor:** Give the object the behavior. `user.normalized_name`, `user.to_record`,
`user.notify`. The locals disappear; the object keeps its own data together and exposes
intention-revealing methods instead of raw attributes.

---

## 3. Primitive obsession

**What:** Raw hashes / dicts / maps / anonymous structs / tuples built ad hoc and passed
to functions — especially across module boundaries.

**Spot it:** `process({ id: 1, status: "open", items: [...] })` where the hash has an
implicit, repeated shape; functions reaching in with `data[:status]` / `data["status"]`;
the same bag of keys assembled in multiple places.

**Why it costs:** The hash has a *type* — a real concept — but it's invisible. No validation,
no behavior, no single place the shape is defined. Every consumer re-discovers the keys and
can typo them. Concepts that deserve names stay anonymous.

**Refactor:** Name the concept as a class/struct/value object. `Order.new(...)` with
`order.open?`, `order.items`. The shape lives in one place, behavior attaches to it, and
boundaries pass a *thing* instead of a bag of primitives. Worst across module boundaries —
that's where the missing type hurts most.

---

## 4. Grab-bag hash returns

**What:** Functions returning a hash/dict/object of loosely-related values, often to leak
progress or internal state back to the caller.

**Spot it:**
```
return { success: true, count: 12, errors: [], last_id: 99, duration_ms: 40 }
```
Unrelated fields bundled so the caller can fish out whichever it cares about; different
callers use different keys; the return shape grows over time as callers need "just one more
field."

**Why it costs:** The return value is a peek into the function's private internals. Coupling
runs deep — callers depend on incidental implementation state. No single responsibility: the
function is doing work *and* reporting a dashboard of its insides.

**Refactor:** Return one meaningful value or a small purpose-built result object with
behavior (`result.succeeded?`, `result.errors`). If callers need different things, that's a
sign of different operations — split them. Don't expose internal state; expose *answers*.

---

## 5. Multi-purpose / side-effecting functions

**What:** A function that does more than its name claims — or has side effects unrelated to
its name. `get_user` that also writes a cache and logs analytics; `validate` that also
mutates the thing it validates.

**Spot it:** Name says one thing (`get_`, `validate_`, `calculate_`) but the body also
writes files, sends notifications, mutates arguments, or hits the network. A reader can't
trust the name to predict the effects.

**Why it costs:** Hidden effects break reasoning and reuse. You can't call `get_user`
without triggering the cache write. Tests need elaborate setup. The "and also" work is
invisible at the call site.

**Refactor:** One function, one job — match the name to the effects. Separate the query from
the command (`user.find` vs `cache.store`). Push side effects to explicit, named calls the
caller chooses. If the work genuinely belongs together, make that an object whose name
covers all of it.

---

## 6. Call-site-driven signatures

**What:** Functions, parameters, and return values shaped because *one specific* call site is
known — not because the shape is intrinsic to the operation.

**Spot it:** A param that exists only to tweak behavior for one caller (`render(x, for_email:
true)`); a return value pre-formatted for one consumer (returns an HTML string because the
one caller renders HTML); parameter order or grouping that mirrors how the single caller
happens to hold its data.

**Why it costs:** The function is secretly coupled to its caller. A second caller can't reuse
it without contortion or adding more flags. The abstraction leaks the caller's context into
a thing that should be context-free.

**Refactor:** Design the signature around the operation's *own* concept, not the caller.
Return data/objects, let callers format. Replace boolean mode-flags with separate methods or
polymorphic objects. Ask: "if a second, different caller appeared, would this signature still
make sense?" If not, it's call-site-driven.

---

## 7. Combined patterns & counter-examples

These smells travel together. A classic procedural cluster:

```
def process_order(order_hash)            # 3: primitive obsession (raw hash)
  total = 0                              # 2: temp shuffling
  order_hash[:items].each { |i| total += i[:price] * i[:qty] }
  log("processed")                       # 5: side effect unrelated to name
  return { total: total, ok: true, ts: now }   # 4: grab-bag return
end
```
Refactored toward objects:
```
class Order
  def total = items.sum(&:subtotal)   # behavior owns its data
  def receipt = Receipt.new(self)     # named result object
end
class LineItem
  def subtotal = price * quantity
end
```
Every smell dissolves once `Order` and `LineItem` own their data and behavior.

**Counter-examples — do NOT flag:**
- Pure stateless helpers with no owning concept: `clamp`, `slugify`, `deep_merge`.
- Framework-required free functions (handlers, reducers, CLI entrypoints) shaped by the framework.
- Data-transfer objects at true system boundaries (serialization, wire formats) where a plain
  hash/struct is the contract.
- Performance-critical hot paths where an object allocation per call is a measured cost.

When unsure, flag it with low confidence and explain the trade-off — let the reader decide.
