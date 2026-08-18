# Senior Java Interview Prep — Full Answer Key

Companion to `senior-java-interview-prep.md`. Every question and every probe is answered in depth.

**Conventions and caveats used throughout:**

- "Modern JDK" means JDK 17 or 21 (the current LTS releases most enterprises run). Where behaviour differs by version, the version is stated explicitly.
- Where behaviour is vendor-specific (HotSpot vs OpenJ9), database-specific (PostgreSQL vs MySQL/InnoDB vs Oracle), or Spring-version-specific, that is called out rather than generalised.
- Where something is genuinely contested or depends on context, the answer says so rather than inventing a rule. An interviewer at senior level respects "it depends, and here is what it depends on" far more than false certainty.
- Precise numeric constants (thresholds, defaults) are given where they are stable parts of the JDK/library source. Treat any default as something to verify against your actual version before quoting it in an interview — defaults change between releases.

---

# 1. Java Internals & Language

---

## Q1. Walk me through what happens in memory when you write `String s = "abc" + someVar;` in a loop. Now do the same for Java 8 vs Java 9+.

### Answer

**The source-level fact:** `String` is immutable. Every concatenation that produces a new logical string must produce a new `String` object with a new backing array. Nothing at any JDK version changes that. What changes between Java 8 and Java 9+ is *how the compiler and runtime construct* that result.

**Java 8 and earlier — `StringBuilder` desugaring.**

`javac` rewrites `"abc" + someVar` into roughly:

```java
new StringBuilder().append("abc").append(someVar).toString()
```

This is a purely compile-time transformation; the bytecode literally contains `new java/lang/StringBuilder`, `invokevirtual append`, `invokevirtual toString`. Per iteration of a loop you therefore allocate:

1. A `StringBuilder` object.
2. Its internal `char[]` (default capacity 16, doubling as needed — each growth is a fresh array plus an `Arrays.copyOf`).
3. The final `String` returned by `toString()`, which in Java 8 copies the builder's `char[]` into a new `char[]`.

So a loop of *n* concatenations produces on the order of *3n* allocations plus any intermediate growth arrays. All of these are short-lived and die in the young generation, so they are cheap to collect — but they are not free, and they inflate allocation rate, which is the dominant driver of minor GC frequency.

**The genuinely bad case** is accumulating into a variable across iterations:

```java
String s = "";
for (int i = 0; i < n; i++) {
    s = s + item[i];   // O(n²)
}
```

Here the compiler creates a *new* `StringBuilder` each iteration and copies the entire accumulated string into it. Total work is quadratic in the final length. This is the classic bug. The fix is to hoist a single `StringBuilder` outside the loop, or use `String.join` / `Collectors.joining` / `StringJoiner`.

**Java 9+ — `invokedynamic` and `StringConcatFactory` (JEP 280).**

`javac` no longer emits the `StringBuilder` dance. It emits a single `invokedynamic` instruction whose bootstrap method is `java.lang.invoke.StringConcatFactory.makeConcatWithConstants`. The constant-pool entry carries a *recipe* string describing the shape of the concatenation (literal fragments plus argument placeholders) and the argument types are encoded in the call site's `MethodType`.

On first execution the bootstrap runs and builds a `MethodHandle` chain (the default strategy generates code that computes the exact total length first, allocates one byte array of exactly the right size, and fills it). That call site is then linked, so subsequent executions are a direct invocation of the generated handle with no re-bootstrapping.

The practical consequences:

- **Fewer allocations.** In the good case, one `byte[]` and one `String` — no `StringBuilder`, no growth-and-copy.
- **Better JIT behaviour.** The generated handles inline well, and the length is computed up front rather than discovered by growth.
- **Slower first execution.** Bootstrapping a call site costs microseconds and pulls in the `MethodHandle` machinery. On a program with thousands of distinct concat sites this shows up as startup latency, which is why `-Djava.lang.invoke.stringConcat=BC_SB` exists as an escape hatch reverting to the old `StringBuilder` strategy (relevant for startup-sensitive workloads; largely irrelevant for long-running servers).
- **The quadratic loop bug is unchanged.** `s = s + x` in a loop is still O(n²) on Java 21. Indy makes each individual concatenation cheaper; it does not stop you from re-copying the whole accumulated string every iteration.

**Compact Strings (JEP 254, also Java 9)** is a separate but related change often conflated with the above. `String` switched its backing field from `char[]` to `byte[]` plus a `byte coder` field. Strings whose characters all fit in Latin-1 use one byte per character instead of two, roughly halving memory for typical Western text. Strings containing any character outside Latin-1 use UTF-16 and two bytes per character, as before. This is why heap dumps of Java 9+ applications show `byte[]` dominating where `char[]` used to.

### Probes

**`StringBuilder` desugaring vs `invokedynamic`/`StringConcatFactory`.** Covered above. The key interview point is that the Java 8 behaviour is a *compile-time* rewrite baked into the class file, whereas the Java 9+ behaviour is a *runtime* linkage decision. That matters because the JDK can change its concatenation strategy without recompiling your code — your class file just says "concatenate these things", and the runtime decides how.

**Why the JDK changed it.** Three reasons. (1) The `StringBuilder` shape was frozen into every class file ever compiled, so the JVM could never improve concatenation for existing binaries. Indy moves the decision to runtime. (2) The old shape defeated optimisation: the JIT had to prove a lot about the builder to eliminate the intermediate array, and often couldn't. (3) It allowed exact-size allocation, removing the growth-and-copy pattern.

**String pool and `intern()`.** String *literals* are interned automatically: the constant pool entry is resolved lazily to a single `String` instance per literal value per classloader context, held in the JVM's string table (a native hash table; the objects themselves live on the heap). `"abc" == "abc"` is therefore true, while `new String("abc") == "abc"` is false. `intern()` puts a runtime-computed string into that table and returns the canonical instance.

Practical guidance: **do not call `intern()` as a memory optimisation reflexively.** It has real costs — a native hash table lookup, and interned strings historically caused hard-to-diagnose issues when the table was undersized (`-XX:StringTableSize`; default has grown over versions). Since Java 7 the string table's contents are on the normal heap and are garbage-collectible, so the old PermGen-exhaustion failure mode is gone, but the lookup cost remains. If you have genuine duplicate-string bloat, measure it (a heap dump plus `jmap -histo`, or `-XX:+PrintStringTableStatistics`), then consider either `intern()`, a custom deduplication map, or `-XX:+UseStringDeduplication` (a G1 feature that deduplicates the backing arrays during GC without changing object identity — usually the better answer because it is transparent).

**When concatenation in a loop actually matters vs premature optimisation.** It matters when (a) the accumulation is across iterations — that is a complexity bug, not a constant-factor one, and you fix it regardless of profiling; or (b) the loop is genuinely hot and on the critical path, in which case you profile first. It does *not* matter for logging statements executed once per request, string building in configuration code, or anything outside a measured hot path. The senior answer distinguishes "algorithmic mistake, always fix" from "constant-factor tuning, measure first."

One related real-world point: `log.debug("x=" + x)` builds the string even when debug is disabled. That is worth fixing not because concatenation is slow but because you are doing work you then throw away. Use parameterised logging: `log.debug("x={}", x)`.

---

## Q2. Explain the `hashCode`/`equals` contract. What concretely breaks if you violate it — and what breaks if you make a mutable field part of `hashCode` and then mutate it while the object is in a `HashSet`?

### Answer

**The contract, precisely as specified in `java.lang.Object`:**

For `equals`:
1. **Reflexive:** `x.equals(x)` is true for non-null `x`.
2. **Symmetric:** `x.equals(y)` is true if and only if `y.equals(x)` is true.
3. **Transitive:** if `x.equals(y)` and `y.equals(z)`, then `x.equals(z)`.
4. **Consistent:** repeated calls return the same result provided no information used in the comparison is modified.
5. **Null:** `x.equals(null)` is false.

For `hashCode`:
1. **Consistent:** repeated calls on the same object return the same integer, provided no information used in `equals` is modified.
2. **Equal implies equal hash:** if `x.equals(y)`, then `x.hashCode() == y.hashCode()`.
3. **Unequal does not require unequal hash** — collisions are legal, but distributing distinct objects across distinct hashes improves hash table performance.

Note the asymmetry in rule 2: equality constrains hash codes, not the other way round. That is why the practical rule is "every field used in `equals` must be used in `hashCode`, and no field used in `hashCode` may be absent from `equals`" — the sets should match.

**What concretely breaks.**

*Violating "equal implies equal hash"* (the most common violation — overriding `equals` and forgetting `hashCode`): hash-based collections silently lose data. `map.put(key, v)` places the entry in bucket `hash(k1) % n`. A later `map.get(k2)` with `k2.equals(k1)` computes a different hash, looks in a different bucket, finds nothing, returns `null`. The object *is* in the map. You cannot retrieve it. `set.contains(x)` returns false for an element the set contains. `set.add` on a duplicate creates a second entry. Nothing throws; the data is simply wrong.

*Violating symmetry:* typically arises when a subclass's `equals` compares against subclass-specific fields, or when a class considers itself equal to a different type. Then `a.equals(b)` is true but `b.equals(a)` is false. `List.contains` behaves differently depending on which object is the receiver, and behaviour changes if the list is reordered. `Collections.disjoint`, `removeAll`, and set operations become order-dependent and unpredictable.

*Violating transitivity:* the classic case is the `Point`/`ColorPoint` inheritance problem. A `ColorPoint` that ignores colour when compared to a `Point` produces `cp1.equals(p)` and `p.equals(cp2)` both true while `cp1.equals(cp2)` is false. Set membership becomes dependent on insertion order. This is the fundamental reason `equals` and inheritance conflict, and why the two standard resolutions are (a) `getClass() != o.getClass()` → not equal (breaks Liskov substitutability for subclasses, but preserves the contract), or (b) favour composition over inheritance so the problem does not arise. `instanceof` checks are only safe when the class is `final` or the equality semantics are deliberately defined across the hierarchy — this is exactly what Hibernate proxies force you to think about, since a proxy is a generated subclass.

*Violating consistency:* if `equals` depends on something external (current time, a database lookup, network state), results become nondeterministic and collections corrupt unpredictably.

**The mutable-field-in-`hashCode` scenario.**

```java
Set<Order> set = new HashSet<>();
Order o = new Order(status: NEW);   // hashCode includes status
set.add(o);                          // stored in bucket for hash(NEW)
o.setStatus(SHIPPED);                // hashCode now differs
set.contains(o);                     // false — looks in the bucket for hash(SHIPPED)
set.remove(o);                       // no-op — same reason
```

The object is now **unreachable through the set's normal API but still strongly referenced by it**. Consequences:

- `contains` and `remove` fail for an object you hold a reference to.
- Iterating the set *does* yield the object (iteration walks buckets directly), so `set.iterator()` returns something `set.contains()` denies. This inconsistency is what makes the bug so confusing in the field.
- `set.size()` still counts it, so the set never shrinks — a memory leak in a long-lived set.
- Adding the object again succeeds, because the lookup in the new bucket finds nothing. You now have the same object twice in a `Set`.
- If you later mutate the field *back* to its original value, the object becomes findable again. Bugs that fix themselves are the worst kind to diagnose.

The identical problem occurs with `TreeSet`/`TreeMap` if you mutate a field used by the `Comparator` — the element is now in the wrong position in the tree and binary search will not find it.

**The rules that follow.** Prefer immutable value types as keys and set elements — `record`s are ideal for this. If a type must be mutable, either exclude mutable fields from `equals`/`hashCode` (basing them on an immutable business key or a surrogate ID), or never place instances in hash-based collections while mutable, or remove-mutate-reinsert as a disciplined pattern (fragile; easy to forget).

### Probes

**Lost entries / "contains returns false for an object that's in the set".** Explained above. The precise mechanism is bucket index computation: `HashMap` computes `hash(key)` once at insertion to choose the bucket, and again at lookup. If the two differ, the lookup examines the wrong bucket and short-circuits before any `equals` call ever happens. This is why the failure is silent: no `equals` comparison is even attempted.

**Why `equals` must be reflexive/symmetric/transitive.** Because the entire collections framework and much library code assumes `equals` defines an *equivalence relation*, and equivalence relations are by definition reflexive, symmetric and transitive. Algorithms rely on this: `List.contains` may compare in either direction; `Set` semantics ("no two equal elements") is only well-defined if equality partitions elements into disjoint classes; `removeAll`/`retainAll` assume you can test membership from either side. Break the relation and these operations produce results that depend on internal iteration order, which is an implementation detail that can change between JDK versions.

A concrete symmetry-violation trap in the JDK itself: a case-insensitive string wrapper that does `if (o instanceof String) return s.equalsIgnoreCase((String) o);` — then `wrapper.equals("abc")` is true but `"abc".equals(wrapper)` is false, because `String.equals` type-checks. `list.contains(wrapper)` and `list.indexOf(wrapper)` then give different answers depending on the list implementation's comparison order.

**`Comparable` consistency with `equals`, and what `TreeMap` does when it isn't.** The `Comparable` javadoc *strongly recommends* — but does not require — that `x.compareTo(y) == 0` be consistent with `x.equals(y)`. `TreeMap` and `TreeSet` are the reason. They are documented as behaving "strangely" when this is violated because **they use `compareTo`/`Comparator.compare`, never `equals`, to determine membership.**

Concretely: `BigDecimal` is the canonical inconsistent class. `new BigDecimal("1.0").equals(new BigDecimal("1.00"))` is **false** (`equals` compares scale as well as value), but `compareTo` returns 0 (numeric comparison ignores scale). Therefore:

```java
Set<BigDecimal> hash = new HashSet<>();
hash.add(new BigDecimal("1.0"));
hash.add(new BigDecimal("1.00"));   // size 2

Set<BigDecimal> tree = new TreeSet<>();
tree.add(new BigDecimal("1.0"));
tree.add(new BigDecimal("1.00"));   // size 1 — second is "already present"
```

The same collection contract, two different answers, both correct per their own documentation. This means `TreeSet` does not satisfy the general `Set` contract when its ordering is inconsistent with `equals` — the javadoc admits this explicitly and says such a set is "well-defined but fails to obey the general contract of `Set`". Practical impact: swapping a `HashSet` for a `TreeSet` (or vice versa) can silently change de-duplication behaviour. Also note that a `TreeMap` with a `Comparator` that returns 0 for distinct objects will overwrite entries.

---

## Q3. Describe `HashMap`'s internal structure in a modern JDK. What happens on collision, and what changes at 8 entries in a bucket?

### Answer

**Structure.** A `HashMap` holds a `Node<K,V>[] table`. Each `Node` stores the cached `hash`, the `key`, the `value`, and a `next` pointer. The table length is always a power of two. The bucket index is computed as `hash & (table.length - 1)`, which is equivalent to modulo for powers of two but is a single AND instruction.

The table is allocated lazily on first `put`, not in the constructor — so `new HashMap<>()` allocates almost nothing. Default initial capacity is 16, default load factor 0.75, so the first resize occurs when the 13th entry is inserted.

**Hash spreading.** `HashMap` does not use `key.hashCode()` directly. It applies:

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

XOR-ing the high 16 bits into the low 16 bits. Also note `null` keys are permitted and hash to 0, always landing in bucket 0.

**Collision handling.** Two keys colliding on bucket index are chained. New entries are appended at the **tail** of the list (Java 8 changed this from head insertion — relevant to the concurrency point below). On `get`, the map walks the chain comparing first the cached `hash` (an int comparison, cheap, and a fast rejection), then `==` on the key reference, then `equals`. The cached hash means `equals` is only invoked for genuine hash matches.

**Treeification — what changes at 8.**

Two constants govern this (from the JDK source):

- `TREEIFY_THRESHOLD = 8` — when a bucket's chain reaches 8 nodes, the bucket *may* convert to a red-black tree.
- `MIN_TREEIFY_CAPACITY = 64` — but only if the table length is at least 64. **If the table is smaller, the map resizes instead of treeifying.** This is the detail most candidates miss: at small table sizes, long chains are usually a symptom of too few buckets, and resizing is the cheaper, more effective fix.
- `UNTREEIFY_THRESHOLD = 6` — when a tree bucket shrinks to 6 or fewer nodes (during a resize/split), it reverts to a linked list. The gap between 8 and 6 provides hysteresis, preventing thrashing between the two representations around a single boundary value.

Converting a bucket to a red-black tree changes worst-case lookup within that bucket from O(n) to O(log n). The motivation was security as much as performance: hash-collision denial-of-service attacks, where an attacker submits many keys engineered to collide (trivially achievable for `String` if the attacker knows the algorithm, which is public), degrading `HashMap` operations to linear and turning an O(n) request handler into O(n²). Treeification bounds that damage. The JDK's `String.hashCode` is not randomised, so this mitigation matters.

The thresholds are chosen with reference to the Poisson distribution: with a load factor of 0.75 and well-distributed hashes, the probability of a bucket reaching 8 entries is roughly 6 in 100 million. So under normal conditions treeification never happens — it is a defence against pathological input, not a normal operating mode.

**Resize.** When `size > capacity * loadFactor`, the table doubles. Java 8 introduced a clever optimisation: because capacity is a power of two and doubles, each entry's new index is either its old index `i` or `i + oldCapacity`, determined by a single bit — `(hash & oldCapacity) == 0`. So a chain can be split into two chains with no rehashing and no modulo, preserving relative order. Tree bins are split the same way and untreeified if either half falls to 6 or fewer.

Resize is O(n) and, critically, is *not* amortised away from a latency perspective: a single `put` can trigger a full table rebuild. For large maps built in a known-size loop, pre-size with `new HashMap<>(expectedSize / 0.75f + 1)` to avoid repeated resizes. (Java 19 added `HashMap.newHashMap(int numMappings)` which does this arithmetic for you — a useful detail because the constructor argument is *capacity*, not expected entry count, which is a very common misunderstanding.)

### Probes

**Treeification threshold and untreeify.** Covered above: 8 to treeify, 6 to untreeify, 64 minimum table size. The hysteresis gap and the resize-instead-of-treeify rule are the two points that distinguish a memorised answer from an understood one.

**Why `Comparable` keys matter for the tree.** A red-black tree needs a total ordering. `HashMap`'s tree bins order primarily by hash. When two keys have *equal* hashes and are not `equals`, the map needs a tiebreaker to place them deterministically. It first checks whether the key class implements `Comparable` (via `comparableClassFor`, which also verifies the class's `Comparable` type parameter is the class itself, guarding against odd generic hierarchies) and uses `compareTo`. If the key is not `Comparable`, it falls back to `tieBreakOrder`, which compares `System.identityHashCode` values — an arbitrary but consistent-within-a-run ordering.

The consequence: a non-`Comparable` key type with a pathological `hashCode` gets a tree that is correctly balanced but cannot be *searched* by ordering alone — the code must scan both subtrees when hashes are equal and ordering is arbitrary. So the O(log n) guarantee degrades toward O(n) for keys that are neither well-distributed nor `Comparable`. `String` is `Comparable`, which is why the attack mitigation works well for the most commonly attacked key type.

**Resize/rehash cost.** Discussed above. The concrete numbers: 16 → resize at 13 entries → 32 → resize at 25 → 64 → resize at 49, and so on. Building a 1,000,000-entry map from an empty `HashMap` performs about 17 resizes and copies roughly 2,000,000 node references in total. Pre-sizing eliminates this. In a latency-sensitive path, the resize appears as an outlier in the p99.9 that is invisible in the mean.

**The hash spreading function (`h ^ (h >>> 16)`) and why.** The bucket index uses only the low bits of the hash: `hash & (n-1)` with `n = 16` uses only the low 4 bits. Many real-world `hashCode` implementations have poor entropy in the low bits and good entropy in the high bits. The canonical example is `Integer.hashCode()`, which returns the value itself — so keys 0, 65536, 131072 differ only in high bits and all map to bucket 0 in a 16-bucket table. Similarly, `Float.hashCode` and hash codes derived from memory addresses or sequential IDs cluster in the low bits.

XOR-ing the top 16 bits down mixes high-bit entropy into the region that actually selects the bucket. It is deliberately a *single* cheap operation — the JDK comment notes this is a trade-off between speed, utility, and bit-spreading quality, accepting that it is a weaker mix than a full avalanche function because most hash codes are already reasonable and treeification handles the rest.

**Why `HashMap` is unsafe under concurrent writes.** Two distinct failure modes, and knowing that they differ by version is the mark of someone who has actually read the source:

*Java 7 and earlier — infinite loop.* Resize transferred entries using **head insertion**, which reversed each chain. Two threads resizing concurrently could interleave such that a chain became circular (node A pointing to B and B back to A). A subsequent `get` on that bucket would loop forever. The symptom in production was a thread pegged at 100% CPU inside `HashMap.get`, sometimes years after the code was written, on a map someone assumed was single-threaded. This is one of the most famous production Java bugs.

*Java 8 and later — no infinite loop, but still broken.* The split-preserving-order resize removed the cycle formation. But `HashMap` is still not thread-safe: concurrent `put`s can lose entries (two threads both read the same bucket head and both write, one overwriting the other), `size` can drift and become permanently wrong, and a thread can observe a partially-constructed table (there is no synchronisation or `volatile` on `table`, so there is no happens-before edge — one thread's writes may never become visible to another). Resize during concurrent access can also produce `ConcurrentModificationException` or, worse, silently missing entries.

The correct answers are `ConcurrentHashMap` for concurrent mutation, `Collections.synchronizedMap` only when you also need `null` keys/values or must wrap an existing map (and remember it requires manual synchronisation around iteration), or an immutable map published safely if the contents never change after construction.

---

## Q4. What guarantees does `volatile` give you that a plain field doesn't? What does it not give you?

### Answer

**What it gives you.**

1. **Visibility (atomicity of access).** A write to a `volatile` field is immediately visible to any thread that subsequently reads it. Without `volatile`, the JMM permits a thread to cache a field's value in a register or a per-core cache line indefinitely — a reader may *never* observe another thread's write. The canonical demonstration is a `boolean running` flag: without `volatile`, a spin loop `while (running) {}` can be hoisted by the JIT into `if (running) while(true) {}`, and the loop never terminates even after another thread sets `running = false`. This is not theoretical; it is a standard JIT optimisation that is legal precisely because the field is not volatile.

2. **Ordering (happens-before).** A `volatile` write *happens-before* every subsequent `volatile` read of the same field. Crucially, this edge carries **everything the writing thread did before the write**, not just the volatile field itself. If thread A writes several plain fields and then writes a volatile flag, and thread B reads that volatile flag and sees the new value, B is guaranteed to see all of A's earlier plain writes. This is why volatile is usable as a publication mechanism for entire object graphs.

3. **No tearing for `long` and `double`.** The JMM permits non-atomic treatment of 64-bit primitives — a plain `long` write may be performed as two 32-bit writes, and a reader may observe a mix of the high word from one write and the low word from another. Declaring the field `volatile` guarantees the read/write is atomic. (On 64-bit HotSpot this is atomic in practice regardless, but that is an implementation detail, not a specification guarantee.)

4. **Prevention of certain compiler/hardware reorderings.** The compiler must not move ordinary reads/writes across a volatile access in ways that would break the above. On hardware, this is implemented with memory barriers/fences (on x86 a volatile write typically compiles to a store followed by a `lock`-prefixed instruction acting as a full fence; on weaker architectures like ARM/POWER, explicit barrier instructions are required, which is why concurrency bugs sometimes appear only on ARM).

**What it does not give you.**

1. **Atomicity of compound operations.** `count++` is read-modify-write: three operations. Two threads can both read 5, both compute 6, both write 6. Volatile guarantees each individual read and each individual write is visible and atomic — it does not make the sequence atomic. Same for `if (x == null) x = new Thing()`, `x = x + 1`, `if (!set) set = true`.

2. **Mutual exclusion.** No lock is taken. Multiple threads execute concurrently.

3. **Protection of the referenced object.** `volatile List<String> list` makes the *reference* volatile. `list.add("x")` is entirely unprotected — you have a volatile reference to a non-thread-safe object, which is one of the most common misunderstandings. Similarly, `volatile int[] arr` does nothing for `arr[0] = 5`; element accesses are plain. (`AtomicIntegerArray` or `VarHandle` exist for that.)

4. **Any guarantee across *different* volatile fields beyond program order.** Two volatile fields are each individually ordered, but you cannot build arbitrary invariants across them without additional synchronisation.

**Cost.** A volatile read on x86 is essentially free (an ordinary load; the ordering is provided by the hardware's memory model). A volatile write is more expensive — it requires a fence that prevents subsequent loads from being reordered before it, typically tens of cycles, and it forces store buffer drainage. It also inhibits JIT optimisations: the field cannot be hoisted out of a loop or kept in a register. So volatile is cheap to read, moderately expensive to write, and free of lock contention entirely.

### Probes

**Visibility + ordering (happens-before).** Covered above. The point worth emphasising in an interview is that people learn volatile as "visibility" and stop there, missing that the *ordering* guarantee is what makes it useful for safe publication of other data. The visibility-only mental model leads to the belief that volatile is only good for flags.

**No atomicity for compound ops.** Covered. The interviewer's likely follow-up: "so how do you make a counter thread-safe?" Answers, in increasing order of contention tolerance: `synchronized`; `AtomicInteger` (CAS-based, lock-free, but degrades under heavy contention because failed CAS attempts spin); `LongAdder` (maintains per-thread cells and sums on read — dramatically better under high write contention, at the cost of a slower and non-atomic `sum()`, so it suits statistics/metrics where you write often and read rarely).

**Memory barriers.** The JMM is specified abstractly in terms of happens-before; barriers are the implementation. HotSpot conceptually emits `LoadLoad`/`LoadStore` after a volatile read and `StoreStore` before / `StoreLoad` after a volatile write. `StoreLoad` is the expensive one and is the reason volatile writes cost more than volatile reads. You do not need to recite this, but knowing that "volatile write is expensive, volatile read is nearly free, and the asymmetry comes from `StoreLoad`" is a credible level of detail.

**Double-checked locking and why the `volatile` is required.**

```java
private volatile Helper helper;   // volatile is MANDATORY

public Helper getHelper() {
    Helper result = helper;                // 1 read
    if (result == null) {
        synchronized (this) {
            result = helper;
            if (result == null) {
                helper = result = new Helper();
            }
        }
    }
    return result;
}
```

Without `volatile`, this is broken. `new Helper()` is not one operation — it is (a) allocate memory, (b) run the constructor, (c) assign the reference to the field. The JIT and the hardware are permitted to reorder (b) and (c), because within a single thread the reordering is unobservable. Another thread executing the first, unsynchronised `if (helper == null)` check can therefore observe a **non-null reference to a partially constructed object** — fields still at their default values. It then returns and uses that object. This is why the pattern was declared broken before Java 5; the Java 5 JMM revision made it correct *provided the field is volatile*, because the volatile write forbids the reordering and the volatile read establishes the happens-before edge.

The `result` local variable is a real optimisation, not noise: it reduces volatile reads from two to one on the common (already-initialised) path, which measurably matters in hot code.

That said, for a static singleton the **initialization-on-demand holder idiom** is simpler and equally correct, relying on the JVM's guarantee that class initialisation is thread-safe and lazy:

```java
private static class Holder { static final Helper INSTANCE = new Helper(); }
public static Helper getHelper() { return Holder.INSTANCE; }
```

No volatile, no synchronisation cost after the first access, no subtlety. Prefer this. Double-checked locking is only necessary for *instance* fields or when the value depends on runtime parameters.

**When `AtomicInteger` is the right answer instead.** Whenever you need read-modify-write atomicity rather than just visibility: counters, sequence generators, CAS-based state machines (`compareAndSet` to transition state exactly once), accumulating results. Also `AtomicReference` for lock-free swap of an immutable snapshot (read the current reference, compute a new immutable value, CAS it in, retry on failure — a very useful pattern for configuration hot-reload). Use `volatile` when the only requirement is publishing a value that one thread writes and others read.

---

## Q5. Define happens-before. Give me three different ways to establish it besides `synchronized`.

### Answer

**Definition.** Happens-before is a partial order over the actions in a program, defined by the Java Memory Model (JSR-133, specified in JLS Chapter 17). If action A happens-before action B, then:

- the memory effects of A are **visible** to B, and
- A is **ordered before** B from the perspective of any thread that can observe both.

The critical framing: happens-before is **not** about wall-clock time. "A happened earlier" does not imply "A happens-before B". Two actions in different threads with no synchronisation between them are *unordered* regardless of when they occur, and the JMM permits each thread to observe them in either order or not at all.

The corollary that matters: **if two accesses to the same non-final, non-volatile variable are not ordered by happens-before, and at least one is a write, that is a data race.** A program containing a data race has no meaningful guarantees at all about the values observed — the JMM only promises sequentially consistent behaviour for *correctly synchronised* (race-free) programs. This is why "it works on my machine" is not evidence of correctness for concurrent code: a racy program can behave correctly on x86 for years and fail on ARM, or fail after a JIT recompilation changes the optimisation applied.

Happens-before is **transitive**: if A hb B and B hb C, then A hb C. Almost all real reasoning chains three or more edges together.

**Program order** gives happens-before within a single thread: earlier statements happen-before later ones *as observed by that thread*. Note the qualification — the JIT and CPU freely reorder within a thread as long as the single-threaded result is unchanged. Program order only becomes globally meaningful when combined with a cross-thread edge.

**Ways to establish it besides `synchronized`:**

**1. `volatile` write → read.** A write to a volatile field happens-before every subsequent read of that field. Combined with transitivity and program order, this publishes everything the writer did beforehand:

```java
// Thread A
config = new Config(...);   // plain writes inside the constructor
ready = true;               // volatile write

// Thread B
if (ready) {                // volatile read; if it sees true...
    use(config);            // ...then all of A's earlier writes are visible
}
```

**2. `Thread.start()` and `Thread.join()`.** Everything a thread does before calling `t.start()` happens-before any action in `t`. Symmetrically, every action in `t` happens-before `t.join()` returns in the joining thread. This is why you can populate a data structure, start a thread that reads it, and need no other synchronisation — and why you can have a thread compute a result into a plain field, `join()` it, and read the field safely.

**3. `final` field semantics.** When an object with `final` fields is constructed and the `this` reference does not escape during construction, any thread that obtains a reference to that object — *even through a data race* — is guaranteed to see the correctly initialised final fields. This is the "freeze action" at the end of the constructor. It is the reason immutable objects can be shared without synchronisation. Note the precondition about `this` escaping: it is not decorative (see below).

**4. `java.util.concurrent` mechanisms.** The `java.util.concurrent` package documents happens-before for essentially all of its classes:
- Actions before placing an object into a concurrent collection happen-before actions after removing it (`BlockingQueue.put` → `take`, `ConcurrentHashMap.put` → `get`).
- `CountDownLatch.countDown()` happens-before a returning `await()`.
- `Semaphore.release()` happens-before a corresponding `acquire()`.
- `Lock.unlock()` happens-before a subsequent `lock()` on the same lock.
- Submitting a `Runnable`/`Callable` to an `Executor` happens-before its execution begins; the task's completion happens-before `Future.get()` returns.
- `CyclicBarrier` and `Phaser` arrival happens-before actions after the barrier trips.

**5. Class initialisation.** The JVM guarantees static initialisers run exactly once under an internal lock, and the effects are visible to every thread that subsequently accesses the class. This is the basis of the holder idiom.

**6. Interruption.** `Thread.interrupt()` happens-before the interrupted thread detecting the interrupt (via `InterruptedException` or `isInterrupted()`).

### Probes

**`volatile` write → read.** Covered above, with the important note that the edge carries *all prior writes*, not just the volatile field.

**`Thread.start()`/`join()`.** Covered. A practical implication: a common "fix" for a visibility bug is to `join()` the thread and read the result — that is genuinely correct, not a coincidence. Another: the fields you set on a `Thread` subclass or a `Runnable` *before* `start()` are safely visible to the new thread, so constructor-injected state needs no synchronisation.

**Final field freeze semantics.** The guarantee is unusual because it holds *even for unsafely published objects*. `String` relies on this — its backing array is final, so a `String` reference shared through a data race still shows correct contents. The precondition — `this` must not escape the constructor — is easy to violate:

```java
public class Broken {
    private final int x;
    public Broken(Registry r) {
        r.register(this);   // 'this' escapes BEFORE the freeze
        this.x = 42;        // another thread may observe x == 0
    }
}
```

Also common: starting a thread in a constructor, registering an event listener in a constructor, or calling an overridable method from a constructor (the subclass's override runs before the subclass's fields are initialised — a distinct but related hazard). The rule is simply: do not let `this` escape during construction. Use a static factory that constructs fully, then registers.

Note that the guarantee applies to the final field itself and to anything reachable from it *at the time of the freeze*. A final reference to a mutable `ArrayList` protects the reference, not subsequent mutations of the list.

**`CountDownLatch`.** `countDown()` in any thread happens-before `await()` returns in a waiting thread. Useful for one-shot startup coordination and for deterministic concurrency tests (make N threads await a shared start latch to maximise the chance of an actual race). Note it is one-shot; `CyclicBarrier` is the reusable equivalent.

**Executor submit → execute.** Formally: actions in a thread prior to submitting a task happen-before the task begins executing; and the task's actions happen-before `Future.get()` returns or before another thread observes completion via `isDone()`/callback. This is why you can build a request object, submit it, and have the worker read it without any synchronisation on the object — and why the result written by a worker into an ordinary field of the task object is safely readable after `get()`.

**`ConcurrentHashMap` operations.** `put` (or any insertion) happens-before a subsequent `get` that returns that value. This makes CHM a legitimate publication mechanism between threads, not merely a race-free container.

---

## Q6. What is `final` doing for you in a multithreaded context, in terms of safe publication?

### Answer

**Safe publication** means making an object visible to other threads in a state where they can observe it correctly and completely. The problem it solves: object construction is not atomic. Allocation, field initialisation, and reference assignment are separate operations that the compiler and CPU may reorder. Another thread can obtain a non-null reference to an object whose fields still hold default values (`0`, `null`, `false`).

The JMM defines four safe publication idioms (Goetz's *Java Concurrency in Practice* enumerates these):

1. Initialising the reference from a **static initialiser** (class-init lock provides the edge).
2. Storing the reference into a **`volatile` field** or an `AtomicReference`.
3. Storing the reference into a **`final` field** of a properly constructed object.
4. Storing the reference into a field **guarded by a lock** (write and read both under the same lock).

**What `final` specifically provides.** The JMM's final-field semantics (introduced in the JSR-133 revision for Java 5) specify a *freeze action* at the end of every constructor that writes a final field. The rule:

> If a thread obtains a reference to an object *after* that object's constructor has completed, then that thread is guaranteed to see the correctly initialised values of the object's final fields — **and of anything reachable from those final fields at the time of the freeze** — without any synchronisation.

The "without any synchronisation" part is the remarkable bit. This guarantee holds **even if the reference is published through a data race**. Every other visibility guarantee in the JMM requires a happens-before edge; this one does not. It is implemented by forbidding the reordering of final-field writes past the constructor's end (conceptually a `StoreStore` barrier before the constructor returns) and, on architectures with speculative address loads, a corresponding barrier on the reading side.

**Why this matters in practice.** It is what makes immutable objects genuinely free to share. `String`, `Integer`, `BigDecimal`, `LocalDate`, records, and any properly built immutable value class can be passed between threads with no locks, no volatile, no cost. It underpins the entire "make it immutable and stop worrying" strategy for concurrent design — which is by far the cheapest correct answer to most shared-state problems.

The reachability clause matters too: if a final field points to an array or a `List` that you populated *inside the constructor*, the contents of that array or list at freeze time are also guaranteed visible. That is why `List.copyOf(input)` assigned to a final field in a constructor produces a genuinely safely-publishable object.

**The limits.**

- `final` protects the *reference*, and the state reachable through it *as of the freeze*. Later mutations of a mutable object held in a final field are ordinary writes with no guarantees. `private final List<String> items = new ArrayList<>();` followed by `items.add(...)` from multiple threads is a data race.
- The guarantee requires the object to be **properly constructed** — `this` must not escape. If a reference to the object becomes visible to another thread before the constructor finishes, all bets are off.
- Non-final fields of the same object get **no** guarantee. Mixing final and non-final fields means the final ones are safe and the others are not, which is a subtle trap: the object *looks* safely published because it worked in testing.
- `final` on a local variable or parameter has no memory-model meaning whatsoever; it is purely a compile-time immutability check (and, historically, a requirement for lambda/anonymous-class capture, now relaxed to "effectively final").

### Probes

**Final field semantics guarantee correctly-constructed immutable objects are visible without synchronisation.** Covered above. The interview-grade phrasing: "final fields give you safe publication for free, including through an unsafe publication path, provided `this` never escapes the constructor."

A worked example of why this is not academic:

```java
class Config {
    private final Map<String, String> settings;
    Config(Map<String, String> in) { this.settings = Map.copyOf(in); }
    String get(String k) { return settings.get(k); }
}

// somewhere, no synchronisation at all:
static Config current;              // plain static field!
current = new Config(loaded);       // unsafe publication of the *reference*
```

A reader thread might see `current` as `null` (there is no visibility guarantee on the plain static field), but if it sees a non-null reference, that `Config` is guaranteed fully initialised — `settings` will never be observed as `null` or partially built. In practice you would still make `current` `volatile` so readers reliably see the update; the final-field guarantee protects you from the far nastier failure of seeing a *broken* object.

**The "`this` escapes during construction" bug that breaks it.** Concrete forms, all of which appear in real codebases:

```java
// 1. Registering with an observer/registry
public Service(EventBus bus) {
    bus.subscribe(this);      // another thread may now call our methods
    this.handler = new Handler();   // ...before this line runs
}

// 2. Starting a thread
public Worker() {
    this.thread = new Thread(this);
    thread.start();           // new thread sees a half-built Worker
    this.queue = new LinkedBlockingQueue<>();
}

// 3. Anonymous/inner class capturing 'this'
public Cache() {
    scheduler.scheduleAtFixedRate(() -> this.evict(), 0, 1, MINUTES);
    this.entries = new ConcurrentHashMap<>();   // evict() may NPE
}

// 4. Calling an overridable method
public Base() { init(); }     // subclass override runs before subclass fields init
```

The fix in every case is the same shape: complete construction, then perform the escaping action separately. A static factory method is the idiomatic vehicle:

```java
public static Service create(EventBus bus) {
    Service s = new Service();      // fully constructed, frozen
    bus.subscribe(s);               // now safe
    return s;
}
```

Or, in a Spring context, do the registration in `@PostConstruct` / `afterPropertiesSet()` rather than the constructor — which is one of the real reasons those lifecycle callbacks exist, beyond dependency availability. Note also that case 4 is why calling overridable methods from constructors is flagged by every static analyser, and why Spring's own initialisation is split into construct-then-callback phases.

---

## Q7. Compare G1, Parallel, and ZGC/Shenandoah. How would you choose for (a) a batch job, (b) a low-latency API, (c) a 512 MB container?

### Answer

**Parallel GC (`-XX:+UseParallelGC`).** Generational, stop-the-world, multi-threaded, compacting. Young collections copy survivors between survivor spaces and promote to old; old collections are a full stop-the-world mark-sweep-compact. It does the least bookkeeping of any collector — no concurrent phases, no write barriers of consequence — so it achieves the **highest raw throughput** (most application work per unit of CPU) at the cost of pauses proportional to heap size. A full GC on a 16 GB old generation can pause for many seconds.

**G1 (`-XX:+UseG1GC`, default since JDK 9).** Divides the heap into equal-sized regions (typically 1–32 MB, chosen so there are ~2048 regions) which are dynamically labelled Eden, Survivor, Old, or Humongous. Marking is concurrent; evacuation (copying) is stop-the-world but **incremental** — G1 chooses a *collection set* of regions to evacuate, prioritising those with the most garbage ("garbage first"), sized to fit a soft pause target (`-XX:MaxGCPauseMillis`, default 200 ms). Because it copies, it compacts as it goes and does not suffer fragmentation the way CMS did. It maintains remembered sets and a card table via write barriers, which is a real throughput tax (commonly cited around 10–15% versus Parallel, workload-dependent).

**ZGC (`-XX:+UseZGC`) and Shenandoah (`-XX:+UseShenandoahGC`).** Concurrent compacting collectors targeting sub-millisecond pauses **independent of heap size**. Nearly all work — marking, relocation, reference processing — happens while the application runs. Pauses are limited to short root-scanning operations.

- **ZGC** uses coloured pointers: metadata bits stored inside the 64-bit reference itself (mark state, remap state), combined with **load barriers**. When the application loads a reference, the barrier checks the colour bits; if the object has been relocated, the barrier fixes up the reference on the spot ("self-healing"). This makes concurrent relocation safe without stopping the application. ZGC is 64-bit only. Since JDK 21 it has a **generational** mode (`-XX:+ZGenerational`), which was the major missing piece — the original non-generational ZGC had to scan the whole heap every cycle, making it expensive for workloads with high allocation rates and short-lived objects. In JDK 23 generational became the default and non-generational was deprecated.
- **Shenandoah** (Red Hat; present in OpenJDK builds, may be absent from some vendor builds) uses Brooks forwarding pointers plus load-reference barriers to achieve similar concurrent relocation. Practically it occupies the same niche; the choice is often driven by which is better supported in your JDK distribution.

Both trade **throughput and memory** for pause time. Expect a throughput cost versus Parallel (often quoted at 10–20%, but measure your workload) and materially higher heap overhead — concurrent collectors must have headroom to allocate while collecting, and running them near the heap ceiling causes allocation stalls, which are pauses by another name.

**Serial GC (`-XX:+UseSerialGC`)** deserves mention as the fourth option: single-threaded, minimal footprint, minimal startup cost. It is the right answer more often than people expect for small containers and short-lived processes.

**Choosing for the three scenarios:**

**(a) Batch job.** Pauses are irrelevant; total wall-clock time is everything. **Parallel GC**, with a generously sized heap to reduce collection frequency. Give it all the CPUs. If the job streams large volumes of short-lived objects, tune young generation size upward — most objects die young, and a bigger Eden means fewer, more efficient young collections. Do not use ZGC here: you would pay concurrent-collection overhead for a property you do not need.

**(b) Low-latency API.** Depends on how low. If your SLO is a p99 in the hundreds of milliseconds and your heap is a few GB, **G1 with a tuned `MaxGCPauseMillis`** is usually sufficient and is the path of least resistance (it is the default, it is the best-understood, and there is more operational knowledge available). If your SLO is single-digit-millisecond p99.9, or your heap is tens to hundreds of GB where G1's evacuation pauses grow, **generational ZGC**. Be explicit about the trade: you will need more heap and you will lose some throughput, meaning more instances for the same request rate. Also verify that GC is actually your latency problem — in most services, p99 latency is dominated by downstream calls, lock contention, connection pool waits, or CPU throttling, not by GC. Measure before switching collectors.

**(c) 512 MB container.** **Serial GC** is a serious candidate and frequently the right one. At this size, a full GC is fast anyway, and Serial has the smallest memory and CPU footprint — no GC worker threads, no remembered sets, no concurrent threads competing for the one or two CPUs you likely have. G1's per-region bookkeeping and its ~2048-region target produce small regions and relatively high overhead at this heap size. ZGC is a poor fit: its footprint overhead is significant relative to 512 MB and it needs headroom.

Note that the JVM's **ergonomics** already do something like this: on a machine (or container, with `UseContainerSupport` active) with fewer than 2 CPUs *or* less than ~1792 MB of memory, HotSpot classifies it as a "client-class" machine and selects Serial GC automatically. So a 512 MB container with 1 CPU already gets Serial without you asking. The interview-relevant point is knowing *why*, and knowing that a container with 2 CPUs and 1 GB might get G1 when Serial would serve better.

### Probes

**Throughput vs pause time.** The fundamental trade. Throughput = fraction of CPU time spent on application work. Pause time = duration of individual stop-the-world events. You can generally have low pauses or high throughput, not both, because achieving low pauses requires doing work concurrently with the application, and concurrent work requires barriers, coordination, and re-scanning of mutated state — all of which is overhead. A third axis is **footprint**: concurrent collectors need spare heap. Any GC choice is a point in this three-way trade-off.

**Region-based collection.** G1's core structural idea. Benefits: the collector can choose *which* regions to collect rather than being forced to collect an entire generation, letting it bound pause time by bounding the collection set; region granularity also allows generation boundaries to move dynamically instead of being fixed at startup. Cost: remembered sets tracking cross-region references, maintained by write barriers on every reference store, plus concurrent refinement threads processing dirty cards.

**Concurrent marking.** All of G1, ZGC, and Shenandoah mark concurrently, which requires solving the problem of the application mutating the object graph mid-mark. The standard solution is a **snapshot-at-the-beginning (SATB)** invariant (G1, Shenandoah): a pre-write barrier records the *previous* referent of any overwritten reference so that objects reachable at the start of marking are not lost. The consequence is **floating garbage** — objects that become unreachable during the cycle are not collected until the next one, which is part of why concurrent collectors need more heap headroom. ZGC uses load barriers and colour-based marking instead, achieving a similar guarantee from the read side.

**Humongous allocations in G1.** An object larger than **half a region** is "humongous" and is allocated directly into old-generation regions, contiguously spanning as many regions as needed. Consequences:

- Space at the end of the last region is wasted (an object of 1.1 regions occupies 2 regions).
- Humongous regions historically could only be reclaimed at certain points, so a churn of large arrays could drive frequent full/mixed collections. Modern G1 has improved this (humongous regions with no references can be eagerly reclaimed at young collections) but the risk remains.
- Finding contiguous free regions can fail even when total free space is adequate — a fragmentation failure that manifests as an unexpected full GC.

**Symptom to recognise:** frequent full GCs with plenty of apparent free heap, plus `Humongous` entries in GC logs. **Causes:** large `byte[]`/`char[]` buffers, big `String`s, large collections' backing arrays, protobuf/JSON buffers. **Fixes:** increase `-XX:G1HeapRegionSize` (must be a power of two between 1 MB and 32 MB) so the objects are no longer humongous, or restructure the code to avoid large contiguous allocations (chunked/streaming processing, pooled direct buffers).

**ZGC's coloured pointers/load barriers and its memory overhead.** Coloured pointers store GC metadata in unused high bits of the 64-bit reference. The load barrier is inserted on every reference read; it tests the colour and, if stale, remaps and heals the reference before the application sees it. Costs: (1) the barrier itself, on every reference load, which is why ZGC's throughput is lower; (2) the address-space technique historically used **multi-mapping** — mapping the same physical heap page at several virtual addresses — which makes tools like `ps`/`top` report inflated virtual (and sometimes resident) memory, a genuine operational confusion in containers; (3) real extra heap headroom for concurrent relocation. Also, because ZGC uses pointer bits, **compressed oops are not available**, so every reference is 8 bytes rather than 4 — a substantial footprint increase for reference-dense heaps under 32 GB, and a strong argument against ZGC for small heaps.

**Why default GC choice changes with container CPU/heap size.** As above: HotSpot's ergonomics inspect available processors and memory (container-aware since JDK 10 / 8u191) and pick Serial for "client-class" machines, G1 otherwise. This is why the same image behaves differently in a 1-CPU pod and a 4-CPU pod, and why setting CPU limits in Kubernetes silently changes your GC, your GC thread count (`-XX:ParallelGCThreads`), your JIT compiler thread count, and the default `ForkJoinPool.commonPool` parallelism. If you want deterministic behaviour across environments, set the collector and thread counts explicitly rather than relying on ergonomics.

---

## Q8. Your service has no memory leak in the Java sense — no growing live set — but RSS keeps climbing and the pod gets OOMKilled. Where do you look?

### Answer

The premise is that heap is stable (confirmed via `jstat -gcutil`, GC logs showing post-collection live set flat, or a heap dump comparison). Yet **RSS** — the resident set size the kernel accounts against the cgroup limit — grows until the OOM killer fires. Kubernetes kills on *container* memory, which is total RSS plus page cache attributable to the cgroup, not on Java heap. So the answer lives outside the heap.

**Systematic breakdown of a JVM process's memory:**

```
RSS ≈ Java heap
    + Metaspace + compressed class space
    + Code cache (JIT-compiled code)
    + Thread stacks (n_threads × -Xss)
    + GC internal structures (card tables, remembered sets, mark bitmaps)
    + Direct/mapped ByteBuffers (off-heap NIO)
    + Native allocations by libraries (JNI, native compression, TLS, etc.)
    + Native allocator overhead/fragmentation (glibc arenas)
    + Symbol/string tables, internal JVM structures
```

**The investigation sequence:**

**1. Establish the baseline split.** Enable Native Memory Tracking: `-XX:NativeMemoryTracking=summary` (roughly 5–10% overhead) and then `jcmd <pid> VM.native_memory summary` — or `summary.diff` after a baseline, which is far more useful because it shows *growth* by category. This immediately tells you whether the growth is Metaspace, Thread, Code, GC, Internal, or Other. NMT does *not* track allocations made by native libraries outside the JVM's own allocator, so "Other/Unknown" growth points toward JNI or a native library.

**2. Compare heap ceiling to container limit.** A very common cause is not a leak at all but **arithmetic**: `-Xmx` (or the default `MaxRAMPercentage`, historically 25%, so a 2 GB container defaults to a 512 MB heap) plus all the non-heap regions simply exceeds the limit under load. The heap grows to its legitimate maximum over hours as traffic ramps, non-heap grows with thread count and JIT compilation, and the sum crosses the limit. This looks exactly like a leak on a graph. Rule of thumb: reserve 25–35% of the container limit for non-heap; more for thread-heavy or off-heap-heavy applications.

**3. Metaspace.** Unbounded by default in the sense that `MaxMetaspaceSize` is unlimited unless set. Growth indicates class loading without unloading. Causes: dynamic proxy generation (Spring AOP, Hibernate, mocking frameworks in tests), scripting engines (Groovy/JavaScript generating classes per script evaluation), repeated deployment in an app server, bytecode-generating libraries, or a classloader leak. Set `-XX:MaxMetaspaceSize` so you get a clear `OutOfMemoryError: Metaspace` rather than a silent container kill — a diagnosable failure is worth a lot.

**4. Threads.** Each platform thread reserves stack (`-Xss`, default typically 1 MB on 64-bit Linux; committed lazily but often fully touched). A thread leak — creating executors per request, un-shut-down `ExecutorService`s, a library spawning threads per connection — grows RSS linearly and invisibly to heap monitoring. Check `Thread.activeCount()`, a thread dump, or NMT's Thread category. Threads also each carry JVM-internal metadata and thread-local allocation buffers.

**5. Code cache.** Bounded by `ReservedCodeCacheSize` (default 240 MB with tiered compilation). It grows during warmup and then plateaus; continued growth suggests heavy dynamic class generation causing repeated compilation, or code cache flushing thrash. Rarely the culprit alone but contributes.

**6. Direct / mapped buffers.** `ByteBuffer.allocateDirect` allocates outside the heap and is freed only when the buffer object is collected (via a `Cleaner`). If those buffers are held by long-lived objects, or if GC pressure is low so collection is infrequent, native memory accumulates. Netty, Kafka clients, gRPC, NIO-based servers, and compression libraries all use direct buffers heavily. `-XX:MaxDirectMemorySize` bounds it (defaults to `-Xmx` if unset) and converts a silent container kill into an `OutOfMemoryError: Direct buffer memory`. Netty additionally has its own pooled allocator with its own leak detector (`-Dio.netty.leakDetection.level=paranoid` in a test environment).

**7. Native libraries and JNI.** Anything calling into C: native compression (zstd, snappy), some crypto providers, native database drivers, JNA-based code, image processing. These allocate with `malloc` and are invisible to NMT. Diagnose with `jemalloc`/`tcmalloc` profiling (`MALLOC_CONF=prof:true` with jemalloc, then `jeprof`) or `perf`/`bpftrace`. This is the hardest category.

**8. Allocator fragmentation.** glibc's `malloc` creates per-thread **arenas** (up to `8 × ncores` on 64-bit) to reduce lock contention. Each arena retains freed memory rather than returning it to the OS, so a many-threaded JVM can hold a large amount of freed-but-unreturned native memory. Symptom: RSS plateaus far above what NMT accounts for, and does not shrink after load subsides. Mitigations: `MALLOC_ARENA_MAX=2` (or 4) as an environment variable, or switch the allocator entirely to jemalloc/tcmalloc via `LD_PRELOAD` — both are standard practice for JVMs in memory-constrained containers.

**9. Page cache.** In cgroups v1, page cache generated by the container's file I/O counts toward the memory limit and can trigger reclaim/OOM. An application writing large log files or reading large files can push the cgroup over. cgroups v2 accounting differs but file cache still participates. Check `memory.stat` for `cache`/`file` versus `rss`.

**10. Classloader leaks.** A retained classloader retains every class it loaded and their static fields — invisible in a naive heap analysis if you only look at instance counts, and it inflates Metaspace. Classic causes: a `ThreadLocal` on a container-managed thread holding an application class; a JDBC driver registered in `DriverManager`; a shutdown hook; a running timer thread.

**The pragmatic path:** set `-XX:MaxMetaspaceSize`, `-XX:MaxDirectMemorySize`, and an explicit `-Xmx`, and set `-XX:+HeapDumpOnOutOfMemoryError`. This converts an opaque OOMKill into a specific, named `OutOfMemoryError` that tells you which region overflowed. Then enable NMT with a baseline and diff over time. That two-step usually identifies the category within one incident cycle.

### Probes

**Non-heap: metaspace, code cache, thread stacks, direct/`ByteBuffer` off-heap, GC native structures, glibc arena fragmentation.** All covered above. One addition on **GC native structures**: G1's remembered sets and mark bitmaps scale with heap size and with the density of cross-region references; a heap with many old→young references can have surprisingly large remembered sets. NMT's "GC" category shows this. ZGC's structures are also significant.

**Native Memory Tracking.** `-XX:NativeMemoryTracking=summary|detail`. `detail` gives call-site attribution but costs more. The workflow that actually works: `jcmd <pid> VM.native_memory baseline` early, then `jcmd <pid> VM.native_memory summary.diff` hours later. Absolute numbers are hard to interpret; deltas are not. Note NMT reports *committed* and *reserved* JVM-internal memory and will not sum exactly to RSS, because it does not see third-party native allocations and because reserved-but-untouched pages are not resident.

**`MaxRAMPercentage` vs container limit.** With container support enabled, the JVM reads the cgroup limit and applies `InitialRAMPercentage`/`MinRAMPercentage`/`MaxRAMPercentage`. The naming is confusing: `MinRAMPercentage` applies to *small* physical memory (under ~250 MB) and `MaxRAMPercentage` to larger — they are not a floor and ceiling on the same thing. The default `MaxRAMPercentage` of 25% is conservative and often leaves memory unused; 50–75% is common in practice, tuned by measuring actual non-heap usage. Always prefer percentage flags over hard-coded `-Xmx` in containers so the same image behaves correctly at different limit settings.

**Heap is only part of RSS.** The single most important framing for this question. A dashboard showing "JVM heap used" flat while the pod dies is not a contradiction; it is the expected appearance of every non-heap problem. Monitor container RSS *and* JVM heap *and* the JVM's own non-heap metrics (Micrometer exposes `jvm.memory.used` by area, including Metaspace and code cache) side by side.

**`MALLOC_ARENA_MAX`.** Explained above. Setting it to 2 trades some native allocation contention for substantially lower and more predictable RSS. It is one of the highest-value, lowest-risk settings for containerised JVMs and costs nothing to try.

**cgroups v1 vs v2 differences.** v1 exposes limits at `/sys/fs/cgroup/memory/memory.limit_in_bytes`; v2 uses a unified hierarchy with `memory.max` and `memory.high` (a soft limit that triggers throttling/reclaim rather than a kill). JVM support for v2 arrived in JDK 15 and was backported; **older JDKs on a v2 host may fail to detect limits at all and size the heap from the host's total memory** — a classic cause of instant OOMKills after a Kubernetes node OS upgrade. Verify with `java -XX:+PrintFlagsFinal -version | grep MaxHeapSize` inside the actual container. v2 also accounts memory differently (`memory.current` includes more kernel-side allocations), so identical workloads can show different numbers after a migration.

**Leaked `ThreadLocal`s.** `ThreadLocalMap` uses weak references for keys but **strong references for values**. On a pooled thread that lives forever, a value set during request 1 is retained until the thread dies or someone calls `remove()`. Even after the `ThreadLocal` key is collected, the stale entry's value remains until a subsequent map operation happens to clear it — which is not guaranteed. This leaks heap, and if the value transitively holds a classloader, Metaspace too. Always `remove()` in a `finally` block; framework filters (Spring Security, MDC, transaction context) do this for you, but hand-rolled ones frequently do not.

**Classloader leaks.** Covered above. Diagnose by taking a heap dump and looking for multiple instances of the same application classloader class, then computing the retained set and finding the GC root path. Common in hot-redeploy scenarios and long-lived test JVMs; less common in immutable-container deployments, which is one underappreciated operational benefit of container-per-version deployment.

---

## Q9. Explain strong / soft / weak / phantom references with a real use case for each.

### Answer

The four strengths, from strongest to weakest, control when the collector may reclaim an object.

**Strong reference** — an ordinary reference (`Object o = new Object()`). The object is reachable and will never be collected while the reference is live. Everything is strong unless explicitly wrapped. Almost every memory leak in Java is an unintended strong reference from a long-lived object to a short-lived one — a static collection, a listener list, a cache with no eviction, a `ThreadLocal` value on a pooled thread.

**Soft reference** (`SoftReference<T>`) — collected **at the collector's discretion when memory is tight**, and guaranteed to be cleared before the JVM throws `OutOfMemoryError`. The specification deliberately leaves the policy vague; HotSpot's default heuristic clears softly-reachable objects based on how recently they were accessed and how much free heap remains (governed by `-XX:SoftRefLRUPolicyMSPerMB`, defaulting to 1000 ms of survival per free MB of heap). *Intended* use case: a memory-sensitive cache — hold results softly so they survive while memory allows and evaporate under pressure. **In practice, this is a bad cache design** (see probes).

**Weak reference** (`WeakReference<T>`) — cleared as soon as the object is no longer *strongly* reachable, at the next collection, regardless of memory pressure. Use case: **associating metadata with objects you do not own**, where the association must not keep the object alive. The canonical example is `WeakHashMap`, whose keys are weak — used for per-object caches keyed by identity, or for canonicalising maps. Also used by `ThreadLocal`'s internal map for its keys, and by many frameworks to attach state to user objects without leaking them.

**Phantom reference** (`PhantomReference<T>`) — `get()` **always returns `null`**, by design; you can never resurrect the object. Its sole purpose is to receive a notification, via a `ReferenceQueue`, that an object has become phantom-reachable — meaning it has been finalised (if applicable) and is genuinely ready for reclamation. Use case: **deterministic cleanup of native resources**. When the Java object wrapping a native handle becomes unreachable, you get a queue notification and can free the native memory, close the file descriptor, or unmap the buffer. This is what `java.lang.ref.Cleaner` wraps in a usable API, and what the JDK itself uses for `DirectByteBuffer` deallocation.

**Reachability levels** (JLS/`java.lang.ref` package doc), strongest first: strongly reachable → softly reachable → weakly reachable → finalizer-reachable → phantom-reachable → unreachable. An object is at the strongest level by which it can be reached, so a single strong reference anywhere defeats every weak/soft reference to the same object.

**`ReferenceQueue`** is the common mechanism: register a reference with a queue at construction, and when the collector clears the reference, it enqueues it. A cleanup thread polls the queue. Note carefully that for soft/weak references the referent is cleared *before* enqueueing, so by the time you receive it you cannot access the object — which is why you typically subclass the reference type to carry the cleanup data you need alongside it.

### Probes

**Weak keys in `WeakHashMap` and why the *value* can still pin the key.** `WeakHashMap` holds keys weakly and **values strongly**. If a value refers, directly or transitively, to its own key, the value's strong reference keeps the key strongly reachable, the entry is never cleared, and the map grows forever. This is a real and easily-made bug:

```java
Map<Session, SessionData> map = new WeakHashMap<>();
map.put(session, new SessionData(session));   // value holds the key → never collected
```

The fix is to wrap the value in a `WeakReference` too, or to ensure values never reference keys. Also note `WeakHashMap` only purges cleared entries during subsequent map operations (its `expungeStaleEntries` runs on access), so an idle map does not shrink, and it is **not thread-safe** — there is no concurrent equivalent in the JDK (Guava's `CacheBuilder.weakKeys()` or Caffeine fill that gap, and they also give you *identity* semantics explicitly, whereas `WeakHashMap` uses `equals`, which is often not what you want for identity-based association).

**`ThreadLocal`'s weak key + strong value leak in thread pools.** `Thread` holds a `ThreadLocalMap`. Its `Entry` extends `WeakReference<ThreadLocal<?>>` — the *key* is weak — but the value field is a plain strong reference. Consequences on a pooled thread that lives for the application's lifetime:

- If the `ThreadLocal` object itself becomes unreachable (e.g. it was a local or a field of a discarded object), the key is cleared but the entry remains with a strong value. The map only purges such stale entries opportunistically during `set`/`get`/`remove` operations that happen to encounter them — so a value can be retained indefinitely.
- If the `ThreadLocal` is a `static final` field (the normal, recommended pattern), the key is *never* collected, so the value is retained until the thread dies or `remove()` is called. On a Tomcat/Undertow worker or a fixed thread pool, that is forever.

Failure modes: memory growth proportional to (pool size × retained object size); **cross-request data leakage** where request N sees request N−1's tenant/user/locale, which is a security incident, not just a leak; and classloader retention in redeployable containers. The rule is absolute: every `ThreadLocal.set` in request-scoped code needs a matching `remove()` in a `finally`. Frameworks that do this correctly (`MDC`, Spring's `RequestContextHolder`, `SecurityContextHolder`) do it in filters; hand-rolled context propagation usually does not.

Virtual threads change the calculus: they are cheap and short-lived, so a `ThreadLocal` set on a virtual thread dies with it — but with millions of virtual threads, per-thread copies of a large value become a memory problem of a different shape. `ScopedValue` (a preview feature in recent JDKs) addresses this with immutable, structurally-scoped bindings that are shared rather than copied and are automatically unbound at scope exit.

**`Cleaner` replacing `finalize`.** `Object.finalize()` is deprecated for removal, and rightly so: finalisers run on a single unbounded JDK-managed thread with no ordering or timeliness guarantee; an exception in a finaliser is swallowed; a finaliser can *resurrect* the object by storing `this` somewhere, forcing a second collection cycle; finalisable objects survive at least one extra GC cycle by construction, inflating promotion; and a slow finaliser blocks the queue for every other object. Multiple real production incidents trace to finaliser queue backlog.

`java.lang.ref.Cleaner` (Java 9+) is the replacement, built on `PhantomReference`:

```java
public class NativeResource implements AutoCloseable {
    private static final Cleaner CLEANER = Cleaner.create();
    private final Cleaner.Cleanable cleanable;
    private final long handle;

    public NativeResource() {
        this.handle = allocateNative();
        // NOTE: the cleanup action must NOT capture 'this'
        long h = this.handle;
        this.cleanable = CLEANER.register(this, () -> freeNative(h));
    }

    @Override public void close() { cleanable.clean(); }  // idempotent
}
```

The critical rule: **the cleanup lambda must not capture the object being registered.** If it does, the object is strongly reachable from the cleaning action and is never collected, so the cleaner never runs — a leak that silently disables the very mechanism meant to prevent it. Hence copying `handle` to a local, or using a static nested class holding only the resource state.

Even with `Cleaner`, the primary mechanism should be `AutoCloseable` + try-with-resources. The cleaner is a **safety net** for the case where a caller forgets, not the main path — because you still have no guarantee about *when* it runs.

**Why soft references are a poor cache.** Several reasons, and this is worth stating confidently because it contradicts the textbook description:

1. **The policy is not yours.** Clearing is at the collector's discretion. You cannot express "keep the 1000 most recent" or "expire after 5 minutes" or "bound this cache to 100 MB".
2. **All-or-nothing behaviour.** Under memory pressure the collector tends to clear soft references in bulk, so the cache goes from full to empty at exactly the moment the system is stressed — causing a cache-miss storm and a load spike on the backing store precisely when you can least afford it.
3. **They defer GC work and increase pause times.** Softly-reachable objects survive collections until pressure builds, inflating the live set, driving promotion to old generation, and making the eventual old collection more expensive. Reference processing itself is a phase of GC that scales with the number of references.
4. **They mask undersized heaps.** The application appears to work while spending increasing time in GC.
5. **Better tools exist.** **Caffeine** (or Guava Cache) gives you explicit size or weight bounds, TTL/TTI expiry, refresh-ahead, an eviction policy (W-TinyLFU, which outperforms LRU on realistic workloads), and hit/miss statistics you can put on a dashboard. A bounded cache with known behaviour is operationally far superior to an unbounded one with unknowable behaviour.

The legitimate remaining niche for `SoftReference` is narrow: a genuinely optional, cheaply-recomputable, potentially large value where you would rather have it than not but never want it to cause an OOM — and even then, an explicitly bounded cache is usually clearer.

---

## Q10. What is escape analysis and when does it actually help you?

### Answer

Escape analysis is a **JIT compiler** (C2, and to a lesser degree Graal) analysis that determines, for each allocation, whether the resulting object's reference can be observed outside the compiled scope. Three outcomes:

- **NoEscape** — the object cannot be seen outside the current compilation unit (typically an inlined method tree).
- **ArgEscape** — the reference is passed to a callee but does not escape into the heap or another thread.
- **GlobalEscape** — the reference is stored in a field, returned, thrown, or otherwise published.

For NoEscape allocations, C2 can apply three optimisations:

1. **Scalar replacement** — the dominant one. The object is *never allocated at all*; its fields are decomposed into individual local values held in registers or on the stack. Note that HotSpot does not really do "stack allocation of objects" as commonly described; scalar replacement is the mechanism, and it is stronger, because register-resident fields avoid memory traffic entirely.
2. **Lock elision (synchronization elimination)** — if the object being locked cannot escape, no other thread can contend for it, so the monitor operations are removed. This is why `StringBuffer` used inside a method costs roughly what `StringBuilder` costs after JIT warmup.
3. **Scalar replacement of allocations feeding other optimisations** — removing an allocation often unlocks constant folding and dead code elimination downstream.

**Crucial precondition: inlining.** Escape analysis operates on the compiled method after inlining. If a method that would let the object escape is *not* inlined (too large, megamorphic call site, exceeded inlining budget, not hot enough), the analysis conservatively marks the object as escaping and no optimisation occurs. This makes the effect **fragile and non-local**: adding a log statement, introducing a third implementation at a call site (turning a bimorphic call megamorphic), or growing a method past the inlining threshold can silently remove the optimisation and change allocation rate significantly.

**When it actually helps you.**

- **`Optional`** — arguably its main practical justification. `map.find(k).map(X::getName).orElse("none")` allocates zero `Optional` objects in hot JIT-compiled code when everything inlines. This is why "`Optional` allocates and is therefore slow" is usually wrong in practice, though it *is* right in cold code and when the call site is megamorphic.
- **Iterators** — enhanced-for over an `ArrayList` typically scalar-replaces the iterator entirely.
- **Small value objects, boxed primitives in local computation, `Point`-like structs, streams in simple pipelines** — often optimised away.
- **Autoboxing in arithmetic-heavy local code** — sometimes eliminated, but be careful: `Integer.valueOf` caches −128..127, and boxes taken from the cache are *shared*, hence global, and cannot be scalar-replaced.

**When it will not help:** anything stored in a field, returned from a non-inlined method, placed in a collection, captured by a lambda that escapes, passed to a virtual call site with many receiver types, or in code that never gets hot enough to be C2-compiled. Also, arrays are only scalar-replaced when the length is a compile-time constant and small (`-XX:EliminateAllocationArraySizeLimit`, default 64).

**How to use this knowledge in an interview.** The honest senior position: escape analysis is a reason **not** to hand-optimise away small short-lived objects prematurely, because the JIT frequently removes them for free. It is *not* something to design around, because you cannot reliably predict or control it. Write clear code; measure; only then look at whether allocations you assumed were free are actually happening.

### Probes

**Scalar replacement.** Covered above. The distinction from "stack allocation" is worth making explicitly — many candidates say "the object is allocated on the stack", which is a reasonable mental model but not what HotSpot implements.

**Stack allocation.** HotSpot C2 does not implement general stack allocation of objects; scalar replacement covers the common cases. (Some other runtimes and research JVMs do. Project Valhalla's value classes aim to make this a *language-level* guarantee rather than an opportunistic optimisation — which is precisely because the JIT's version is unreliable.)

**Lock elision.** As described. Related but distinct optimisations: **lock coarsening** (merging adjacent synchronized blocks on the same object into one) and, historically, **biased locking** (removed in JDK 15+/18 because it complicated the runtime and its benefit shrank as concurrency-aware libraries replaced legacy synchronized collections). Do not cite biased locking as a current mechanism.

**How to observe it.** Options, in increasing order of rigour:

- `-XX:+UnlockDiagnosticVMOptions -XX:+PrintEscapeAnalysis -XX:+PrintEliminateAllocations` (requires a debug or diagnostic-enabled build for some flags) prints EA decisions.
- `-XX:+UnlockDiagnosticVMOptions -XX:+PrintInlining` shows whether the necessary inlining happened — often more diagnostic than EA output itself, since failed inlining is the usual cause.
- **JMH with `-prof gc`** is the practical answer: it reports normalised allocation rate in bytes per operation. If EA is working, `gc.alloc.rate.norm` will be zero (or much lower than the object size implies). This is the measurement that actually settles arguments, and it is what you should say in an interview because it demonstrates you validate rather than assume.
- `-XX:-DoEscapeAnalysis` to disable it and compare — a clean A/B that isolates the effect.
- Async-profiler in allocation mode (`-e alloc`) on a running application shows where allocation actually occurs, which naturally reveals where EA is *not* helping.

**Why it's fragile and profile-dependent.** Summarised above: it depends on inlining, which depends on method size, call-site profile (mono/bi/megamorphic), compilation tier, and the inlining budget. It also does not survive deoptimisation — if the JIT deoptimises back to the interpreter (uncommon trap, class loading invalidating an assumption), the allocations reappear. And profile pollution is real: a shared utility method called from many sites with different types can become megamorphic and lose inlining for *all* callers, including the hot one.

---

## Q11. Explain classloading and the delegation model. How would you deliberately break it, and why does an app server do so?

### Answer

**The hierarchy (modern JDK, post-Jigsaw):**

- **Bootstrap** class loader — native, no Java object (`getClassLoader()` returns `null`). Loads core platform classes from the `java.base` and other core modules.
- **Platform** class loader (formerly "extension") — loads remaining platform modules; may be granted permissions the application loader lacks.
- **System/Application** class loader — loads classes from the class path and application modules.
- **Custom loaders** — anything you or a framework defines, usually with the application loader as parent.

**Parent-first delegation.** `ClassLoader.loadClass` default implementation: (1) check whether this loader has already defined/loaded the class (`findLoadedClass`); (2) if not, delegate to the parent, recursively up to bootstrap; (3) only if every ancestor fails, call `findClass` to load it locally. This has two purposes:

- **Security** — application code cannot substitute its own `java.lang.String`. (There is a second, independent protection: package names beginning with `java.` are rejected by `defineClass` regardless.)
- **Type consistency** — a class loaded once by an ancestor is shared by all descendants, so `java.util.List` means the same type everywhere.

**Class identity is (name, defining loader).** This is the single most important fact for debugging classloader problems. `com.acme.Foo` loaded by loader A and `com.acme.Foo` loaded by loader B are **different runtime types**. Assigning one to the other throws `ClassCastException` with the maddening message "class com.acme.Foo cannot be cast to class com.acme.Foo". The extended message in modern JDKs helpfully appends each class's loader and module, which is how you diagnose it.

**Loading is lazy and has phases:** loading (bytes → `Class` object) → linking (verification, preparation of statics to defaults, optional resolution) → initialisation (static initialisers and static field assignments, triggered on first active use: `new`, static field access, static method call, reflection, subclass initialisation). Initialisation is thread-safe: the JVM holds an initialisation lock per class, which is the basis of the holder idiom. If a static initialiser throws, the class is marked erroneous and every subsequent use throws `NoClassDefFoundError` with the *original* exception as cause only on the first attempt — later attempts lose the cause, which is why the first stack trace in the log matters.

**How to break delegation deliberately** — override `loadClass` (not `findClass`) to attempt local loading *before* delegating, usually with an allowlist of packages that must still come from the parent (`java.*`, `javax.*`, the container's own SPI packages, and any type crossing the boundary).

**Why app servers do it — parent-last / child-first.** A Jakarta EE or servlet container hosts multiple applications in one JVM. Each needs its own library versions: app A on Jackson 2.12, app B on Jackson 2.17, container on something else. Parent-first would force all of them onto whatever the container ships. Parent-last gives each web application its own `WebappClassLoader` that prefers `WEB-INF/lib` and `WEB-INF/classes`, achieving version isolation and enabling hot redeploy (discard the loader, discard the classes). The servlet spec explicitly permits this, with the caveat that the servlet API classes themselves must come from the container so the types match.

The cost is exactly the class-identity problem: types that cross the boundary must be loaded by the shared parent, or you get the two-`Foo` `ClassCastException`. This is why containers maintain explicit lists of packages that are always delegated.

### Probes

**Parent-first vs parent-last.** Covered above. Add: OSGi takes a third approach — a graph rather than a hierarchy, where each bundle declares imported and exported packages and the framework wires them, allowing multiple versions to coexist with explicit contracts. It is more powerful and considerably more complex, which is why it thrived in Eclipse and enterprise middleware but lost the mainstream to "one app per container image".

**`NoClassDefFoundError` vs `ClassNotFoundException`.** Different causes, and mixing them up is a red flag:

- **`ClassNotFoundException`** — a *checked* exception thrown by explicit, reflective loading (`Class.forName`, `loader.loadClass`, deserialisation) when the class is not found on the loader's search path. Cause: a missing dependency, a typo in a class name string, or a configuration file naming a class that isn't packaged.
- **`NoClassDefFoundError`** — an `Error` thrown by the JVM when *linking* code that references a class which was present at compile time but cannot be loaded now, **or** — the case that surprises people — when the class was found but its **initialisation previously failed**. So `NoClassDefFoundError: Could not initialize class X` almost always means a static initialiser in X threw earlier, and you must scroll back in the log to find the *first* occurrence, which carries `ExceptionInInitializerError` and the real cause. Treating the `NoClassDefFoundError` as the bug leads people to chase a nonexistent packaging problem.

**Same class name from two loaders → `ClassCastException`.** Covered. Diagnostic technique: print `obj.getClass().getClassLoader()` and `Target.class.getClassLoader()` and compare; or `-verbose:class` to see which loader defined each. Frequent real causes: a JAR present both in the container's `lib` and in `WEB-INF/lib`; a shaded/relocated dependency also present unshaded; a Spring Boot fat jar plus the same library on the external classpath.

**Spring Boot's nested-jar `LaunchedURLClassLoader`.** A standard `java -jar` can only read classes from the top level of a jar; the JAR spec has no notion of a jar inside a jar on the classpath. Spring Boot solves this without unpacking:

- The fat jar's manifest `Main-Class` is `org.springframework.boot.loader.JarLauncher` (or `PropertiesLauncher`/`WarLauncher`), not your class. Your class is named in `Start-Class`.
- The launcher creates a `LaunchedURLClassLoader` (renamed `LaunchedClassLoader` in Spring Boot 3.2's rewritten loader) that understands a custom `jar:` URL protocol addressing entries inside `BOOT-INF/lib/*.jar`.
- Nested library jars are stored **uncompressed** (`STORED`) so they can be read by offset without inflating the whole archive; your application classes in `BOOT-INF/classes` are compressed normally.
- The launcher then invokes `Start-Class.main` on a thread whose context classloader is the launched loader.

Practical consequences worth knowing: code that assumes `getClass().getResource(...)` returns a `file:` URL, or that calls `new File(url.toURI())`, breaks inside a fat jar — use `getResourceAsStream` instead. Agents and tools that scan the classpath may not see nested jars. And `PropertiesLauncher` exists specifically for the case where you need to add external directories to the classpath at runtime.

---

## Q12. What's the practical difference between a record and a Lombok `@Value` class? When would a record be the wrong choice?

### Answer

**Records** (finalised in Java 16) are a *language* feature: a restricted, nominal form of class declaring its state up front. `record Point(int x, int y) {}` gives you private final fields, a canonical constructor, accessors named `x()`/`y()` (no `get` prefix), and `equals`, `hashCode`, `toString` derived from the components. Records are implicitly `final`, cannot extend another class (they extend `java.lang.Record`), and cannot declare additional instance fields — the components *are* the state. This is the key semantic guarantee: the state description in the header is the whole state, which is what allows deconstruction patterns to work and what makes `equals` provably consistent with the accessors.

**Lombok `@Value`** is an *annotation processor* that mutates the AST during compilation to generate a final class with private final fields, getters (`getX()`), `equals`/`hashCode`/`toString`, and an all-args constructor. It is a code generator, not a language feature.

**Practical differences that matter:**

| | Record | Lombok `@Value` |
|---|---|---|
| Nature | Language feature, in the JLS | Third-party compile-time AST manipulation |
| Tooling | Understood by javac, IDEs, debuggers natively | Requires IDE plugin; can break on javac/JDK upgrades |
| Accessors | `x()` | `getX()` — matters for JavaBeans-based tooling |
| Inheritance | Cannot extend a class | Can extend |
| Extra fields | Forbidden | Allowed |
| Pattern matching | Supports record deconstruction patterns | No |
| Reflection | `Class.isRecord()`, `getRecordComponents()` — used by Jackson, Hibernate, serialization frameworks | Indistinguishable from a hand-written class |
| Serialization | Uses the canonical constructor, bypassing the usual `readObject` back door | Standard (fragile) Java serialization |
| Build risk | None | Adds a dependency on a tool that hooks unsupported javac internals |

The strategic point: **records remove a dependency**. Lombok's value proposition was largely eliminating boilerplate that records now eliminate natively. Many teams keep Lombok only for `@Builder` and `@Slf4j`; a growing number drop it entirely because it has repeatedly required urgent upgrades to survive new JDK releases.

**Record serialization deserves emphasis.** A record's `readObject` path goes through the canonical constructor, so any validation in a compact constructor is enforced on deserialization. For an ordinary Serializable class, deserialization bypasses constructors entirely, which is a long-standing source of security vulnerabilities (an attacker can construct an instance violating every invariant). Records close that hole by construction.

**When a record is the wrong choice:**

1. **JPA entities.** Entities must be mutable (dirty checking works by mutating managed instances), need a no-arg constructor, need non-final fields for lazy proxies, and often need identity-based `equals` rather than value equality. Records violate all of this. Records *are* excellent as JPA **DTO projections** (`select new com.x.Dto(e.a, e.b) from ...`, or Spring Data interface/class-based projections), and Hibernate 6 supports record projections directly.
2. **Anything requiring inheritance or extending a framework base class.**
3. **Mutable domain objects** — an aggregate whose state changes over its lifetime.
4. **Classes with derived or lazily-computed cached state** — you cannot add a `private int cachedHash` field. (You can compute derived values in accessors, but not memoise them in a field.)
5. **Large parameter lists where argument order is error-prone** — a record's canonical constructor is positional. `new Address(a, b, c, d, e, f, g)` with seven `String`s is a bug waiting to happen; a builder is safer. (Records plus a hand-written or generated builder is a reasonable hybrid.)
6. **Frameworks requiring JavaBeans naming** — anything relying on `getX()`/`setX()`. Most modern frameworks (Jackson 2.12+, Spring's `@ConfigurationProperties` via constructor binding, Bean Validation) handle records, but older or bespoke tooling may not.
7. **When you need to hide the representation.** A record's header is public API; changing a component is a breaking change. A class can change its internal fields while keeping its accessors stable.

### Probes

**Nominal tuple semantics.** A record is a *named* tuple: `record Point(int x, int y)` and `record Dimension(int x, int y)` are unrelated types despite identical shape. This is deliberate — Java rejected structural tuples in favour of nominal types so that the type name carries meaning and API evolution is controlled.

**Implicit `equals`/`hashCode`/`toString`.** Generated to compare/derive from all components. `equals` uses `==` for primitives and `Objects.equals` for references (with `Double`/`Float` compared per `Double.compare` semantics, so `NaN` equals `NaN` and `0.0` does not equal `-0.0` — a subtle difference from `==` that matters in financial or scientific code). The implementations are generated via `invokedynamic` against `ObjectMethods.bootstrap`, so they are not visible as ordinary bytecode. You may override any of them explicitly if you need different semantics.

**Compact constructors for validation.**

```java
record Range(int lo, int hi) {
    Range {                                   // compact form: no parameter list, no assignment
        if (lo > hi) throw new IllegalArgumentException("lo > hi");
        lo = Math.max(lo, 0);                 // reassigning the PARAMETER normalises the field
    }
}
```

The compact constructor's body runs before the implicit field assignments, and assigning to a parameter changes what gets stored. This is the idiomatic place for validation and normalisation. You may also write an explicit canonical constructor, but then you must assign all fields yourself.

**No inheritance.** Records are implicitly `final` and extend `Record`. They *can* implement interfaces, which is how they combine with sealed hierarchies: `sealed interface Shape permits Circle, Square {}` with `record Circle(double r) implements Shape {}`. That combination — sealed interface plus record implementations plus exhaustive `switch` with deconstruction patterns — is the modern Java idiom for algebraic data types and is very likely to come up.

**Defensive copying is *not* automatic (arrays/collections).** This is the most important practical gotcha. A record is **shallowly immutable**: the component fields are final, but a component of type `List<String>` or `byte[]` can be mutated by anyone holding a reference.

```java
record Message(byte[] payload) {}
byte[] b = {1,2,3};
Message m = new Message(b);
b[0] = 99;                 // m.payload()[0] is now 99
m.payload()[1] = 99;       // also works — the accessor returns the same array
```

Worse, `equals`/`hashCode` on an array component use **reference identity** (`Objects.equals` on arrays is `==`), so two records with identical array contents are unequal, and a record with an array component is unusable as a map key. Fixes: copy in the compact constructor *and* in an overridden accessor; or use an immutable type (`List.copyOf`, `String`, a wrapper) instead of an array; or, for byte payloads, wrap in something with value semantics.

```java
record Message(byte[] payload) {
    Message { payload = payload.clone(); }
    public byte[] payload() { return payload.clone(); }
}
```

**Serialization behaviour.** Covered above: canonical constructor is used, so invariants hold; `serialVersionUID` defaults to 0 for records and record serialization is based on component names/types rather than field layout, which makes evolution rules different from ordinary classes. Records cannot use `writeObject`/`readObject`/`readResolve` to customise the process (those are ignored), by design.

**JPA entities shouldn't be records.** Covered above. Worth stating the reason crisply: JPA's contract requires a mutable, proxyable, no-arg-constructible class, and records are none of those.

---

## Q13. What problem do sealed interfaces solve that an enum or plain interface doesn't? Combine with pattern matching for a real example.

### Answer

**The gap being filled.** Java historically offered two extremes:

- **`enum`** — a *closed* set of values, exhaustively checkable by the compiler, but each constant is a single instance with no per-constant type or state beyond shared fields. You cannot have `Circle(radius)` and `Rectangle(w,h)` as enum constants carrying different data.
- **`interface`/abstract class** — arbitrary per-implementation state and behaviour, but an *open* set. The compiler cannot know all implementations, cannot check exhaustiveness, and cannot let you reason about the complete hierarchy.

**Sealed types** (finalised in Java 17, JEP 409) give the third option: a closed set of *types*, each with its own shape and state.

```java
public sealed interface PaymentResult
        permits Approved, Declined, RequiresAction, Error {}

public record Approved(String authCode, Money captured)          implements PaymentResult {}
public record Declined(DeclineReason reason, boolean retryable)  implements PaymentResult {}
public record RequiresAction(URI redirectUrl, Duration expiry)   implements PaymentResult {}
public record Error(String code, String message)                 implements PaymentResult {}
```

Every permitted subtype must be `final`, `sealed`, or explicitly `non-sealed`, and (unless in a named module) must live in the same package — so the compiler can see the whole hierarchy and enforce closure.

**What this buys, concretely:**

1. **Compiler-checked exhaustiveness.** A `switch` covering all permitted subtypes needs no `default`. If someone adds a fifth subtype, **every** such switch fails to compile until updated. That is the entire point: adding a case to a domain becomes a compile-time task list rather than a runtime `UnsupportedOperationException` discovered in production.
2. **Modelling with data, not polymorphism.** Sometimes behaviour genuinely belongs on the type (`shape.area()`). But often the behaviour belongs to the *caller* — how the HTTP layer renders a payment result is not the payment domain's concern, and putting `toHttpResponse()` on the result type couples the domain to the web layer. Sealed types plus pattern matching let external code branch exhaustively without the visitor pattern's boilerplate.
3. **API contract.** "These are the only possible outcomes" is expressed in the type system and enforced by the compiler, not documented in a comment.
4. **Optimisation and reasoning.** The compiler and JIT know the complete set of implementations, which can enable devirtualisation.

**Worked example with pattern matching (Java 21):**

```java
String describe(PaymentResult r) {
    return switch (r) {
        case Approved(String code, Money amount)
                when amount.isGreaterThan(Money.of(10_000))
            -> "Large payment approved: " + code + ", manual review queued";
        case Approved(String code, Money amount)
            -> "Approved " + amount + " (" + code + ")";
        case Declined(var reason, boolean retryable) when retryable
            -> "Declined (" + reason + ") — will retry";
        case Declined(var reason, var __)
            -> "Declined permanently: " + reason;
        case RequiresAction(URI url, var expiry)
            -> "3-D Secure required at " + url + ", expires in " + expiry;
        case Error(var code, var msg)
            -> "Gateway error " + code + ": " + msg;
        // no default — the compiler proves this is exhaustive
    };
}
```

Note three things: **record deconstruction patterns** destructure components directly into named variables; **guards** (`when`) refine a case without nesting `if`s; and the absence of `default`, which is what makes the switch a compile-time safety net rather than a runtime hazard.

**Why not `default`?** Adding `default -> throw new IllegalStateException()` seems defensive but is actively harmful: it silences the exhaustiveness check, so adding a new subtype compiles cleanly and fails at runtime. Omit `default` deliberately in exhaustive switches over sealed types. (The compiler inserts a synthetic `MatchException` path for the case where the hierarchy changes after separate compilation — so you get an error, not silent misbehaviour, even in that scenario.)

### Probes

**Exhaustive `switch` without a `default`.** Covered. Two supporting details: (1) exhaustiveness also works for sealed hierarchies nested several levels deep, and covering an intermediate sealed type covers its subtypes; (2) `switch` on a sealed type is `null`-hostile by default — it throws `NullPointerException` unless you write `case null ->`, which was a deliberate design choice to preserve compatibility with existing switch semantics while making null handling explicit.

**Algebraic data types.** Sealed interface = sum type ("one of these"); record = product type ("all of these together"). Java now has both, which is what makes the "make illegal states unrepresentable" style feasible. Compare with the common alternative — a single mutable class with a `type` field and a dozen nullable fields, where every consumer must know which fields are meaningful for which type. That design has no compiler support at all and is a reliable source of NPEs.

**Compile-time safety when a new subtype is added.** The core value. Contrast with the alternatives: with an open interface, a new implementation silently falls into some `else` branch; with an enum plus a separate data class, the compiler checks the enum switch but nothing ties the data validity to the enum value; with the visitor pattern you *do* get compile-time exhaustiveness, but at the cost of a `Visitor` interface, an `accept` method on every type, and double dispatch — sealed types achieve the same guarantee with none of that machinery, and unlike visitors they let you add new *operations* freely without touching the type hierarchy.

**Modelling domain results/state machines.** Two high-value applications:

*Results* — replacing exceptions or nullable returns for expected outcomes: `sealed interface Result<T> permits Success, Failure`. Expected business outcomes ("insufficient funds") are values; genuinely exceptional conditions remain exceptions. This makes the possible outcomes visible in the signature.

*State machines* — `sealed interface OrderState permits Draft, Submitted, Paid, Shipped, Cancelled`, where each state carries only the data valid in that state (`Shipped(TrackingNumber t, Instant at)` — a tracking number cannot exist on a `Draft`). Transition functions become total, exhaustive switches. This is dramatically safer than an `OrderStatus` enum plus nullable `trackingNumber`, `shippedAt`, `cancellationReason` fields on one entity, where every consumer must remember which combinations are legal.

The practical caveat for interviews: this style is excellent for the domain layer, but sealed hierarchies do not map cleanly onto JPA entities or onto Jackson without configuration (you need `@JsonTypeInfo`/`@JsonSubTypes` or a custom deserialiser to round-trip polymorphic JSON — though Jackson does support this well). The usual architecture is sealed types in the domain, mapped to and from persistence and wire representations at the boundary.

---

## Q14. Explain virtual threads. What kind of workload gets faster, what gets no benefit, and what is "pinning"?

### Answer

**What they are.** Virtual threads (JEP 444, final in Java 21) are `java.lang.Thread` instances scheduled by the JVM rather than the OS. A platform thread is a thin wrapper over an OS thread with a fixed, large stack (typically 1 MB reserved) and context switches costing microseconds through the kernel. A virtual thread's stack lives on the Java heap as a growable *continuation* chain; creating one costs on the order of a few hundred bytes to a few kilobytes, and blocking one costs a heap write plus a scheduler handoff — no kernel involvement.

**How blocking works.** When a virtual thread executes a blocking operation that the JDK has been retrofitted for (socket I/O, `Thread.sleep`, `BlockingQueue`, `java.util.concurrent` locks, most `InputStream`/`OutputStream` on sockets, `Future.get`), the runtime **unmounts** it: its continuation is copied to the heap and its **carrier** — a platform thread from a dedicated `ForkJoinPool` (default parallelism = available processors) — is released to run another virtual thread. When the operation completes, the virtual thread is rescheduled onto some carrier (not necessarily the same one) and its stack is copied back.

The result: the classic thread-per-request model, which was abandoned because threads were too expensive, becomes viable again at a scale of millions. You get the *readability* of synchronous blocking code with the *scalability* of asynchronous code — without the callback or reactive-operator complexity, and with stack traces, debuggers, profilers, and `try/finally` all working normally.

**What gets faster.** I/O-bound, high-concurrency workloads with many concurrent blocking operations: a service that fans out to several downstream APIs per request; an application whose throughput is capped because the request thread pool is exhausted waiting on I/O; anything currently written reactively purely to avoid thread exhaustion. The improvement is in **throughput under concurrency**, not per-operation latency — a single request does not get faster.

**What gets no benefit.**

- **CPU-bound work.** Parallelism is still bounded by cores. Virtual threads do not create CPU capacity, and the ForkJoinPool carrier pool is sized to the core count.
- **Low-concurrency applications.** If you serve 50 concurrent requests, a 200-thread platform pool is fine and virtual threads add nothing.
- **Anything whose bottleneck is elsewhere** — and this is the crucial one. Most services are limited by the **database connection pool**, downstream rate limits, or the database itself. Making it possible to have 100,000 concurrent virtual threads all waiting for one of 20 HikariCP connections does not increase throughput; it converts a bounded queue at the thread pool into an unbounded queue at the connection pool, which is usually *worse* — you have removed your backpressure. Adopting virtual threads therefore requires re-examining every downstream limit and adding explicit concurrency limits (a `Semaphore`, a bulkhead) where the thread pool used to provide them implicitly.

**Pinning.** A virtual thread is *pinned* when it cannot unmount from its carrier, so a blocking operation blocks the underlying platform thread. Two causes in Java 21:

1. **Inside a `synchronized` block or method.** The monitor is associated with the carrier thread in the JVM's implementation, so the continuation cannot be moved.
2. **Inside a native frame (JNI) or a class initialiser.**

If enough virtual threads pin simultaneously, the carrier pool is exhausted and the application stalls — a deadlock-like hang that is confusing because thread dumps show idle virtual threads. The JDK mitigates by temporarily expanding the carrier pool (`jdk.virtualThreadScheduler.maxPoolSize`, default 256), but that is a bound, not a solution.

Detection: `-Djdk.tracePinnedThreads=short|full` prints stack traces when a pinned thread blocks; JFR emits `jdk.VirtualThreadPinned` events, which is the better production mechanism.

Remedy in Java 21: replace `synchronized` with `ReentrantLock` in code that blocks while holding the lock. `ReentrantLock` is implemented on `AbstractQueuedSynchronizer` and is virtual-thread-aware, so the thread unmounts correctly.

**JDK 24 (JEP 491) removed the `synchronized` limitation** by reimplementing monitors so that a virtual thread holding one can unmount. This materially reduces the risk of adopting virtual threads with legacy libraries — but only on 24+; on 21 (the LTS most teams are on) the concern is live, and the fact that you cannot audit every transitive dependency for `synchronized` blocks around I/O is a real adoption risk worth stating.

### Probes

**IO-bound vs CPU-bound.** Covered. Say plainly: virtual threads are a *concurrency* mechanism, not a *parallelism* mechanism. For CPU-parallel work you still want a bounded pool sized to cores, or the fork/join framework, or parallel streams.

**Carrier threads.** The `ForkJoinPool` in FIFO mode with parallelism defaulting to `Runtime.availableProcessors()`, tunable via `jdk.virtualThreadScheduler.parallelism`. Note that in a container with a CPU limit, `availableProcessors` reflects the limit, so the carrier pool is sized accordingly — another reason container CPU settings have outsized effects.

**`synchronized` blocks pinning (and the JDK 24 fix).** Covered. Practical guidance for Java 21: `synchronized` on a short, purely in-memory critical section is harmless (no blocking occurs, so nothing is lost by staying mounted). The problem is *blocking while pinned* — I/O, `wait()`, lock acquisition, or a slow computation inside a `synchronized` region. Audit hot paths and known-risky libraries (older JDBC drivers, connection pools, some logging appenders) rather than attempting a wholesale conversion.

**`ThreadLocal` memory cost at scale.** Virtual threads support `ThreadLocal`, but each of a million virtual threads gets its own map and its own copy of every value. If a framework stores a large object per thread (a security context, a Jackson `ObjectMapper`, a buffer), the memory multiplies. Also, the traditional *use* of `ThreadLocal` — caching an expensive object to avoid reallocation — is now counterproductive, since virtual threads are short-lived and there is no pooling to amortise the cost. `ScopedValue` (JEP 446 and successors, still preview in recent releases) is the intended replacement: an immutable value bound for a dynamic scope, shared by structure rather than copied, automatically unbound on scope exit, and inherited efficiently by structured-concurrency children. Do not claim `ScopedValue` is final unless you have checked your target JDK.

**Why thread pools become an anti-pattern.** Pooling exists to amortise the cost of an expensive resource. Virtual threads are cheap, so pooling them defeats the purpose and reintroduces the ceiling you were trying to remove. The idiomatic replacement is `Executors.newVirtualThreadPerTaskExecutor()` — which is not a pool at all, just a thread-per-task factory — or **structured concurrency** (`StructuredTaskScope`), which ties the lifetime of concurrent subtasks to a lexical scope and gives you cancellation and error propagation for free. The corollary: **you must reintroduce concurrency limits explicitly** where the pool used to enforce them. A `Semaphore` sized to the downstream's capacity is the usual tool.

**Why connection pools are still the real bottleneck.** The point made above, worth repeating because it is the most common misconception. Spring Boot's `spring.threads.virtual.enabled=true` switches Tomcat to virtual threads in one line, but a service doing `@Transactional` work still holds a JDBC connection for the duration of each transaction, and the pool is typically 10–20 connections. Throughput is `pool_size / avg_connection_hold_time`, unchanged by the threading model. Realising throughput gains requires shortening transactions, sizing the pool to what the database can actually serve (which is bounded by database cores and I/O, not by your desire), or caching. Note also that a common HikariCP configuration relies on `connectionTimeout` to fail fast when the pool is exhausted — with vastly more concurrent requests, far more of them will hit that timeout, so error-handling and load-shedding behaviour must be reviewed too.

---

## Q15. When is checked-exception design justified, and how do you handle exceptions across a layered service without either swallowing or leaking them?

### Answer

**The design question.** The original intent of checked exceptions was to force callers to acknowledge recoverable failure conditions. In practice the industry has largely concluded they were over-applied: they leak implementation details into signatures (a `throws SQLException` on a repository interface couples every caller to JDBC), they compose badly with lambdas and streams (functional interfaces do not declare checked exceptions, forcing wrapper boilerplate), and they encourage the worst possible response — an empty `catch` block — because the compiler demands *something*.

A defensible modern position:

**Use a checked exception when all of these hold:** (1) the condition is expected in normal operation, not a bug; (2) the immediate caller can plausibly do something specific about it other than propagate; (3) the failure is part of the abstraction's contract rather than an implementation leak. Example: a parser that offers `parseOrThrow` and expects callers to handle malformed input as a normal case.

**Use an unchecked exception when:** the condition indicates a programming error (`IllegalArgumentException`, `IllegalStateException`, `NullPointerException`), or when it is a systemic failure that most callers cannot handle and should propagate (database unavailable, network failure, configuration error). This is exactly why Spring wraps `SQLException` in the unchecked `DataAccessException` hierarchy, and why almost every modern JVM library — Spring, Hibernate JPA operations, Jackson's runtime layer, most HTTP clients — uses unchecked exceptions.

**Consider neither** for expected, frequent, business-meaningful outcomes: return a value instead. `Optional<T>` for absence; a sealed `Result` type for "succeeded or failed with one of these reasons". Exceptions are expensive (stack trace capture dominates the cost; `fillInStackTrace` walks the whole stack) and, more importantly, they are invisible in the type signature in the unchecked case. "Insufficient funds" is not exceptional — it is an outcome.

**Handling across layers.**

The organising principle is **exception translation at architectural boundaries**, preserving the cause chain:

- **Persistence layer.** Let Spring translate `SQLException`/`PersistenceException` into `DataAccessException` (this happens automatically for `@Repository` beans via `PersistenceExceptionTranslationPostProcessor`, and inside Spring Data repositories). Catch only the specific subtypes you can act on — `DuplicateKeyException` when you want to convert a unique-constraint violation into a domain-level "already exists", `OptimisticLockingFailureException` when you want to retry.
- **Domain/service layer.** Throw domain exceptions that mean something to the business: `InsufficientFundsException`, `OrderNotFoundException`, `AccountFrozenException`. These should not mention JDBC, HTTP, or Kafka. Always pass the original as the cause: `throw new OrderNotFoundException(id, e);`.
- **API/adapter layer.** A single `@RestControllerAdvice` maps domain exceptions to HTTP status codes and a consistent error body (see Q59). Nothing below this layer should know about HTTP status codes.
- **Never let raw infrastructure exceptions reach the client.** A stack trace or SQL fragment in a response body is an information-disclosure vulnerability.

**The rules that prevent both swallowing and leaking:**

1. **Never catch and ignore.** If you catch, either handle it, translate and rethrow, or log-and-rethrow at exactly one place (not both log and rethrow at every level, which produces the same failure logged eight times).
2. **Never log and swallow** unless you can articulate precisely why continuing is correct — and then log at WARN with the reason, not DEBUG.
3. **Always preserve the cause.** `new ServiceException(msg)` without `e` destroys the diagnostic trail. This is one of the most damaging habits in enterprise code.
4. **Catch narrowly.** `catch (Exception e)` around a whole method catches `NullPointerException` and every programming error alongside the one condition you meant to handle, turning bugs into silent misbehaviour.
5. **Log once, at the boundary.** The handler that decides the outcome logs the full stack trace with a correlation ID. Intermediate layers add context by wrapping, not by logging.
6. **Never catch `Throwable`/`Error`** except in a top-level supervisor. `OutOfMemoryError` and `StackOverflowError` are not recoverable in general; catching them leaves the JVM in an undefined state.
7. **Never use exceptions for control flow.** Throwing to exit a loop or signal a normal branch is slow and obscures intent.

### Probes

**Translate at boundaries, don't lose the cause.** Covered. Reinforce with the diagnostic consequence: a `ServiceException: failed to save order` with no cause tells you nothing; the same with a chained `ConstraintViolationException: duplicate key value violates unique constraint "orders_ref_key"` tells you the bug in one line. Use the two-arg constructor. Also add *context*, not just wrapping: include the identifiers (`orderId`, `customerId`) in the message so the log line is actionable without re-running the request. Avoid putting PII in exception messages, since they end up in logs.

**`@ControllerAdvice`.** A `@RestControllerAdvice` class with `@ExceptionHandler` methods centralises the mapping. Practical details: order matters when handlers overlap (more specific types win; `@Order` on multiple advice classes resolves ties); extending `ResponseEntityExceptionHandler` gives you Spring MVC's built-in handling for framework exceptions (validation, unsupported media type, malformed JSON) which you can selectively override; and Spring Boot 3 supports `ProblemDetail` (RFC 9457) natively, which is the right response shape. Handle a catch-all `Exception` returning 500 with a correlation ID and *no* internal detail, and log that one at ERROR — an unmapped exception reaching the catch-all is itself a signal worth alerting on.

**Don't use exceptions for control flow.** Cost breakdown: constructing an exception captures the stack trace, which is proportional to stack depth and is the dominant expense — often microseconds, orders of magnitude more than a branch. (You can suppress it by overriding `fillInStackTrace` or using the four-arg `Exception` constructor with `writableStackTrace=false`, which is a legitimate technique for high-frequency, stack-trace-irrelevant control exceptions such as reactive cancellation signals — but if you find yourself needing it, that is evidence the design should return a value instead.) The stronger argument is readability: a method that signals "not found" by throwing has a signature that says nothing, whereas `Optional<Order> find(id)` documents itself.

**`try-with-resources` and suppressed exceptions.** Try-with-resources closes resources in reverse declaration order and, critically, handles the case where both the body and `close()` throw. The body's exception is the primary one; the close exception is attached via `addSuppressed` and is visible in the stack trace under "Suppressed:". With a manual `finally { r.close(); }`, an exception from `close()` **replaces** the original exception and you lose the actual cause — a classic bug that hides the real failure behind a misleading "connection already closed". Always use try-with-resources. For resources not implementing `AutoCloseable`, a local wrapper is worth writing. Note the effectively-final resource variable form added in Java 9 (`try (existingResource) { ... }`).

**Why `catch (Exception e)` around a whole method is a smell.** It conflates unrelated failure modes; it catches `RuntimeException`s that indicate bugs (NPE, `ArrayIndexOutOfBounds`, `ClassCastException`) and turns a loud, fixable defect into a silent wrong answer; and it makes the code's failure behaviour unanalysable. It is *sometimes* correct — at a top-level request handler, in a message consumer's poll loop that must not die, in a scheduled task — but in those cases the handling must be deliberate: log with full context, increment an error metric, and decide explicitly whether to continue, retry, or dead-letter. A blanket catch that logs at DEBUG and continues is how systems silently produce wrong data for months. Note also `InterruptedException`: catching it without either rethrowing or calling `Thread.currentThread().interrupt()` destroys the interruption signal and breaks cancellation and graceful shutdown — one of the most common and most damaging swallow bugs.

---

# 2. Concurrency

---

## Q16. Design a bounded cache that must be thread-safe and must compute expensive values at most once per key. Walk from naive to correct.

### Answer

**Stage 1 — `synchronized` map.** A `HashMap` guarded by a lock, or `Collections.synchronizedMap`. Correct, but the lock is held across the *computation*, so every thread requesting any key blocks behind one slow computation. Throughput collapses to serial.

**Stage 2 — `ConcurrentHashMap` with `get` then `putIfAbsent`.** Removes the global lock, but two threads that both miss will both compute. `putIfAbsent` ensures only one result is stored, so correctness of the *cache contents* is fine, but the expensive work is duplicated — unacceptable if the computation is a database query, an HTTP call, or something with side effects.

**Stage 3 — `computeIfAbsent`.** Atomic per key: the mapping function runs at most once per key, and other threads requesting the *same* key block until it completes. Threads requesting different keys generally proceed (locking is per bin). For most applications this is the correct answer.

```java
private final ConcurrentHashMap<K, V> cache = new ConcurrentHashMap<>();
V get(K key) { return cache.computeIfAbsent(key, this::expensiveCompute); }
```

Two hazards, both documented and both real: the mapping function must not modify the same map (attempting to do so may throw `IllegalStateException` for a detected recursive update, or deadlock), and because the bin is locked during computation, a long computation blocks other keys that hash to the same bin and blocks `size()`/resize.

**Stage 4 — `FutureTask` memoizer** (the *Java Concurrency in Practice* solution) when you need the computation to run outside the map's lock:

```java
private final ConcurrentHashMap<K, Future<V>> cache = new ConcurrentHashMap<>();

V get(K key) throws InterruptedException {
    while (true) {
        Future<V> f = cache.get(key);
        if (f == null) {
            FutureTask<V> task = new FutureTask<>(() -> compute(key));
            f = cache.putIfAbsent(key, task);
            if (f == null) { f = task; task.run(); }   // we won: compute
        }
        try { return f.get(); }
        catch (CancellationException e) { cache.remove(key, f); }   // retry
        catch (ExecutionException e) { cache.remove(key, f); throw launder(e); }
    }
}
```

The map stores a *promise* rather than a value, so it is populated atomically and instantly; only the losing threads block, on `Future.get()`, outside any map lock. The `remove` on failure is essential — otherwise a single failure is cached permanently ("negative caching by accident").

**Stage 5 — bounded, with eviction.** None of the above is bounded; an unbounded cache is a memory leak with a nicer name. Adding correct bounded eviction requires tracking access order under concurrency, which is genuinely hard to do without either a global lock or losing accuracy. `LinkedHashMap` with `removeEldestEntry` gives LRU but is not thread-safe and needs external synchronisation, reintroducing the stage-1 problem.

**Stage 6 — use Caffeine.** In production this is the answer, and saying so is a strength, not a cop-out:

```java
LoadingCache<K, V> cache = Caffeine.newBuilder()
    .maximumSize(10_000)                     // or .maximumWeight + .weigher
    .expireAfterWrite(Duration.ofMinutes(10))
    .refreshAfterWrite(Duration.ofMinutes(5))
    .recordStats()
    .build(this::expensiveCompute);
```

Caffeine gives per-key single-flight loading, W-TinyLFU eviction (materially better hit rates than LRU on realistic skewed workloads), size or weight bounds, expiry-after-write/access, refresh-ahead (serve stale while reloading asynchronously, avoiding a latency spike on expiry), asynchronous loading, removal listeners, and statistics you can export to Micrometer. Reimplementing that correctly is weeks of work and will be worse.

### Probes

**`synchronized` map → `putIfAbsent` (duplicate computation) → `computeIfAbsent` (and its deadlock/re-entrancy hazard) → `FutureTask` memoizer.** All covered above. The re-entrancy hazard is worth stating precisely: if the mapping function calls back into the same `ConcurrentHashMap` for a key in the same bin, you can deadlock; if it mutates the map, you may get `IllegalStateException: Recursive update`. This bites in practice when the "compute" function is a service call that transitively touches the same cache. `computeIfAbsent` is also documented as requiring a short computation for exactly this reason.

**Eviction.** Without a bound, a cache keyed on anything user-supplied (tenant IDs, search terms, URLs) grows until OOM. Bound by **entry count** when values are uniform, by **weight** when they vary (a cache of response bodies should be bounded by bytes, not entries). Add TTL to bound *staleness* as well as size — even an unbounded-in-practice key space needs expiry so that changed upstream data eventually propagates. Also add jitter to TTLs if many entries are populated simultaneously, or they all expire together and you get a thundering herd (see Q87).

**Why Caffeine exists.** Because correct, high-performance, bounded concurrent caching requires solving problems the JDK does not address: lock-free read paths (Caffeine buffers access records in ring buffers and replays them under a single eviction lock, so reads scale linearly), an eviction policy that beats LRU under skew, and correct interaction between expiry, refresh, and loading. Guava Cache was the predecessor; Caffeine is its successor by the same maintainer and is generally faster. Spring's `@Cacheable` abstraction supports Caffeine as a backing provider, so you can use it without coupling application code to the library.

---

## Q17. What does `ConcurrentHashMap` guarantee, and what doesn't it? Is `size()` reliable? Is iterating it safe?

### Answer

**Guarantees.**

1. **Per-operation atomicity and thread safety.** Each individual method (`get`, `put`, `remove`, `putIfAbsent`, `replace`, `compute`, `merge`, `computeIfAbsent`) is atomic with respect to other operations on the map.
2. **Memory visibility.** Operations that insert a value happen-before operations that read it. CHM is a legitimate safe-publication mechanism.
3. **Lock-free reads.** `get` takes no lock at all — nodes hold `volatile` value and next fields, so a read is a plain traversal with volatile loads. Reads never block, ever, even during a resize.
4. **Fine-grained write locking.** Since Java 8, writes lock the *first node of the bin* (`synchronized` on the head node) rather than using the pre-Java-8 segment array. Concurrency is therefore roughly proportional to the number of occupied bins, not a fixed 16 segments.
5. **No `null` keys or values.** Unlike `HashMap`. The reason is disambiguation: in a concurrent map you cannot distinguish "key absent" from "key present with null value" using `get` alone, because you cannot atomically follow up with `containsKey`. Rejecting null removes the ambiguity. This surprises people migrating from `HashMap` and causes NPEs on `map.put(k, maybeNull)`.

**What it does not guarantee.**

1. **Atomicity across multiple operations.** `if (!map.containsKey(k)) map.put(k, v)` is a race. Use `putIfAbsent`, `computeIfAbsent`, `merge`, or `replace(k, expected, new)`.
2. **Any guarantee about the map as a whole at a point in time.** There is no snapshot. `size()`, `isEmpty()`, `containsValue()`, `toString()`, `equals()`, and bulk operations are all computed over a moving target.
3. **Thread safety of the *values*.** `map.get(k).add(x)` on a mutable value is unsynchronised.
4. **Blocking-free `computeIfAbsent`.** That method *does* lock the bin.

**Is `size()` reliable?** Only as an estimate. CHM maintains a `baseCount` plus an array of `CounterCell`s (the same striped-counter design as `LongAdder`) to avoid contention on a single counter. `size()` sums them, and the sum is not taken atomically — concurrent modifications during the summation are not accounted consistently. In a quiescent map (no concurrent writers) it is exact. Under concurrent mutation it is approximate, and the javadoc says so. `mappingCount()` returns a `long` and is preferred for large maps since `size()` saturates at `Integer.MAX_VALUE`. Never use `size()` for control flow that requires exactness (e.g. "if size < limit then put" is a race and will exceed the limit).

**Is iterating it safe?** Yes, in the sense that it never throws `ConcurrentModificationException` and never corrupts. Iterators and spliterators are **weakly consistent**: they reflect the state of the map at some point at or since the iterator's creation; they are guaranteed to traverse each element exactly once *as of that state*; they may or may not reflect modifications made after creation. So you may see an entry that was removed after iteration started, and you may or may not see an entry added after. Iteration is safe but not a snapshot — do not use it to compute a consistent aggregate.

### Probes

**Per-bin locking.** Covered. Detail: on insertion into an *empty* bin, CHM uses a CAS on the table slot with no lock at all; only a non-empty bin requires locking its head node. Resize is cooperative — threads that encounter a bin being transferred help with the transfer rather than blocking, which is why CHM resizes without a large latency spike.

**Weakly consistent iterators.** Covered. Contrast with `HashMap`'s fail-fast iterators, which track a `modCount` and throw `ConcurrentModificationException` on detecting structural modification — a *best-effort* bug detector, not a guarantee (it can miss races, and it can also fire spuriously in single-threaded code when you remove via the collection rather than the iterator). Note also that CHM's bulk operations (`forEach`, `search`, `reduce`, including parallel variants) have the same weak-consistency semantics.

**`size()` is an estimate under concurrency.** Covered. The design rationale is worth stating: an exact concurrent size would require either a global lock or a strongly-consistent counter, both of which would serialise every write. The library deliberately trades exactness for scalability, and that trade is right for the overwhelming majority of uses.

**Atomicity only for single operations.** Covered, with the compound-action fixes. Add: `merge(k, v, remapper)` is the clean way to do "insert or update", e.g. `map.merge(key, 1L, Long::sum)` as a counter. And `compute`/`computeIfPresent` returning `null` removes the entry, which is a compact way to express conditional removal.

**`computeIfAbsent` blocking the bin.** The mapping function runs while holding the bin lock. Consequences: a slow function blocks all other keys hashing to that bin; a function that touches the same map may deadlock or throw; and in a virtual-thread context a blocking I/O call inside `computeIfAbsent` holds a lock across the block. Guidance: keep mapping functions short and in-memory. If the computation is I/O, use the `Future` memoizer pattern or Caffeine's async loading instead.

---

## Q18. Explain the difference between `CompletableFuture.thenApply`, `thenApplyAsync`, and `thenCompose`. Which thread runs your callback?

### Answer

**`thenApply(Function<T,U>)`** — transforms the result, returning `CompletableFuture<U>`. **`thenCompose(Function<T,CompletionStage<U>>)`** — the monadic bind: the function itself returns a future, and `thenCompose` flattens it. Using `thenApply` where the function returns a future yields `CompletableFuture<CompletableFuture<U>>`. The `map` versus `flatMap` distinction.

```java
cf.thenApply(user -> user.getName())                      // CF<String>
cf.thenCompose(user -> fetchOrdersAsync(user.getId()))    // CF<List<Order>>, flattened
```

**Which thread runs the callback?** This is the part people get wrong. For the non-async variants:

- If the future is **already complete** when you attach the callback, the callback runs **on the calling thread**, synchronously, inside `thenApply` itself.
- If the future is **not yet complete**, the callback runs on **whichever thread completes the future** — i.e. the thread that calls `complete()`, or the thread running the preceding stage.

So the non-async variants have *nondeterministic* execution context, decided by a race. That matters enormously: if your callback is expensive and the completing thread is a Netty event loop, a Kafka consumer poll thread, or an HTTP client's I/O thread, you have just blocked shared infrastructure. This is one of the most common serious `CompletableFuture` bugs.

**`thenApplyAsync(fn)`** — always submits the callback to an executor: `ForkJoinPool.commonPool()` by default. **`thenApplyAsync(fn, executor)`** — submits to *your* executor. This is the form you should almost always use in production code, because it makes the execution context explicit.

**The `commonPool` hazard.** Default parallelism is `availableProcessors() - 1`, and in a container with a CPU limit of 1, parallelism can be **zero**, in which case the JDK falls back to running each task in a new thread — behaviour most people do not expect. Worse, the common pool is shared with parallel streams and any other library using it. Submitting **blocking** work (JDBC calls, HTTP requests) to a pool sized for CPU-bound work starves everything else in the JVM. Rule: never do blocking I/O on the common pool; pass an explicit executor sized for the work.

### Probes

**Caller vs completing thread.** Covered above.

**Common ForkJoinPool and why that's dangerous for blocking work.** Covered. Add: if you must block inside a ForkJoinPool, `ForkJoinPool.ManagedBlocker` lets the pool compensate by starting an additional thread — rarely used and easy to get wrong, but worth knowing it exists.

**`exceptionally`/`handle`/`whenComplete` differences.**

- `exceptionally(Function<Throwable,T>)` — runs **only on failure**, recovers by producing a fallback value. Result type unchanged.
- `handle(BiFunction<T,Throwable,U>)` — runs on **both** success and failure; exactly one argument is non-null. Can transform the type and can recover.
- `whenComplete(BiConsumer<T,Throwable>)` — runs on both, but is a **consumer**: it cannot change the result. The original outcome (value or exception) propagates unchanged, so it is the right hook for logging, metrics, and cleanup. If the consumer itself throws, that exception replaces the original (or is suppressed if there already was one) — a subtle trap.
- `exceptionallyCompose` (Java 12+) — recover by returning another future, e.g. failover to a secondary endpoint.

Important detail: exceptions arrive wrapped in `CompletionException` (or `ExecutionException` from `get()`), so handlers usually need to unwrap `t.getCause()`. And **an exception thrown inside a stage is not thrown to the caller** — it completes the future exceptionally. If nothing ever consumes the future, the failure is silently lost. Always terminate a chain with error handling.

**`allOf` and collecting results.** `CompletableFuture.allOf(cf1, cf2, ...)` returns `CompletableFuture<Void>` — it does **not** collect results, so you must join each future afterwards (safe at that point, since all are complete):

```java
List<CompletableFuture<Order>> futures = ids.stream()
        .map(id -> supplyAsync(() -> fetch(id), executor)).toList();

CompletableFuture<List<Order>> all = CompletableFuture
        .allOf(futures.toArray(CompletableFuture[]::new))
        .thenApply(v -> futures.stream().map(CompletableFuture::join).toList());
```

Semantics to know: `allOf` completes exceptionally if *any* input fails, but it does **not cancel the others** — they keep running, wasting resources. `anyOf` completes with the first result, success or failure, which is often not what you want for a "first successful" hedge; you need `exceptionallyCompose` or manual handling for that. Also, `cancel(true)` on a `CompletableFuture` does **not** interrupt the running task — unlike `FutureTask`, there is no thread to interrupt from the future's perspective; it merely completes the future with `CancellationException`.

**Timeouts (`orTimeout`).** Java 9 added `orTimeout(n, unit)` (complete exceptionally with `TimeoutException`) and `completeOnTimeout(value, n, unit)` (complete with a fallback). Before Java 9 you had to build this with a scheduler. Note the same caveat: the timeout completes the *future*, it does not stop the underlying work. A timed-out HTTP call continues consuming a connection unless the client itself is configured with a timeout. **Always set timeouts at the client level as well** — the `CompletableFuture` timeout is a coordination mechanism, not resource control.

**Context propagation (MDC, security context).** Because callbacks run on arbitrary threads, anything stored in a `ThreadLocal` is lost: SLF4J MDC (so your correlation ID vanishes from logs mid-chain), Spring `SecurityContextHolder` (so downstream calls become unauthenticated), `RequestContextHolder`, transaction context, and tracing spans. Mitigations: Spring Security's `DelegatingSecurityContextExecutor`; Micrometer's `ContextSnapshot`/`ContextPropagation` library; a decorator that captures the context at submission and installs/clears it around execution; or `TaskDecorator` on Spring's `ThreadPoolTaskExecutor`. **And you must clear it afterwards** in a `finally`, or you leak context onto a pooled thread and the next task inherits it — the cross-request data leak described in Q9. This is one of the strongest practical arguments for virtual threads plus plain blocking code: the context simply stays on the thread.

---

## Q19. You have a thread pool that's "randomly hanging". Give me your diagnostic sequence.

### Answer

**1. Confirm the symptom.** Is it hung (no progress) or slow (progress at low rate)? Check whether tasks are being *submitted* and *completed* — instrument `ThreadPoolExecutor` (`getActiveCount`, `getQueue().size()`, `getCompletedTaskCount`) or use Micrometer's `ExecutorServiceMetrics`, which exposes these automatically. A growing queue with a flat completed count is a hang; a growing queue with a rising completed count is insufficient capacity.

**2. Take thread dumps.** `jstack <pid>` or `jcmd <pid> Thread.print` — take **three, ten seconds apart**. One dump shows state; three show whether it is stuck or moving. Look for pool threads (`pool-1-thread-*`) and classify:

- `BLOCKED` — waiting to enter a `synchronized` block. The dump names the monitor and its owner. Two threads each `BLOCKED` on a monitor the other owns is a deadlock.
- `WAITING` / `TIMED_WAITING` on `Object.wait`, `LockSupport.park`, `Condition.await` — waiting for a signal. The stack tells you for what.
- `RUNNABLE` inside a socket read — this is the deceptive one: a blocking socket read shows as `RUNNABLE` because the JVM cannot tell it is blocked in the kernel. **A pool of threads all `RUNNABLE` in `socketRead0` is a hang, not busy work.** This is the single most common cause of an apparently mysterious pool hang.

**3. Check for JVM-detected deadlock.** `jcmd <pid> Thread.print` reports "Found one Java-level deadlock" automatically for monitor cycles. Note it only detects `synchronized` and `java.util.concurrent.locks` cycles — it cannot detect a logical deadlock via semaphores, latches, or a connection pool.

**4. Check pool saturation and queue.** If `activeCount == maximumPoolSize` and the queue is growing, you are saturated, not deadlocked. Then the question becomes *why are tasks slow* — go to step 5.

**5. Look for missing timeouts.** Any blocking call without a timeout can hang a thread forever: HTTP clients (connect and read timeouts are separate; many clients default to infinite read), JDBC (`queryTimeout`, socket timeout, plus the pool's `connectionTimeout`), Kafka, Redis (Lettuce/Jedis have separate connect and command timeouts), gRPC deadlines. Audit every outbound call. A single downstream that stops responding will consume the entire pool within seconds.

**6. Look for pool starvation from nested submission.** See probes.

**7. If nothing is obvious:** async-profiler in wall-clock mode (`-e wall`) shows where time is spent including blocked time, which CPU profilers miss entirely. JFR with `jdk.ThreadPark`, `jdk.JavaMonitorEnter`, and socket I/O events gives the same picture with lower overhead and is safe to leave enabled in production.

### Probes

**Thread dump analysis.** Covered. Practical tips: `jcmd Thread.print` is preferred over `jstack` (fewer failure modes, no attach issues). Name your pools (`new ThreadFactoryBuilder().setNameFormat("order-fetch-%d")`) — the default `pool-3-thread-7` tells you nothing when you have twelve executors, and this five-minute investment pays for itself in the first incident. Deadlocks are easier to spot with a dump analyser than by eye.

**Blocked vs waiting vs parked.** Covered. Summary: `BLOCKED` = contending for a monitor (the owner is named in the dump); `WAITING` = indefinite wait for a signal; `TIMED_WAITING` = same with a deadline; `RUNNABLE` = scheduled, *or* blocked in a native/kernel call the JVM cannot see. The last is the one that misleads.

**Deadlock detection.** Covered. Two additional forms the JVM will not detect: (a) **lock-ordering deadlock via distinct resources** such as two database rows locked in opposite order — invisible to the JVM, visible in the database's deadlock log; (b) **resource deadlock** where thread A holds a connection and waits for a task submitted to a full pool, while that task needs a connection. Prevention is architectural: consistent lock ordering, and never acquire two pooled resources of different kinds in an order that varies.

**Pool starvation from nested task submission on the same pool.** The classic: a task submitted to a fixed pool submits a subtask *to the same pool* and blocks on its result. With N threads, if all N are blocked waiting on subtasks that cannot be scheduled because there are no free threads, the pool is permanently deadlocked. It is load-dependent — works in testing, fails under production concurrency — which is why it presents as "random hanging".

Fixes: never block on a task submitted to the pool you are running on; use separate pools per stage of a pipeline (which also gives you bulkhead isolation); use `ForkJoinPool`, whose work-stealing and `join()` semantics allow a joining worker to execute other tasks rather than idling; or move to `CompletableFuture` composition so nothing blocks; or virtual threads, where blocking is cheap.

**Unbounded queue hiding backpressure.** `Executors.newFixedThreadPool(n)` uses an **unbounded** `LinkedBlockingQueue`. Consequences: (a) the pool never rejects, so you never learn you are overloaded — until you OOM on queued tasks; (b) `maximumPoolSize` is meaningless, because `ThreadPoolExecutor` only creates threads beyond `corePoolSize` when the queue is *full*, and an unbounded queue never fills; (c) latency degrades unboundedly as the queue grows, so clients time out and retry, adding more work to a queue you cannot drain. `Executors.newCachedThreadPool` has the opposite failure: a `SynchronousQueue` with `Integer.MAX_VALUE` max threads, so it creates a thread per task under load and dies of thread exhaustion.

Correct practice: construct `ThreadPoolExecutor` explicitly with a **bounded** queue and a deliberate rejection policy. A full queue is information — surface it as a metric and an alert.

**Missing timeouts on blocking calls.** Covered in step 5. The framing for an interview: an operation without a timeout is an operation that can take forever, and a thread pool is a fixed pool of things that can be consumed forever. Timeouts convert an unbounded failure into a bounded one. This connects directly to Q86.

**Caller-runs as crude backpressure.** `CallerRunsPolicy` executes the rejected task on the submitting thread. This slows the producer — genuine backpressure — and is often better than dropping work. Caveat: if the submitter is your HTTP request thread, you have moved the work into the request path and the request now takes as long as the task; and if the submitter is a single-threaded reader loop, throughput drops to one task at a time. It is a reasonable default for internal pipelines, a poor one for user-facing request handling, where `AbortPolicy` plus a 503 (shed load explicitly) is usually more honest.

---

## Q20. How do you size a thread pool? Give the reasoning, not a formula you memorised.

### Answer

**Start from what the pool is protecting.** A thread pool is a *concurrency limiter* first and a performance optimisation second. The right size is the one that keeps the constrained resource fully utilised without overwhelming it.

**CPU-bound work:** threads ≈ number of available cores (sometimes cores + 1 to cover occasional page faults). Beyond that, extra threads add context-switching and cache pressure without adding throughput, because there is no more CPU to use. In a container, "cores" means the CPU limit, not the node's core count — and if the limit is fractional (500m), you have half a core and probably want a very small pool.

**I/O-bound work:** threads must cover the *waiting* time. The standard reasoning (Brian Goetz's formulation):

```
threads = cores × targetUtilisation × (1 + waitTime/serviceTime)
```

where `waitTime` is time blocked and `serviceTime` is CPU time per task. If a task spends 90 ms waiting on a database and 10 ms computing, the ratio is 9, so with 4 cores at full target utilisation you need roughly 40 threads to keep the CPUs busy. The formula's value is not the number it produces but what it tells you: **you need the actual wait/service ratio**, which you get from a profiler or from APM span data, not from guessing.

**Little's Law is the more useful framing** for a service: `L = λ × W`, where `L` is concurrent requests in the system, `λ` is arrival rate, and `W` is average latency. If you serve 500 req/s at 200 ms mean latency, you have 100 requests in flight on average — so a pool of 20 will queue and a pool of 500 is mostly idle. Size for the concurrency you actually observe, with headroom for latency spikes, and remember that when `W` rises (a slow downstream), `L` rises proportionally and the pool saturates.

**But the binding constraint is usually downstream, not the pool.** If your database accepts 20 concurrent connections usefully, a 200-thread pool merely moves the queue from your executor to the connection pool, adding latency without adding throughput — and often *reducing* throughput, since a database with 200 concurrent queries thrashes. The correct sizing exercise starts at the most constrained downstream resource and works backward.

**Practical method:** set a plausible starting value, load test, and watch (a) CPU utilisation, (b) queue depth, (c) latency percentiles, (d) downstream saturation. Increase the pool until latency degrades or a downstream saturates; that is your ceiling. Then size below it and shed load explicitly beyond that.

### Probes

**CPU-bound ≈ cores; IO-bound depends on wait/service ratio; Little's Law.** Covered above.

**Queue choice.** `ThreadPoolExecutor`'s behaviour is determined jointly by core size, max size, and queue type — and the interaction is counter-intuitive:

- **`SynchronousQueue`** — zero capacity; every task must be handed directly to a thread. Combined with a large max size, the pool grows a thread per concurrent task (this is `newCachedThreadPool`). Good for many short tasks; dangerous under load spikes.
- **`LinkedBlockingQueue` unbounded** — the queue never fills, so the pool never grows past `corePoolSize` and never rejects. This is `newFixedThreadPool`, and it is the configuration that turns overload into an OOM.
- **`ArrayBlockingQueue` / bounded `LinkedBlockingQueue`** — the one you should usually choose. The pool grows from core to max only when the queue fills, and rejects when both are full. Bounded queues are what make backpressure possible.
- **`PriorityBlockingQueue`** — unbounded and ordered; use only when you genuinely need priority, and beware starvation of low-priority tasks.

**How `ThreadPoolExecutor` only grows past core size when the queue is full.** This is the ordering rule and it surprises nearly everyone: on `execute`, if fewer than `corePoolSize` threads exist, create one; **else try to enqueue**; only if enqueueing fails, create a thread up to `maximumPoolSize`; only if that fails, reject. Therefore a large `maximumPoolSize` with an unbounded queue is dead configuration — those extra threads will never be created. If you want a pool that scales up under load, you must bound the queue.

**Rejection policies.** `AbortPolicy` (default) throws `RejectedExecutionException` — explicit, and appropriate when the caller can translate it into a 503. `CallerRunsPolicy` — backpressure (see Q19). `DiscardPolicy` — silently drops, almost always wrong. `DiscardOldestPolicy` — drops the head of the queue, which is defensible only for lossy telemetry. Custom policies can record a metric, apply a bounded blocking offer with timeout, or route to a slower path. **Whatever you choose, instrument rejections** — an unmonitored rejection is silently lost work.

**Caller-runs as crude backpressure.** Covered in Q19.

Two additions worth mentioning: `allowCoreThreadTimeOut(true)` lets the pool shrink to zero when idle, useful for memory-sensitive services with bursty load; and virtual threads change this whole calculus for I/O-bound work — instead of sizing a pool, you use `newVirtualThreadPerTaskExecutor` and impose limits with an explicit `Semaphore` at the point of the actual constraint (see Q14).

---

## Q21. What is `ThreadLocal` good for, and what are the two ways it burns you in production?

### Answer

**Legitimate uses.**

1. **Ambient request context.** Values that logically belong to a unit of work and would be intolerable to thread through every method signature: correlation/trace IDs (SLF4J `MDC`), the authenticated principal (`SecurityContextHolder`), tenant ID, locale, transaction state (`TransactionSynchronizationManager`), Hibernate's current session. Essentially every framework uses this.
2. **Per-thread instances of expensive, non-thread-safe objects.** The historical example is `SimpleDateFormat` — mutable and unsafe, so a shared instance corrupts silently. Also `Random` (though `ThreadLocalRandom` supersedes it), and reusable buffers. Note this use case is largely obsolete: `DateTimeFormatter` is immutable and thread-safe, and `ObjectMapper` is thread-safe once configured, so both should simply be `static final` singletons. Reaching for `ThreadLocal` to cache an object is now usually a sign you have the wrong object.

**The two production burns.**

**Burn 1 — memory leak on pooled threads.** `ThreadLocalMap.Entry extends WeakReference<ThreadLocal<?>>`: the *key* is weak, the *value* is strong. On a pooled thread that lives as long as the JVM:

- With the recommended `static final ThreadLocal` pattern, the key is never collected, so the value is retained until `remove()` is called or the thread dies — i.e. forever.
- If the `ThreadLocal` object itself does become unreachable, the key clears but the entry (with its strong value) remains; the map only purges such stale entries opportunistically during later `set`/`get`/`remove` calls that happen to encounter them. There is no guarantee.

Total retained memory ≈ pool size × retained object graph size. If the value transitively references a classloader, you also leak Metaspace and prevent redeployment.

**Burn 2 — cross-request data leakage.** Worse than the leak, because it is a correctness and security bug. Thread T handles request 1 for tenant A and sets a tenant `ThreadLocal`. The filter fails to clear it (an exception path bypassed the `finally`, or the code simply omitted it). Thread T is returned to the pool and picks up request 2 for tenant B. If request 2's code path reads the tenant before writing it — or if an authorisation check reads a stale principal — tenant B sees tenant A's data. This is a reportable data breach, and it is entirely silent: no exception, no log, correct-looking output.

**The rule:** every `set` in request-scoped code must have a matching `remove()` in a `finally`, in a filter or interceptor that wraps the entire unit of work. Setting to `null` is not sufficient — it leaves the entry in the map. Prefer setting context in exactly one place (a servlet `Filter`, a `HandlerInterceptor`, a Kafka listener wrapper) rather than scattering `set` calls.

### Probes

**Leaks in pooled threads (never `remove()`).** Covered.

**Stale values leaking data between requests.** Covered. Additional detection technique: in a test or staging environment, assert at the *start* of request handling that the context is empty; a non-empty context proves a leak from a prior request. This is cheap and catches the bug before production.

**`InheritableThreadLocal` and executors.** `InheritableThreadLocal` copies values from parent to child **at `Thread` construction time**. With a thread pool this is nearly useless and actively dangerous: pool threads are constructed once, so they inherit whatever context existed at *pool creation*, not at task submission, and every subsequent task on that thread sees that stale value. It is a reliable source of "why does this background task think it's running as the user who happened to trigger pool initialisation".

The correct pattern for propagating context into an executor is to capture at submission and install/clear around execution — Spring's `TaskDecorator`, Micrometer's `ContextPropagation` (`ContextSnapshot.wrap`), or `DelegatingSecurityContextExecutor`. Write the decorator once, apply it to every executor, and clear in a `finally`.

**Virtual threads changing the cost calculus.** Discussed in Q14: leaks largely disappear (the thread dies with the task) but per-thread memory multiplies across potentially millions of threads, so storing large values becomes expensive. Also, `InheritableThreadLocal` values *are* inherited by virtual threads created in a structured scope, which can be surprising. And with `newVirtualThreadPerTaskExecutor`, the "clear before reuse" concern vanishes because there is no reuse — a genuine safety improvement.

**`ScopedValue` as the modern alternative.** `ScopedValue` (JEP 429 → 446 → later revisions; check preview status for your JDK) binds a value for a *lexical/dynamic scope*:

```java
ScopedValue.where(TENANT, tenantId).run(() -> handleRequest(req));
```

Properties that fix the `ThreadLocal` problems: **immutable** (no `set`, only rebinding in a nested scope, so no accidental mutation); **automatically unbound** at scope exit even on exception, so no leak and no stale value is structurally possible; **shared, not copied** when inherited by structured-concurrency child threads, so it scales to many virtual threads; and it makes the scope of the binding visible in the code rather than implicit. The trade-off is that it requires restructuring code into a scope-passing shape, and until it is final and until frameworks adopt it, `ThreadLocal` remains the practical mechanism. Be accurate about its status rather than presenting it as available today.

---

## Q22. Explain optimistic vs pessimistic concurrency with a concrete example at three layers: a Java object, a DB row, and an HTTP resource.

### Answer

**Pessimistic:** assume conflict is likely; acquire an exclusive lock before reading, hold it through the update. Guarantees no conflict but serialises access, holds resources, and risks deadlock. Correct when contention is high or a conflict is expensive to resolve.

**Optimistic:** assume conflict is rare; read without locking, and at write time verify nothing changed since the read. If it did, fail and retry. No locks held during think-time, better throughput under low contention, but requires the caller to handle retry, and degrades badly (livelock-ish retry storms) under high contention.

**Layer 1 — a Java object.**

*Pessimistic:*
```java
synchronized (account) { account.setBalance(account.getBalance() - amount); }
```

*Optimistic (CAS):*
```java
AtomicReference<Balance> ref = ...;
Balance cur, next;
do {
    cur = ref.get();
    next = cur.minus(amount);           // immutable value
} while (!ref.compareAndSet(cur, next)); // retry if someone else won
```
`compareAndSet` is a single hardware instruction (`lock cmpxchg` on x86). This is the foundation of `AtomicInteger`, `ConcurrentHashMap`, and `AbstractQueuedSynchronizer`.

**Layer 2 — a database row.**

*Pessimistic:* `SELECT ... FOR UPDATE` (JPA: `LockModeType.PESSIMISTIC_WRITE`) takes a row lock held until commit; any other transaction attempting to lock the same row blocks. Add `NOWAIT` to fail immediately or `SKIP LOCKED` to skip contended rows (essential for queue-table patterns).

*Optimistic:* a `@Version` column. Hibernate issues `UPDATE ... SET ..., version = 6 WHERE id = ? AND version = 5`. If zero rows are affected, someone else updated it and Hibernate throws `OptimisticLockException`. No locks are held between read and write, so a user can hold an edit form open for ten minutes without blocking anyone.

**Layer 3 — an HTTP resource.**

*Optimistic:* the server returns `ETag: "5"` with `GET`. The client sends `PUT`/`PATCH` with `If-Match: "5"`. If the current ETag differs, the server returns `412 Precondition Failed` and the client refetches, merges, retries. `428 Precondition Required` tells a client that omitted `If-Match` that it must supply one — which is how you make optimistic concurrency mandatory rather than optional.

*Pessimistic:* an explicit lock/checkout resource (`POST /documents/1/lock` returning a lock token required for subsequent writes). Rare in REST because it is stateful and requires lock expiry, orphan-lock recovery, and a heartbeat — all of which reintroduce the problems locking was meant to avoid. WebDAV specifies it; most APIs avoid it.

**Choosing.** Optimistic when conflicts are rare, think-time is long (human editing), or you cannot hold a transaction open across a user interaction. Pessimistic when conflicts are common, retry is expensive or has side effects, or you need to serialise access to a scarce resource (last item in stock, seat booking).

### Probes

**CAS/`AtomicReference` + ABA.** ABA: thread 1 reads value A; threads 2 and 3 change it to B and back to A; thread 1's CAS succeeds even though the state changed in between. Harmless for a counter (the value is all that matters) but broken for pointer-based structures where "same reference" does not mean "unchanged structure" — a classic lock-free-stack bug where a popped-and-reused node lets a CAS succeed on a stale view. Solutions: `AtomicStampedReference` (reference + monotonically increasing stamp) or `AtomicMarkableReference`; or version the value; or avoid node reuse. Java's garbage collector actually prevents the most common form (a node cannot be reclaimed while a thread holds a reference), which is why ABA is rarer in Java than in C++.

**`@Version` / `SELECT FOR UPDATE`.** Covered. Notes: `@Version` may be `int`, `long`, `short`, or `Timestamp` — prefer an integer type; timestamps have clock-resolution and clock-skew problems that can cause missed conflicts. The version column must be managed by the persistence provider; manual updates break it. `LockModeType.OPTIMISTIC` forces a version *check* even on a read-only entity at commit; `OPTIMISTIC_FORCE_INCREMENT` bumps the version when you modify a child but need the parent aggregate's version to change (useful for aggregate-level consistency in DDD).

**Lost-update anomaly.** Two transactions read balance 100, one subtracts 30, the other subtracts 50, both write; the final balance is 70 or 50 instead of 20. Under `READ COMMITTED` this is entirely possible with a read-then-write pattern. It is prevented by: optimistic version checking, pessimistic locking, `SERIALIZABLE`/`REPEATABLE READ` depending on engine, or — best — by not reading first at all and issuing a single atomic statement (below).

**Row-level locks and deadlock ordering.** Pessimistic locking creates deadlock risk whenever two transactions lock multiple rows in different orders. Mitigations: always acquire locks in a deterministic order (e.g. ascending primary key — if you update multiple accounts, sort the IDs first); keep transactions short; set `lock_timeout` so a stuck transaction fails rather than hanging; and expect to retry, since the database will kill one transaction as the deadlock victim. Deadlock is a *normal, recoverable* outcome to be retried, not necessarily a bug — though a rising rate is.

**Single atomic `UPDATE ... SET qty = qty - 1 WHERE qty >= 1` as the often-better answer.** For simple decrements this beats both locking strategies. The database performs the read and write atomically under its own row lock, held for microseconds; the `WHERE qty >= 1` guard prevents going negative; and the affected-row count tells you the outcome (`1` = success, `0` = insufficient stock). No version column, no retry loop, no `OptimisticLockException`, minimal lock duration, and it works correctly at `READ COMMITTED`.

The caveats: it bypasses the JPA persistence context, so any in-memory entity for that row is now stale (call `entityManager.refresh` or, better, do not hold the entity); it does not compose when the decision depends on several rows or on business logic that cannot be expressed in SQL; and under extreme contention on a single row you still serialise on that row lock, so a hot SKU becomes a bottleneck regardless of technique (the answer there is sharding the counter into N sub-rows and summing, accepting approximate reads).

**Idempotency of the retry.** Central and often missed. If you retry an optimistic failure, the retried operation must not double-apply side effects. Safe: re-read state and recompute the new value from scratch (the CAS loop and the JPA retry both do this). Unsafe: retrying a method that has already sent an email, published a Kafka event, or charged a card. Therefore: put external side effects *after* the successful commit (via `TransactionSynchronization.afterCommit` or an outbox), keep the retried region purely computational, and give the whole operation an idempotency key if it crosses a service boundary. Bound the retries (three is typical) with jittered backoff, and surface exhaustion as a real error rather than looping forever.

---

## Q23. What is false sharing and how would you even notice it?

### Answer

**Mechanism.** CPU caches operate on **cache lines**, typically 64 bytes, not on individual variables. Cache coherence protocols (MESI and variants) maintain consistency at line granularity. If two threads on different cores write to two *distinct* variables that happen to occupy the same cache line, each write invalidates the other core's copy of the whole line. The variables are logically independent, but the hardware treats them as one unit — so the cores ping-pong the line back and forth, each write incurring a coherence miss costing tens to hundreds of cycles.

The result: code that appears perfectly parallel and lock-free scales *negatively* — adding threads makes it slower. This is why the phenomenon is sometimes called "the silent performance killer": there is no lock, no contention visible in a thread dump, and no correctness problem.

**Canonical example.** An array of per-thread counters:

```java
long[] counters = new long[numThreads];   // 8 bytes each — 8 counters per 64-byte line
// thread i does: counters[i]++
```
Eight threads writing to `counters[0..7]` all hammer one cache line. Padding each counter to occupy its own line eliminates the effect and can improve throughput by an order of magnitude.

**In real code** it typically appears in: arrays of per-thread state; adjacent hot fields in an object written by different threads (a producer writing `head` and a consumer writing `tail` in a ring buffer — the classic Disruptor motivation); and object headers/fields of small objects allocated adjacently.

**How you notice it.**

1. **The scaling signature.** Throughput fails to increase, or decreases, with more threads, while CPU utilisation stays high and no lock contention appears in profiling. That combination — high CPU, no locks, no scaling — should immediately raise false sharing as a hypothesis.
2. **Hardware performance counters.** This is the definitive method. On Linux, `perf stat -e cache-misses,LLC-load-misses,mem_load_l3_hit_retired.xsnp_hitm` (the exact event names are CPU-specific). The event that matters is **HITM** — a load that hits a *modified* line in another core's cache, which is the direct signature of coherence ping-pong. `perf c2c` (cache-to-cache) is purpose-built for this: it reports which cache lines are shared between cores and which offsets within them are being written, effectively naming the variables.
3. **JMH.** Write a benchmark with and without padding and compare. JMH also provides `@State` classes and the `-prof perfasm`/`-prof perfnorm` profilers that surface the counters directly.
4. **Intel VTune / AMD uProf** report false sharing explicitly with source attribution — the least painful route if available.

### Probes

**Cache line granularity.** Covered. Add: line size is typically 64 bytes on x86-64 and modern ARM, but 128 bytes is used by some architectures and by Intel's adjacent-line prefetcher, which effectively makes the sharing unit 128 bytes. This is why `@Contended` pads to 128 by default (`-XX:ContendedPaddingWidth`).

**`@Contended`.** `jdk.internal.vm.annotation.Contended` instructs the JVM to pad a field (or an entire class) so it occupies its own cache line. It requires `-XX:-RestrictContended` to be usable outside the JDK, and since JDK 9 the package is not exported to application code without `--add-exports`, so in practice **application code cannot easily use it** — a fact worth knowing, because candidates often propose it as if it were readily available. The JDK uses it internally in `LongAdder`'s `Cell`, `ForkJoinPool`, and `Thread`'s random seed fields. For application code the practical alternatives are manual padding (declaring unused `long` fields around the hot field — fragile, since the JIT may reorder or eliminate them), or restructuring to avoid the sharing (per-thread objects rather than arrays of primitives, which also lets the allocator separate them).

**JMH + perf counters.** Covered above. The key discipline: false sharing is a *microarchitectural* effect that ordinary profilers cannot see, because it shows up as time spent in ordinary instructions, not in any identifiable method. You need either a controlled A/B benchmark or hardware counters. Sampling profilers will point at the hot loop, which you already knew.

**Why this is usually a last-resort concern.** Being honest about this is the mark of a senior answer. False sharing matters in a very narrow band of code: lock-free data structures, high-frequency counters, message-passing ring buffers, and other work at the level of tens of millions of operations per second per core. In a typical Spring service, the time is spent in JDBC round-trips, JSON serialisation, HTTP, and GC — false sharing is not measurable against that background. Optimising for it first is a textbook case of misplaced effort.

The right framing: know what it is so you recognise the scaling signature if you ever meet it, know that `LongAdder` exists precisely to solve it for the common counter case (and use `LongAdder` rather than hand-padding), and otherwise leave it alone until measurement says otherwise. If asked "have you ever fixed one?", answering "no, and here is why I would expect not to in the systems I have worked on, but here is how I would identify one" is a better answer than an invented war story.

---

# 3. Spring Framework Core

---

## Q24. Trace the lifecycle of a singleton bean from container start to shutdown, naming the extension points.

### Answer

**Phase A — Bean definition loading.** The `ApplicationContext` reads bean *definitions* (metadata: class, scope, dependencies, init/destroy methods) from component scanning, `@Configuration` classes, XML, or programmatic registration. Nothing is instantiated yet.

**Phase B — `BeanDefinitionRegistryPostProcessor`.** Runs first. Can **add, remove, or modify bean definitions**. This is how `ConfigurationClassPostProcessor` works — it parses `@Configuration` classes, processes `@Bean` methods, `@Import`, `@ComponentScan`, and registers the resulting definitions. It is the earliest meaningful hook.

**Phase C — `BeanFactoryPostProcessor`.** Runs after all definitions are registered. Can modify definition *metadata* but not add/remove. `PropertySourcesPlaceholderConfigurer` (resolving `${...}`) lives here.

**Phase D — For each singleton, in dependency order:**

1. **Instantiation** — constructor invoked (after `SmartInstantiationAwareBeanPostProcessor.determineCandidateConstructors` picks which one). Constructor-injected dependencies are resolved and created first.
2. **Population** — field and setter injection. `@Autowired`/`@Value`/`@Resource` are processed by `AutowiredAnnotationBeanPostProcessor` and `CommonAnnotationBeanPostProcessor`, both of which are `InstantiationAwareBeanPostProcessor`s.
3. **`Aware` callbacks** — `BeanNameAware`, `BeanClassLoaderAware`, `BeanFactoryAware`, then (via `ApplicationContextAwareProcessor`) `EnvironmentAware`, `ResourceLoaderAware`, `ApplicationEventPublisherAware`, `ApplicationContextAware`.
4. **`BeanPostProcessor.postProcessBeforeInitialization`** — for every registered BPP.
5. **Initialisation callbacks**, in this order: `@PostConstruct` → `InitializingBean.afterPropertiesSet()` → the `initMethod` named on `@Bean`.
6. **`BeanPostProcessor.postProcessAfterInitialization`** — **this is where proxies are created.** `AbstractAutoProxyCreator` (the base of AOP, `@Transactional`, `@Async`, `@Cacheable`) returns a *different object* — the proxy — from this method, and that proxy is what gets stored in the singleton cache and injected everywhere.

**Phase E — `SmartInitializingSingleton.afterSingletonsInstantiated()`** — called on beans implementing it once *all* singletons are created. The right hook when you need the complete set of beans (e.g. collecting all `Strategy` implementations and validating there are no duplicates).

**Phase F — `Lifecycle` / `SmartLifecycle.start()`** — for beans managing background processes (Kafka listener containers, schedulers, embedded servers). `SmartLifecycle` adds a `getPhase()` for ordering and `isAutoStartup()`.

**Phase G — `ContextRefreshedEvent`** published; in Boot, `ApplicationReadyEvent` after the runner phase.

**Shutdown (reverse order):** `ContextClosedEvent` → `SmartLifecycle.stop()` (reverse phase order) → `@PreDestroy` → `DisposableBean.destroy()` → the `destroyMethod` named on `@Bean`. Destruction callbacks only run if the JVM shuts down cleanly and a shutdown hook is registered (Spring Boot registers one by default) — `kill -9` runs nothing.

### Probes

**The ordering chain.** As enumerated. The three-way initialisation ordering (`@PostConstruct` → `afterPropertiesSet` → `initMethod`) and its mirror at destruction is a common precise-recall question. Prefer `@PostConstruct`/`@PreDestroy`: they do not couple your class to Spring interfaces. Note that in Spring 6 / Boot 3, `@PostConstruct` moved to `jakarta.annotation` and requires the `jakarta.annotation-api` dependency (present transitively in most Boot starters).

**Why AOP proxies appear in the *after* phase and what that means for self-references.** The proxy must wrap a fully initialised target, so it can only be created once initialisation callbacks have run. Two important consequences:

1. **`@PostConstruct` runs on the raw target, not the proxy.** So calling a `@Transactional` or `@Async` method from `@PostConstruct` gets no transaction and no async — the proxy does not exist yet.
2. **Self-invocation bypasses the proxy** (Q25), and any reference the bean holds to `this` is the raw object.

There is a subtlety worth knowing: when a **circular dependency** exists, Spring must expose an early reference before initialisation completes. It does this via `getEarlyBeanReference` on `SmartInstantiationAwareBeanPostProcessor`, which creates the proxy *early* for that case. Spring then checks at the end that the object it exposed matches the final one and throws `BeanCurrentlyInCreationException` if a mismatch would leave some beans holding the raw object and others the proxy. This is one of the reasons circular dependencies with AOP are genuinely dangerous rather than merely inelegant.

Another practical implication: a `BeanPostProcessor` is itself a bean, and it must be instantiated before the beans it processes. So BPPs (and their dependencies) are created very early, **before** `BeanFactoryPostProcessor`-driven property resolution is necessarily complete for them, and they are not eligible for post-processing themselves. Injecting a heavyweight dependency into a BPP forces that dependency to be created early, often producing the warning "is not eligible for getting processed by all BeanPostProcessors" — meaning that bean silently misses out on AOP, transactions, and configuration property binding. This is a real, subtle production bug: a `@Transactional` service injected into a BPP gets no transactions.

---

## Q25. Why does calling a `@Transactional` method from another method in the same class not open a transaction? Give me three ways to fix it and their trade-offs.

### Answer

**The mechanism.** Spring's `@Transactional` is implemented with **proxy-based AOP**. The container registers the bean in the context as a proxy wrapping your object. When another bean calls `service.doWork()`, the call goes through the proxy, which starts a transaction, delegates to the target, and commits or rolls back.

But when code *inside* the target calls another of its own methods, the call is `this.otherMethod()` — a direct virtual invocation on the target object. The proxy is not in the call path at all. No interception, no transaction. Same for `@Async`, `@Cacheable`, `@Retryable`, `@PreAuthorize`, `@Timed`, and every other proxy-based annotation.

```java
@Service
public class OrderService {
    public void process(List<Order> orders) {
        orders.forEach(this::saveOne);          // no transaction, ever
    }
    @Transactional
    public void saveOne(Order o) { repo.save(o); }
}
```

The failure is silent: `repo.save` still works because Spring Data repository methods are themselves transactional, but the *intended* boundary (one transaction per order, or rollback semantics you expected) does not exist. That silence is what makes this the single most common Spring bug.

**Fix 1 — extract to a separate bean.** Move the transactional method into another Spring-managed class and inject it.

*Trade-offs:* the cleanest and most idiomatic; the proxy is genuinely in the path; testable in isolation. Cost: you may create an anaemic class that exists only to satisfy the proxy, which can feel like framework-driven design. Usually the right answer — and often the extraction reveals that the two responsibilities genuinely were separate.

**Fix 2 — `TransactionTemplate` (programmatic transactions).**

```java
private final TransactionTemplate tx;
public void process(List<Order> orders) {
    orders.forEach(o -> tx.execute(status -> { saveOne(o); return null; }));
}
```

*Trade-offs:* explicit, no proxy involvement, no self-invocation problem, and the boundary is visible in the code rather than hidden in an annotation. Also the only clean way to express "transaction around part of a method". Cost: more verbose; couples the class to Spring's transaction API; lambdas returning `null` are slightly awkward (`TransactionCallbackWithoutResult` or `executeWithoutResult` in Spring 5.3+ helps). This is arguably the most *honest* fix and is underused.

**Fix 3 — self-injection.**

```java
@Service
public class OrderService {
    @Autowired @Lazy private OrderService self;   // or ObjectProvider<OrderService>
    public void process(...) { orders.forEach(self::saveOne); }
}
```

*Trade-offs:* minimal change, keeps the code together. Costs: it is a circular dependency (needs `@Lazy` or `ObjectProvider` to avoid failing under Boot 2.6+'s default ban on cycles); it is confusing to readers who do not know why it exists; and it makes unit testing awkward (you must set `self` manually). Acceptable as a pragmatic fix, but it advertises that the design wants splitting.

**Fix 4 — `AopContext.currentProxy()`.** Requires `@EnableAspectJAutoProxy(exposeProxy = true)`, then `((OrderService) AopContext.currentProxy()).saveOne(o)`. *Trade-offs:* works, but binds your code to Spring AOP internals, requires a cast, and depends on a global configuration flag that is easy to lose. Generally discouraged.

**Fix 5 — AspectJ load-time or compile-time weaving.** `@EnableTransactionManagement(mode = AdviceMode.ASPECTJ)` plus weaving. This modifies the bytecode of the class itself, so **self-invocation is intercepted**, as are `private` and `final` methods and calls on non-Spring-managed objects. *Trade-offs:* it genuinely solves the whole class of problem — and it introduces a weaving agent or a build-time weaving step, harder debugging, unfamiliar failure modes, and an extra thing for every developer and every environment to configure. In practice most teams accept the proxy limitations rather than adopt AspectJ; it is worth knowing as the "correct but costly" answer.

### Probes

**Proxy-based AOP; self-invocation bypasses the proxy.** Covered.

**Fixes and their trade-offs.** Covered above (five, rather than three — an interviewer will appreciate knowing the full landscape and which you would actually choose).

**Same issue with `@Async`, `@Cacheable`, `@Retryable`.** All are proxy-based, all fail identically and silently. `@Async` is the most insidious: a self-invoked `@Async` method runs synchronously on the caller's thread, so the code works — just not concurrently — and the bug is invisible until a load test. `@Cacheable` self-invoked simply executes the method every time, so behaviour is correct but the cache does nothing; you discover it when you notice the hit ratio is zero (which is an argument for always exposing cache statistics). `@PreAuthorize` self-invoked skips the authorisation check entirely, which is a security hole rather than a performance issue.

Two related proxy limitations worth mentioning in the same breath, since they have the same root cause:
- **`private` methods are never intercepted** — a CGLIB proxy subclasses the target and can only override visible, non-final methods. Annotating a private method with `@Transactional` compiles and does nothing. (Spring will log a warning for some cases, but not reliably.)
- **`final` classes and `final` methods** cannot be CGLIB-proxied at all. A `final` `@Service` class fails at startup with a proxy creation error, and a `final` method on a proxied class is silently not intercepted. This is a common surprise with Kotlin, where classes and methods are final by default — hence the `kotlin-spring` (all-open) compiler plugin.

---

## Q26. JDK dynamic proxy vs CGLIB — when does Spring pick each, and what breaks under each?

### Answer

**JDK dynamic proxy** (`java.lang.reflect.Proxy`) generates, at runtime, a class implementing a given set of **interfaces**, delegating every call to an `InvocationHandler`. It requires interfaces; it cannot proxy a class. The proxy is *not* an instance of your class — it is a sibling implementing the same interfaces.

**CGLIB** (repackaged inside `spring-core` since Spring 3.2, so no separate dependency) generates a **subclass** of the target class at runtime and overrides its methods. It requires a non-final class with a visible constructor, and it cannot override `final` or `private` methods.

**Which Spring picks.** Historically: JDK proxies if the target implements at least one interface, CGLIB otherwise; forced to CGLIB with `proxyTargetClass = true`. **Spring Boot changed the default to `proxyTargetClass = true`** — `@EnableTransactionManagement`, `@EnableAsync`, `@EnableCaching` are auto-configured with CGLIB, and Spring AOP defaults to class-based proxying in Boot applications. Spring Framework 6 moved further in this direction. So in a modern Boot application you should assume **CGLIB unless you have deliberately configured otherwise**.

**What breaks under JDK proxies:**

- **You can only inject by interface.** `@Autowired MyServiceImpl impl` fails with a `BeanNotOfRequiredTypeException`: the proxy implements `MyService` but is not a `MyServiceImpl`. This is the classic "it worked until I added `@Transactional`" failure.
- **Methods not on the interface are unreachable** through the proxy.
- **`instanceof MyServiceImpl` is false**; casting throws.
- **Annotations on the implementation class are not visible** on the proxy's interface methods for some reflection-based tooling.

**What breaks under CGLIB:**

- **`final` classes cannot be proxied** — startup failure. **`final` methods are silently not intercepted** — no error, just missing behaviour. Same for `private` and `static` methods.
- **A non-private, non-arg-less constructor is needed.** Modern CGLIB in Spring uses Objenesis to instantiate without calling a constructor, which mostly removes this constraint but introduces the next one.
- **Constructor side effects and field initialisers.** Historically CGLIB invoked the target's constructor a second time on the proxy instance, so any side effect ran twice. With Objenesis the constructor is *not* called on the proxy at all — meaning **field initialisers and constructor-assigned fields are null/default on the proxy object**. This is normally invisible because every method call is delegated to the real target, but it breaks any code that reads a field directly through the proxy reference, and it is very confusing in a debugger where the proxy's fields all appear empty.
- **Self-invocation is still not intercepted** (the subclass's overridden method calls `super`, and internal `this` calls go to the subclass's inherited implementations without advice — the proxy is a distinct object from the target in Spring's implementation).
- **Kotlin**: classes and members are `final` by default, so CGLIB proxying fails until the `kotlin-allopen`/`kotlin-spring` plugin is applied.

**Neither can proxy:** `private` methods, `static` methods, constructors, fields, or calls made on objects Spring did not create.

### Probes

**Interface-based vs subclass.** Covered.

**`final` classes/methods.** Covered. Add: `record`s are implicitly final and therefore not CGLIB-proxyable — relevant if someone tries to make a record a `@Configuration` class or a proxied bean.

**`private` methods.** Never intercepted by either mechanism, in any configuration, and annotating them is silently ineffective.

**Constructor invoked twice (older CGLIB).** Covered above, with the modern Objenesis behaviour and its own trap. The interview-safe statement: "older Spring versions invoked the constructor twice; modern Spring uses Objenesis so the constructor is not invoked on the proxy at all, which means the proxy's own fields are uninitialised — harmless for delegated method calls, confusing in a debugger, and broken if anything reads fields directly."

**`proxyTargetClass=true`.** Set on `@EnableTransactionManagement`, `@EnableAsync`, `@EnableCaching`, `@EnableAspectJAutoProxy`, or via `spring.aop.proxy-target-class=true`. Forces CGLIB. The main reason to want it: it removes the "must inject by interface" constraint and makes proxying behave uniformly whether or not a class happens to have an interface — which is exactly why Boot made it the default.

**Why Spring Boot defaults to CGLIB.** Because interface-or-not should not silently change injection semantics. Under the old default, adding an interface to a class changed the proxy type and could break `@Autowired` by concrete type in an unrelated part of the codebase. CGLIB everywhere is more predictable. The trade is that `final` becomes hazardous.

**`instanceof` and injection-by-concrete-type surprises.** Under JDK proxies, injecting or casting to the concrete class fails. Under CGLIB, `instanceof MyServiceImpl` is *true* (the proxy is a subclass), but `getClass()` returns something like `MyServiceImpl$$SpringCGLIB$$0`, so `getClass() == MyServiceImpl.class` is false and `getClass().getSimpleName()` returns a mangled name. Code that switches on `getClass()`, uses it as a map key, or logs it will misbehave. `AopUtils.getTargetClass(bean)` and `AopProxyUtils.ultimateTargetClass(bean)` return the real class; `AopTestUtils.getTargetObject(bean)` unwraps the target in tests. Also relevant: reading annotations off `bean.getClass()` may miss them (CGLIB subclasses do not inherit non-`@Inherited` annotations) — use `AnnotationUtils`/`MergedAnnotations` with the target class instead.

---

## Q27. Explain all `@Transactional` propagation modes with a scenario where the "obvious" choice is wrong.

### Answer

Propagation defines what happens when a transactional method is invoked with or without an existing transaction.

| Mode | No existing transaction | Existing transaction |
|---|---|---|
| `REQUIRED` (default) | Start a new one | **Join it** |
| `REQUIRES_NEW` | Start a new one | **Suspend** the outer, start an independent one |
| `SUPPORTS` | Run non-transactionally | Join it |
| `NOT_SUPPORTED` | Run non-transactionally | **Suspend** the outer, run non-transactionally |
| `MANDATORY` | Throw `IllegalTransactionStateException` | Join it |
| `NEVER` | Run non-transactionally | Throw `IllegalTransactionStateException` |
| `NESTED` | Start a new one | Create a **savepoint** within it |

**Key semantic points:**

- `REQUIRED` joining means **one physical transaction**. An inner method's "rollback" marks the *shared* transaction rollback-only; the outer method cannot commit. It is not a nested unit of work despite looking like one.
- `REQUIRES_NEW` uses a **second physical database connection**. The outer transaction's uncommitted changes are invisible to it (they are uncommitted, in a different session). This is the source of the most common surprise: a `REQUIRES_NEW` method that queries a row the outer transaction just inserted will not find it.
- `NESTED` uses **JDBC savepoints** — a single connection, single transaction, with a rollback point. The inner work can be rolled back without killing the outer transaction, but if the outer rolls back, the inner is lost too. Requires a `DataSourceTransactionManager` with savepoint support; **`JpaTransactionManager` supports it only with certain configurations, and it is not supported by all databases** — verify before relying on it.
- `MANDATORY` is genuinely useful: put it on repository/domain methods that must never run outside a transaction, and you get a loud failure instead of silent auto-commit.

**Scenario where the obvious choice is wrong.**

*The setup:* an order service that must write an audit record even when order processing fails.

```java
@Transactional                                    // REQUIRED
public void placeOrder(Order o) {
    audit.record("attempting", o);                // REQUIRES_NEW — "so it survives rollback"
    inventory.reserve(o);                         // may throw
    orders.save(o);
}
```

`REQUIRES_NEW` on the audit *does* work for the stated goal — the audit commits independently and survives the outer rollback. But it is frequently the wrong choice:

1. **It consumes a second connection while the first is held.** With a HikariCP pool of 10 and 10 concurrent requests each holding one connection and requesting a second, the pool is exhausted and every thread waits for a connection that no one can release: a **self-inflicted distributed deadlock**. This is a real production outage pattern and it only appears under concurrency, so tests pass. If you use `REQUIRES_NEW` at all, the pool must be sized for the maximum nesting depth.
2. **The inner transaction cannot see the outer's uncommitted data.** If `audit.record` needs to read the order row the outer transaction just wrote, it sees nothing — or worse, blocks on a row lock held by the outer transaction, deadlocking against itself.
3. **It doubles transaction overhead** and lengthens the outer transaction's duration (the outer connection is held while the inner one does its work).

*Better options:* buffer the audit in memory and write it after the outer transaction resolves, via a `TransactionSynchronization`; or write the audit to a different store entirely (a log aggregator, an append-only event stream); or use the **transactional outbox** if the audit must be exactly consistent with the business data — in which case it should be *inside* the same transaction, not outside it.

**A second common wrong choice:** using `REQUIRED` (the default) for a batch loop, so that one bad record rolls back the whole batch. `REQUIRES_NEW` or a `TransactionTemplate` per item is right there — but note the connection-pool point again, and remember that the self-invocation rule (Q25) means the per-item method must be on another bean.

### Probes

**`REQUIRES_NEW` needing a second connection (pool exhaustion/deadlock).** Covered in detail above. The arithmetic to state: with pool size P and nesting depth D, you can safely support only ⌊P/D⌋ concurrent requests before deadlock.

**`NESTED` needing savepoints and JDBC support.** Covered. Practical guidance: `NESTED` is attractive in theory (partial rollback on one connection) and rarely used in practice because support is patchy — `JpaTransactionManager` historically did not support it without additional configuration, and Hibernate's flush ordering interacts with savepoints in non-obvious ways. If you need partial rollback with JPA, the reliable route is usually restructuring so the risky work is a separate transaction, or catching and compensating explicitly.

**Audit-log-must-survive-rollback case.** Covered above, including the reasons `REQUIRES_NEW` is a poor default fit and the alternatives.

**`SUPPORTS`/`NOT_SUPPORTED` pitfalls.** `SUPPORTS` is deceptive: without a transaction, each statement runs in **auto-commit**, so a method that issues three writes produces three independently committed units — no atomicity, and no error to tell you. With Hibernate it is worse: outside a transaction there is no persistence context bound to the request, so lazy loading fails, dirty checking does not happen, and the session lifecycle is per-operation. `SUPPORTS` is almost never what you want; `MANDATORY` or `REQUIRED` express intent better.

`NOT_SUPPORTED` suspends the current transaction, which means the outer connection is held (idle, uncommitted, still holding locks) while the inner work runs on a different connection — so it has the same pool-pressure and lock-duration problems as `REQUIRES_NEW`. Its legitimate use is a long read-only report that you do not want participating in a write transaction's locks, and even then the better design is to move it out of the transactional path entirely.

**`readOnly=true` effects on Hibernate flush mode.** More than a hint:

- Spring sets the JDBC connection to read-only (`Connection.setReadOnly(true)`), which some databases and proxies use for **routing to a read replica** — this is the main practical reason to set it, since it makes replica routing declarative.
- For Hibernate, Spring sets `FlushMode.MANUAL`, so the persistence context does **not** flush at commit. That skips dirty-check snapshotting on every managed entity, which is a real performance saving on queries that load many entities.
- **Modifications are silently not persisted.** Marking a method `readOnly=true` and then mutating an entity produces no `UPDATE` and no error — the change is simply lost. That silence makes `readOnly` a genuine footgun when someone later adds a write to an existing read method.
- Some databases (PostgreSQL with an explicitly read-only transaction) will actively reject writes with an error; others silently allow them. Behaviour is therefore inconsistent across environments, which is another reason to treat `readOnly` as a real contract rather than an optimisation hint.

---

## Q28. Your `@Transactional` method calls an external REST API and then writes to the DB. What's wrong with that, and how do you fix it?

### Answer

**What's wrong.**

1. **A database connection is held for the duration of a network call.** The transaction begins, acquires a connection from the pool, and holds it while you wait — potentially seconds — on an HTTP call. With a pool of 10 and a downstream at 2 s latency, throughput collapses to ~5 requests/second regardless of how many threads you have. Connection pool exhaustion follows, then `connectionTimeout` failures on unrelated endpoints, then a service-wide outage caused by one slow dependency.

2. **Database locks are held for the duration of the network call.** If the transaction has already written or taken a `FOR UPDATE` lock, those locks persist across the call. Other transactions block. Deadlock probability rises with lock duration; a slow downstream turns lock contention into a cascade.

3. **Timeout amplification.** The transaction timeout, the JDBC socket timeout, the HTTP read timeout, and any upstream client timeout must all be reasoned about together. If the HTTP call has no timeout (many defaults are infinite), a hung downstream holds the connection *forever* and the pool never recovers.

4. **The external call cannot be rolled back.** If the HTTP call succeeds and the database commit then fails, the remote side has been mutated and your side has not. You have created an inconsistency that no transaction manager can repair. If the call is a payment authorisation or an email send, that is a business problem, not a technical one.

5. **Retries double-apply.** If the transaction is retried (optimistic lock failure, deadlock victim), the HTTP call is re-executed. Without an idempotency key, you charge the card twice.

**Fixes, in order of preference.**

**(a) Move the call outside the transaction.** Usually the simplest and best:

```java
public void placeOrder(Order o) {
    PricingResult p = pricingClient.quote(o);   // no transaction open
    persistOrder(o, p);                          // separate bean, @Transactional, fast
}
```
Do the I/O first, then open a short transaction to persist. Or read what you need in a short transaction, do the call, then open another short transaction to write — accepting that the state may have changed in between and handling it with optimistic locking.

**(b) `TransactionSynchronization.afterCommit` for fire-after-write.** If the call must happen only if the write succeeds (send a confirmation email after the order is saved):

```java
TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
    @Override public void afterCommit() { notifier.send(order); }
});
```
Or declaratively, `@TransactionalEventListener(phase = AFTER_COMMIT)`. The connection is released before the callback runs. **Caveat:** if the callback fails, the transaction has already committed and cannot be undone — you have at-most-once semantics with silent loss. Acceptable for a confirmation email; not acceptable for anything another system depends on.

**(c) Transactional outbox** when the external effect must be guaranteed. Write an outbox row inside the same transaction; a separate relay reads it and performs the call with retries. This converts a dual-write problem into a single-write problem plus at-least-once delivery, which the receiver deduplicates (Q76).

**(d) Saga / compensating action** when the external call must happen *before* your write and cannot be deferred (reserving inventory in another service, authorising a payment). Model the flow as steps with explicit compensations, and accept eventual consistency (Q77).

**(e) If none of the above is possible**, minimise the damage: set aggressive connect and read timeouts on the HTTP client, use a circuit breaker, set an explicit `@Transactional(timeout = n)`, size the pool for the worst-case hold time, and isolate the endpoint behind its own connection pool and thread pool so it cannot starve the rest of the application.

### Probes

**Connection held during network IO.** Covered, with the throughput arithmetic. Also note this is precisely why `spring.jpa.open-in-view=true` (the Boot default) is dangerous: it holds the persistence context — and, depending on configuration, the connection — for the entire request including view rendering and any I/O the controller performs (Q43).

**Timeout amplification.** Every layer needs a timeout, and they must form a decreasing sequence from the outside in, so that an inner timeout fires before the outer one gives up. Concretely: client timeout > server request timeout > transaction timeout > HTTP call read timeout + connect timeout, with headroom for retries (a call with 3 retries at 2 s each needs at least 6 s of budget upstream). Also set `spring.transaction.default-timeout` or per-method `@Transactional(timeout=...)` so a runaway transaction is killed rather than holding a connection indefinitely — note this maps to the JDBC statement query timeout and is enforced per statement, not as a hard wall-clock bound on the whole method.

**Non-transactional side effects that can't roll back.** Covered. The general principle: a transaction gives you atomicity only over resources that participate in it. HTTP calls, file writes, message publishes to a non-transactional broker, and in-memory cache updates do not participate. Every one of them is a potential inconsistency point on rollback. Cache updates deserve specific mention — updating a cache inside a transaction that then rolls back leaves the cache holding data that was never committed, which is a stale-read bug that persists until TTL. Update caches in `afterCommit`.

XA/two-phase commit exists to make heterogeneous resources atomic, and is generally avoided in modern architectures: it requires a transaction coordinator, it holds locks across the prepare phase (so a coordinator failure blocks participants indefinitely), it performs poorly, and support in cloud-managed databases and message brokers is limited or absent. The outbox pattern is the standard replacement.

**Transactional outbox.** See Q76 for full detail.

**`TransactionSynchronizationManager.afterCommit`.** Covered. Additional detail: the available phases are `BEFORE_COMMIT` (still inside the transaction — a throw here rolls it back, which is useful for last-moment validation), `AFTER_COMMIT`, `AFTER_ROLLBACK`, and `AFTER_COMPLETION`. `@TransactionalEventListener` exposes these directly. A frequently-hit trap: an `AFTER_COMMIT` listener that tries to write to the database gets **no transaction** by default and runs in auto-commit — you must annotate the listener with `@Transactional(propagation = REQUIRES_NEW)` if it needs one. Another: if there is no active transaction when the event is published, a `@TransactionalEventListener` does **not fire at all** by default (`fallbackExecution = false`), which surprises people in tests.

**Shrinking transaction scope.** The general discipline: a transaction should span only the database work that must be atomic. Everything else — validation, mapping, external calls, serialisation, business rule evaluation that does not need the database — belongs outside. In practice this usually means the transactional annotation belongs on a narrow persistence-facing method, not on a broad orchestration method, which is the opposite of where most codebases put it. Measure it: transaction duration is a metric worth graphing, and a p99 above a few tens of milliseconds usually indicates I/O inside a transaction.

---

## Q29. Which exceptions roll back a transaction by default, and why is that default a footgun?

### Answer

**The default rule.** Spring's `DefaultTransactionAttribute` rolls back on `RuntimeException` and `Error`, and **commits** on checked exceptions. (This mirrors the EJB convention, which is where it originated.) So:

```java
@Transactional
public void transfer(...) throws InsufficientFundsException {   // checked
    debit(from, amount);
    if (balanceTooLow) throw new InsufficientFundsException();  // COMMITS the debit
    credit(to, amount);
}
```

The debit is committed. The transfer is half-applied. Nothing in the code or the annotation hints at this.

**Why it is a footgun.**

1. **It is counter-intuitive.** Most developers assume "exception thrown → transaction rolled back". The distinction between checked and unchecked has no obvious connection to whether data should persist.
2. **It is invisible.** No warning, no log, no failure. The transaction commits and returns normally from the container's perspective.
3. **It couples an unrelated design decision to transactional behaviour.** Changing an exception from unchecked to checked — a purely API-level refactor — silently changes persistence semantics.
4. **The rule is per-method and easy to forget** when adding a new checked exception to an existing method.

**Fixes:** `@Transactional(rollbackFor = Exception.class)` on methods that throw checked exceptions (many teams apply this as a project-wide convention, sometimes via a custom meta-annotation `@BusinessTransaction`); or use unchecked exceptions for failure signalling, which is the modern norm anyway; or `noRollbackFor` for the inverse case where a specific runtime exception should still commit.

**The second, larger footgun: rollback-only.**

```java
@Transactional
public void outer() {
    try {
        inner();                                   // @Transactional, REQUIRED → same physical tx
    } catch (SomeException e) {
        log.warn("inner failed, continuing", e);   // handled!
    }
    repo.save(somethingElse);
}                                                  // → UnexpectedRollbackException
```

When `inner()` throws a rollback-triggering exception, Spring marks the **shared** transaction as `rollbackOnly` before the exception propagates. Catching it in `outer()` does not clear that flag. `outer()` completes normally, Spring attempts to commit, the transaction manager refuses because the transaction is marked rollback-only, and Spring throws `UnexpectedRollbackException: Transaction rolled back because it has been marked as rollback-only`.

This is confusing for three reasons: the exception surfaces at the *end* of the method, far from the cause; the original exception has been swallowed and may not be logged; and the code looks like it correctly handles a recoverable failure.

*Fixes:* make `inner()` `REQUIRES_NEW` so its rollback is genuinely independent (with the connection-pool caveat from Q27); or do not catch the exception, and handle it above the transaction boundary; or restructure so the optional work is not transactional at all; or use `NESTED` with a savepoint if your stack supports it. Hibernate adds a further constraint: once a `PersistenceException` has occurred, the persistence context is in an **undefined state** and must not be reused, so "catch and continue" with JPA is unsafe even if you clear the rollback flag.

### Probes

**Runtime + Errors only; checked exceptions commit.** Covered.

**`rollbackFor`.** `@Transactional(rollbackFor = Exception.class)` is the common blanket fix. `rollbackForClassName` accepts strings for cases where the class is not on the compile classpath. `noRollbackFor` handles the reverse: e.g. a validation `RuntimeException` after which you deliberately want the audit row to persist. Note that the rules are evaluated by *closest match in the class hierarchy*, so `rollbackFor = Exception.class` combined with `noRollbackFor = MyRuntimeException.class` behaves as you would hope.

Note also that `Error` always rolls back and cannot be excluded, and that the rollback rules apply to the exception that **propagates out of the annotated method** — an exception caught and handled inside never reaches the interceptor and has no effect (except via the rollback-only flag set by a nested transactional call, as above).

**Catching an exception inside the tx and swallowing it → "marked rollback-only" `UnexpectedRollbackException` at commit.** Covered in detail. Diagnostic tip: enable `logging.level.org.springframework.transaction=DEBUG` to see the transaction boundaries, the participation decisions, and the point at which rollback-only is set — it turns a mystifying error into an obvious one, and it is worth doing the first time anyone on the team hits this.

One more related trap: **`@Transactional` on a method that Spring cannot proxy** (private, self-invoked, called on `this`, or on a class not managed by Spring) results in *no transaction at all*, so exceptions neither commit nor roll back anything — each statement auto-commits. The symptom is partial writes surviving a failure, which looks exactly like the checked-exception footgun but has a different cause. When diagnosing "my rollback did not happen", verify first that a transaction existed at all: `TransactionSynchronizationManager.isActualTransactionActive()`.

---

## Q30. How does Spring resolve a circular dependency, and why is constructor injection "worse" here — but still the right default?

### Answer

**The three-level cache.** `DefaultSingletonBeanRegistry` maintains:

1. `singletonObjects` — fully initialised singletons.
2. `earlySingletonObjects` — instantiated but not yet fully initialised beans, exposed early.
3. `singletonFactories` — `ObjectFactory` instances that can produce an early reference (applying `getEarlyBeanReference` on `SmartInstantiationAwareBeanPostProcessor`s, which is where an early AOP proxy is created if needed).

For **setter/field injection**, resolution works like this: creating A instantiates A (constructor runs, no dependencies needed), registers a singleton factory for A in level 3, then populates A's fields — which triggers creation of B. B instantiates, then populates its `A` field: the container finds A's factory in level 3, obtains an early reference (possibly a proxy), promotes it to level 2, and injects it. B finishes initialising and is placed in level 1. Control returns to A, which finishes and moves to level 1. B holds a reference to A that was not fully initialised *at injection time* but is by the time anything calls it.

For **constructor injection**, this cannot work. To construct A you must already have B; to construct B you must already have A. There is no point at which a partially-constructed instance could be handed out, because the constructor has not returned. Spring detects the cycle (`beansCurrentlyInCreation`) and throws `BeanCurrentlyInCreationException`.

**So constructor injection is "worse" in that it fails where field injection succeeds. Why is it still right?**

1. **The failure is the correct behaviour.** A cycle means A cannot function without B and B cannot function without A — the design has no valid initialisation order. Field injection does not remove the problem; it hides it, and the hidden version has real hazards (see below).
2. **Immutability.** Constructor-injected dependencies can be `final`, so they cannot be reassigned and are safely published (Q6). Field-injected ones cannot be `final`.
3. **Explicit contract.** The constructor signature states the dependencies. A class with nine constructor parameters is visibly doing too much; the same class with nine `@Autowired` fields looks fine.
4. **Testability without a container.** `new OrderService(repo, client, clock)` in a plain JUnit test. Field injection forces either a Spring context or reflection (`ReflectionTestUtils`) — and the fact that it is *awkward* to test is a signal that the design is coupled to the framework.
5. **Fail-fast.** Missing dependencies fail at context startup rather than at first use.
6. **No Spring annotation needed.** Since Spring 4.3, a single constructor is autowired implicitly, so the class has no Spring-specific annotations at all.

**Spring Boot 2.6+ disallows circular references by default** (`spring.main.allow-circular-references=false`), so even field-injected cycles now fail at startup unless explicitly re-enabled. That flag exists as a migration aid, not a solution.

**How to actually fix a cycle:** extract the shared behaviour into a third component that both depend on; invert one direction using an event (`ApplicationEventPublisher`) or a callback interface; introduce an interface owned by the lower-level module and implemented by the higher one (dependency inversion); or merge the two classes if the cycle reveals they are one cohesive responsibility. `@Lazy` on one side works — it injects a proxy that resolves the real bean on first use — but it is a workaround that leaves the design defect in place and moves the failure from startup to runtime.

### Probes

**Three-level cache / early references for setter/field injection.** Covered. The reason three levels rather than two: level 3 holds a *factory* rather than an object so that the early reference can be an AOP proxy created on demand. If Spring exposed the raw object directly, a bean involved in a cycle would receive the unproxied target — meaning no transactions, no async, no security on calls made through that reference. The factory indirection lets `getEarlyBeanReference` apply the proxying post-processors at the moment the early reference is requested. Spring still validates at the end that the object it exposed early is the same one that ended up in the singleton cache, and fails with `BeanCurrentlyInCreationException` if not — which is what happens when a cycle combines with certain proxying scenarios.

**Constructor cycles fail fast.** Covered.

**Boot 2.6+ disallows cycles by default.** Covered. The flag is `spring.main.allow-circular-references=true`. Treat re-enabling it as technical debt with a ticket, not a configuration choice.

**`@Lazy` as an escape hatch.** `@Lazy` on the injection point causes Spring to inject a proxy that resolves the target on first method call, breaking the initialisation-time cycle. It works, and it has costs: an extra proxy layer on every call; the failure moves from startup to the first request, so a genuinely broken wiring is discovered in production rather than in CI; and it obscures the dependency structure. `ObjectProvider<T>` is a more honest variant — it makes the deferred lookup explicit in the code (`provider.getObject()`) rather than hiding it behind an annotation, and it also handles optional and multiple candidates cleanly.

**Why a cycle usually means a design problem.** A cycle means the two components cannot be understood, tested, deployed, or reasoned about independently — they are one unit with two names. It usually indicates one of: a missing abstraction (both need something that should be a third component); a layering violation (a lower layer calling back up into a higher one, which should be an event or a callback); or an over-decomposed service split along the wrong lines. The pragmatic test: ask what the *initialisation order* should be. If there is no sensible answer, the cycle is real and structural. Constructor injection asks that question at compile-and-start time; field injection lets you avoid answering it.

---

## Q31. `@Component` + `@Bean` + `@Configuration` — what does `proxyBeanMethods` do, and when would you set it to `false`?

### Answer

**`@Configuration(proxyBeanMethods = true)` — "full" mode, the default.** Spring creates a CGLIB subclass of the configuration class and intercepts every `@Bean` method. When one `@Bean` method calls another directly, the interceptor checks the singleton cache and returns the **existing singleton** instead of executing the method body again.

```java
@Configuration
public class AppConfig {
    @Bean public DataSource dataSource() { return new HikariDataSource(...); }
    @Bean public OrderRepository orderRepo() { return new OrderRepository(dataSource()); }
    @Bean public UserRepository userRepo()  { return new UserRepository(dataSource()); }
}
```
Both repositories receive the **same** `DataSource` singleton, even though `dataSource()` appears to be called twice. This is what most people assume Java semantics would never allow, and it is why `@Configuration` classes need CGLIB.

**`proxyBeanMethods = false` — "lite" mode.** No proxy. `@Bean` methods are plain Java methods. The inter-bean call above would create **two separate `HikariDataSource` instances** — two connection pools, one of them not managed by the container, not shut down on context close, invisible to metrics. A silent and serious bug.

Lite mode is also what you get automatically for `@Bean` methods declared inside `@Component`, `@Service`, or any class annotated with something other than `@Configuration`.

**When to set it to `false`.**

1. **When there are no inter-bean method calls** — which is the case whenever `@Bean` methods take their dependencies as **method parameters** rather than calling sibling methods:

```java
@Configuration(proxyBeanMethods = false)
public class AppConfig {
    @Bean DataSource dataSource() { ... }
    @Bean OrderRepository orderRepo(DataSource ds) { return new OrderRepository(ds); }
}
```
Parameter injection is resolved by the container, so the singleton guarantee comes from the container rather than from the proxy. This is the better style regardless: it is explicit, it works in both modes, and it makes the dependency visible in the signature.

2. **Startup performance and memory.** Every full-mode `@Configuration` class requires CGLIB subclass generation at startup — class generation, definition, and Metaspace. In an application with many configuration classes (and a Boot application has hundreds, mostly from auto-configuration), this is measurable. Spring Boot's own auto-configuration classes are almost all annotated `@Configuration(proxyBeanMethods = false)` for exactly this reason, and that change was one of the meaningful startup improvements in Boot 2.2.

3. **GraalVM native image / AOT.** CGLIB subclassing at runtime is incompatible with the closed-world model. Spring's AOT engine handles `@Configuration` classes, but lite mode is simpler and more native-friendly.

**Guidance:** default to `proxyBeanMethods = false` **and** pass dependencies as method parameters. Use full mode only when you genuinely need inter-bean method calls and do not want to restructure. The risk of lite mode is entirely in the inter-bean call case, and that case is avoidable.

### Probes

**Inter-bean method calls returning the singleton vs a new instance.** Covered above with the duplicate-`DataSource` example. Worth emphasising how bad the failure is: two connection pools means double the connections against the database (potentially exceeding `max_connections`), one pool with no health checks or metrics, and no clean shutdown. And because both objects work, nothing fails — you just have twice the resources and inconsistent behaviour.

**"Lite" mode.** The formal term for `@Bean` methods processed without CGLIB — either `proxyBeanMethods = false` on a `@Configuration`, or `@Bean` methods on a `@Component`/`@Service`/plain `@Import`ed class. Restrictions in lite mode: no inter-bean singleton semantics, and `@Bean` methods cannot be `private` or `final` in full mode (because they must be overridable) — a constraint lite mode removes. Lite-mode `@Bean` methods also cannot declare `@Bean` methods that participate in the same cross-referencing guarantees, obviously, and Spring will not warn you about it.

**Startup/native-image cost.** Covered. Concretely: full mode requires loading the CGLIB machinery, generating a subclass per configuration class, and running an interceptor on every `@Bean` method invocation during context refresh. For a large Boot application this is on the order of tens to low hundreds of milliseconds and several MB of Metaspace — not decisive on a long-running server, but significant for serverless, CLI tools, and scale-to-zero deployments where startup is the dominant cost.

Related detail worth knowing: `@Configuration` classes are also subject to the `final` restriction (CGLIB cannot subclass a final class), and a `@Configuration` class with a `private` `@Bean` method in full mode will fail because the method cannot be overridden. Both restrictions disappear in lite mode, which occasionally is the reason to choose it.

---

## Q32. How would you conditionally register beans based on environment, a property, and the presence of a class — and control the ordering of that resolution?

### Answer

**By environment — `@Profile`.**

```java
@Bean @Profile("!prod")  DataSeeder seeder() { ... }
@Bean @Profile({"prod","staging"}) AlertSender alerts() { ... }
```
Profiles are activated with `spring.profiles.active` (property, env var, or command line). `@Profile` supports `!`, `&`, `|` expressions. In Boot 2.4+, prefer **profile groups** (`spring.profiles.group.prod=prod,monitoring,tracing`) over the deprecated nested `spring.profiles` in YAML. A caution: profiles are convenient but easy to abuse — a codebase where behaviour differs across five profiles is one where nothing is tested as deployed. Prefer configuration properties for values, and reserve profiles for genuinely different *wiring* (e.g. an in-memory stub versus a real client).

**By property — `@ConditionalOnProperty`.**

```java
@Bean
@ConditionalOnProperty(prefix = "feature.export", name = "enabled",
                       havingValue = "true", matchIfMissing = false)
ExportService exportService() { ... }
```
`matchIfMissing` decides the default when the property is absent — get this wrong and a feature silently defaults the wrong way. Also `@ConditionalOnExpression("#{...}")` for SpEL when the logic is more complex, though anything needing SpEL here is usually better as a custom `Condition`.

**By class presence — `@ConditionalOnClass` / `@ConditionalOnMissingClass`.**

```java
@Configuration
@ConditionalOnClass(name = "io.lettuce.core.RedisClient")
class RedisConfig { ... }
```
Use the `name = "..."` string form when the class may genuinely be absent, so that the annotation itself does not fail to load. (Spring uses ASM to read annotation metadata without loading classes, which is what makes `@ConditionalOnClass` work at all — but referring to a `Class` literal in code that is itself loaded can still cause `NoClassDefFoundError` in edge cases.)

**Other built-ins:** `@ConditionalOnBean`, `@ConditionalOnMissingBean`, `@ConditionalOnSingleCandidate`, `@ConditionalOnResource`, `@ConditionalOnWebApplication(type = SERVLET|REACTIVE)`, `@ConditionalOnJava`, `@ConditionalOnCloudPlatform`, `@ConditionalOnJndi`.

**Custom `Condition`:**

```java
public class OnKubernetes implements Condition {
    @Override public boolean matches(ConditionContext ctx, AnnotatedTypeMetadata md) {
        return ctx.getEnvironment().containsProperty("KUBERNETES_SERVICE_HOST");
    }
}
@Bean @Conditional(OnKubernetes.class) PodInfo podInfo() { ... }
```
`ConditionContext` gives access to the `Environment`, the `BeanFactory` (use with care — bean definitions may not all be registered yet), the `ResourceLoader`, and the `ClassLoader`. For auto-configuration, extend `SpringBootCondition` so failures are reported in the condition evaluation report.

**Ordering.** Two distinct concerns, often conflated:

- **Bean instantiation order** — normally determined by the dependency graph and nothing else. `@Order`/`Ordered` affects the *order within an injected collection or list* (`List<Validator>`), and the order of `BeanPostProcessor`s, `Filter`s, and `ApplicationListener`s — **not** general instantiation order. `@DependsOn("beanName")` forces instantiation order when there is a hidden dependency (e.g. a bean that must register a JDBC driver before another bean uses it). Needing `@DependsOn` usually indicates an implicit dependency that should be made explicit.
- **Auto-configuration order** — `@AutoConfigureBefore`, `@AutoConfigureAfter`, `@AutoConfigureOrder`. These control the order in which auto-configuration *classes* are processed, which is what makes `@ConditionalOnMissingBean` deterministic within auto-configuration.

### Probes

**`@ConditionalOnProperty/Class/MissingBean`, `@Profile`, custom `Condition`.** Covered above.

**`@Order`/`@AutoConfigureBefore`.** Covered. Note `@Order` on a `@Bean` method or component determines position in injected `List<T>`/`Map<String,T>` and stream ordering — this is how you build an ordered chain of handlers or filters declaratively. For `Filter` registration in Boot, `FilterRegistrationBean.setOrder` is the reliable mechanism, and Spring Security's filter chain has its own ordering constants.

**Why `@ConditionalOnMissingBean` in *user* config is a trap (ordering is undefined; it's for auto-config).**

This is the important part of the question. `@ConditionalOnMissingBean` asks "has a bean of this type been registered *at the time this condition is evaluated*?" That makes it order-dependent. The ordering guarantee exists **only** for auto-configuration, because Spring Boot processes all user configuration first, then auto-configuration (in an order controlled by `@AutoConfigureBefore`/`After`). That is precisely why auto-configuration can say "back off if the user defined their own" and be correct.

Between two *user* `@Configuration` classes there is no defined processing order — it depends on classpath scanning order, class names, and implementation details that can change between Spring versions or between operating systems. So:

```java
@Configuration
class ConfigA { @Bean @ConditionalOnMissingBean ObjectMapper mapper() { ... } }
@Configuration
class ConfigB { @Bean @ConditionalOnMissingBean ObjectMapper mapper() { ... } }
```
Which one wins is undefined, and may differ between your machine and CI. It may even work consistently for a year and then change after a dependency upgrade. Spring's own documentation states that these conditions are only intended for auto-configuration.

The correct tools in user configuration are: `@Primary` (pick a default among candidates), `@Qualifier` (select explicitly at the injection point), `@Profile` or `@ConditionalOnProperty` (make the choice explicit and deterministic), or simply not defining two beans of the same type. And if you are writing a **library** meant to be overridable by consumers, package it as a proper auto-configuration (an entry in `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`) so the ordering guarantee actually applies.

One further related trap: `@ConditionalOnBean` has the same ordering sensitivity in the other direction, and is even more fragile — it is straightforwardly recommended to use it only in auto-configuration, and only with `@AutoConfigureAfter` naming the configuration that would define the bean.

---

## Q33. Explain the difference between the ApplicationContext event mechanism and a message broker. When is `@EventListener` the wrong tool?

### Answer

**What `ApplicationEventPublisher` is.** An **in-process, in-memory observer pattern** managed by the Spring context. `publishEvent` looks up matching `ApplicationListener`s / `@EventListener` methods and, by default, invokes them **synchronously on the publishing thread**, inside the same call stack and the same transaction.

**Differences from a message broker (Kafka, RabbitMQ, SQS):**

| | Spring events | Message broker |
|---|---|---|
| Scope | One JVM | Across processes, services, languages |
| Delivery | In-memory method call | Network, with persistence |
| Durability | None — lost on crash or if no listener exists | Persisted; survives consumer downtime |
| Delivery guarantee | At-most-once, and only to listeners present now | At-least-once, with retry and DLQ |
| Threading | Publisher's thread (unless `@Async`) | Consumer's own threads |
| Backpressure | None | Queue depth, consumer lag, flow control |
| Ordering | Listener registration order | Per-partition/per-queue ordering |
| Failure | Exception propagates to publisher (sync) | Redelivery, retry topics, DLQ |
| Observability | Effectively none built in | Lag, depth, throughput metrics |

**The one-sentence framing:** Spring events are a **decoupling mechanism within a process**; a broker is an **integration and durability mechanism between processes**. Using Spring events for cross-cutting concerns inside a module is good design; using them where you need durability, retry, or cross-service delivery is a category error.

**Legitimate uses of `@EventListener`:** decoupling a domain operation from its side effects within one deployable (order saved → invalidate a cache, update a search index, emit a metric); reacting to framework lifecycle events (`ApplicationReadyEvent`, `ContextClosedEvent`); implementing a lightweight in-process publish/subscribe where losing an event on crash is acceptable; and — importantly — `@TransactionalEventListener` for "do this only if the transaction commits".

**When it is the wrong tool:**

1. **When the effect must not be lost.** If the process dies between the publish and the listener completing, the event is gone with no trace. Anything another system or another user depends on needs durability — outbox plus broker.
2. **When the listener does slow or blocking work synchronously.** It runs on the publisher's thread, inside the publisher's transaction, extending both.
3. **When you need retry, backoff, or a dead-letter path.** None of that exists.
4. **When the consumer is in another process.** Obviously.
5. **When you are using it to hide a call graph.** A codebase where business flow is threaded through a dozen events becomes very hard to follow: "what happens when an order is placed?" has no answer you can find by reading code, because the listeners are discovered at runtime. Events decouple, but they also make control flow invisible. Use them where the coupling genuinely should be broken, not as a default.
6. **When you need ordering or exactly-once.** Listener ordering is controlled only by `@Order`, and there is no delivery guarantee at all.

### Probes

**Synchronous by default.** Covered. The practical consequence: an exception thrown by a listener propagates back to `publishEvent()` and therefore to the publishing business method — which will roll back the publisher's transaction. Many developers assume events are fire-and-forget and are surprised when a failing cache-invalidation listener rolls back an order. If listener failure should not affect the publisher, the listener must catch its own exceptions (and report them somewhere) or be `@Async`.

**In-process only.** Covered.

**Lost on crash.** Covered.

**`@TransactionalEventListener` phases.** `AFTER_COMMIT` (default), `BEFORE_COMMIT`, `AFTER_ROLLBACK`, `AFTER_COMPLETION`. `AFTER_COMMIT` is the valuable one: it lets you publish domain events inside the transactional method (so the code reads naturally) while ensuring side effects only occur if the data actually persisted. This solves the "send email then roll back" problem elegantly.

Two traps: (a) by default, if there is **no transaction active** when the event is published, the listener does **not run at all** — set `fallbackExecution = true` to change that, and be aware this bites in tests where you forgot `@Transactional`; (b) an `AFTER_COMMIT` listener runs **after** the transaction has committed and the connection has been released, so it has **no transaction** — database writes inside it run in auto-commit unless you add `@Transactional(propagation = REQUIRES_NEW)`. Both of these are extremely common bugs and both are silent.

**`@Async` events losing exceptions/transaction context.** Adding `@Async` to a listener moves it to a `TaskExecutor` thread. Consequences:

- **The exception goes nowhere.** For a `void` `@Async` method, an uncaught exception is handled by the configured `AsyncUncaughtExceptionHandler`, which by default only logs. If you have not configured one via `AsyncConfigurer`, failures are effectively invisible. Always configure one.
- **Transaction context is not propagated.** The listener runs with no transaction; `@Transactional` on the listener starts a fresh, independent one.
- **`ThreadLocal` context is lost** — MDC correlation IDs, `SecurityContextHolder`, tenant context (Q18/Q21). Configure a `TaskDecorator` to propagate and clear it.
- **The default executor matters.** If you have not defined one, Boot's `applicationTaskExecutor` (a `ThreadPoolTaskExecutor` with a bounded queue in recent versions) is used; older behaviour with `SimpleAsyncTaskExecutor` created a **new thread per invocation**, which is catastrophic under load. Always define your executor explicitly and give it a bounded queue and a rejection policy.
- **`@Async` + `@TransactionalEventListener(AFTER_COMMIT)`** is a common combination and is reasonable — the transaction has committed, so there is nothing to propagate — but it converts the operation to at-most-once with no retry, so it is only appropriate for effects you can afford to lose.

---

# 4. Spring Boot

---

## Q34. Explain how auto-configuration actually works end to end, and how you'd debug "my bean isn't being created".

### Answer

**The chain.**

1. `@SpringBootApplication` is a meta-annotation combining `@SpringBootConfiguration`, `@ComponentScan`, and `@EnableAutoConfiguration`.
2. `@EnableAutoConfiguration` imports `AutoConfigurationImportSelector` (a `DeferredImportSelector`, which is why auto-configuration is processed **after** all user configuration).
3. The selector reads the list of candidate auto-configuration classes from every jar on the classpath. **Since Spring Boot 2.7** the file is `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` — a plain newline-separated list of class names. Before 2.7 it was the `EnableAutoConfiguration` key in `META-INF/spring.factories`. Boot 3 removed support for the old location. Getting this right is a good signal of currency.
4. Candidates are filtered: exclusions (`@SpringBootApplication(exclude=...)`, `spring.autoconfigure.exclude`), then fast ASM-based filters using `META-INF/spring-autoconfigure-metadata.properties` (which pre-records simple `@ConditionalOnClass` requirements so Boot can discard most candidates without loading them — a significant startup optimisation), then full `@Conditional` evaluation.
5. Surviving classes are sorted by `@AutoConfigureOrder`, `@AutoConfigureBefore`, `@AutoConfigureAfter` and registered as configuration classes.
6. Their `@Bean` methods are evaluated, each subject to its own conditions — most importantly `@ConditionalOnMissingBean`, which is why defining your own bean makes Boot back off.

**The ordering guarantee that makes it work:** user configuration is processed first, so by the time auto-configuration runs, `@ConditionalOnMissingBean` sees your beans and steps aside. This is the entire mechanism behind "convention with easy override".

**Debugging "my bean isn't being created."**

**Step 1 — the condition evaluation report.** Run with `--debug` (or `debug=true` in properties, or `logging.level.org.springframework.boot.autoconfigure=DEBUG`). Boot prints:
- **Positive matches** — conditions that matched and why.
- **Negative matches** — conditions that did *not* match **and the specific reason** ("did not find required class X", "found bean of type Y").
- **Exclusions** and **unconditional classes**.

For an auto-configuration problem this almost always gives the answer in one run. Search the negative matches for the relevant class name.

At runtime, the `/actuator/conditions` endpoint exposes the same report as JSON, which is useful when you cannot easily restart with `--debug`.

**Step 2 — is it even a candidate?** If the class does not appear in *either* the positive or negative match lists, it was never considered. Causes: the jar is not on the classpath; the `.imports` file is missing or malformed (a very common bug in hand-rolled starters, especially after a 2.7 migration); or it was excluded.

**Step 3 — is it your own bean rather than an auto-configured one?** Then the causes are different:
- **Outside the component scan.** `@SpringBootApplication` scans the package of the main class and below. A bean in a sibling package is invisible. This is the single most common cause.
- **Missing stereotype annotation** (`@Component`/`@Service`/`@Repository`/`@Configuration`).
- **A `@Conditional` on your own bean that does not match.**
- **A profile that is not active.**
- **`@ComponentScan` explicitly configured and narrowed**, overriding the default.

**Step 4 — inspect the actual context.** `/actuator/beans` lists every bean with its type, scope, and dependencies. `context.getBeanDefinitionNames()` in a breakpoint or a startup runner does the same. If the bean is there but the wrong instance is injected, look for `@Primary`, `@Qualifier`, and multiple candidates.

**Step 5 — startup ordering / early instantiation.** If your bean exists but is not proxied (no `@Transactional` effect), check for the "not eligible for getting processed by all BeanPostProcessors" warning (Q24) — a bean pulled in too early by a `BeanPostProcessor` or a `@Configuration` dependency misses post-processing.

### Probes

**`@EnableAutoConfiguration` → `AutoConfiguration.imports` (post-2.7) vs `spring.factories`.** Covered. If you maintain an internal starter, the migration matters: Boot 3 will simply ignore the old file and your starter will silently do nothing.

**`@Conditional` evaluation.** Covered. Add that `@ConditionalOnMissingBean` without arguments defaults to the return type of the `@Bean` method, which is usually what you want but occasionally not — if the method's declared return type is an interface and the user registers a different implementation of that interface, Boot backs off, which is correct but can surprise.

**`--debug` / `ConditionEvaluationReport`.** Covered. It is worth emphasising in an interview that this is the *first* thing to reach for, not the last. Many engineers debug auto-configuration by trial and error for hours without knowing the report exists.

**`spring.autoconfigure.exclude`.** Excludes by class name, as a property (so it can be set per environment) rather than in code. The `exclude` attribute on `@SpringBootApplication` does the same at compile time. A common legitimate use: excluding `DataSourceAutoConfiguration` in a service that has a JDBC driver on the classpath (pulled in transitively) but does not use a database — otherwise startup fails with "Failed to configure a DataSource".

**Ordering annotations.** `@AutoConfigureAfter(DataSourceAutoConfiguration.class)` on a configuration that needs the `DataSource` bean to exist before its `@ConditionalOnBean` is evaluated. `@AutoConfigureBefore` for the inverse. `@AutoConfigureOrder` for coarse ordering. These apply **only between auto-configuration classes** — they have no effect on user configuration, and no effect on bean instantiation order (Q32).

---

## Q35. You need to override one bean from a starter's auto-configuration but keep the rest. How, and what could go wrong?

### Answer

**The intended mechanism: define your own bean and let `@ConditionalOnMissingBean` back off.**

```java
@Configuration
public class MyJacksonConfig {
    @Bean
    public ObjectMapper objectMapper() { ... }   // auto-config backs off
}
```
This works because user configuration is processed before auto-configuration (Q34).

**But it is often the wrong approach**, because auto-configuration beans are usually assembled from many settings, and by replacing the whole bean you discard all of them. Replacing Boot's `ObjectMapper` discards every `spring.jackson.*` property, the registered modules (JavaTimeModule, Kotlin, JDK8), and the serialisation defaults Boot configured — usually including `WRITE_DATES_AS_TIMESTAMPS=false`, whose loss silently changes every date in your API from ISO-8601 strings to epoch numbers. That is a breaking API change introduced by what looked like a local configuration tweak.

**The better mechanism: customiser interfaces.** Most starters expose a callback seam specifically so you can adjust without replacing:

- `Jackson2ObjectMapperBuilderCustomizer` — adjust Jackson while keeping Boot's setup.
- `WebMvcConfigurer` — add converters, interceptors, CORS, argument resolvers (note: implementing `WebMvcConfigurer` is additive; annotating with `@EnableWebMvc` *disables* Boot's MVC auto-configuration entirely, which is a classic accidental foot-gun).
- `RestClientCustomizer` / `RestTemplateCustomizer` / `WebClientCustomizer`.
- `HibernatePropertiesCustomizer`, `TomcatConnectorCustomizer`, `KafkaListenerContainerFactory` post-processing, `SecurityFilterChain` beans, `FlywayConfigurationCustomizer`, and so on.

These run *after* Boot's own configuration and mutate the builder, so you keep the defaults and change only what you intend.

**Third option: configuration properties.** Before writing any Java, check whether a `spring.*` property already does it. The properties surface is very large and most "override the bean" instincts are unnecessary.

**Fourth option: `BeanPostProcessor`** to adjust the auto-configured bean after creation. Powerful but blunt; runs for every bean, requires type checks, and interacts badly with early instantiation (Q24). Use as a last resort.

**What could go wrong.**

1. **Losing the rest of the configuration**, as above. The most common and most damaging.
2. **Bean overriding is disabled by default.** Since Boot 2.1, `spring.main.allow-bean-definition-overriding=false` is the default, so *two definitions of the same bean name* fail at startup with `BeanDefinitionOverrideException`. Note this is about **name** collision, not type — and it does not apply to the `@ConditionalOnMissingBean` case, which is a back-off rather than an override. Enabling the flag to "fix" a collision hides a real ambiguity.
3. **Ordering assumptions.** `@ConditionalOnMissingBean` in *your* configuration referring to an auto-configured bean is unreliable (Q32) — auto-configuration has not run yet, so the bean does not exist yet, so your condition always matches.
4. **Breaking other auto-configuration that depends on the replaced bean.** Auto-configurations frequently `@ConditionalOnBean` or `@ConditionalOnSingleCandidate` on each other's outputs. Replacing one can cause an unrelated one to back off. The condition report tells you.
5. **`@Primary` as a workaround.** Adding `@Primary` to your bean resolves ambiguity at injection points but leaves both beans in the context. For a `DataSource` that means two connection pools, only one of which is used — wasted connections and confusing metrics.
6. **Type mismatch with `@ConditionalOnMissingBean`.** If the auto-configuration's condition is on interface `Foo` and you register a `Foo` under a different name, it backs off correctly; but if it conditions on the concrete type and you register the interface, it may not back off and you get two beans.

### Probes

**Define your own bean → `@ConditionalOnMissingBean` backs off; but only if user config is processed first.** Covered — and it *is* processed first, by design, via `DeferredImportSelector`. The unreliability is in the reverse direction (your config conditioning on auto-config beans).

**`@Primary`.** Marks a bean as the default candidate when several match a type. Useful for a multi-datasource setup where one is the primary. Downsides as above. `@Qualifier` at the injection point is more explicit and does not leave a silent second candidate. Spring 6.2 added `@Fallback` as the inverse (a bean to use only if no other candidate exists), which is a cleaner fit for some library scenarios.

**Bean overriding disabled by default in newer Boot.** Covered. The error message names both definitions and their sources, which usually identifies the duplicate immediately. The most common real cause is two component-scanned classes producing the same bean name (`@Bean` method name or class simple name), often from a copy-pasted configuration class in a different package.

**Customizer interfaces (`WebMvcConfigurer`, `Jackson2ObjectMapperBuilderCustomizer`) as the *intended* seam.** Covered — this is the point of the question. The general principle to state: **a framework that offers a customiser is telling you that replacing the bean is not the supported extension path.** Look for the customiser before replacing anything. And when writing your own starter, provide customiser interfaces for the same reason — it lets consumers adjust without forking your configuration.

---

## Q36. What is the full precedence order of configuration properties, and where do you put secrets?

### Answer

**Precedence, highest wins** (Spring Boot's documented order; abbreviated to the entries that matter in practice, highest first):

1. Devtools global settings (`~/.config/spring-boot`), when devtools is active.
2. `@TestPropertySource` and `properties` on `@SpringBootTest` (tests only).
3. Command-line arguments (`--server.port=8081`).
4. `SPRING_APPLICATION_JSON` (inline JSON in an environment variable or system property).
5. `ServletConfig` / `ServletContext` init parameters.
6. JNDI attributes.
7. **Java System properties** (`-Dserver.port=8081`).
8. **OS environment variables** (`SERVER_PORT=8081`).
9. A `RandomValuePropertySource` (`random.*`).
10. **Profile-specific application properties outside the jar** (`application-{profile}.yml` next to the jar).
11. **Profile-specific application properties inside the jar**.
12. **Application properties outside the jar**.
13. **Application properties inside the jar**.
14. `@PropertySource` on `@Configuration` classes.
15. Default properties (`SpringApplication.setDefaultProperties`).

The two rules to remember for day-to-day work: **more specific beats less specific** (profile-specific beats plain; external beats packaged), and **environment variables beat everything packaged** — which is what makes the same container image deployable to every environment.

**Relaxed binding.** Boot maps a canonical property name to many forms: `spring.jpa.show-sql`, `spring.jpa.showSql`, `spring.jpa.show_sql`, and the environment-variable form `SPRING_JPA_SHOWSQL`/`SPRING_JPA_SHOW_SQL`. The environment-variable convention is: uppercase, dots and hyphens become underscores. For a property inside a list, use `_0_` (e.g. `MY_LIST_0_NAME`). Relaxed binding applies to `@ConfigurationProperties`; it does **not** apply to `@Value("${...}")`, which requires the exact name — a real and frequently-hit inconsistency.

**`spring.config.import`** (Boot 2.4+) lets a configuration file pull in others declaratively: `spring.config.import=optional:configserver:`, `optional:file:./config/`, `configtree:/run/secrets/`. The `configtree:` form is specifically designed for Kubernetes Secrets mounted as a directory of files, where each filename is a property name and the file content is the value — this is the cleanest way to consume mounted secrets.

**Where secrets go.**

The invariant: **secrets must not be in the image, in the repository, or in the build artefact.** Everything else is a trade-off.

- **A managed secret store** — HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager, Azure Key Vault. Best option: centralised access control, audit logs, rotation, versioning. Integrated via Spring Cloud Vault / Spring Cloud AWS, or via an external agent that injects them.
- **Kubernetes Secrets mounted as files**, consumed with `spring.config.import=configtree:/run/secrets/`. Better than environment variables because: files can be updated in place without a pod restart (with some caveats); they do not appear in `/proc/<pid>/environ`; they are not inherited by child processes; and they do not leak into crash dumps, `docker inspect`, or log lines that dump the environment. Note that Kubernetes Secrets are only base64-*encoded* at rest in etcd unless encryption-at-rest is configured — enable it.
- **Environment variables** — widely used and acceptable, but leak more readily (any library that logs the environment on startup, any error reporter that captures it, `ps e` for a privileged user, child processes).
- **Never** in `application.yml` in the repository, in a Dockerfile `ENV` or `ARG` (build args are visible in image history), or in a config map.

Additionally: use short-lived credentials where the platform supports it (IAM roles for service accounts, workload identity, database IAM auth) so there is no static secret to leak; and mask secrets in logs and in Actuator (`/actuator/env` and `/actuator/configprops` sanitise keys matching password/secret/key/token patterns by default — verify that your custom key names match the sanitisation patterns, and configure `management.endpoint.env.show-values`).

### Probes

**Command line > env vars > profile-specific > application.yml > defaults.** Covered with the full list.

**Relaxed binding.** Covered, including the `@Value` exception, which is the practical trap: `@Value("${my.property-name}")` will not resolve from `MY_PROPERTYNAME` reliably, whereas a `@ConfigurationProperties` binding will.

**`@ConfigurationProperties` vs `@Value` (validation, type-safety, refresh).**

`@ConfigurationProperties` advantages: binds a whole prefix into a typed object; supports nested objects, `List`, `Map`, `Duration`, `DataSize`, and custom converters; supports JSR-380 validation with `@Validated` so misconfiguration **fails at startup** with a clear message rather than at first use; supports **constructor binding** (immutable, `final` fields, works with records); appears in IDE autocompletion when you add `spring-boot-configuration-processor`, which generates metadata; and can be documented with Javadoc that surfaces in the IDE.

`@Value` advantages: convenient for a single value; supports SpEL. Disadvantages: no validation, no relaxed binding, no metadata, typos fail at runtime (or inject the literal placeholder string if a default is supplied), and it scatters configuration knowledge across the codebase.

Guidance: use `@ConfigurationProperties` with constructor binding and `@Validated` for anything non-trivial. A record plus `@ConfigurationProperties` is the modern idiom:

```java
@ConfigurationProperties("app.export")
@Validated
public record ExportProperties(@NotBlank String bucket,
                               @DefaultValue("30s") Duration timeout,
                               @Min(1) int batchSize) {}
```

On refresh: neither is refreshable by default. `@RefreshScope` (Spring Cloud) recreates beans on a refresh event, which works for `@ConfigurationProperties` beans; `@Value` in a non-refresh-scoped bean is bound once and never updates.

**K8s Secrets mounted as files vs env.** Covered above.

**Vault.** Covered. Spring Cloud Vault can fetch secrets at startup as a property source, and supports dynamic database credentials (Vault generates a short-lived DB user per instance and revokes it), which eliminates static database passwords entirely. Operationally more complex; worth it at scale or under compliance requirements.

**Why `spring.config.import` matters.** It replaced the confusing pre-2.4 `spring.config.location`/`spring.profiles` mechanisms with a single, ordered, composable directive that works uniformly for files, directories, config trees, Consul, and Config Server, and supports `optional:` prefixes so a missing source does not fail startup. It also fixed the long-standing confusion around profile-specific documents in multi-document YAML, which the 2.4 release restructured (`spring.config.activate.on-profile` replaced nested `spring.profiles`). If migrating from an older Boot version, this is one of the areas most likely to break silently.

---

## Q37. Describe your testing strategy for a Boot service. When do you use `@SpringBootTest` vs a slice vs plain JUnit?

### Answer

**The layered strategy.**

**Plain JUnit, no Spring — the majority of tests.** Domain logic, calculations, mappers, validators, state machines, anything with behaviour independent of infrastructure. Instantiate the class with constructor injection and test it. Milliseconds per test, no context, no flakiness. If a service class cannot be tested this way, that is a design signal — usually too much infrastructure coupling. **This should be the bulk of your suite**, and constructor injection is what makes it possible (Q30).

**Slice tests — targeted integration.** Boot's slices start a *partial* context with only the relevant auto-configuration:

- `@WebMvcTest(OrderController.class)` — MVC layer only: controllers, `@ControllerAdvice`, converters, filters, Spring Security config. No repositories, no services (mock them with `@MockitoBean`). Tests routing, request/response binding, validation, status codes, serialisation, and security rules — things you genuinely cannot verify by calling the controller method directly.
- `@DataJpaTest` — JPA layer: entities, repositories, `TestEntityManager`. Transactional and rolled back per test by default. Tests mappings, custom queries, constraints.
- `@JsonTest`, `@RestClientTest`, `@DataRedisTest`, `@JdbcTest`, `@WebFluxTest`, `@KafkaTest` (via `@EmbeddedKafka`).

Slices are faster than a full context and, more importantly, they *scope the failure*: a failing `@WebMvcTest` implicates the web layer.

**`@SpringBootTest` — full context.** Use sparingly, for genuine end-to-end wiring verification: the application starts, the beans wire together, a request flows through every layer to a real database. `webEnvironment = RANDOM_PORT` with `TestRestTemplate`/`WebTestClient` for real HTTP. A handful of these — the critical happy paths and a few key failure paths — not one per feature.

**The shape:** many plain unit tests, a moderate number of slices, a small number of full-context tests. Not because of dogma about pyramids, but because of feedback speed and failure localisation.

**Testcontainers throughout the integration layers** (see probes).

### Probes

**Test pyramid.** As above. The practical argument is about **feedback latency and diagnosis**, not test count ratios: a unit test that fails tells you exactly which method is wrong; a full end-to-end test that fails tells you something in the system is wrong. Both are useful; the first is cheaper.

**Context caching (and what makes a *new* context — dirty context, different `@MockBean` set, properties).**

This is the single biggest determinant of suite runtime, and it is poorly understood. Spring's `TestContext` framework caches `ApplicationContext`s keyed by their full configuration. A cache hit reuses the context across test classes — near-instant. A cache miss starts a new context — seconds each, and the cached ones stay in memory.

The cache key includes: the configuration classes/locations, active profiles, property sources and inline properties, `ContextCustomizer`s (which is where **the set of `@MockitoBean`/`@MockBean` types** lives), the `ContextLoader`, the parent context, and the web environment.

Therefore each of these creates a **new** context:
- A different combination of mocked beans — even the same types in a different set. Ten test classes each mocking a slightly different set produce ten contexts.
- Different `@TestPropertySource` values or `@SpringBootTest(properties=...)`.
- Different `@ActiveProfiles`.
- Different slice annotations.
- **`@DirtiesContext`** — explicitly evicts the context, forcing a rebuild for the next test that needs it. Occasionally necessary; frequently used as a workaround for tests that mutate shared state, and it is expensive. Fixing the state leakage is almost always better.

Practical measures: standardise on a small number of test configurations; define a shared `@SpringBootTest` base class or a composed annotation so many test classes share one cache key; avoid per-class inline properties; and set `logging.level.org.springframework.test.context.cache=DEBUG` to log cache statistics — the hit/miss ratio and context count usually reveal an easy win. The default cache size is 32 contexts (`spring.test.context.cache.maxSize`), evicted LRU, so exceeding it causes rebuilds even for configurations you will need again.

**`@WebMvcTest`/`@DataJpaTest`.** Covered. Note `@DataJpaTest` defaults to replacing your `DataSource` with an embedded one — override with `@AutoConfigureTestDatabase(replace = NONE)` when using Testcontainers. And `@WebMvcTest` includes Spring Security's configuration by default, so tests will 401 unless you supply `@WithMockUser` or disable it — which is a feature, since it means your security rules are actually tested.

**Testcontainers over H2 and why.** H2 (or HSQLDB) in `MODE=PostgreSQL` is not PostgreSQL. Differences that cause tests to pass while production fails, and vice versa:

- SQL dialect gaps: window function support, `ON CONFLICT`, `RETURNING`, `LATERAL`, `DISTINCT ON`, array and JSONB types and operators, full-text search, regex operators.
- Different type systems: `uuid`, `jsonb`, `interval`, `enum`, `citext` — either absent or behaving differently.
- Different constraint, index, and sequence behaviour; different identity generation.
- Different transaction isolation implementation (H2 has no MVCC equivalent to PostgreSQL's), so concurrency and locking tests are meaningless.
- Different error codes and messages, so exception-translation tests are wrong.
- Migrations that work on H2 but fail on PostgreSQL (or must be forked per database, which means your migrations are not tested).

Testcontainers runs the **real** database in a container for the test. Costs: Docker required in CI, slower startup. Mitigations: `.withReuse(true)` plus `testcontainers.reuse.enable=true` keeps the container alive between runs locally; a `@Container static` field shares one container across all tests in a class; and Spring Boot 3.1's `@ServiceConnection` wires the container's connection details into the context automatically, removing the `@DynamicPropertySource` boilerplate:

```java
@Testcontainers
@SpringBootTest
class OrderIT {
    @Container @ServiceConnection
    static PostgreSQLContainer<?> db = new PostgreSQLContainer<>("postgres:16");
}
```
Boot 3.1 also added **Testcontainers at development time** (`spring-boot-testcontainers` plus a `TestApplication` main class), so `./gradlew bootTestRun` starts your dependencies automatically. This is now the default recommended approach and knowing it signals currency.

The same argument applies to embedded Kafka versus a Kafka container, and to embedded Redis versus a Redis container.

**`@MockBean` deprecation → `@MockitoBean`.** `@MockBean` and `@SpyBean` were deprecated in **Spring Boot 3.4** in favour of `@MockitoBean` and `@MockitoSpyBean` in `org.springframework.test.context.bean.override.mockito` — part of a general bean-override mechanism in Spring Framework 6.2 that also includes `@TestBean` (replace a bean with one produced by a static factory method, no Mockito involved). If your project is on 3.4+, use the new annotations; on earlier versions `@MockBean` remains correct. Either way, remember that each distinct set of overridden beans is a separate cached context.

**Why H2 hides real SQL bugs.** Covered above.

---

## Q38. Actuator is exposed in production. Configure it responsibly and explain what liveness vs readiness mean here.

### Answer

**Baseline configuration.**

```yaml
management:
  server:
    port: 8081                      # separate port, not exposed via the public ingress
  endpoints:
    web:
      exposure:
        include: health,info,prometheus,metrics
        # NOT "*" — that exposes env, configprops, heapdump, threaddump, loggers, mappings
      base-path: /internal
  endpoint:
    health:
      probes:
        enabled: true               # adds /health/liveness and /health/readiness
      show-details: when-authorized # or 'never'; details can leak infrastructure info
      show-components: when-authorized
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

**The controls that matter:**

1. **Separate management port.** Bind Actuator to a different port and expose that port only inside the cluster. The ingress/load balancer routes only the application port. This is the strongest single control because it removes the endpoints from the public attack surface entirely, regardless of any path or authentication mistake.
2. **Explicit exposure allowlist.** `include: "*"` is a serious misconfiguration. `/actuator/env` and `/actuator/configprops` reveal configuration (sanitised for obvious secret-like keys, but only for keys matching the sanitisation patterns — a key named `apiCredential` may not be masked); `/actuator/heapdump` downloads a full heap dump containing every in-memory secret, session token, and customer record; `/actuator/threaddump` reveals internal structure; `/actuator/loggers` allows changing log levels at runtime (also a DoS vector — set everything to TRACE); `/actuator/shutdown` stops the application (disabled by default, and should stay that way). Exposed Actuator endpoints are a well-known and actively scanned vulnerability class.
3. **Authentication and authorisation** even on the internal port, via a `SecurityFilterChain` matching `EndpointRequest.toAnyEndpoint()` and requiring a role. Defence in depth: the internal network is not a trust boundary.
4. **`show-details`**: `never` or `when-authorized`. Health details name your database host, disk paths, broker addresses, and downstream services.

**Liveness vs readiness.**

- **Liveness** answers: *is this process broken beyond recovery?* A failing liveness probe causes Kubernetes to **restart the container**. It should therefore check only conditions that a restart would fix: the JVM is running, the application context started, no unrecoverable internal state (a deadlocked critical thread, an exhausted-and-stuck state). Spring's `LivenessState` is `CORRECT` or `BROKEN`; you signal `BROKEN` by publishing an `AvailabilityChangeEvent`.
- **Readiness** answers: *can this instance serve traffic right now?* A failing readiness probe causes Kubernetes to **remove the pod from the Service endpoints** — no restart, no traffic. It should check dependencies required to serve requests: the database is reachable, the connection pool is not exhausted, required caches are warm, the Kafka consumer has joined its group. Spring's `ReadinessState` is `ACCEPTING_TRAFFIC` or `REFUSING_TRAFFIC`, and Boot automatically sets it to `REFUSING_TRAFFIC` at the start of graceful shutdown.

Boot's `/health/liveness` and `/health/readiness` groups map exactly onto this, and by default the readiness group includes the standard health indicators (including `db`) while the liveness group does not.

**The critical rule: never put an external dependency check in the liveness probe.** If the database goes down, every pod's liveness fails, Kubernetes restarts every pod, restarts do not fix the database, the restarts are counted as crashes, `CrashLoopBackOff` kicks in with exponential backoff, and when the database recovers your entire fleet is in backoff and takes minutes to return. You have converted a recoverable dependency outage into a self-inflicted total outage. Readiness failing is the correct response: pods stay alive and stop receiving traffic until the dependency returns.

### Probes

**Separate management port.** Covered. Note that with a separate port, Spring Security's configuration for that port is separate too, and health endpoints are typically permitted unauthenticated on it so probes work without credentials — while other endpoints require auth. `EndpointRequest.to(HealthEndpoint.class)` versus `EndpointRequest.toAnyEndpoint()` lets you express that.

**Exposure allowlist.** Covered with the specific risks per endpoint. Add: `/actuator/prometheus` exposes your metric names and label values, which can leak tenant identifiers or URL structure if you have high-cardinality labels — another reason to keep the management port internal.

**Securing endpoints.** Covered.

**`/health/liveness` vs `/readiness` and their K8s mapping.** Covered.

```yaml
livenessProbe:
  httpGet: { path: /internal/health/liveness, port: 8081 }
  failureThreshold: 3
  periodSeconds: 10
readinessProbe:
  httpGet: { path: /internal/health/readiness, port: 8081 }
  failureThreshold: 2
  periodSeconds: 5
startupProbe:
  httpGet: { path: /internal/health/liveness, port: 8081 }
  failureThreshold: 30
  periodSeconds: 5     # allows up to 150s for startup before liveness applies
```

**Custom `HealthIndicator`.** Implement `HealthIndicator` (or `ReactiveHealthIndicator`) and the bean is auto-registered under its bean-name prefix. Guidance:

- Assign each indicator to the right **group**: `management.endpoint.health.group.readiness.include=readinessState,db,redis` and keep `liveness` to `livenessState` only.
- **Timeouts.** A health check that hangs makes the probe hang, and the probe's own `timeoutSeconds` then fails it — possibly restarting a healthy pod. Bound every check.
- **Do not make health checks expensive.** They run every few seconds per pod. A `SELECT count(*)` on a large table as a "database health check" is a self-inflicted load problem. `SELECT 1` or the pool's own validation query is right.
- **Be careful with transitive checks.** If service A's readiness depends on service B's health endpoint, and B's depends on C's, a single failure cascades readiness across the fleet and takes everything out of rotation simultaneously. Check only your *own* ability to serve; degrade gracefully for optional dependencies rather than reporting unhealthy.

**Why a DB check in liveness causes cascading restarts.** Covered in detail above. This is the highest-value point in the answer and worth leading with.

---

## Q39. How does your app shut down gracefully behind a load balancer?

### Answer

**What happens on a pod delete, in order:**

1. The pod is marked `Terminating`. **Two things then happen concurrently and independently:**
   - The kubelet runs the `preStop` hook, then sends `SIGTERM` to PID 1.
   - The endpoints controller removes the pod from the Service's `EndpointSlice`, which then propagates to kube-proxy on every node, to ingress controllers, and to any client-side load balancer.
2. After `terminationGracePeriodSeconds` (default 30), the kubelet sends `SIGKILL`.

**The race:** step 1's two branches are not ordered. The application can receive `SIGTERM` and begin shutting down *before* the load balancer has stopped sending it traffic. Endpoint propagation takes anywhere from tens of milliseconds to several seconds depending on cluster size and ingress implementation. Requests arriving in that window hit a socket that is closing — connection reset, 502, or a dropped in-flight request.

**The fix has four parts.**

**1. `preStop` sleep.** Delay the `SIGTERM` long enough for endpoint removal to propagate:

```yaml
lifecycle:
  preStop:
    exec: { command: ["sh", "-c", "sleep 10"] }
```
This is the standard, slightly unsatisfying, and genuinely necessary remedy. During the sleep the pod keeps serving normally; it is simply no longer receiving *new* connections from anything that has processed the endpoint update. Five to fifteen seconds is typical; measure your cluster's propagation time.

**2. Readiness flips to `REFUSING_TRAFFIC` before shutdown begins.** Spring Boot does this automatically when graceful shutdown starts (publishing `AvailabilityChangeEvent` for `ReadinessState.REFUSING_TRAFFIC`), which causes `/health/readiness` to fail and any load balancer polling readiness to remove the pod. This helps clients that poll rather than watch endpoints.

**3. Graceful shutdown in the application.**

```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 25s
```
With `server.shutdown=graceful`, the embedded server stops accepting new connections and waits for in-flight requests to complete before closing. `timeout-per-shutdown-phase` bounds each `SmartLifecycle` phase — including the web server's — after which it proceeds anyway.

**4. `terminationGracePeriodSeconds` must exceed preStop + shutdown time.**

```
terminationGracePeriodSeconds  >  preStop sleep + timeout-per-shutdown-phase + margin
e.g. 60 > 10 + 25 + margin
```
If it does not, `SIGKILL` arrives mid-drain and you drop exactly the requests you were trying to protect. This arithmetic is where most implementations are wrong.

**Beyond HTTP:**

- **Kafka consumers.** On shutdown, the listener container stops polling, commits offsets, and leaves the consumer group. Leaving cleanly triggers a rebalance immediately, which is better than waiting for `session.timeout.ms` to expire. Ensure the container's shutdown participates in the Spring lifecycle (it does by default) and that `timeout-per-shutdown-phase` allows the current batch to finish.
- **Scheduled tasks and executors.** `ThreadPoolTaskExecutor.setWaitForTasksToCompleteOnShutdown(true)` and `setAwaitTerminationSeconds(...)`, otherwise in-flight background work is abandoned mid-operation.
- **Database connections.** HikariCP closes the pool on context close; in-flight transactions should have completed with the requests.

### Probes

**SIGTERM → `server.shutdown=graceful`.** Covered. Prerequisite: the JVM must actually *receive* `SIGTERM`, which requires it to be PID 1 or to have signals forwarded (Q64). A shell-form `CMD` that spawns `/bin/sh -c "java ..."` puts the shell at PID 1, and the shell does not forward signals — so the JVM never sees `SIGTERM`, no shutdown hook runs, and every pod termination is effectively a `SIGKILL` after the grace period. Verify this; it is very common and completely invisible until you look.

**`spring.lifecycle.timeout-per-shutdown-phase`.** Covered. Note it is per *phase*, not total, so a context with several `SmartLifecycle` phases can take a multiple of this value in the worst case — factor that into the grace period.

**Readiness flipping to OUT_OF_SERVICE *before* shutdown starts.** Covered. You can also trigger it manually ahead of shutdown by publishing `AvailabilityChangeEvent.publish(ctx, ReadinessState.REFUSING_TRAFFIC)` — useful if you want a longer, application-controlled drain before the server stops accepting connections.

**K8s `preStop` sleep to let endpoints propagate.** Covered. Two refinements: the container image needs a shell for `exec: sleep` (distroless images do not have one — use `httpGet` to an endpoint that sleeps, or a `sleep` binary, or Kubernetes 1.30+'s `sleep` lifecycle action which requires no shell). And the `preStop` hook's duration counts *against* `terminationGracePeriodSeconds`, it does not extend it.

**In-flight requests.** Graceful shutdown waits for requests already being processed. It does **not** wait for long-polling connections, SSE streams, or WebSockets, which may need explicit handling — close them with a reconnect signal so clients fail over cleanly. And note that a request already accepted but blocked on a slow downstream will hold shutdown up to the timeout; aggressive downstream timeouts (Q86) make graceful shutdown bounded.

**Consumer group rebalancing.** Covered. Additional consideration: during a rolling update every pod restarts, so the consumer group rebalances once per pod. With the default eager rebalancing this is a stop-the-world event each time; `partition.assignment.strategy=CooperativeStickyAssignor` makes rebalances incremental so unaffected partitions keep processing. **Static group membership** (`group.instance.id`, requiring a stable identity such as a StatefulSet ordinal) avoids the rebalance altogether for restarts within `session.timeout.ms` — the returning consumer reclaims its partitions. Both are worth knowing for a rolling-update discussion (Q82).

---

## Q40. What actually changes when you build a Boot app as a GraalVM native image?

### Answer

**The fundamental change: closed-world assumption.** A native image is produced by ahead-of-time compilation that statically analyses the whole program, determines every reachable method, and compiles them to a native executable. Anything not provably reachable is **not included**. There is no class loading at runtime, no JIT, no bytecode interpretation, and no ability to define new classes.

**What that breaks, and what Spring does about it.**

- **Reflection, dynamic proxies, resource loading, and serialisation** cannot be discovered statically. Every reflective access needs a *hint* recorded at build time. Spring 6 / Boot 3 include an **AOT processing** step (`process-aot`) that runs the application's configuration at build time and emits: generated Java source for the bean definitions, plus `reflect-config.json`, `proxy-config.json`, `resource-config.json`, and `serialization-config.json`. Library authors supply their own hints via `RuntimeHintsRegistrar` or the GraalVM Reachability Metadata Repository. If a library does neither, you must write hints yourself or it fails at runtime.
- **Conditions are evaluated at build time, not runtime.** `@ConditionalOnProperty`, `@ConditionalOnClass`, and `@Profile` are resolved during AOT processing and **baked in**. So you cannot change a profile or a conditional feature flag at deploy time and get different wiring — the beans that were not created at build time do not exist in the binary. This is a genuine behavioural difference that surprises teams: `SPRING_PROFILES_ACTIVE=prod` at runtime does not re-run conditions. You must build per-profile images, or move the variation to runtime configuration values rather than conditional bean registration.
- **CGLIB proxying is done at build time.** `@Configuration(proxyBeanMethods = false)` is preferred, and Spring's AOT engine transforms configuration classes accordingly.
- **No runtime bytecode generation.** Libraries that generate classes at runtime (some mocking frameworks, some ORMs' bytecode enhancement, Groovy, older AOP approaches) either need build-time equivalents or do not work.
- **Static initialisers**: Graal can run some at build time and snapshot the heap into the image (which is why startup is fast), but that means anything captured at build time is frozen — a `static final` field holding a timestamp, a hostname, a random seed, or an open file handle is a bug. Graal's `--initialize-at-run-time` controls this per class.

**What you gain.**

- **Startup time** measured in tens of milliseconds rather than seconds. No class loading, no JIT warmup, and the heap is pre-initialised.
- **Memory footprint** typically a fraction of JVM RSS — no JIT compiler, no code cache, no metaspace, smaller heap requirement.
- **Instant peak performance** — no warmup curve.
- **Smaller attack surface** — no interpreter, no dynamic class loading.

**What you lose.**

- **Peak throughput.** Without a profile-guided JIT, hot code is compiled from static analysis only. AOT-compiled code is typically slower than long-run JIT-compiled code (GraalVM Enterprise/Oracle GraalVM's **PGO** — profile-guided optimisation from an instrumented run — closes much of this gap, at the cost of a more complex build).
- **Build time.** Native compilation takes minutes and a lot of memory (commonly 8–16 GB), which affects CI cost and developer feedback loops significantly.
- **Tooling.** JFR support exists but is more limited; standard JVM profilers, `jcmd`, `jstack`, heap dumps, and attach-based agents largely do not apply. Debugging is via native debug info and gdb-style tooling.
- **Runtime flexibility.** No agents (no APM Java agents — you need OpenTelemetry compiled in rather than attached), no `-XX` heap tuning in the usual sense (though ZGC/Serial options exist for native), no dynamic log-level libraries that rely on reflection unless hinted.
- **A different testing burden.** Because build-time evaluation differs from JVM behaviour, you must run your test suite against the native image (`-PnativeTest`), which is slow. A bug that only manifests natively is not caught by JVM tests.

**When it is worth it.** Serverless (AWS Lambda, Cloud Run) where cold start dominates cost and latency; scale-to-zero workloads; CLI tools; very high-density deployments where per-instance memory is the binding constraint; short-lived batch jobs. **Not** worth it for a long-running service where throughput matters and instances live for days — there the JIT wins and the operational cost of native is pure downside.

**The alternatives worth mentioning:** Class Data Sharing (`-XX:ArchiveClassesAtExit`, and Boot 3.3+'s support for CDS with a training run) gives a meaningful startup improvement with none of the native-image constraints, and Project Leyden is working toward a spectrum of AOT options within the JVM. For most teams wanting faster startup, CDS is the far cheaper first step.

### Probes

**AOT processing.** Covered. Note that AOT processing also runs for JVM deployments if you enable it (`spring.aot.enabled` / the `process-aot` goal), giving some startup benefit without native compilation — the generated bean definitions skip reflection-heavy configuration parsing.

**Closed-world assumption.** Covered — the root cause of every other item.

**Reflection/proxy/resource hints.** Covered. Practical guidance: `RuntimeHintsRegistrar` plus `@ImportRuntimeHints` for your own code; the GraalVM tracing agent (`-agentlib:native-image-agent=config-output-dir=...`) run against your test suite to *discover* hints empirically — useful but only as good as your test coverage, since anything not exercised is not recorded, which is exactly how you ship a binary that fails on an untested code path.

**No runtime `@Conditional` re-evaluation.** Covered — this is the most operationally significant difference and the one most likely to catch a team out. State it explicitly: *the wiring is fixed at build time*.

**Faster startup / lower memory vs slower peak throughput and longer builds.** Covered with the trade-offs quantified qualitatively. Be honest about the throughput point rather than presenting native image as strictly better.

**When it's worth it (serverless, scale-to-zero).** Covered, including when it is not.

---

# 5. JPA / Hibernate

---

## Q41. Explain the persistence context. What does dirty checking cost you, and how do you avoid an accidental UPDATE?

### Answer

**What it is.** The persistence context (`EntityManager` / Hibernate `Session`) is a **first-level cache and unit of work**. It holds every entity loaded or persisted during its lifetime, keyed by entity type plus identifier, and guarantees that within one context there is **exactly one Java object per database row**. So `em.find(Order.class, 1L) == em.find(Order.class, 1L)` is `true`, and a repeated `find` for a loaded entity issues no SQL at all.

Entity states: **transient** (new, never associated), **managed/persistent** (in the context, tracked), **detached** (was managed, context closed or `detach`ed), **removed** (marked for deletion at flush).

In Spring, the persistence context is normally bound to the **transaction** — one context per `@Transactional` boundary — unless `open-in-view` extends it to the whole request (Q43).

**Dirty checking.** On `flush` (at commit, before a query whose results could be affected, or on explicit `flush()`), Hibernate compares every managed entity against a **snapshot** taken when it was loaded, field by field. Any difference produces an `UPDATE`. This is why you never call `save()` on a managed entity — mutation alone is enough:

```java
@Transactional
public void rename(Long id, String name) {
    Order o = repo.findById(id).orElseThrow();
    o.setName(name);        // that's it — UPDATE at commit
}
```

**What it costs.**

1. **Memory.** Every loaded entity is retained *twice* — the entity and its snapshot array — for the life of the context. Loading 100,000 entities in one transaction means 200,000 objects live until commit, and Hibernate never evicts from the first-level cache automatically. This is the classic OOM in batch jobs.
2. **CPU at flush.** The comparison is O(managed entities × fields). It happens at every flush, not just at commit. In a loop that queries repeatedly, `FlushModeType.AUTO` triggers a flush before each query that touches affected tables — so you can pay the dirty-check cost dozens of times in one transaction.
3. **Unintended updates.** Any mutation of a managed entity is persisted, including ones you did not intend — a mapper that "normalises" a field, a `toString` that lazily populates something, a bidirectional-association helper, a `@PrePersist` listener.

**How to avoid an accidental UPDATE.**

- **`@Transactional(readOnly = true)`** — sets Hibernate's flush mode to `MANUAL`, so no automatic flush occurs and no dirty checking runs. This is the primary tool. It also skips snapshot retention in some Hibernate versions (`readOnly` entities are loaded without a snapshot), saving memory. Caveat from Q27: *writes are silently discarded*, so it is a real contract.
- **DTO projections** — select into a DTO/record rather than an entity. Nothing is managed, nothing is dirty-checked, nothing is snapshotted, and you fetch only the columns you need. This is the strongest option and should be the default for read paths.
- **`em.detach(entity)` / `session.evict(entity)`** to remove a specific entity from tracking, or `em.clear()` to clear the whole context (essential in batch loops).
- **`Session.setReadOnly(entity, true)`** or `@QueryHints(@QueryHint(name = HINT_READ_ONLY, value = "true"))` for a specific query — entities are loaded without a snapshot and are never dirty-checked, even in a read-write transaction.
- **Immutable entities** — `@Immutable` on the entity tells Hibernate never to generate updates for it.

### Probes

**First-level cache.** Covered. Two consequences worth stating: (a) it is not a performance cache you configure — it is a correctness mechanism providing identity guarantees within a unit of work, and it is always on; (b) it means a JPQL query returning a row already in the context returns the **existing managed instance**, discarding the freshly-read column values. So a query cannot be used to "refresh" an entity you have already modified — you need `em.refresh()`. This surprises people who expect a query to reflect the database.

**Snapshot comparison at flush.** Covered. Note also that Hibernate's `AUTO` flush mode flushes before a query when it detects that the query's tables overlap with pending changes — the detection is conservative for native SQL queries (Hibernate cannot parse arbitrary SQL to know which tables are involved, so it may flush everything, or with `FlushMode.COMMIT` may not flush when it should, giving stale results). Using `@Modifying(flushAutomatically = true, clearAutomatically = true)` on Spring Data bulk update methods addresses the analogous problem there.

**Detached vs managed.** A detached entity is a plain Java object. Mutating it does nothing. `merge()` copies its state onto a managed instance (**returning that managed instance** — a critical detail: `merge` does not make the argument managed, so `em.merge(o); o.setX(...)` does not persist the change; you must use the returned object). `merge` also issues a `SELECT` first unless the entity is versioned and detectably new, which makes it more expensive than `persist`.

**`@Transactional(readOnly=true)`.** Covered here and in Q27, including the replica-routing benefit and the silent-write-loss hazard.

**DTO projections skipping the persistence context.** The single most effective technique for read-heavy paths. Forms available:

- JPQL constructor expression: `select new com.x.OrderSummary(o.id, o.total, c.name) from Order o join o.customer c`
- Spring Data **interface projections** — declare an interface with getters; Spring generates a proxy. Closed projections (only direct properties) generate an optimised query selecting just those columns; open projections using `@Value("#{target...}")` fetch the whole entity, which defeats the purpose.
- Spring Data **class/record projections** — a record constructor matching the selected columns; Hibernate 6 supports records directly.
- `Tuple` or native queries with a `ResultTransformer` for complex cases.

The benefits compound: fewer columns transferred, no entity instantiation, no snapshot, no dirty check, no lazy-loading proxies, no `LazyInitializationException` risk downstream, and the result is immutable and safe to cache or return from the service layer.

**`evict`/`detach`.** Covered. The canonical batch loop:

```java
for (int i = 0; i < items.size(); i++) {
    em.persist(items.get(i));
    if (i % 50 == 0) { em.flush(); em.clear(); }   // bound the context
}
```
`flush` pushes the SQL, `clear` releases the entities and snapshots. Without this, memory grows linearly and dirty checking cost grows quadratically (each flush re-checks everything accumulated so far). Combine with `hibernate.jdbc.batch_size` (Q45) so the flushed statements are actually batched.

---

## Q42. Diagnose and fix an N+1 problem four different ways, and say when each is wrong.

### Answer

**The problem.** One query fetches N parent rows; accessing a lazy association on each parent triggers one query per parent. Total: N+1 queries. With N=500 and 2 ms per round trip, that is a second of pure latency, and it scales with data volume rather than being a constant cost — so it passes testing with 10 rows and destroys production with 10,000.

**Diagnosis.** Do not rely on reading code:

- Enable SQL logging (`spring.jpa.show-sql` for a quick look; better, `logging.level.org.hibernate.SQL=DEBUG` plus `org.hibernate.orm.jdbc.bind=TRACE` for parameters in Hibernate 6).
- **Better: count queries per request.** Datasource-proxy or `p6spy` can log a per-transaction query count; a test using `SQLStatementCountValidator` (from `hypersistence-utils`) can *assert* the count, turning N+1 into a failing test rather than a production incident. This is the technique to mention — it is preventive rather than reactive.
- APM traces show the repeated statement immediately.
- `hibernate.generate_statistics=true` exposes query counts via `SessionFactory` statistics and Micrometer.

**Fix 1 — `JOIN FETCH`.**
```java
@Query("select o from Order o join fetch o.items where o.customer.id = :id")
```
One query, associations initialised eagerly for that query only.

*When it is wrong:* (a) **Pagination breaks.** With a `JOIN FETCH` on a collection plus `setFirstResult/setMaxResults`, Hibernate cannot apply `LIMIT` in SQL (the join multiplies rows, so `LIMIT 20` would truncate mid-parent). Older versions emitted `HHH000104: firstResult/maxResults specified with collection fetch; applying in memory` and fetched **the entire result set** into memory, then paginated — a latent OOM. Hibernate 6 makes this an exception by default in some configurations; either way, do not do it. (b) **Cartesian products** with two or more collection fetches: fetching 10 items and 5 payments produces 50 rows per order, and Hibernate must de-duplicate. Hibernate 6 de-duplicates automatically; Hibernate 5 needed `distinct` plus `hibernate.query.passDistinctThrough=false`. Either way, the *data transfer* is quadratic. `MultipleBagFetchException` is thrown outright when you fetch two `List` associations. (c) It fetches whole entities, so it is heavier than a projection when you only need a few fields.

**Fix 2 — `@EntityGraph`.**
```java
@EntityGraph(attributePaths = {"items", "customer"})
List<Order> findByStatus(Status s);
```
Declarative equivalent of `JOIN FETCH`, reusable, and composable with Spring Data derived queries and Specifications. `FETCH` type overrides the mapping's laziness for listed attributes; `LOAD` uses the mapping's default for unlisted ones.

*When it is wrong:* the same pagination and cartesian-product problems, because it produces the same SQL. It is a better *expression* of the fix, not a different fix.

**Fix 3 — `@BatchSize` / `hibernate.default_batch_fetch_size`.**
```java
@OneToMany(mappedBy = "order") @BatchSize(size = 50) private List<Item> items;
```
Instead of one query per parent, Hibernate collects up to 50 pending proxies and issues `... where order_id in (?, ?, ... )`. N+1 becomes N/50 + 1.

*When it is wrong:* it is not a single query, so there is still more than one round trip; the `IN` list can hit database parameter limits (and can cause query-plan cache pollution in PostgreSQL/Oracle with varying list sizes — mitigated by Hibernate 6's padding of batch sizes to powers of two). But as a **global default** (`spring.jpa.properties.hibernate.default_batch_fetch_size=50`) it is one of the highest-value single settings in a JPA application: it converts every unnoticed N+1 into a bounded number of queries automatically. Strongly recommended even alongside explicit fixes.

**Fix 4 — Subselect fetching.**
```java
@OneToMany(mappedBy = "order") @Fetch(FetchMode.SUBSELECT) private List<Item> items;
```
Hibernate re-runs the original query as a subquery: `... where order_id in (select id from orders where status = ?)`. Two queries total regardless of N.

*When it is wrong:* the subquery re-executes the (possibly expensive) original query; it applies to all collections of that mapping, not per query site; and it interacts awkwardly with pagination.

**Fix 5 — DTO projection.** Often the real answer: write a query that returns exactly the data the caller needs, in one statement, with no entities.

*When it is wrong:* when you need managed entities to modify them.

**Why `FetchType.EAGER` on the mapping is the worst fix.** It is global: *every* query that loads the entity now joins or issues extra selects, including ones that never touch the association. It cannot be turned off per query (`JOIN FETCH` can add eagerness, but nothing removes it — you would have to switch to a projection). It causes cartesian explosions when several eager associations combine, and it makes `findById` on one entity drag in an object graph of unknown size. `@ManyToOne` defaults to `EAGER` in the JPA spec, which is itself a design mistake — set `fetch = LAZY` explicitly on every association and add fetching per query. This is one of the most valuable habits in JPA work.

### Probes

All five fixes and their failure modes are covered above. Additional detail on the specific probe items:

**`JOIN FETCH` breaks pagination — HHH warns about in-memory paging.** Covered. The correct pattern when you need both pagination and a fetched collection is the **two-query approach**: first page the IDs (`select o.id from Order o where ... order by ... limit/offset` — no join, so `LIMIT` works), then `select o from Order o join fetch o.items where o.id in :ids`. Hibernate 6 can do this automatically for some cases, and Spring Data's `@EntityGraph` on a `Page` method will typically issue the count query and the fetch separately, but verify the generated SQL rather than assuming.

**`@EntityGraph`.** Covered. Named entity graphs (`@NamedEntityGraph` on the entity) are useful when the same graph is needed in several places; ad-hoc `attributePaths` are more readable for one-offs. Sub-graphs (`items.product`) are supported via dotted paths in Spring Data.

**`@BatchSize`.** Covered, including the recommendation to set it globally.

**Subselect fetching.** Covered.

**Second-level cache.** A `SessionFactory`-wide cache (Ehcache, Hazelcast, Infinispan, Caffeine via a provider) that survives across transactions. It can eliminate N+1 for reference data. Requirements: `@Cacheable` on the entity, `hibernate.cache.use_second_level_cache=true`, a cache region factory, and — for collections — `@Cache` on the association. Caveats that make it a poor first resort: it is invalidated only by Hibernate, so any direct SQL update (a migration, a batch job, another service) leaves it stale; in a multi-instance deployment you need a distributed cache or you get per-instance divergence; the query cache in particular is notorious for being slower than no cache when it has a low hit rate, because it validates timestamps against every table involved. Use it for genuinely static reference data with a clear invalidation story, not as a fix for a query problem.

**Why `FetchType.EAGER` on the mapping is the worst fix.** Covered above in full.

---

## Q43. What is `LazyInitializationException` really telling you, and why is Open-Session-In-View a bad default?

### Answer

**What the exception means.** A lazy association is represented by a **proxy** (for `@ManyToOne`/`@OneToOne`, a bytecode-generated subclass of the entity; for collections, a `PersistentBag`/`PersistentSet` wrapper). The proxy holds a reference to the `Session` that created it. When you touch it, it asks that session to load the data. If the session has been closed — because the transaction ended — the proxy throws `LazyInitializationException: could not initialize proxy - no Session`.

So the exception is telling you: **you tried to load data outside the boundary where loading was possible.** The real message is that your fetch plan does not match your access pattern. It is a design signal, not an inconvenience to be configured away.

**The correct fix** is to load what you need *inside* the transaction: `JOIN FETCH`/`@EntityGraph` for the associations you will use, or — better — return a DTO from the service layer so nothing lazy ever escapes it. The persistence model should not leak past the service boundary at all.

**Open Session In View.** OSIV keeps the `EntityManager`/`Session` open for the entire HTTP request, via a filter/interceptor, so lazy loading still works in the controller and during response serialisation. Spring Boot enables it by default (`spring.jpa.open-in-view=true`) and logs a warning about it at startup.

**Why it is a bad default:**

1. **It holds a database connection for the whole request.** With OSIV plus a `DataSource` that binds a connection to the session, the connection is acquired when the session opens and held until the response is written — including time spent in template rendering, JSON serialisation, and any external HTTP call the controller makes. A pool of 10 connections and a 200 ms request means a hard ceiling of ~50 req/s, and a slow downstream in the controller exhausts the pool. (Hibernate 5.2+ with `hibernate.connection.handling_mode=DELAYED_ACQUISITION_AND_RELEASE_AFTER_TRANSACTION` mitigates this somewhat, but OSIV still extends the session.)
2. **Hidden queries during serialisation.** Jackson walks the object graph and touches lazy associations, silently issuing queries. You get N+1 problems that are invisible in the service layer — the code looks like one repository call, and the database sees hundreds of statements. Worse, these queries run **outside any transaction** (the transaction ended at the service boundary), each in its own auto-commit, so the response can be assembled from an inconsistent mix of snapshots.
3. **It hides the design problem.** Developers never learn what their fetch plan is, because everything "just works" until load testing. It systematically produces applications whose data access is unanalysable.
4. **Failures move to the worst possible place.** An exception thrown during serialisation happens after the response status and headers may already have been committed, producing truncated responses and unhelpful errors.
5. **It encourages returning entities from controllers**, which couples the API contract to the persistence model — so a mapping change becomes a breaking API change, and lazy proxies leak into serialisation.

**Recommendation:** set `spring.jpa.open-in-view=false`, fix the resulting `LazyInitializationException`s properly (they will point at exactly the places where the fetch plan is wrong), and return DTOs. Do this early in a project; retrofitting it later is painful precisely because OSIV has been hiding the problems.

### Probes

**Session closed at tx boundary.** Covered.

**OSIV holds the connection for the whole request → pool exhaustion.** Covered with the throughput arithmetic.

**Hidden queries during serialisation.** Covered, including the auto-commit and consistency consequences.

**Boot warns about it.** The startup message is `spring.jpa.open-in-view is enabled by default. Therefore, database queries may be performed during view rendering. Explicitly configure spring.jpa.open-in-view to disable this warning.` Boot deliberately left the default `true` for backwards compatibility while warning, which is why so many applications run with it enabled unknowingly. Turning it off is a deliberate decision every project should make consciously.

**Correct fix is loading what you need in the service layer.** Covered. Practical patterns:

- Return records/DTOs from the service; map inside the transaction.
- Use `@EntityGraph` on the repository method for the specific use case.
- For a REST API, project directly to the response shape in the query — the mapping layer disappears entirely and you fetch minimal columns.
- If you must return entities (e.g. to a template), initialise explicitly with `Hibernate.initialize(o.getItems())` inside the transaction, which at least makes the intent visible.

Two related items worth mentioning: `Hibernate.isInitialized(proxy)` lets you check without triggering a load; and with bytecode enhancement (`hibernate-enhance-maven-plugin`) you can get lazy *attribute* loading (`@Basic(fetch = LAZY)` on a large `@Lob` column) so a big blob is not fetched with every row — a genuinely useful optimisation that requires build-time enhancement.

---

## Q44. Compare optimistic locking (`@Version`) with `PESSIMISTIC_WRITE`. Design a "decrement stock" operation under high contention.

### Answer

**Optimistic (`@Version`).** A version column on the entity; Hibernate appends `AND version = ?` to every `UPDATE` and increments it. Zero rows affected → `OptimisticLockException` (Spring translates to `ObjectOptimisticLockingFailureException`). No database locks are held between read and write. Excellent when conflicts are rare and think-time is long. Requires the caller to retry.

**Pessimistic (`LockModeType.PESSIMISTIC_WRITE`).** Issues `SELECT ... FOR UPDATE`, taking a row lock held until commit. Concurrent transactions attempting to lock the same row **block**. Guarantees the read-modify-write is serialised. Costs: lock duration equals transaction duration, so a slow transaction blocks everyone; deadlock risk when multiple rows are locked in varying order; and connections pile up waiting.

**Designing "decrement stock" under high contention.**

**Approach A — the correct default: a single atomic conditional UPDATE.**

```sql
UPDATE product SET qty = qty - :n WHERE id = :id AND qty >= :n
```
```java
@Modifying
@Query("update Product p set p.qty = p.qty - :n where p.id = :id and p.qty >= :n")
int reserve(@Param("id") Long id, @Param("n") int n);
// caller: if (reserve(id, n) == 0) throw new InsufficientStockException();
```

Why this wins under contention:
- The read and write are one statement, so the database holds the row lock for microseconds rather than for the duration of your transaction.
- No version conflict is possible — there is no separate read to become stale.
- No retry loop, no exception handling for concurrency.
- The `WHERE qty >= :n` guard makes overselling impossible regardless of isolation level.
- The affected-row count *is* the business outcome.

Costs: it bypasses the persistence context, so any loaded `Product` entity is now stale (use `clearAutomatically = true` on `@Modifying`, or simply do not load the entity); and it cannot express logic that requires more than SQL.

**Approach B — pessimistic lock**, when the decision requires reading several things and applying business rules:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
Optional<Product> findForUpdate(Long id);
```
Keep the transaction as short as possible: acquire the lock as late as you can, do nothing slow while holding it, and never make an external call inside it (Q28). Set a lock timeout so a stuck transaction fails rather than piling up waiters.

**Approach C — optimistic with bounded retry**, when contention is genuinely low or the operation is user-driven with think-time:

```java
@Retryable(retryFor = ObjectOptimisticLockingFailureException.class,
           maxAttempts = 3, backoff = @Backoff(delay = 50, multiplier = 2, random = true))
@Transactional
public void reserve(Long id, int n) { ... }   // must be on a different bean than the caller
```
Under *high* contention this degrades badly: many concurrent writers means most retries also fail, wasting work and adding latency. Optimistic locking is the wrong tool for a hot row.

**Approach D — remove the contention.** If a single SKU is genuinely hot (flash sale), even a microsecond-long row lock serialises everything at that row. Options: **shard the counter** into N rows (`product_stock(product_id, shard, qty)`), decrement a random shard with fallback to others, and sum for display (accepting approximate reads); or move the reservation to a queue and process serially at a known rate; or pre-allocate blocks of inventory to instances. These are the techniques that actually scale a hot counter, and mentioning them shows you understand the limit of the database-level answers.

**Design note on correctness:** whichever approach, "reserve" and "confirm" should be separate steps with a timeout, so abandoned carts release inventory. And the operation must be idempotent at the API level (Q56) so a client retry does not double-decrement.

### Probes

**`OptimisticLockException` + retry.** Covered. Critical constraint: the retried work must be free of external side effects (Q22), and the retry must re-read state rather than reusing the stale entity — with Spring Retry on a `@Transactional` method, the transaction is rolled back and a fresh one starts, so the re-read happens naturally, provided the method does the read itself rather than receiving an entity as a parameter.

**Lost-update anomaly.** Two transactions read `qty = 10`, both compute `9`, both write `9`; one decrement is lost. `READ COMMITTED` does not prevent this. `@Version` catches it (the second update matches zero rows). `SELECT FOR UPDATE` prevents it. Approach A avoids it structurally by never reading first.

**Row-level locks and deadlock ordering.** If a transaction locks several products, always acquire in a deterministic order — sort the IDs ascending before locking. Two transactions reserving products {1,2} and {2,1} in insertion order will deadlock; sorted, they serialise cleanly. This applies to any multi-row locking, and it is one of the few genuinely universal rules in this area.

**Single atomic `UPDATE ... SET qty = qty - 1 WHERE qty >= 1` as the often-better answer.** Covered as approach A, with its costs.

**Idempotency of the retry.** Covered.

---

## Q45. What's wrong with using `@GeneratedValue(strategy = AUTO/IDENTITY)` when you want JDBC batch inserts?

### Answer

**The mechanism.** `IDENTITY` maps to a database auto-increment column (`SERIAL`/`IDENTITY`/`AUTO_INCREMENT`). The generated value is only known **after** the row is inserted. But JPA requires that `persist()` assign the identifier immediately, because the persistence context is keyed by identifier — it must be able to put the entity in the identity map right away.

Consequently Hibernate must execute the `INSERT` **at `persist()` time**, immediately, and read back the generated key. It cannot defer the statement to flush time, and therefore **it cannot batch inserts at all**. Hibernate disables JDBC batching for `IDENTITY` entities silently — you set `hibernate.jdbc.batch_size=50`, see no error, and get 10,000 individual round trips.

**The fix: sequences.**

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
@SequenceGenerator(name = "order_seq", sequenceName = "order_seq", allocationSize = 50)
private Long id;
```

With a sequence, Hibernate obtains identifiers **before** inserting, so it can assign IDs at `persist()` and defer all the `INSERT`s to flush time, where they batch.

**`allocationSize` (the pooled optimiser)** is the second half of the win. With `allocationSize = 50`, Hibernate calls `nextval` once and then hands out 50 identifiers in memory — 1 round trip instead of 50. **The database sequence must be created with a matching `INCREMENT BY 50`**, or you get duplicate-key collisions between instances. Hibernate's default `allocationSize` is 50, but a sequence created by a hand-written migration usually defaults to `INCREMENT BY 1` — this mismatch is a classic production bug. Either set the sequence increment to match, or set `allocationSize = 1` (and lose the batching of ID fetches). Hibernate's `pooled` and `pooled-lo` optimisers handle this correctly and interoperate with other writers of the same sequence.

**`AUTO`** delegates the choice to the dialect. In Hibernate 5+ on most databases it resolves to `SEQUENCE`, but historically it resolved differently per database and produced a shared `hibernate_sequence` table or sequence used by *all* entities — a single hot row and a portability hazard. Being explicit is always better than `AUTO`.

**`TABLE`** strategy uses a database table as a sequence emulator. Portable but slow and contended (it requires a row lock per allocation); avoid unless you have no alternative.

**MySQL note:** MySQL had no sequences before 8.0 and still lacks `CREATE SEQUENCE` in the standard sense, so `IDENTITY` (auto-increment) is effectively forced, and batch inserts are therefore unavailable through Hibernate. Workarounds: assign IDs in the application (UUIDs, or a TSID/snowflake generator), or use the `TABLE` strategy, or bypass JPA for bulk inserts with `JdbcTemplate.batchUpdate` — which is often the right answer for genuine bulk loading regardless of database.

### Probes

**`IDENTITY` disables batching (needs the ID immediately).** Covered — the core of the answer.

**Sequence + pooled/hi-lo optimiser.** Covered. `hi-lo` is the older scheme (a "hi" value from the sequence, multiplied by a factor, with "lo" values allocated in memory) and is **not interoperable** with other applications inserting into the same table, because the sequence value does not correspond to actual IDs. `pooled` and `pooled-lo` are the modern, interoperable equivalents and are what Hibernate uses by default for `allocationSize > 1`. If another system writes to the same table, this distinction matters a great deal.

**`hibernate.jdbc.batch_size`.** Set it (typically 20–100) or nothing batches regardless of ID strategy. Complementary settings:

```properties
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
spring.jpa.properties.hibernate.batch_versioned_data=true
```
Also add `reWriteBatchedInserts=true` to the PostgreSQL JDBC URL — without it, the driver sends batched inserts as separate statements over the wire and you get much of the round-trip cost back; with it, they are rewritten into multi-row `INSERT` statements, which is a large improvement.

**`order_inserts`/`order_updates`.** JDBC batching only works for **consecutive identical statements**. If you persist Order, Item, Order, Item, the statement changes every time and every batch has size 1. `order_inserts=true` sorts pending inserts by entity type so all Orders batch together and all Items batch together. `order_updates=true` does the same for updates (and additionally sorts by primary key, which reduces deadlock probability by imposing a consistent lock order — a useful side benefit). `batch_versioned_data=true` allows batching for entities with `@Version`, which requires the JDBC driver to report accurate per-statement update counts (true for PostgreSQL and most modern drivers; historically false for some, which is why it is not the default everywhere).

Verify batching actually happened rather than assuming: enable `hibernate.generate_statistics` and check the JDBC batch counts, or use datasource-proxy to log actual batch sizes. It is easy to configure batching and silently not get it.

**UUID v7 vs random UUID as a PK and index locality.** A random UUID (v4) as a primary key has poor **index locality**: successive inserts land at random positions in the B-tree, so every insert touches a different leaf page. Consequences: the working set of dirty pages is large, buffer-pool hit rate falls, page splits are frequent, and the index becomes fragmented. On a large table this can be several times slower than a sequential key, and it inflates WAL/redo volume. Also, a UUID is 16 bytes versus 8 for a `bigint`, and in PostgreSQL every secondary index stores the heap tuple ID rather than the PK (so the PK width matters less than in InnoDB, where the clustered PK is embedded in every secondary index — making wide random PKs especially costly on MySQL).

**UUID v7** (RFC 9562, published 2024) embeds a millisecond Unix timestamp in the high bits, so values are time-ordered. Inserts append to the right-hand edge of the index like a sequence, restoring locality, while keeping the properties that make UUIDs attractive: client-side generation (no database round trip, no coordination), safe merging of data from multiple sources, and no enumeration of your row counts by an attacker. **TSID/ULID** are similar time-ordered alternatives, with ULID offering a sortable string encoding. Hibernate 6.2+ supports `@UuidGenerator(style = TIME)` for time-based UUIDs.

The trade-off to state: v7 leaks creation time (usually harmless, occasionally not), and it still costs 16 bytes. If you do not need distributed generation, a `bigint` sequence remains the cheapest and fastest option — use UUIDs when you actually need the property they provide, not by default.

---

## Q46. When would you drop JPA and use JDBC/jOOQ/`JdbcClient` instead?

### Answer

JPA optimises for a specific case: **loading a graph of objects, mutating them, and writing them back**, with the ORM managing identity, change tracking, and cascade. When the work does not have that shape, the ORM is overhead — often substantial — and sometimes an obstacle.

**Cases where I would not use JPA:**

**1. Bulk operations.** Updating or deleting a million rows through entities means loading a million objects, snapshotting them, and issuing a million statements. `UPDATE orders SET status = 'ARCHIVED' WHERE created_at < ?` is one statement. JPQL bulk operations (`@Modifying`) work but bypass the persistence context and the second-level cache, so you must be careful about staleness — at which point plain JDBC is clearer about what is happening.

**2. Reporting and analytics queries.** Aggregations, window functions, CTEs, pivots, `GROUPING SETS`, recursive queries. JPQL/Criteria either cannot express these or express them painfully. Native queries in JPA are possible but you lose type safety and gain string SQL — jOOQ gives you the same SQL with compile-time checking against the actual schema.

**3. Complex, database-specific SQL.** `ON CONFLICT DO UPDATE`, `RETURNING`, `LATERAL`, `DISTINCT ON`, `SKIP LOCKED`, JSONB operators, full-text search, array operations, geospatial. These are the reasons you chose PostgreSQL; JPA abstracts them away.

**4. Write-heavy paths with simple shapes.** Inserting events, appending to a log, recording metrics. There is no object graph, no identity requirement, no change tracking need — just `INSERT`. `JdbcTemplate.batchUpdate` is faster and simpler.

**5. Read models in a CQRS design.** The read side wants denormalised, query-shaped data returned as DTOs. There is nothing to track and nothing to mutate. Projections in JPA get you most of the way; jOOQ or `JdbcClient` gets you there more directly and with better SQL.

**6. When the SQL matters and must be reviewed.** In systems where a DBA reviews queries, or where query plans are tuned deliberately, generated SQL is a liability — you cannot review what the ORM might emit next release.

**7. High-performance paths** where entity instantiation, snapshotting, and proxying show up in a profile.

**The tools:**

- **`JdbcClient`** (Spring Framework 6.1+) — a fluent wrapper over `JdbcTemplate`: `jdbcClient.sql("select ...").param("id", id).query(Order.class).list()`. The modern default for plain SQL in Spring, considerably more pleasant than `JdbcTemplate` and with no extra dependency.
- **`NamedParameterJdbcTemplate` / `JdbcTemplate`** — still fine, still everywhere.
- **jOOQ** — generates typed Java classes from your schema, so queries are compile-time checked against real columns and types, with full support for advanced SQL. Costs: a code-generation step tied to a live schema or migration scripts, a commercial licence for commercial databases (open source for PostgreSQL/MySQL and other open databases), and a learning curve. Very strong choice for query-heavy systems.
- **Spring Data JDBC** — a lighter alternative to JPA: aggregate-oriented, no lazy loading, no dirty checking, no proxies, explicit `save()`. Closer to DDD aggregate semantics and far simpler to reason about, at the cost of no automatic change tracking.
- **MyBatis** — SQL in XML or annotations with result mapping; popular where SQL ownership matters.

**The honest position for an interview:** **mixing is normal and correct.** JPA for the transactional aggregate-mutation paths where identity and change tracking earn their keep; `JdbcClient`/jOOQ for reads, reports, and bulk work. Both can share the same `DataSource` and the same Spring transaction, so they compose cleanly. Insisting on one tool for everything is the mistake.

### Probes

**Bulk operations, reporting queries, complex SQL, write-heavy paths, CQRS read side.** All covered above.

**Mixing both in one app and the flush-ordering hazards.** This is the important technical caveat and the reason the probe exists.

When JPA and raw JDBC share a transaction, they share the same JDBC `Connection` (Spring's `DataSourceUtils` ensures this, which is what makes them transactionally consistent). But **Hibernate defers its SQL until flush**, while `JdbcTemplate` executes immediately. So:

```java
@Transactional
public void process(Order o) {
    o.setStatus(SHIPPED);                                  // pending UPDATE, not yet flushed
    jdbcClient.sql("select status from orders where id=?")  // executes NOW
              .param(o.getId()).query(String.class).single();  // sees the OLD status
}
```

Hibernate's auto-flush only triggers for **JPQL/Criteria queries** whose tables it can determine — it has no visibility into a `JdbcTemplate` call and will not flush for it. Results:

- Native/JDBC reads see stale data.
- Native/JDBC writes can conflict with pending Hibernate changes, producing constraint violations or lost updates at flush time.
- Ordering of statements differs from source order, which matters for constraint checking and for locking order (deadlocks).

**Mitigations:** call `entityManager.flush()` explicitly before any JDBC access that could be affected; or perform the JDBC work in a separate transaction or before any entity mutation; or annotate JPA bulk operations with `@Modifying(flushAutomatically = true, clearAutomatically = true)` so Hibernate flushes before and clears after (avoiding both staleness in the database and staleness in the persistence context). The general discipline is to keep entity-mutating code and raw-SQL code in **separate methods and separate transactions** where possible, precisely so this interleaving cannot occur.

A related hazard: a JDBC `UPDATE` that changes rows Hibernate has loaded leaves those entities stale in the persistence context and in the second-level cache. Hibernate will not know. `em.clear()` or `em.refresh(entity)` after such an update, and consider whether the second-level cache needs eviction (`SessionFactory.getCache().evict(...)`).

---

# 6. SQL & Databases

---

## Q47. Given `SELECT * FROM orders WHERE customer_id = ? AND status = ? ORDER BY created_at DESC LIMIT 20`, design the index. Justify column order.

### Answer

**The index:**

```sql
CREATE INDEX idx_orders_cust_status_created
    ON orders (customer_id, status, created_at DESC);
```

**Why that column order — the rule**

A B-tree index is a sorted structure over the concatenated key. To use it efficiently, the engine reads it left to right, and it can only continue past a column if that column was constrained by **equality**. The general rule for composite index ordering is:

1. **Equality predicates first** (`customer_id = ?`, `status = ?`) — any order among themselves for *matching* purposes.
2. **Then the `ORDER BY` / range column** (`created_at`).
3. **Then columns needed only for output** (covering — see below).

The reason step 2 works: once `customer_id` and `status` are pinned to single values, the remaining index entries for that prefix are **already physically sorted by `created_at`**. The database can walk that slice backwards and stop after 20 rows. No sort node, no reading the other 4,999,980 orders. This is the whole point of the index here — it satisfies both the filter *and* the ordering, and `LIMIT 20` becomes genuinely O(20) rather than "find everything, sort, discard".

**Why `(status, customer_id)` may be worse**

For *this single query*, `(status, customer_id, created_at)` performs identically — both are equality predicates, so both prefixes are fully pinned and the `created_at` ordering still holds. The B-tree does not care about the order of equality columns for this access path.

It matters for everything else:

- **Left-prefix rule.** A composite index `(a, b, c)` can serve queries on `(a)`, `(a, b)`, and `(a, b, c)`, but not `(b)` or `(b, c)` alone (Postgres can technically scan the whole index for a non-prefix predicate, but that is a full index scan, not a seek — and MySQL will typically just ignore it). So `(customer_id, status, ...)` also serves "all orders for a customer", which is an extremely common query. `(status, customer_id, ...)` serves "all orders with status X across all customers" — usually a much less useful and much larger result.
- **Selectivity and clustering.** `customer_id` is high-cardinality (millions of distinct values); `status` is low-cardinality (perhaps 6 values, and heavily skewed — most orders are `COMPLETED`). Putting the highly selective column first means the seek lands on a very small, tightly clustered slice of the index. Putting `status` first means the index is organised into six enormous runs, and while the seek to `(status='NEW', customer_id=42)` still works, the index is less useful for *other* queries and its leading column provides almost no pruning on its own.
- **Statistics and planner behaviour.** With low-cardinality leading columns, correlated-column misestimates are more likely, and the planner may choose a bitmap scan or a seq scan.

The honest nuance: the "most selective column first" heuristic is widely repeated and only *loosely* true. For pure equality prefixes it makes no difference to this query's cost; it matters for index reusability across the workload and for cases involving ranges. The precise rule is the three-step one above.

**Why `DESC` in the index definition**

PostgreSQL and MySQL 8+ support per-column ordering in the index. A plain ascending index can usually be scanned backwards at near-identical cost, so `DESC` is often unnecessary for a *single* sort column. It becomes necessary with **mixed-direction sorts** — `ORDER BY status ASC, created_at DESC` cannot be satisfied by scanning an all-ascending index in either direction; you need `(status ASC, created_at DESC)`. (MySQL 5.7 and earlier ignored `DESC` in index definitions entirely, silently creating an ascending index — a genuine source of confusion.)

**`SELECT *` is the remaining problem**

Even with the perfect index, `SELECT *` forces the engine to fetch each of the 20 matching rows from the heap/clustered table to get the other columns. That is 20 random I/Os on top of the index seek. Fine at `LIMIT 20`; catastrophic at `LIMIT 10000`. Two responses: select only the columns you need, and/or make the index **covering**.

### Probes

**Equality columns first, then the sort column.** Covered above, with the reasoning about why the sort is satisfied "for free" only when the preceding columns are pinned by equality.

**Selectivity.** Covered. The key correction to the common heuristic: selectivity determines how useful a column is as a *leading* column across the workload, not whether this particular query works. Measure it — `SELECT n_distinct, correlation FROM pg_stats WHERE tablename='orders'`.

**Covering index / index-only scan.** If the index contains every column the query touches, the engine never visits the table. In PostgreSQL:

```sql
CREATE INDEX idx_orders_covering
    ON orders (customer_id, status, created_at DESC)
    INCLUDE (total_amount, currency);
```

`INCLUDE` (PostgreSQL 11+) stores extra columns in the **leaf pages only** — they are payload, not part of the sortable key, so they do not bloat internal pages and cannot be used for seeking or ordering. The plan node becomes `Index Only Scan`.

Two PostgreSQL-specific caveats that separate a good answer from a great one:
- An index-only scan in PostgreSQL still consults the **visibility map** to determine whether a page's tuples are visible to all transactions. If the table has recently been updated and not vacuumed, the visibility map is unset for those pages and the engine must visit the heap anyway — you will see `Heap Fetches: N` in `EXPLAIN ANALYZE` with a non-zero N. So "index-only" depends on `VACUUM` keeping up. This is a real production surprise.
- SQL Server's equivalent is also `INCLUDE`. **MySQL/InnoDB has no `INCLUDE`** — you must add the columns to the key itself, which does bloat the index; but InnoDB secondary indexes always implicitly contain the primary key, so covering is cheap when the PK is what you need.

**Why `(status, customer_id)` may be worse.** Covered above — reusability, left-prefix, and selectivity of the leading column.

**Left-prefix rule.** Covered. The practical corollary: before adding an index, check whether an existing one already has your columns as a prefix. Redundant indexes (`(a)` when `(a, b)` exists) cost write throughput and space for nothing. Find them with `pg_stat_user_indexes` (`idx_scan = 0`) or MySQL's `sys.schema_unused_indexes`.

**`INCLUDE` columns.** Covered above, including the visibility-map caveat and the MySQL difference.

---

## Q48. How do you read an execution plan? What do you look for first?

### Answer

**Get the right plan first.** `EXPLAIN` shows the planner's *estimate*. `EXPLAIN (ANALYZE, BUFFERS)` actually **executes** the query and shows real timings and real row counts alongside the estimates. Always use `ANALYZE` when diagnosing — the gap between estimated and actual is usually the answer. Add `BUFFERS` to see shared block hits vs reads (cache vs disk) and `FORMAT JSON` if you want to feed it to a visualiser. Note that `EXPLAIN ANALYZE` on an `UPDATE`/`DELETE` really does modify data — wrap it in a transaction and roll back.

MySQL: `EXPLAIN ANALYZE` exists from 8.0.18; `EXPLAIN FORMAT=JSON` gives cost detail; before that you only had the estimate table.

**Reading order.** Plans are trees, printed with the root at the top. Execution flows **bottom-up and inside-out**: the most indented nodes run first, feeding their parents. Read from the deepest node upward.

Also understand that in PostgreSQL, **`cost` and `actual time` are cumulative** (a node's numbers include its children's) and are expressed as `startup..total`. A high startup cost means the node must consume all its input before producing a row (a sort, a hash build, an aggregate) — which is why `LIMIT` does not help those. And **`rows` is per-loop**: if a node shows `loops=500`, multiply `actual rows` and `actual time` by 500 to get the real totals. Missing that is the single most common misreading.

**What I look for, in order:**

**1. The biggest estimate-vs-actual discrepancy.** `rows=12` alongside `actual rows=840000` means the planner chose its entire strategy on a false premise. Causes: stale statistics (run `ANALYZE`), correlated columns the planner assumes are independent (fix with `CREATE STATISTICS ... (dependencies, ndistinct)` in PG 10+), a low `default_statistics_target` for a skewed column (raise it per-column with `ALTER TABLE ... ALTER COLUMN ... SET STATISTICS 1000`), expressions the planner cannot estimate (`WHERE lower(email) = ?` without a matching expression index), or opaque parameters. Fix the estimate and the plan usually fixes itself. **This is almost always the first thing to check.**

**2. Where the actual time is really spent.** Because costs are cumulative, the most expensive-looking node is often just the root. Subtract children to find the node that *adds* the time. Tools like explain.dalibo.com or pgMustard do this arithmetic for you and are worth using.

**3. Scan types.** Then join types. Then sorts and memory. Then loop counts.

**Scan types**

- **Seq Scan** — reads the whole table. Correct and *optimal* for small tables or when returning a large fraction of rows (say >5–20%); random I/O per row would be worse. A seq scan is only a problem when it is selective — filtering 5 million rows to return 3.
- **Index Scan** — walks the index, then fetches each matching row from the heap. Best when few rows match.
- **Index Only Scan** — the index covers all needed columns; check `Heap Fetches` (see Q47).
- **Bitmap Heap Scan** (+ `Bitmap Index Scan`) — a PostgreSQL middle ground: build a bitmap of matching pages from the index, then read the heap in *physical page order*, converting random I/O to sequential. Chosen when the row count is too high for a plain index scan but too low for a seq scan. Multiple bitmaps can be `BitmapAnd`/`BitmapOr`-ed, which is how PG combines several single-column indexes. Watch for `Recheck Cond` with `lossy=true` blocks — the bitmap overflowed `work_mem` and degraded to page granularity.

**Join types**

- **Nested Loop** — for each outer row, probe the inner side. Excellent when the outer side is tiny and the inner has an index on the join key. **Disastrous** when the outer row estimate is wrong: `loops=500000` against an unindexed inner side is the classic "query that used to be fast and now takes 40 minutes".
- **Hash Join** — build a hash table from the smaller side, probe with the larger. The workhorse for large equi-joins. Watch for `Batches: > 1` in `EXPLAIN ANALYZE`, which means the hash table exceeded `work_mem` and spilled to disk.
- **Merge Join** — both inputs sorted on the join key, then merged. Good when inputs are already sorted (index order) or very large. If it required explicit sorts, a hash join is often cheaper.

Non-equi joins (`<`, `BETWEEN`, `LIKE`) can only use nested loops or a merge join, never a hash join — which is why range joins are frequently slow.

**Sorts and memory**

`Sort Method: quicksort Memory: 25kB` is fine. `Sort Method: external merge Disk: 102400kB` means it spilled — raise `work_mem` (per-operation, per-connection; be careful, a query with 5 sort nodes and 100 connections can use 500× `work_mem`), or provide an index that supplies the order, or reduce the row count before sorting. The same applies to `HashAggregate` spilling (PG 13+ reports `Disk Usage`).

**Other red flags:** `Rows Removed by Filter: 4900000` (the index isn't selective enough or a predicate isn't indexable); a `Materialize` node under a nested loop with huge loop counts; `SubPlan` executed once per row (a correlated subquery that didn't get flattened); `Parallel` workers `Workers Launched: 0` when you expected parallelism; and `Filter` conditions that should have been `Index Cond` (meaning the predicate was applied after the fetch, not during the seek — often caused by wrapping a column in a function, or by a type mismatch such as comparing `bigint` to a `numeric` literal).

**Why "the most expensive node is not the root cause"**

Two reasons. First, cumulative costs mean the root always looks worst. Second, an expensive node is often the *symptom* of a bad estimate three levels down — the sort is slow because it received 800,000 rows instead of the 12 predicted, and the fix is upstream. Always trace back to the estimate error.

### Probes

**`EXPLAIN ANALYZE`.** Covered, including `BUFFERS`, the fact that it executes, and MySQL availability.

**Estimated vs actual rows (stats staleness).** Covered, with the specific remedies: `ANALYZE`, extended statistics for correlated columns, per-column statistics targets, expression indexes, and autovacuum/autoanalyze tuning (`autovacuum_analyze_scale_factor` is 10% by default, which is far too lax for a large table — a billion-row table gets analysed after 100 million changes).

**Seq scan vs index scan vs bitmap heap scan.** Covered, including when a seq scan is correct.

**Nested loop vs hash vs merge join.** Covered, including which are possible for non-equi joins.

**Sorts spilling to disk.** Covered, including the `work_mem` multiplication hazard.

**The most expensive node ≠ the root cause.** Covered.

---

## Q49. Explain the four isolation levels via the anomalies they prevent, and name an anomaly `SERIALIZABLE` prevents but `REPEATABLE READ` doesn't.

### Answer

**The ANSI SQL levels and the anomalies they exclude:**

| Level | Dirty read | Non-repeatable read | Phantom read |
|---|---|---|---|
| READ UNCOMMITTED | possible | possible | possible |
| READ COMMITTED | prevented | possible | possible |
| REPEATABLE READ | prevented | prevented | possible (per ANSI) |
| SERIALIZABLE | prevented | prevented | prevented |

**The anomalies:**

- **Dirty read** — you read a row another transaction has written but not committed. If it rolls back, you acted on data that never existed.
- **Non-repeatable read** — you read a row, another transaction commits an `UPDATE` to it, you read it again and get a different value. Breaks any logic that reads the same row twice.
- **Phantom read** — you run a *range* query, another transaction commits an `INSERT` (or `UPDATE` moving a row into the range), you re-run the query and new rows appear. Distinct from non-repeatable read because it concerns the *set membership* of a predicate, not the value of a known row.

**The anomaly SERIALIZABLE prevents but REPEATABLE READ does not: write skew.**

Two transactions read an overlapping set, each checks an invariant that holds, and each writes a *different* row. Neither write conflicts, both commit, and the invariant is now violated — a state no serial ordering could have produced.

The canonical example — an on-call roster requiring at least one doctor on duty:

```sql
-- T1                                     -- T2
BEGIN;                                    BEGIN;
SELECT count(*) FROM duty                 SELECT count(*) FROM duty
  WHERE on_call = true;  -- returns 2       WHERE on_call = true;  -- returns 2
-- "2 > 1, safe for me to leave"          -- "2 > 1, safe for me to leave"
UPDATE duty SET on_call = false           UPDATE duty SET on_call = false
  WHERE doctor = 'alice';                   WHERE doctor = 'bob';
COMMIT;                                   COMMIT;
-- Now zero doctors are on call.
```

Under snapshot isolation / REPEATABLE READ both commit: they updated different rows, so there is no write-write conflict, and each read a consistent snapshot. Under SERIALIZABLE, one is aborted.

Other write-skew instances: double-booking a meeting room (each transaction checks for overlapping bookings and inserts a non-overlapping-looking one), allowing an account balance to go negative across two accounts with a joint overdraft limit, and allocating the last item of stock from two warehouses.

**Snapshot isolation is not REPEATABLE READ — the vendor reality**

This is where the ANSI table stops describing real databases, and knowing it is the mark of experience.

**PostgreSQL** implements isolation with **MVCC**: every row version carries `xmin`/`xmax` transaction IDs, and each transaction reads according to a snapshot of which transactions were committed when it started (or when each statement started, at READ COMMITTED). Consequences:
- `READ UNCOMMITTED` **does not exist** — requesting it silently gives you READ COMMITTED. Dirty reads are impossible by construction.
- `REPEATABLE READ` in PostgreSQL is genuine **snapshot isolation**: one snapshot taken at the first statement, held for the whole transaction. It therefore **prevents phantom reads**, which ANSI does not require. But it permits write skew. It also introduces a new failure: `ERROR: could not serialize access due to concurrent update` if you try to update a row that changed after your snapshot — so **you must be prepared to retry the entire transaction**.
- `SERIALIZABLE` uses **SSI (Serializable Snapshot Isolation)**: snapshot isolation plus tracking of read/write dependencies (via predicate locks, `SIReadLock`, which do not block anything — they are bookkeeping). If it detects a dangerous structure of rw-dependencies that could produce a non-serialisable outcome, it aborts one transaction with `ERROR: could not serialize access due to read/write dependencies among transactions` (SQLSTATE `40001`). It is optimistic: no extra blocking, but a mandatory retry loop and some false positives.

**MySQL/InnoDB** uses MVCC plus locking:
- `REPEATABLE READ` is the **default** (unlike PostgreSQL's READ COMMITTED). Plain `SELECT`s are consistent non-locking reads from a snapshot, so they don't see phantoms.
- **Locking** reads (`SELECT ... FOR UPDATE`, `FOR SHARE`) and writes at RR use **gap locks** and **next-key locks** (a record lock plus the gap before it) to prevent another transaction from inserting into the scanned range — so phantoms are prevented for locking reads too. Gap locks are also a leading cause of InnoDB deadlocks and of surprising lock waits on rows that "don't exist yet".
- InnoDB's `REPEATABLE READ` still permits write skew, and it has a famous quirk: a plain `SELECT` reads the snapshot while an `UPDATE ... WHERE` in the same transaction reads the *latest committed* version, so you can update rows you cannot see.

**Oracle** offers only READ COMMITTED and SERIALIZABLE, and its SERIALIZABLE is actually snapshot isolation — so Oracle "SERIALIZABLE" permits write skew. SQL Server defaults to READ COMMITTED with locking (readers block writers) unless `READ_COMMITTED_SNAPSHOT` is enabled, and offers `SNAPSHOT` as a separate level.

**Practical guidance:** default to READ COMMITTED. When you need a multi-statement invariant, do not reach for a higher isolation level first — reach for an explicit lock (`SELECT ... FOR UPDATE`) or a database constraint (`UNIQUE`, `EXCLUDE`, `CHECK`), which are cheaper and deterministic. Use SERIALIZABLE when the invariant genuinely spans a predicate that no constraint can express, and only if you have a retry loop.

### Probes

**Dirty read / non-repeatable read / phantom / write skew.** All defined above with examples.

**MVCC snapshots.** Covered — `xmin`/`xmax`, statement-level vs transaction-level snapshots, and the fact that MVCC readers never block writers (and vice versa) in PostgreSQL, which is the fundamental architectural difference from lock-based SQL Server defaults.

**PostgreSQL RR ≠ ANSI RR.** Covered — PG's RR is snapshot isolation, which is strictly stronger than ANSI RR (no phantoms) but still permits write skew, and adds serialisation failures on update conflicts.

**SSI and serialisation failures needing retry.** Covered. The application-level requirement: catch SQLSTATE `40001` (and `40P01` for deadlocks) and retry the whole transaction with jittered backoff, capped at a few attempts. In Spring this means catching `CannotAcquireLockException` / `ConcurrencyFailureException` — note the retry must wrap the transaction *from outside*, so a `@Retryable` on the `@Transactional` method does not work (the transaction has already rolled back and the proxy ordering is wrong); put the retry on an outer, non-transactional method.

**MySQL InnoDB gap locks.** Covered — what they are, why they exist at RR, and that they cause deadlocks and locks on non-existent rows.

---

## Q50. What is a deadlock at the DB level, how do you reproduce one, and how do you prevent them?

### Answer

**Definition.** Two or more transactions each hold a lock the other needs, forming a cycle in the wait-for graph. Neither can proceed. Unlike an application deadlock, the database **detects** this: PostgreSQL runs deadlock detection after `deadlock_timeout` (default 1s) of waiting; InnoDB maintains a wait-for graph and detects almost immediately. The engine then chooses a victim and aborts it — PostgreSQL raises SQLSTATE `40P01` (`deadlock detected`), InnoDB error 1213. The survivor proceeds.

Important distinction: a deadlock is **not** the same as a lock wait/timeout. A lock wait resolves on its own when the holder commits; a deadlock never resolves and must be broken.

**Reproducing one — two psql sessions:**

```sql
-- Session A                              -- Session B
BEGIN;                                    BEGIN;
UPDATE accounts SET bal = bal - 100
  WHERE id = 1;                           UPDATE accounts SET bal = bal - 50
                                            WHERE id = 2;
-- A now holds a row lock on id=1         -- B holds a row lock on id=2
UPDATE accounts SET bal = bal + 100
  WHERE id = 2;   -- BLOCKS on B
                                          UPDATE accounts SET bal = bal + 50
                                            WHERE id = 1;   -- BLOCKS on A → cycle
-- one session gets: ERROR: deadlock detected
```

The essential ingredient is **opposite lock acquisition order**. A transfers 1→2 while B transfers 2→1.

**Prevention**

**1. Consistent lock ordering.** The primary fix. Any transaction touching multiple rows must acquire them in a globally agreed order — sort by primary key:

```java
List<Long> ids = Stream.of(fromId, toId).sorted().toList();
accountRepo.lockAll(ids);   // SELECT ... WHERE id IN (:ids) ORDER BY id FOR UPDATE
```

The same applies across *tables*: if some code paths write `orders` then `order_items` and others reverse it, you have a latent deadlock. Establish and document a table ordering.

Note that `ORDER BY` inside `SELECT ... FOR UPDATE` does deterministically order lock acquisition in PostgreSQL and InnoDB in practice, but a bulk `UPDATE ... WHERE id IN (...)` does **not** guarantee row order — the plan decides. If order matters, lock explicitly first.

**2. Shorter transactions.** Locks are held until commit. A transaction that locks a row and then makes an HTTP call holds that lock for the network round trip, multiplying the collision window. Do external I/O outside the transaction (see Q28).

**3. Lower isolation where safe.** At MySQL RR, gap locks lock ranges, causing deadlocks between inserts that don't overlap on any actual row. Switching to READ COMMITTED removes most gap locking and is a well-known InnoDB deadlock remedy (it also changes replication requirements — needs row-based binlog).

**4. Fewer, coarser lock acquisitions.** A single atomic statement takes and releases locks in one planned order:
```sql
UPDATE stock SET qty = qty - 1 WHERE sku = ? AND qty >= 1;
```
beats read-then-write for both correctness and deadlock avoidance.

**5. Indexes.** Under InnoDB, an `UPDATE ... WHERE unindexed_col = ?` must scan and lock **every row it examines**, not just those it matches. Adding the index converts a table-wide lock storm into a single row lock. A missing index is a frequent, non-obvious deadlock cause.

**6. `SKIP LOCKED` for queue tables.** The classic "job queue in a database" deadlocks or serialises when many workers contend for the head:
```sql
SELECT id FROM jobs
 WHERE status = 'PENDING'
 ORDER BY created_at
 FOR UPDATE SKIP LOCKED
 LIMIT 10;
```
`SKIP LOCKED` (PostgreSQL 9.5+, MySQL 8+, Oracle) makes each worker silently ignore rows already locked by another, so N workers take N disjoint batches with no contention and no deadlock. `NOWAIT` is the alternative when you want an immediate error instead. This is the correct answer for polling-based work distribution, and it is what most "database as a queue" libraries use.

**7. Retry.** Deadlocks are a *normal* concurrency outcome, not a bug to be eliminated entirely. Any write path must tolerate one:
```java
@Retryable(retryFor = {CannotAcquireLockException.class, DeadlockLoserDataAccessException.class},
           maxAttempts = 3, backoff = @Backoff(delay = 50, multiplier = 2, random = true))
public void transfer(...) { transferTx.execute(...); }   // retry OUTSIDE the transaction
```
Jitter matters: without it, both victims retry simultaneously and collide again.

**Diagnosis**

- PostgreSQL logs the full deadlock report by default, including both statements and the process IDs. Set `log_lock_waits = on` and lower `deadlock_timeout` awareness to also see long waits before they become deadlocks. Inspect live blocking with `pg_locks` joined to `pg_stat_activity` (`SELECT pg_blocking_pids(pid)`).
- MySQL: `SHOW ENGINE INNODB STATUS` shows the **latest** deadlock only; enable `innodb_print_all_deadlocks = ON` to get every one in the error log. `performance_schema.data_locks` / `data_lock_waits` for live state.
- Application side: log the transaction's statements with a correlation ID so the two halves of the cycle can be reconstructed.

### Probes

**Two transactions acquiring row locks in opposite order.** Covered with a reproducible script.

**Consistent lock ordering.** Covered, including the cross-table dimension and the caveat about `UPDATE ... IN (...)` not guaranteeing order.

**Shorter transactions.** Covered.

**Lower isolation where safe.** Covered, with the InnoDB gap-lock rationale and the binlog caveat.

**`SKIP LOCKED` for queue tables.** Covered.

**Deadlock logs.** Covered for both PostgreSQL and MySQL, including the "only the latest" InnoDB default.

**Retry with backoff.** Covered, including the critical structural point that the retry must wrap the transaction from outside, and that jitter is required.

---

## Q51. Offset pagination is timing out on page 5000. Fix it.

### Answer

**Why it degrades.** `LIMIT 20 OFFSET 100000` does not skip cheaply. The database must **produce and discard** the first 100,000 rows — reading them, applying the sort, then throwing them away. Cost is O(offset + limit), so page 5000 is 5000× more expensive than page 1. There is no index that fixes this, because the offset is a positional concept the index cannot evaluate.

Worse, it is also **incorrect under concurrency**: if a row is inserted before your position between requests, one row shifts across the page boundary and the user sees a duplicate; if a row is deleted, the user silently skips one.

**The fix: keyset (seek) pagination.** Instead of "skip N rows", say "give me rows after this specific point":

```sql
-- Page 1
SELECT id, created_at, title
  FROM articles
 ORDER BY created_at DESC, id DESC
 LIMIT 20;

-- Subsequent pages: pass the last row's sort key back as the cursor
SELECT id, created_at, title
  FROM articles
 WHERE (created_at, id) < (:last_created_at, :last_id)   -- row-value comparison
 ORDER BY created_at DESC, id DESC
 LIMIT 20;
```

With an index on `(created_at DESC, id DESC)` this is an index seek directly to the position followed by a 20-row scan. **Cost is O(limit), constant for every page.** Page 5000 is exactly as fast as page 1.

**The row-value syntax matters.** `(a, b) < (:a, :b)` is standard SQL row-value comparison, supported by PostgreSQL and MySQL 8, and it lets the planner use the composite index directly. The naive expansion people write instead —
`WHERE created_at < :a OR (created_at = :a AND id < :b)` — is logically equivalent but frequently produces a worse plan because the `OR` defeats a clean index range scan. Use the tuple form where available.

**The stable sort key and tiebreaker**

The cursor must identify a **unique, total** ordering. `created_at` alone is not unique — two articles created in the same millisecond means the comparison is ambiguous and you will skip or duplicate rows at the boundary. Always append a unique tiebreaker (the primary key) to both the `ORDER BY` and the `WHERE`, and to the index. The sort key must also be **immutable**, or a row that changes value moves position between requests. `created_at` and `id` are good; `updated_at`, `score`, and `price` are bad unless combined with the PK *and* you accept that mutating rows can be missed.

**Cursor encoding**

Don't expose raw column values — it leaks schema and invites tampering. Base64url-encode an opaque payload, and consider signing it:

```json
{"v":1,"created_at":"2026-03-01T12:00:00Z","id":98213,"sort":"created_desc"}
```

Return it as `nextCursor` in the response (or as a `Link: <...>; rel="next"` header). Include the sort specification in the cursor so a client cannot mix a cursor from one sort order with a different one. Validate and version it, and return `400` on a malformed or expired cursor.

**`COUNT(*)` for total pages is also expensive**

Almost everyone forgets this: even with perfect keyset pagination, `SELECT COUNT(*) FROM articles WHERE <same filters>` is a full scan of the matching set on every request. On a large table this dominates the response time. Options:

- **Drop the total.** "Load more" / infinite scroll needs only `hasNext`. Get it by fetching `LIMIT 21` and returning 20 — if you got 21, there is a next page. Zero extra cost.
- **Approximate it.** `reltuples` from `pg_class` for an unfiltered count, or `EXPLAIN`-derived row estimates for a filtered one. Show "about 12,000 results".
- **Cache it** per filter combination with a short TTL.
- **Maintain it** in a counter table/materialised view if it must be exact and is read constantly.

**The trade-off: no random page access.** Keyset pagination cannot jump to "page 500" — there is no cursor for a page you have not visited. You get first/next/previous (previous requires reversing the comparison and the sort, then reversing the result set in the application) but not a numbered page control.

This is almost always acceptable, and worth arguing in an interview: real users do not navigate to page 5000. If page 5000 is being requested, it is a bot, a scraper, or a batch export — and the right answer for those is a different endpoint (a keyset-based cursor export or a bulk data API), not a faster `OFFSET`. If a numbered UI is a hard product requirement, hybrid approaches exist: keyset for the common path, and cap `OFFSET` at a few hundred pages with a "refine your search" message.

**Middle grounds if you cannot change the API:** if the sort is on the primary key and there are no gaps, `WHERE id > :n` is exact. If the filter is highly selective, a covering index makes the discarded rows cheap enough to scan (index-only), which can buy an order of magnitude without changing the contract. And deferred joins help when `SELECT *` is the cost: `SELECT * FROM articles JOIN (SELECT id FROM articles ORDER BY ... LIMIT 20 OFFSET 100000) t USING (id)` — the offset scan then happens on a narrow covering index rather than on full rows.

### Probes

**Keyset/seek pagination.** Covered, with the row-value comparison and the index it requires.

**Stable sort key + tiebreaker.** Covered, including immutability and why a non-unique sort key corrupts the boundary.

**Why `COUNT(*)` for total pages is also expensive.** Covered, with four concrete alternatives.

**Cursor encoding.** Covered — opaque, versioned, sort-bound, validated.

**Trade-off: no random page access.** Covered, with the argument for why that is usually the correct product decision and what to do when it isn't.

---

## Q52. Return, per customer, their most recent order — three ways, compared.

### Answer

Setup: `orders(id, customer_id, created_at, total)`, and an index on `(customer_id, created_at DESC)`.

**1. Correlated subquery**

```sql
SELECT o.*
  FROM orders o
 WHERE o.created_at = (SELECT MAX(o2.created_at)
                         FROM orders o2
                        WHERE o2.customer_id = o.customer_id);
```

*Behaviour:* readable and portable to every database. But it **returns duplicates** when a customer has two orders sharing the maximum timestamp — a correctness bug, not just a performance one. Modern planners often rewrite it into a join or a hash aggregate; older ones execute the subquery per row.

A variant that avoids the tie problem is the anti-join formulation, which is often the fastest portable option:

```sql
SELECT o.*
  FROM orders o
  LEFT JOIN orders o2
    ON o2.customer_id = o.customer_id
   AND (o2.created_at, o2.id) > (o.created_at, o.id)
 WHERE o2.id IS NULL;
```

**2. Window function**

```sql
SELECT id, customer_id, created_at, total
  FROM (SELECT o.*,
               ROW_NUMBER() OVER (PARTITION BY customer_id
                                  ORDER BY created_at DESC, id DESC) AS rn
          FROM orders o) t
 WHERE rn = 1;
```

*Behaviour:* standard SQL, works on PostgreSQL, MySQL 8+, SQL Server, Oracle. Unambiguous — `ROW_NUMBER` breaks ties deterministically given a total ordering. Trivially generalises to "top 3 per customer" (`rn <= 3`), which the other approaches do not.

*Cost:* the window function must process **every row** — typically a `Sort` (or an index scan supplying the order) followed by `WindowAgg`, then a filter discarding all but the first per partition. If the index provides `(customer_id, created_at DESC)` order, no explicit sort is needed and this is a single pass over the index. If not, it sorts the whole table.

Note `RANK()` and `DENSE_RANK()` behave differently on ties (both return all tied rows); use `ROW_NUMBER()` when you want exactly one.

**3. `DISTINCT ON` (PostgreSQL-specific)**

```sql
SELECT DISTINCT ON (customer_id) *
  FROM orders
 ORDER BY customer_id, created_at DESC, id DESC;
```

*Behaviour:* the most concise and usually the fastest single-statement form in PostgreSQL. The `ORDER BY` must lead with the `DISTINCT ON` expression. Non-portable.

**4. `LATERAL` join — the one that wins at scale**

```sql
SELECT c.id, o.*
  FROM customers c
  CROSS JOIN LATERAL (
        SELECT * FROM orders o
         WHERE o.customer_id = c.id
         ORDER BY o.created_at DESC, o.id DESC
         LIMIT 1) o;
```

(Use `LEFT JOIN LATERAL ... ON true` to keep customers with no orders.) MySQL 8.0.14+ supports `LATERAL`; SQL Server has `CROSS APPLY`/`OUTER APPLY`; Oracle 12c+ supports both.

**Plan differences at scale — the key comparison**

The decisive factor is the ratio of **customers to orders**.

- **Many orders per customer** (say 10,000 customers, 50 million orders): window function and `DISTINCT ON` must read and sort **all 50 million rows** to discard 49,990,000 of them. `LATERAL` performs 10,000 index seeks, each returning exactly one row: ~10,000 index lookups total. **Orders of magnitude faster.** This is the "loose index scan" / "skip scan" problem, and `LATERAL` over a driving table is the standard PostgreSQL solution (as is the recursive-CTE skip-scan trick when there is no customers table to drive from).
- **Few orders per customer** (5 million customers, 8 million orders): the window function's single sequential pass is competitive or better, because 5 million individual index seeks cost more in random I/O than one sorted scan. `DISTINCT ON` typically wins here in PostgreSQL.
- **Correlated subquery:** unpredictable. Fine when the planner rewrites it, terrible when it doesn't. And it has the tie bug.

**How to decide:** run all of them with `EXPLAIN (ANALYZE, BUFFERS)` against production-shaped data volumes. This is not a question you answer from first principles — the point of the interview question is whether you know that the answer changes with data distribution and that you must measure. The honest senior answer: "`DISTINCT ON` or the window function for readability; `LATERAL` when the driving table is small relative to the detail table; and I'd check the plan on real cardinalities before committing."

### Probes

**Correlated subquery, `ROW_NUMBER() OVER (PARTITION BY ...)`, `DISTINCT ON`, `LATERAL` join.** All four given, with portability noted for each.

**Plan differences at scale.** Covered — the customer:order ratio is the deciding variable, with the reasoning for each regime, plus the tie-handling correctness difference between `ROW_NUMBER`, `RANK`, and the naive `MAX` subquery.

---

## Q53. Zero-downtime column rename under a rolling deployment.

### Answer

**Why the naive `ALTER TABLE ... RENAME COLUMN` fails.** During a rolling deployment, version N and version N+1 run **simultaneously** against **one** database. A rename is instantaneous and atomic; the deployment is not. The moment you rename, every still-running instance of version N issues SQL referencing the old name and fails. You have an outage for the duration of the rollout — and if you must roll back, version N+1's code now fails against the old name.

The rule: **during a rolling deployment, the schema must be compatible with both the old and the new application version simultaneously.** Every migration must therefore be either backwards-compatible or split into stages.

**Expand / contract (parallel change), in six deployments**

Take renaming `orders.cust_ref` → `orders.customer_reference`:

**Stage 1 — Expand (migration only).** Add the new column, nullable, no default that rewrites the table.
```sql
ALTER TABLE orders ADD COLUMN customer_reference varchar(64);
```
Safe: old code ignores it. Note that adding a nullable column with no default is metadata-only (instant) in PostgreSQL 11+ and MySQL 8 (`ALGORITHM=INSTANT`); adding a `NOT NULL` column *with* a default is instant in PostgreSQL 11+ but historically rewrote the whole table — verify for your version.

**Stage 2 — Dual write (code deployment).** Version N+1 writes **both** columns and reads the **old** one. Now every new or updated row has both populated. Old instances still work.

**Stage 3 — Backfill (batched job).** Copy historical data in bounded batches, committing each, so you never hold a long transaction or lock the table:
```sql
UPDATE orders SET customer_reference = cust_ref
 WHERE customer_reference IS NULL AND id BETWEEN ? AND ?;
```
Throttle it, monitor replication lag, and make it resumable and idempotent. A single `UPDATE` over 200 million rows will bloat the table, blow up WAL, block autovacuum, and lag replicas.

**Stage 4 — Read switch (code deployment).** Version N+2 reads the **new** column, still writes both. Verify with a reconciliation query that the columns agree (`SELECT count(*) FROM orders WHERE cust_ref IS DISTINCT FROM customer_reference`). This deployment is safely reversible: rolling back to N+1 still works, because both columns are current.

**Stage 5 — Stop writing the old column (code deployment).** Version N+3 uses only the new column. Now you can add the constraint you wanted:
```sql
ALTER TABLE orders ADD CONSTRAINT ck_cr_nn CHECK (customer_reference IS NOT NULL) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT ck_cr_nn;   -- takes only SHARE UPDATE EXCLUSIVE
```
The `NOT VALID` + `VALIDATE` two-step avoids the long `ACCESS EXCLUSIVE` lock that a direct `SET NOT NULL` full-table verification takes. (PostgreSQL 12+ can also use a validated `CHECK` to make `SET NOT NULL` cheap.)

**Stage 6 — Contract (migration only), after a soak period.** Drop the old column. Wait long enough that you are certain you will not roll back — typically at least one release cycle.
```sql
ALTER TABLE orders DROP COLUMN cust_ref;
```

**Locking behaviour — the part that causes real outages**

PostgreSQL DDL takes `ACCESS EXCLUSIVE` locks, which conflict with *everything* including `SELECT`. Two compounding hazards:
- A DDL statement waiting behind a long-running query **queues**, and every subsequent query queues behind *it*. One `ALTER TABLE` blocked by a 10-minute analytics query stalls the entire table for 10 minutes. **Always** set `SET lock_timeout = '3s'` before DDL so it fails fast and can be retried, rather than causing a pile-up.
- Index creation must be `CREATE INDEX CONCURRENTLY` (and `DROP INDEX CONCURRENTLY`), which does not block writes — at the cost of two table passes, an inability to run inside a transaction, and the possibility of leaving an `INVALID` index behind on failure that must be dropped and retried. Since it cannot run in a transaction, Flyway needs the migration marked as non-transactional (a `-- flyway:executeInTransaction=false` script or a separate callback).

MySQL 8 has online DDL with `ALGORITHM=INPLACE, LOCK=NONE`; verify per-operation support, and for the operations it doesn't support use `pt-online-schema-change` or `gh-ost`, which build a shadow table and swap it.

**Flyway/Liquibase ordering**

- Migrations are **versioned and immutable** — never edit an applied migration; Flyway's checksum validation will fail (and if you disable that, environments silently diverge). Add a new migration instead.
- Order is by version number, applied inside a transaction where the database supports transactional DDL (PostgreSQL does; **MySQL and Oracle do not**, so a failed multi-statement migration on MySQL leaves a half-applied state requiring manual repair — keep MySQL migrations to one statement each).
- Flyway takes an advisory lock, so concurrent instances starting simultaneously in Kubernetes serialise rather than racing. This still means all pods wait for the migration; for long migrations, run them as a separate **Kubernetes Job / init step in the pipeline** before the rollout rather than at application startup, so a slow migration doesn't trip the pods' startup probes and cause a `CrashLoopBackOff`.
- **Never let the application's runtime DB user own DDL.** Use a separate migration user with DDL rights and give the app runtime user only DML. Never use `hibernate.ddl-auto` beyond `validate` in any deployed environment.

**Long-running migrations vs the deployment timeout.** If a backfill takes hours, it cannot be a blocking migration. Structure it as: fast schema migration in the pipeline, then a separate idempotent, resumable, throttled backfill job that runs independently and is monitored, with the read-switch deployment gated on its completion.

### Probes

**Expand/contract (add → dual-write → backfill → read-switch → drop).** Covered as six explicit stages with the deployment/migration boundary marked for each.

**Backwards/forwards compatibility across two running versions.** Covered — stated as the governing rule, with rollback safety called out at each stage.

**Flyway/Liquibase ordering.** Covered — immutability and checksums, transactional DDL differences by vendor, the advisory lock, running migrations as a pipeline step rather than at startup, and DDL/DML user separation.

**Locking behaviour of `ALTER TABLE`.** Covered — `ACCESS EXCLUSIVE`, the queue-behind-a-long-query pile-up, `lock_timeout`, `CREATE INDEX CONCURRENTLY`, `NOT VALID` + `VALIDATE`, and MySQL's online DDL / `gh-ost`.

**Long-running migrations vs the deployment timeout.** Covered.

---

## Q54. Clustered vs covering vs partial indexes. When do indexes hurt?

### Answer

**Clustered index.** The index whose leaf level *is* the table — rows are stored in index key order. There can be only one per table.
- **SQL Server / MySQL InnoDB:** every table has one. In InnoDB it is the primary key (or the first `UNIQUE NOT NULL` column, or a hidden 6-byte rowid). Consequences: PK lookups are single-seek (no extra fetch), range scans on the PK are sequential, and **every secondary index stores the PK as its row pointer** — so a wide PK (a `UUID` as `char(36)`, or a composite natural key) bloats every secondary index. And because rows are physically ordered, a random PK (UUIDv4) causes page splits and fragmentation on insert, while a monotonic PK causes append-only inserts but can create a "hot" last page under high concurrency.
- **PostgreSQL has no clustered index.** It uses heap storage with all indexes secondary. The `CLUSTER` command reorders a table by an index **once**; new writes are not maintained in that order, so it decays. PostgreSQL's `correlation` statistic measures how well physical order matches an index's order, and drives the planner's estimate of random vs sequential I/O.

**Covering index.** An index that contains every column a given query references, so the query is answered from the index alone — an *index-only scan*, avoiding the table fetch entirely. Built either by adding columns to the key or (PostgreSQL 11+ / SQL Server) via `INCLUDE`, which stores them only in leaf pages. See Q47 for the full treatment and the PostgreSQL visibility-map caveat.

**Partial (filtered) index.** An index over a subset of rows:

```sql
CREATE INDEX idx_orders_pending
    ON orders (created_at)
 WHERE status = 'PENDING';
```

PostgreSQL calls it partial; SQL Server calls it filtered; **MySQL does not support it** (the usual workaround is a generated column plus an index, or a functional index in 8.0.13+).

Why it is powerful: if 0.1% of a 500-million-row `orders` table is `PENDING`, this index has 500,000 entries instead of 500 million. It is small enough to stay entirely in memory, it is cheap to maintain (rows that are not `PENDING` are never touched), and when an order transitions out of `PENDING` its entry is *removed*, keeping the index permanently small. This is the ideal shape for a work-queue or "unprocessed items" pattern — precisely the case where a full index is mostly dead weight.

Two constraints: the planner will only use it when it can **prove** the query's `WHERE` implies the index predicate (so the query must include `status = 'PENDING'` literally, or something the planner can deduce it from — a bound parameter usually works, but an `IN` list or an `OR` may not); and a partial **unique** index is a very useful trick for "only one active row per key":

```sql
CREATE UNIQUE INDEX uq_one_active_sub ON subscriptions (user_id) WHERE status = 'ACTIVE';
```

Related: **expression/functional indexes** (`CREATE INDEX ON users (lower(email))`) which are required for case-insensitive lookups to be indexable, and **partitioned tables**, which are the right answer when the pruning dimension is large-scale (time-series retention, per-tenant isolation) — partitioning gives you cheap bulk deletion via `DROP PARTITION`, which no index can.

**When indexes hurt**

1. **Write amplification.** Every `INSERT` and `DELETE` updates *every* index on the table; an `UPDATE` updates every index whose columns changed. Ten indexes means an insert does eleven B-tree modifications. On a write-heavy table this dominates. (PostgreSQL's HOT — heap-only tuple — optimisation avoids index updates when no indexed column changed *and* the new row version fits on the same page; leaving `fillfactor` headroom on update-heavy tables preserves that. This is a genuine PostgreSQL-specific tuning lever.)
2. **Bloat and fragmentation.** PostgreSQL's MVCC means an `UPDATE` writes a new tuple and leaves the old one dead; indexes accumulate dead entries until vacuumed. Index bloat degrades scans and inflates memory. `REINDEX CONCURRENTLY` (PG 12+) rebuilds without blocking. InnoDB indexes fragment via page splits, remedied by `OPTIMIZE TABLE` (which rebuilds).
3. **Memory pressure.** Indexes compete with data for the buffer pool / shared buffers. A large unused index evicts hot data pages, so an index nobody queries actively slows down queries that would otherwise be cached.
4. **Planner confusion.** More indexes means a larger plan search space and more opportunities to pick a worse plan from bad estimates.
5. **Low-cardinality columns.** An index on `status` with 5 distinct values across 50 million rows has no useful selectivity — matching 20% of the table, the planner correctly prefers a seq scan, and the index is pure cost. (Exception: it may still help as part of a composite, and it is exactly the case where a **partial** index is right instead.)
6. **Lock and DDL cost.** Every index makes `CREATE INDEX`, restores, `pg_dump`/reload, and major-version upgrades slower.
7. **Insert hot spots.** A monotonically increasing index key (a timestamp, a sequence) concentrates all inserts on the rightmost leaf page, which becomes a contention point on high-throughput systems.

**Finding unused indexes:**
```sql
SELECT relname, indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
  FROM pg_stat_user_indexes
 WHERE idx_scan = 0
 ORDER BY pg_relation_size(indexrelid) DESC;
```
Caveats before dropping: `idx_scan` is cumulative since the last stats reset, so check `pg_stat_reset` timing; the index may serve a monthly report or a replica (statistics are per-node, so check replicas too); and a unique index may exist to enforce a constraint rather than to serve queries. MySQL: `sys.schema_unused_indexes` and `sys.schema_redundant_indexes`.

**`IS NULL` handling.** Worth knowing because it differs by vendor:
- **PostgreSQL B-tree indexes do store NULLs**, so `WHERE col IS NULL` is indexable, and `NULLS FIRST/LAST` ordering can be indexed.
- **Oracle B-tree indexes do not store rows where all indexed columns are NULL**, so `IS NULL` cannot use a single-column index — a classic Oracle gotcha, worked around with a composite index or a function-based index on `NVL(col, ...)`.
- For "sparse" columns that are mostly NULL, a **partial index** `WHERE col IS NOT NULL` gives you a tiny index covering exactly the interesting rows.
- Also remember `NULL` semantics: `col = NULL` is never true (use `IS NULL`), `NOT IN (subquery containing NULL)` returns no rows (a very common bug — use `NOT EXISTS`), and `UNIQUE` constraints in standard SQL permit multiple NULLs (PostgreSQL 15 added `UNIQUE NULLS NOT DISTINCT` to change this).

### Probes

**Clustered / covering / partial distinction.** Covered, with the PostgreSQL-vs-InnoDB architectural difference made explicit.

**Write amplification.** Covered, including HOT updates and `fillfactor`.

**Bloat.** Covered, with `REINDEX CONCURRENTLY` and `OPTIMIZE TABLE`.

**Index maintenance on updates.** Covered — only indexes on changed columns are updated, and the HOT exception.

**Unused index detection.** Covered, with the caveats that make a naive drop dangerous.

**Low-cardinality columns.** Covered, including why a partial index is usually the right response.

**`IS NULL` handling.** Covered, with the PostgreSQL/Oracle divergence and the `NOT IN`/NULL trap.

---

# 7. REST & API Design

---

## Q55. Design the API for a long-running operation (e.g. "generate report"). Show the request/response cycle.

### Answer

**The principle:** an HTTP request should not stay open for minutes. Long-held connections break through load balancers (idle timeouts), proxies, and mobile networks; they consume server resources; they cannot be retried safely; and the client has no visibility into progress. The standard solution is to make the *operation itself* a resource.

**The cycle**

```http
POST /reports HTTP/1.1
Content-Type: application/json
Idempotency-Key: 7c9e6679-7425-40de-944b-e07fc1f90ae7

{ "type": "SALES_SUMMARY", "from": "2026-01-01", "to": "2026-06-30", "format": "csv" }
```

```http
HTTP/1.1 202 Accepted
Location: /reports/1f4b2a
Retry-After: 5
Content-Type: application/json

{ "id": "1f4b2a", "status": "PENDING", "createdAt": "2026-08-13T09:00:00Z" }
```

`202 Accepted` means "I have accepted this for processing; it is not done". `Location` points at the **status resource**, and `Retry-After` tells the client how long to wait before polling — this is important, because without it every client invents its own interval and hammers you.

**Polling:**

```http
GET /reports/1f4b2a
```
```http
HTTP/1.1 200 OK
Retry-After: 5
{ "id": "1f4b2a", "status": "RUNNING", "progress": 0.42,
  "startedAt": "...", "estimatedCompletion": "..." }
```

On completion, two acceptable designs:

*Option A — the status resource links to the result:*
```json
{ "id": "1f4b2a", "status": "SUCCEEDED", "completedAt": "...",
  "result": { "href": "/reports/1f4b2a/content", "sizeBytes": 4194304, "expiresAt": "..." } }
```

*Option B — `303 See Other` redirect to the result:*
```http
HTTP/1.1 303 See Other
Location: /reports/1f4b2a/content
```
`303` is the semantically correct code here: "the response to your request is at another URI, fetch it with GET". Option A is friendlier to programmatic clients (a redirect on a JSON API surprises many HTTP libraries); Option B is more RESTful. Either is defensible — state the trade-off.

Failure is *not* an HTTP error on the status resource. The `GET` succeeded; the *job* failed:
```json
{ "id": "1f4b2a", "status": "FAILED",
  "error": { "type": "https://errors.acme.com/report-timeout", "title": "Report generation timed out",
             "detail": "Query exceeded 600s", "retryable": true } }
```
Returning `500` from the status endpoint would be wrong and would confuse client retry logic.

**Polling vs webhook vs SSE**

| Mechanism | When to use | Costs |
|---|---|---|
| **Polling** | Default. Always implement it, even alongside push. Works everywhere, through every proxy and firewall, trivially resumable, no client infrastructure. | Latency ≈ poll interval; wasted requests. Mitigate with `Retry-After`, exponential backoff, and ETag/`304` on the status resource. |
| **Webhook** | Server-to-server, long jobs (minutes to hours), many clients. Client registers a callback URL. | Client must expose a public HTTPS endpoint; you must sign payloads (HMAC over body + timestamp), retry with backoff, dedupe on the receiver, and handle the client being down. Delivery is at-least-once. Cannot be the *only* mechanism — always pair with polling for reconciliation. |
| **SSE** (`text/event-stream`) | Browser client, seconds-to-minutes jobs, live progress. One-way server→client over plain HTTP, auto-reconnects with `Last-Event-ID`. | Holds a connection (mind proxy idle timeouts and the HTTP/1.1 six-connections-per-origin limit); needs sticky routing or a shared pub/sub if you have multiple instances. |
| **WebSocket** | Bidirectional, high-frequency. Overkill for job status. | Stateful connections, harder to scale and to route. |

The mature answer: **polling is the contract; push is an optimisation.** Build the status resource first.

**Idempotency of the trigger**

Without it, a client timeout and retry generates a second identical report — double the cost, ambiguous results. Accept an `Idempotency-Key` header, store it with the created job ID and a hash of the request body, and on a repeat key return the **original** `202`/`Location` rather than creating a new job. Enforce it with a unique constraint on the key, not with a read-then-write check (which races). Return `409 Conflict` if the same key arrives with a *different* body — that indicates a client bug. See Q56 for the full mechanism.

**Retention of results**

State it explicitly in the API: results expire (e.g. 7 days), communicated via `expiresAt` in the payload and, once expired, `410 Gone` on the content URL (not `404` — `410` tells the client it existed and is permanently gone, so don't retry). Store the artifact in object storage (S3/GCS) rather than your database, and hand out **pre-signed, short-lived URLs** so the download bypasses your application entirely. Set a lifecycle policy on the bucket so cleanup is automatic rather than a job you'll forget.

**Cancellation semantics**

```http
DELETE /reports/1f4b2a
```
`DELETE` on the job resource means "cancel and discard". Design decisions to state:
- Return `202 Accepted` if cancellation is asynchronous (the worker must notice), `204 No Content` if it took effect immediately.
- Cancellation must be **cooperative**: the worker checks a `cancelRequested` flag at safe points and unwinds cleanly. You cannot forcibly kill a task mid-transaction.
- It must be **idempotent**: cancelling an already-cancelled or already-completed job returns success (or `409` for a terminal state — pick one and document it).
- Model the state machine explicitly: `PENDING → RUNNING → {SUCCEEDED | FAILED | CANCELLED}`, with `CANCELLING` as an intermediate if cancellation is async. Terminal states never transition.

**Server-side implementation notes:** the `POST` handler must only *enqueue* (write the job row and publish a message — ideally via the transactional outbox of Q76, so you never have a job row without a message or vice versa) and return immediately. Workers are separate and horizontally scalable. Include a lease/heartbeat on running jobs so a crashed worker's job is reclaimed rather than stuck in `RUNNING` forever, and a maximum attempt count leading to `FAILED` rather than infinite retry.

### Probes

**`202 Accepted` + `Location`.** Covered, plus `Retry-After` and the `303` alternative on completion.

**Polling vs webhook vs SSE.** Covered in the comparison table, with the "polling is the contract" conclusion.

**Status resource.** Covered, including the important detail that job failure is a `200` with a failure status, not an HTTP error.

**Idempotency of the trigger.** Covered.

**Retention of results.** Covered — `expiresAt`, `410 Gone`, object storage with pre-signed URLs and lifecycle policies.

**Cancellation semantics.** Covered — cooperative, idempotent, explicit state machine, and the `202` vs `204` choice.

---

## Q56. Idempotency per HTTP method, and making `POST /payments` safe to retry.

### Answer

**Definitions first, because they are routinely conflated.**

- **Safe** — the method is read-only and has no side effects the client is responsible for. `GET`, `HEAD`, `OPTIONS`, `TRACE`.
- **Idempotent** — executing the request N times has the same *effect on server state* as executing it once. Safe methods are trivially idempotent. `PUT` and `DELETE` are idempotent by specification.
- **Cacheable** — a separate property (`GET`, `HEAD`, and `POST` under specific conditions).

| Method | Safe | Idempotent | Notes |
|---|---|---|---|
| GET / HEAD | Yes | Yes | Must never mutate. A `GET /orders/1/cancel` endpoint is a real bug — crawlers and prefetchers will fire it. |
| PUT | No | **Yes** | "Make the resource have this state." Two identical PUTs leave the same state. |
| DELETE | No | **Yes** | Deleting twice leaves the resource deleted. The *response* may differ (`204` then `404`), which does **not** break idempotency — idempotency is about server state, not the response. |
| POST | No | **No** | Deliberately unconstrained: "process this payload." Two POSTs create two resources. |
| PATCH | No | **Not guaranteed** | Depends entirely on the patch document. `{"status":"SHIPPED"}` is idempotent; `{"op":"add","path":"/tags/-","value":"x"}` or "increment balance by 10" is not. |

**Why this matters practically:** an idempotent method can be safely retried by any layer — the client's HTTP library, a proxy, a service mesh (Envoy will retry idempotent methods on connection failure by default), a load balancer. A non-idempotent method must not be automatically retried, which is why `POST` needs an explicit mechanism.

**Making `POST /payments` safe to retry**

The failure mode: the client sends the request, the server charges the card and commits, and then the response is lost (network partition, client timeout, pod eviction). The client cannot distinguish "never processed" from "processed, response lost". If it retries, you double-charge. If it doesn't, you may have lost a payment. **Retries in a distributed system are unavoidable, so the server must make them safe.**

**The mechanism — `Idempotency-Key`.** A client-generated unique value (a UUIDv4) sent as a header:

```http
POST /payments HTTP/1.1
Idempotency-Key: 7c9e6679-7425-40de-944b-e07fc1f90ae7
Content-Type: application/json

{ "orderId": "A-1001", "amount": "49.99", "currency": "EUR", "card": {...} }
```

This is the pattern Stripe popularised and it is now the subject of an IETF draft (`draft-ietf-httpapi-idempotency-key-header`). It is a de-facto standard, not yet an RFC — say so rather than citing a nonexistent RFC number.

**The server-side algorithm:**

```sql
CREATE TABLE idempotency_keys (
    key            varchar(255) PRIMARY KEY,
    request_hash   varchar(64)  NOT NULL,      -- SHA-256 of method+path+body
    state          varchar(16)  NOT NULL,      -- IN_PROGRESS | COMPLETED
    response_status int,
    response_body  jsonb,
    resource_id    varchar(64),
    created_at     timestamptz  NOT NULL DEFAULT now(),
    expires_at     timestamptz  NOT NULL
);
```

1. Compute `request_hash` from the request.
2. Attempt `INSERT ... (key, request_hash, 'IN_PROGRESS')`. **The insert is the concurrency control** — the primary key constraint is what makes this atomic. Do *not* do `SELECT` then `INSERT`; two concurrent retries both see nothing and both charge.
3. **Insert succeeded** → this is the first request. Process the payment. On completion, in the *same transaction* as the business write, update the row to `COMPLETED` with the stored response. Return it.
4. **Insert failed with a unique violation** → a request with this key already exists. Read the row:
   - `request_hash` **differs** → the client reused a key for a different payload. Return **`422 Unprocessable Entity`** (Stripe's choice) or `409 Conflict`. This is a client bug and must be loud.
   - `state = COMPLETED` → return the **stored original response** verbatim, with the same status code. Add a header such as `Idempotent-Replay: true` so the client can distinguish.
   - `state = IN_PROGRESS` → the original is still running. Return **`409 Conflict`** with `Retry-After`, telling the client to poll or retry shortly. (Blocking and waiting is the alternative but ties up a thread and can exceed the client's timeout.)

**Why the unique constraint is the enforcement point.** Any check-then-act is a race. Two pods processing simultaneous retries will both pass a `SELECT` check. Only the database's atomic constraint is a correctness boundary. This generalises: the same argument applies to "create user if email not taken", "reserve the last item", and every other check-then-act in a distributed system. Let the database say no, and handle the exception.

**Duplicate request vs duplicate business intent**

These are different problems and conflating them is a design error:
- A **duplicate request** is the same logical operation delivered twice — a retry. The idempotency key solves this. The key must be generated **once per logical operation** by the client (at the moment the user presses "Pay") and **reused across all retries of that operation**. If the client generates a new UUID per HTTP attempt, the mechanism does nothing.
- **Duplicate business intent** is the user genuinely pressing "Pay" twice, or two operators creating the same invoice. Different keys, different requests, both legitimate at the HTTP layer. The defence is a **business-level uniqueness constraint** — e.g. `UNIQUE (order_id, status) WHERE status IN ('PENDING','SUCCEEDED')`, or a domain rule that an order can have at most one successful payment. Never rely on idempotency keys for this.

**TTL.** Keys must expire — 24 hours (Stripe) to 7 days is typical. Rationale: the table grows unboundedly otherwise, and a retry arriving days later is almost certainly a bug rather than a legitimate retry. Document the window. Purge with a partitioned table or a scheduled delete; index `expires_at`.

**At-least-once delivery reality**

The deeper point, worth stating explicitly: **exactly-once delivery is impossible** over an unreliable network (this is the Two Generals problem). You can have at-most-once (don't retry; risk loss) or at-least-once (retry; risk duplicates). Since losing payments is unacceptable, you choose at-least-once and make the *receiver* idempotent. That combination — at-least-once delivery plus an idempotent consumer — is what people mean when they say "effectively exactly-once", and it is the only honest formulation. The same reasoning drives Kafka consumer design (Q83) and the outbox pattern (Q76).

**Extending to the payment service provider.** Your own idempotency does not protect you if you call the PSP twice. Every serious PSP accepts an idempotency key of its own — derive it deterministically from your payment ID so a retry of your handler sends the same key to them. See Q100.

### Probes

**Idempotency-Key header.** Covered, with the honest note about its standardisation status.

**Storing the key + response.** Covered, with the schema, the `IN_PROGRESS`/`COMPLETED` states, and the replay behaviour.

**Uniqueness constraint as the enforcement point.** Covered, with the explanation of why check-then-act is broken.

**TTL.** Covered.

**Distinguishing duplicate request from duplicate business intent.** Covered, including where the client must generate the key.

**At-least-once delivery reality.** Covered, including why "exactly-once" is a misnomer.

---

## Q57. Two clients update the same resource concurrently — prevent the silent clobber.

### Answer

**The problem (lost update):**

```
Client A: GET /orders/42        → {"status":"NEW","notes":"","version":3}
Client B: GET /orders/42        → {"status":"NEW","notes":"","version":3}
Client A: PUT /orders/42        → {"status":"SHIPPED","notes":""}       (saved)
Client B: PUT /orders/42        → {"status":"NEW","notes":"call back"}  (saved)
```

B's write, based on a stale read, silently reverts A's change. Nobody is told. This is the single most common concurrency bug in CRUD APIs, and "last write wins" is a *choice* — usually an unconsidered one.

**The HTTP solution: conditional requests with ETags.**

```http
GET /orders/42
```
```http
HTTP/1.1 200 OK
ETag: "3"
Content-Type: application/json
{ "id": 42, "status": "NEW", "notes": "" }
```

```http
PUT /orders/42
If-Match: "3"
Content-Type: application/json
{ "status": "SHIPPED", "notes": "" }
```

- If the server's current ETag is `"3"` → apply the update, return `200`/`204` with the **new** ETag.
- If it is not → **`412 Precondition Failed`**. The client must re-fetch, re-apply its change (or show a merge UI), and retry.

**`428 Precondition Required`** (RFC 6585) is the complement: when a client sends `PUT`/`PATCH`/`DELETE` **without** `If-Match`, reject it with `428` rather than silently permitting a blind overwrite. This converts an opt-in safety mechanism into a mandatory one — which is what you want for any resource with concurrent editors. Document it, because it will break naive clients (deliberately).

Related conditional headers: `If-None-Match: *` on a `POST`/`PUT` to a known URI prevents accidental creation-over-existing; `If-None-Match: "3"` on a `GET` gives you caching (`304 Not Modified`); `If-Unmodified-Since` is the weaker date-based equivalent of `If-Match`, limited by one-second granularity — prefer ETags.

**Weak vs strong ETags**

- **Strong** (`ETag: "3"`) — guarantees **byte-for-byte** equality of the representation. Required for `If-Match` and for range requests, because a partial fetch of a different byte sequence would corrupt the result.
- **Weak** (`ETag: W/"3"`) — guarantees only *semantic* equivalence; the bytes may differ (different whitespace, a re-ordered JSON object, a different compression). Usable for `If-None-Match` caching but **not** for `If-Match` concurrency control, per RFC 9110.

Practical consequence: if you generate the ETag by hashing the serialised response body, gzip or a Jackson configuration change alters it, so you must either mark it weak or derive it from something stable. **The robust approach is to derive a strong ETag from the entity's version number or a hash of its persistent state** — not from the rendered bytes. `ETag: "42-7"` (id-version) is a common, honest scheme. Note that Spring's `ShallowEtagHeaderFilter` hashes the rendered body and produces a *strong* ETag, and it saves bandwidth but not server work (the response is fully generated before hashing); it is fine for caching, less appropriate as your concurrency token.

**Mapping to the database `@Version`**

This is the important connection — the HTTP layer's ETag should be backed by real database-level optimistic locking, or it is theatre:

```java
@Entity
class Order {
    @Id Long id;
    @Version Long version;   // Hibernate increments on flush
    // ...
}

@PutMapping("/orders/{id}")
ResponseEntity<OrderDto> update(@PathVariable Long id,
                                @RequestHeader(value = "If-Match", required = false) String ifMatch,
                                @RequestBody UpdateOrder body) {
    if (ifMatch == null) throw new PreconditionRequiredException();     // → 428
    long expected = ETags.parseVersion(ifMatch);
    try {
        Order updated = service.update(id, expected, body);             // passes version into the UPDATE
        return ResponseEntity.ok().eTag("\"" + updated.getVersion() + "\"").body(toDto(updated));
    } catch (ObjectOptimisticLockingFailureException e) {
        throw new PreconditionFailedException();                        // → 412
    }
}
```

Hibernate issues `UPDATE orders SET ..., version = 4 WHERE id = 42 AND version = 3`. If zero rows are affected, someone else won and it throws `OptimisticLockException` / Spring's `ObjectOptimisticLockingFailureException`. Two subtleties:
1. Checking the `If-Match` version in application code and *then* saving is a check-then-act race across requests; you must carry the expected version into the `UPDATE`'s `WHERE` clause. Setting the version field on a detached entity before `merge` achieves this; Spring Data's `save` on a detached entity with a set version does too.
2. `@Version` only increments when Hibernate flushes a change to *that* entity. Modifying a `@OneToMany` collection's children does not bump the parent's version unless you use `@OptimisticLock(excluded=false)` semantics or force it with `LockModeType.OPTIMISTIC_FORCE_INCREMENT`. If your API treats the aggregate as one resource, force the increment — otherwise two clients editing different children both succeed and your ETag lied.

**The three layers, unified.** The same optimistic-concurrency idea appears at each level, and a good answer connects them: `AtomicReference.compareAndSet` in memory; `UPDATE ... WHERE version = ?` in the database; `If-Match` → `412` over HTTP. All are "verify the state you read is still current, atomically, as part of the write." (See Q22.)

**Alternatives, and when they are better**

- **Pessimistic locking over HTTP** — a lock/lease endpoint (`POST /orders/42/lock` returning a lease token with a TTL). Appropriate for long human edits (a document editor) where losing 20 minutes of work to a `412` is unacceptable. Costs: leases must expire, clients must renew, and you need lock-stealing/override for abandoned sessions.
- **Field-level merge / CRDTs / operational transformation** — for genuinely collaborative editing, where rejecting the second writer is the wrong product behaviour. Far more complex; only justified for real-time collaboration.
- **Command-shaped endpoints instead of state-replacing PUTs.** `POST /orders/42/ship` describes an *intent* rather than a full new state, so two concurrent commands (`ship` and `add-note`) do not conflict at all. This is often the best fix — much of the lost-update problem is created by modelling everything as a whole-resource `PUT`. Worth raising in the interview as the design-level answer rather than only the mechanical one.

### Probes

**ETag + `If-Match` → `412 Precondition Failed`.** Covered with the full request/response cycle.

**`428 Precondition Required`.** Covered, including that it should be used to *mandate* conditional writes.

**Weak vs strong ETags.** Covered, with the rule that only strong ETags may be used with `If-Match`, and the practical advice to derive them from the version rather than the rendered bytes.

**How this maps to a DB `@Version`.** Covered, with the generated SQL, the exception mapping, and the two subtleties (carrying the version into the `WHERE` clause, and collection changes not bumping the parent version).

---

## Q58. Versioning strategies — compare and defend a choice.

### Answer

**The strategies**

**1. URI path** — `/v1/orders`, `/v2/orders`
- *Pros:* immediately visible, trivially routable at the gateway (route `/v2/*` to a different deployment), easy to test in a browser or curl, obvious in logs and dashboards. By far the most common in practice (Stripe's dashboard aside, GitHub, Twilio, and most public APIs use it).
- *Cons:* strictly, it violates REST's principle that a URI identifies a resource — `/v1/orders/42` and `/v2/orders/42` are the same order, so the "same thing" has two identities, which breaks link relations and client-side caching by URI. Bumping the version means rewriting every URI, including those embedded in stored links and documentation.

**2. Custom header** — `X-API-Version: 2` or `API-Version: 2026-08-13`
- *Pros:* URIs stay stable and canonical. Fine-grained; **date-based versioning** (Stripe's model) lets each customer pin to the date they integrated, and the server applies a chain of request/response transformations to translate between versions.
- *Cons:* invisible in a browser, harder to test by hand, easy to forget (so you need a sensible default and a deprecation path), and caches must be told to vary on it (`Vary: API-Version`) or they will serve v1 responses to v2 clients — a nasty and very real bug.

**3. Media type / content negotiation** — `Accept: application/vnd.acme.order.v2+json`
- *Pros:* the most theoretically correct: the URI identifies the resource, the media type identifies the *representation*. Natively supported by HTTP's `Accept` negotiation and by `Vary: Accept`, which caches already honour. Allows per-resource versioning rather than one global version.
- *Cons:* verbose, unfamiliar to many client developers, awkward in browsers and simple tools, and per-resource versioning creates a combinatorial support matrix. Genuinely rare outside hypermedia-oriented APIs (GitHub used it, then moved to a header).

**4. Query parameter** — `/orders?version=2`
- *Pros:* trivial to add and to test.
- *Cons:* mixes a control concern with resource filtering, easy to drop accidentally, cache keys often ignore or mishandle it, and it's generally considered the weakest option.

**My position, and the argument I'd make**

**The real answer is that versioning strategy is the *least* important decision here.** All four work. What determines whether your API is maintainable is whether you can **avoid breaking changes in the first place** — because every versioning scheme has the same underlying cost: from the day you ship v2, you operate and support *both* versions, with duplicated tests, duplicated bug fixes, and a migration project for every consumer.

So the strategy I would defend is:

**Primary: additive evolution with no version bump.**
- Add fields; never remove or rename them. Never change a field's type or its semantics.
- Never make an optional request field required, and never add a required field without a default.
- Never add a value to an enum that existing clients must handle, unless the contract already specifies unknown-value behaviour (document `UNKNOWN` handling from day one — this is one of the most common accidental breaks).
- Never change error codes or status codes for existing conditions.
- Never tighten validation on inputs you previously accepted.

**Enabled by the tolerant reader pattern (Postel's law, applied carefully).** Consumers must ignore unknown fields rather than failing on them — `FAIL_ON_UNKNOWN_PROPERTIES=false` in Jackson (which is Spring Boot's default, deliberately), and no strict schema validation that rejects extras. Publish this as a contract requirement for your consumers, because an additive strategy only works if clients are tolerant. Note the caveat: tolerant reading is right for *evolution* but wrong for *security-sensitive* inputs, where unknown fields should be rejected (mass-assignment vulnerabilities).

**Secondary: URI path versioning for genuine breaking changes**, because operationally it is the simplest — a gateway route, an unambiguous log line, and a metric per version telling you exactly who is still on v1. I would version at the **API level, not per resource** (one `/v2`, not `/orders/v2` and `/customers/v3`), because per-resource versioning produces a support matrix nobody can reason about.

**With a deprecation policy that is actually enforced:**
- `Deprecation: @1767225600` and `Sunset: Wed, 31 Dec 2026 23:59:59 GMT` headers (RFC 8594 defines `Sunset`; `Deprecation` is RFC 9745) plus a `Link: <docs>; rel="deprecation"`.
- A **metric per version per consumer** so you know exactly who must migrate. Without this you can never turn v1 off, and you will run it forever. This is the single most important operational piece.
- A published timeline (e.g. minimum 6 months' notice, 12 for major consumers), proactive contact with the remaining consumers, and — for internal APIs — an agreed deadline with teeth.
- Optionally, "brownouts": short scheduled periods where v1 returns `410` before the real sunset, to smoke out forgotten integrations.

**Internal vs external is the deciding context**, and saying so demonstrates judgement: for internal microservices you can often coordinate a breaking change across teams in a sprint and skip versioning entirely (with consumer-driven contract tests to catch breakage in CI — see Q94). For a public API with thousands of unknown consumers, you can never remove anything, and Stripe's date-based versioning with server-side transformation chains is the gold standard — expensive, but it lets a 2015 integration keep working unchanged.

### Probes

**URI path, header/media type, query param.** All four covered with pros and cons.

**Consumer migration cost.** Covered — the point that the cost is dominated by operating two versions and by consumer migration, not by the URL scheme.

**The real answer is usually "avoid breaking changes".** Covered with the concrete additive-change rules and the non-obvious ones (enums, validation tightening, error codes).

**Additive fields, tolerant reader.** Covered, with the Jackson setting, the requirement to publish it as a consumer contract, and the security caveat.

**Deprecation headers, sunset policy.** Covered — `Deprecation` (RFC 9745), `Sunset` (RFC 8594), per-consumer usage metrics as the enabling mechanism, timelines, and brownouts.

---

## Q59. Error responses for an API consumed by 20 teams.

### Answer

**Standardise on RFC 9457 `application/problem+json`** (which obsoletes RFC 7807 — cite 9457, it's a differentiator). Spring Framework 6 supports it natively via `ProblemDetail` and `ErrorResponseException`.

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{
  "type": "https://errors.acme.com/insufficient-funds",
  "title": "Insufficient funds",
  "status": 422,
  "detail": "Account A-1001 has 12.50 EUR available; 49.99 EUR requested.",
  "instance": "/payments/7c9e6679",
  "code": "PAYMENT_INSUFFICIENT_FUNDS",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "available": "12.50",
  "requested": "49.99",
  "currency": "EUR"
}
```

The five standard members are `type` (a URI identifying the problem *kind*, and the primary identifier), `title` (a short human-readable summary, stable for a given `type`), `status`, `detail` (human-readable, specific to *this* occurrence), and `instance` (a URI for this occurrence). Extensions are permitted at the top level, which is where `code`, `traceId`, and structured data go.

**Machine-readable codes vs human messages — the central discipline**

- **`code` (or `type`) is the contract.** It is a stable, enumerated, documented string that clients branch on. It never changes once published, even if the wording changes. Publish the full list as an enum in your OpenAPI spec so client SDKs get generated constants.
- **`title`/`detail` are for humans** — logs, support tickets, developer consoles. They may be reworded, localised, or improved at any time. **Clients must never parse them.** State this explicitly in your API documentation; with 20 teams, someone *will* write `if (message.contains("insufficient"))` unless you give them a better option and tell them not to.

Guidance for good codes: namespace them (`PAYMENT_`, `ORDER_`), make them specific enough to act on (`PAYMENT_CARD_EXPIRED` beats `PAYMENT_FAILED`), and include a `retryable: true|false` flag so clients don't have to encode your retry policy in a switch statement — this is one of the highest-value extension fields you can add.

**Validation errors need structure**

A single `detail` string cannot drive a form UI. Use a documented extension array:

```json
{
  "type": "https://errors.acme.com/validation-failed",
  "title": "Request validation failed",
  "status": 400,
  "code": "VALIDATION_FAILED",
  "traceId": "...",
  "errors": [
    { "pointer": "/customer/email", "code": "INVALID_FORMAT",
      "detail": "must be a well-formed email address" },
    { "pointer": "/items/0/quantity", "code": "OUT_OF_RANGE",
      "detail": "must be between 1 and 100", "min": 1, "max": 100 }
  ]
}
```

Use **JSON Pointer** (RFC 6901) for `pointer` so it maps unambiguously to a location in the request body, including array indices. Always return **all** validation errors, not just the first — a client that must round-trip once per field is a bad experience. In Spring, that means iterating `MethodArgumentNotValidException.getBindingResult().getFieldErrors()` rather than returning the first one.

**Correlation / trace ID**

Include the **trace ID** in every error body (and ideally in a response header on *all* responses, e.g. `traceparent` or a custom `X-Trace-Id`). This is the single most valuable field for supporting 20 teams: a consumer pastes it into a ticket and you find the exact request across every service in your tracing backend in seconds. With W3C Trace Context propagation and Micrometer Tracing/OpenTelemetry, `Span.current().getSpanContext().getTraceId()` is available in the exception handler for free.

**Not leaking internals**

- **Never** return stack traces, SQL fragments, class names, internal hostnames, file paths, or upstream vendor error text. They are an information-disclosure vulnerability (they reveal your stack, versions, and schema) and they are useless to the caller. Explicitly set `server.error.include-stacktrace=never` and `include-message=never` in Spring Boot — and be aware the *defaults* changed across Boot versions, so set them rather than assuming.
- For `5xx`, return a generic `detail` and the trace ID; log the full exception server-side, once, at the boundary.
- Be careful that `detail` doesn't leak data across a security boundary: "User bob@example.com not found" on a login endpoint is a user-enumeration vulnerability. Authentication failures should be uniformly `401` with an identical body regardless of whether the user exists.
- Similarly, `403` vs `404` for a resource the caller may not access: returning `404` hides existence (better for security), `403` is more honest (better for usability). Pick per-endpoint based on whether existence is sensitive, and be *consistent* — an attacker can distinguish them by timing or by which endpoints return which.

**Status code selection — the ones people get wrong**

| Situation | Code | Note |
|---|---|---|
| Malformed JSON, wrong type, missing required field | **400** Bad Request | The request is syntactically unprocessable. |
| Syntactically valid, semantically invalid (business rule, cross-field validation, "quantity exceeds stock") | **422** Unprocessable Content | This is the 400-vs-422 line: 400 = "I can't parse this", 422 = "I understand it and it's wrong". Both are defensible; **pick one convention and apply it everywhere**, because inconsistency is worse than either choice. |
| Not authenticated / bad or missing credentials | **401** Unauthorized | Misnamed in the spec — it means *unauthenticated*. Must include a `WWW-Authenticate` header. |
| Authenticated but not permitted | **403** Forbidden | |
| Resource doesn't exist | **404** Not Found | |
| Method not supported on this URI | **405** Method Not Allowed | Must include an `Allow` header. |
| State conflict — duplicate creation, version conflict without `If-Match`, illegal state transition ("cannot ship a cancelled order") | **409** Conflict | Not `400`. The request is fine; the *current state* makes it impossible. |
| `If-Match` precondition failed | **412** Precondition Failed | See Q57. |
| Conditional header required but absent | **428** Precondition Required | |
| Rate limit exceeded | **429** Too Many Requests | Must include `Retry-After`. |
| Resource existed and is permanently gone | **410** Gone | Tells the client to stop retrying — better than `404` for expired jobs/exports. |
| Payload too large | **413** Content Too Large | |
| Unexpected server fault | **500** Internal Server Error | Never use `500` for client mistakes; it pollutes your error-rate SLO and triggers client retries that make an outage worse. |
| Upstream dependency failed / timed out | **502** / **504** | Or `503` with `Retry-After` for deliberate load shedding and maintenance. |

The systemic point for 20 consumers: publish this table as documentation, enforce it with a shared `@RestControllerAdvice` in a company library so every service behaves identically, and add contract tests that assert the error shape. Consistency across services is worth more than the perfection of any individual choice — a client that can write one error handler for all 20 APIs is the goal.

### Probes

**RFC 9457 (problem+json).** Covered, with the correction that it obsoletes 7807 and with Spring's native support.

**Stable machine-readable error codes vs human messages.** Covered, including `retryable` and the instruction to consumers never to parse prose.

**Validation error structure.** Covered, with JSON Pointer and the return-all-errors rule.

**Correlation / trace ID in the body.** Covered, including how to obtain it.

**Not leaking internals.** Covered, with the specific Spring Boot properties, the user-enumeration hazard, and the `403`-vs-`404` decision.

**Correct status code selection (422 vs 400, 409 vs 400).** Covered in the table, with the explicit advice that internal consistency beats theoretical correctness.

---

## Q60. `PATCH` semantics, and "set this field to null".

### Answer

**What `PATCH` actually means.** RFC 5789 defines `PATCH` as applying a **set of changes described by the request body** to the target resource. Critically, the RFC does **not** define the format of that body — it only says the media type must describe the changes. So `PATCH` with `Content-Type: application/json` is under-specified: you are inventing a private protocol. This is why so many `PATCH` implementations disagree.

`PATCH` is also **not required to be idempotent** and is **not safe**. It *may* be idempotent depending on the patch format (see below). It supports `If-Match` for conditional application, and RFC 5789 explicitly recommends it.

**The two standard formats**

**JSON Merge Patch — RFC 7386, `application/merge-patch+json`**

```http
PATCH /orders/42
Content-Type: application/merge-patch+json

{ "notes": "call back", "discountCode": null }
```

Rules: a present member replaces the target's value; a member with the value **`null` means *remove* the member**; absent members are untouched. Objects merge recursively.

- *Pros:* intuitive, looks like the resource, trivial for clients to construct.
- *Cons:* **you cannot set a value to literal `null`** — `null` is reserved for deletion. And **arrays are replaced wholesale**, never merged, so you cannot append one element to a 500-element array without sending all 501.
- *Idempotent:* yes.

**JSON Patch — RFC 6902, `application/json-patch+json`**

```http
PATCH /orders/42
Content-Type: application/json-patch+json

[
  { "op": "test",    "path": "/version", "value": 3 },
  { "op": "replace", "path": "/notes", "value": "call back" },
  { "op": "replace", "path": "/discountCode", "value": null },
  { "op": "remove",  "path": "/couponId" },
  { "op": "add",     "path": "/tags/-", "value": "urgent" }
]
```

An ordered array of operations (`add`, `remove`, `replace`, `move`, `copy`, `test`) with JSON Pointer paths. It is applied atomically — all or nothing.

- *Pros:* fully expressive. `replace` with `value: null` sets null; `remove` deletes — **the ambiguity disappears entirely**. Array element operations work (`/tags/-` appends, `/tags/0` targets an index). The `test` op provides built-in optimistic concurrency without a header.
- *Cons:* verbose, unintuitive for client developers, and array-index paths are fragile under concurrency (index 2 may not be the element you meant by the time it applies).
- *Idempotent:* only for some operation sets — `replace` is, `add` to `/tags/-` is not.

In Java: `json-patch` (by `com.github.java-json-tools`) or Jackson's `JsonMergePatch`/`JsonPatch` in `jackson-datatype-jsonp`, typically applied by converting the entity to a `JsonNode`, applying the patch, and deserialising back — with **validation after application**, not before.

**The `Optional<T>` / `JsonNullable<T>` technique for plain JSON PATCH**

If you must accept a plain JSON body (as most APIs do), the core difficulty is that Jackson deserialises both `{"discountCode": null}` and `{}` into a DTO field holding `null`. You cannot distinguish "set to null" from "leave alone". The standard solutions:

*Option 1 — `JsonNullable<T>`* (from `org.openapitools:jackson-databind-nullable`, and what OpenAPI Generator emits):

```java
public class OrderPatch {
    private JsonNullable<String> notes         = JsonNullable.undefined();
    private JsonNullable<String> discountCode  = JsonNullable.undefined();
}

// Applying:
if (patch.getNotes().isPresent()) order.setNotes(patch.getNotes().get());   // may be null
```
Three states: `undefined()` (absent), `of(null)` (explicit null), `of(value)`. This is the cleanest and most widely used approach.

*Option 2 — `Optional<T>` fields.* Works (`null` field = absent, `Optional.empty()` = explicit null, `Optional.of(v)` = value) but abuses `Optional`, which was explicitly not designed for fields, and requires `Jdk8Module` plus careful configuration. Workable but less clear.

*Option 3 — capture the raw `JsonNode`* and check `has("discountCode")` before applying. Simple, no dependency, but you lose type safety and validation.

*Option 4 — `@JsonAnySetter` / a `Set<String> presentFields`* populated during deserialisation. Effective; more machinery.

**Validating a partial update**

The subtle part everyone gets wrong: Bean Validation annotations (`@NotNull`, `@Size`) on a patch DTO are wrong, because absence is legal. Two correct approaches:
1. **Validate the merged result**, not the patch: apply the patch to a copy of the entity, then run the validator on the entity. This catches cross-field invariants ("`shippedAt` must be after `createdAt`") that a per-field check cannot.
2. Use **validation groups** so `@NotNull` applies to the `Create` group only.

Also validate that the patch does not touch immutable or privileged fields — never bind `id`, `version`, `createdAt`, `ownerId`, or `role` from a patch document. This is a **mass-assignment vulnerability**, and it is materially more likely with PATCH than with a strict `PUT` DTO. Use an explicit allowlist of patchable paths, and reject unknown ones with `422`.

**Practical recommendation for an interview**

State the trade-off and pick: for most internal and product APIs, **plain JSON with `merge-patch` semantics, documented explicitly, using `JsonNullable` to distinguish absent from null**, plus an allowlist and post-merge validation. Advertise `Content-Type: application/merge-patch+json` so the semantics are stated in the protocol rather than in prose. Use RFC 6902 JSON Patch when clients need array-element operations or built-in `test` preconditions. And note the design-level alternative from Q57: **many `PATCH` endpoints should be command endpoints instead** (`POST /orders/42/cancel`), which sidesteps the entire null-versus-absent problem, expresses intent, is naturally concurrency-safe, and is far easier to authorise.

### Probes

**JSON Merge Patch (RFC 7386) vs JSON Patch (RFC 6902).** Both covered with examples, rules, idempotency, and the array/null trade-offs. Also noted that RFC 5789 leaves the format undefined.

**`Optional<T>` / `JsonNullable` in DTOs to distinguish absent from null.** Covered, with four implementation options and a recommendation.

**Partial-update validation.** Covered — validate the merged result or use groups, plus the mass-assignment/allowlist warning.

---

## Q61. gRPC or GraphQL over REST for an internal service — and what you give up.

### Answer

**When gRPC**

Choose it for **high-volume, low-latency, service-to-service** communication where both ends are yours:
- **Schema-first contract.** `.proto` files are the single source of truth, with generated clients and servers in every language. The contract cannot drift from the implementation, which is the biggest practical advantage over hand-written REST plus a hand-maintained OpenAPI file.
- **Protobuf binary encoding** — typically 3–10× smaller than JSON and much faster to parse (no string parsing, no reflection-heavy binding). Matters at high message rates or with large payloads.
- **HTTP/2 by default** — multiplexed streams over one connection, header compression (HPACK), no head-of-line blocking at the HTTP layer, and long-lived connections that avoid per-request TLS handshakes.
- **Four call types**, including genuine **streaming**: unary, server-streaming, client-streaming, bidirectional. This is the killer feature — REST has no good answer for bidirectional streaming, and server-streaming beats both polling and SSE for internal use.
- **Built-in deadlines** propagated across hops, cancellation, per-call metadata, pluggable interceptors, and standard status codes.
- **Backwards compatibility is a first-class design goal** of protobuf: add fields with new tag numbers, never reuse or renumber, and old and new binaries interoperate. Field presence in proto3 needs `optional` to distinguish unset from default — a classic source of bugs.

**What you give up with gRPC:** no browser support without a proxy (gRPC-Web via Envoy, or Connect; browsers cannot control HTTP/2 frames from `fetch`). Poor human debuggability — you cannot curl it, and you need `grpcurl` plus server reflection. Weaker ecosystem for API gateways, WAFs, caching, and rate limiting, all of which understand HTTP verbs and paths but not gRPC methods. **No HTTP caching at all** — every call is a `POST` to an opaque path, so CDNs and reverse proxies cannot help. Load balancing is harder: HTTP/2 connections are long-lived and multiplexed, so an L4 load balancer pins all a client's traffic to one backend; you need L7-aware balancing (a service mesh, or client-side balancing via `grpclb`/xDS) or you get badly skewed load. A build-time code-generation step in every consumer. And organisationally, you now maintain proto artifacts and a schema registry.

**When GraphQL**

Choose it when **many diverse clients need different shapes of the same data**, and the shapes change faster than you can ship endpoints:
- Solves **over-fetching** (mobile downloads 40 fields to show 3) and **under-fetching / the N+1 round-trip problem** (a screen needing 6 REST calls becomes 1). This is genuinely valuable for mobile on poor networks.
- **Schema and introspection** give strong typing, generated clients, and excellent tooling (GraphiQL, codegen).
- A **federated graph** (Apollo Federation) lets multiple backend teams own slices of one schema, which is a real organisational win for a large product surface.
- Frontend teams can iterate without a backend deployment.

**What you give up with GraphQL — the honest list**, and these are why it is usually the *wrong* choice for a plain internal service-to-service call:

- **HTTP caching is lost.** Everything is a `POST /graphql` with the query in the body, so CDN, proxy, and browser caching are all unavailable. You replace them with application-level caching, persisted queries (hashing the query so it can be a cacheable `GET`), and client-side normalised caches — significant extra machinery to regain what REST gets free.
- **Resolver N+1.** The `posts { author { name } }` query calls the author resolver once per post. The fix is **DataLoader** batching (per-request batching and caching by key), which every serious GraphQL server needs — but it is easy to forget in one resolver and silently emit 500 queries. Spring for GraphQL provides `BatchLoaderRegistry`/`@BatchMapping`.
- **Query complexity attacks.** A client can request a deeply nested or cyclic query (`user { friends { friends { friends { ... } } } }`) that explodes into millions of resolutions. You *must* implement query depth limiting, complexity/cost analysis with a budget, and preferably an allowlist of persisted queries for production clients. This is a mandatory security control, not an optimisation.
- **Observability and rate limiting are harder.** Every request is `POST /graphql` returning `200 OK` — including for errors, since GraphQL puts errors in the response body. Standard HTTP metrics (per-endpoint latency, error rate by status) become useless; you need GraphQL-aware instrumentation (per-field timings, per-operation-name metrics) and rate limiting by query cost rather than request count. Tooling here is far less mature than for REST.
- **Error handling is non-standard** — partial successes with a `data` object *and* an `errors` array, all under `200`.
- **File uploads, `PATCH`-like partial updates, and long-running operations** all need bespoke conventions.
- Server complexity is meaningfully higher, and authorisation must be enforced **per field**, not per endpoint — a much larger surface to get right.

**The recommendation I would defend**

- **Internal, high-throughput, service-to-service, polyglot, or streaming → gRPC.** The schema-driven contract and performance are worth the tooling cost, and the "no browser, no caching" objections mostly don't apply between services.
- **A backend-for-frontend aggregating many services for varied clients → GraphQL** is a reasonable choice, though a hand-written BFF exposing REST is often simpler and sufficient. GraphQL earns its complexity at a certain scale of client diversity, and not before.
- **A plain internal CRUD service → REST/JSON.** It is boring, universally understood, debuggable with curl, cacheable, and every tool in your infrastructure already speaks it. "We used REST because it was sufficient" is a perfectly strong interview answer, and reaching for gRPC or GraphQL without a driving requirement is the more common mistake.
- Also worth mentioning: these are not exclusive. A common mature architecture is gRPC internally with a REST/GraphQL gateway at the edge (grpc-gateway generates one from your protos), getting the internal efficiency and the external accessibility.

### Probes

**Schema/codegen.** Covered for both, including protobuf's compatibility rules and the proto3 field-presence caveat.

**Streaming.** Covered — the four gRPC call types and why streaming is the differentiator.

**Binary efficiency.** Covered, with the caveat that it matters at volume.

**Browser support.** Covered — gRPC-Web/Connect and why a proxy is required.

**Caching loss.** Covered for both gRPC and GraphQL, including persisted queries as GraphQL's partial mitigation.

**N+1 in GraphQL resolvers.** Covered, with DataLoader and Spring for GraphQL's `@BatchMapping`.

**Over-fetching vs query complexity attacks.** Both covered, with the specific controls (depth limits, cost analysis, persisted-query allowlists).

**Observability tooling maturity.** Covered — the "everything is POST /graphql returning 200" problem, and gRPC's gateway/WAF/load-balancing gaps.

---

# 8. Docker

---

## Q62. Write an optimal multi-stage Dockerfile for a Spring Boot app and explain each decision.

### Answer

```dockerfile
# syntax=docker/dockerfile:1.7

########################  Stage 1: build  ########################
FROM eclipse-temurin:21-jdk-jammy AS build
WORKDIR /build

# Copy only what is needed to resolve dependencies, so this layer
# is cached until the build files themselves change.
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN --mount=type=cache,target=/root/.m2 ./mvnw -B dependency:go-offline

# Now the source. Changes here invalidate only the layers below.
COPY src/ src/
RUN --mount=type=cache,target=/root/.m2 ./mvnw -B clean package -DskipTests

########################  Stage 2: explode the jar  ########################
FROM eclipse-temurin:21-jre-jammy AS extract
WORKDIR /app
COPY --from=build /build/target/*.jar app.jar
RUN java -Djarmode=tools -jar app.jar extract --layers --destination extracted

########################  Stage 3: runtime  ########################
FROM eclipse-temurin:21-jre-jammy
RUN groupadd --system --gid 1001 app \
 && useradd  --system --uid 1001 --gid app --no-create-home app
WORKDIR /app

# Layers ordered from least to most frequently changed.
COPY --from=extract --chown=app:app /app/extracted/dependencies/          ./
COPY --from=extract --chown=app:app /app/extracted/spring-boot-loader/    ./
COPY --from=extract --chown=app:app /app/extracted/snapshot-dependencies/ ./
COPY --from=extract --chown=app:app /app/extracted/application/           ./

USER app
EXPOSE 8080
ENV JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=70 -XX:+ExitOnOutOfMemoryError \
    -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp \
    -Djava.security.egd=file:/dev/./urandom"
ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Decision by decision**

**Multi-stage.** The build stage needs a full JDK, Maven, the local repository, and your entire source tree — several hundred MB to a couple of GB. None of that belongs in the runtime image, where it is both wasted space and attack surface (a compiler and a package manager inside a running container are useful to an attacker, not to you). Only the named artifacts cross the `COPY --from` boundary.

**Dependencies before source — the cache decision that matters most.** Docker layers are cached by the checksum of their inputs; changing any layer invalidates every layer after it. Source code changes on every commit; `pom.xml` changes rarely. Copying `pom.xml` and resolving dependencies *before* copying `src/` means a normal code change reuses the dependency layer and skips a multi-minute download. Getting this wrong is the single most common Dockerfile mistake (see Q65).

**`--mount=type=cache`** (BuildKit) keeps `~/.m2` across builds without baking it into a layer. It survives even when the `pom.xml` layer *is* invalidated, so adding one dependency re-downloads one dependency instead of four hundred. Requires BuildKit, which is the default in modern Docker; on CI, ensure the cache mount is persisted or backed by a registry cache (`--cache-to`/`--cache-from`) or it does nothing between ephemeral runners.

**JRE, not JDK, at runtime.** A JRE image is roughly half the size and removes `javac`, `jlink`, and the rest. Smaller still: build a custom runtime with `jlink` (`jdeps` to find required modules, then `jlink --add-modules ... --strip-debug --no-man-pages --compress=2`), which for a typical Spring Boot app produces a ~60–90 MB runtime instead of ~180 MB. Worth it when image pull time matters (large fleets, frequent scaling); note that Spring Boot's heavy reflection makes `jdeps` analysis incomplete, so you usually add modules manually and test carefully.

**Layered jars / `jarmode`.** A fat Spring Boot jar is one ~60 MB file. Change one line of your code and Docker must ship all 60 MB, because the jar's checksum changed — even though 55 MB of it is unchanged third-party libraries. Boot's layered-jar support splits it into four directories ordered by change frequency: `dependencies` (rarely change), `spring-boot-loader` (changes with the Boot version), `snapshot-dependencies`, `application` (your code, changes constantly). Copying each as its own layer means a code change ships only the small `application` layer. This routinely cuts push/pull bytes by 10–20×.

*Version note:* `-Djarmode=layertools ... extract` is the Boot 2.3–3.2 syntax; Boot 3.3+ introduced `-Djarmode=tools -jar app.jar extract --layers`, and `layertools` is deprecated. Verify against your Boot version — quoting the wrong one is an easy interview stumble.

**Alternatives to hand-writing this:** **Cloud Native Buildpacks** (`./mvnw spring-boot:build-image`) produce an optimally layered, CVE-patched, non-root image with correct JVM container settings and *no Dockerfile*, and support rebasing (patching the OS layer without rebuilding the app). **Jib** (Google) builds daemonless, reproducible, layered images straight from Maven/Gradle. For most Spring Boot services these are better than a hand-rolled Dockerfile; a hand-written one is justified when you need unusual control. Saying this demonstrates current knowledge.

**Non-root user.** Containers run as root by default, and a container escape or a mounted volume then has root privileges. Create a system user with a fixed high UID/GID and `USER` before the entrypoint. A fixed numeric UID matters because Kubernetes' `runAsNonRoot` check compares numeric UIDs, and `securityContext.runAsUser` must match a user that can read the files. Also `--chown` on `COPY` so the app can read its own files. In Kubernetes, pair this with a `securityContext` of `runAsNonRoot: true`, `readOnlyRootFilesystem: true` (with an `emptyDir` on `/tmp`), `allowPrivilegeEscalation: false`, and `capabilities: drop: [ALL]`.

**Distroless / Alpine trade-offs.**
- **Distroless** (`gcr.io/distroless/java21-debian12`) contains the JRE and its libc dependencies and *nothing else* — no shell, no package manager, no coreutils. Smallest attack surface and the fewest CVEs in a scan. Cost: you cannot `kubectl exec` into a shell to debug (mitigated by the `:debug` tag, which adds busybox, and by `kubectl debug` ephemeral containers), and any `HEALTHCHECK` or entrypoint script that needs a shell won't work — the exec form of `ENTRYPOINT` is mandatory.
- **Alpine** is small (~5 MB base) but uses **musl libc** rather than glibc. For Java this is a real concern: it was historically unsupported, and while there are now official musl builds (and Alpine avoids the glibc malloc-arena problem from Q8), you can hit subtle differences in DNS resolution, locale/charset handling, stack sizes, and any native library that assumes glibc. Performance differences in `malloc`-heavy workloads have been measured in both directions. **The safe default for Java is a slim Debian/Ubuntu-based image** (`-jammy`, `-jre-jammy`) or distroless; choose Alpine only if you've tested it.

**Pinned base image digest.** `FROM eclipse-temurin:21-jre-jammy` is a mutable tag — it points at different bytes over time, so your build is not reproducible and a rebuild can silently change your JVM patch version. Pin by digest for reproducibility:
```dockerfile
FROM eclipse-temurin:21-jre-jammy@sha256:abc123...
```
The tension: pinning also freezes your CVE patches. Resolve it with automated digest bumping (Renovate/Dependabot both do this) so upgrades are explicit, reviewed, and tested — rather than either "silently changing" or "never updating".

**Build-time args vs run-time env.** `ARG` exists only during the build and is **visible in the image history** (`docker history`), so **never pass secrets as `ARG`**. Use BuildKit secret mounts instead:
```dockerfile
RUN --mount=type=secret,id=gradle_props,target=/root/.gradle/gradle.properties ./gradlew build
```
which makes the file available during that `RUN` only and never writes it to a layer. `ENV` values persist into the running container and are visible in `docker inspect` and in the pod spec — fine for configuration, wrong for secrets. Runtime secrets belong in Kubernetes Secrets mounted as files, or a secrets manager (Q36).

**`JAVA_TOOL_OPTIONS`** is picked up by the JVM automatically, so it can be overridden per-environment without changing the entrypoint. `-XX:+ExitOnOutOfMemoryError` is important in Kubernetes: without it, a heap-exhausted JVM can limp along failing every request while still passing a naive liveness probe. Better to die and be restarted. `HeapDumpOnOutOfMemoryError` with the path on a mounted volume gives you the evidence.

**Other essentials not shown:** a `.dockerignore` (next question), `EXPOSE` as documentation only (it publishes nothing), OCI labels (`org.opencontainers.image.revision`, `.source`, `.created`) so an image can be traced to a commit, and a `HEALTHCHECK` only if you run under plain Docker — **Kubernetes ignores `HEALTHCHECK` entirely** and uses probes instead.

### Probes

**Layer caching (dependencies before source).** Covered, plus BuildKit cache mounts and the CI caveat.

**`.dockerignore`.** Covered in Q65.

**JRE not JDK.** Covered, plus `jlink` for going further.

**Layered jars / jib / buildpacks.** Covered, including the Boot 3.3 `jarmode` syntax change.

**Non-root user.** Covered, with the fixed-UID rationale and the matching Kubernetes `securityContext`.

**Distroless/Alpine trade-offs (musl vs glibc).** Covered in detail, with a recommendation.

**Pinned base image digest.** Covered, including the tension with patching and how to resolve it.

**Build-time vs run-time args.** Covered, including why `ARG` must never hold secrets and what to use instead.

---

## Q63. The JVM ignores the container memory limit and gets OOMKilled.

### Answer

**What is actually happening**

A container is not a VM. It is a process in cgroups and namespaces on the host kernel. `/proc/meminfo` and `sysconf(_SC_NPROCESSORS_ONLN)` report the **host's** resources, not the cgroup's limits. A JVM that reads those sees a 64 GB, 32-core machine, applies default ergonomics (max heap = 1/4 of physical memory = 16 GB), grows the heap toward 16 GB inside a 512 MB container, and the kernel's cgroup OOM killer terminates it with SIGKILL. Exit code **137** (128 + 9). Kubernetes reports `OOMKilled`.

**Container awareness**

`-XX:+UseContainerSupport` makes the JVM read cgroup limits (`memory.limit_in_bytes` on v1, `memory.max` on v2) instead of host memory, and derive `availableProcessors()` from CPU quota/shares. It has been **on by default since JDK 10 and backported to 8u191**, so any JDK 11+ image already has it. If you are on an older JDK 8, the manual equivalents were `-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap`.

**cgroups v1 vs v2.** This is where "container aware" quietly fails. cgroups v2 became the default on RHEL 9, Ubuntu 21.10+, Debian 11+, and modern Kubernetes distributions, and the file layout and semantics differ from v1. JDK support for cgroups v2 landed in **JDK 15** and was backported to 11.0.16 and 8u372. **A JDK 11.0.9 container on a cgroups v2 host does not detect its limits** and falls back to host memory — producing exactly the failure described, on a JVM everyone assumed was container-aware. Verify with:
```
java -XX:+PrintFlagsFinal -version | grep -E 'MaxHeapSize|UseContainerSupport'
```
run *inside* the container with the real limits applied. This one-liner is the definitive check and a strong thing to mention.

**`MaxRAMPercentage` vs `-Xmx`**

- `-Xmx512m` is absolute. It ignores the container limit entirely, so the same image behaves differently across environments and you must remember to change it whenever you change the limit. The two drift, always.
- `-XX:MaxRAMPercentage=70.0` computes the heap as a percentage of the *detected container limit*. One image, correct behaviour at any size. It is a `double`, so `-XX:MaxRAMPercentage=70` works but `=70.0` is the documented form. It applies **only when `-Xmx` is absent** — setting both means `-Xmx` wins, which is a common silent misconfiguration.
- Related: `InitialRAMPercentage` (equivalent to `-Xms`; setting it equal to `MaxRAMPercentage` avoids heap resizing and is often good practice in a fixed-size container), and `MinRAMPercentage`, which despite its name applies only to *small* physical memory (under ~250 MB) — a genuinely confusing API.

**Heap is only part of RSS.** The central misconception. Setting `MaxRAMPercentage=90` in a 512 MB container leaves ~50 MB for metaspace, code cache, thread stacks, direct buffers, GC structures, and the JVM's own native allocations — nowhere near enough, and you will be OOMKilled with a heap that never came close to its maximum. Sensible starting points: **70–75%** for containers above ~1 GB, and **50–60%** for small containers, because the non-heap overhead is roughly fixed (~150–250 MB for a typical Spring Boot service) and therefore a *larger fraction* of a small limit. Then verify with Native Memory Tracking under real load (Q8) rather than guessing.

You can also bound the non-heap regions explicitly for predictability: `-XX:MaxMetaspaceSize`, `-XX:ReservedCodeCacheSize`, `-XX:MaxDirectMemorySize`, and `-Xss`. `-XX:MaxRAM` sets the value the JVM treats as "physical memory" if detection is wrong.

**`MALLOC_ARENA_MAX`.** glibc allocates up to `8 × cores` per-thread malloc arenas, each reserving 64 MB of virtual address space and retaining freed memory. On a 16-core host this can add hundreds of MB of RSS that no JVM flag controls. `ENV MALLOC_ARENA_MAX=2` is a standard, almost free mitigation for containerised Java and is applied by Cloud Native Buildpacks automatically. Alternatives: jemalloc/tcmalloc via `LD_PRELOAD`, or a musl-based image.

**CPU limits → `availableProcessors()` → thread counts — the second-order effect**

`Runtime.availableProcessors()` is derived from the cgroup CPU quota: roughly `ceil(cpu.max quota / period)`, i.e. `limits.cpu`. If no limit is set, it falls back to `cpu.shares`-derived logic or the host count (`-XX:PreferContainerQuotaForCPUCount` controls the preference). That single number drives:

- **GC parallel threads** (`ParallelGCThreads`) and concurrent GC threads
- **JIT compiler threads** (`CICompilerCount`)
- **The common ForkJoinPool** size — so `parallelStream()` and default `CompletableFuture` async execution
- Tomcat's and Netty's default thread counts, HikariCP sizing advice, Reactor's default schedulers, and the virtual-thread carrier pool

Two opposite failures:
- **No limit set, so the JVM sees the host's 64 cores** and starts 40+ GC threads and a 63-wide ForkJoinPool inside a container entitled to 1 CPU. Enormous context-switching overhead and memory for thread stacks.
- **`limits.cpu: 1` on a multi-threaded JVM** gives `availableProcessors() == 1`, disabling parallel GC entirely in some configurations and serialising work that should overlap.

The practical guidance is `-XX:ActiveProcessorCount=N` to set it explicitly when the derived value is wrong, and to be deliberate about CPU limits at all — see Q67, where the recommendation is usually to set CPU *requests* but not CPU *limits* for JVM services, precisely because CFS throttling interacts badly with GC and JIT.

**Diagnostic checklist for an OOMKilled pod**

1. `kubectl describe pod` → `Last State: Terminated, Reason: OOMKilled, Exit Code: 137`. Confirms the kernel killed it, not a Java `OutOfMemoryError` (which would produce exit code 1 or 3 and a stack trace in the logs — *the two are completely different failures and are constantly confused*).
2. Run `java -XX:+PrintFlagsFinal -version` inside the container; check `MaxHeapSize` against `limits.memory`.
3. Check the JDK patch version against the host's cgroup version.
4. Enable NMT and diff over time to attribute the growth.
5. Check whether the container writes to `emptyDir` or does heavy file I/O — **page cache counts toward the cgroup memory limit** in cgroups v2 and is a frequently missed cause of OOMKill with a perfectly healthy JVM.

### Probes

**cgroup awareness (`UseContainerSupport`, default since 8u191/10).** Covered, with the verification command.

**`MaxRAMPercentage` vs `-Xmx`.** Covered, including that `-Xmx` overrides it and the related percentage flags.

**Heap is only part of RSS.** Covered, with concrete percentage guidance and why small containers need a lower percentage.

**`MALLOC_ARENA_MAX`.** Covered.

**cgroups v1 vs v2 differences.** Covered, including the specific JDK versions where v2 support landed — the failure mode this probe is really testing for.

**CPU limits → `availableProcessors` → GC/FJP thread counts.** Covered, with the full list of what it drives and both failure directions.

---

## Q64. Why `docker stop` takes 10 seconds and kills the app mid-request.

### Answer

**What `docker stop` does.** It sends **SIGTERM** to PID 1 in the container, waits for `--time` (default **10 seconds**), and if the container is still running sends **SIGKILL**. Kubernetes does the same with `terminationGracePeriodSeconds` (default 30s). So a container that takes exactly 10 seconds to stop is almost always a container that **never received or never handled SIGTERM** and was killed by the timeout.

**Cause 1 — shell-form `CMD`/`ENTRYPOINT` (the usual culprit)**

```dockerfile
ENTRYPOINT java -jar app.jar          # SHELL form
```

Shell form is executed as `/bin/sh -c "java -jar app.jar"`. So **PID 1 is `/bin/sh`**, and the JVM is a child process. `sh` is not an init system: it does not forward signals to its children. Docker sends SIGTERM to `sh`, which either ignores it or exits without telling the JVM, and 10 seconds later SIGKILL kills the whole cgroup — with in-flight requests dropped mid-response and no graceful shutdown at all.

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]   # EXEC form — the JVM IS PID 1
```

Exec form (a JSON array) runs the binary directly with no shell. Verify with `docker exec <c> ps -ef` — you want `java` at PID 1, not `sh`.

The trap: exec form does **no shell processing**, so `ENTRYPOINT ["java", "-jar", "$APP_HOME/app.jar"]` does not expand `$APP_HOME`, and `ENTRYPOINT ["java", "-jar", "app.jar", "$JAVA_OPTS"]` passes a literal string. If you need variable expansion, either use `JAVA_TOOL_OPTIONS` (which the JVM reads from the environment itself, so no shell is required — this is the clean solution), or use `exec` explicitly in a wrapper script:
```sh
#!/bin/sh
exec java $JAVA_OPTS -jar /app/app.jar     # 'exec' REPLACES the shell, so java becomes PID 1
```
The `exec` keyword is the whole point — without it, `sh` stays alive as PID 1 and you have the original bug.

**Cause 2 — PID 1 semantics**

PID 1 is special in Linux: the kernel **does not apply default signal handlers** to it. For any other process, an unhandled SIGTERM terminates it; for PID 1, an unhandled SIGTERM is simply **ignored**. So a process that installs no SIGTERM handler and runs as PID 1 cannot be stopped gracefully at all.

The JVM *does* install handlers for SIGTERM/SIGINT/SIGHUP and runs shutdown hooks, so a JVM at PID 1 behaves correctly. But this bites any wrapper script, and it is why "add a `trap` to your entrypoint script" is necessary if you insist on one.

**Cause 3 — zombie reaping**

PID 1 is also responsible for reaping orphaned children (calling `wait()` on processes whose parent died). A JVM does not do this. If your container spawns subprocesses — a sidecar helper, `ProcessBuilder` calls to an external tool, a script — orphans accumulate as zombie entries in the process table. Rarely fatal for a typical Spring Boot service, but real for containers that fork.

**The fix for both:** run a minimal init as PID 1. `docker run --init` injects **tini**; in a Dockerfile, `ENTRYPOINT ["/tini", "--", "java", "-jar", "app.jar"]`; in Kubernetes, `shareProcessNamespace` or simply designing so the app is PID 1 and forks nothing. Tini forwards signals to its child and reaps zombies, in ~10 KB.

**Cause 4 — the app receives SIGTERM but does not shut down gracefully**

Even with correct signal delivery, the JVM's default behaviour is to run shutdown hooks and exit — it does **not** wait for in-flight HTTP requests unless the server is configured to. In Spring Boot:

```properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=25s
```

`graceful` makes the embedded server stop accepting new connections and wait for active requests to complete, up to the timeout. Then ensure the *container's* grace period exceeds the app's: `terminationGracePeriodSeconds: 45` for a 25s shutdown phase. If the app's timeout is longer than the container's, SIGKILL arrives first and the graceful shutdown is pointless — a very common misconfiguration where both values were set by different people.

Also register shutdown behaviour for everything else: Kafka consumers (commit offsets and leave the group cleanly), scheduled tasks (`ThreadPoolTaskExecutor.setWaitForTasksToCompleteOnShutdown(true)` with `setAwaitTerminationSeconds`), and connection pools.

**Cause 5 (Kubernetes-specific) — the endpoint propagation race**

Even a perfect graceful shutdown drops requests in Kubernetes, because SIGTERM and endpoint removal happen **concurrently**: the kubelet sends SIGTERM at the same moment the endpoints controller begins removing the pod from Service endpoints, and that removal must propagate to every kube-proxy/iptables rule and every ingress controller. For a few seconds after SIGTERM, traffic is still being routed to the dying pod. If the app stops accepting connections immediately, those requests get connection-refused.

The fix is a `preStop` hook that simply waits, letting propagation finish *before* the app begins shutting down:
```yaml
lifecycle:
  preStop:
    exec: { command: ["sh", "-c", "sleep 10"] }
terminationGracePeriodSeconds: 45
```
The `preStop` hook runs **before** SIGTERM is sent, and its duration counts against the grace period. Better still (and shell-free, so it works on distroless), flip the readiness probe to failing first — Spring Boot's `ApplicationShutdownEvent`/availability state does this automatically when graceful shutdown is enabled. See Q39 and Q68 for the full sequence.

### Probes

**PID 1 signal handling.** Covered — the kernel's special-casing of PID 1 and why an unhandled SIGTERM is ignored.

**Shell-form `CMD` spawning a shell that doesn't forward SIGTERM.** Covered, with the `exec` keyword fix and the variable-expansion trade-off.

**Exec form.** Covered, including its limitations and `JAVA_TOOL_OPTIONS` as the clean workaround.

**`tini` / `--init`.** Covered.

**Zombie reaping.** Covered.

**Graceful shutdown window.** Covered — Spring Boot's settings, the requirement that the container grace period exceed the app's, and the Kubernetes endpoint-propagation race with `preStop`.

---

## Q65. `COPY . .` then `RUN mvn package` — what's wrong and what's the cost?

### Answer

```dockerfile
FROM maven:3.9-eclipse-temurin-21
WORKDIR /app
COPY . .                       # everything, on every build
RUN mvn package                # therefore always re-runs
CMD ["java", "-jar", "target/app.jar"]
```

**Problem 1 — cache invalidation on every source change.** The `COPY . .` layer's cache key is a hash of every copied file. Change one character in one `.java` file and that layer is invalidated, which invalidates the `RUN mvn package` layer beneath it. Maven re-resolves and re-downloads the **entire dependency tree** from scratch on every single build, because `~/.m2` lives in an invalidated layer.

The cost is concrete: a typical Spring Boot service has 150–300 transitive jars, 100–250 MB. On CI that is 2–5 minutes of download per build, every build, for a one-line change. Split the copy (Q62) and it becomes seconds. Across a team doing 40 builds a day, this is hours of CI time and real money.

**Problem 2 — build context size.** Before executing anything, the Docker client tars the **entire build directory** and sends it to the daemon. Without a `.dockerignore` that includes:
- `target/` or `build/` — potentially hundreds of MB of prior build output, *including a previous fat jar*
- `.git/` — the full repository history, frequently the largest single item
- `.idea/`, `.vscode/`, `*.iml`
- `node_modules/` if there's a frontend
- Local `.env` files, `*.pem`, `*.p12`, kubeconfigs, `~/.aws`-style credentials that someone dropped in the repo

Two consequences beyond slowness. First, the context is uploaded even for a fully cached build. Second, `target/` in the context means the *previous* build output is present during the new build, which can produce genuinely confusing results (stale classes on the classpath, a jar that appears to build without your changes).

**Problem 3 — secrets baked into layers, and deleting them does not help.** This is the most serious issue and the part the probe is really about.

```dockerfile
COPY . .                              # includes .env, id_rsa, settings.xml with a repo password
RUN mvn package
RUN rm -f .env id_rsa                 # DOES NOT REMOVE THEM
```

Docker images are **stacked, immutable layers**. `rm` in a later layer records a *whiteout* entry that hides the file in the merged filesystem — the bytes remain in the earlier layer, present in the image, pushed to your registry, and extractable by anyone who can pull it:

```bash
docker save myimage:latest | tar -xO --wildcards '*/layer.tar' | tar -tv | grep id_rsa
# or simply: docker history --no-trunc, then extract the layer
```

The same applies to `ENV SECRET=...` and `ARG SECRET=...`, both of which appear in `docker history` and `docker inspect`. **Any secret that ever existed in any layer must be treated as compromised and rotated** — you cannot delete it from a pushed image.

The correct mechanisms:
- **Multi-stage builds:** secrets used in the build stage never reach the runtime stage, because only explicit `COPY --from` artifacts cross over. This alone fixes most cases — but note the *build* stage image can still be pushed or cached, so it is not sufficient on its own for a shared cache.
- **BuildKit secret mounts:** `RUN --mount=type=secret,id=mvn_settings,target=/root/.m2/settings.xml mvn package` — the file exists only for that `RUN`, in a tmpfs, and is never committed to a layer.
- **SSH agent forwarding** for private Git dependencies: `RUN --mount=type=ssh git clone ...`.
- **Runtime secrets** come from Kubernetes Secrets, a CSI driver, or a secrets manager — never from the image.

**Problem 4 — tests and reproducibility.** `RUN mvn package` runs tests inside the image build, which mixes concerns: you cannot publish test reports, cannot cache test results, cannot parallelise, and a flaky test fails your image build. The mainstream practice is to build and test in CI and copy the resulting artifact into a minimal runtime image, or to run tests in a dedicated stage that CI targets explicitly (`--target test`).

**Problem 5 — non-determinism.** `mvn package` with unpinned plugin versions or `SNAPSHOT` dependencies resolves differently over time, so the same commit produces different images. Pin plugin versions, avoid snapshots in release builds, and consider `--fail-at-end`-free, offline-capable builds (`-o` with a pre-populated cache) for reproducibility.

### Probes

**Cache invalidation on every source change.** Covered, with quantified cost.

**Dependency re-download.** Covered.

**Build context size.** Covered, with the specific `.dockerignore` entries and the stale-`target/` hazard.

**Secrets accidentally baked into layers, and that deleting them in a later layer doesn't remove them.** Covered in detail, with the extraction command, the `ENV`/`ARG` variants, the rotation requirement, and the four correct alternatives.

**`--mount=type=cache`.** Covered here and in Q62 — the mechanism that keeps `~/.m2` warm across builds even when the dependency layer is invalidated.

---

# 9. Kubernetes

---

## Q66. Liveness, readiness, and startup probes — and a real failure from getting them wrong.

### Answer

| Probe | Question it answers | Failure action | Runs |
|---|---|---|---|
| **Startup** | "Has the app finished booting?" | Kill the container (restart) | From container start until first success, then never again |
| **Readiness** | "Can this pod serve traffic *right now*?" | Remove the pod from Service endpoints — **no restart** | Continuously, for the pod's whole life |
| **Liveness** | "Is this process broken beyond recovery?" | **Kill and restart the container** | Continuously, after the startup probe succeeds |

The crucial distinction: **readiness is reversible and cheap; liveness is destructive.** A readiness failure temporarily removes one pod from the load balancer and it comes back on its own. A liveness failure destroys the process, losing in-flight requests, its JIT profile, its caches, and its connections.

**The correct semantics for each**

- **Liveness should test only "is this process irrecoverably stuck?"** — a deadlock, an exhausted thread pool, a permanently wedged event loop. In practice, for the vast majority of Spring Boot services, **an empty 200 endpoint is the right liveness probe**. If the JVM can serve an HTTP request at all, restarting it will not help.
- **Readiness should test "can I successfully handle a request?"** — which legitimately includes checking dependencies the pod cannot function without, and reporting not-ready while warming up, while draining, or while a circuit breaker to a critical dependency is open.
- **Startup exists to give slow-booting apps a long grace period without weakening liveness afterwards.** A JVM app taking 60s to start needs `failureThreshold × periodSeconds ≥ 60s` on the startup probe, after which liveness can use a tight 3×10s.

**Spring Boot support:**
```properties
management.endpoint.health.probes.enabled=true      # auto-enabled on Kubernetes
management.health.livenessState.enabled=true
management.health.readinessState.enabled=true
```
This exposes `/actuator/health/liveness` and `/actuator/health/readiness`. The key design point is that Boot's **liveness group contains no health indicators by default** — it reflects only the internal `LivenessState`, which is `BROKEN` only if the application context has failed irrecoverably. The readiness group reflects `ReadinessState` plus, if you add them, `readinessState` group indicators. This default is deliberately correct, and matches the advice above. `AvailabilityChangeEvent.publish(ctx, ReadinessState.REFUSING_TRAFFIC)` lets you flip readiness programmatically.

**The real failure: a database check in the liveness probe**

This is the canonical Kubernetes incident, and it is worth telling as a story.

A team sets `livenessProbe: httpGet /actuator/health` — the *default* aggregate endpoint, which includes the `DataSourceHealthIndicator`. The database has a brief incident: a failover, a connection-pool exhaustion, a network blip, thirty seconds of high latency.

1. Every pod's liveness probe fails simultaneously, because they all share the database.
2. Kubernetes kills **every pod in the deployment at once**.
3. All pods restart. A cold JVM opens a full connection pool, runs Flyway validation, warms caches — a **thundering herd** hitting the already-struggling database far harder than steady-state traffic did.
4. The database, now under a connection storm from 40 restarting pods, degrades further.
5. Liveness fails again. `CrashLoopBackOff`.
6. What would have been a 30-second partial degradation becomes a 20-minute total outage, and the restarts actively prevent recovery.

The diagnosis is unmistakable in hindsight: `kubectl get pods` shows every pod restarting in lockstep with `Restart Count` climbing, and the database's connection count spiking at each restart. **Restarting a process never fixes a dependency.** Liveness must be independent of anything external.

The correct configuration for that team: liveness on `/actuator/health/liveness` (no DB check), readiness on `/actuator/health/readiness` (with the DB check). Then a database outage takes the pods out of the load balancer, they return automatically when it recovers, and nothing restarts. Note the second-order consideration: if *all* pods go unready, the Service has no endpoints and clients get connection failures rather than useful errors — for some services it is better to stay ready and return `503` with a clear error, so you keep serving the requests that don't need the database. That is a genuine design judgement worth raising.

**Other real failure modes**

- **Probe timeouts shorter than actual latency under load.** `timeoutSeconds: 1` (the default!) against an endpoint that takes 1.2s during a GC pause or a traffic spike. Pods flap in and out of the endpoint list, capacity halves, latency rises, more probes time out — a self-amplifying failure. Set `timeoutSeconds` generously (3–5s) and `failureThreshold: 3` so a single blip is tolerated.
- **No startup probe on a slow JVM.** Liveness with `initialDelaySeconds: 30` on an app that takes 90s under contention → killed at 30s, restarts, killed again: an infinite `CrashLoopBackOff` that only manifests under load or on a slow node.
- **A readiness probe that is also expensive.** Called every 10s per pod across 50 pods = 5 requests/sec of health checks. If it queries the database, you've added constant load and made the health check a dependency. Cache the result for a few seconds (Boot's health indicators are cheap by default; custom ones often are not).
- **The probe endpoint requires authentication** — probes carry no credentials, so it returns `401` and every pod is killed. Expose actuator on a separate management port and permit it in the security configuration.
- **Probes hitting the same thread pool as traffic.** Under saturation, health checks queue behind real requests and time out, so Kubernetes kills the pod that is merely *busy*. A separate `management.server.port` gives the probe its own connector and avoids this.

**Recommended baseline:**
```yaml
startupProbe:
  httpGet: { path: /actuator/health/liveness, port: 8081 }
  periodSeconds: 5
  failureThreshold: 30        # up to 150s to start
livenessProbe:
  httpGet: { path: /actuator/health/liveness, port: 8081 }
  periodSeconds: 10
  timeoutSeconds: 3
  failureThreshold: 3         # ~30s of consecutive failure before restart
readinessProbe:
  httpGet: { path: /actuator/health/readiness, port: 8081 }
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 2
  successThreshold: 1
```

### Probes

**Liveness failure = restart, readiness = traffic gating.** Covered, with the "reversible vs destructive" framing.

**Why a DB check in liveness causes cascading restarts.** Covered as a step-by-step incident, with the diagnosis and the correct configuration.

**Startup probe for slow JVM boot so liveness doesn't kill it.** Covered, with the `failureThreshold × periodSeconds` arithmetic.

**Probe timeouts vs actual latency under load.** Covered, including the flapping amplification and the separate-management-port fix.

**`failureThreshold` tuning.** Covered in the recommended baseline, with the reasoning for each value.

---

## Q67. Requests vs limits for CPU and memory — and what happens when you exceed each.

### Answer

**Requests** are for the **scheduler**. `resources.requests` is what the pod is guaranteed; the scheduler finds a node with that much *unallocated* (not unused) capacity and places the pod. Requests are also what the kernel uses for proportional CPU sharing under contention (`cpu.shares` / `cpu.weight`). The sum of requests on a node cannot exceed its allocatable capacity.

**Limits** are for the **kernel**. `resources.limits` is enforced at runtime by cgroups — but *how* differs fundamentally between the two resources, and this is the heart of the question.

**CPU is compressible. Memory is not.**

- **Exceeding a CPU limit → throttling.** CPU limits are enforced by the CFS bandwidth controller: the cgroup is granted `quota` microseconds of CPU per `period` (default 100ms). Once the quota is consumed, **every thread in the container is descheduled until the next period begins**. The process is not killed; it is simply frozen, in increments of up to 100ms.
- **Exceeding a memory limit → OOMKill.** Memory cannot be reclaimed from a process that is using it, so the kernel's cgroup OOM killer sends SIGKILL. Exit code 137, `Reason: OOMKilled`. Kubernetes restarts the container per its `restartPolicy`; repeated kills produce `CrashLoopBackOff`.

**QoS classes**, assigned automatically from requests and limits:

| Class | Condition | Eviction order under node memory pressure |
|---|---|---|
| **Guaranteed** | Every container has `requests == limits` for **both** CPU and memory | Evicted **last** |
| **Burstable** | At least one request is set, but not matching limits everywhere | Evicted second, ordered by how far usage exceeds requests |
| **BestEffort** | No requests or limits at all | Evicted **first** |

Eviction is distinct from OOMKill: the kubelet evicts whole pods when the *node* is under pressure (based on `evictionHard` thresholds), while OOMKill is per-container against its own limit. Guaranteed pods also get `oom_score_adj` values that make the kernel prefer to kill other processes.

**Why CPU limits on a JVM often hurt**

This is the interesting, contested part and where a senior answer distinguishes itself.

Consider `limits.cpu: 1` (100ms quota per 100ms period) on a JVM with 8 GC threads. During a young collection, those 8 threads all run: they consume the entire 100ms quota in **12.5ms of wall time**, and the container is then frozen for the remaining 87.5ms. A GC that should take 15ms takes 100ms+. The same applies to JIT compilation threads during warmup, to `parallelStream()`, and to any burst of parallelism. Your p99 latency shows unexplained ~100ms plateaus.

Worse, the JVM sized its thread pools from `availableProcessors()`, which the container support derives from the *limit* — so the JVM believes it has 1 CPU and starts fewer GC threads, but any library that spawns threads independently (Netty, Tomcat, Reactor, a connection pool) can still burst past the quota.

The measurement that proves it: `container_cpu_cfs_throttled_seconds_total` and `container_cpu_cfs_throttled_periods_total` in cAdvisor/Prometheus. A throttled-periods ratio above a few percent means the limit is actively hurting you. **Alert on this** — it is one of the highest-value and least-configured alerts in a Kubernetes Java estate.

(Historical note worth knowing: a Linux CFS bug causing spurious throttling of multi-threaded workloads even below quota was fixed in kernel 4.18/5.4. Many "CPU limits are broken" articles predate the fix. The remaining behaviour is not a bug — it is the quota working as designed against bursty parallelism.)

**The resulting recommendation**, which is now mainstream practice for JVM services:

- **Always set memory requests == memory limits.** Memory is incompressible; a pod using more than its request can be OOMKilled or evicted unpredictably, and a JVM's footprint is fairly stable and knowable. Equal values also give you predictable `MaxRAMPercentage` sizing and Guaranteed QoS.
- **Set CPU requests, and usually omit CPU limits.** The request guarantees your share under contention; omitting the limit lets the pod burst into idle node capacity for GC, JIT warmup, and traffic spikes, at no cost to anyone (CFS shares still enforce fairness when the node is busy).
- **The counter-argument, which is legitimate:** without CPU limits, one misbehaving pod can consume a node's spare capacity and make neighbours' latency unpredictable; multi-tenant clusters, chargeback models, and strict performance-isolation requirements may justify limits. If you set them, set them generously (2–4× the request, not equal to it) and monitor throttling. Also note that you cannot get **Guaranteed** QoS without CPU limits, so if eviction protection matters more than burst capacity, that changes the calculus.
- Use a **VerticalPodAutoscaler in recommendation mode**, or historical usage percentiles, to set requests from data rather than guesswork. Typical failure: requests set from a laptop benchmark, then 3× over-provisioned across 500 pods.
- Consider `LimitRange` (namespace defaults so nobody ships a BestEffort pod) and `ResourceQuota` (namespace-level caps).

**Memory sizing specifically for the JVM:** set `limits.memory` to heap + non-heap overhead with headroom (Q63), set `-XX:MaxRAMPercentage` accordingly, and remember that **page cache from file I/O counts toward the cgroup v2 memory limit**, so a container writing logs or temp files to disk can be OOMKilled with an entirely healthy JVM.

### Probes

**Scheduling vs enforcement.** Covered — requests are scheduler input plus CPU shares; limits are kernel enforcement.

**CPU throttling (CFS quota) vs memory OOMKill.** Covered, with the compressible/incompressible framing and the exact CFS mechanics.

**QoS classes and eviction order.** Covered in the table, with the eviction-vs-OOMKill distinction.

**Why CPU limits on a JVM often hurt.** Covered in detail with the GC arithmetic, the resulting recommendation, and the honest counter-argument.

**`throttled_seconds` metric.** Covered, including the historical CFS bug and the recommendation to alert on the throttled-periods ratio.

---

## Q68. Rolling update — achieving truly zero dropped requests.

### Answer

**What Kubernetes does by default.** A `Deployment` with `strategy: RollingUpdate` creates a new ReplicaSet and shifts replicas across, governed by:
- **`maxSurge`** (default 25%) — how many pods above the desired count may exist during the rollout. Higher = faster, more capacity needed.
- **`maxUnavailable`** (default 25%) — how many pods below the desired count may be unavailable. **Set this to 0** if you cannot afford reduced capacity: combined with `maxSurge: 1` (or 25%), it means "always add a new ready pod before removing an old one".

A pod counts as available only after its **readiness probe passes** and it has been ready for `minReadySeconds`. So readiness gating is what prevents the rollout from replacing all your working pods with broken ones — if the new version fails its readiness probe, the rollout stalls (and `progressDeadlineSeconds`, default 600s, eventually marks it failed) rather than taking down the service. This is the single most valuable safety property of a rolling update, and it only works if your readiness probe actually tests something meaningful.

**Why requests still get dropped, and the fixes**

Even with correct surge settings, there are four distinct failure points.

**1. New pods receive traffic before they can serve it.** A JVM that binds its port at t=20s but has not warmed its connection pool, filled its caches, or JIT-compiled anything will serve the first requests slowly or fail them. Fix: the readiness probe must reflect genuine readiness — after the datasource is initialised, after Flyway completes, after any cache warmup — and use `minReadySeconds: 10–30` so a pod that passes readiness and immediately breaks doesn't cause the rollout to proceed. For JVM warmup specifically, consider a startup warmup routine that issues synthetic requests before reporting ready, and be aware that a fresh pod receiving 1/Nth of production traffic instantly will have a worse p99 for the first minute or two.

**2. The endpoint propagation race — the big one.** When a pod is deleted, two things happen **in parallel**:
- The kubelet sends SIGTERM to the container.
- The endpoints controller removes the pod from the `EndpointSlice`, which must then propagate to every kube-proxy on every node (rewriting iptables/IPVS rules) *and* to every ingress controller, service mesh sidecar, and client-side load balancer.

Propagation typically takes hundreds of milliseconds to several seconds, and there is **no ordering guarantee**. So for a window after SIGTERM, traffic is still arriving at a pod that has begun shutting down. If the app closes its listener on SIGTERM, those connections are refused.

The fix is to make the app keep serving for longer than propagation takes:
```yaml
lifecycle:
  preStop:
    exec: { command: ["sh", "-c", "sleep 10"] }   # or use a readiness flip
terminationGracePeriodSeconds: 45
```
`preStop` runs **before** SIGTERM. During that sleep the pod is fully functional while its endpoint is being withdrawn everywhere. Ten seconds is a common, safe value. On a distroless image with no shell, use `httpGet` to an endpoint that sleeps, or rely on Spring Boot's readiness-first shutdown: with `server.shutdown=graceful`, Boot publishes `ReadinessState.REFUSING_TRAFFIC` **before** it stops accepting connections, so the readiness probe fails while the server still serves — achieving the same effect without a sleep, provided your readiness probe period is short enough (`periodSeconds: 5`, `failureThreshold: 1–2`).

**3. In-flight requests are cut off.** Covered in Q39/Q64: `server.shutdown=graceful` plus `spring.lifecycle.timeout-per-shutdown-phase`, with `terminationGracePeriodSeconds` strictly greater than the app's total shutdown time (`preStop` duration + graceful timeout + margin). Get the inequality backwards and SIGKILL arrives mid-request.

**4. Connection draining at the L7 layer.** Keep-alive connections are the subtlety people miss. An ingress controller or a client with an established HTTP/1.1 keep-alive or HTTP/2 connection to a terminating pod will **keep using it** — endpoint removal does not close existing connections. Two mitigations: ensure the server sends `Connection: close` on responses during shutdown (Tomcat does this when gracefully shutting down), and configure the ingress/mesh for proper draining (NGINX Ingress `worker-shutdown-timeout`, Envoy/Istio drain settings, an AWS target group `deregistration_delay` that matches your grace period). For gRPC, send `GOAWAY` before closing.

**PodDisruptionBudget — for the disruptions a Deployment doesn't control**

A rolling update is a *voluntary* disruption initiated by the Deployment controller, which already respects `maxUnavailable`. A PDB protects against **other** voluntary disruptions: `kubectl drain` during a node upgrade, cluster autoscaler scale-down, node pool rotation.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: 80%          # or maxUnavailable: 1
  selector: { matchLabels: { app: orders } }
```

Without one, a node drain can evict every replica of a 2-replica deployment simultaneously — an outage caused by routine maintenance. Two practical warnings: a PDB with `minAvailable` equal to `replicas` **blocks node drains forever**, which will make your platform team unhappy and can wedge a cluster upgrade; and a PDB does **not** protect against involuntary disruptions (node crash, kernel OOM, preemption) — for those you need enough replicas and `topologySpreadConstraints` to spread them across nodes and zones.

**Putting it together — the complete configuration:**
```yaml
spec:
  replicas: 6
  minReadySeconds: 15
  progressDeadlineSeconds: 600
  strategy:
    rollingUpdate: { maxSurge: 2, maxUnavailable: 0 }
  template:
    spec:
      terminationGracePeriodSeconds: 45
      containers:
      - name: app
        lifecycle:
          preStop: { exec: { command: ["sh","-c","sleep 10"] } }
        readinessProbe: { httpGet: {path: /actuator/health/readiness, port: 8081},
                          periodSeconds: 5, failureThreshold: 2 }
```
plus `server.shutdown=graceful`, `spring.lifecycle.timeout-per-shutdown-phase=25s`.

**Verify it, don't assume it.** The only way to know you have zero dropped requests is to run a continuous load generator (`vegeta`, `k6`, `hey`) at realistic RPS *through the real ingress path* during a deployment and assert zero non-2xx responses. Every one of the four failure points above is invisible at low traffic and obvious under load. Make this a standing pre-production test.

### Probes

**`maxSurge` / `maxUnavailable`.** Covered, with the `maxUnavailable: 0` recommendation.

**Readiness gating new pods.** Covered, including `minReadySeconds`, `progressDeadlineSeconds`, and JVM warmup.

**The endpoint-propagation race → `preStop` sleep.** Covered in detail, with both the sleep and the readiness-flip approaches.

**`terminationGracePeriodSeconds` > app shutdown time.** Covered, with the explicit inequality.

**Connection draining.** Covered — keep-alive connections surviving endpoint removal, `Connection: close`, ingress/mesh/ALB draining settings, and gRPC `GOAWAY`.

**PodDisruptionBudget for voluntary disruptions.** Covered, including the distinction from involuntary disruption and the "PDB that blocks drains forever" trap.

---

## Q69. Deployment vs StatefulSet vs DaemonSet vs Job — pick one for five workloads.

### Answer

**The controllers**

- **Deployment** — manages a ReplicaSet of **interchangeable, fungible** pods. Random name suffixes, random start/stop order, shared or no storage, replaced freely. The default for anything stateless.
- **StatefulSet** — pods have **stable network identity** (`app-0`, `app-1`, ordinal-indexed, with stable DNS via a headless Service) and **stable per-pod storage** (each gets its own PVC from `volumeClaimTemplates`, which survives pod rescheduling and, by default, StatefulSet deletion). Creation is ordered (0, then 1, then 2, each waiting for the previous to be Ready) and deletion is reverse-ordered. Updates roll in reverse ordinal order with `partition` support for staged rollouts.
- **DaemonSet** — exactly one pod per node (matching a node selector), automatically added when a node joins. For node-level agents.
- **Job** — runs pods to **completion**. `completions` (how many must succeed), `parallelism`, `backoffLimit`, `activeDeadlineSeconds`. **CronJob** wraps it on a schedule.

**The five workloads**

**1. A stateless API → Deployment.** Nothing to preserve between restarts, replicas are identical, ordering is irrelevant. Anything else adds constraints for no benefit.

**2. A Kafka consumer group → Deployment.** This is the trap in the question, and the reason it's asked.

The intuition "consumers are stateful, so StatefulSet" is wrong. A Kafka consumer's state — its **committed offsets** — lives in the `__consumer_offsets` topic *in the broker*, not on the pod. Partition assignment is negotiated by the group coordinator at runtime based on group membership, not on pod identity. A consumer pod is therefore completely fungible: kill `consumer-xyz`, start `consumer-abc`, it joins the group, gets partitions assigned, and resumes from committed offsets. No stable identity, no persistent volume, no ordering requirement. **Deployment.**

The consequences that *do* need attention are unrelated to the controller choice: graceful shutdown so the consumer commits offsets and leaves the group cleanly (avoiding a `session.timeout.ms` wait before rebalance), scaling capped by partition count, and rebalancing behaviour (Q82). One legitimate reason to reach for a StatefulSet is **static group membership** (`group.instance.id`), which benefits from stable pod names — but you can derive that from the pod name via the downward API in a Deployment too, and Kafka Streams with local RocksDB state stores *is* a genuine StatefulSet case, because rebuilding a large local state store from the changelog topic on every restart is expensive. Say that: "Deployment for a plain consumer; StatefulSet for Kafka Streams with local state, so the RocksDB directory survives a restart."

**3. Kafka itself → StatefulSet.** Every requirement lines up: each broker has a **stable identity** (`broker-0` must remain broker ID 0, because partition replica assignments reference broker IDs), **stable storage** (the log directories are the data — losing them means a full replica re-sync of potentially terabytes), **stable DNS** (`kafka-0.kafka-headless.ns.svc` for the advertised listener, so clients and peers can reach a specific broker), and **ordered startup** for controller election. Same reasoning for ZooKeeper/KRaft controllers, Cassandra, Elasticsearch, PostgreSQL, MongoDB replica sets.

The honest caveat: **prefer a managed service or a mature Operator** (Strimzi for Kafka, CloudNativePG for Postgres) over hand-rolling a StatefulSet. StatefulSets provide identity and storage but nothing else — no leader election, no backup, no safe rolling restart that respects in-sync replicas, no rebalancing. Running a database on raw StatefulSets is a well-known way to lose data.

**4. A log shipper → DaemonSet.** It must read `/var/log/containers` on **every** node, so it needs exactly one instance per node, with `hostPath` mounts and typically host networking and elevated privileges. New node joins → agent appears automatically. Same for node exporters, CNI plugins, security agents, and CSI node plugins. Tolerations for control-plane/tainted nodes are usually required, and `priorityClassName: system-node-critical` so it isn't evicted.

**5. A nightly reconciliation → CronJob.** Runs on a schedule, terminates, must not be restarted forever.

**CronJob concurrency policy and missed runs** — the details this probe targets:

```yaml
spec:
  schedule: "0 2 * * *"
  timeZone: "Europe/Warsaw"        # stable from Kubernetes 1.27; before that, UTC only
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 300
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 3600
```

- **`concurrencyPolicy`**: `Allow` (default — a long-running job overlaps with the next scheduled run, which for a reconciliation job usually means two jobs fighting over the same rows), `Forbid` (skip the new run if the previous is still going), `Replace` (kill the old, start the new). For reconciliation, **`Forbid`**, plus an alert on skipped runs so a job that has silently grown past its interval doesn't go unnoticed.
- **`startingDeadlineSeconds`**: if the controller was down (control-plane upgrade, a paused cluster) and a scheduled time was missed, the job starts late only if within this deadline. **The critical trap:** if you leave it unset and more than **100** start times are missed, the CronJob controller gives up and logs "Cannot determine if job needs to be started: too many missed start times" — and **stops scheduling entirely, permanently**, until you edit the object. Always set `startingDeadlineSeconds` (to less than the interval, so at most one catch-up run occurs).
- **Missed runs are not queued.** A CronJob does not "make up" runs. If the semantics require every period to be processed, the job must derive its work window from persisted state (a watermark table), not from the assumption that it runs exactly once per period.
- **Jobs are at-least-once, not exactly-once.** A node failure can leave a Job pod partially complete and then rerun it, so the job body must be **idempotent** and ideally resumable. `backoffLimit` bounds retries; `activeDeadlineSeconds` bounds total runtime (which prevents a hung job from blocking a `Forbid` policy forever); `ttlSecondsAfterFinished` cleans up finished Job objects.
- For a *single* leader-elected task rather than a scheduled one, consider ShedLock or a database advisory lock inside a normal Deployment instead — a common alternative when the task needs the application's warm context.

### Probes

**Stable identity and ordered start/stop.** Covered under StatefulSet.

**Per-pod PVCs.** Covered, including that they survive pod and StatefulSet deletion.

**Headless services.** Covered, with the Kafka advertised-listener use case.

**Why a Kafka *consumer* is still a Deployment.** Covered in detail — offsets live in the broker, pods are fungible — plus the two legitimate exceptions (static membership, Kafka Streams with local state).

**CronJob concurrency policy and missed-run handling.** Covered — all three policies with a recommendation, `startingDeadlineSeconds` including the 100-missed-runs permanent-failure trap, the fact that runs are not queued, and the at-least-once/idempotency requirement.

---

## Q70. How HPA decides to scale, and why it might not help a Java service.

### Answer

**The algorithm.** Every 15 seconds (`--horizontal-pod-autoscaler-sync-period`), the HPA controller fetches the metric for every ready pod and computes:

```
desiredReplicas = ceil( currentReplicas × ( currentMetricValue / desiredMetricValue ) )
```

For `Resource` metrics with `averageUtilization`, `currentMetricValue` is the mean across pods of **usage ÷ request** — note that it is a percentage of the **request**, not of the limit. So a pod requesting 500m and using 400m is at 80%, regardless of any limit.

Details that matter:
- **Tolerance:** if the ratio is within **10%** of 1.0, no action is taken. This prevents constant ±1 replica churn.
- Pods that are not `Ready`, or whose metrics are unavailable, are excluded; pods still starting up are treated conservatively (assumed at 0% for scale-up decisions, 100% for scale-down) to avoid reacting to warmup.
- Metrics come from **metrics-server** (Resource metrics) or from a **custom/external metrics adapter** (Prometheus Adapter, KEDA). Metrics-server itself scrapes on a ~15s interval and reports a windowed average, so there is inherent latency of tens of seconds before the HPA even sees a change.

**Stabilisation and behaviour policies** (`autoscaling/v2`):

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300     # default: use the HIGHEST recommendation from the last 5 min
    policies: [{ type: Percent, value: 50, periodSeconds: 60 }]
  scaleUp:
    stabilizationWindowSeconds: 0       # default: react immediately
    policies:
      - { type: Percent, value: 100, periodSeconds: 30 }   # may double
      - { type: Pods,    value: 4,   periodSeconds: 30 }
    selectPolicy: Max
```

The asymmetry is deliberate: **scale up fast, scale down slowly.** The 300-second scale-down stabilisation window means the HPA takes the *maximum* recommendation over the past 5 minutes, so a brief dip in load does not remove pods you will need again in 30 seconds (which would cause thrashing and, for a JVM, repeated cold starts).

**Why it might not help a Java service**

**1. You are scaling on the wrong signal.** CPU is the default because it is available, not because it is the constraint. For a typical Spring Boot service the bottleneck is usually elsewhere:
- **Database connection pool saturation.** 20 connections, 200 threads waiting. CPU is at 15% because everything is blocked on I/O. HPA sees low CPU and does nothing — and if it *did* scale, adding pods would add more connections to an already-saturated database and make it worse. **Scaling out is actively harmful when the bottleneck is a shared downstream resource.** This is the most important point in the answer.
- **Downstream latency.** A slow dependency inflates your latency with no CPU cost. More replicas send more concurrent requests to the struggling dependency.
- **Lock contention or a saturated thread pool** — again, low CPU, high latency.

The fix is to scale on the metric that actually reflects saturation: request queue depth, active threads vs pool size, `hikaricp_connections_pending`, in-flight requests, or p99 latency — exposed via Micrometer, scraped by Prometheus, and surfaced through the Prometheus Adapter as a custom metric. For queue workers, **KEDA scaling on Kafka consumer lag** or SQS queue depth is far more direct and is the standard answer for event-driven workloads (it also supports scale-to-zero, which the HPA alone does not).

**2. JVM warmup means new pods are slow.** A fresh JVM runs interpreted, then C1, then C2; it has an empty connection pool, cold caches, unresolved DNS, and no JIT profile. For the first 30–120 seconds a new pod's p99 may be 5–10× steady state. Consequences:
- Scaling up *during* a latency incident briefly makes latency **worse**, because the load balancer immediately sends 1/Nth of traffic to a cold pod.
- The HPA may see the elevated CPU of warming pods and scale up further — a feedback loop.
- Mitigations: `minReadySeconds` and a warmup routine before reporting ready; a slow-start / gradual traffic ramp at the load balancer (Istio and some ingress controllers support it); Class Data Sharing (`-XX:SharedArchiveFile`, or AppCDS) to cut class-loading time; and where startup dominates, **GraalVM native image** or CRaC/checkpoint-restore. Also consider over-provisioning slightly and scaling on a leading indicator rather than a lagging one.

**3. Reaction time vs traffic shape.** Metrics latency (~15–30s) + HPA period (15s) + pod scheduling + image pull + JVM startup (30–90s) + readiness = **one to three minutes** from load arriving to new capacity serving. For a flash-sale or a scheduled batch trigger, that is far too slow. Answers: pre-scale with a scheduled HPA `minReplicas` change or KEDA cron scaler; keep headroom; use **pause pods**/`priorityClassName` over-provisioning so the cluster autoscaler doesn't also need to add a node (which adds minutes more); and understand that HPA handles *gradual* load changes well and *step* changes poorly.

**4. `minReplicas` and the thundering herd.** Scaling from 2 to 20 pods simultaneously means 18 JVMs starting at once: 18 connection pools opening, 18 Flyway validations, 18 config-server fetches, 18 service-registry registrations. This can overwhelm the very dependencies you are scaling to protect. Use `behavior.scaleUp` policies to bound the rate, and make startup work cheap and staggered.

**5. Interactions to be aware of.** The HPA and a **VPA** in `Auto` mode must not both target CPU/memory on the same workload — they will fight. Scaling below the **PDB's** `minAvailable` is blocked. If the metrics API is unavailable, the HPA holds its current count (fails static, which is the safe behaviour). And `replicas` in your Deployment manifest will fight the HPA on every GitOps sync unless you omit the field or configure the sync to ignore it — a very common operational annoyance.

**Baseline recommendation:** scale on a saturation metric relevant to the workload (connection pool pending, queue lag, or in-flight requests) with CPU as a secondary safety net; `minReplicas` at least 2–3 for availability; a generous `scaleDown` stabilisation window; bounded `scaleUp` policies; and verify the whole loop with a load test that actually triggers scaling before you rely on it.

### Probes

**Metric averaging window.** Covered — metrics-server scrape interval, HPA sync period, and the resulting total reaction latency.

**Stabilisation / cooldown.** Covered, with the asymmetric defaults and the reasoning.

**Scaling on CPU when the bottleneck is DB connections or downstream latency.** Covered as the primary failure mode, including why scaling out can make it worse.

**Custom/external metrics (KEDA on Kafka lag).** Covered, with scale-to-zero noted.

**JVM warmup meaning new pods are slow.** Covered, with the feedback-loop hazard and five concrete mitigations.

**`minReplicas` and thundering herd.** Covered.

---

## Q71. A pod is in `CrashLoopBackOff` — triage order.

### Answer

`CrashLoopBackOff` is not itself an error; it is Kubernetes saying "this container keeps exiting and I am backing off before restarting it again" (10s, 20s, 40s… capped at 5 minutes). The real information is *why* it exited.

**1. `kubectl describe pod <name>` — events and the last state.**

```
Last State:  Terminated
  Reason:    OOMKilled          Exit Code: 137
Events:
  Warning  Failed    ...  Error: ImagePullBackOff
  Warning  Unhealthy ...  Liveness probe failed: HTTP probe failed with statuscode: 503
```

Read `Reason`, `Exit Code`, and the **Events** section at the bottom — events explain scheduling, image pulls, volume mounts, and probe failures that never produce application logs.

**Exit codes:**

| Code | Meaning |
|---|---|
| **0** | Exited successfully — but the container was expected to run forever. A batch command in a Deployment, or a `main` that returned. |
| **1** | Generic application error. Usually an uncaught exception at startup; check the logs. |
| **125–127** | Container runtime error, entrypoint not executable, or command not found. Usually a Dockerfile problem or an architecture mismatch (arm64 image on amd64 → `exec format error`). |
| **137** | 128 + 9 = **SIGKILL**. Almost always **OOMKilled** (confirm via `Reason`), or a liveness kill whose grace period expired. |
| **143** | 128 + 15 = **SIGTERM**. Something asked it to stop — liveness failure, eviction, or a rollout. |
| **139** | 128 + 11 = SIGSEGV. A JVM crash; look for `hs_err_pid*.log`, usually a native library. |

**2. `kubectl logs <pod> --previous`.** The `--previous` flag is essential — without it you see the *current* (possibly still-starting) container, while the crash you care about is in the previous instance. Use `-c <container>` for multi-container pods. An **empty** log means the process died before logging initialised, which points at the entrypoint, a missing file, or a bad JVM flag (`Unrecognized VM option` prints to stderr and exits immediately).

**3. Work through the common causes, roughly by frequency:**

- **OOMKilled** (137) → Q63. Check `limits.memory` against `-XX:MaxRAMPercentage`, check the JDK's cgroup v2 detection, check for page-cache usage from file I/O. Note again: this is **not** a Java `OutOfMemoryError`, which appears in the logs with a stack trace and exits with code 1 (or 3 under `ExitOnOutOfMemoryError`). Confusing the two sends people down the wrong path constantly.
- **Application startup exception** (1) → visible in the logs: a failed `ApplicationContext` (`UnsatisfiedDependencyException`, missing bean), a failed Flyway migration, an unreachable config server, malformed `application.yml`, or a port conflict.
- **Missing config or secret** → often the container never starts at all: `CreateContainerConfigError` with `secret "db-credentials" not found` or `couldn't find key password in Secret`. Visible only in `describe`, never in logs.
- **Image problems** → `ImagePullBackOff` / `ErrImagePull`: wrong tag or registry, missing `imagePullSecrets`, registry rate limiting (anonymous Docker Hub limits are a classic), or the architecture mismatch above.
- **Failing probes** → `Liveness probe failed` events, with restarts on a fixed cadence matching `periodSeconds × failureThreshold`. Q66: a probe that is too aggressive, times out under load, hits an authenticated endpoint, or checks an external dependency. Distinguish from a genuine crash by whether the app logs a clean startup before dying.
- **Init container failure** → `Init:CrashLoopBackOff` / `Init:Error`; the main container never runs. `kubectl logs <pod> -c <init-container>`. Common causes: a migration init container failing, or a wait-for-dependency loop timing out.
- **Volume mount failures** → `FailedMount`, a PVC stuck `Pending` (no matching StorageClass, no capacity), or a ReadWriteOnce PVC already attached to a pod on another node.
- **Node pressure / eviction** → `Evicted` with `The node was low on resource: ephemeral-storage` — frequently caused by unrotated logs or heap dumps written to the container filesystem — or memory.
- **Permissions** → `readOnlyRootFilesystem: true` with an app that writes to `/tmp` (mount an `emptyDir`), or `runAsNonRoot` with an image whose `USER` is root or numerically unspecified.
- **Admission rejection** → PodSecurityAdmission or a validating webhook denying the pod; visible in events only.

**4. Tools when the above isn't enough:**

- `kubectl get events --sort-by=.lastTimestamp -n <ns>` — cluster-wide context (node conditions, scheduler messages) that `describe pod` can miss.
- `kubectl debug -it <pod> --image=busybox --target=<container>` — an **ephemeral container** sharing the target's namespaces. Indispensable for distroless images with no shell.
- `kubectl debug <pod> --copy-to=debug --set-image=*=<image>` with the command overridden to `sleep infinity`, so you can inspect the filesystem, environment, and mounts on a container that would otherwise crash.
- `kubectl get pod -o jsonpath='{.spec.containers[0].command}'` — a `command`/`args` override in the manifest silently replacing the image's `ENTRYPOINT` is a surprisingly frequent cause.
- Compare against a working replica in another environment: `kubectl exec` and diff the environment and mounted config.

**5. Prevent recurrence.** A `CrashLoopBackOff` reaching production is usually environment-specific (config, secrets, limits, network policy) rather than a code defect — so the durable fix belongs in the pipeline: validate manifests, run the image in a staging namespace with production-shaped limits, and alert on `kube_pod_container_status_restarts_total` increasing rather than waiting for a human to notice.

### Probes

**`describe` for events/exit code.** Covered, with a full exit-code table.

**`logs --previous`.** Covered, including why it is required and what an empty log implies.

**Exit 137 (OOMKill) vs 143 (SIGTERM) vs 1.** Covered, plus 0, 125–127, and 139, and the OOMKill-vs-`OutOfMemoryError` distinction.

**Image pull errors.** Covered — auth, rate limits, architecture mismatch.

**Failing probes.** Covered, with the cadence signature that identifies them.

**Missing config/secret.** Covered, including that it surfaces as `CreateContainerConfigError` in events rather than in logs.

**Init container failure.** Covered.

**Node pressure.** Covered, including ephemeral-storage eviction caused by heap dumps and unrotated logs.

---

## Q72. How config reaches the app, and rolling out a config change.

### Answer

**The mechanisms**

**1. ConfigMap/Secret as environment variables**
```yaml
envFrom:
  - configMapRef: { name: orders-config }
  - secretRef:    { name: orders-secrets }
```
*Pros:* simplest possible; maps directly onto Spring's relaxed binding (`SPRING_DATASOURCE_URL` → `spring.datasource.url`), so no application code is needed. High precedence.
*Cons:* **environment variables are snapshotted at process start and can never change in a running process.** Updating the ConfigMap has no effect until the pod restarts. They are also exposed in `kubectl describe pod`, in `/proc/<pid>/environ`, in crash dumps, and are inherited by child processes — so env vars are a poor place for secrets.

**2. ConfigMap/Secret as a mounted volume**
```yaml
volumes:
  - name: config
    configMap: { name: orders-config }
volumeMounts:
  - { name: config, mountPath: /config, readOnly: true }
```
*Pros:* the kubelet **updates the files in place** when the ConfigMap changes, via an atomic symlink swap so a reader never sees a partial file — typically within about a minute (sync period plus cache TTL). Not exposed in the process environment. Spring Boot picks up `/config/application.yml` automatically, since it is in the default `spring.config.location` search path.
*Cons:* **the file changing does not reconfigure the application** — Spring binds at startup, so a changed file is inert unless something re-reads it. And **`subPath` mounts are never updated**, which is an extremely common gotcha when mounting a single key into an existing directory.

**3. Command-line args / `JAVA_TOOL_OPTIONS`** — highest precedence, immutable for the process lifetime, the right place for JVM flags.

**4. External config servers** — Spring Cloud Config Server, Consul, or Spring Cloud Kubernetes, which can watch ConfigMaps and trigger a refresh event.

**5. Secrets managers** — Vault, AWS Secrets Manager, GCP Secret Manager, Azure Key Vault — surfaced either through the **Secrets Store CSI Driver** (mounts them as files and supports rotation) or the **External Secrets Operator** (syncs them into native Kubernetes Secrets). This is the correct answer for anything genuinely sensitive, because native Secrets are only **base64-encoded, not encrypted**, unless etcd encryption-at-rest is enabled — and even then, anyone with `get secrets` RBAC in the namespace can read them in plaintext.

**Rolling out a config change**

**The default problem:** `kubectl apply` on a ConfigMap updates the object and **restarts nothing**. Env-var consumers keep the old values indefinitely; volume consumers receive new files that nothing reads. The change silently does not take effect — and worse, it *will* take effect at the next unrelated deployment, potentially weeks later, completely decoupling cause from effect during an incident.

**Option A — force a rollout with a checksum annotation (the standard approach).**
```yaml
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```
Changing the ConfigMap changes the checksum, which changes the pod template, which triggers a normal rolling update — with readiness gating, `maxUnavailable`, `kubectl rollout undo`, and a clear audit trail. **Kustomize** does this automatically via `configMapGenerator`, which appends a content hash to the ConfigMap *name* (`orders-config-7d9f2k`). That variant is arguably better: old pods keep referencing the old, still-existing ConfigMap throughout the rollout, so there is never a window where new and old pods read different content from the same name.

**Option B — immutable ConfigMaps.** `immutable: true` (stable since 1.21) forbids updates entirely, forcing the name-versioning discipline of Option A and letting the kubelet stop watching the object (a real performance win at scale).

**Option C — `kubectl rollout restart deployment/orders`.** Operationally fine, but it is a side-channel change that GitOps will not reproduce.

**Option D — live refresh without a restart.** Spring Cloud's `@RefreshScope` plus `/actuator/refresh` (or Spring Cloud Bus for fan-out) re-binds annotated beans from the current `Environment`; Spring Cloud Kubernetes can watch ConfigMaps and fire it automatically.

**Why `@RefreshScope` is only a partial solution** — the substance of this probe:
- Only `@RefreshScope`-annotated beans and `@ConfigurationProperties` beans are re-bound. `@Value` fields in ordinary singletons are **not**, unless the bean itself is refresh-scoped. So a config change applies to some of your code and not the rest — the worst possible outcome.
- Many things cannot be re-bound at all: the server port, the `DataSource` (changing the URL needs the pool drained and recreated; already-checked-out connections still point at the old target), Kafka consumer config, thread pool sizes, the security filter chain, and anything a library captured at startup.
- Refresh-scoped beans are proxied and recreated lazily on next access, producing a latency spike and a window in which concurrent in-flight requests observe *different* configuration — a genuine consistency hazard for a pricing parameter or a threshold.
- It creates **configuration drift**: a running pod's live config no longer matches what a freshly started pod would receive, so restarting one pod silently changes its behaviour relative to its peers, and incidents become very hard to reason about.

**Recommendation.** Treat configuration as **immutable and versioned with the deployment** — checksum annotations or Kustomize hash-suffixed ConfigMaps, accepting a rolling restart. It is predictable, auditable, and rolls back cleanly, and a rolling restart is cheap once graceful shutdown works (Q68). Reserve live refresh for the narrow cases where a restart is genuinely too expensive.

**And separate the two concepts.** Configuration that changes with a release belongs in the deployment artifact. Values that must change *at runtime, frequently, without a deploy* — kill switches, rollout percentages, rate limits — belong in a **feature-flag system** (Unleash, Flagsmith, LaunchDarkly) built for exactly that, with targeting, audit history, and instant rollback. Most "we need hot config reload" requests are really feature-flag requirements, and answering them with ConfigMaps is the underlying design error.

### Probes

**ConfigMap/Secret as env (needs restart) vs volume (updates, but Spring won't reload without refresh).** Covered, including the atomic symlink swap, the ~1-minute propagation delay, and the `subPath` exception.

**Checksum annotation to force a rollout.** Covered, plus the Kustomize name-hash variant and why it avoids a mixed-read window.

**External secret managers.** Covered — CSI driver vs External Secrets Operator, and why native Secrets are not encrypted.

**Why `@RefreshScope` is a partial solution.** Covered in detail — what is and isn't re-bound, the mid-flight inconsistency window, and configuration drift — with the recommendation to prefer immutable config plus a rolling restart, and to use feature flags for genuinely dynamic values.

---

# 10. Design Patterns & Architecture

---

## Q73. A pattern you *removed* because it was the wrong abstraction.

### Answer

This question tests judgement, not recall. There is no correct pattern to name — the interviewer is checking whether you can recognise that abstractions have costs, and whether you have enough production scar tissue to have seen one fail. Prepare a real example from your own work; the structure below is what a strong answer contains.

**Structure:** what existed → what it cost → what triggered the removal → what replaced it → how you knew it was better.

**Example 1 — a Strategy interface with one implementation.**

A `PricingStrategy` interface, a `DefaultPricingStrategy`, a `PricingStrategyFactory`, and a Spring configuration wiring them. Written "because we might have regional pricing later". Eighteen months later there was still exactly one implementation, and every developer touching pricing navigated three files to find twelve lines of logic. Debugging meant stepping through a factory to discover the only possible answer. I inlined it into a `PricingService` method. When regional pricing eventually arrived, it needed a completely different shape anyway — the speculative abstraction had guessed wrong about the axis of variation, which is what speculative abstractions almost always do.

**Example 2 — a generic repository/DAO layer over Spring Data.**

A hand-written `BaseRepository<T, ID>` with `findAll`, `findBy(Map<String,Object> criteria)`, and a criteria-builder, wrapping Spring Data JPA "so we could swap the persistence layer". It produced untyped, unreviewable queries; it made N+1 problems invisible because you could not see the fetch strategy at the call site; and nobody ever swapped the persistence layer (nobody ever does). Replaced with plain Spring Data repositories with explicit, named query methods and `@EntityGraph` annotations. Query intent became visible at the declaration, and the N+1s surfaced immediately.

**Example 3 — an event bus for in-process decoupling.**

Domain logic communicating through `ApplicationEventPublisher` "to keep modules decoupled". The result: no call graph. You could not answer "what happens when an order is placed?" without grepping for every `@EventListener`. Stack traces were useless. Ordering was implicit and undocumented. Exceptions in async listeners vanished. I replaced most of it with direct method calls on an explicit `OrderPlacedHandler`, keeping events only where the coupling genuinely crossed a bounded context and asynchrony was wanted. Readability improved dramatically and one long-standing "sometimes the email doesn't send" bug turned out to be a swallowed exception in an async listener.

**The general principle to articulate:** an abstraction is a **bet that a specific axis of variation will matter**. If the bet is wrong — and speculative bets usually are — you pay indirection, navigation cost, and debugging cost forever and receive nothing. **Rule of three**: don't abstract until you have three real cases, because two points fit any line and the third tells you the actual shape. Prefer removing an abstraction over adding a configuration flag to it.

The important nuance to add: **the cost is not symmetric.** Adding an abstraction later is a refactor, usually a mechanical and safe one. Removing an entrenched abstraction is much harder, because code has grown to depend on its extension points. That asymmetry argues for erring toward concrete code.

### Probes

**Judgement over recall.** The answer must be a specific, remembered decision with a cost you can name, not a rehearsed opinion about patterns.

**Abstraction cost.** Name them concretely: indirection when reading, more files to navigate, worse stack traces, harder debugging, more surface for tests, a barrier to newcomers, and constraints on future change (the abstraction dictates the shape of anything added).

**Premature generalisation.** Covered — the "we might need it later" bet, and why the guessed axis of variation is usually wrong.

**"Strategy pattern with one implementation."** Covered as example 1.

**Inheritance → composition.** The related move worth mentioning: a deep `AbstractBaseService` hierarchy where subclasses override protected template methods, each knowing more about its ancestors than it should. Replaced with small injected collaborators. The tell is a `protected` method that exists only so a subclass can hook it, or a subclass that must call `super` in exactly the right place or break.

**Over-layering.** Covered — a `Controller → Facade → Service → Manager → Repository` chain where three layers only forward calls and map DTOs. Every field addition touches five files. The honest test: **if a layer has no behaviour other than delegation, it is not a layer, it is a tax.** Deleting it is usually the right call, and doing so is one of the highest-value refactorings available.

---

## Q74. Strategy vs Template Method vs a lambda — what to reach for in modern Java.

### Answer

**Template Method** — an abstract base class defines the algorithm's skeleton and delegates variable steps to subclass overrides.

```java
abstract class ReportGenerator {
    public final Report generate(Query q) {   // the skeleton, final
        var data = fetch(q);
        var transformed = transform(data);    // varies
        return render(transformed);           // varies
    }
    protected abstract Data transform(Data d);
    protected abstract Report render(Data d);
}
```
*Strength:* the invariant sequence is enforced in one place, and subclasses cannot break it.
*Weaknesses:* **inheritance coupling** — the subclass is welded to the base class, can extend nothing else (single inheritance), and depends on the base's `protected` surface, which becomes a de facto public API you can never change. Testing a step in isolation requires instantiating the whole hierarchy. Behaviour cannot be changed at runtime. And it interacts badly with Spring: an abstract base with `@Autowired` protected fields is a well-known source of confusing wiring.

**Strategy** — the varying behaviour is a separate object implementing an interface, injected into a context.

```java
interface DiscountPolicy { Money apply(Order o); }

@Service
class Pricing {
    private final Map<CustomerTier, DiscountPolicy> policies;   // Spring injects by qualifier/key
    Money price(Order o) { return policies.get(o.tier()).apply(o); }
}
```
*Strengths:* composition rather than inheritance; each strategy is independently testable with no scaffolding; behaviour is selectable at runtime; new strategies are added without touching existing code (open/closed); it composes (a decorating strategy can wrap another).
*Weaknesses:* more types; and the *selection* logic has to live somewhere.

**Lambda / functional interface** — the same idea without the ceremony.

```java
Money price(Order o, UnaryOperator<Money> discount) { ... }
price(order, m -> m.multiply(0.9));
```
*Strengths:* zero boilerplate for one-off behaviour; composable (`Function.andThen`, `Predicate.and`); the JIT can often inline and scalar-replace it (Q10).
*Weaknesses:* anonymous — a `Function<Order, Money>` in a signature says nothing about intent, whereas `DiscountPolicy` does. Hard to name, discover, or find implementations of. No place to hang metadata (a code, a description, an applicability check). Poor stack traces. Not injectable or independently configurable.

**What I reach for in modern Java, and the reasoning**

1. **A named functional interface (a "strategy shaped like a lambda")** is the default sweet spot. Declare `@FunctionalInterface interface DiscountPolicy { Money apply(Order o); }` — you get the readability of a named domain concept *and* the option to implement it with a lambda, a method reference, or a full class as appropriate. This is the single most useful pattern in the list and the answer I'd lead with.

2. **Spring injecting `List<Strategy>` or `Map<String, Strategy>`.** Spring populates a `List<T>` with every bean of that type (ordered by `@Order`/`Ordered`), and a `Map<String, T>` keyed by bean name. Even better for a domain key, give the interface a `supports`/`key` method and build the map yourself:

```java
interface PaymentHandler {
    PaymentMethod method();
    Receipt handle(PaymentRequest r);
}

@Service
class PaymentDispatcher {
    private final Map<PaymentMethod, PaymentHandler> byMethod;
    PaymentDispatcher(List<PaymentHandler> handlers) {
        this.byMethod = handlers.stream()
            .collect(toMap(PaymentHandler::method, identity()));  // fails fast on duplicates
    }
    Receipt handle(PaymentRequest r) {
        var h = byMethod.get(r.method());
        if (h == null) throw new UnsupportedPaymentMethodException(r.method());
        return h.handle(r);
    }
}
```
This is the idiomatic Spring form of Strategy. Adding a payment method means adding one `@Component` and nothing else — no `switch` to update, no registration list, no configuration change. It is the concrete answer to "how do you make this open for extension".

3. **Enums with behaviour** — for a small, closed, stable set:
```java
enum Rounding {
    HALF_UP   { Money apply(Money m) { ... } },
    BANKERS   { Money apply(Money m) { ... } };
    abstract Money apply(Money m);
}
```
Exhaustive, serialisable, self-documenting, usable as a database column and as a `switch` subject, and impossible to forget a case. Constrained by being a compile-time-fixed set with no dependency injection — so it is right for pure functions over their input, wrong when a strategy needs collaborators.

4. **Sealed interface + pattern matching** — when the cases carry *different data* and you need exhaustiveness (Q13). This is the modern alternative to both Strategy and the visitor pattern, and it inverts the trade-off: easy to add operations, harder to add types. Choose it when the case set is stable and the operations grow.

5. **Template Method** survives in narrow cases: framework base classes where the skeleton is genuinely invariant and the extension points are the framework's contract (`AbstractController`, `OncePerRequestFilter`, `JdbcTemplate`'s callbacks). Even there, note that Spring itself increasingly prefers a **callback/lambda** form — `JdbcTemplate.query(sql, rowMapper)` is Template Method with the varying step passed as an argument rather than inherited, which is exactly the composition-over-inheritance move. When you find yourself writing an abstract class purely so subclasses can fill in two methods, prefer injecting those two behaviours instead.

### Probes

**Inheritance coupling.** Covered — single inheritance, the `protected` surface becoming an unchangeable API, and the Spring wiring hazard.

**Testability.** Covered — a strategy is a standalone unit; a template-method step needs the hierarchy instantiated.

**Spring injecting `List<Strategy>` or `Map<Type, Strategy>`.** Covered with a full worked example, including duplicate detection and unknown-key handling.

**Enums with behaviour.** Covered, with the conditions under which they are the best choice and the constraint that rules them out.

**Sealed interface + pattern matching as an alternative.** Covered, with the expression-problem trade-off that decides between it and Strategy.

---

## Q75. Where Builder genuinely beats a record or a constructor — and what a bad builder lets through.

### Answer

**What the alternatives do well.** A constructor is the simplest thing that works and guarantees an object is never observed half-built. A record gives you a canonical constructor, value equality, and deconstruction for free. For an immutable value with 2–5 required components, **a record is better than a builder** and reaching for a builder there is over-engineering.

**Where Builder genuinely wins**

**1. Many parameters, especially many optional ones.** A constructor with 12 parameters, 8 of which are optional, produces a telescoping-constructor explosion (`new Order(a,b)`, `new Order(a,b,c)`, `new Order(a,b,c,d)`…) that is combinatorially unmaintainable and, worse, unreadable at the call site: `new Order(id, cust, null, null, true, false, null, 3)` — nobody can tell what `true, false` mean. A builder names every argument:
```java
Order.builder().id(id).customer(cust).expedited(true).giftWrap(false).retries(3).build();
```
Java has no named or default parameters, which is precisely why the Builder pattern is more necessary here than in Kotlin, Python, or C#.

**2. Same-typed adjacent parameters.** `new Rectangle(width, height)` and `new Money(amount, fee)` — the compiler cannot help you if you swap them, and the bug is silent. A builder makes the mistake visible in the source. (The stronger fix is domain types — `Width`, `Height` — but a builder is cheaper.)

**3. Validation that must consider the whole object.** Cross-field invariants ("`endDate` must be after `startDate`", "`discount` requires a `promoCode`") belong in `build()`, where all values are present, rather than being scattered.

**4. Deriving or defaulting values from other fields** at construction time.

**5. Stepwise construction across code** — parsing, a fluent DSL, test data builders (`anOrder().withStatus(SHIPPED).build()`, which makes test intent obvious and keeps tests resilient to constructor changes). Test data builders are one of the highest-value uses of the pattern.

**6. Combining with immutability + evolution.** A builder plus `toBuilder()` gives cheap "copy with one change" semantics on an immutable object. Records get this partially via Java's `with`-expression proposals, but as of Java 21 you write it by hand.

**And where a record wins outright:** few components, all required, no cross-field derivation. Then a record's compact constructor handles validation and you avoid ~60 lines of builder.

**What an improperly implemented builder lets through**

**1. Missing required fields, discovered at runtime.** The core weakness: a builder converts a *compile-time* guarantee (a constructor demands all its arguments) into a *runtime* check. `Order.builder().build()` compiles cleanly and throws — or, far worse, silently produces an object with `null` fields that fails three layers away.
*Fix:* validate in `build()` and throw immediately with a message naming every missing field, or use a **staged/typed builder**.

**2. No validation at all in `build()`.** A builder that just copies fields into the constructor and skips checks. The validation must live in `build()` **or** in the target's constructor — and putting it in the *constructor* is strictly better, because it cannot be bypassed by any other construction path. The builder then merely provides ergonomics. (Lombok's `@Builder` on a class with a validating constructor gives you exactly this.)

**3. A mutable builder reused.** The single most common real bug:
```java
var b = Order.builder().customer(c);
var o1 = b.item(itemA).build();
var o2 = b.item(itemB).build();     // o2 contains BOTH items
```
Builders accumulate state and `build()` typically does not reset it. If the builder holds a mutable collection and passes the *same reference* to the built object, then `o1` and `o2` share a list and mutating the builder afterwards mutates the already-built objects.
*Fixes:* copy collections defensively inside `build()` (`List.copyOf(items)`); optionally make `build()` single-use by throwing on a second call; and document that a builder instance is not reusable. A related trap: **`@Singular` in Lombok** solves the collection case properly, which is why it exists.

**4. Escaping mutable state.** `builder.items(myList).build()` where the built object stores `myList` directly — the caller retains a reference and can mutate the "immutable" object afterwards. Copy on the way in *and* return unmodifiable views on the way out. Same issue as records (Q12).

**5. Thread safety.** A builder is **not** thread-safe and should not be shared. `build()` should produce a consistent snapshot; if the builder is mutated concurrently, the built object can contain a mix of states. The built *object* is thread-safe (if immutable); the builder never is. Static shared builders are a genuine production bug.

**6. Losing the "never invalid" guarantee.** A builder used to construct an entity progressively across several methods means the object exists in invalid intermediate states, defeating the point.

**Staged / typed builders — enforcing required fields at compile time**

```java
Order.builder()          // returns CustomerStep
     .customer(c)        // returns ItemsStep
     .items(items)       // returns FinalStep — build() only exists here
     .expedited(true)    // optional, stays on FinalStep
     .build();
```
Each required step returns an interface exposing only the *next* legal call, so omitting a required field is a **compile error** rather than a runtime exception. You can also make illegal orderings unrepresentable. Costs: significant boilerplate (mitigated by generators — Immutables' `@Value.Style(stagedBuilder = true)`, or record-builder), and less flexible call ordering. Worth it for a widely used public API or a domain type where an invalid instance is expensive; overkill for internal DTOs.

**Practical recommendation:** records or constructors by default; a builder when you cross roughly 5–6 parameters or have several optional ones; validation in the target's constructor regardless; defensive copies of collections in both directions; and staged builders only for high-value public APIs. Lombok's `@Builder` with `@Singular` and a validating constructor covers most cases — while noting the general caution that Lombok is a compile-time dependency with annotation-processor ordering issues alongside MapStruct and QueryDSL.

### Probes

**Optional/many params.** Covered, with the named-parameter argument.

**Telescoping constructors.** Covered.

**Validation in `build()`.** Covered, with the argument that the constructor is the better home for it.

**Thread safety.** Covered — the builder is never thread-safe, the product can be.

**Mutable builder reused.** Covered as the most common real bug, with concrete fixes.

**Staged/typed builders to enforce required fields at compile time.** Covered, with costs and a recommendation on when the boilerplate is justified.

---

## Q76. The transactional outbox pattern.

### Answer

**The problem it solves: the dual-write problem.**

```java
@Transactional
public void placeOrder(Order o) {
    orderRepository.save(o);                              // write 1: the database
    kafkaTemplate.send("orders", new OrderPlaced(o));     // write 2: the broker
}
```

This looks atomic because of `@Transactional`, but it is not — **the transaction covers only the database.** Kafka has no idea it exists. Three concrete failure modes:

1. **`send()` succeeds, the transaction rolls back.** A downstream service receives `OrderPlaced` for an order that does not exist. Phantom event. Irrecoverable without compensation, and impossible to detect from the database.
2. **The transaction commits, then the process crashes before `send()`.** The order exists; nobody is told. Silent loss — the worst kind, because there is no error anywhere. Nothing retries.
3. **`send()` is asynchronous** (the default: `KafkaTemplate.send` returns a future and the producer batches in the background). So the commit and the actual network send are unordered, and both of the above can occur even when the code *looks* sequential.

The general statement: **you cannot atomically write to two independent systems without a distributed transaction**, and distributed transactions (XA/2PC) are not a realistic option — Kafka does not support XA, and 2PC brings a blocking coordinator, poor availability under partition, and operational pain that the industry abandoned for good reasons.

**The outbox pattern**

Make the second write part of the *same database transaction* as the first, by writing the message to a table:

```sql
CREATE TABLE outbox (
    id             uuid PRIMARY KEY,
    aggregate_type varchar(64)  NOT NULL,     -- 'Order'
    aggregate_id   varchar(64)  NOT NULL,     -- for partition key / ordering
    event_type     varchar(64)  NOT NULL,     -- 'OrderPlaced'
    payload        jsonb        NOT NULL,
    headers        jsonb,                     -- trace context, schema version
    created_at     timestamptz  NOT NULL DEFAULT now(),
    published_at   timestamptz                -- NULL until relayed (polling variant)
);
CREATE INDEX idx_outbox_unpublished ON outbox (created_at) WHERE published_at IS NULL;
```

```java
@Transactional
public void placeOrder(Order o) {
    orderRepository.save(o);
    outboxRepository.save(OutboxMessage.of("Order", o.getId(), "OrderPlaced", o));
}   // ONE commit: either both rows exist or neither does
```

Now the atomicity is real, because both writes go through the same transaction manager and the same ACID guarantee. A separate **relay** reads the outbox and publishes to Kafka.

Note the partial index: it keeps the "unpublished" index tiny even as the table grows, which is exactly the partial-index use case from Q54.

**Relay: polling vs CDC**

**Polling publisher.** A scheduled job selects unpublished rows, publishes them, and marks them published.
```sql
SELECT * FROM outbox WHERE published_at IS NULL
 ORDER BY created_at
 FOR UPDATE SKIP LOCKED
 LIMIT 100;
```
`SKIP LOCKED` (Q50) lets multiple relay instances work disjoint batches with no contention. Simple, no extra infrastructure, easy to test and reason about. Costs: polling latency (a poll interval of 100ms–1s), constant database load, and the table needs archiving/partitioning so it doesn't grow forever.

**CDC (Change Data Capture) with Debezium.** Debezium tails the database's **write-ahead log** (PostgreSQL logical replication, MySQL binlog) and publishes changes to Kafka. Its `EventRouter` SMT is purpose-built for the outbox pattern: it reads the outbox insert, uses `aggregate_id` as the Kafka key, routes by `aggregate_type`, and emits just the payload.
*Advantages:* no polling load on the database, very low latency, no application code in the publish path, and it captures the events in commit order from the WAL. *Costs:* Debezium and Kafka Connect to operate, replication slots to monitor (an unconsumed slot causes **unbounded WAL growth and will fill the disk** — a well-known production incident), schema/DDL handling, and more moving parts.

Choose polling for one or two services and simplicity; CDC when you have many services, need low latency, or already run Kafka Connect.

**At-least-once + consumer idempotency**

Crucially, the outbox gives **at-least-once**, not exactly-once. The relay can publish and then crash before marking the row published, so it republishes on restart. This is by design and unavoidable (Q56, Q83). Therefore:

- Every event carries a stable, unique **event ID** (the outbox `id`).
- Every consumer must be idempotent: record processed event IDs in a `processed_events` table with a unique constraint, **committed in the same transaction as the consumer's business write**, and skip duplicates. Or make the business operation naturally idempotent (a `SET` rather than an increment, an upsert rather than an insert).

**Ordering guarantees**

- Within a single aggregate, publish with `aggregate_id` as the **Kafka key**, so all events for one order land on one partition and are consumed in order (Q81).
- The relay must publish in **outbox insertion order** for a given key. `ORDER BY created_at` (or better, a monotonic sequence/`id`) plus single-threaded publishing per key achieves this. Careful: with a polling relay and multiple instances using `SKIP LOCKED`, two instances *can* publish two events for the same aggregate out of order. If per-aggregate ordering matters, either partition the outbox work by `aggregate_id` hash across relay instances, or run a single relay instance, or use CDC (which preserves commit order from the WAL).
- Also careful with `created_at` as the ordering column: `now()` in PostgreSQL is the **transaction start time**, so two concurrent transactions can commit in a different order than their timestamps suggest. Prefer a sequence, and be aware that even sequence values can become visible out of order — CDC avoids this entirely by reading commit order.

**Alternatives worth naming:** the **listen-to-yourself** pattern (publish the event first, consume your own event to do the write) — inverts the problem but adds latency and its own consistency issues; Kafka **transactions** (exactly-once *within* Kafka, but they do not extend to your database, so they don't solve this); and simply **deriving events from the database state** via CDC on the business tables directly — simpler, but couples consumers to your internal schema, which is why the explicit outbox table (a deliberate, versioned public contract) is usually preferred.

### Probes

**Dual-write problem.** Covered with all three failure modes, including the asynchronous-send subtlety.

**No distributed transaction between DB and broker.** Covered, including why XA/2PC is not a practical answer.

**Outbox table written in the same tx.** Covered with schema and code.

**Relay via polling or CDC (Debezium).** Both covered, with the `SKIP LOCKED` polling query, Debezium's `EventRouter` SMT, and the replication-slot/WAL-growth operational hazard.

**At-least-once + consumer idempotency.** Covered, including committing the dedupe record in the same transaction as the business write.

**Ordering guarantees.** Covered in detail — keying by aggregate ID, and the two non-obvious ordering hazards (parallel relays with `SKIP LOCKED`, and `now()` being transaction start time).

---

## Q77. The saga pattern — a concrete 3-service flow, choreography or orchestration.

### Answer

**The problem.** A business transaction spans services with separate databases. `CreateOrder` must reserve inventory, take payment, and schedule shipping. There is no distributed transaction, so you cannot roll all three back atomically. A saga models the process as a **sequence of local transactions**, each publishing an event that triggers the next, with a **compensating transaction** for each step that can undo its effect if a later step fails.

**A concrete flow: Order → Payment → Inventory**

| Step | Local transaction | Compensation |
|---|---|---|
| 1. Order Service | Create order in `PENDING` | Mark order `CANCELLED` |
| 2. Payment Service | Authorise the card, record the authorisation | Void/refund the authorisation |
| 3. Inventory Service | Decrement stock, create a reservation | Release the reservation, restore stock |
| 4. Order Service | Mark order `CONFIRMED` | — (terminal success) |

Failure at step 3 (out of stock) triggers compensation in reverse: void the payment authorisation, then cancel the order, then notify the customer.

**Compensating actions are not rollbacks.** This distinction is the heart of the question. A database rollback makes it as if the transaction never happened — nothing was ever visible. A compensation is a **new forward transaction** that semantically undoes the effect, and it runs *after* the original effect was already visible to the world. Consequences:

- The customer may have already seen a "payment taken" email and a bank notification, then a refund. You cannot un-send that.
- A refund is not the inverse of a charge: fees may be non-refundable, the exchange rate may have moved, the money may take days to return, and the accounting must record both movements.
- Some actions are **not compensatable at all** — an email sent, an SMS delivered, a physical item shipped, a report filed with a regulator. **Order your saga so that non-compensatable steps come last**, after everything that can fail has succeeded. This is the single most practical design rule for sagas.
- Compensations themselves can fail, and must be retried indefinitely (they cannot be "given up on" without leaving inconsistent state) — so they must be **idempotent** and, if they exhaust retries, must escalate to a human via a dead-letter/manual-intervention queue.

**Semantic locks.** Because intermediate states are visible, you need application-level guards against concurrent interference. The order sits in `PENDING` — a semantic lock that tells other processes "this is in flight, don't act on it as if it were confirmed". The inventory reservation is another: stock is decremented but held in a `RESERVED` state with a TTL, so it is neither available to others nor yet permanently consumed. Related countermeasures from Garcia-Molina's original work: *commutative updates* (design operations so order doesn't matter — increments rather than sets), *pessimistic view* (reorder steps so the risky ones happen before anything a user can see), and *re-read value* (re-check preconditions before acting, to detect interference).

**Choreography vs orchestration**

**Choreography** — no central coordinator. Each service listens for events and publishes its own.
```
Order → OrderCreated → Payment → PaymentAuthorised → Inventory → StockReserved → Order → OrderConfirmed
                                 PaymentFailed → Order (cancel)
```
*Pros:* no single point of coordination, services are independently deployable, adding a new interested party requires no change to existing services, and it feels naturally decoupled.
*Cons:* **the process exists nowhere.** No file describes the flow; it is emergent from N services' event handlers. To answer "what happens when an order is placed?" you must read all of them. Debugging a stuck saga means correlating logs across services. Cyclic dependencies creep in. Compensation logic is scattered — each service must know what to do when a *later* step fails, which means it must know about steps it should not know about. This is the "implicit coupling" the probe names: services appear decoupled but are in fact tightly coupled through an undocumented, unenforced protocol. It degrades badly beyond about 3–4 steps.

**Orchestration** — a central saga orchestrator drives the flow, sending commands and reacting to replies.
```java
// conceptually
onOrderRequested   → send ReserveCredit  → onCreditReserved  → send ReserveStock
                                          → onCreditRejected → send CancelOrder
onStockReserved    → send ConfirmOrder
onStockUnavailable → send ReleaseCredit  → send CancelOrder
```
*Pros:* **the process is explicit, in one place, and readable.** State is persisted, so you can query "which sagas are stuck and where". Compensation is defined centrally in reverse order. Testing the flow is possible without deploying five services. Participants are simpler — they expose commands and know nothing about the wider process. Timeouts and retries live in one place.
*Cons:* the orchestrator is a component to build, deploy, and operate; it can become a "god service" accumulating business logic that belongs in the participants (the discipline is that the orchestrator owns *sequencing*, participants own *decisions*); and it is a potential availability bottleneck, mitigated by making it stateless-with-persisted-state and horizontally scalable.

**Which I would choose, and why**

**Orchestration for anything beyond two or three steps** — which describes essentially every real business process. The decisive argument is **operability**: when a saga is stuck at 3am, orchestration lets you query one table and see exactly which step failed and why. With choreography you are reconstructing a distributed state machine from logs. In my experience that operational difference outweighs the coupling argument, and the coupling argument is largely illusory anyway — choreography does not remove coupling, it hides it.

Choreography is defensible for genuinely simple, two-step, fire-and-forget flows, and for **notification** fan-out (which is not a saga at all — no compensation is needed if a downstream consumer just wants to know something happened). A useful rule: **use choreography for events (facts others may care about) and orchestration for processes (steps that must happen).**

**Implementation options in the Java ecosystem:** Temporal (durable execution — arguably the strongest current answer, because it makes the orchestration code look like a plain method with automatic state persistence and retries), Camunda/Zeebe (BPMN-modelled), Axon Framework (`@SagaEventHandler` with `@StartSaga`/`@EndSaga`), Eventuate Tram Sagas, or a hand-rolled state machine (a `saga_instances` table plus Spring Statemachine and a command/reply topic pair) — which is entirely reasonable and often the right call for one or two processes.

**Eventual consistency visible to the user.** The user presses "Place order" and gets `202` with an order in `PENDING`. The UI must be designed for this: show the state, poll or subscribe for updates, and never imply completion before it is real. The most common failure here is a *product* failure, not a technical one — a UI that says "Order confirmed!" while the saga is still running, then emails a cancellation two minutes later. Design the customer-facing language around the actual state machine.

**Observability of a distributed flow.** Non-negotiable for sagas. Every message carries a **saga/correlation ID** and W3C trace context, so one trace spans all services. Persist saga state with a timestamp per step, so you can query stuck instances. Add: a metric per saga type for started/completed/compensated/stuck; an alert on sagas older than N minutes in a non-terminal state; a **timeout per step** so a lost reply doesn't hang forever (it must trigger compensation, not silent abandonment); and an operator tool to inspect and manually advance or compensate an individual saga. Without these, a saga architecture is unsupportable.

### Probes

**Compensating actions (not rollbacks).** Covered as the central distinction, with the practical consequences and the "non-compensatable steps last" rule.

**Semantic locks.** Covered, along with the other classic countermeasures.

**Eventual consistency visible to the user.** Covered, framed as a product-design requirement.

**Observability of a distributed flow.** Covered — correlation IDs, persisted state, per-step timeouts, metrics, alerts, and an operator tool.

**Orchestrator as a single point of coordination vs choreography's implicit coupling.** Both covered, with a defended recommendation and the "events vs processes" rule of thumb.

---

## Q78. CQRS — and when it is over-engineering.

### Answer

**What CQRS actually is.** Command Query Responsibility Segregation separates the model used to **change** state (commands) from the model used to **read** it (queries). That is the entire definition. It does not require event sourcing, separate databases, message brokers, or eventual consistency — those are optional amplifications that people routinely mistake for the pattern itself.

**The spectrum, from cheapest to most expensive**

1. **Separate models, one database, one transaction.** Commands go through the domain model (JPA entities, aggregates, invariants); queries bypass it entirely and use `JdbcClient`/jOOQ to return query-shaped DTOs. Strong consistency preserved, no new infrastructure, and it immediately solves the most common real problem: an aggregate designed for consistency boundaries makes a terrible source for a 12-column dashboard, and forcing reads through it produces N+1s, over-fetching, and contorted queries. **This is CQRS, and for most systems it is where you should stop.**

2. **Separate models, separate read schema in the same database** — materialised views or denormalised read tables, refreshed synchronously in the same transaction or by a trigger. Still strongly consistent.

3. **Separate read database**, updated asynchronously from domain events (or CDC). Introduces eventual consistency and replication lag. Enables an entirely different storage technology for reads — Elasticsearch for search, a document store for a denormalised view, a columnar store for analytics — and independent scaling of the read side, which is genuinely valuable at high read:write ratios.

4. **CQRS + event sourcing.** The write side stores an append-only event log as the system of record; read models are **projections** built by replaying events. Maximum power, maximum cost.

**Separate read model vs full event sourcing.** These are independent choices and conflating them is the most common misunderstanding. You can do CQRS without event sourcing (levels 1–3) — and you should, usually. You can technically event-source without CQRS, but it is impractical, because an event log is a terrible query target. Event sourcing brings a complete audit trail, temporal queries ("what did this look like on 3 March?"), and the ability to build a *new* read model retroactively over all history — which is a genuinely remarkable capability. It also brings schema evolution of events (upcasting old events forever), snapshotting for aggregates with long histories, a GDPR problem (an immutable log versus the right to erasure — usually solved by crypto-shredding, encrypting per-subject data and destroying the key), and a permanent conceptual tax on every developer who joins.

**Replication lag and read-your-writes**

The defining operational problem of level 3+. The user submits a form, is redirected to the list page, and their change is not there — because the projection has not caught up. This is not a bug you can fix; it is inherent. You **manage** it:

- **Return the result from the command.** The `POST` response contains the created/updated resource, so the UI can render it without a read. Cheapest and most effective.
- **Optimistic UI.** The client applies the change locally and reconciles later.
- **Version tokens.** The command returns a version/sequence number; the query includes it and the read side waits (briefly, with a timeout) until its projection has reached it. Correct, but adds latency and complexity.
- **Route the reader to the write model** for the affected entity for a short window, or use sticky routing so a user's reads go to a replica known to be caught up.
- **Just show the state honestly** — "Processing…" with the actual status.

Whichever you choose, **measure projection lag as a first-class metric** and alert on it. A stalled projector is invisible to users until it isn't, and by then it may be hours behind.

**Projection rebuild.** A significant operational advantage *and* a significant cost. Because the read model is derived, you can drop and rebuild it — to fix a projection bug, add a field, or change the shape. In practice this means: replaying potentially millions of events (which can take hours), doing so without downtime (build the new projection alongside the old, then switch reads over — the same expand/contract idea as Q53), making projectors **idempotent** so a partial rebuild can be resumed, tracking a checkpoint per projection, and having tooling to trigger and monitor a rebuild. Teams that adopt event sourcing without building this tooling discover they cannot actually change their read models, which removes the main benefit.

**When it is over-engineering — the important part**

CQRS beyond level 1 is over-engineering when:

- **Reads and writes have similar shapes and similar volumes.** A standard CRUD admin application. The separation buys nothing and doubles the model count.
- **A DTO projection would be enough.** Spring Data interface/class projections, a constructor expression in JPQL, or a hand-written SQL query returning a DTO solves 80% of "the entity model is wrong for this screen" with zero architectural change. **Try this first, always.**
- **A materialised view would be enough.** If the problem is an expensive aggregate query, a refreshed materialised view is one DDL statement and no new consistency model.
- **The team is small or new to the pattern.** CQRS with eventual consistency multiplies the number of things that can go wrong and the number of concepts a new joiner must hold. That cost is real and permanent.
- **Strong consistency is a genuine requirement** for the read path — financial balances shown before a transfer, stock levels at the point of purchase. Adding eventual consistency here creates business bugs.
- **You adopted it for "scalability" without measuring.** Read/write splitting via a **database read replica** gives you most of the read-scaling benefit for a configuration change, with no new model, no projection code, and a lag you can monitor with existing tools. If that suffices — and it usually does — CQRS is not justified by scaling alone.

**How I would frame it in an interview:** "I use CQRS in its cheap form routinely — the write model is the aggregate, reads are separate query-shaped DTOs against the same database. I would only introduce a separate read store when there's a concrete, measured driver: a read:write ratio that a replica can't absorb, a query shape the transactional store can't serve well (full-text search, graph traversal, analytics), or a genuine need for multiple independent read models. And I'd introduce event sourcing only where the audit log or temporal queries are themselves a business requirement — not as a means to CQRS."

### Probes

**Separate read model vs full event sourcing.** Covered as independent choices, with the four-level spectrum and the specific costs event sourcing adds (upcasting, snapshots, GDPR/crypto-shredding).

**Replication lag / read-your-writes.** Covered, with five concrete mitigations and the requirement to monitor projection lag.

**Projection rebuild.** Covered — the capability, the cost, and the tooling required for it to be real.

**When a materialised view or a DTO query is enough.** Covered as the leading over-engineering test, along with read replicas as the cheaper scaling answer.

---

## Q79. Hexagonal architecture — the actual benefit beyond folder structure.

### Answer

**The structure.** Ports and adapters (Cockburn, 2005). The application core — domain model plus use cases — defines **ports**, which are interfaces expressed entirely in domain terms. **Adapters** implement or drive those ports and contain all technology-specific code.

- **Driving (primary/inbound) adapters** call the application: a REST controller, a Kafka listener, a CLI, a scheduled job, a test.
- **Driven (secondary/outbound) adapters** are called by the application: a JPA repository, an HTTP client for a payment provider, an email sender, a message publisher.

```
UserController ──▶ PlaceOrderUseCase (port, in) ──▶ OrderService (core)
                                                        │
                                            OrderRepository (port, out)
                                                        ▲
                                              JpaOrderRepositoryAdapter
```

**The mechanism: dependency inversion at the boundary.** In a conventional layered application, `service` depends on `repository`, and `repository` depends on JPA — so the dependency arrow points from your domain toward the framework. In hexagonal, the **core owns the interface**: `OrderRepository` is declared in the domain package, in domain language (`Optional<Order> findById(OrderId)`), and the adapter in the infrastructure package implements it. The compile-time dependency now points **inward**, from infrastructure to domain. That inversion is the whole architecture; everything else is consequence.

**The actual benefits — beyond folder structure**

**1. The domain is testable without a framework.** This is the benefit that pays for itself daily. Use-case tests instantiate the service with in-memory fake adapters and run in **milliseconds**, with no Spring context, no database, no Testcontainers. A domain with genuine complexity — pricing rules, eligibility, state machines, allocation logic — gets hundreds of fast tests covering combinations that would be impractical to set up through HTTP and SQL. Teams that adopt hexagonal well usually report test suite time as the most tangible win. The corollary: if your domain is thin CRUD, this benefit is small, which is a real argument against the pattern for such systems.

**2. It makes the domain model *possible*.** The subtle, more important point. When the "domain object" is a JPA entity, its design is dictated by the ORM: a no-arg constructor, mutable fields, no final fields, getters and setters everywhere, bidirectional associations for query convenience, an identity that is a database sequence. You cannot make it immutable, you cannot enforce invariants in a constructor, and you cannot use rich value objects freely. Separating the persistence model (an `@Entity`) from the domain model (a plain class or record) lets the domain be designed for **the business rules** rather than for Hibernate. The cost is a mapping layer — which is exactly the trade-off in the next section.

**3. Genuine substitutability at the boundary — but not the one people claim.** "You could swap PostgreSQL for MongoDB" is a bad justification; nobody does that, and if they did, the domain-shaped repository interface would leak anyway. The substitutions that *actually happen* are at the edges: replacing a payment provider, a search backend, a notification channel, an identity provider; adding a second inbound adapter (a Kafka listener alongside the REST controller, both invoking the same use case with no duplicated logic); or swapping a real third-party client for a stub in a test environment. These are common and the port boundary makes them cheap.

**4. It localises framework churn.** Spring Boot 2→3 (with the `javax`→`jakarta` migration) touched adapters; a well-isolated domain compiled unchanged. Over a decade of framework upgrades this compounds.

**5. It makes architectural intent enforceable.** With ArchUnit you can assert the dependency rule in CI:
```java
@ArchTest static final ArchRule domain_is_independent =
    noClasses().that().resideInAPackage("..domain..")
      .should().dependOnClassesThat()
      .resideInAnyPackage("..infrastructure..", "org.springframework..", "jakarta.persistence..");
```
This matters enormously: without automated enforcement, every architecture degrades into a big ball of mud within a year, because the shortcut is always locally cheaper. A failing build is the only durable defence. (Java's module system or Spring Modulith can serve a similar role for module boundaries.)

**The costs — state them honestly**

- **Mapping layers.** Domain `Order` ↔ persistence `OrderEntity` ↔ API `OrderResponse`. Three representations of the same concept, plus mappers (MapStruct helps, but adds an annotation processor and generated code to debug). Every field addition touches several files. This is the single biggest and most legitimate objection.
- **Indirection.** "Where does this actually happen?" requires following an interface to its single implementation. IDEs help; new joiners still slow down.
- **Lost ORM conveniences.** Dirty checking, lazy loading, and cascades operate on entities, so a domain-model/persistence-model split means explicit loading and explicit saving. Sometimes that is a *benefit* (it makes the query cost visible), but it is more code.
- **Over-application.** The most common failure is applying it uniformly — a full hexagon around a service whose entire domain logic is "save this row".

**When a Spring-flavoured layered application is fine — and it often is.** If the service is predominantly CRUD, if the business rules are thin, if the team is small, then `Controller → Service → Spring Data Repository` with JPA entities used throughout is honest, fast to write, well understood by every Java developer, and perfectly maintainable. Adding ports and adapters there produces ceremony with no offsetting benefit.

**The recommendation I would defend:** apply it **selectively, by bounded context**. In a system with one genuinely complex core domain and six supporting CRUD modules, put the hexagon around the complex one and leave the rest layered. Architectural uniformity for its own sake is a cost, not a virtue — and being able to say *which* parts of a system deserve which treatment is precisely what senior judgement looks like here.

### Probes

**Dependency inversion at the boundary.** Covered as the actual mechanism, contrasted with conventional layering.

**Domain testable without Spring/DB.** Covered as the primary daily benefit, with the honest caveat about thin domains.

**Ports vs adapters.** Covered, including the driving/driven distinction.

**Cost: mapping layers and indirection.** Covered candidly, along with lost ORM conveniences.

**When a Spring-flavoured layered app is fine.** Covered, with a selective, per-bounded-context recommendation and ArchUnit enforcement as the thing that makes either choice survive contact with a deadline.

---

## Q80. Design a circuit breaker from scratch — states, thresholds, and behaviour when open.

### Answer

**Purpose.** A circuit breaker stops calls to a failing dependency so that (a) the caller fails fast instead of tying up threads and connections waiting for a doomed call, and (b) the failing dependency gets breathing room to recover instead of being hammered by retries. It converts a slow, resource-consuming failure into an instant, cheap one.

**The three states**

```
        failure rate ≥ threshold
CLOSED ───────────────────────────▶ OPEN
   ▲                                  │ after waitDurationInOpenState
   │  success rate OK                 ▼
   └────────────────────────────── HALF_OPEN
              (a limited number of trial calls)
                    │ failures
                    └──────────▶ OPEN
```

- **CLOSED** — normal. Calls pass through; outcomes are recorded in a sliding window.
- **OPEN** — calls are rejected immediately without touching the dependency (Resilience4j throws `CallNotPermittedException`). After a fixed wait, transition to half-open.
- **HALF_OPEN** — allow a small, bounded number of trial calls (`permittedNumberOfCallsInHalfOpenState`). If enough succeed, close; if they fail, open again. Half-open is what prevents a single probe from flipping the breaker back and forth, and it must **cap concurrency** — otherwise the moment the breaker half-opens, all queued traffic stampedes the still-fragile dependency.

Resilience4j adds two useful non-automatic states: `DISABLED` (always allow) and `FORCED_OPEN` (always reject) — invaluable as a manual kill switch during an incident.

**Sliding window: count-based vs time-based**

- **Count-based** (`slidingWindowSize=100`) — the last N calls. Deterministic and memory-bounded, but on a low-traffic endpoint those 100 calls may span an hour, so the breaker reacts to ancient history.
- **Time-based** (`slidingWindowSize=60`, seconds) — all calls in the last N seconds, bucketed per second. Better for variable traffic; reacts in real time. Generally the better default for HTTP services.

Both require a `minimumNumberOfCalls` (e.g. 20) before the failure rate is evaluated — otherwise a single failure on a cold endpoint (1 of 1 = 100%) trips the breaker instantly.

**Thresholds: failure rate AND slow-call rate**

```yaml
resilience4j.circuitbreaker.instances.payments:
  slidingWindowType: TIME_BASED
  slidingWindowSize: 60
  minimumNumberOfCalls: 20
  failureRateThreshold: 50           # percent
  slowCallRateThreshold: 50          # percent
  slowCallDurationThreshold: 2s
  waitDurationInOpenState: 10s
  permittedNumberOfCallsInHalfOpenState: 5
  automaticTransitionFromOpenToHalfOpenEnabled: true
  recordExceptions: [java.io.IOException, java.util.concurrent.TimeoutException]
  ignoreExceptions: [com.acme.BusinessValidationException]
```

Two points that separate a good answer:

- **Slow-call rate is as important as failure rate.** The worst dependency failure is not one that returns errors — it is one that returns successfully after 30 seconds. Failure rate stays at 0% while every caller thread blocks and the pool exhausts. `slowCallDurationThreshold` treats "too slow" as a failure, which is the behaviour you actually want.
- **`ignoreExceptions` is essential.** A `400 Bad Request` or a domain validation failure is **not** a dependency failure. Counting it opens the breaker because *your clients* sent bad input, taking down a healthy dependency. Record only transport-level failures, timeouts, and `5xx`. Getting this wrong is one of the most common circuit-breaker misconfigurations.

**What to do in the open state**

Two legitimate choices, and the answer should distinguish them:

- **Fail fast** — return `503` with `Retry-After`, or an error the caller can act on. Correct when the data is essential and a wrong answer is worse than no answer (payment authorisation, stock levels at checkout).
- **Fallback** — return degraded but useful data: a cached previous value (with a staleness indicator), a default (free shipping if the shipping-rates service is down), an empty list, or a reduced feature set. Correct when partial function is better than none — recommendations, personalisation, enrichment, non-critical panels on a page.

The rule: **a fallback must be a deliberate product decision, not a technical convenience.** Silently returning an empty list where the real answer was "5 items" can be worse than an error, because downstream logic treats it as truth. Whatever you choose, make it observable — count fallback invocations as a first-class metric, because a service quietly running on fallbacks for a week is a real and common failure.

**Combining with timeout, retry, and bulkhead — and the ordering**

A circuit breaker alone is insufficient. The full stack, in Resilience4j's recommended decoration order (outermost first):

```
Retry ( CircuitBreaker ( RateLimiter ( TimeLimiter ( Bulkhead ( call ) ) ) ) )
```

- **Timeout is the foundation.** Without one, a hung call blocks a thread forever and no breaker ever sees a result to record. Set connect *and* read timeouts on every client, and make the total timeout shorter than your own caller's timeout (a **timeout budget** that decreases with depth — otherwise an inner call is still running when the outer caller has already given up).
- **Bulkhead** bounds concurrent calls to that dependency (a semaphore, or a dedicated small thread pool), so one slow dependency cannot consume every thread in the service. This is what stops a single failing dependency from taking down endpoints that don't even use it.
- **Retry sits outside the breaker**, so a retried call that fails is recorded once per attempt and retries stop once the breaker opens.

**Why retry without a circuit breaker amplifies an outage.** This is the key insight. A dependency degrades and starts failing 30% of requests. Every client retries three times. The dependency now receives **up to 3× its normal load** precisely when it is least able to serve it — a *retry storm*. It fails harder, more retries fire, and the system converges on total failure. Retries are only safe when bounded, jittered, and gated by a breaker that stops them entirely once failure is systemic. Related mechanisms worth naming: **retry budgets** (allow retries only up to e.g. 10% of total request volume — what gRPC and Envoy implement, and strictly better than per-call limits), and **load shedding** at the server (reject early with `503` when the queue is deep, rather than accepting work you cannot complete).

**Jittered exponential backoff.** `delay = min(base × 2^attempt, cap)` **plus randomisation**. Without jitter, all clients that failed at the same instant retry at the same instant, producing synchronised waves that repeatedly re-break the recovering dependency. Full jitter (`random(0, computedDelay)`) is the AWS-recommended form and is measurably better than equal or decorrelated jitter for most workloads. Also: **never retry non-idempotent operations** without an idempotency key (Q56), and never retry a `4xx` (except `429`, honouring `Retry-After`).

**Practical notes.** Resilience4j is the standard choice on Spring Boot 3 (Hystrix is long dead and in maintenance; Spring Cloud Circuit Breaker abstracts over implementations). Breaker state is **per-instance** by default — 20 pods each learn independently, which is usually acceptable and arguably desirable, but means the effective load on a failing dependency during recovery is 20× your half-open permit count; account for it. A **service mesh** (Istio/Envoy) can provide outlier detection and circuit breaking at the infrastructure layer instead, with the advantage of covering every language and the disadvantage of no application-level fallbacks. Expose breaker state transitions as metrics and events (`resilience4j.circuitbreaker.state`) and **alert on any breaker opening** — it is one of the highest-signal alerts you can have.

### Probes

**Closed/open/half-open.** Covered, plus the manual `DISABLED`/`FORCED_OPEN` states and why half-open must cap concurrency.

**Sliding window (count vs time).** Covered, with `minimumNumberOfCalls` and a recommendation.

**Failure rate + slow-call rate.** Covered, with the argument that slow calls are the more dangerous failure, and the `ignoreExceptions` misconfiguration.

**Fallback vs fail-fast.** Covered, with the rule that a fallback is a product decision and must be measured.

**Combining with timeout + retry + bulkhead.** Covered, including the decoration order and the timeout-budget-decreasing-with-depth rule.

**Why retry without a circuit breaker amplifies an outage.** Covered — the retry storm, retry budgets, and load shedding.

**Jittered exponential backoff.** Covered, including full jitter and the rules on which responses may be retried.

---

# 11. Messaging (Kafka-centric)

---

## Q81. Partitions, consumer groups, and ordering — per-entity ordering while scaling.

### Answer

**The model.** A topic is split into **partitions**. Each partition is an ordered, immutable, append-only log; each record has a monotonically increasing **offset** within its partition. **Kafka guarantees ordering within a partition, and nowhere else.** There is no global ordering across a topic, and there cannot be — that is precisely what makes Kafka horizontally scalable.

A **consumer group** is a set of consumers sharing a `group.id`. The group coordinator assigns partitions such that **each partition is consumed by exactly one consumer in the group** at any time. That constraint is what gives you both parallelism and ordering: partitions are processed concurrently across consumers, while each partition's records are processed serially by one consumer.

Two immediate consequences:
- **Partition count is your maximum parallelism per group.** With 12 partitions and 20 consumer instances, 8 sit idle consuming nothing. Adding pods past the partition count does nothing.
- Multiple *groups* each get a full copy of the stream, independently. That is how you fan out to several services from one topic.

**Achieving per-entity ordering while scaling: key-based partitioning**

Set the record key to the entity's identifier:

```java
kafkaTemplate.send("orders", order.getId().toString(), event);
```

The default partitioner computes `murmur2(keyBytes) % numPartitions`. Because the function is deterministic, **every record for a given key always lands on the same partition**, and therefore is consumed in order by a single consumer. Different keys spread across partitions and are processed in parallel.

So the answer to "ordering per entity, parallelism across entities" is: **key by the entity, and choose your partition count as your parallelism target.** For an order-processing system, key by `orderId`; for a per-customer ledger, key by `customerId`; for a device telemetry stream, key by `deviceId`.

Choose the key at the granularity your ordering requirement actually demands. Keying by `customerId` when you only need per-order ordering unnecessarily serialises all of a customer's orders and increases skew risk.

*(Implementation note: with `null` keys, older clients used round-robin; since Kafka 2.4 the default is the **sticky partitioner**, which batches records to one partition at a time for better throughput. Ordering is unaffected — there is none to preserve without a key.)*

**Why repartitioning breaks the key→partition mapping**

`murmur2(key) % numPartitions` depends on `numPartitions`. Increase partitions from 12 to 24 and the modulus changes, so **existing keys are remapped to different partitions**. Records for `order-42` written before the change sit in partition 5; records written after go to partition 17. Two different consumers now process one order's history, **concurrently and out of order**.

This is not a transient glitch — it is a permanent break in your ordering guarantee for every key in flight at the moment of the change, and Kafka gives you no warning. Kafka also **cannot decrease** partition count at all (it would orphan data), so the operation is irreversible.

Mitigations:
- **Over-provision partitions up front.** They are cheap relative to a repartition. Size for expected growth over a few years, bearing in mind the per-partition costs: broker file handles and memory, longer leader-election and rebalance times, more end-to-end latency for replication, and more consumer-side buffering. Thousands of partitions per broker is workable on modern (KRaft) clusters; tens of thousands is a problem.
- **Custom partitioner with a stable mapping** — hash to a large fixed virtual-bucket space, then map buckets to partitions, so growth reassigns whole buckets rather than rehashing everything. Or use consistent hashing.
- **If you must repartition:** stop producers, drain all consumers to the end of every partition, then add partitions and resume. This trades a maintenance window for correctness.
- **Or migrate to a new topic** with the desired partition count and cut over, which is often cleaner and reversible.

**Hot partitions (key skew)**

Key-based partitioning assumes reasonably uniform key distribution. Real data rarely is: one enterprise customer generating 40% of events, one device malfunctioning and emitting at 1000×, a `null`-ish sentinel key used for many records. The result is one partition with a huge backlog and enormous consumer lag while eleven others are idle. **Consumer lag per partition** — not aggregate lag — is the metric that reveals this; aggregate lag hides it entirely.

Remedies, in increasing order of complexity:
- **Composite keys** — `customerId + ":" + (orderId.hashCode() % N)`, which splits one hot entity across N partitions. Only valid if your ordering requirement tolerates it (per-order rather than per-customer).
- **Isolate the hot tenant** onto a dedicated topic or a dedicated consumer group.
- **Custom partitioner** with explicit awareness of known-hot keys.
- **Rate-limit or shed at the producer** for pathological sources.

**One consumer per partition — and the parallelism ceiling**

Since a partition maps to one consumer, per-partition throughput is bounded by a single consumer's processing speed. If each record takes 200ms and you need 1000 records/sec, you need 200 partitions — unless you break the coupling. Options:

- **Parallelise within a partition** while preserving per-key order: the consumer polls a batch, groups records by key, and dispatches each key's records to a worker (e.g. a key-affine executor) so different keys run concurrently and same-key records run sequentially. Offsets must then be committed only up to the lowest continuously-completed offset, which is the hard part — libraries like Confluent's **Parallel Consumer** implement exactly this, and rolling your own is a genuine source of duplicate-processing bugs.
- **Reduce per-record work** — batch database writes, remove synchronous external calls from the consumer path, or hand off to a separate async stage.
- **Add partitions** (accepting the repartitioning cost above).

**Rebalancing note.** Because assignment is dynamic, adding or removing a consumer triggers a rebalance and partitions move between consumers. In-flight, uncommitted work may be reprocessed by the new owner — which is why consumers must be idempotent regardless of your ordering design. See Q82.

### Probes

**Key-based partitioning.** Covered, including the default `murmur2 % n` partitioner and choosing key granularity.

**One consumer per partition max.** Covered, with the parallelism ceiling and three ways around it.

**Ordering only within a partition.** Covered as the fundamental guarantee, with the reason no global ordering exists.

**Repartitioning breaks key→partition mapping.** Covered in detail, including that partitions cannot be decreased, and four mitigations.

**Hot partitions.** Covered — causes, why per-partition lag is the detecting metric, and four remedies.

---

## Q82. What happens during a rebalance, and how to make it less painful.

### Answer

**What a rebalance is.** The process by which the group coordinator reassigns partitions among the members of a consumer group. It is triggered by: a consumer joining (scale-up, deployment), a consumer leaving gracefully (`close()`/`LeaveGroup`), a consumer being evicted (heartbeat or poll timeout), a change in topic partition count, or a subscription change.

**The classic (eager) protocol — "stop-the-world"**

With the default `RangeAssignor` or `RoundRobinAssignor`:

1. The coordinator signals a rebalance; every consumer is told to rejoin.
2. **Every consumer revokes *all* of its partitions** — `onPartitionsRevoked` fires; the consumer typically commits offsets here.
3. All members send `JoinGroup`; one becomes the group leader.
4. The leader computes the assignment; the coordinator distributes it via `SyncGroup`.
5. `onPartitionsAssigned` fires; consumers resume.

Between steps 2 and 5, **the entire group consumes nothing**. On a large group with heavy state, this can be seconds to minutes. Adding one pod to a 30-pod group stops all 30. During a rolling deployment of 30 pods, this happens 60 times.

**Cooperative rebalancing** (`CooperativeStickyAssignor`, KIP-429, available since Kafka 2.4 and the default for Kafka Streams; for plain consumers set `partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor`) fixes this with **incremental** reassignment: partitions that are not moving are never revoked, so consumers keep processing them throughout. Only the partitions actually being transferred pause, and the rebalance happens in two lightweight passes. For a rolling deployment this is transformative — it is the single highest-impact change on this list.

*Migration caveat:* moving from eager to cooperative requires a **two-step rolling upgrade** (first deploy with both assignors configured, then a second deploy with only the cooperative one), because the protocols are incompatible within a group. Doing it in one step causes a failure. Knowing this is a good signal of real experience.

*Forward note:* **KIP-848** (the "next generation" consumer rebalance protocol, GA in Kafka 4.0) moves assignment computation to the broker-side coordinator and removes the global synchronization barrier entirely, making rebalances fully incremental and much faster. Worth mentioning as the direction of travel.

**`max.poll.interval.ms` vs `session.timeout.ms` — the distinction that matters**

These detect two different failures, and conflating them is the most common Kafka consumer misconfiguration.

| Setting | Default | Detects | Mechanism |
|---|---|---|---|
| `session.timeout.ms` | 45s (was 10s before 3.0) | The consumer **process** is dead or partitioned | Missed **heartbeats**, sent by a background thread every `heartbeat.interval.ms` (default 3s) |
| `max.poll.interval.ms` | 300s (5 min) | The consumer is **alive but stuck** — processing is taking too long | Time between successive `poll()` calls on the main thread |

Since Kafka 0.10.1, heartbeats run on a **background thread**, so a consumer whose processing thread is blocked for 10 minutes keeps heartbeating happily. That is exactly why `max.poll.interval.ms` exists: it catches the "alive but not making progress" case that heartbeats cannot.

**The slow-processing failure loop**, which the probe asks about:

1. A batch of 500 records (`max.poll.records` default) each takes 800ms → 400 seconds of processing.
2. `poll()` is not called again within `max.poll.interval.ms` (300s).
3. The consumer is **evicted** from the group. Its partitions are reassigned to another consumer, which starts from the **last committed offset**.
4. Meanwhile the evicted consumer finishes its batch and tries to commit → `CommitFailedException` ("This consumer has been kicked out of the group").
5. The reassigned consumer **reprocesses every record the evicted consumer already handled** — duplicates, potentially thousands.
6. The evicted consumer rejoins, triggering another rebalance. If processing is still slow, the cycle repeats: a **rebalance storm** in which the group spends more time rebalancing than consuming, lag grows monotonically, and duplicates multiply.

The signature in logs: repeated `Member ... sending LeaveGroup request ... due to consumer poll timeout has expired` and `Attempt to heartbeat failed since group is rebalancing`.

**Fixes:**
- **Reduce `max.poll.records`** so a batch finishes well within the interval. This is usually the correct first move — it is a one-line change with no downside beyond slightly more poll round-trips.
- **Raise `max.poll.interval.ms`** if processing legitimately takes long. The cost: a genuinely hung consumer now takes that long to be detected.
- **Make processing faster** — batch the database writes, remove synchronous HTTP calls, parallelise within the partition (Q81).
- **Decouple**: poll quickly onto a bounded queue with a worker pool, and `pause()`/`resume()` the partitions when the queue is full so you keep calling `poll()` (which is what maintains group membership) without fetching more. This is the robust pattern for genuinely long processing, and it is essentially what Spring Kafka's async/`ContainerPausingBackOffHandler` mechanisms and the Parallel Consumer do.

**Static membership** (`group.instance.id`, KIP-345, Kafka 2.3+)

Normally, a consumer that restarts gets a **new** member ID and is treated as a new member, triggering a full rebalance on shutdown *and* another on startup. With static membership, the consumer declares a stable identity:

```properties
group.instance.id=orders-consumer-3
session.timeout.ms=45000
```

The coordinator now remembers the assignment for that ID. If the consumer restarts and rejoins **within `session.timeout.ms`**, it receives its **same partitions back with no rebalance at all**. For a rolling deployment, this eliminates the rebalance per pod entirely, which is enormous.

Requirements and caveats: each instance needs a genuinely unique, stable ID (in Kubernetes, use a StatefulSet's ordinal, or the pod name via the downward API); `session.timeout.ms` must exceed your restart time, so you may need to raise it; a duplicate `group.instance.id` causes a `FencedInstanceIdException` that fences the older instance; and a permanently dead instance's partitions stay unassigned until the session times out, so detection of real failures is slower. Combine with cooperative rebalancing for the best result.

**Offset commit strategy**

- **`enable.auto.commit=true`** (default) commits asynchronously every `auto.commit.interval.ms` (5s), **based on what was polled, not on what was processed**. So a crash mid-batch loses records that were polled and committed but never processed — silent **message loss**, which is usually much worse than duplication. **Disable it** for anything that matters.
- **Manual commit after processing** (`enable.auto.commit=false`, then `commitSync()`/`commitAsync()`, or Spring Kafka's `AckMode.RECORD`/`BATCH`/`MANUAL`) gives at-least-once: the record is processed, then the offset is committed, so a crash between the two causes reprocessing rather than loss. This is the correct default.
- `commitSync` is safe but blocks; `commitAsync` is faster but may not complete before a rebalance. The standard idiom is `commitAsync` in the loop and `commitSync` in `finally`/`onPartitionsRevoked` to flush before losing the partition.
- In `onPartitionsRevoked` (or `onPartitionsLost` for the cooperative protocol's abrupt case), commit what you have finished — this is the only chance to narrow the duplicate window.
- Because at-least-once is the honest guarantee, **the consumer must be idempotent** (Q56, Q76, Q83) — every technique above narrows the duplicate window, none closes it.

**Graceful shutdown.** Call `consumer.close()` (Spring Kafka does this on context shutdown) so the consumer sends an explicit `LeaveGroup`, triggering an immediate rebalance rather than making the group wait out `session.timeout.ms`. Pair with Kubernetes graceful shutdown (Q64/Q68) and a `terminationGracePeriodSeconds` long enough to finish the in-flight batch and commit.

### Probes

**Stop-the-world vs cooperative-sticky.** Covered, including the two-step migration requirement and KIP-848 as the future.

**`max.poll.interval.ms` vs `session.timeout.ms`.** Covered in a comparison table with the background-heartbeat-thread explanation of why both exist.

**Slow processing → eviction + duplicates.** Covered as a six-step failure loop with the log signature and four fixes.

**Static membership.** Covered, with configuration, Kubernetes identity sourcing, and the caveats.

**Offset commit strategy.** Covered — why auto-commit risks loss, manual commit patterns, `commitSync` vs `commitAsync`, revocation-time commits, and the residual need for idempotency.

---

## Q83. Is "exactly-once" real in Kafka?

### Answer

**The short answer:** exactly-once **delivery** is impossible; exactly-once **processing** is real but only within a bounded scope — Kafka-to-Kafka. The moment an external system is involved, you are back to at-least-once plus idempotency.

**Why delivery cannot be exactly-once.** Over an unreliable network, a sender that gets no acknowledgement cannot distinguish "the message was lost" from "the message arrived and the ack was lost". It must either resend (risking duplicates — at-least-once) or not (risking loss — at-most-once). This is the Two Generals problem, and it is a proof, not an engineering limitation. Any product claiming exactly-once delivery is either wrong or redefining the term.

What Kafka actually provides is **exactly-once semantics (EOS)**: duplicates may be *delivered*, but their *effects* occur once, within the systems Kafka controls.

**The two mechanisms**

**1. Idempotent producer** (`enable.idempotence=true`, the **default since Kafka 3.0**).

Each producer gets a **Producer ID (PID)** and attaches a monotonic **sequence number** per partition. The broker tracks the last sequence per (PID, partition) and rejects duplicates. So a producer retry after a lost ack does not create a second copy on the log. It also preserves ordering under retries — which is why `max.in.flight.requests.per.connection` may be up to 5 with idempotence enabled without risking reordering (without it, retries could reorder). Requires `acks=all` and `retries > 0`, which idempotence sets implicitly.

Scope: this solves **producer-side duplicates within a session and a partition**. It does not deduplicate application-level double-sends of logically distinct records, and (before KIP-890 hardening) PID expiry after `transactional.id.expiration.ms` could theoretically allow duplicates across very long gaps.

**2. Kafka transactions** (`transactional.id` set; `beginTransaction`/`sendOffsetsToTransaction`/`commitTransaction`).

A transaction atomically commits (a) a set of records written to one or more topics/partitions **and** (b) the consumer offsets for the records that produced them. Because the offsets are stored in a Kafka topic (`__consumer_offsets`), Kafka can commit both atomically via its transaction coordinator and two-phase commit markers on the log.

```java
producer.initTransactions();
while (true) {
    var records = consumer.poll(d);
    producer.beginTransaction();
    for (var r : records) producer.send(transform(r));
    producer.sendOffsetsToTransaction(offsetsOf(records), consumer.groupMetadata());
    producer.commitTransaction();   // records + offsets commit atomically, or neither
}
```

This is the **consume–transform–produce** loop, and it is where EOS genuinely works. Kafka Streams implements it for you with one setting: `processing.guarantee=exactly_once_v2`.

**`isolation.level=read_committed`** is the required consumer-side counterpart: it makes the consumer skip records from aborted transactions and not read past the **last stable offset** (the point before any open transaction). Without it (`read_uncommitted`, the default), the consumer sees aborted records and the guarantee is void. Note the trade-off: `read_committed` adds latency, because a consumer cannot read past an in-flight transaction until it commits — so a long-running transaction stalls downstream consumers.

**Why it does not extend to an external database**

Kafka's transaction coordinator has no authority over PostgreSQL. Committing a Kafka transaction and committing a database transaction are two independent commits, so you are back to the dual-write problem of Q76:

```java
producer.beginTransaction();
orderRepository.save(order);      // DB commit — Kafka knows nothing about this
producer.send(event);
producer.commitTransaction();     // crash between the two commits → inconsistent
```

There is no ordering of the two commits that is safe. XA/2PC across Kafka and a database is not supported and would be a poor idea anyway. **So end-to-end exactly-once with an external system requires either an idempotent consumer or the outbox pattern.**

**End-to-end: at-least-once + a dedupe key (the pattern you actually build)**

For the **consumer** side writing to a database:

```java
@Transactional
public void handle(ConsumerRecord<String, OrderEvent> r) {
    String eventId = r.value().eventId();               // stable, producer-assigned
    try {
        processedEvents.insert(eventId);                // UNIQUE constraint on event_id
    } catch (DuplicateKeyException e) {
        return;                                          // already handled — skip
    }
    orderRepository.save(...);                           // business write
}   // dedupe record and business write commit together — the essential part
```

The dedupe record **must commit in the same database transaction as the business write**. Otherwise a crash between them either loses the business effect or permanently suppresses it. As always, the unique constraint — not a `SELECT` check — is the concurrency control (Q56).

For the **producer** side writing from a database, use the **transactional outbox** (Q76).

Alternatives to an explicit dedupe table:
- **Naturally idempotent operations** — an upsert (`INSERT ... ON CONFLICT DO UPDATE`), a `SET` rather than an increment, or a state transition guarded by the current state (`UPDATE orders SET status='SHIPPED' WHERE id=? AND status='PAID'`). Often the cleanest solution: design the operation so replaying it is harmless.
- **Offset-based dedupe** — store the last processed `(topic, partition, offset)` alongside the business data and skip anything at or below it. Works only when a partition is processed strictly serially, and breaks under the parallel-within-partition designs of Q81.
- **Version/sequence checks** — reject an event whose version is not the expected next one, which also catches out-of-order delivery.

Practical notes on the dedupe table: it needs a TTL/partitioning strategy or it grows without bound; the retention window must exceed the maximum plausible redelivery delay (topic retention, DLQ replay windows); and it must be keyed on a **producer-assigned** event ID, not on something derived at consumption time.

**Cost of EOS.** Transactions add coordinator round-trips and commit markers; `read_committed` adds latency. Real-world overhead is often modest (Confluent has published figures in the low tens of percent for throughput), but it is not free, and it constrains your design (a transaction spans one producer, so you cannot easily share producers across threads). Use `exactly_once_v2` in Kafka Streams where it is nearly free to enable, and be deliberate elsewhere.

**The honest summary for an interview:** "Kafka gives exactly-once *processing* for Kafka-to-Kafka pipelines via the idempotent producer plus transactions plus `read_committed`, and Kafka Streams makes that a one-line setting. As soon as a database or an external API is in the path, that guarantee stops at the boundary, and the real design is at-least-once delivery plus an idempotent consumer keyed on a stable event ID, with the dedupe record committed in the same transaction as the business write — and an outbox on the producing side."

### Probes

**Idempotent producer + Kafka transactions.** Covered — PIDs and sequence numbers, the consume–transform–produce loop, `sendOffsetsToTransaction`, and `exactly_once_v2`.

**End-to-end with an external DB needs idempotent consumers or an outbox.** Covered, with the explanation of why Kafka's coordinator cannot span the database.

**At-least-once + dedupe key.** Covered with working code, the same-transaction requirement, three alternative dedupe strategies, and the operational notes on the dedupe table.

**`read_committed`.** Covered, including the last-stable-offset latency trade-off.

---

## Q84. Dead-letter strategy — poison messages, retries, and replay.

### Answer

**The problem: a poison message.** A record that fails processing every time — malformed payload, a schema the consumer cannot deserialise, a referenced entity that no longer exists, a bug triggered by a specific value. Because Kafka does not remove records and a consumer cannot "skip" without committing, naive retry-in-place means the consumer retries the same record forever, **never advances the offset, and blocks every subsequent record in that partition**. Lag grows without bound and an entire partition's traffic is halted by one bad record. This is the failure the whole strategy exists to prevent.

**Transient vs permanent failures — the classification that drives everything**

| | Transient | Permanent |
|---|---|---|
| Examples | Network timeout, downstream `503`, DB deadlock, connection pool exhausted, rate-limited | Deserialisation failure, schema violation, validation failure, `404` on a required entity, a null field the code requires, a business rule violation |
| Right response | **Retry** with backoff — it will probably succeed | **Do not retry** — it will fail identically every time |
| Wrong response | Sending to DLQ immediately wastes a recoverable message and creates manual work | Retrying wastes time, generates noise, and delays everything behind it |

Implement this explicitly: classify the exception and route accordingly. In Spring Kafka:

```java
var backOff = new ExponentialBackOffWithMaxRetries(4);       // ~1s, 2s, 4s, 8s
backOff.setInitialInterval(1000);
backOff.setMultiplier(2.0);

var handler = new DefaultErrorHandler(deadLetterRecoverer, backOff);
handler.addNotRetryableExceptions(                            // straight to DLQ
        DeserializationException.class,
        MethodArgumentNotValidException.class,
        ValidationException.class);
handler.addRetryableExceptions(
        TransientDataAccessException.class,
        ResourceAccessException.class);
```

Deserialisation failures deserve special mention: they occur *before* your listener runs, so they cannot be caught in your code. Use `ErrorHandlingDeserializer` wrapping your real deserialiser, which converts the failure into a header the error handler can route to the DLQ — otherwise the container fails to deserialise the same record forever.

**Blocking vs non-blocking retry — and the ordering cost**

**Blocking retry** (the default `DefaultErrorHandler`): the consumer retries the record in place, sleeping between attempts, and **does not poll** meanwhile.
- *Preserves ordering* within the partition — nothing overtakes the failing record.
- *Blocks the whole partition* for the duration of the retries. With 4 retries at up to 8s, that is ~15 seconds of complete partition stall. With longer backoffs it is catastrophic.
- **Careful:** total blocking retry time must stay under `max.poll.interval.ms`, or you are evicted from the group (Q82). Spring Kafka's `DefaultErrorHandler` handles this by pausing the container between polls, but the constraint remains real when rolling your own.

**Non-blocking retry** (`@RetryableTopic`): the failing record is immediately forwarded to a dedicated **retry topic** with increasing delay, and the consumer commits and moves on.

```java
@RetryableTopic(
    attempts = "4",
    backoff = @Backoff(delay = 1000, multiplier = 3.0),      // 1s, 3s, 9s
    topicSuffixingStrategy = TopicSuffixingStrategy.SUFFIX_WITH_INDEX_VALUE,
    dltStrategy = DltStrategy.FAIL_ON_ERROR,
    exclude = { DeserializationException.class })
@KafkaListener(topics = "orders")
public void handle(OrderEvent e) { ... }
// creates: orders-retry-0, orders-retry-1, orders-retry-2, orders-dlt
```

- *The main partition keeps flowing* — one bad record no longer stalls thousands of good ones. This is usually the correct trade-off.
- **The ordering cost is the point of the probe:** a record moved to a retry topic and processed 9 seconds later is now processed **after** records that came behind it in the original partition. If you have a per-entity ordering requirement (Q81), non-blocking retry **breaks it**. Two records for the same order can be applied out of sequence, producing genuinely wrong state.
- Mitigations when ordering matters: use blocking retry with a short, bounded backoff; or make handlers order-independent (idempotent, commutative, or version-guarded — `UPDATE ... WHERE version = expected` so an out-of-order event is rejected rather than applied); or pause the specific key. **State the trade-off explicitly in an interview: you cannot have both non-blocking retry and strict per-key ordering.**

**The DLQ (dead-letter topic)**

Records that exhaust retries go to a dead-letter topic. Spring Kafka's `DeadLetterPublishingRecoverer` copies the original key, value, and headers and adds diagnostics: `kafka_dlt-original-topic`, `-original-partition`, `-original-offset`, `-original-timestamp`, `-exception-fqcn`, `-exception-message`, `-exception-stacktrace`.

Design points:
- **Preserve the original key** so replayed records still partition correctly.
- **Retain the full original payload bytes**, not a parsed form — a deserialisation failure means you cannot parse it, and you need the raw bytes to diagnose and replay.
- **Set generous retention** on the DLQ (weeks, not the default), because triage may take days.
- One DLQ per topic (or per consumer group) rather than one global DLQ, so ownership and replay are unambiguous.

**Replay tooling — the part teams skip and regret**

A DLQ with no replay path is a graveyard, and this is where the probe earns its place. What you need:

1. **Inspection** — a way to browse DLQ records with their error headers, without a developer writing a one-off consumer each time (AKHQ, Conduktor, Redpanda Console, or an internal admin UI).
2. **Selective replay** — republish chosen records back to the source topic (or to a dedicated replay topic) after the bug is fixed or the data corrected. Filtering by exception type, time range, or key matters: replaying all 50,000 records when only 12 are relevant creates a new incident.
3. **Idempotency on replay** — replayed records will be reprocessed, and some may have partially succeeded before failing. The consumer must be idempotent (Q83) or replay is dangerous.
4. **A replay audit trail** — who replayed what, when, and why. Replays touch production data.
5. **A discard path** — some records are genuinely unprocessable and must be archived and dropped, with a record of the decision.
6. **Rate limiting on replay** — dumping a large DLQ back into a live topic can overwhelm consumers and downstream systems.

**Alerting on DLQ depth**

A DLQ that nobody watches is worse than no DLQ, because it converts loud failures into silent data loss. Minimum alerting:
- **Any** record arriving in a DLQ should page or ticket, depending on the topic's criticality. For low-volume, high-value streams (payments), alert on the first record.
- **Rate of arrival** — a sudden spike means a systemic problem (a bad deploy, a schema change upstream, a downstream outage), not a data problem, and it needs a different response.
- **Total depth and age of the oldest record** — surfaces a DLQ that is quietly accumulating.
- **Retry-topic lag** — high lag on a retry topic means retries are failing en masse.

Instrument the *classification* too: a metric of permanent-vs-transient failures per topic tells you whether your classification rules are right. A steady stream of "transient" failures exhausting all retries usually means they were misclassified.

**A note on scope.** Everything above is the standard pattern, but it is worth saying in an interview that the best fix for poison messages is often upstream: **schema enforcement** (a schema registry with compatibility checks, so an incompatible producer cannot publish at all) prevents a large fraction of deserialisation failures from ever occurring. A DLQ is a safety net, not a substitute for a contract.

### Probes

**Poison messages.** Covered, including why they block a partition indefinitely and why deserialisation failures need `ErrorHandlingDeserializer`.

**Retry topics with increasing delay.** Covered with `@RetryableTopic` configuration.

**Blocking vs non-blocking retry and the ordering cost.** Covered as the central trade-off, with the `max.poll.interval.ms` constraint and three mitigations when ordering matters.

**DLQ replay tooling.** Covered as six concrete requirements.

**Alerting on DLQ depth.** Covered, with four distinct signals and the classification metric.

**Transient vs permanent failures.** Covered in a table and implemented in the error-handler configuration.

---

## Q85. Consumer lag is growing — diagnostic path.

### Answer

**What lag is.** For each partition, `lag = log end offset − committed consumer offset`: the number of records produced but not yet acknowledged as processed. Growing lag means **consumption rate is below production rate**. The whole diagnosis is: which of the two changed, and why.

**Step 0 — look at lag per partition, not aggregate.**

The first question is *shape*:
- **All partitions lagging evenly** → the consumer group as a whole is too slow, or production rose. A capacity/throughput problem.
- **One or a few partitions lagging** → **key skew / hot partition** (Q81), or one unhealthy consumer instance, or one partition on a struggling broker.
- **Lag flat but high** → consumption is keeping pace but never catching up from an earlier backlog; you need temporary excess capacity, not a fix.
- **Sawtooth pattern** → periodic slowness (a batch job, a cache expiry, a GC cycle) or repeated rebalances.

Get it from `kafka-consumer-groups.sh --describe --group X`, Burrow, or the `kafka_consumergroup_lag` metric per partition in Prometheus.

**Step 1 — did production rate change, or did consumption rate?**

This single comparison splits the problem space in half, and skipping it is why people spend hours tuning a consumer that was never the problem.
- **Produced records/sec rose** (a marketing campaign, a backfill, an upstream retry storm, a new producer, a bug producing duplicates) → a **load** problem. Response: scale consumers (up to the partition count), rate-limit the producer, or accept the backlog and let it drain.
- **Produced rate is flat and consumed rate fell** → a **regression** on the consumer side. Correlate the inflection point with your deployment timeline. This is where most incidents land, and "what changed at 14:20?" is usually the fastest route to the answer.

**Step 2 — if the consumer slowed, is it your processing or a downstream dependency?**

Compare **records/sec per consumer** with **processing time per record**, and instrument the two separately:
- If per-record processing time rose and CPU rose with it → your code, a data-shape change, or GC.
- If per-record time rose while CPU is **flat and low** → you are **blocked on I/O**. This is the most common case. Look at:
  - **Downstream latency** — a slow database, a slow HTTP dependency, a slow cache. Check that service's own latency metrics, and check whether *its* dependencies degraded.
  - **Connection pool saturation** — `hikaricp_connections_pending` and acquisition timeouts. Adding consumer threads here makes it worse.
  - **Lock contention** or an exhausted internal thread pool.
  - A new synchronous call added to the consumer path — an extremely common regression.

Practical tooling: a thread dump (`jcmd Thread.print`) of a lagging consumer immediately shows whether threads are parked on a socket read, waiting on a connection, or actually running. An async profiler in wall-clock mode is the definitive answer.

**Step 3 — check for rebalancing storms.**

Repeated rebalances make lag grow while the consumer *appears* healthy. Symptoms: log lines about `poll timeout has expired`, `group is rebalancing`, or members joining/leaving; a `kafka_consumer_coordinator_rebalance_total` counter climbing; lag climbing in steps rather than smoothly. Causes and fixes are Q82: `max.poll.records` too high for the per-record cost, `max.poll.interval.ms` too low, pods being restarted (a `CrashLoopBackOff`, OOMKills, an HPA scaling repeatedly), a rolling deployment with eager rebalancing, or network instability. Confirm by checking pod restart counts — a lagging consumer group whose pods are restarting every four minutes is not a throughput problem.

**Step 4 — check the partition-count ceiling.**

If consumers ≥ partitions, **adding pods does nothing** — the extra ones are idle. Verify with `kafka-consumer-groups.sh --describe`, which shows the consumer ID assigned to each partition; idle members appear with no assignment. If you are at the ceiling and each record's work is irreducible, your options are: increase partitions (accepting the repartitioning hazards of Q81), parallelise within a partition while preserving per-key order (the Parallel Consumer approach), or make the per-record work cheaper.

**Step 5 — tune the consumer's fetch and batch settings.**

Often overlooked, and sometimes the whole answer:
- **`max.poll.records`** (default 500) — the classic tension. Too high and the batch takes longer than `max.poll.interval.ms` (eviction, Q82); too low and you pay poll round-trip overhead per record and lose the ability to batch downstream writes.
- **`fetch.min.bytes`** (default 1) and **`fetch.max.wait.ms`** (default 500) — raising `fetch.min.bytes` lets the broker accumulate a larger batch before responding, dramatically improving throughput at the cost of latency. For a lagging consumer, throughput is what you want.
- **`max.partition.fetch.bytes`** / **`fetch.max.bytes`** — cap the data returned per poll; too small starves the consumer.
- **Batch your downstream writes.** Processing 500 records with 500 individual `INSERT`s is the single most common consumer performance bug. One batched `INSERT` (or a `COPY`) is often 10–50× faster. Similarly, batch HTTP calls where the API supports it.
- **`receive.buffer.bytes`** and compression settings matter for high-throughput, high-latency links.

**Step 6 — check the broker and infrastructure side.**

Less common, but real: an under-replicated partition, a broker with a failing disk, a leader election in progress, cross-AZ network saturation, or a topic whose partitions are unevenly distributed across brokers. Check `UnderReplicatedPartitions`, broker request-handler idle ratio, and per-broker network throughput.

**Immediate mitigation while you diagnose.** Lag is often an active incident, so have a response ready in parallel with the investigation:
- Scale consumers to the partition count if not already there.
- Temporarily raise `fetch.min.bytes` / `max.poll.records` for throughput if per-record cost is low.
- Shed load: if some message types are lower value, route them to a separate topic and deprioritise.
- If the data is genuinely disposable and the backlog is unrecoverable, seek offsets forward — but this is **data loss**, requires an explicit decision by an accountable owner, and should never be a reflex.
- If the consumer is producing bad output, stop it rather than letting it process a backlog incorrectly.

**Prevention.** Alert on lag *and* on lag **rate of change** — a slowly climbing lag caught early is routine; the same lag noticed at 4 million records is an incident. Alert on time-based lag where possible (`records behind × average processing time` ≈ how far behind in *time* you are), because that is what a business stakeholder actually cares about. And load-test consumers at realistic production rates, since the failure is almost always at the throughput ceiling nobody measured.

### Probes

**Slow processing vs downstream latency.** Covered as step 2, with the CPU-flat-but-slow signature and thread-dump/profiler tooling.

**Partition count as a scaling ceiling.** Covered as step 4, with how to verify it and the three options once you hit it.

**Batch size / `max.poll.records`.** Covered as step 5, including the fetch settings and the "batch your downstream writes" point.

**Parallelism within a partition.** Covered, cross-referenced to Q81.

**Rebalancing storms.** Covered as step 3, with the diagnostic signature.

**Producer spike vs consumer regression.** Covered as step 1 — the first comparison to make, and the one that halves the search space.

---

# 12. Resilience, Caching, Observability, Security

---

## Q86. A downstream dependency degrades from 50ms to 5s. Trace what happens to your service.

### Answer

**The cascade, step by step.** Assume a Spring Boot service on Tomcat with 200 request threads, a HikariCP pool of 20 connections, and an endpoint that calls the degraded dependency.

**t=0 — latency rises, throughput does not.** Requests keep arriving at the same rate. Each one now occupies a thread for 5 seconds instead of 50ms — a **100× increase in thread occupancy time**.

**t≈2s — thread pool exhaustion.** Little's Law makes this precise: `L = λW`, where `L` is concurrent requests in the system, `λ` is arrival rate, and `W` is time in the system. At 100 req/s and 50ms, `L = 5` — five threads busy out of 200, comfortable. At 100 req/s and 5s, `L = 500` — but you only have 200 threads. The pool is exhausted almost immediately, and the arithmetic tells you exactly when: `200 / 100 = 2 seconds`.

**t≈2s+ — the accept queue grows.** Requests that cannot get a thread queue in Tomcat's `acceptCount` backlog (default 100). Once *that* fills, the OS refuses connections and clients see connection-refused or connection-timeout errors.

**The critical consequence: every endpoint is now down.** The threads are a *shared* resource. An endpoint that has nothing to do with the degraded dependency — a health check, a cached lookup, a completely unrelated feature — cannot get a thread either. **One slow dependency has taken down the entire service.** This is the single most important thing to articulate, because it explains why bulkheads exist.

**The connection pool amplifies it.** If the call happens while holding a database connection (a `@Transactional` method making an external call — the anti-pattern of Q28), then 20 requests exhaust the Hikari pool and the remaining 180 threads block on `getConnection()` until `connectionTimeout` (default 30s) expires. Now you have two exhausted resources and the database is holding 20 transactions open for 5 seconds each, growing its own lock contention and, in PostgreSQL, blocking vacuum by holding old snapshots.

**t≈10s — cascading failure upward.** Your callers now experience *your* service at 5s or as connection-refused. If they lack timeouts and bulkheads, the same collapse happens to them. The failure propagates up the call graph until it reaches the user. A single degraded leaf service takes down the tree.

**Retries make it worse, not better.** If your callers retry 3×, your arrival rate triples exactly when you are least able to serve it — a **retry storm** (Q80). The system converges on total failure rather than partial degradation.

**Kubernetes then finishes the job.** Health checks queue behind real requests and time out. If liveness probes hit the same thread pool and the same aggregate `/actuator/health` (Q66), Kubernetes concludes the pods are dead and **restarts them all** — dropping in-flight work, emptying caches, and creating a cold-start thundering herd against the already-struggling dependency.

**"No timeout" is the most common production bug**

Java's defaults are hostile here. `HttpURLConnection`, many JDBC drivers, and older HTTP clients default to **infinite** connect and read timeouts. `RestTemplate` with a default `SimpleClientHttpRequestFactory` has no timeouts. A call with no timeout does not fail — it waits forever, holding a thread, and no circuit breaker ever observes an outcome to record.

Set timeouts at **every** layer, explicitly:

```java
@Bean
RestClient ordersClient(RestClient.Builder b) {
    var f = new SimpleClientHttpRequestFactory();   // or JdkClientHttpRequestFactory / Reactor
    f.setConnectTimeout(Duration.ofSeconds(2));
    f.setReadTimeout(Duration.ofSeconds(3));
    return b.requestFactory(f).baseUrl("https://orders").build();
}
```

- **Connect timeout** — short (1–2s). If a TCP handshake takes longer, the host is unreachable.
- **Read/response timeout** — derived from the dependency's actual p99, not a guess. A common error is setting it to 30s "to be safe", which guarantees thread exhaustion.
- **Connection-acquisition timeout** on the HTTP connection pool *and* the JDBC pool (Hikari `connectionTimeout`).
- **Statement/query timeout** at the JDBC layer (`javax.persistence.query.timeout`, `spring.jpa.properties.jakarta.persistence.query.timeout`, or PostgreSQL `statement_timeout`) — a slow *query* is the same failure mode as a slow HTTP call.
- **Transaction timeout** (`@Transactional(timeout = 5)`).
- **A total request budget** at the edge, so an endpoint cannot exceed its SLO regardless of what it calls.

**Timeout budgets must decrease with depth.** If A (10s timeout) calls B (10s) calls C (10s), then when C is slow, B is still waiting at t=10s while A has already given up — B's work is wasted and it is still holding resources. Each hop should get a strictly smaller budget than its caller, with the remaining budget ideally propagated in a header (gRPC deadlines do this natively; for HTTP you can pass a `X-Request-Deadline` or use the `grpc-timeout` convention).

**Bulkhead — isolating the blast radius**

Named after ship compartments: partition resources so a flood in one does not sink the vessel. Give each dependency a **bounded** share of concurrency:

```java
@Bulkhead(name = "inventory", type = Bulkhead.Type.SEMAPHORE)   // Resilience4j
```
```yaml
resilience4j.bulkhead.instances.inventory:
  maxConcurrentCalls: 20
  maxWaitDuration: 100ms
```

Now at most 20 threads can be blocked on the inventory service; the 21st is rejected instantly. The other 180 threads remain available for every other endpoint. **This is what stops one dependency's degradation from being a total outage.** Two flavours: a semaphore bulkhead (cheap, caller's thread) and a thread-pool bulkhead (isolates fully, allows timeouts on the pool, costs a context switch — Hystrix's model).

Separate connection pools per dependency serve the same purpose at the I/O layer. And note that **virtual threads do not solve this** (Q14): they remove the thread-count ceiling but the constrained downstream resource is still constrained — you just get 10,000 virtual threads all blocked instead of 200, and you must add explicit admission control that the thread pool was previously providing implicitly.

**Circuit breaker — failing fast once it is systemic**

Once the failure rate or the **slow-call rate** crosses a threshold, stop calling entirely (Q80). The slow-call threshold is the relevant one here: the dependency is *succeeding*, just slowly, so a failure-rate-only breaker never trips while every thread blocks. Configure `slowCallDurationThreshold` at or slightly above your read timeout's intent, and `slowCallRateThreshold` around 50%. When open, either fail fast with `503` + `Retry-After` or serve a deliberate fallback.

**Load shedding — protecting yourself at the front door**

The last line of defence. When your own queue depth or concurrency exceeds what you can serve within your SLO, **reject new work immediately with `503`** rather than accepting it and failing slowly. Rejecting in 1ms is vastly better than timing out in 5s: it frees resources, gives the client a fast, actionable signal, and keeps the requests you *did* accept within SLO. Implement as a concurrency limiter (Netflix's `concurrency-limits` library implements adaptive limits using TCP-Vegas-style algorithms), a bounded queue with a rejection policy, or a gateway-level limit. Prioritise if you can — shed background and low-value traffic before user-facing traffic.

**The complete defensive stack, in order:** timeouts everywhere → bulkheads per dependency → circuit breakers with slow-call detection → bounded, jittered retries gated by the breaker → load shedding at the edge → liveness probes that do not depend on downstreams → alerting on saturation metrics (thread pool utilisation, pool pending count, queue depth) rather than only on errors, since saturation precedes failure.

### Probes

**Thread/connection pool exhaustion.** Covered with Little's Law arithmetic and the exact time to exhaustion.

**Queue growth.** Covered — Tomcat's accept queue, then connection refusal.

**Cascading failure.** Covered — upward propagation, retry amplification, and Kubernetes restarting healthy pods.

**Timeouts at every layer.** Covered as an enumerated list, plus the decreasing-budget-with-depth rule.

**Bulkhead.** Covered, including both Resilience4j types and why virtual threads do not remove the need.

**Circuit breaker.** Covered, with the point that only slow-call detection catches this specific failure.

**Load shedding.** Covered, including adaptive concurrency limits and prioritised shedding.

**"No timeout" as the most common production bug.** Covered — Java's infinite defaults and why a call with no timeout is invisible to a circuit breaker.

---

## Q87. Design a cache for a read-heavy endpoint. What breaks?

### Answer

**Start by asking what you're caching and what staleness costs.** The design follows from that answer, so state it first: caching a product catalogue where a 5-minute-old price is acceptable is a completely different problem from caching an account balance where it is not. If the business cost of staleness is unknown, that is the first thing to establish — not a technical question but the one that determines every subsequent choice.

**Cache-aside (lazy loading) — the default**

```java
public Product get(long id) {
    var cached = cache.getIfPresent(id);
    if (cached != null) return cached;
    var p = repository.findById(id).orElseThrow();
    cache.put(id, p);
    return p;
}
```
The application owns the logic. Only requested data is cached. A cache failure degrades to slower reads, not to errors. On a write, you **invalidate** rather than update (updating races with concurrent reads and can leave stale data permanently). This is the right default for the large majority of cases.

**Write-through / write-behind**

*Write-through*: writes go to the cache, which synchronously writes to the store. The cache is never stale, but every write pays cache latency, and you cache data that may never be read.
*Write-behind*: the cache acknowledges the write and persists asynchronously. Fast writes, but a cache failure loses data — acceptable only for genuinely tolerant data (view counts, last-seen timestamps), never for a system of record.

**Read-through** is cache-aside with the loading logic inside the cache (`Caffeine.build(loader)`), which is cleaner and gives you single-flight for free (below).

**TTL, and why it needs jitter**

Every entry gets a TTL as a correctness backstop — it bounds staleness even when invalidation fails, which it will. But a fixed TTL creates **synchronised expiry**: if 10,000 entries were populated during a deployment or a cache flush, they all expire at the same instant, and the next moment 10,000 requests miss simultaneously and hit the database. This is a **cache stampede** (or "thundering herd").

Add randomisation: `ttl = base + random(0, base × 0.2)`. It costs nothing and spreads the load. Caffeine supports this via a custom `Expiry`; with Redis, jitter the TTL at write time.

Related: **refresh-ahead** (`refreshAfterWrite` in Caffeine) reloads an entry asynchronously on access *after* a threshold, while continuing to serve the old value. The requesting thread never blocks on a reload, so p99 stays flat. This is often better than pure expiry for hot keys.

**Single-flight on miss — the essential protection**

Even with jitter, a *single* hot key expiring can cause hundreds of concurrent requests to all miss and all query the database for the same row. Coalesce them so exactly one load happens and the rest wait for its result:

```java
LoadingCache<Long, Product> cache = Caffeine.newBuilder()
    .maximumSize(50_000)
    .expireAfterWrite(Duration.ofMinutes(10))
    .refreshAfterWrite(Duration.ofMinutes(8))
    .recordStats()
    .build(id -> repository.findById(id).orElseThrow());
```

Caffeine's `LoadingCache` guarantees the mapping function runs **at most once per key** for concurrent misses — this is `ConcurrentHashMap.computeIfAbsent` semantics, and it is the single most valuable property of using a real cache library rather than a `Map`. For a *distributed* cache the equivalent is a short-lived Redis lock (`SET key:lock NX PX 5000`) around the load, or accepting that N instances each do one load (usually fine).

Two important caveats on `computeIfAbsent`: the loader must not be slow enough to block the map's bin for long, and it must not recursively access the same cache — that deadlocks.

**Negative caching**

If a key genuinely does not exist and you do not cache that fact, every request for it is a guaranteed database miss. An attacker (or a broken client looping over IDs) can trivially bypass your cache entirely — **cache penetration**. Cache the absence, with a *shorter* TTL than positive entries (30–60s), because a newly created entity should become visible quickly. Use a sentinel value or `Optional` rather than `null`, since most cache APIs cannot distinguish "cached null" from "absent". For very large key spaces, a **Bloom filter** in front of the cache answers "definitely not present" cheaply.

**The `@Cacheable` self-invocation trap**

```java
@Service
public class ProductService {
    @Cacheable("products")
    public Product get(long id) { ... }

    public List<Product> getAll(List<Long> ids) {
        return ids.stream().map(this::get).toList();   // BUG: no caching at all
    }
}
```

Spring's caching is implemented by a **proxy** (Q25/Q26). `this.get(id)` is a direct call on the target object, bypassing the proxy entirely — so the cache is never consulted and never populated. The method appears cached; it isn't. Identical mechanics to `@Transactional` self-invocation, and equally easy to miss because there is no error.

Fixes, in order of preference: **move the cached method to a separate bean** (best — the boundary becomes explicit); inject a self-reference (`@Lazy ProductService self`) and call `self.get(id)`; use `AopContext.currentProxy()` (requires `exposeProxy = true`, ugly); or switch to AspectJ load-time weaving (heavy). Related traps: `@Cacheable` on a `private`, `final`, or `static` method silently does nothing under CGLIB, and the default key generator uses **all** parameters, which surprises people when they add a parameter and the hit rate collapses.

**Caffeine vs Redis vs two-level**

| | Caffeine (local) | Redis (distributed) |
|---|---|---|
| Latency | ~50–100ns, in-heap | ~0.5–2ms, network round trip + serialisation |
| Consistency | **Per-instance** — 20 pods hold 20 divergent copies | Single shared view |
| Invalidation | Hard — must broadcast to all instances | Trivial — delete one key |
| Capacity | Bounded by heap; competes with your application for memory and GC | Independently scalable, GBs |
| Failure mode | Cannot fail independently | A network partition or Redis outage must degrade gracefully |
| Warm-up | Cold on every pod start and every deploy | Survives deployments |

**Caffeine** is the right choice for small, hot, read-mostly, staleness-tolerant data — reference data, feature flags, configuration, per-request memoisation. Use `maximumSize` or `maximumWeight` (never unbounded; an unbounded cache is a memory leak with a nice name), and prefer **W-TinyLFU** (Caffeine's default eviction) over LRU — it has materially better hit rates on skewed workloads.

**Redis** is right when you need shared state, invalidation across instances, capacity beyond the heap, or survival across deployments. Costs: a network hop, serialisation (which is often the dominant cost — prefer a compact format over Java serialisation), and a new dependency whose failure must not take you down (set short timeouts, and decide fail-open vs fail-closed deliberately — for a *cache*, fail-open to the database is almost always correct, but be aware that means a Redis outage transfers full load to the database, so the database must be able to survive it).

**Two-level (L1 Caffeine + L2 Redis)** gives near-zero latency for the hottest keys and shared capacity behind them. The cost is the hardest problem in caching: **invalidating L1 across all instances.** You need a pub/sub invalidation channel (Redis pub/sub, a Kafka topic, or Redis client-side caching with RESP3 invalidation push), and you must accept a propagation window during which instances disagree. Only take this on when you have measured that L2 latency is actually your problem.

**What breaks — the failure catalogue**

1. **Stampede / thundering herd** — mass expiry or a cold cache after deployment. Fixes: jittered TTL, single-flight, refresh-ahead, and pre-warming critical keys at startup (but beware the herd *that* creates).
2. **Penetration** — repeated misses on non-existent keys. Fix: negative caching, Bloom filter.
3. **Avalanche** — the cache layer itself fails and 100% of traffic hits the database, which was sized for 5%. Fix: capacity-plan the database for a cache outage, or add a circuit breaker + load shedding on the database path so you degrade rather than collapse.
4. **Staleness after a write** — the classic invalidation race: reader misses, reads the DB, writer updates the DB and invalidates, reader then writes its *stale* value into the cache, which persists until TTL. Fixes: TTL as a backstop, delete-after-write (not update), or a short "delayed double delete". Accept that cache-aside is not linearizable and design around it.
5. **Unbounded growth** — no `maximumSize`, or caching per-user data with a large user base. `OutOfMemoryError`.
6. **Caching mutable objects** — putting a mutable entity in a local cache and letting callers mutate it corrupts every subsequent reader. Cache immutable DTOs, or copy on read.
7. **Caching per-user or security-sensitive data under a shared key** — a genuine data-leak vulnerability. Ensure the cache key includes every dimension that affects the result (tenant, user, locale, permissions).
8. **Serialisation incompatibility on deploy** — a changed DTO shape makes existing Redis entries undeserialisable, causing errors on every hit. Version the cache key prefix or the serialised payload.

**Measure it.** `recordStats()` and expose hit rate, miss rate, load time, and eviction count via Micrometer. A cache with a 40% hit rate is usually adding latency and complexity for nothing, and you will never know without the metric. Also measure the *database* load with and without the cache — the point is the load you avoided, not the hit rate itself.

### Probes

**Cache-aside vs write-through.** Covered, plus write-behind and read-through.

**TTL + jitter.** Covered, with the synchronised-expiry mechanism and refresh-ahead as an alternative.

**Single-flight on miss.** Covered, with Caffeine's guarantee and the distributed equivalent.

**Negative caching.** Covered, including penetration and Bloom filters.

**`@Cacheable` self-invocation trap.** Covered with the mechanism, four fixes, and the related private/final/key-generator traps.

**Caffeine vs Redis vs two-level.** Covered in a comparison table with clear selection criteria and the L1-invalidation problem.

**Business cost of staleness.** Covered as the framing question that determines every other decision.

---

## Q88. What do you instrument on day one, and what would you refuse to add?

### Answer

**Day one: the RED metrics on every endpoint.**

- **R**ate — requests per second
- **E**rrors — failed requests per second, split by cause (`4xx` vs `5xx` — client error and server error are different signals and must never be aggregated)
- **D**uration — latency **distribution**, not an average

In Spring Boot this is nearly free: `micrometer-registry-prometheus` on the classpath plus `management.endpoints.web.exposure.include=prometheus` gives you `http_server_requests_seconds` tagged by `uri`, `method`, `status`, and `outcome`, automatically.

**USE metrics for every resource** (Brendan Gregg's complement to RED): **U**tilisation, **S**aturation, **E**rrors — for thread pools, connection pools, queues, CPU, memory, and disk. **Saturation is the leading indicator** and the most commonly missing one. `hikaricp_connections_pending` rising tells you about an outage several minutes before the error rate does. Instrument: thread pool active/queued, connection pool active/idle/pending, Kafka consumer lag, HTTP client pool leases, and internal queue depths.

Also day one: **JVM metrics** (heap by generation, GC pause time and frequency, thread count, class loading) — Micrometer's binders provide these automatically and they cost nothing. **Dependency call metrics** — rate/errors/duration for every outbound call, tagged by dependency, so you can immediately answer "is it us or them?". And **a handful of business metrics** — orders placed, payments authorised, signups. These are what actually tell you the system is working; a service can be 100% healthy by RED while silently processing zero orders because an upstream stopped producing.

**Percentiles, not averages — non-negotiable**

An average latency hides everything that matters. With 99 requests at 10ms and 1 at 5s, the mean is 60ms and looks fine, while 1% of your users had a terrible experience. Averages are also mathematically insensitive to exactly the tail you care about.

Two implementation details that separate a real answer from a superficial one:

- **Percentiles do not aggregate.** You cannot average the p99 of ten pods to get the fleet p99 — that is arithmetically meaningless. You need either histograms with shared bucket boundaries aggregated server-side (`management.metrics.distribution.percentiles-histogram.http.server.requests=true`, which exports cumulative buckets that Prometheus' `histogram_quantile()` can combine correctly across instances), or a mergeable sketch (DDSketch/T-Digest, which Micrometer supports). Client-side pre-computed percentiles (`percentiles=0.99`) are per-instance only and must not be aggregated. This is a very common and silently wrong configuration.
- **Watch bucket cardinality.** A histogram with 40 buckets × 20 endpoints × 5 status codes is 4,000 series from one metric. Set explicit SLO-aligned boundaries (`management.metrics.distribution.slo.http.server.requests=100ms,300ms,1s`) rather than the default wide range.

Report p50, p95, p99, and p99.9. The tail is where the interesting failures live, and for a service called N times per page render, the *user-visible* latency is closer to your p99 than your p50.

**Tracing, and context propagation**

Distributed tracing (OpenTelemetry, or Micrometer Tracing on Spring Boot 3) answers "where did those 5 seconds go across seven services?" — which no metric can. Instrument with **W3C Trace Context** (`traceparent`), which is the interoperable standard.

The hard part is **propagation across execution boundaries**:
- **Across threads** — trace context lives in a `ThreadLocal` (or `Context` in OTel). Hand work to an `ExecutorService` and it is lost. Fix: wrap executors (`ContextExecutorService`, Micrometer's `ContextPropagatingTaskDecorator`, or `ContextSnapshot`). Spring Boot 3 wires this for its own executors, but any executor *you* create needs decorating.
- **Across async/reactive** — `CompletableFuture` chains and Reactor need explicit context propagation (Reactor's `contextWrite` + the `context-propagation` library).
- **Across Kafka/messaging** — the trace must ride in message **headers**, and the consumer must extract it and start a *linked* span. Without this, your trace ends at the producer and the consumer's work appears as an unrelated orphan trace. Spring Kafka's observability support does this; hand-rolled consumers usually do not.
- **Across virtual threads and `ScopedValue`** (Q14) — a current concern worth mentioning.

**Sample deliberately.** 100% tracing at high volume is expensive in CPU, network, and storage. Head-based sampling at 1–10% is typical; **tail-based sampling** (decide after the trace completes, keeping all errors and all slow traces) is much better but requires a collector that buffers spans. Always sample errors and slow requests at 100%.

**Structured logs with a trace ID**

Log as JSON, not as prose, so fields are queryable: `logstash-logback-encoder` or Boot 3.4+'s built-in structured logging (`logging.structured.format.console=ecs`). Every line must carry `traceId` and `spanId` — Micrometer Tracing puts them in the MDC automatically, so a log pattern of `%X{traceId:-}` links logs to traces. That link is the single highest-value thing in the whole observability stack: from a slow trace you jump straight to the exact log lines for that request across every service.

Include a consistent set of fields: service, version, environment, trace ID, user/tenant ID (hashed if sensitive), and the event-specific data as **fields rather than interpolated into the message**. `log.info("order placed", kv("orderId", id), kv("amount", amt))` is queryable; `log.info("Order " + id + " placed for " + amt)` is not.

**Cardinality explosion — what I would refuse to add**

This is the question's real target. In a dimensional metrics system, **every unique combination of label values creates a separate time series**, each with its own memory in the scrape target, its own index entry, and its own storage. Prometheus holds active series in memory; a cardinality explosion OOMs the Prometheus server and takes down monitoring for *everyone* — precisely when you need it.

I would refuse, on a metric label:
- **User ID, session ID, request ID, trace ID, order ID** — unbounded by definition. One label with a million values is a million series.
- **Raw URL paths with embedded identifiers** — `/orders/12345`. This must be templated to `/orders/{id}`. Spring does this correctly via `uri` tags derived from the route pattern, but hand-rolled `Timer` instrumentation frequently gets it wrong, and a 404 on an arbitrary path can inject unbounded values.
- **Email addresses, IP addresses, full user agents.**
- **Exception messages** — which often embed IDs or values. Use the exception *class* instead.
- **Timestamps or anything monotonically increasing.**

The arithmetic is brutal because it multiplies: 5 services × 20 endpoints × 8 status codes × 3 regions × 40 histogram buckets = 96,000 series from *one* metric, before anyone adds a bad label. Adding a `customerId` label with 50,000 values multiplies that by 50,000.

The rule I apply: **a metric label must have a small, bounded, and known set of values — ideally under ~100, and never derived from user input.** High-cardinality data belongs in **traces** (where each trace is a separate object, not a time series) and in **structured logs** (where the backend is designed for high-cardinality search). That is the correct division of labour between the three pillars, and being able to state it is the point of the probe.

I would also push back on: logging at `DEBUG` in production by default (see below), logging full request/response bodies (cost and PII), a metric per business entity, and "let's just add a dashboard for it" without an owner or an alert — an unwatched dashboard is not observability.

**SLI / SLO / error budget**

- **SLI** — a measured indicator of user-visible quality: *the proportion of requests served successfully in under 300ms*. It must be measured from as close to the user as possible (at the load balancer, not inside the app).
- **SLO** — the target for that SLI over a window: *99.5% over 30 days*.
- **Error budget** — the permitted failure: `100% − 99.5% = 0.5%`, which over 30 days is about 3.6 hours. This is the crucial reframing: **the budget is something you are entitled to spend.** If it is intact, ship faster and take more risk; if it is exhausted, freeze feature work and spend the effort on reliability. It converts an unwinnable "how reliable is reliable enough?" argument into a data-driven policy.
- **Alert on burn rate, not on thresholds.** A multi-window, multi-burn-rate alert (fast burn: 14.4× over 1h → page; slow burn: 3× over 6h → ticket) fires on things that genuinely threaten the SLO and stays silent on brief blips. This is the single biggest improvement most teams can make to their alerting, because it eliminates the "CPU > 80%" class of alert that wakes people for something no user noticed.
- **Every page must be actionable.** If the responder can do nothing, it should be a ticket. Alert fatigue is a reliability risk in itself.

**Log level and sampling cost**

Logging is not free and its cost is regularly underestimated:
- **Volume and money.** Ingest-priced log platforms make `DEBUG` in production genuinely expensive — often more than the compute running the service.
- **CPU and I/O.** Synchronous logging to stdout can block the request thread under load. Use an `AsyncAppender` with a **bounded** queue, and decide deliberately whether it discards or blocks when full (blocking turns your logger into a source of latency; discarding loses data — for most services, discard `INFO` and below, never discard `ERROR`).
- **String construction.** `log.debug("state: " + expensiveToString())` builds the string **even when DEBUG is disabled**, because arguments are evaluated before the call. Use parameterised logging (`log.debug("state: {}", obj)`) or a supplier (`log.atDebug().addArgument(() -> ...)`), and guard genuinely expensive calls with `isDebugEnabled()`.
- **Sample high-volume logs** — a per-message-type rate limiter (Logback's `DuplicateMessageFilter`, or an application-level sampler) prevents one hot loop from producing 10 GB/hour.
- **Dynamic log levels** — Actuator's `/actuator/loggers` lets you raise a specific logger to `DEBUG` at runtime for a few minutes during an incident and then lower it. This is the correct pattern: `INFO` by default, temporary targeted `DEBUG` on demand.

**What I would actually do on day one, concretely:** Micrometer + Prometheus with the automatic HTTP and JVM binders; OpenTelemetry tracing at 5% head sampling with 100% on errors; JSON logs at `INFO` with trace IDs in the MDC; four or five business counters; SLOs defined for the two or three user-facing endpoints that matter, with burn-rate alerts; and a dashboard showing RED per endpoint plus saturation for every pool. That is perhaps a day of work and covers the large majority of incidents.

### Probes

**RED/USE.** Covered, with the point that saturation is the leading indicator.

**Percentiles not averages.** Covered, including the non-aggregability of pre-computed percentiles and histogram bucket cardinality.

**Trace context propagation across threads, async, and Kafka.** Covered, with the specific mechanisms for each and the sampling strategy.

**Structured logs with trace ID.** Covered, including fields-not-interpolation.

**Label cardinality explosion.** Covered as the primary "what I'd refuse" answer, with the multiplication arithmetic and the metrics/traces/logs division of labour.

**SLI/SLO/error budget.** Covered, including burn-rate alerting and actionability.

**Log level and sampling cost.** Covered — ingest cost, async appenders and their queue policy, string construction, sampling, and dynamic level changes.

---

## Q89. Walk through OAuth2 authorization code flow with PKCE. Why each step exists.

### Answer

**The actors:** the **resource owner** (the user), the **client** (your SPA or mobile app), the **authorization server** (Keycloak, Auth0, Okta, Entra ID), and the **resource server** (your API).

**The flow**

**1. The client generates a PKCE pair.**
```
code_verifier  = base64url(random 32–96 bytes)          # kept secret in the client
code_challenge = base64url(SHA256(code_verifier))        # sent in the open
code_challenge_method = S256
```

**2. Front-channel: redirect the user's browser to the authorization server.**
```
GET https://auth.example.com/authorize
  ?response_type=code
  &client_id=spa-client
  &redirect_uri=https://app.example.com/callback
  &scope=openid profile orders:read
  &state=<random, bound to the user's session>
  &nonce=<random>
  &code_challenge=E9Melhoa2Owv...
  &code_challenge_method=S256
```

**3. The user authenticates directly with the authorization server** — username/password, MFA, passkey, SSO. This is the central security property: **the client never sees the user's credentials.** The user types their password only into the authorization server's own domain, which is what makes phishing detectable and lets the AS own MFA and session policy.

**4. Front-channel: redirect back with an authorization code.**
```
302 https://app.example.com/callback?code=SplxlOBeZQ&state=<echoed>
```

**5. Back-channel: exchange the code for tokens** — a direct server-to-server (or app-to-AS) HTTPS POST, not through the browser.
```
POST /token
grant_type=authorization_code
&code=SplxlOBeZQ
&redirect_uri=https://app.example.com/callback
&client_id=spa-client
&code_verifier=<the original random value>
```
The AS computes `SHA256(code_verifier)` and compares it to the stored `code_challenge`. Match → issue tokens; mismatch → reject.

**6. Response:**
```json
{ "access_token": "eyJ...", "token_type": "Bearer", "expires_in": 900,
  "refresh_token": "def502...", "id_token": "eyJ...", "scope": "openid profile orders:read" }
```

**Why each step exists**

**Why a code at all, rather than returning the token directly?** Because step 4 travels through the **front channel** — the user's browser — where the value lands in the URL, and therefore in browser history, in `Referer` headers, in proxy logs, in server access logs, and in any browser extension. An authorization code is designed to survive that exposure: it is **single-use**, **short-lived** (seconds to a minute), **bound to the client_id and redirect_uri**, and **useless without the token exchange**. An access token in the same position would be immediately usable by anyone who read the log. This is exactly why the **implicit flow** (`response_type=token`) is deprecated and removed in OAuth 2.1.

**Front channel vs back channel** is the organising concept:
- *Front channel* — via the browser (redirects). Visible to the user, tamperable, logged, no client authentication possible. Carries only the code, `state`, and public parameters.
- *Back channel* — direct TLS from client to AS. Confidential, integrity-protected, and the only place a client secret may be used. Carries the tokens.

**Why PKCE.** Consider a public client — a SPA or a native mobile app — which **cannot hold a secret** (the code ships to the user's device and can be decompiled or read in DevTools). Without a client secret, anything that intercepts the authorization code can redeem it. On mobile this was a real attack: a malicious app registers the same custom URI scheme (`myapp://callback`) and the OS hands *it* the redirect. On the web, a code can leak through an open redirect, a `Referer` header, or a compromised browser extension.

PKCE (RFC 7636, "pixie") binds the code to the specific client instance that started the flow. The attacker sees `code_challenge` (public, in the front channel) and the `code` (intercepted), but redeeming the code requires the `code_verifier`, which never left the legitimate client. SHA-256 is one-way, so the challenge does not reveal the verifier.

Two things to state precisely: **always use `S256`, never `plain`** (`plain` sends the verifier in the authorization request, defeating the purpose — it exists only for constrained devices that cannot compute SHA-256). And PKCE is now recommended for **confidential clients too** in OAuth 2.1 — it defends against code injection regardless of client type, so "PKCE is only for public clients" is outdated advice.

**Why `state`?** CSRF protection for the flow itself. Without it, an attacker can initiate their own authorization flow, capture their own code, and then trick a victim's browser into hitting the victim's callback with the *attacker's* code — silently linking the victim's session to the attacker's account. `state` must be a random value bound to the user's session and verified on return. PKCE does not replace it; `state` protects the *session binding*, PKCE protects the *code*.

**Why `nonce`?** An OIDC-specific value echoed inside the ID token, protecting against ID token replay. Verify it matches what you sent.

**The three tokens**

| Token | Audience | Purpose | Lifetime |
|---|---|---|---|
| **Access token** | The resource server (your API) | Authorization — proves the bearer may perform an action within a scope | Short: 5–15 min |
| **Refresh token** | The authorization server only | Obtain new access tokens without re-authenticating | Long: hours to months |
| **ID token** | The client only | Authentication — asserts *who* the user is (an OIDC concept, not OAuth2) | Short |

The most common mistake in the industry: **sending the ID token to your API as a bearer token.** The ID token's `aud` is the *client*, not the API; it is proof of authentication for the client's own use. Your API must validate the **access token** and check that `aud` matches itself. Symmetrically, the client should never parse the access token — it is opaque to the client by design (even when it happens to be a JWT).

**Lifetimes and rotation**

- Short access tokens bound the damage from a leak, because there is **no practical revocation** for a self-contained JWT (Q90). 5–15 minutes is standard.
- **Refresh token rotation** (mandatory for public clients in OAuth 2.1): each use returns a *new* refresh token and invalidates the old one. If an old token is presented again, the AS detects **reuse** — meaning either the token was stolen or a race occurred — and **revokes the entire token family**, forcing re-authentication. This turns a silent, indefinite compromise into a detected event. Implement with a *grace window* for legitimate races (a mobile app firing two refreshes concurrently).
- Refresh tokens must be `sender-constrained` where possible: **DPoP** (RFC 9449) or **mTLS-bound tokens** (RFC 8705) bind the token to a key the client holds, so a stolen token is useless to anyone else. This is the current direction of travel and worth naming.

**Where to store tokens in a browser — and why BFF wins**

- **`localStorage` / `sessionStorage`** — readable by **any JavaScript on the page**. One XSS anywhere (your code, a dependency, a tag manager, a compromised CDN script) exfiltrates the token, and the attacker has the user's full API access for the token's lifetime with no trace. Also readable by browser extensions. **This is the wrong answer**, though it remains extremely common.
- **In-memory only** — better: an XSS must run *while the page is open* rather than reading a durable store. But it is lost on refresh, requiring a silent re-authentication that third-party cookie restrictions increasingly break.
- **`httpOnly`, `Secure`, `SameSite` cookies** — invisible to JavaScript entirely, so XSS cannot read them. The trade-off is that cookies are sent automatically, reintroducing **CSRF** — mitigated by `SameSite=Lax`/`Strict` and, for cross-site cases, a CSRF token.

**The BFF (Backend-for-Frontend) pattern** is the current recommended architecture for browser-based apps (and what the IETF's browser-based-apps BCP recommends):

```
Browser ──httpOnly session cookie──▶ BFF (your server) ──Bearer access token──▶ API
```

The BFF is a confidential client. It performs the code exchange server-side, holds access and refresh tokens **server-side**, and gives the browser only an opaque, `httpOnly`, `Secure`, `SameSite` session cookie. Benefits: **no OAuth token ever reaches JavaScript**, so XSS cannot steal one; the BFF can hold a real client secret; refresh happens invisibly; and revocation is trivial (delete the server-side session). Costs: a stateful component, a session store, and CSRF protection on the BFF. Spring's `spring-cloud-gateway` with `TokenRelay`, or Spring Security's OAuth2 client with a server-side session, implements this directly.

**Practical Spring notes.** The resource server side is `spring-boot-starter-oauth2-resource-server` with `spring.security.oauth2.resourceserver.jwt.issuer-uri`, which auto-discovers the JWKS endpoint and configures issuer/expiry validation. The client side is `spring-boot-starter-oauth2-client`, which handles PKCE automatically for public clients (and, since Spring Security 6.3, can be enabled for confidential clients too).

### Probes

**Front channel vs back channel.** Covered as the organising concept, with what each may carry and why.

**PKCE preventing code interception.** Covered — the threat model for public clients, the mechanism, `S256` vs `plain`, and the OAuth 2.1 recommendation to use it universally.

**Access/refresh/ID tokens.** Covered in a table, including the very common ID-token-to-API mistake.

**BFF and httpOnly cookie vs `localStorage`/XSS.** Covered in detail, with the intermediate options and a recommendation.

**Lifetimes and rotation.** Covered, including reuse detection, family revocation, grace windows, and sender-constrained tokens (DPoP/mTLS).

---

## Q90. A JWT arrives at your API. Everything you validate, and everything that can go wrong.

### Answer

**Structure.** `header.payload.signature`, each base64url-encoded. The header and payload are **encoded, not encrypted** — anyone can read them. Never put anything confidential in a JWT unless you are using JWE.

**The validation sequence, in order**

**1. Parse and check `alg` against an allowlist.** Do not read the algorithm from the token and use it — decide the acceptable algorithms yourself (`RS256`, `ES256`) and reject anything else *before* verifying.

**2. Resolve the key via `kid` from the JWKS endpoint.** The issuer publishes public keys at `/.well-known/jwks.json` (discoverable from `/.well-known/openid-configuration`). Match the token's `kid` header to a key in the set.

**3. Verify the signature** with that key.

**4. Validate `iss`** — exact string match against your expected issuer. A valid signature from the *wrong* issuer is worthless.

**5. Validate `aud`** — the token must be intended for *your* API. This is the check most often omitted, and its absence is a serious vulnerability (below).

**6. Validate `exp`**, and `nbf`/`iat` if present, allowing for clock skew.

**7. Check `scope`/`roles`/`permissions`** for the specific operation. Authentication is not authorization.

**8. Optionally check `jti`** against a revocation list, and `sub` against a "user disabled" store, if your threat model requires it.

Only then map to a principal and proceed.

**JWKS: `kid`, caching, and rotation**

The `kid` (key ID) header tells you *which* published key signed the token, which is what makes **key rotation** possible: the issuer publishes both the old and new keys during an overlap window, signs new tokens with the new `kid`, and retires the old key once all tokens signed with it have expired.

Caching is required — fetching JWKS per request would add a network round trip to every call and let an issuer outage take down your API. But cache carefully:
- Cache the key set with a TTL (5–15 minutes is typical), respecting `Cache-Control` if provided.
- On an **unknown `kid`**, refetch once (the issuer may have just rotated) — but **rate-limit that refetch**, or an attacker can force unbounded outbound requests by sending tokens with random `kid` values. Spring Security's `NimbusJwtDecoder` and the underlying `RemoteJWKSet` handle this with a rate-limited caching JWK source.
- Have a fallback for JWKS unavailability: serve from a stale cache rather than rejecting all traffic (the keys are still valid; only the refresh failed).

**`alg: none` and algorithm confusion — the two classic attacks**

**`alg: none`.** The JWT spec defines an "unsecured JWS" with `{"alg":"none"}` and an empty signature. A naive library that honours the header's algorithm accepts such a token as valid — so an attacker forges any payload they like, sets `alg` to `none`, and becomes an administrator. Multiple major libraries shipped this vulnerability (CVE-2015-9235 and relatives). **Defence: an explicit algorithm allowlist that never includes `none`.**

**Algorithm confusion (RS256 → HS256).** The subtler and more dangerous attack. `RS256` is asymmetric (verify with the public key); `HS256` is symmetric (verify with a shared secret). If your verification code is "look up the key and verify with the algorithm the token specifies", an attacker can:
1. Take your **public** RSA key (it is published at the JWKS endpoint — it is meant to be public).
2. Forge a token with `alg: HS256`, signing it with the public key's bytes as the HMAC secret.
3. Your code sees `HS256`, fetches "the key" (the public key), and uses it as an HMAC secret. The signature verifies. **The attacker forges arbitrary tokens using entirely public information.**

**Defence: pin the algorithm at the verifier, not from the token.** Never let the token choose. In Spring Security:
```java
NimbusJwtDecoder decoder = NimbusJwtDecoder
        .withJwkSetUri(jwkSetUri)
        .jwsAlgorithm(SignatureAlgorithm.RS256)   // pinned
        .build();
decoder.setJwtValidator(JwtValidators.createDefaultWithIssuer(issuer));  // iss + exp + nbf
```
And **always use a maintained library.** Hand-rolled JWT verification is a reliable way to reintroduce both attacks.

**Why `aud` matters.** Suppose the same authorization server issues tokens for a partner's low-security API and for your payments API. Without an `aud` check, a token legitimately issued for the partner API — signed by the correct issuer, unexpired, perfectly valid — is accepted by yours. Any partner can now call your payments API with a token their own users freely obtain. This is a **confused deputy** and it is a full authorization bypass. Spring Security does *not* validate `aud` by default; you must add an `OAuth2TokenValidator`:
```java
decoder.setJwtValidator(new DelegatingOAuth2TokenValidator<>(
        JwtValidators.createDefaultWithIssuer(issuer),
        new JwtClaimValidator<List<String>>("aud", a -> a != null && a.contains("payments-api"))));
```

**No revocation — the fundamental trade-off**

A JWT is **self-contained**: your API validates it using only the signature and its claims, with no call to the issuer. That is the entire performance benefit — stateless verification, horizontally scalable, no per-request round trip. It is also the entire limitation: **there is no way to un-issue a valid token.** A user logs out, is fired, has their permissions reduced, or their token is stolen — and the token keeps working until `exp`.

Consequences and mitigations:
- **Keep access tokens short** (5–15 min). The exposure window equals the token lifetime. This is the primary mitigation and why lifetimes are the number you tune.
- **Refresh tokens carry the revocation**: they *are* checked against server state at the authorization server, so revoking a refresh token stops re-issuance within one access-token lifetime.
- **A denylist by `jti`** for emergency revocation. This reintroduces state and a lookup, but only for the small set of explicitly revoked tokens, and entries can be evicted at `exp`. Reasonable as a break-glass mechanism.
- **Token introspection** (RFC 7662) — the API asks the AS whether the token is still active on every request. Fully revocable, but you have given up the statelessness that motivated JWTs. Use for high-value operations only, or use opaque tokens with introspection when revocation matters more than latency.
- Be honest about this in an interview: "JWTs trade revocability for statelessness. If your requirement is immediate revocation, JWTs are the wrong tool and opaque tokens with introspection (or a server-side session) are correct."

**No PII in the payload**

The payload is base64url — **anyone holding the token can read it**, including the browser, any JavaScript, browser extensions, proxies, and anything that logs the `Authorization` header. So:
- No email addresses, names, phone numbers, addresses, dates of birth, national IDs, or health data.
- No internal secrets, database IDs that leak schema, or internal hostnames.
- Under GDPR, an opaque `sub` identifier plus roles/scopes is the right content; anything personal is fetched from your own store using `sub`.
- Also: **tokens leak into logs.** Never log the `Authorization` header, and configure your logging and APM to redact it. A token in a log aggregator is a credential in a log aggregator.
- Keep them small — a JWT with 40 roles can exceed header size limits (nginx's default 8 KB) and adds bytes to every request.
- If you genuinely must carry confidential data, use **JWE** (encrypted), not JWS.

**Clock skew**

`exp` and `nbf` are absolute timestamps compared against the verifier's clock. If the issuer's clock is 30 seconds ahead, freshly issued tokens fail `nbf` on your server; if behind, tokens survive slightly past their nominal expiry. Allow a small tolerance — **30 to 60 seconds**, no more:
```java
var withClockSkew = new DelegatingOAuth2TokenValidator<Jwt>(
        new JwtTimestampValidator(Duration.ofSeconds(60)),
        new JwtIssuerValidator(issuer));
```
Spring Security's default skew is 60 seconds. Do not "fix" intermittent validation failures by setting a five-minute skew — that extends the life of every revoked token by five minutes. Run NTP on all hosts; in Kubernetes the node clock is shared, so skew between pods is usually not the issue, but skew between your cluster and an external issuer can be.

**Other checks worth mentioning:** enforce HTTPS (a bearer token over HTTP is a credential in cleartext); reject tokens above a size limit; validate that required claims are present rather than defaulting them; and consider `typ: at+jwt` (RFC 9068) to distinguish access tokens from ID tokens structurally, closing the ID-token-as-access-token confusion from Q89.

### Probes

**JWKS + `kid` + caching/rotation.** Covered, including rate-limiting refetch on unknown `kid` and stale-cache fallback.

**`alg: none` and algorithm confusion.** Both covered in detail, with the RS256→HS256 mechanism and the pin-the-algorithm defence.

**`iss` / `aud` / `exp` / `nbf`.** Covered in the validation sequence, with `aud` singled out as the confused-deputy vulnerability and the Spring code to add it.

**No revocation → short lifetimes.** Covered, with four mitigations and the honest statement of when JWTs are simply the wrong tool.

**No PII.** Covered, including logging leakage and header-size effects.

**Clock skew.** Covered, with the recommended tolerance and why a large one is dangerous.

---

## Q91. Explain the Spring Security filter chain. Where does authentication actually happen?

### Answer

**The architecture.** Spring Security is a chain of **servlet filters** inserted into the container's filter chain by a single entry point, `springSecurityFilterChain` — a `DelegatingFilterProxy` registered with the servlet container. It delegates to `FilterChainProxy`, which holds one or more `SecurityFilterChain` beans. For each request, `FilterChainProxy` finds the **first** chain whose `RequestMatcher` matches and runs that chain's filters. Everything happens *before* the request reaches `DispatcherServlet`, which is why security failures produce responses that never touch your controllers.

**The ordering, and what each filter does**

A representative chain, in order:

1. `DisableEncodeUrlFilter`
2. `WebAsyncManagerIntegrationFilter` — propagates the `SecurityContext` to async request threads.
3. `SecurityContextHolderFilter` — loads the `SecurityContext` from the `SecurityContextRepository` (the HTTP session for stateful apps) at the start, and **clears the `ThreadLocal` in a `finally` block** at the end. (In Spring Security 6 this replaced `SecurityContextPersistenceFilter`, and saving is now explicit rather than automatic — a notable migration gotcha.)
4. `HeaderWriterFilter` — security response headers (HSTS, `X-Content-Type-Options`, etc.).
5. **`CorsFilter`** — before authentication, deliberately (see below).
6. **`CsrfFilter`**
7. `LogoutFilter`
8. **Authentication filters** — `UsernamePasswordAuthenticationFilter`, `BearerTokenAuthenticationFilter`, `OAuth2LoginAuthenticationFilter`, `BasicAuthenticationFilter`, etc.
9. `RequestCacheAwareFilter` — restores the originally requested URL after a login redirect.
10. `SecurityContextHolderAwareRequestFilter`
11. `AnonymousAuthenticationFilter` — if nothing authenticated, install an `AnonymousAuthenticationToken` so downstream code never sees `null`.
12. `SessionManagementFilter`
13. **`ExceptionTranslationFilter`** — catches `AuthenticationException` → `AuthenticationEntryPoint` (`401`, or a redirect to login) and `AccessDeniedException` → `AccessDeniedHandler` (`403`). It must sit *above* the authorization filter to catch what it throws.
14. **`AuthorizationFilter`** — the last filter; evaluates the URL-based rules. (Spring Security 6 replaced `FilterSecurityInterceptor` with this.)

Order matters and is not arbitrary: context loading must precede anything reading the principal; CORS must precede authentication; exception translation must wrap authorization; and authorization must be last so it sees a fully populated context.

**Where authentication actually happens**

Inside an **authentication filter**, and the delegation chain is the answer the question wants:

```
AuthenticationFilter
   → builds an unauthenticated Authentication token from the request
   → AuthenticationManager (interface)
        → ProviderManager (the standard implementation)
             → iterates AuthenticationProviders, calling supports(tokenClass)
                  → DaoAuthenticationProvider
                       → UserDetailsService.loadUserByUsername(...)
                       → PasswordEncoder.matches(raw, encoded)
                       → returns a fully populated, authenticated Authentication
   → SecurityContextHolder.getContext().setAuthentication(result)
```

Key points:
- **`AuthenticationManager`** is the interface; **`ProviderManager`** is the implementation that holds a list of `AuthenticationProvider`s and tries each one that `supports()` the token type. This is how multiple mechanisms coexist (form login *and* JWT *and* LDAP) in one application.
- A provider returns an authenticated token, throws an `AuthenticationException`, or returns `null` meaning "not my concern — try the next provider". `ProviderManager` can also delegate to a **parent** manager if none of its providers handles the token.
- For a **resource server**, `BearerTokenAuthenticationFilter` extracts the token and `JwtAuthenticationProvider` uses a `JwtDecoder` (Q90) to validate it and a `JwtAuthenticationConverter` to map claims to `GrantedAuthority`s. There is no `UserDetailsService` in that path — a frequent point of confusion.
- The filter then **stores the result** in `SecurityContextHolder`, and (for stateful apps in Spring Security 6) must explicitly save it via the `SecurityContextRepository`.

**Configuring the chain (Spring Security 6, component-based):**
```java
@Bean
@Order(1)
SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
    return http
        .securityMatcher("/api/**")                       // this chain handles only /api/**
        .authorizeHttpRequests(a -> a
            .requestMatchers("/api/public/**").permitAll()
            .requestMatchers("/api/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated())
        .oauth2ResourceServer(o -> o.jwt(Customizer.withDefaults()))
        .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
        .csrf(CsrfConfigurer::disable)                    // justified below
        .cors(Customizer.withDefaults())
        .build();
}
```
Multiple chains are matched **in `@Order`**, first match wins — so a broad chain placed before a narrow one silently swallows it. That is the most common multi-chain bug.

**`SecurityContextHolder`, `ThreadLocal`, and async propagation**

`SecurityContextHolder` stores the `SecurityContext` in a `ThreadLocal` by default (`MODE_THREADLOCAL`). Consequences:

- It is **cleared in a `finally`** by `SecurityContextHolderFilter`. Without that, a pooled container thread would carry one user's principal into the next user's request — a catastrophic authorization bug, and precisely the `ThreadLocal` leak pattern of Q21.
- **It does not propagate to threads you create.** `@Async`, a raw `ExecutorService`, a `CompletableFuture.supplyAsync`, a Kafka listener thread, or a `parallelStream()` will all see an empty context, so `getAuthentication()` returns `null` (or anonymous) and method security denies access. Fixes:
  - `MODE_INHERITABLETHREADLOCAL` — propagates to threads *created* by the current thread. It does **not** work for pooled executors, because the pool's threads were created before the request. A common half-fix that appears to work in testing and fails in production.
  - **`DelegatingSecurityContextExecutor` / `DelegatingSecurityContextAsyncTaskExecutor`** — wrap the executor so each submitted task copies and installs the context. This is the correct answer for thread pools, and Spring Boot applies it to `@Async` when a `SecurityContextHolderStrategy`-aware task executor is configured.
  - For **reactive** (WebFlux), there is no `ThreadLocal` at all — the context lives in the Reactor `Context`, accessed via `ReactiveSecurityContextHolder`, and propagates through the reactive chain automatically.
  - For **virtual threads** (Q14), the `ThreadLocal` still works but does not survive a hand-off; `ScopedValue` is the direction of travel.

**Method security vs URL security**

- **URL security** (`authorizeHttpRequests`) is coarse and pattern-based, evaluated in the filter chain before the request is routed. It is easy to get subtly wrong: pattern/servlet-mapping mismatches, trailing slashes, case sensitivity, path traversal and encoding tricks, and — most commonly — a rule ordering error, since the **first matching rule wins** and putting `anyRequest().authenticated()` before a specific `permitAll()` makes the latter unreachable.
- **Method security** (`@EnableMethodSecurity`, then `@PreAuthorize`, `@PostAuthorize`, `@PreFilter`, `@PostFilter`) is implemented by **AOP proxies**, not filters. It sits next to the business logic, can reference method arguments and the returned object (`@PreAuthorize("#order.ownerId == authentication.name")`, `@PostAuthorize("returnObject.owner == authentication.name")`), and is therefore the right place for **ownership and domain-level rules** that a URL pattern cannot express.

Because it is proxy-based, it inherits every proxy caveat from Q25/Q26: **self-invocation bypasses it**, and it does not apply to `private`/`final` methods. Note also that `@PostAuthorize` executes the method *before* denying, so it must not be used on anything with side effects.

Use both: URL rules for broad coarse gating, method security for fine-grained and ownership checks. Defence in depth — and put method security on the **service** layer, not only the controller, so a second inbound adapter (a Kafka listener, a scheduled job) cannot bypass it.

**CSRF for stateless APIs**

CSRF exploits the browser's habit of attaching **ambient credentials** — cookies, HTTP Basic, client certificates — to cross-site requests automatically. An attacker's page issues a request to your domain and the browser helpfully includes the victim's session cookie.

If your API is genuinely stateless and authenticates with an `Authorization: Bearer` header, **no ambient credential exists** — the attacker's page cannot set that header on a cross-origin request (and a preflight would block it anyway). So **disabling CSRF for a bearer-token API is correct**, not a shortcut.

But disable it only when that condition truly holds. It does **not** hold if: you use cookie-based sessions anywhere (including a `JSESSIONID` for a server-rendered admin page in the same app); you use a **BFF** with an `httpOnly` session cookie (Q89) — the BFF absolutely needs CSRF protection; or you accept HTTP Basic from a browser. In those cases keep CSRF enabled, using `CookieCsrfTokenRepository.withHttpOnlyFalse()` for SPA double-submit, and pair it with `SameSite=Lax` or `Strict` cookies as defence in depth. Spring Security 6 changed CSRF token resolution (the `BREACH`-protected `XorCsrfTokenRequestAttributeHandler` and deferred token loading), which broke many SPA setups on upgrade — a good detail to know.

**CORS preflight must precede authentication**

A cross-origin request with a non-simple method or header triggers a preflight:
```
OPTIONS /api/orders
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: authorization, content-type
```
**The browser deliberately sends no credentials on a preflight** — no `Authorization` header, no cookies. So if the security chain authenticates before handling CORS, the `OPTIONS` request is rejected with `401`, the browser never receives the `Access-Control-Allow-*` headers, and it blocks the real request. The symptom is the classic and misleading "CORS error" in the browser console for an endpoint that works perfectly in curl.

That is why `CorsFilter` is placed **before** the authentication filters, and why `.cors(Customizer.withDefaults())` (which wires a `CorsConfigurationSource` bean into that filter) is the correct configuration rather than adding a `@CrossOrigin` annotation on the controller — annotations are handled by Spring MVC, which the request never reaches. Also ensure `OPTIONS` is permitted in your authorization rules if you have not relied on the filter ordering.

### Probes

**`SecurityFilterChain` ordering.** Covered — the representative chain, why the order is what it is, and the multi-chain `@Order` first-match-wins trap.

**`AuthenticationManager` / `AuthenticationProvider`.** Covered as the delegation sequence, including `ProviderManager`, the `supports`/`null` protocol, parent managers, and the resource-server variant.

**`SecurityContextHolder` `ThreadLocal` and async propagation.** Covered, including why clearing matters, why `INHERITABLETHREADLOCAL` fails for pools, and the correct `DelegatingSecurityContext*` wrappers plus the reactive and virtual-thread cases.

**Method vs URL security.** Covered, with what each is good at, the proxy caveats, and the recommendation to apply method security at the service layer.

**CSRF for stateless APIs.** Covered — why disabling is correct for bearer tokens, and the specific cases where it is not.

**CORS preflight before auth.** Covered, with the mechanism, the misleading symptom, and why `@CrossOrigin` is insufficient.

---

# 13. Build, CI/CD, Testing

---

## Q92. Two libraries need conflicting versions of a transitive dependency.

### Answer

**First: see the actual graph.** Never guess.

```bash
mvn dependency:tree -Dverbose -Dincludes=com.fasterxml.jackson.core:jackson-databind
mvn dependency:analyze          # used-undeclared and declared-unused
./gradlew dependencies --configuration runtimeClasspath
./gradlew dependencyInsight --dependency jackson-databind --configuration runtimeClasspath
```

`-Dverbose` is essential in Maven — it shows the *omitted* nodes and the reason (`omitted for conflict with 2.15.2`), which is exactly the information you need. Gradle's `dependencyInsight` is the better tool of the two: it shows every requester, the requested version, and the reason the winner won.

**Understand the resolution strategy — they differ, and this surprises people**

- **Maven: "nearest wins."** The version at the shallowest depth in the tree wins; ties are broken by **declaration order** in the POM. Note this is *not* "highest version wins" — Maven can and does select an **older** version if it appears closer to the root. A transitive dependency at depth 1 requesting Jackson 2.12 beats one at depth 3 requesting 2.17.
- **Gradle: "highest wins."** Gradle selects the highest requested version by default, which is usually the safer outcome given semantic-versioning conventions. Gradle also supports **rich versions** (`strictly`, `require`, `prefer`, `reject`) and will **fail the build** on an unresolvable conflict between two `strictly` constraints rather than silently picking one.

Being able to state this difference precisely is a good senior signal, because it explains why the same dependency set behaves differently in two builds.

**Fixing it, in order of preference**

**1. `dependencyManagement` / a BOM — the correct default.**

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>com.fasterxml.jackson</groupId>
      <artifactId>jackson-bom</artifactId>
      <version>2.17.2</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

`dependencyManagement` **overrides transitive resolution entirely** — it wins over nearest-wins regardless of depth. Importing a **BOM** is better than pinning one artifact, because it aligns the whole family (`jackson-databind`, `jackson-core`, `jackson-annotations`, and every datatype module) to versions tested together. Mixing Jackson module versions is a classic source of `NoSuchMethodError`.

Spring Boot's `spring-boot-dependencies` BOM is the large-scale version of this: it curates hundreds of compatible versions, and the correct way to override one is the property (`<jackson.version>2.17.2</jackson.version>`), not a separate `<dependency>` declaration — otherwise you desynchronise the family.

Gradle equivalent: `implementation(platform("com.fasterxml.jackson:jackson-bom:2.17.2"))`, or a `constraints { }` block.

**2. Exclusions — targeted, and a last resort.**

```xml
<dependency>
  <groupId>com.acme</groupId><artifactId>legacy-lib</artifactId><version>3.1</version>
  <exclusions>
    <exclusion><groupId>com.fasterxml.jackson.core</groupId><artifactId>jackson-databind</artifactId></exclusion>
  </exclusions>
</dependency>
```

Use when a library drags in something you deliberately supply yourself, or an unwanted logging binding. Risks: the exclusion is invisible at the point of failure, it must be repeated on every declaring dependency, and if nothing else supplies the artifact you get `NoClassDefFoundError` at runtime. Prefer `dependencyManagement`; reach for exclusions when you need something *gone*, not merely *pinned*.

**3. Shading / relocation — when the versions are genuinely incompatible.**

If library A requires Guava 19 and library B requires Guava 31, and the API changes are breaking, no single version satisfies both. The Maven Shade Plugin (or Gradle Shadow) can **relocate** one copy into a private package:

```xml
<relocations>
  <relocation>
    <pattern>com.google.common</pattern>
    <shadedPattern>com.acme.shaded.com.google.common</shadedPattern>
  </relocation>
</relocations>
```

Now both versions coexist under different fully-qualified names. Costs, and they are real: a larger artifact; broken stack traces and IDE navigation; anything relying on reflection, `Class.forName`, service loaders, or resource paths can break because the package names changed; licence and attribution obligations; and debugging becomes materially harder. **Shading is for library authors publishing artifacts, and for genuine last-resort application cases** — Elasticsearch's and Spark's clients are well-known examples. Prefer upgrading, or replacing one library.

**4. Upgrade, or drop one library.** Often the honest answer. If `legacy-lib` pins a Jackson version with known CVEs and is unmaintained, the conflict is a symptom.

**`NoSuchMethodError` at runtime — the reason this matters**

The failure mode that makes this question worth asking. Java linkage is **late**: `javac` compiles against the API it sees, recording a symbolic reference (class, method name, descriptor). The JVM resolves that reference on first use, against whatever class is actually on the classpath.

So if you compile against Guava 31's `Iterables.getOnlyElement(Iterable, Object)` but Guava 19 is on the runtime classpath, the code **compiles cleanly**, deploys cleanly, and throws `NoSuchMethodError` the first time that line executes — possibly weeks later, in a rarely-hit branch, in production. Related symptoms: `NoClassDefFoundError`, `AbstractMethodError` (an interface gained a method the older implementation doesn't have), `IncompatibleClassChangeError`, and `LinkageError`.

Two diagnostic techniques:
- `java -verbose:class | grep Guava` (or `-Xlog:class+load=info` on modern JDKs) to see **which jar** each class was actually loaded from.
- `ClassName.class.getProtectionDomain().getCodeSource().getLocation()` printed at startup for a suspect class — the definitive answer to "where did this actually come from?".

This is also why **the classpath order matters** when duplicate classes exist across jars: the first one wins, and "first" is a fragile, build-tool-dependent property.

**Enforcement in CI — making it not happen again**

**Maven Enforcer Plugin:**
```xml
<plugin>
  <artifactId>maven-enforcer-plugin</artifactId>
  <executions><execution>
    <goals><goal>enforce</goal></goals>
    <configuration><rules>
      <dependencyConvergence/>       <!-- fail if the tree requests conflicting versions -->
      <banDuplicateClasses/>          <!-- extra-enforcer-rules -->
      <requireUpperBoundDeps/>        <!-- fail if the resolved version is LOWER than requested -->
    </rules></configuration>
  </execution></executions>
</plugin>
```

`requireUpperBoundDeps` is the single most valuable rule for Maven, because it catches exactly the nearest-wins-selects-an-older-version case that silently causes `NoSuchMethodError`. `dependencyConvergence` is stricter (any disagreement fails) and is excellent for a new project but often produces a long backlog on an existing one — introduce it with a documented exception list.

Also worth having: `mvn dependency:analyze` failing on **used-undeclared** dependencies (you compile against something you get only transitively, so an upstream change can remove it without warning); Gradle's `failOnVersionConflict()` or explicit `constraints`; **Renovate/Dependabot** to keep versions current so conflicts surface as small, reviewable PRs rather than as a big-bang upgrade; a **lock file** (Gradle's dependency locking, Maven's `dependency:go-offline` plus a committed resolved list) for reproducible builds; and **CVE scanning** (OWASP Dependency-Check, Snyk, Trivy, or GitHub's dependency graph) — which frequently *is* the reason you are forcing a version in the first place.

Finally, the structural point: this problem is why **JPMS** and, in other ecosystems, isolated classloaders exist — the JVM's flat classpath fundamentally cannot express "two versions of one library". OSGi and application servers solve it with classloader isolation (Q11); shading solves it by renaming; everything else is choosing a winner.

### Probes

**`dependency:tree` / `gradle dependencies`.** Covered, including `-Dverbose` and `dependencyInsight`.

**Maven nearest-wins vs Gradle highest-wins.** Covered precisely, including tie-breaking by declaration order and Gradle's rich versions.

**`dependencyManagement` / BOM.** Covered as the preferred fix, with the Spring Boot property-override detail.

**Exclusions.** Covered, with their specific risks.

**Shading/relocation.** Covered, with an honest account of the costs and when it is justified.

**`NoSuchMethodError` at runtime.** Covered — the late-linkage mechanism, related errors, and two diagnostic techniques.

**Enforcer plugin.** Covered, with the three highest-value rules and the wider CI toolchain around them.

---

## Q93. What is your policy on flaky tests, and how do you fix a time-dependent one?

### Answer

**Why flakiness is a serious problem, not an annoyance.** A test suite that fails 5% of the time for no reason destroys the only thing a test suite is for: a trustworthy signal. The observable consequences are predictable and severe — engineers reflexively re-run red builds, which means **real** failures get re-run and ignored too; merge queues clog; deploy confidence erodes; and eventually people stop reading test results at all. A suite with 200 tests where 10 are flaky is, in practice, a suite with no tests, because you cannot distinguish a genuine regression from noise.

**The policy: quarantine, then fix on a deadline, then delete.**

1. **Detect systematically.** Do not rely on someone noticing. Track pass/fail history per test (Gradle Enterprise / Develocity, Datadog CI Visibility, or a simple database populated from JUnit XML). A test that has failed and then passed on the same commit is flaky by definition. Re-run failed tests once *and record it* — silently retrying without recording is how flakiness becomes permanent.
2. **Quarantine immediately.** Move it out of the blocking build (a JUnit `@Tag("quarantined")` excluded from the PR gate) but keep running it on a schedule so its status stays visible. This restores a green, trustworthy main build within minutes. Quarantine is not absolution — it is triage.
3. **Assign an owner and a deadline.** Two weeks is a reasonable default. A quarantined test with no owner is a deleted test that still costs CI minutes.
4. **Fix or delete at the deadline.** Deleting is a legitimate outcome, and saying so is important: a test nobody will fix provides zero coverage and negative signal value. Deleting it honestly reports "we do not test this" instead of pretending otherwise. What you must *not* do is leave it disabled indefinitely with `@Disabled("flaky")` and no record — that is the worst of both worlds.
5. **Cap the quarantine list.** If it exceeds a threshold (say 10, or 1% of the suite), that becomes the team's priority. Otherwise it grows monotonically.
6. **Treat a flaky test as a defect report about the code, not just the test.** A meaningful proportion of flakiness reveals genuine race conditions, ordering dependencies, or timing assumptions in *production* code. `@Retry` masks real bugs. Investigate before you retry.

**Fixing a time-dependent test**

Time dependence comes in three flavours, with three different fixes.

**(a) `Thread.sleep` waiting for asynchrony — replace with polling.**

```java
// Flaky: 100ms is enough on your laptop, not on a loaded CI runner
service.processAsync(order);
Thread.sleep(100);
assertThat(repository.findById(id).get().getStatus()).isEqualTo(PROCESSED);
```

The failure is structural: the sleep is simultaneously **too short** (fails under CI contention, on a cold JIT, during a GC pause) and **too long** (every run pays the full duration even when the work took 3ms). A suite with 200 such sleeps wastes minutes per run.

```java
// Correct: poll until the condition holds, with a generous ceiling
await().atMost(Duration.ofSeconds(5))
       .pollInterval(Duration.ofMillis(50))
       .untilAsserted(() -> assertThat(repository.findById(id))
               .get().extracting(Order::getStatus).isEqualTo(PROCESSED));
```

Awaitility returns as soon as the condition is met (usually milliseconds) and only fails after a ceiling that is generous enough to absorb CI variance. Better still where possible: **remove the asynchrony from the test** — expose a deterministic hook (a `CountDownLatch`, a returned `CompletableFuture`, a synchronous executor injected in tests via `Runnable::run`). A test that does not have to wait cannot be flaky about waiting.

**(b) Dependence on the current clock — inject a `Clock`.**

```java
// Flaky: fails when run at 23:59:59.999, on a DST boundary, on 29 February,
// or when a "30-day expiry" test runs across a month boundary
if (order.getCreatedAt().plusDays(30).isBefore(LocalDateTime.now())) { ... }
```

```java
@Service
class ExpiryService {
    private final Clock clock;                       // injected
    ExpiryService(Clock clock) { this.clock = clock; }
    boolean isExpired(Order o) {
        return o.getCreatedAt().plus(RETENTION).isBefore(Instant.now(clock));
    }
}

@Bean Clock clock() { return Clock.systemUTC(); }    // production

// test
var fixed = Clock.fixed(Instant.parse("2026-03-01T12:00:00Z"), ZoneOffset.UTC);
var service = new ExpiryService(fixed);
```

`java.time.Clock` exists precisely for this, and every `now()` in `java.time` accepts one. Now you can test the boundary exactly (`Clock.offset(fixed, Duration.ofDays(30))`), test leap days and DST transitions deliberately, and the test is deterministic forever. **Ban bare `Instant.now()` / `LocalDate.now()` in production code** — it is enforceable with an ArchUnit rule or a Checkstyle regex, and it is one of the highest-value rules you can add.

Alternatives when injection is impractical: `mockStatic(Instant.class)` with `mockito-inline` (works, but static mocking is a smell and does not survive into other threads), or Testcontainers with a controlled container clock for integration tests. Prefer injection.

**(c) Dependence on timing/ordering of concurrent code.** Genuinely hard. Use deterministic scheduling (inject a `DirectExecutor` in tests), `CountDownLatch`/`Phaser` to force specific interleavings, or a tool like `jcstress` for JMM-level testing. If a test is flaky because production code has a race, **fix the race** — this is the case where flakiness is doing its job.

**Test isolation and shared state**

The other major flakiness source, and the one that produces the maddening "passes alone, fails in the suite" symptom.

- **Static/singleton state** — a static cache, a `ThreadLocal`, a static counter, a `Locale`/`TimeZone` default changed by one test. Reset in `@AfterEach`, or avoid statics.
- **Database state** — the most common. Tests that assume an empty table, or that depend on IDs from a sequence. Fixes, in order of preference: make each test create its own data with unique identifiers (`UUID.randomUUID()` in the name/key) so tests are independent; use `@Transactional` rollback for *unit-ish* tests (with the caveat in the next section); truncate/reset between tests (`@Sql`, or a fast truncation utility); and give each test class a fresh schema or container where isolation matters more than speed.
- **Filesystem** — `@TempDir` (JUnit 5) instead of a fixed path.
- **Ports** — never hard-code a port; use `@SpringBootTest(webEnvironment = RANDOM_PORT)` and Testcontainers' dynamic port mapping. Fixed ports collide under parallel execution.
- **Order dependence** — a test that passes only after another test ran. JUnit 5 does not guarantee order by default; enable `junit.jupiter.testmethod.order.default` with a *random* orderer in CI to surface these deliberately. If tests must share expensive setup, share the *setup*, not the *state*.
- **External services** — any test hitting a real network endpoint is flaky by construction. Use WireMock or Testcontainers.

**Testcontainers reuse**

Starting a PostgreSQL container per test class costs seconds; across 80 classes it dominates the build.

- **Singleton container pattern** — a `static` container started once per JVM, shared by all tests (either a static field with `@Container` omitted from lifecycle management, or a shared base class). Combined with `@DynamicPropertySource` to point Spring at it. This is the standard approach and usually the biggest single build-time win.
- **`.withReuse(true)`** plus `testcontainers.reuse.enable=true` in `~/.testcontainers.properties` keeps the container alive **between builds** on a developer machine — excellent locally, and deliberately not enabled by default on CI (where the container should be ephemeral and the daemon is usually torn down anyway).
- **The trade-off:** a shared container means shared state, so reuse and isolation are in tension. Resolve it by making tests data-independent (unique keys per test) rather than by restarting containers. Where a test genuinely needs a pristine database, give that one class its own container.
- Spring Boot 3.1+ makes this much cleaner with `@ServiceConnection` (no manual `@DynamicPropertySource`) and `spring-boot-testcontainers` for local development.

**Parallel execution**

Once tests are genuinely isolated, parallelism is where the build-time win is (`junit.jupiter.execution.parallel.enabled=true`, `...mode.default=concurrent`). But it is the ultimate flakiness amplifier: every shared-state assumption that was hidden by sequential execution now fails intermittently. Sequence matters — **isolate first, parallelise second.** Use `@Execution(SAME_THREAD)` and JUnit 5's `@ResourceLock` for the tests that genuinely cannot be parallelised, rather than abandoning parallelism entirely.

### Probes

**Quarantine vs delete vs fix.** Covered as an explicit six-step policy, including deletion as a legitimate outcome and a cap on the quarantine list.

**`Thread.sleep` → Awaitility.** Covered, with the "too short and too long simultaneously" explanation and the preference for removing the wait entirely.

**Fixed `Clock` injection.** Covered with production and test code, plus the enforceable ban on bare `now()`.

**Test isolation / shared state.** Covered across seven categories with concrete fixes.

**Ordering dependence.** Covered, including deliberately randomising order in CI to surface it.

**Testcontainers reuse.** Covered — singleton pattern, `withReuse`, the isolation trade-off, and `@ServiceConnection`.

**Parallel execution.** Covered, with the "isolate first, parallelise second" ordering and the escape hatches.

---

## Q94. Design a CI pipeline for a Spring Boot service. What runs on PR vs merge vs release?

### Answer

**The organising principle: fast feedback first, and each stage should be more expensive and more thorough than the last.** Order stages by (cost × probability of catching something) so the cheapest, highest-yield checks fail first. A PR pipeline that takes 40 minutes changes developer behaviour for the worse — people batch changes, context-switch, and stop running it locally.

**On every push to a PR — target under 10 minutes**

1. **Compile** (~30s). Fails fast on the obvious.
2. **Static analysis, in parallel** — Checkstyle/Spotless (formatting; auto-fixable, so fail with the diff), SpotBugs/Error Prone (real bug patterns — Error Prone in particular catches genuine defects at compile time), and ArchUnit (architectural rules from Q79). Run these **concurrently** with tests, not before, so they don't serialise the pipeline.
3. **Unit tests** (~1–3 min). No Spring context, no database, no network. These should be the bulk of your tests and should run in seconds. Parallelised (Q93).
4. **Integration tests with Testcontainers** (~3–6 min). Real PostgreSQL, real Kafka, real Redis. This is where the highest-value coverage lives for a typical service — the interaction between your code and the actual infrastructure, including SQL correctness, transaction behaviour, serialisation, and Flyway migrations. Testcontainers made "spin up real dependencies" cheap enough to be the default; use it rather than mocking your database.
5. **Consumer-driven contract tests (provider side)** — verify you have not broken any consumer (below).
6. **Build the image** and run **CVE + secret scanning** on it (Trivy/Grype for vulnerabilities, gitleaks for secrets). Do not push it, or push to a short-lived PR tag.
7. **Coverage and quality gates** — report, and gate on *changed* lines rather than the project total (below).

**Also on PR:** a fast Flyway migration check — apply all migrations to an empty database *and* to a snapshot of the production schema, catching a migration that works on a fresh database but not on the real one. And a manifest validation step (`kubeconform`, `helm template` + `kubectl --dry-run=server`, Conftest/OPA policies) — a large share of production incidents come from manifest errors that no Java test can catch.

**On merge to main**

Everything from the PR pipeline (against the merge result, not the branch — a PR that was green can break after merging), plus:

1. **Build the definitive, immutable artifact** and image, tagged with the commit SHA. Generate an **SBOM** (CycloneDX or SPDX) and **sign** the image (Cosign/Sigstore) with provenance attestation. Push to the registry.
2. **Deploy automatically to a staging/integration environment.**
3. **Slower suites** that would be too costly on a PR: end-to-end tests against staging, performance smoke tests (a short k6/Gatling run asserting p99 has not regressed beyond a threshold), a rolling-deployment zero-downtime check (Q68 — load generator running through a real deployment asserting zero non-2xx), and mutation testing (PIT) on critical modules.
4. **Publish contract test results** to the broker so consumers can verify.

**On release / promotion to production**

The critical principle: **promote the exact artifact that passed, do not rebuild.**

Rebuilding from the same commit does *not* guarantee the same bytes — a mutable base-image tag has moved, a transitive `latest` dependency resolved differently, a plugin updated, the build ran on a different JDK patch. You would then be deploying something no test ever ran against, which defeats the entire pipeline. So: build once at merge, tag by digest, and promote that digest through staging → production by changing only the environment configuration. Immutable artifacts + config-per-environment is the foundation of a trustworthy pipeline.

Release-specific steps: verify the image signature and its attestations at admission (Kyverno/Sigstore policy controller — this is what makes signing meaningful rather than ceremonial); run database migrations as a separate, gated pipeline step (Q53) rather than at application startup; deploy progressively (canary or blue-green) with automated rollback on SLO burn (Argo Rollouts / Flagger watching error rate and latency); and create the release record — changelog, Git tag, deployment marker sent to your observability platform so a latency graph annotates "deploy at 14:03".

**Static analysis and coverage gates — and how they get gamed**

This is the part of the question with the most substance, and the honest answer matters.

**Coverage gates get gamed, reliably.** Because line coverage measures *execution*, not *verification*, a test that calls every method and asserts nothing scores 100%. Real patterns I have seen: tests with no assertions; asserting `assertNotNull(result)`; adding `@Generated` or exclusion patterns until the number goes up; writing tests for trivial getters to raise the average while leaving the complex branch untested. Chasing a number produces exactly this, because the number is what is rewarded (Goodhart's law: a measure that becomes a target ceases to be a good measure).

More useful approaches:
- Gate on **coverage of changed lines** (`diff` coverage) rather than the project total. It is actionable, it does not punish a team for a legacy codebase, and it prevents the ratchet from going backwards. 80% on new code is a reasonable bar.
- Use coverage as a **discovery tool, not a gate** — look at *what* is uncovered, not the percentage. A 60% figure where the missing 40% is generated code and DTOs is far healthier than 85% where the missing 15% is the payment state machine.
- **Mutation testing (PIT)** measures whether tests actually *detect* changed behaviour, which is much closer to what you want and is essentially ungameable by assertion-free tests. It is too slow for the whole suite on every PR, but excellent on critical modules nightly.

**Static analysis gets gamed** through blanket `@SuppressWarnings`, ratcheting the threshold, or marking everything "won't fix". Mitigations: fail only on **new** issues (SonarQube's "new code" quality gate is designed for this); require a justification comment on suppressions and review them; and keep the rule set small and high-signal — a tool that produces 400 warnings gets ignored entirely, while one that produces three real ones gets fixed.

The honest framing: **quality gates are guardrails against carelessness, not a substitute for review and design.** State that in an interview; it is the thing that distinguishes someone who has operated a pipeline from someone who has configured one.

**CVE scanning — with an important caveat**

Scan dependencies (OWASP Dependency-Check, Snyk, GitHub) and the built image (Trivy, Grype). But calibrate: a scanner reporting 200 CRITICALs in a base image, none reachable from your code, trains everyone to ignore it. Practical approach: fail the build on **new** CRITICAL/HIGH findings in **direct** dependencies and in the application layer; track base-image findings separately with an SLA and automated base-image bumping (Renovate); and use **reachability analysis** where the tool supports it, to distinguish "this CVE is in a code path you actually call" from "this class is on the classpath". Maintain an explicit, expiring exception list with a named owner — not an indefinite ignore file.

**Consumer-driven contract testing** (also probed in Q95): the provider verifies, in *its* pipeline, that it still satisfies every consumer's recorded expectations, published via a Pact Broker or Spring Cloud Contract. This is what allows a breaking change to be caught **before merge** rather than in an integration environment — and it is the mechanism that makes "avoid versioning by not breaking things" (Q58) enforceable rather than aspirational.

**Cross-cutting pipeline practices worth naming:** cache dependencies and build outputs aggressively (Gradle remote build cache, or `--mount=type=cache` from Q62); run independent stages in parallel; keep the *critical path* short rather than the total work small; make every stage reproducible locally with one command (if developers cannot run it, they will not); pin all tool versions; and monitor the pipeline itself — build duration, flake rate, and time-from-merge-to-production are engineering metrics worth tracking (they are essentially the DORA metrics).

### Probes

**Fast feedback ordering.** Covered as the organising principle, with a stage-by-stage PR pipeline under 10 minutes.

**Testcontainers integration tests.** Covered, with the argument for why they carry the highest-value coverage.

**Static analysis / coverage gates and their gaming.** Covered candidly — the specific gaming patterns, diff coverage, mutation testing, "new code only" gating, and the framing that gates are guardrails rather than a substitute for review.

**CVE scanning.** Covered, including the alert-fatigue calibration problem and reachability analysis.

**Image signing / SBOM.** Covered, with the point that signing is meaningless without admission-time verification.

**Artifact immutability.** Covered.

**Promotion vs rebuild.** Covered, with the concrete reasons a rebuild produces different bytes.

---

## Q95. How do you test a Kafka consumer, a `@Transactional` service method, and a REST controller?

### Answer

**Framing: choose the smallest test that gives real confidence.** Each of these three has a different correct default, and the interesting part is knowing what each level does *not* catch.

**1. A Kafka consumer**

Three options, in increasing fidelity and cost:

**(a) Test the handler as a plain method — the default.** Most consumer logic has nothing to do with Kafka. Extract the business logic from the `@KafkaListener` method and unit-test it directly with a constructed domain object. Milliseconds, no infrastructure. The listener method itself should be a thin adapter: deserialise, delegate, done.

**(b) `EmbeddedKafka` (`spring-kafka-test`).** An in-JVM broker. Faster to start than a container (a few seconds) and convenient, but it is *not* the same software as production — different version, different defaults, and it has historically had behavioural differences around rebalancing, transactions, and timing. Adequate for testing wiring; not for testing anything subtle.

**(c) Testcontainers with a real Kafka (or Redpanda) — the recommended integration test.** You run the actual broker, at the actual version. This is what catches serialisation problems, consumer configuration errors, partition assignment behaviour, offset commit semantics, retry/DLQ routing (Q84), and error-handler configuration. Redpanda's container starts in ~1 second and is Kafka-API-compatible, which removes most of the speed argument for `EmbeddedKafka`.

```java
@SpringBootTest
@Testcontainers
class OrderConsumerIT {
    @Container @ServiceConnection                       // Boot 3.1+: no @DynamicPropertySource needed
    static final KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.0"));

    @Autowired KafkaTemplate<String, OrderEvent> template;
    @Autowired OrderRepository repository;

    @Test
    void persistsOrderOnEvent() {
        template.send("orders", "order-1", new OrderEvent("order-1", PLACED));

        await().atMost(Duration.ofSeconds(10)).untilAsserted(() ->
            assertThat(repository.findById("order-1")).isPresent());
    }
}
```

Note the Awaitility poll rather than a sleep (Q93) — consumption is asynchronous and the delay is unpredictable.

**What to test at this level specifically:** the happy path end-to-end; a **poison message** routing to the DLQ; **idempotency** — send the same event twice and assert exactly one effect (Q83); and **ordering** if you depend on it. These are the behaviours that only appear with a real broker.

**2. A `@Transactional` service method**

**Why `@Transactional` on the test class hides bugs — the important part.**

Spring's `@Transactional` on a *test* wraps the whole test in a transaction that is **rolled back** at the end. Convenient, and genuinely useful for isolation — but it silently changes semantics in four ways that mask real defects:

1. **Everything shares one transaction and one persistence context.** The test, the service, and the repository all run inside it. So `@Transactional(propagation = REQUIRES_NEW)` in your service **does not actually get a new transaction the way it would in production** relative to the caller's, and any code that depends on transaction boundaries behaves differently.
2. **The persistence context is never cleared**, so entities remain managed for the whole test. An assertion that appears to read from the database is actually reading the **first-level cache**. A missing `save()`, a broken mapping, an unflushed change, or a column that does not exist can all pass — the object is returned from memory. Then it fails in production. This is the classic false-green.
3. **Nothing is ever flushed or committed**, so `NOT NULL` violations, unique-constraint violations, foreign-key violations, check constraints, and column-length overflows are never triggered — the SQL is never sent. Add `entityManager.flush()` (or `TestEntityManager.flush()`) before assertions to force it, and `clear()` to force a real re-read.
4. **`@TransactionalEventListener(phase = AFTER_COMMIT)` never fires**, because there is no commit. Neither does anything registered via `TransactionSynchronization`, including outbox relays and cache evictions. A whole class of logic is silently untested.

**How to test transactional behaviour properly:**

- **Do not put `@Transactional` on tests that verify transactional behaviour.** Use a real commit and clean up explicitly (truncate in `@AfterEach`, or use unique data per test — Q93).
- **Test rollback explicitly:** call a method that throws, then in a *fresh* transaction assert that nothing was written. This is the only way to catch the extremely common bug of a checked exception not rolling back by default (Q29).
- **Use `TestTransaction`** (`TestTransaction.flagForCommit()`, `end()`, `start()`) when you need multiple transactions within one test — e.g. write in one, then verify visibility in another.
- **Use `TransactionTemplate`** in the test to control boundaries explicitly.
- **Assert the SQL actually issued.** A `DataSource` proxy (`datasource-proxy`, `p6spy`, or `com.vladmihalcea:db-util`'s `SQLStatementCountValidator`) lets you assert "this operation issues exactly 3 queries" — which is the only reliable regression test for **N+1** problems (Q42). Highly recommended; N+1s otherwise reappear silently.
- Use `@DataJpaTest` (a slice: JPA, transactions, and an embedded or Testcontainers datasource, no web layer) for repository-level tests — but **override the default embedded database** with Testcontainers PostgreSQL via `@AutoConfigureTestDatabase(replace = NONE)` plus `@ServiceConnection`, because H2 accepts SQL PostgreSQL rejects and vice versa. Testing against a different database than you deploy to is a false-confidence generator.

**3. A REST controller**

**`@WebMvcTest` + `MockMvc` — the default.** A slice test loading only the web layer: the controller, `@ControllerAdvice`, converters, filters, and Spring Security configuration, with collaborators mocked (`@MockitoBean` in Boot 3.4+, formerly `@MockBean`). It starts in a second or two and no server socket is opened.

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mvc;
    @MockitoBean OrderService service;

    @Test
    void returns404WithProblemDetail() throws Exception {
        given(service.find("A-1")).willThrow(new OrderNotFoundException("A-1"));

        mvc.perform(get("/orders/A-1").accept(APPLICATION_JSON))
           .andExpect(status().isNotFound())
           .andExpect(content().contentType(APPLICATION_PROBLEM_JSON))
           .andExpect(jsonPath("$.type").value("https://errors.acme.com/order-not-found"))
           .andExpect(jsonPath("$.orderId").value("A-1"));
    }
}
```

This is where you test the things that are *the controller's job*: status codes, request validation (`@Valid` and the resulting `400`/`422` shape), serialisation of the response, error mapping through `@ControllerAdvice` (Q59), content negotiation, header handling (`If-Match` → `412`, Q57), and security rules (`@WithMockUser`, and asserting that an unauthenticated call gets `401`). Testing security through `MockMvc` requires `@AutoConfigureMockMvc` with security enabled or `springSecurity()` in the standalone setup — a very common omission that leaves authorization untested.

**`MockMvc` vs `WebTestClient` vs `@SpringBootTest(RANDOM_PORT)`:**

| | Loads | Real HTTP? | Use for |
|---|---|---|---|
| `MockMvc` | Web slice | No — mocked servlet environment | Controller logic, fast, the default for MVC |
| `WebTestClient` | Slice or full | Optional — bound to a `MockMvc`/mock server, or to a real port | The default for **WebFlux**; also a nicer fluent API for MVC (`@AutoConfigureWebTestClient`) |
| `@SpringBootTest(webEnvironment = RANDOM_PORT)` + `TestRestTemplate`/`WebTestClient` | Whole application | **Yes** | Full-stack verification, filter chains, real serialisation over the wire |

`MockMvc` does **not** exercise the real servlet container, so it can miss: container-level filters, actual HTTP header and encoding behaviour, connection handling, `Content-Length`/chunking, and anything Tomcat-specific. Use a `RANDOM_PORT` test for a small number of end-to-end paths and `MockMvc` for breadth.

**Consumer-driven contract testing — what none of the above catches**

All three levels test *your* service against *your* assumptions. None of them catches "I renamed a field and three consumers broke", because your tests were updated alongside the change and stayed green.

Contract testing closes that gap. With **Pact** or **Spring Cloud Contract**:
- Each **consumer** writes a test against a stub, declaring what it needs: "given order A-1 exists, `GET /orders/A-1` returns `200` with a `status` field". Running that test **publishes a contract** to a broker.
- The **provider's** pipeline (Q94) replays every published contract against the real provider. Rename `status` to `state` and the provider build fails, naming the consumer that will break — **before merge**.
- Pact's `can-i-deploy` then gates deployment on the compatibility matrix of what is actually running in each environment.

This is fundamentally different from an integration test: it is **consumer-driven** (the consumer states what it needs, so you only guarantee what someone actually uses), it runs without deploying both services together, and it scales to many consumers. It is the practical enabler for the additive-evolution strategy of Q58, and it is what makes the "avoid breaking changes" policy verifiable rather than a promise.

**The overall shape.** Many fast unit tests for domain logic; a solid layer of Testcontainers integration tests for repository, consumer, and adapter behaviour against real infrastructure; slice tests for controllers; a handful of full-stack tests for critical paths; and contract tests at every service boundary. What I would *avoid*: a large number of `@SpringBootTest` tests that load the whole context for logic that needed no context at all — they dominate build time and give little more confidence than the slice.

### Probes

**Embedded Kafka vs Testcontainers vs a fake.** Covered with all three, a recommendation, and what each fails to catch.

**Why `@Transactional` on the test class hides bugs.** Covered in detail — four distinct mechanisms (shared transaction, unclear persistence context, no flush/commit, no after-commit events) and the correct alternatives.

**`@WebMvcTest` + `MockMvc` vs `WebTestClient`.** Covered in a comparison table, including when a real port is required and what `MockMvc` cannot see.

**Consumer-driven contract testing.** Covered as the gap none of the other levels closes, with the mechanism and its link to API evolution policy.

---

# 14. System Design

---

## Q96. Design a URL shortener. 50,000 reads/sec, 500 writes/sec.

### Answer

**Start with the numbers, because they determine the architecture.**

- 500 writes/sec ≈ 43M/day ≈ **15.8 billion URLs over 3 years**.
- 50,000 reads/sec — a **100:1 read:write ratio**. This is a read-dominated system, and everything follows from that.
- Storage: ~500 bytes/record (long URL, short code, owner, timestamps) × 15.8B ≈ **8 TB** over three years. Too large for one node's working set, small enough that sharding is straightforward.
- Read bandwidth: mostly redirects — small responses, but 50k/sec of them.

**Key-space sizing.** Base62 (`[a-zA-Z0-9]`) gives 62^n codes: 62^6 ≈ 56 billion, 62^7 ≈ 3.5 trillion. **7 characters** comfortably covers 15.8B with room to spare and keeps URLs short. Start at 7.

**ID generation — the central design decision**

**(a) A counter, base62-encoded.** Take a monotonically increasing integer and encode it. Guarantees uniqueness with zero collision checking, and produces the shortest possible codes. The problem is generating the counter at scale without a bottleneck:
- A single database sequence is a single point of contention and failure.
- **Better: pre-allocated ranges.** Each application instance claims a block of 100,000 IDs from a central allocator (a row in a database, or Redis `INCRBY`) and hands them out from memory. One coordination call per 100,000 writes instead of per write. Gaps on restart are irrelevant.
- Downside: codes are **sequential and therefore enumerable**. Anyone can walk the key space and discover every URL — a real privacy problem, since shortened URLs often point at unlisted documents. Mitigate by multiplying the counter by a large odd number modulo 62^7 (a bijective "Feistel"-style scramble), which preserves uniqueness while destroying ordering.

**(b) Hash the URL** (MD5/SHA-256, take the first 42 bits, base62-encode). Naturally deduplicates identical URLs. But collisions are certain at scale — by the birthday bound, with 62^7 ≈ 3.5×10^12 codes and 1.58×10^10 stored, you expect collisions well before you finish. So you must **check and retry on collision**, which means a read before every write.

**(c) Snowflake-style IDs** (timestamp + machine ID + sequence). Coordination-free and time-ordered, but 64 bits base62-encodes to 11 characters — too long for a URL shortener, where brevity is the product.

**(d) Random codes with a uniqueness check.** Generate 7 random base62 characters, `INSERT` and let the unique constraint reject duplicates, retry on failure. With 15.8B of 3.5T used (0.45% occupancy), the collision probability per attempt is 0.45%, so retries are negligible. Non-enumerable by construction.

**My choice: pre-allocated counter ranges with a bijective scramble** (option a), which gives no collision checks, no reads on the write path, and non-enumerable codes. **Random-with-unique-constraint (d) is an equally defensible answer** and is simpler — say so, since the interviewer is testing whether you can reason about the trade-off rather than recite one answer.

**Handling collisions** regardless of scheme: the **unique constraint on the short code is the enforcement point**, never a check-then-insert (Q56). Catch the duplicate-key exception and retry with a new candidate, capped at 3–5 attempts before returning `503`.

**The read path — where the design actually lives**

50,000 reads/sec against a database is the wrong shape. The read path should barely touch it:

1. **CDN / edge cache.** A redirect is a tiny, highly cacheable response. Serving it from an edge PoP eliminates most of the 50k/sec before it reaches you and gives users a much lower latency redirect. This is the single biggest lever.
2. **Redis cache** in front of the database, keyed by short code. The access distribution is heavily Zipfian — a small fraction of links carry most traffic (a viral link can be millions of hits while most links get single digits). A modest cache achieves a very high hit rate. Cache-aside with a long TTL (mappings are immutable once created, which makes caching unusually easy) and **negative caching** for unknown codes to prevent scanning attacks from hitting the database (Q87).
3. **Database** only on a cache miss.

With a 95% edge + 95% Redis hit rate, the database sees roughly 125 reads/sec — trivial.

**Database choice and sharding**

The access pattern is a pure key-value lookup by short code, with no joins and no range scans. That argues for a key-value or wide-column store — **DynamoDB or Cassandra** partitioned by short code — which scales horizontally by design and handles this shape natively.

**PostgreSQL is also a perfectly good answer** at this scale if you shard: partition or shard by `hash(short_code)` across N nodes, each holding a manageable slice, with read replicas per shard. Given 8 TB and simple access, 8–16 shards is comfortable.

**Shard key: the short code**, because that is the only key the read path uses. Sharding by user ID would make every redirect a scatter-gather — the classic sharding mistake. Secondary access patterns (a user listing their links) need a separate index or table sharded by user ID, kept eventually consistent.

Schema:
```sql
CREATE TABLE links (
    short_code   varchar(10) PRIMARY KEY,
    long_url     text        NOT NULL,
    owner_id     bigint,
    created_at   timestamptz NOT NULL DEFAULT now(),
    expires_at   timestamptz,
    is_active    boolean     NOT NULL DEFAULT true
);
```

**301 vs 302 — and why analytics decides it**

| | `301 Moved Permanently` | `302 Found` / `307 Temporary Redirect` |
|---|---|---|
| Browser caching | Cached aggressively, often indefinitely | Not cached by default |
| Subsequent clicks | Never reach your server | Every click reaches your server |
| Analytics | **You see the first click only** | You see every click |
| Server load | Much lower | Full load |
| Changing the target later | Effectively impossible — browsers keep the old target | Works immediately |

For a URL shortener, **click analytics is usually the product** — it is what customers pay for. That makes `302` (or `307`, which additionally guarantees the method is preserved) the correct default, accepting the higher load. Use `301` only for links explicitly marked immutable and non-analytic, where SEO link-equity transfer matters.

A useful refinement: `302` with a short `Cache-Control: max-age` (say 60s) to blunt bursts from a viral link while keeping analytics substantially accurate.

**Custom aliases.** A user requests `acme.co/spring-sale`. Same table, same unique constraint — the insert either succeeds or returns `409 Conflict`. Requirements: a reserved-word list (`api`, `admin`, `login`, `health`, and any existing route) so a custom alias cannot shadow application paths; a profanity/abuse filter; a minimum length so custom aliases do not collide with the generated key space (or better, generate into a namespace that custom aliases cannot occupy, e.g. generated codes are always exactly 7 characters and custom aliases must differ in length or be checked against the generator's space); and rate limiting, since alias squatting is a real problem.

**Expiry.** Support `expires_at`. Serve `410 Gone` for expired links (not `404` — it tells clients and crawlers to stop retrying, Q59). Deletion at scale: do **not** run `DELETE ... WHERE expires_at < now()` over 15 billion rows. Instead, **partition by month** and drop whole partitions, or use the store's native TTL (DynamoDB TTL, Cassandra TTL), which is O(1) operationally. Expired entries must also be evicted from Redis and the CDN — or accept a bounded staleness window and set cache TTLs shorter than the minimum link lifetime.

**Rate limiting.** On writes, per API key and per IP (Q98) — otherwise you are a free, anonymous, high-volume redirect service, which is exactly what phishing and spam campaigns want. On reads, limit per IP to frustrate enumeration scanning, alongside negative caching. Also needed: malicious-URL scanning at creation time (Google Safe Browsing or equivalent), an abuse-report path, and the ability to disable a link instantly (`is_active = false`, with cache invalidation) — which is a hard requirement in practice, not a nice-to-have.

**Additional pieces worth naming briefly:** analytics writes must not be synchronous on the redirect path — fire an event to Kafka and aggregate asynchronously, so a slow analytics pipeline never slows a redirect. Multi-region deployment with the read path served locally and writes routed to a primary region (or a globally distributed store) if latency targets demand it. And the write path is straightforward at 500/sec — a single primary handles it comfortably; the entire engineering challenge is the read path and the key space.

### Probes

**ID generation (counter/hash/snowflake).** All covered with a defended choice and an acknowledged alternative.

**Collisions.** Covered — birthday-bound reasoning, and the unique constraint as the enforcement point.

**Read caching + CDN.** Covered as the primary lever, with hit-rate arithmetic and negative caching.

**DB choice and sharding key.** Covered, including why sharding by user ID would be wrong.

**301 vs 302 for analytics.** Covered in a table with a defended default and a hybrid refinement.

**Custom aliases.** Covered, including reserved words and key-space separation.

**Expiry.** Covered, with `410` and partition-drop/TTL rather than bulk `DELETE`.

**Rate limiting.** Covered on both read and write paths, with the abuse considerations that make it mandatory.

---

## Q97. Design a notification service. Guaranteed delivery, no double-send.

### Answer

**Scope and interface.** A `POST /notifications` API accepting `{ userId, templateId, channel?, data{}, idempotencyKey, priority }`. Channels: email, SMS, push. The service owns templating, channel selection, provider integration, delivery tracking, and preferences.

**The architecture**

```
API ──▶ [validate, persist, dedupe] ──▶ outbox/Kafka ──▶ dispatcher
                                                            │
                                    ┌───────────────────────┼───────────────────────┐
                                    ▼                       ▼                       ▼
                              email workers           sms workers            push workers
                                    │                       │                       │
                              SendGrid/SES            Twilio/Vonage           APNs/FCM
                                    └───────── delivery webhooks ────────────▶ status store
```

**The API only enqueues.** It validates, resolves the recipient, applies deduplication, persists a `notification` row in `PENDING`, and publishes to a queue — then returns `202 Accepted` with the notification ID (Q55). It must never call a provider synchronously: providers have multi-second latencies and outages, and a slow provider would otherwise become your API's latency and availability.

Use the **transactional outbox** (Q76) between the database write and the queue publish, so you can never have a persisted notification with no queued work, or a queued message with no record.

**Guaranteed delivery**

"Guaranteed" is at-least-once, and it comes from a chain of durable handoffs, each of which must not lose work:

1. **Durable persistence before acknowledging.** The `202` is only returned after the row is committed.
2. **Durable queue** (Kafka with `acks=all` and `min.insync.replicas=2`, or SQS). Never an in-memory queue.
3. **Acknowledge only after the provider accepts.** Commit the Kafka offset (or delete the SQS message) *after* the provider returns a success, not before (Q82). A crash mid-send therefore causes redelivery, not loss.
4. **Retry with exponential backoff and jitter** on transient failures, bounded, then **DLQ** (Q84).
5. **A reconciliation sweeper** — a scheduled job finding notifications stuck in `PENDING`/`SENDING` beyond a threshold and re-enqueuing them. This is the safety net that catches everything the happy path missed (a worker that died between dequeue and send, a lost message, a bug). Without it, "guaranteed" is aspirational.
6. **Provider-level durability** — most providers accept and queue, then deliver asynchronously, so their `202` means "accepted", not "delivered". Track the difference (below).

**No double-send — three independent layers, because one is not enough**

**(a) Request-level idempotency.** The caller supplies an `Idempotency-Key`; a unique constraint on it means a retried API call returns the original notification ID rather than creating a second one (Q56). This handles caller retries.

**(b) Worker-level idempotency.** Queues are at-least-once, so a worker *will* occasionally process the same message twice. Guard with a state transition that is atomic and conditional:

```sql
UPDATE notifications
   SET status = 'SENDING', attempt = attempt + 1, locked_until = now() + interval '2 minutes'
 WHERE id = ? AND status IN ('PENDING','FAILED_RETRYABLE')
   AND (locked_until IS NULL OR locked_until < now());
-- 0 rows affected → another worker already has it → skip
```

The database, not the application, decides who owns the send. A lease (`locked_until`) rather than a permanent lock means a dead worker's notification is reclaimed automatically.

**(c) Provider-level idempotency key.** The genuinely hard case: the worker calls the provider, the provider sends the email, and the response is lost. The worker cannot tell "not sent" from "sent, ack lost". Retrying double-sends; not retrying risks loss.

Solution: pass a **deterministic** idempotency key to the provider, derived from the notification ID — SendGrid, Twilio, Stripe, and most serious providers support one. The provider then deduplicates on their side, and a retry is safe. Where a provider offers no such key, you must choose a side and be explicit about it: for transactional notifications (password reset, payment receipt), **prefer at-least-once** — a duplicate email is an annoyance, a missing password reset is a support ticket and possibly a lockout. For marketing sends, prefer at-most-once.

Record the provider's message ID on success, so a later reconciliation can query "did this actually go out?".

**Per-channel provider abstraction and failover**

```java
interface NotificationSender {
    Channel channel();
    SendResult send(RenderedNotification n);   // throws TransientSendException / PermanentSendException
}
```

One implementation per provider, selected via the Strategy/Map pattern of Q74. Benefits: providers are swappable; a second provider per channel enables **failover** when the primary is down or rate-limiting; and you can shift traffic by percentage for cost or deliverability reasons.

Failover needs care: wrap each provider in a **circuit breaker** (Q80), fail over on breaker-open or on transient errors, and ensure the failover path carries its own idempotency key — otherwise failing over *after* a lost ack double-sends across two providers, which no single provider's deduplication can prevent. This is the one case where you may accept a duplicate, and it should be a documented decision.

Also per-channel: different **rate limits** (providers cap throughput), different payload constraints (SMS 160 chars/segment and per-country regulations, push payload size limits), different failure taxonomies, and different costs.

**Preferences and quiet hours**

Checked at dispatch time, not at API time — a notification queued at 22:00 for a user whose quiet hours end at 08:00 should be *delivered* at 08:00, so the check must happen when it is about to be sent.

- Per-user, per-category, per-channel opt-in/opt-out (`user_id, category, channel, enabled`). Categories matter: a user who mutes marketing must still receive security alerts, so classify notifications as **transactional** (cannot be opted out of, and legally must not be treated as marketing) versus **promotional** (opt-out mandatory under CAN-SPAM/GDPR/PECR).
- **Quiet hours in the user's timezone**, stored as an IANA zone (`Europe/Warsaw`), never as a fixed offset — offsets change with DST.
- Implementation: on dispatch, if the current local time is within quiet hours, requeue with a delay until the window ends (a delayed queue, a scheduled topic, or a `scheduled_for` column polled by the sweeper). Add jitter so a million users in one timezone do not all fire at 08:00:00 — otherwise you create your own thundering herd and hit provider rate limits.
- **Frequency capping / digesting** — "at most 3 push notifications per hour", or batching low-priority notifications into a daily digest. This is a product requirement that shows up as an engineering one.
- Also: a global **kill switch** per category and per channel, for when a bug starts sending. You will need it.

**Templating**

Store templates versioned and externally (database or object storage), not in code, so content changes do not require a deployment. Render with a sandboxed engine (Thymeleaf, Handlebars, or Freemarker with the sandbox enabled — **never** an engine that permits arbitrary code, since templates may be authored by non-engineers).

Key points: **localisation** (template per `templateId + locale`, with a fallback chain); **render at dispatch time, not at enqueue time**, so a template fix applies to queued notifications; validate that the supplied `data` satisfies the template's required variables at enqueue time so failures surface synchronously; escape by context (HTML-escape for email, no escaping for plain text) to avoid injection; and keep a rendered snapshot on the notification record for audit and support ("what exactly did we send this customer?").

**Fan-out**

"Notify all 2 million users of an outage" must not be one API call producing one message. Decompose:
- The API accepts a **campaign/audience** request and returns immediately.
- A fan-out worker materialises the audience in **batches** (paged queries, not one huge result set), producing one notification message per recipient.
- Rate-limit the fan-out so it does not saturate providers or your own workers, and give it a **lower priority lane** than transactional traffic.

**Priority lanes** are essential: a 2-million-recipient marketing blast must not delay a password reset. Implement as separate topics/queues (`notifications.high`, `notifications.default`, `notifications.bulk`) with separate consumer groups and separate worker pools — a shared queue with a priority field does not work, because a bulk backlog still sits ahead of new high-priority messages in a partition. Give the high-priority lane guaranteed capacity.

**Delivery receipts and status**

Provider acceptance ≠ delivery. Model the lifecycle explicitly:

`PENDING → SENDING → SENT (provider accepted) → DELIVERED → OPENED/CLICKED`
with terminal branches `FAILED`, `BOUNCED`, `REJECTED` (suppressed), `CANCELLED`.

Providers report progress via **webhooks**. Handling them correctly is a small system in itself (and shares everything with Q100): verify the signature; expect **out-of-order** delivery (a `delivered` webhook can arrive before `sent` — so apply only monotonic state transitions, never blind overwrites); expect **duplicates** (dedupe on the provider's event ID); return `2xx` fast and process asynchronously, or the provider will time out and retry; and reconcile periodically against the provider's API for events whose webhooks were lost.

Act on the results: **hard bounces and complaints must add the address to a suppression list** and stop future sends. Continuing to send to bouncing addresses destroys sender reputation and gets your domain blocked — the most common way an email system fails in practice, and it is an operational requirement, not an optimisation.

Expose `GET /notifications/{id}` returning current status and history, both for callers and for support staff.

**Observability.** Per channel and per provider: send rate, latency, error rate by cause, delivery rate, bounce rate, DLQ depth, queue lag per priority lane, and the count of notifications stuck in non-terminal states. Alert on delivery rate dropping — a provider silently rejecting mail is invisible in your error rate.

### Probes

**Queue + workers.** Covered, with the API-only-enqueues rule and the outbox between DB and queue.

**Per-channel provider abstraction and failover.** Covered, including the circuit breaker and the failover double-send hazard.

**Idempotency key.** Covered as three independent layers, with the deterministic provider key for the lost-ack case.

**Retry + backoff + DLQ.** Covered, plus the reconciliation sweeper as the real guarantee.

**Preferences / quiet hours.** Covered, including timezone handling, transactional-vs-promotional classification, jitter, frequency capping, and a kill switch.

**Templating.** Covered — versioned, external, sandboxed, localised, rendered at dispatch, with an audit snapshot.

**Fan-out.** Covered, with batching and rate limiting.

**Delivery receipts.** Covered — the state machine, webhook handling hazards, reconciliation, and the suppression-list requirement.

**Priority lanes.** Covered, with the argument for separate queues rather than a priority field.

---

## Q98. Rate limiting across 20 instances.

### Answer

**Why local counters fail.** A limit of 100 requests/minute per API key enforced independently on 20 instances permits 2,000/minute, because each instance sees only its own share. Worse, the actual allowance depends on load-balancer distribution, so it is neither correct nor predictable. **State must be shared.**

(Local limiting is still useful as a coarse *self-protection* backstop — but it is not the answer to "limit each customer to N".)

**Algorithms**

**Fixed window.** Count requests per key per calendar window (`key:2026-08-13T09:41`), reset each minute. Trivial and cheap — one `INCR` with a TTL.
*Fatal flaw:* the **boundary burst**. A client sends 100 requests at 09:41:59 and 100 more at 09:42:00 — 200 in one second, both windows technically compliant. You have effectively permitted 2× the limit over any window that straddles a boundary.

**Sliding window log.** Store a timestamp per request in a sorted set; on each request, discard entries older than the window and count the rest. **Perfectly accurate**, and supports exact "N in any 60-second period" semantics.
*Cost:* O(N) memory per key — 10,000 req/min per key means 10,000 stored timestamps per key. At scale this is prohibitive, and the `ZREMRANGEBYSCORE` work grows too. Correct choice for low limits on high-value operations (login attempts, password resets, expensive exports); wrong for general API traffic.

**Sliding window counter.** The pragmatic compromise: keep the current and previous fixed-window counts and interpolate.
```
estimate = current_count + previous_count × (fraction of the previous window still in view)
```
Two integers per key regardless of traffic, and it eliminates the boundary burst almost entirely. The approximation assumes uniform arrival within the previous window, which can be slightly wrong under extreme bursts, but the error is small and bounded. **This is what Cloudflare and most large-scale API gateways use, and it is the right default.**

**Token bucket.** A bucket of capacity `B` refilled at rate `R` per second; each request consumes a token, and an empty bucket means rejection.
*Strength:* it explicitly separates **sustained rate** (`R`) from **burst allowance** (`B`), which matches how APIs are actually consumed — a client that has been idle should be allowed a burst. Requires only two values per key (token count and last-refill timestamp), computed lazily on access rather than by a background refill.
*Leaky bucket* is the sibling that smooths output to a constant rate (a queue draining at fixed speed) — use it when you need to protect a downstream system from bursts entirely, rather than merely limiting the client.

**My default: token bucket** for general API rate limiting (burst semantics match reality and it is cheap), **sliding window log** for security-sensitive low-count limits, and **sliding window counter** where you need simple "N per minute" semantics with accurate boundaries.

**Redis + Lua for atomicity — the essential mechanism**

Every algorithm above is read-modify-write. Done as separate Redis commands, two concurrent requests both read `count = 99`, both decide to allow, and both write `100`. The limit is breached — a check-then-act race, exactly as in Q56.

Redis executes a **Lua script atomically** (single-threaded, no interleaving), making the entire decision one indivisible operation:

```lua
-- token bucket: KEYS[1]=key, ARGV[1]=capacity, ARGV[2]=refill/sec, ARGV[3]=now(ms), ARGV[4]=cost
local b = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(b[1]) or tonumber(ARGV[1])
local ts     = tonumber(b[2]) or tonumber(ARGV[3])
local delta  = math.max(0, tonumber(ARGV[3]) - ts) / 1000
tokens = math.min(tonumber(ARGV[1]), tokens + delta * tonumber(ARGV[2]))
local allowed = tokens >= tonumber(ARGV[4])
if allowed then tokens = tokens - tonumber(ARGV[4]) end
redis.call('HSET', KEYS[1], 'tokens', tokens, 'ts', ARGV[3])
redis.call('PEXPIRE', KEYS[1], 60000)
return { allowed and 1 or 0, math.floor(tokens) }
```

Load once with `SCRIPT LOAD` and invoke with `EVALSHA` to avoid shipping the body each time. Alternatives: Redis's `CL.THROTTLE` (RedisCell module), or **Bucket4j** with a Redis/Hazelcast backend, which is the standard Java library and implements this correctly — prefer it over hand-rolling.

Note that in **Redis Cluster**, all keys touched by one script must hash to the same slot; use a hash tag (`ratelimit:{apiKey}:tokens`) to guarantee that.

**Limit dimensions**

Rate limiting is rarely one limit. Apply several, and reject if *any* is exceeded:
- **Per API key / tenant** — the commercial limit, tied to the customer's plan. The primary one.
- **Per user** within a tenant, so one user cannot consume the whole tenant quota.
- **Per IP** — for unauthenticated endpoints and as an abuse control. Careful: NAT, corporate proxies, and mobile carriers put thousands of legitimate users behind one IP, so IP limits must be generous; and IPv6 should be limited per /64 prefix, not per address, since a single client can have vast numbers.
- **Per endpoint** — an expensive report endpoint deserves a much tighter limit than a health check. Weight requests by cost (`cost` in the script above) rather than counting them equally.
- **Global** — a total-capacity backstop that protects you regardless of who is calling.

Key design: `ratelimit:{dimension}:{identifier}:{window}`, with short TTLs so keys expire automatically and memory stays bounded.

**Clock issues**

- **Use Redis's clock, not the application's.** Twenty application servers have twenty slightly different clocks; a token bucket computing refill from a locally-read timestamp will drift, and a server whose clock is ahead grants extra tokens. Pass `redis.call('TIME')` inside the script, or accept the application clock only if you run NTP and tolerate the error.
- Redis `TIME` is non-deterministic, which historically made scripts non-replicable to replicas; modern Redis uses effect-based replication, so this is fine — but be aware if you are on a very old version.
- Clock **skew across regions** makes globally consistent limiting genuinely hard; usually the right answer is per-region limits summing to the global budget, rather than a single global counter with cross-region latency on every request.

**When Redis is down: fail open or fail closed?**

A deliberate decision, and the interviewer wants to hear it framed as a trade-off rather than a default:

- **Fail open** (allow requests when the limiter is unavailable) — availability over protection. Correct when the limiter exists for **fairness and cost control**, and where losing it briefly is survivable. This is the right default for most APIs: a Redis outage should not become a total outage.
- **Fail closed** (reject) — protection over availability. Correct when the limit protects something that will otherwise **break or cost real money**: a paid third-party API with a hard quota, a fragile legacy backend, or a security control such as login-attempt limiting, where failing open enables credential stuffing.

The mature answer is **both, by dimension**: fail open on general throughput limiting, fail closed on security-critical limits. And add a **local fallback** — an in-process limiter set to `global_limit / instance_count` that engages when Redis is unreachable, giving approximate protection rather than none. Also: set a very short Redis timeout (5–20ms) so a slow Redis does not add latency to every request (Q86), and put a circuit breaker on the limiter itself.

**The response: `429` plus headers**

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
RateLimit-Limit: 100
RateLimit-Remaining: 0
RateLimit-Reset: 30
Content-Type: application/problem+json

{ "type": "https://errors.acme.com/rate-limited", "title": "Rate limit exceeded",
  "status": 429, "code": "RATE_LIMITED", "retryable": true, "retryAfterSeconds": 30 }
```

- **`Retry-After`** is the important one — it tells a well-behaved client exactly when to retry, converting a retry storm into orderly backoff. Without it, clients guess, and they guess badly.
- **Return the `RateLimit-*` headers on *successful* responses too**, so clients can self-throttle *before* being rejected. This is what turns rate limiting from a punishment into a contract. The header names are being standardised by the IETF (`draft-ietf-httpapi-ratelimit-headers`); the `X-RateLimit-*` prefixed forms remain widely used — mention that the standard is still a draft rather than citing an RFC.
- Never return `503` for a rate limit; `429` is specific and tells clients this is their fault, not yours.
- Document the limits, and give customers visibility into their usage.

**Edge vs application enforcement**

- **At the edge** (API gateway, Kong, Envoy, NGINX, Cloudflare, AWS WAF): rejected requests never consume application resources at all — no thread, no connection, no JVM work. This is where the bulk of limiting belongs, and it is the only place that can absorb a genuine flood. Edge limiters typically use local counters with periodic synchronisation, trading exactness for speed.
- **In the application**: has the context the edge lacks — the authenticated principal, the tenant's plan tier, the actual cost of the operation, per-feature limits. Necessary for anything business-aware.

**Use both**: a coarse, generous limit at the edge for abuse and DDoS protection, and precise business limits in the application (or in a sidecar/mesh with a shared store). And recognise the limitation of both: rate limiting protects against *volume*, not against a single expensive query — for that you need timeouts, query cost limits, and load shedding (Q86).

### Probes

**Token bucket vs sliding window log vs sliding window counter.** All covered, plus fixed window's boundary-burst flaw and leaky bucket, with a defended default per use case.

**Redis + Lua atomicity.** Covered with a working script, the check-then-act race it prevents, `EVALSHA`, cluster hash tags, and Bucket4j as the library answer.

**Per-key / IP / tenant.** Covered as five dimensions, with the NAT and IPv6 caveats and cost-weighted requests.

**Clock issues.** Covered — use Redis's clock, drift across instances, and cross-region limits.

**Redis down → fail open vs closed.** Covered as an explicit trade-off decided per dimension, with a local fallback and a timeout/circuit breaker on the limiter.

**`429` + `Retry-After` + limit headers.** Covered, including returning headers on success and the draft status of the standard.

**Edge vs application enforcement.** Covered, with the recommendation to use both and the limits of rate limiting generally.

---

## Q99. Break a monolith into services — where do you cut first?

### Answer

**Start by asking whether to do it at all.** The strongest senior answer opens here, because the most common failure is not a bad seam — it is a migration that should never have started.

Microservices buy **independent deployability** and **independent scaling**, and they cost you distributed transactions, network failure modes, eventual consistency, distributed debugging, operational overhead per service, and a much higher bar for tooling. That trade is worth it when the pain is *organisational* — many teams blocked on one release train, one team's bug halting everyone's deploys, coordinated releases taking weeks. It is rarely worth it when the pain is technical and local. A well-modularised monolith deploys in minutes, has ACID transactions, and can be profiled in one process.

So: **a modular monolith first.** Enforce module boundaries in the same codebase (Spring Modulith, JPMS, or ArchUnit rules from Q79), with each module owning its tables and exposing an explicit API. You get most of the design benefit, you discover where the real seams are, and extraction later becomes mechanical. If you cannot maintain boundaries inside one process where the compiler can help you, you will not maintain them across a network where nothing can.

**Where to cut first — the criteria**

**By bounded context, not by layer.** The most common structural mistake is extracting a "data service", an "orchestration service", and a "UI service" — horizontal slices. Every feature then requires a change to all three, deployed in lockstep. You have added network calls and gained nothing: **a distributed monolith**. Cut vertically: a service owns its data, its logic, and its API for one business capability (orders, payments, inventory, notifications).

Identify contexts by looking for where the *language* changes. If "customer" means something different in billing than in support — different fields, different lifecycle, different rules — that is a context boundary. Event storming with domain experts is the standard technique.

**Choose the first candidate by these signals:**

1. **Low coupling** — few database joins to the rest of the system, few foreign keys crossing in or out. Measure this: query the schema for cross-module foreign keys and grep the code for cross-module imports. The seam with the fewest crossings is the cheapest cut.
2. **Genuinely different non-functional requirements** — a component that needs to scale 50× independently, or has a different availability requirement, or a different compliance boundary. This is where the benefit is real rather than theoretical.
3. **A different rate of change** — a module changed daily while the rest changes monthly, or vice versa.
4. **Clear team ownership** — Conway's law is not optional. Service boundaries that do not match team boundaries produce constant cross-team coordination, which is the exact cost you were trying to remove.
5. **Low risk if it fails** — pick something valuable enough to learn from but not the payment path. Notifications, reporting, search, file processing, and audit are classic first extractions because they are naturally asynchronous and often already loosely coupled.

**Strangler fig**

The pattern (Fowler's name, after the vine that grows around a tree until the tree dies and the vine stands alone): put a **façade** in front of the monolith — an API gateway, a reverse proxy, or a routing layer — and move functionality behind it one route at a time.

1. Introduce the façade with **all** traffic still routed to the monolith. Verify nothing changed. This step alone is valuable and low-risk.
2. Build the new service for one capability.
3. Route a percentage of that capability's traffic to the new service; keep the monolith path live.
4. Optionally run **parallel-run / shadow traffic**: send requests to both, serve the monolith's response, and compare the new service's output asynchronously. This finds behavioural differences with zero user risk and is the single most effective de-risking technique for a migration where the legacy behaviour is poorly documented (which it always is).
5. Shift to 100%, monitor, then delete the monolith's code path.

The essential properties: **incremental, reversible at every step, and delivering value continuously**. Contrast with the big-bang rewrite, which delivers nothing for eighteen months and then fails — the "second-system effect".

**Database decomposition is the hard part**

Splitting code is straightforward; splitting data is the actual project. In the monolith, modules share a schema, so joins and foreign keys tie everything together.

The sequence:
1. **Logically separate first, physically second.** Within the monolith, stop cross-module joins: each module accesses only its own tables, and other modules' data comes through a method call. Enforce with schema-per-module and separate database users with restricted grants — the database itself then rejects a cross-module query. This surfaces every hidden dependency while you still have transactions and a single deployment to fix them.
2. **Break foreign keys** that cross the boundary, replacing referential integrity with an ID reference plus application-level validation and a reconciliation check. This is a genuine loss and must be an accepted decision.
3. **Replace joins** with either an API call (for small, latency-tolerant lookups), data duplication kept current by events (for read-heavy paths — the receiving service stores the fields it needs), or a CQRS read model (Q78).
4. **Move the tables** to a separate database, typically via CDC replication then a cutover.
5. **Replace cross-module transactions** with sagas (Q77) and the outbox (Q76). Identify these *before* cutting — a transaction spanning the proposed boundary is a strong signal that the boundary is wrong.

**The shared-database anti-pattern.** Two services reading and writing the same tables is the worst outcome: neither can change its schema without coordinating, so you have all the operational cost of distribution and none of the independence. It also destroys encapsulation — the other service depends on your *internal* representation, permanently. It is common precisely because it is the easy intermediate step, and teams stop there. **A read-only replica or a published read model is acceptable; shared write access is not.** If you must pass through this state, put a deadline and an owner on it.

**Anti-corruption layer**

When the new service must integrate with the legacy monolith, do **not** let the monolith's model leak into it. An ACL is a translation layer at the boundary that maps the legacy model onto the new service's own domain model.

Without it, the new service inherits the legacy's accumulated compromises — the nullable columns nobody can remove, the status enum with eleven values that mean three things, the field named `flag_2`. Within a release you have a new service that is just the old design with network latency. The ACL is where you write the ugly translation, deliberately and in one place, so it can be deleted when the legacy is retired.

It works in both directions: the monolith calling the new service should also go through an adapter, so the monolith's expectations do not dictate the new API.

**Distributed monolith — warning signs**

Name these explicitly; recognising them is the core skill:

- **Services must be deployed together**, or in a specific order. The definitive symptom.
- **A change to one feature requires changes to three services** — the boundaries are horizontal, not vertical.
- **Synchronous call chains** — A calls B calls C calls D within one request. Latency compounds, availability multiplies (four services at 99.9% each gives 99.6%), and any one failing fails everything.
- **Shared database tables** across services.
- **A shared library containing domain models**, so bumping its version forces a coordinated release of everyone. (A shared library of *technical* utilities is fine; shared *domain* types are not.)
- **Distributed transactions** or the need for them.
- **A single team owns many services** with no independent release cadence — the split gave organisational nothing.
- **You cannot test or run a service alone** without spinning up five others.

**Measuring whether it worked**

The migration must be justified by outcomes, and the metric must be chosen up front:

- **Deployment independence** — the fraction of deployments involving exactly one service. This is the direct measure of the primary benefit. If it is not rising, the migration is failing.
- **DORA metrics** — deployment frequency, lead time for change, change failure rate, and mean time to restore. If lead time has not improved, you have paid the cost without the benefit.
- **Blast radius** — incidents affecting one service versus many.
- **Cross-team coordination** — how many teams must be involved in a typical change.
- Watch the counter-indicators too: total infrastructure cost, p99 latency (network hops add up), on-call load, and the number of incidents whose root cause is distribution itself.

**When not to do it — say this explicitly.** A small team (under roughly 15–20 engineers) almost certainly should not: you will spend more time on infrastructure than on product, and one team can coordinate a monolith trivially. A system with genuinely high transactional coupling (a core banking ledger) fights the architecture at every step. An unstable domain should not have boundaries frozen into network APIs before you know where they are. And "the monolith is slow" is usually a database, caching, or algorithmic problem that distribution will make *worse*, not better — profile first. Extracting one or two services for specific, measured reasons and leaving the rest as a well-structured monolith is frequently the correct end state, not a failure to finish.

### Probes

**Strangler fig.** Covered, with the five-step sequence and parallel-run/shadow traffic as the key de-risking technique.

**Seams by bounded context, not by layer.** Covered, with how to identify contexts and five criteria for choosing the first cut.

**Database decomposition as the hard part.** Covered as a five-step sequence, including logical separation before physical.

**Shared-DB anti-pattern.** Covered, with the distinction between an acceptable read replica and unacceptable shared writes.

**Anti-corruption layer.** Covered — its purpose, why it matters in both directions, and that it is meant to be deleted.

**Distributed monolith warning signs.** Covered as an explicit checklist.

**Measuring deployment independence.** Covered, with DORA metrics and counter-indicators.

**When not to.** Covered as a substantive position, including the modular-monolith-first recommendation.

---

## Q100. Design an idempotent payment flow across your service, your DB, and a third-party PSP.

### Answer

**The failure that defines the problem.** Your service calls the PSP. The PSP charges the card. The response is lost — a timeout, a pod eviction, a network partition. Your service does not know whether the money moved. Retry and you may double-charge a customer. Do not retry and you may lose a payment you believe succeeded. Neither is acceptable, and **you cannot distinguish the two cases from your side.** Everything below exists to make that state recoverable.

**End-to-end idempotency key**

One key flows through the whole chain:

1. **The client** generates a key when the user presses "Pay" — once per *logical payment attempt*, reused across every HTTP retry of that attempt (Q56). A key generated per HTTP call defeats the entire mechanism.
2. **Your API** stores it with a unique constraint. A repeat returns the original result rather than starting a second payment.
3. **Your service** derives a **deterministic** PSP idempotency key from your own payment ID — `"pay_" + paymentId` — so that every retry of the PSP call, from any pod, in any process, sends the *same* key. This is the critical detail: the key must be derived, never randomly generated at call time, or a retry after a crash generates a new key and the PSP sees a second, distinct payment.
4. **The PSP** deduplicates on it and returns the original charge.

Stripe, Adyen, Braintree, and every serious PSP support this. If one does not, you must instead **query before retrying** (search their API by your reference) — slower, racier, and a reason to prefer a different provider.

**The payment state machine**

Model states explicitly; do not infer them from flags.

```
CREATED → AUTHORIZING → AUTHORIZED → CAPTURING → CAPTURED
             │              │                        │
             ├──────────────┴─▶ FAILED               ├─▶ REFUNDED / PARTIALLY_REFUNDED
             │                                        │
             └─▶ UNKNOWN ──(reconciliation)──▶ AUTHORIZED | FAILED
                                              CANCELLED / EXPIRED
```

Rules:
- Every transition is persisted **before** and **after** the external call.
- Transitions are **guarded and atomic**: `UPDATE payments SET status='CAPTURING' WHERE id=? AND status='AUTHORIZED'`. Zero rows affected means someone else already moved it — abort rather than proceed. The database, not application logic, arbitrates.
- **`UNKNOWN` is a first-class state**, not an error. It is what you record when a PSP call times out. It means "the money may or may not have moved; a human or a reconciliation job must resolve it". Systems that lack this state paper over the ambiguity and lose money.
- Terminal states never transition.
- Store the PSP's reference (`charge_id`), the idempotency key, the attempt count, and a full audit trail of transitions with timestamps.

**Crash after the PSP call but before the commit — the core scenario**

```java
payment.markAuthorizing();  repository.save(payment);   // COMMITTED first
var result = psp.authorize(request, idempotencyKey);    // ← crash here
payment.markAuthorized(result.chargeId);  repository.save(payment);  // never runs
```

State on recovery: PSP has an authorised charge; your database says `AUTHORIZING`. Recovery works **because** you committed `AUTHORIZING` before the call:

1. A **recovery job** finds payments stuck in `AUTHORIZING`/`CAPTURING` beyond a threshold (say 2 minutes).
2. For each, it **queries the PSP by the idempotency key** (or re-sends the same request — idempotency makes that safe, and it returns the original result).
3. The PSP's answer is authoritative: charge exists → transition to `AUTHORIZED`; no charge → transition to `FAILED` and allow a fresh attempt.

This is why the ordering matters absolutely: **persist intent before acting, and act idempotently.** If you called the PSP first and crashed, you would have a charge with no local record, no idempotency key, and no way to find it except a full reconciliation. Never call an external money-moving API without a committed local record of the attempt.

Related: **never hold a database transaction open across the PSP call** (Q28/Q86). A 5-second PSP call inside a transaction holds locks and a connection, and a rollback would erase the very record you need. Commit the intent, close the transaction, make the call, then open a new transaction to record the outcome.

**Webhook handling — out-of-order, duplicate, signed**

PSPs report asynchronously (3-D Secure completion, capture settlement, disputes, refunds). Handling this correctly is where most implementations are weak.

- **Verify the signature** on every webhook — HMAC over the **raw body** (not the re-serialised JSON, which may differ byte-for-byte) plus a timestamp, compared with a **constant-time** comparison. Reject if the timestamp is outside a small window, which prevents **replay** of a captured, validly-signed request. An unverified webhook endpoint is an unauthenticated "mark this payment as paid" API, and it has been exploited in the wild.
- **Expect duplicates.** PSPs retry until they get a `2xx`. Dedupe on the provider's event ID, with the dedupe record committed in the same transaction as the effect (Q83).
- **Expect out-of-order delivery.** A `charge.captured` webhook can arrive before `charge.authorized`. So apply **monotonic state transitions only** — a webhook for an earlier state must not overwrite a later one. Guard with the same conditional `UPDATE ... WHERE status = expected`, and drop (with a log) anything that would move backwards. Where the PSP provides a sequence number or event timestamp, compare it and ignore stale events.
- **Return `2xx` fast.** Verify, persist the raw event, return `200`, and process asynchronously. Doing the work inline risks a PSP timeout and an unnecessary retry storm — and some PSPs disable endpoints that are persistently slow.
- **Never trust the webhook payload's amount** without verifying it against your own record. Treat webhooks as a *notification that something changed*, and where the value matters, fetch authoritative state from the PSP's API.
- Webhooks can be **lost entirely**. They are an optimisation for latency, not a guarantee — which is why reconciliation is mandatory.

**Reconciliation — the actual guarantee**

Everything above narrows the window; reconciliation closes it. Run at least daily:

1. Fetch the PSP's settlement report / transaction list for the period.
2. Match every PSP transaction to a local payment by reference.
3. Investigate every mismatch:
   - **PSP has a charge, you have none** → money taken with no order. Refund or complete the order; always alert.
   - **You show `CAPTURED`, PSP has nothing** → a bug or a lost record; you have delivered goods without payment.
   - **Amount mismatch** → currency, rounding, partial capture, or fee handling.
   - **Stuck non-terminal states** older than the threshold → resolve by querying the PSP.
4. Anything unresolved goes to a **manual review queue** with enough context for a finance operator, and is alerted on.

Reconciliation is not optional in a payment system. It is the control that catches everything the happy path, the retries, and the webhooks missed — and auditors will ask for it. Track "unreconciled items" and "oldest unreconciled item" as monitored metrics.

**Money must never be in two places**

Practical rules that follow:

- **The PSP is the source of truth for whether money moved.** Your database records your *belief*; when they disagree, the PSP wins and you investigate.
- **Use a double-entry ledger** for internal accounting — every movement is two balanced entries, so the books provably balance and an inconsistency is detectable by construction rather than by inspection. Append-only; corrections are new compensating entries, never updates.
- **Never store money as `double`.** Use `BigDecimal` with explicit scale, or integer minor units (cents). `0.1 + 0.2 != 0.3` in binary floating point, and rounding errors in a ledger are unrecoverable.
- **Always store the currency** with the amount, and never sum across currencies.
- **Separate authorisation from capture.** Authorise at order placement, capture at fulfilment. It reduces refunds, matches most regulatory and card-scheme expectations, and gives a clean cancellation path. Be aware authorisations expire (typically 7 days).
- **Refunds are compensating transactions, not rollbacks** (Q77) — a new forward movement, recorded as such, with its own idempotency key.
- **Make delivery of value conditional on the payment state**, and use the outbox (Q76) so "payment captured" and "order marked paid" cannot diverge.
- **Immutability and audit**: never update a payment row destructively. Keep every state transition, every PSP request/response (with card data redacted), and every webhook, so any dispute can be reconstructed. This is a compliance requirement, not just good practice.
- **PCI scope**: never let raw card data touch your servers. Use the PSP's hosted fields or tokenisation so you handle only tokens — this is a design decision that removes an enormous compliance burden, and stating it shows real payments experience.

**Testing this.** Fault injection is mandatory: simulate timeouts, duplicate webhooks, out-of-order webhooks, replayed webhooks, and a crash between the PSP call and the commit. Most PSPs provide a sandbox with triggerable failure scenarios. A payment flow that has not been tested against a lost response is untested where it matters most.

### Probes

**End-to-end idempotency key.** Covered through all four hops, with the critical point that the PSP key must be *derived deterministically*, not generated per call.

**Payment state machine.** Covered, including guarded atomic transitions and `UNKNOWN` as a first-class state.

**PSP webhook handling (out-of-order, duplicate, replay-signed).** Covered in full, including raw-body HMAC, constant-time comparison, timestamp windows, monotonic transitions, fast `2xx`, and not trusting payload values.

**Reconciliation job.** Covered as the actual guarantee, with the four mismatch classes and the manual review queue.

**Crash after the PSP call but before the commit.** Covered as the core scenario, with the recovery procedure and the persist-intent-before-acting rule that makes recovery possible.

**Money never in two places.** Covered — double-entry ledger, `BigDecimal`/minor units, auth vs capture, refunds as compensation, immutability, and PCI tokenisation.

---

# 15. Behavioural / Experience

These have no model answer — the content must be **yours**. What follows is the structure interviewers are assessing against, and the specific traps for each prompt. Prepare three or four real stories in advance and reuse them across questions; you do not need a distinct story per prompt.

**Use STAR, but weight it correctly.** Situation and Task should be brief — two or three sentences of context. **Action** is where 60% of your time goes, and it must be *your* actions in the first person singular. Result must be concrete and, where possible, quantified. A very common failure is spending three minutes on context and thirty seconds on what you actually did.

The two additions that distinguish senior answers: say what you **considered and rejected**, and say what you **learned or would change**. Interviewers are calibrating judgement and self-awareness, not the outcome — a well-reasoned decision with a poor result often scores higher than a lucky one.

**Say "I" for your actions and "we" for team context.** Claiming sole credit for team work reads badly; hiding behind "we" makes it impossible to assess you. Be precise about which parts were yours.

---

**"Tell me about the worst production incident you were part of."**

Structure: what broke and its user impact → how it was detected → how you diagnosed it → the mitigation versus the fix → the root cause → what changed afterwards.

What they are assessing: whether you can diagnose under pressure, whether you distinguish *stopping the bleeding* from *fixing the cause*, and whether you turned the incident into durable improvement.

Include: the actual timeline, the user impact in real terms ("payments failed for 40 minutes, roughly 3,000 customers"), the hypotheses you formed and eliminated, and the *specific* follow-up work. "We added an alert on connection-pool saturation and made the liveness probe stop checking the database" is worth more than "we improved monitoring".

Traps: blaming an individual (name the systemic cause instead — a system that lets one person break production is the problem); claiming you personally saved everything; having no root cause; or having no follow-up. If you did not lead the response, say what your part was honestly — "I owned the database investigation while X coordinated" is a fine answer.

---

**"Tell me about a technical decision you'd make differently."**

Choose something **real and consequential** — an architecture, a technology adoption, a data model — not a token confession. "I should have written more tests" is not an answer.

Structure: the decision, the context and constraints at the time, why it was reasonable *then*, what it actually cost, what you would do instead, and how you now recognise that situation earlier.

The key move: **justify the original decision on the information available**, then explain what new information changed it. That demonstrates judgement rather than hindsight. Also worth saying whether you fixed it or lived with it — deciding *not* to fix something is itself a senior decision, and explaining why can be stronger than describing a heroic rewrite.

Trap: a humblebrag ("I was too thorough"). It reads as evasion and interviewers notice immediately.

---

**"How did you convince a team to adopt or reject a technology?"**

The word "convince" is the test: they want influence without authority, not a decree.

Structure: the problem (not the technology — always start from the problem), how you evaluated options, how you built the case, how you handled disagreement, and the outcome including what it cost.

Strong material: a **spike or prototype** rather than an argument; a written comparison with explicit criteria weighted for *your* context; a **reversible first step** (a pilot on one non-critical service) that let sceptics see evidence rather than take your word; and taking the strongest counter-argument seriously in public.

**A rejection story is often stronger than an adoption story** — talking a team out of Kubernetes, event sourcing, or a rewrite they were excited about shows judgement, courage, and an understanding of cost. If you have one, use it.

Trap: describing consensus you did not actually build, or a technology you adopted because it was interesting. Be ready for "what did it cost, and what went wrong?" — every adoption has both.

---

**"You inherit a service with no tests and poor structure. What do you do in the first month?"**

They are assessing prioritisation and restraint, not enthusiasm for refactoring.

A defensible sequence:
- **Week 1 — understand and stabilise.** Read the code and the incident history; talk to whoever operated it and to its consumers. Establish observability first if it is missing (Q88): you cannot improve what you cannot see. Identify the actual pain — is it defects, deploy risk, performance, or onboarding time?
- **Week 2 — build a safety net at the boundary.** Characterisation tests (Feathers) that capture *current* behaviour, bugs included, at the API level. High-level tests before unit tests, because you do not yet know what the units should be, and because API-level tests survive refactoring while unit tests of bad structure do not. Get the build and deployment reliable.
- **Weeks 3–4 — refactor where you are already working.** Improve the code you must touch for real work; do not open a refactoring project. Add unit tests as you extract seams. Fix the highest-pain item — usually the deployment process or the one module causing most incidents.
- **Throughout — document as you learn**, and agree the plan and its trade-offs with your manager and the team.

The key positions to state: **no big-bang rewrite** (you do not yet know what the code does, and the rewrite will reproduce the bugs without the accumulated fixes); **do not refactor without tests** — the boundary tests come first; **the business keeps getting features** while you improve incrementally; and **the ugliest code is not the priority** — the code that *changes most often* and *breaks most often* is. Measure both and target them.

---

**"Describe a time you disagreed with a senior engineer or architect."**

They are assessing whether you can disagree constructively, and whether you can commit after losing.

Structure: the disagreement and why it mattered → how you raised it → what evidence you brought → the outcome → what you did afterwards, **especially if you lost**.

Strong signals: you sought to understand their reasoning first (they usually have context you lack); you argued about the decision, not the person; you brought data or a prototype rather than opinion; you escalated appropriately if the stakes justified it; and — most important — you **disagreed and committed**, executing the chosen path properly rather than sabotaging it or saying "I told you so" later.

Trap: only telling stories where you were right. A story where you were **wrong**, changed your mind on evidence, and said so is a very strong answer and many candidates never offer one.

---

**"What's a belief you hold about software that most engineers you've worked with disagree with?"**

An unusual question testing whether you have independent, examined views and can defend one under challenge.

It must be a **genuine** position you can argue with real examples, and it must be defensible rather than merely contrarian. Expect immediate pushback — that is the point, and the interviewer is assessing how you handle it.

Examples of positions with real substance behind them (pick your own, and only if you actually believe it): that most microservice migrations should have stopped at a modular monolith; that DRY is over-applied and duplication is often cheaper than the wrong abstraction; that code coverage targets are actively harmful (Q94); that most caching is added before the problem is measured; that comments explaining *why* are undervalued while comments explaining *what* are correctly maligned; that "we'll need it later" is almost always wrong (Q73); that the ORM is the wrong default for read-heavy systems (Q46); or that on-call responsibility should sit with the team that writes the code and this changes design behaviour more than any review process.

Traps: a bland non-opinion ("testing is important"); a position you cannot support with an example; or a genuinely unreasonable take defended by stubbornness. Show that you hold it *provisionally* — say what evidence would change your mind. That is what separates a considered view from dogma.

---

**One general note.** Interviewers can tell rehearsed stories from real ones, and the difference is specificity: real stories have names of technologies, actual numbers, dead ends, and things that went wrong. Prepare the *structure* in advance, not the wording, and be ready for follow-ups two and three levels deep — "why did you choose that?", "what would have happened if you hadn't?", "what did the team think?". The follow-ups are where the assessment actually happens.
