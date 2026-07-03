---
theme: seriph
title: 'PostgreSQL Indexes — From Internals to Strategy'
info: |
  A deep dive into how PostgreSQL indexes work, what they cost,
  and how to use them to optimize performance and storage.
class: text-center
highlighter: shiki
drawings:
  persist: false
transition: slide-left
mdc: true
---

# PostgreSQL Indexes

**From Internals to Strategy**

<div class="pt-12 text-gray-400">
  How they work · What they cost · When to use them
</div>

<!--
Aurel
-->

---
layout: center
class: text-center
---

# What is a database management system?

<div class="text-gray-400 text-base">An abstraction between a client and the file system</div>

<div class="mt-4 flex flex-col items-center gap-0 text-sm">

  <div v-click class="w-72 px-5 py-3 bg-blue-900/50 border-2 border-blue-400 rounded-xl text-center">
    <div class="text-sm font-bold text-blue-300">Client / Application</div>
    <div class="text-gray-400 text-xs mt-0.5">Developer · ORM · BI tool</div>
  </div>

  <div v-click class="flex flex-col items-center">
    <div class="w-px h-3 bg-gray-500"></div>
    <div class="px-3 py-0.5 bg-gray-800 border border-gray-500 rounded-full text-xs text-gray-300 font-mono">
      SELECT * FROM orders WHERE status = 'pending'
    </div>
    <div class="text-gray-500 leading-4">↓</div>
  </div>

  <div v-click class="w-72 px-5 py-3 bg-green-900/50 border-2 border-green-400 rounded-xl text-center">
    <div class="text-sm font-bold text-green-300">PostgreSQL DBMS</div>
    <div class="flex justify-center gap-3 mt-2 text-xs text-gray-400">
      <span>Parse</span>
      <span class="text-gray-600">·</span>
      <span>Plan</span>
      <span class="text-gray-600">·</span>
      <span>Optimize</span>
      <span class="text-gray-600">·</span>
      <span>Execute</span>
    </div>
    <div v-click class="mt-2 text-xs text-green-400 border border-green-800 rounded px-3 py-0.5">
      Hides storage format &amp; I/O complexity
    </div>
  </div>

  <div v-click class="flex flex-col items-center">
    <div class="w-px h-3 bg-gray-500"></div>
    <div class="px-3 py-0.5 bg-gray-800 border border-gray-500 rounded-full text-xs text-gray-300 font-mono">
      read page 42 · write WAL · fetch tuple (3, 7)
    </div>
    <div class="text-gray-500 leading-4">↓</div>
  </div>

  <div v-click class="w-72 px-5 py-3 bg-orange-900/50 border-2 border-orange-400 rounded-xl text-center">
    <div class="text-sm font-bold text-orange-300">File System</div>
    <div class="text-gray-400 text-xs mt-0.5">Heap files · Indexes · WAL</div>
  </div>

</div>

---
layout: center
class: text-center
---

# Before indexes...

<div v-click class="mt-8 text-2xl">
  Where does data actually live?
</div>

---
layout: two-cols
---

# Data lives on disk

PostgreSQL stores all data in **heap files** on the filesystem.

A heap file is made of **8 KB pages**.  
Each page holds several **tuples** (rows).

<div class="mt-6">

When you run a query, Postgres needs to find the matching rows.

</div>

<div v-click class="mt-6 p-4 bg-red-900/30 border border-red-500 rounded-lg">

**Without an index:**  
Postgres reads every single page — a **sequential scan**.  
On a 10M-row table, that means reading gigabytes of data.

</div>

::right::

<div class="ml-8 mt-4">

```
┌─────────────────────────┐
│       Heap File         │
├─────────────────────────┤
│  Page 0  (8 KB)         │
│  ┌──────┬──────┬──────┐ │
│  │ row1 │ row2 │ row3 │ │
│  └──────┴──────┴──────┘ │
├─────────────────────────┤
│  Page 1  (8 KB)         │
│  ┌──────┬──────┬──────┐ │
│  │ row4 │ row5 │ row6 │ │
│  └──────┴──────┴──────┘ │
├─────────────────────────┤
│  Page 2  (8 KB)         │
│  ┌──────┬──────┬──────┐ │
│  │ row7 │ row8 │ row9 │ │
│  └──────┴──────┴──────┘ │
└─────────────────────────┘
```

</div>

---
layout: center
class: text-center
---

# An index is a shortcut

<div class="text-xl text-gray-300 mt-4">
A separate data structure, stored alongside the heap,<br>
that lets Postgres find rows <strong>without reading everything</strong>.
</div>

<div v-click class="mt-10 text-5xl">
📖 → 🗂️
</div>

<div v-click class="mt-6 text-gray-400">
Like the index at the back of a book:<br>
you look up the term, get a page number, jump directly there.
</div>

---
layout: section
---

# Part 1

## Index Families

*Where does the data live relative to the index?*

---
layout: three-cols
---

# Three families of indexes

::left::

<div class="text-center">

### 🗜️ Clustered

<div class="mt-4 text-sm">

Data **lives inside** the index itself.

The table rows are physically stored in index order.

</div>

<div class="mt-4 p-3 bg-blue-900/30 border border-blue-500 rounded text-xs">

MySQL InnoDB,  
SQL Server

</div>

<div class="mt-4 text-xs text-gray-400">

One lookup → you already have the data.  
No extra hop needed.

</div>

</div>

::middle::

<div class="text-center">

### 🔗 Non-Clustered

<div class="mt-4 text-sm">

The index holds a **pointer** (ctid) to the row in the heap file.

</div>

<div class="mt-4 p-3 bg-green-900/30 border border-green-500 rounded text-xs">

**PostgreSQL**  
(always)

</div>

<div class="mt-4 text-xs text-gray-400">

Two steps: traverse index,  
then fetch row from heap.

</div>

</div>

::right::

<div class="text-center">

### 📦 Covering

<div class="mt-4 text-sm">

A **subset of columns** is embedded in the index. No heap access needed for those columns.

</div>

<div class="mt-4 p-3 bg-purple-900/30 border border-purple-500 rounded text-xs">

DynamoDB, and  
`INCLUDE` in Postgres

</div>

<div class="mt-4 text-xs text-gray-400">

Best of both worlds  
for read-heavy queries.

</div>

</div>

---
layout: two-cols
---

# PostgreSQL: always non-clustered

The index and the heap are **two separate structures**.

<div class="mt-6">

```
Index (B-Tree)          Heap File
───────────────         ────────────────
│ "Alice" → ──────────▶ │ Page 3, row 2 │
│ "Bob"   → ──────────▶ │ Page 1, row 0 │
│ "Carol" → ──────────▶ │ Page 7, row 5 │
│ "Dave"  → ──────────▶ │ Page 1, row 3 │
───────────────         ────────────────
```

</div>

<div v-click class="mt-6 p-4 bg-yellow-900/30 border border-yellow-500 rounded-lg text-sm">

⚠️ **Consequence**: even after finding a match in the index, Postgres still needs to fetch the actual row from the heap — a **heap fetch** (also called a *random I/O*).

</div>

::right::

<div class="ml-8">

### The `INCLUDE` trick

You can embed extra columns in the index leaf nodes to avoid the heap fetch entirely:

```sql
CREATE INDEX idx_users_email
  ON users(email)
  INCLUDE (name, avatar_url);
```

```sql
-- This query never touches the heap:
SELECT name, avatar_url
FROM users
WHERE email = 'alice@example.com';
```

<div v-click class="mt-4 p-3 bg-green-900/30 border border-green-500 rounded text-sm">

✅ **Index-only scan** — no heap access needed.

</div>

</div>

---
layout: section
---

# Part 2

## How Indexes Work

*The data structures under the hood*

<!--
Glenn
-->

---
layout: center
class: text-center
---

# PostgreSQL index types

<div class="mt-8 grid grid-cols-4 gap-4 text-sm">

<div v-click class="p-4 bg-blue-900/40 border border-blue-400 rounded-lg">
  <div class="text-2xl mb-2">🌳</div>
  <div class="font-bold">B-Tree</div>
  <div class="text-gray-400 mt-1">Default. Handles =, <, >, BETWEEN, LIKE 'foo%'</div>
</div>

<div v-click class="p-4 bg-green-900/40 border border-green-400 rounded-lg">
  <div class="text-2xl mb-2">#️⃣</div>
  <div class="font-bold">Hash</div>
  <div class="text-gray-400 mt-1">Equality only (=). O(1) lookup.</div>
</div>

<div v-click class="p-4 bg-purple-900/40 border border-purple-400 rounded-lg">
  <div class="text-2xl mb-2">🔍</div>
  <div class="font-bold">GIN</div>
  <div class="text-gray-400 mt-1">Arrays, JSONB, full-text search</div>
</div>

<div v-click class="p-4 bg-orange-900/40 border border-orange-400 rounded-lg">
  <div class="text-2xl mb-2">📐</div>
  <div class="font-bold">GiST</div>
  <div class="text-gray-400 mt-1">Geometry, ranges, nearest-neighbor</div>
</div>

<div v-click class="p-4 bg-red-900/40 border border-red-400 rounded-lg">
  <div class="text-2xl mb-2">🧱</div>
  <div class="font-bold">BRIN</div>
  <div class="text-gray-400 mt-1">Very compact. Naturally ordered data (timestamps, serial IDs)</div>
</div>

<div v-click class="p-4 bg-yellow-900/40 border border-yellow-400 rounded-lg">
  <div class="text-2xl mb-2">🌸</div>
  <div class="font-bold">Bloom</div>
  <div class="text-gray-400 mt-1">Probabilistic. Multi-column equality filters</div>
</div>

<div v-click class="p-4 bg-pink-900/40 border border-pink-400 rounded-lg">
  <div class="text-2xl mb-2">🗺️</div>
  <div class="font-bold">Bitmap</div>
  <div class="text-gray-400 mt-1">Low cardinality. Combinable at query time</div>
</div>

</div>

---

# Hash Index — lookup in 3 steps

<div class="mt-2 grid grid-cols-2 gap-8">

<div>

<div v-click class="flex items-start gap-3 mb-3">
  <span class="text-blue-400 font-bold mt-0.5">①</span>
  <div class="text-sm">
    <span class="text-blue-300 font-semibold">Hash the value</span> — a type-specific function produces a 32-bit code<br>
    <code class="text-xs">"alice" → 0x3A2F</code>
  </div>
</div>

<div v-click class="flex items-start gap-3 mb-3">
  <span class="text-yellow-400 font-bold mt-0.5">②</span>
  <div class="text-sm">
    <span class="text-yellow-300 font-semibold">Find the bucket</span> — <code class="text-xs">hash % num_buckets</code> → bucket 7<br>
    <span class="text-gray-400 text-xs">The <strong>metapage</strong> (page 0 of the index) maps bucket 7 → disk page 42</span>
  </div>
</div>

<div v-click class="flex items-start gap-3 mb-4">
  <span class="text-green-400 font-bold mt-0.5">③</span>
  <div class="text-sm">
    <span class="text-green-300 font-semibold">Scan the bucket page</span> — entries are <code class="text-xs">(hash_code, ctid)</code> pairs<br>
    <span class="text-gray-400 text-xs">Match hash → fetch heap row via ctid → verify value (handles collisions)</span>
  </div>
</div>

<div v-click class="p-3 bg-red-900/30 border border-red-500 rounded text-xs">

❌ **Cannot do:**  
```sql
WHERE age > 30          -- no ordering
WHERE name LIKE 'Al%'   -- no prefix
ORDER BY name           -- no sort
```

</div>

</div>

<div v-click>

```
  "alice"
     │ hash fn
     ▼
  0x3A2F
     │ 0x3A2F % num_buckets
     ▼
  bucket 7
     │ metapage: bucket 7 → page 42
     ▼
  ┌─────────────────────────┐
  │  Page 42 (bucket page)  │
  │  (0x3A2F, page 3 row 2) │◀── match → heap fetch
  │  (0x3A2F, page 9 row 1) │◀── collision: verify value
  │  (0x1109, page 5 row 0) │    skip (different hash)
  └─────────────────────────┘
```

<div class="mt-3 p-3 bg-blue-900/30 border border-blue-400 rounded text-xs">

💡 The hash code is stored alongside the ctid so Postgres can skip non-matching entries without a heap fetch. Only hash collisions trigger an extra heap read to verify the real value.

</div>

</div>

</div>

<!--
hash la value => clé de hash % nombre de bucket => bucket ou sera stocké la value.

plusieurs valeurs dans le même bucket => collision => Posgres stock les 2 valeurs dans le même bucket et compare les valeurs réelles : assez rare
-->

---
layout: center
---

# B-Tree — the workhorse

> Used by default for every `CREATE INDEX`. Handles equality, ranges, ordering, and prefix matching.

---
layout: two-cols
---

# B-Tree: the structure

A B-Tree is made of **pages** (nodes). Each page stores sorted boundary keys.

- **Internal nodes** → keys + pointers to child pages
- **Leaf nodes** → keys + pointers to heap rows (ctid)

The value you're looking for must fall **between two boundary keys**.

<div v-click class="mt-4 p-4 bg-blue-900/30 border border-blue-400 rounded text-sm">

A typical Postgres B-Tree has a **branching factor** of ~100–400 keys per page.

A table with 1 billion rows needs only **~5 levels** to traverse.

</div>

::right::

<div class="ml-6 font-mono text-sm">

```
              [  30  |  70  ]          ← Root (internal)
             /        |        \
    [ 10|20 ]      [40|50|60]    [80|90]   ← Internal
    /   |   \          |              \
[1..9][11..29][31..39][41..69]    [71..99]  ← Leaves
  ↓      ↓      ↓       ↓            ↓
 heap   heap   heap    heap          heap
```

</div>

<div class="ml-6 mt-4 text-sm text-gray-400">

Looking for `55`:  
→ Root: `55 > 30`, `55 < 70` → middle child  
→ Internal: `55 > 50`, `55 < 60` → go right  
→ Leaf: found! → fetch from heap

</div>

---

# B-Tree: reading — a traversal

<div class="grid grid-cols-2 gap-8">

<div>

Looking up `WHERE age = 55`:

<div v-click>

**Step 1** — Start at the root.  
Compare `55` against root keys.  
→ `30 < 55 < 70` → follow middle pointer.

</div>

<div v-click class="mt-4">

**Step 2** — Arrive at internal node `[40|50|60]`.  
→ `50 < 55 < 60` → follow right pointer.

</div>

<div v-click class="mt-4">

**Step 3** — Arrive at leaf node.  
→ Find `55` → read `ctid` → fetch row from heap.

</div>

<div v-click class="mt-4 p-3 bg-blue-900/30 border border-blue-400 rounded text-sm">

Total: **~3–5 page reads** regardless of table size.  
That's O(log n).

</div>

</div>

<div>

```
Root:     [  30  |  70  ]
              ↓
Internal: [ 40 | 50 | 60 ]
                       ↓
Leaf:     [ 51 | 53 | 55 | 58 ]
                        ↓
Heap:     Page 4, row 2 → { age: 55, name: "Bob" }
```

</div>

</div>

<!--
Demo live avec l'outil
-->

---

# B-Tree: range queries — the leaf chain

Leaf nodes are **linked together** in a doubly linked list.

This is what makes range queries efficient.

<div class="mt-6 font-mono text-sm">

```
  ← ─────────────────────────────────────────────── →
  [1..9] ↔ [10..20] ↔ [21..35] ↔ [36..50] ↔ [51..70] ↔ [71..99]
```

</div>

<div v-click class="mt-6">

```sql
WHERE age BETWEEN 25 AND 60
```

1. Traverse from root → find leaf containing `25`
2. Scan **right along the leaf chain** until `60`
3. Fetch matching rows from heap

</div>

<div v-click class="mt-4 p-4 bg-green-900/30 border border-green-500 rounded text-sm">

✅ This also makes `ORDER BY` free when using a B-Tree index — the data is already sorted.

</div>

---

# B-Tree: writes — insertions and splits

<div class="grid grid-cols-2 gap-8">

<div>

**Inserting a new value:**

<div v-click>

1. Traverse the tree to find the correct leaf page.

</div>

<div v-click class="mt-3">

2. If the leaf has space → insert in sorted order. Done.

</div>

<div v-click class="mt-3">

3. If the leaf is **full** → **page split**:  
   - Create a new sibling page  
   - Move half the keys to the new page  
   - Propagate a new key up to the parent

</div>

<div v-click class="mt-3">

4. If the parent is also full → split propagates upward.  
   In rare cases, the **root splits** and the tree grows a level.

</div>

</div>

<div>

```
Before insert (55):
Leaf: [ 51 | 53 | 57 | 59 ]  ← full

After split:
Leaf A: [ 51 | 53 ]
Leaf B: [ 55 | 57 | 59 ]
             ↑
         pushed up to parent
```

<div v-click class="mt-6 p-3 bg-yellow-900/30 border border-yellow-500 rounded text-sm">

⚠️ Page splits are why **write-heavy tables** accumulate index bloat over time.  
`VACUUM` and `REINDEX` help reclaim space.

</div>

</div>

</div>

---

# B-Tree: crash safety — the WAL

What happens if the server crashes mid-split?

<div class="grid grid-cols-2 gap-8 mt-4">

<div>

<div v-click>

A page split modifies **multiple pages simultaneously**.  
Without protection, a crash could leave the tree in an inconsistent state.

</div>

<div v-click class="mt-6">

**Write-Ahead Log (WAL)**:  
Before touching any page, Postgres writes the intended change to the WAL — a sequential log on disk.

</div>

<div v-click class="mt-4">

On restart, Postgres **replays the WAL** to bring the index to a consistent state.

</div>

</div>

<div>

```
Normal operation:
  1. Write intent to WAL  ← sequential I/O
  2. Modify B-Tree pages  ← random I/O

Crash recovery:
  WAL exists → replay changes
  Index restored to consistent state ✅

No WAL entry → operation never happened
  Index unchanged ✅
```

<div v-click class="mt-4 p-3 bg-blue-900/30 border border-blue-400 rounded text-sm">

💡 The WAL is also how PostgreSQL streaming replication works — replicas replay the same log.

</div>

</div>

</div>

<!--
Aurel
-->

---
layout: two-cols
---

# GIN — Generalized Inverted Index

Built for columns that contain **multiple values per row**:  
arrays, JSONB, full-text search vectors.

**The idea:**  
Instead of mapping `row → values`, GIN maps `value → list of rows`.

```sql
-- Index a JSONB column
CREATE INDEX idx_tags ON posts USING GIN(tags);

-- Index for full-text search
CREATE INDEX idx_fts ON articles
  USING GIN(to_tsvector('english', body));
```

::right::

<div class="ml-6">

```
Table:
  row 1 → tags: ["postgres", "index"]
  row 2 → tags: ["postgres", "btree"]
  row 3 → tags: ["index", "gin"]

GIN Index (inverted):
  "postgres" → [row 1, row 2]
  "index"    → [row 1, row 3]
  "btree"    → [row 2]
  "gin"      → [row 3]
```

<div v-click class="mt-6">

```sql
-- Ultra-fast: just look up "postgres" in GIN
SELECT * FROM posts
WHERE tags @> ARRAY['postgres'];
```

</div>

<div v-click class="mt-4 p-3 bg-yellow-900/30 border border-yellow-500 rounded text-sm">

⚠️ GIN indexes are expensive to update — they're best on **read-heavy** or **append-only** data.

</div>

</div>

<!--
Aurel
-->

---
layout: two-cols
---

# GiST — Generalized Search Tree

A framework for **custom search trees**.  
Used for geometric, spatial, and range data.

Supports operators standard indexes can't: **contains**, **overlaps**, **nearest-neighbor**.

```sql
-- Geospatial (with PostGIS)
CREATE INDEX idx_location ON places
  USING GIST(coordinates);

-- Date ranges
CREATE INDEX idx_bookings ON reservations
  USING GIST(during);
```

::right::

<div class="ml-6 mt-4">

```sql
-- Find all places within 10km
SELECT name FROM places
WHERE ST_DWithin(
  coordinates,
  ST_MakePoint(2.35, 48.85),
  10000
);

-- Find overlapping bookings
SELECT * FROM reservations
WHERE during && '[2024-06-01, 2024-06-15]'::daterange;
```

<div v-click class="mt-6 p-4 bg-purple-900/30 border border-purple-500 rounded text-sm">

💡 GiST is also used by Postgres for **exclusion constraints** — ensuring no two reservations overlap for the same resource.

</div>

</div>

<!--
Aurel
-->

---
layout: two-cols
---

# BRIN — Block Range Index

The **most compact** index type. Trades precision for size.

Instead of indexing individual values, BRIN stores the **min/max per block range** (128 pages by default).

**Works well when:** data is naturally ordered by insertion order — timestamps, serial IDs, append-only logs.

```sql
CREATE INDEX idx_created ON events
  USING BRIN(created_at);
```

::right::

<div class="ml-6 mt-2 font-mono text-sm">

```
Blocks 0–127:    min: 2024-01-01  max: 2024-01-15
Blocks 128–255:  min: 2024-01-15  max: 2024-02-01
Blocks 256–383:  min: 2024-02-01  max: 2024-02-20
...
```

```sql
WHERE created_at > '2024-02-01'
-- → Skip blocks 0–255 entirely ✅
-- → Scan only blocks 256+ 
```

<div v-click class="mt-4 p-3 bg-green-900/30 border border-green-500 rounded text-sm">

✅ A BRIN index on a 100M-row table can be **under 1 MB**.  
A B-Tree on the same column: several hundred MB.

</div>

<div v-click class="mt-3 p-3 bg-red-900/30 border border-red-500 rounded text-sm">

❌ Useless on randomly ordered data — min/max ranges overlap everywhere, no blocks can be skipped.

</div>

</div>

<!--
Aurel
-->

---
layout: two-cols
---

# Bloom Filter Index

A **probabilistic** index. Built for tables with many columns where you need to filter on arbitrary combinations.

Uses a bit array: each value is hashed into several bit positions. A query tests whether all bit positions for a value are set.

- **False positives possible** — Postgres filters those in a second pass
- **False negatives impossible** — if the bit is not set, the row is definitely excluded

```sql
CREATE EXTENSION bloom;

CREATE INDEX idx_bloom ON orders
  USING BLOOM(customer_id, product_id, status);
```

::right::

<div class="ml-6 mt-4">

```sql
-- Efficiently filters ANY combination:
WHERE customer_id = 42 AND status = 'shipped'
WHERE product_id = 99 AND customer_id = 7
WHERE status = 'pending'
```

<div class="mt-4 font-mono text-xs">

```
Value "active" hashes to bits: 3, 14, 27
Bit array: 0 0 0 1 0 0 ... 1 0 0 ... 1 0 ...
                 ↑              ↑         ↑
           Match! → row might be here (check heap)
```

</div>

<div v-click class="mt-4 p-3 bg-blue-900/30 border border-blue-400 rounded text-sm">

💡 **When to use:** multi-column equality filters on wide tables where individual B-Tree indexes would be too many and too large.

</div>

</div>

<!--
Aurel
-->

---
layout: section
---

# Part 3

## Multi-Column Indexes

*Column order is everything*

<!--
Glenn
-->

---

# Multi-column indexes: the leading column rule

When you create a composite index, **order matters**.

```sql
CREATE INDEX idx ON orders(customer_id, created_at, status);
```

<div class="grid grid-cols-3 gap-4 mt-6 text-sm">

<div v-click class="p-4 bg-green-900/30 border border-green-500 rounded">

✅ **Index used**

```sql
WHERE customer_id = 42

WHERE customer_id = 42
  AND created_at > '2024-01-01'

WHERE customer_id = 42
  AND created_at > '2024-01-01'
  AND status = 'shipped'
```

The **leading column** must always be present.

</div>

<div v-click class="p-4 bg-yellow-900/30 border border-yellow-500 rounded">

⚠️ **Partial use**

```sql
WHERE customer_id = 42
  AND status = 'shipped'
```

Index used on `customer_id`,  
but `status` is skipped —  
Postgres filters it after.

</div>

<div v-click class="p-4 bg-red-900/30 border border-red-500 rounded">

❌ **Index not used**

```sql
WHERE created_at > '2024-01-01'

WHERE status = 'shipped'
```

Neither `created_at` nor `status` alone can use this index — `customer_id` is missing.

</div>

</div>

---
layout: two-cols
---

# Column order: selectivity matters

Put the **most selective column first** — the one that eliminates the most rows.

<div class="mt-4">

**Bad order:**

```sql
-- status has 3 values: low selectivity
-- customer_id has millions of values: high selectivity
CREATE INDEX idx_bad ON orders(status, customer_id);
```

With `WHERE status = 'active'`, the index still returns  
~33% of all rows — Postgres might prefer a seq scan.

</div>

<div v-click class="mt-6">

**Good order:**

```sql
CREATE INDEX idx_good ON orders(customer_id, status);
```

With `WHERE customer_id = 42`, the index returns  
only a handful of rows — very efficient.

</div>

::right::

<div class="ml-6 mt-4">

<div class="p-4 bg-blue-900/30 border border-blue-400 rounded text-sm">

**Selectivity** = the fraction of rows a condition eliminates.

High selectivity → few rows pass the filter → index is very useful.

Low selectivity → many rows pass the filter → index may be ignored.

</div>

<div v-click class="mt-6 p-4 bg-green-900/30 border border-green-500 rounded text-sm">

💡 **Rule of thumb:**  
`customer_id` (millions of values) before `status` (3–10 values) before `boolean` flags.

</div>

<div v-click class="mt-4 text-sm text-gray-400">

You can check column statistics with:

```sql
SELECT n_distinct, correlation
FROM pg_stats
WHERE tablename = 'orders'
  AND attname = 'status';
```

</div>

</div>

---
layout: section
---

# Part 4

## The Cost of Indexes

*Nothing is free*

---
layout: two-cols
---

# Storage cost

Every index is a **separate structure on disk**, maintained in parallel with the heap.

<div class="mt-4">

```sql
-- Check index sizes
SELECT
  indexname,
  pg_size_pretty(pg_relation_size(indexname::regclass)) AS size
FROM pg_indexes
WHERE tablename = 'orders'
ORDER BY pg_relation_size(indexname::regclass) DESC;
```

</div>

<div v-click class="mt-4 p-3 bg-yellow-900/30 border border-yellow-500 rounded text-sm">

On a large table, a single B-Tree index can easily be  
**20–40% of the table size**. Five indexes = table size doubled.

</div>

::right::

<div class="ml-6">

**Typical sizes for a 100M row table:**

```
Table (heap):          ~15 GB
B-Tree on integer:     ~2.5 GB
B-Tree on text(50):    ~5 GB
GIN on JSONB:          ~8 GB
BRIN on timestamp:     ~1 MB  ← exceptional
```

<div v-click class="mt-6 p-4 bg-red-900/30 border border-red-500 rounded text-sm">

❌ **Index bloat:**  
When rows are deleted or updated, old index entries become **dead tuples** — they take space but are useless.

`VACUUM` cleans the heap. `VACUUM` also cleans indexes, but **`REINDEX`** may be needed for severe bloat.

</div>

</div>

<!--
Aurel
-->

---

# Write latency cost

Every write to the table must **also update every index** on that table.

<div class="grid grid-cols-2 gap-8 mt-4">

<div>

```sql
INSERT INTO orders (customer_id, status, created_at, ...)
VALUES (42, 'pending', now(), ...);
```

<div v-click class="mt-4">

**What Postgres actually does:**

```
1. Write to WAL
2. Insert row into heap page
3. Update B-Tree idx_customer_id
4. Update B-Tree idx_status
5. Update B-Tree idx_created_at
6. Update GIN idx_metadata (if exists)
```

</div>

<div v-click class="mt-4 p-3 bg-red-900/30 border border-red-500 rounded text-sm">

Each index update may trigger a **page split**, making the write even more expensive.

</div>

</div>

<div v-click>

**The tradeoff:**

```
Indexes:    0   1   2   3   4   5
            │   │   │   │   │   │
Read        ●───●───●───●───●───●  faster
            │   │   │   │   │   │
Write       ●───●───●───●───●───●  slower
            │   │   │   │   │   │
Storage     ●───●───●───●───●───●  larger
```

<div class="mt-4 p-4 bg-blue-900/30 border border-blue-400 rounded text-sm">

💡 A table with heavy `INSERT`/`UPDATE` load and rarely queried indexes is paying the write cost for nothing.

Always check `pg_stat_user_indexes` for unused indexes.

</div>

</div>

</div>

<!--
Aurel
-->

---
layout: section
---

# Part 5

## Indexing Strategy

*When, what, and how*

---

# Always start with `EXPLAIN ANALYZE`

Before creating any index, **measure**.

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders
WHERE customer_id = 42
  AND created_at > now() - interval '30 days';
```

<div class="grid grid-cols-2 gap-6 mt-4 text-sm">

<div v-click>

**Seq Scan — no index used:**

```
Seq Scan on orders  (cost=0.00..45231.00
                     rows=12 width=128)
                    (actual time=0.041..312.4
                     rows=12 loops=1)
  Filter: ((customer_id = 42) AND ...)
  Rows Removed by Filter: 2000003
Buffers: shared hit=12388 read=20844

Planning Time: 0.3 ms
Execution Time: 312.7 ms  ← 🐢
```

</div>

<div v-click>

**Index Scan — with index:**

```
Index Scan using idx_orders_cust_date
             on orders  (cost=0.56..24.3
                          rows=12 width=128)
                        (actual time=0.041..0.09
                          rows=12 loops=1)
  Index Cond: (customer_id = 42)
  Filter: (created_at > ...)
Buffers: shared hit=5

Planning Time: 0.4 ms
Execution Time: 0.1 ms  ← 🚀
```

</div>

</div>

<!--
A voir si on garde ou supprime cette slide, ou alors on passe en vitesse en expliquant juste la commande explain analyze
-->

---

# Finding unused and missing indexes

<div class="grid grid-cols-2 gap-8">

<div>

**Find indexes never used:**

```sql
SELECT
  schemaname,
  relname,
  indexrelname
  idx_scan AS times_used,
  pg_size_pretty(pg_relation_size(
    indexrelid
  )) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

<div v-click class="mt-4 p-3 bg-red-900/30 border border-red-500 rounded text-sm">

These are pure overhead — paying write cost and storage for zero read benefit. Drop them.

</div>

</div>

<div v-click>

**Find slow sequential scans (missing index candidates):**

```sql
SELECT
  relname AS table,
  seq_scan,
  seq_tup_read,
  idx_scan,
  n_live_tup AS row_count
FROM pg_stat_user_tables
WHERE seq_scan > 0
  AND n_live_tup > 10000
ORDER BY seq_tup_read DESC;
```

<div class="mt-4 p-3 bg-yellow-900/30 border border-yellow-500 rounded text-sm">

High `seq_tup_read` on a large table = Postgres is doing full scans. Consider adding an index.

</div>

</div>

</div>

<!--
Glenn

relname	Nom de la table. 

seq_scan : Nombre de fois qu'on a lancé un scan séquentiel sur la table (lecture complète sans index). 

seq_tup_read : Nombre total de lignes lues lors des scans séquentiels (c'est le vrai coût I/O). 

idx_scan : Nombre de fois qu'on a utilisé un index pour accéder à la table (pour comparer). 

n_live_tup : Nombre de lignes vivantes actuellement dans la table (lignes non supprimées).
-->

---

# When NOT to add an index

<div class="grid grid-cols-2 gap-6 mt-4">

<div v-click class="p-4 bg-red-900/20 border border-red-700 rounded">

### Small tables

```sql
-- 500-row lookup table
-- Seq scan is faster than index lookup
-- The whole table fits in one I/O
SELECT * FROM countries WHERE code = 'FR';
```

Postgres will ignore the index anyway — the planner knows when a seq scan is cheaper.

</div>

<div v-click class="p-4 bg-red-900/20 border border-red-700 rounded">

### Low-cardinality columns

```sql
-- Only 2 values: true/false
-- Index returns 50% of rows → useless
CREATE INDEX idx ON users(is_active);  -- ❌
```

Unless combined with high-selectivity columns in a composite index.

</div>

<div v-click class="p-4 bg-red-900/20 border border-red-700 rounded">

### Heavily written tables

```sql
-- Events table: 100k inserts/sec
-- 5 indexes = 500k index updates/sec
-- Bottleneck is the index maintenance
```

Reduce index count to the minimum. Use BRIN if the data is ordered.

</div>

<div v-click class="p-4 bg-red-900/20 border border-red-700 rounded">

### Full-table aggregations

```sql
-- COUNT(*) or SUM over the whole table
-- Nothing to skip — must read everything
SELECT SUM(amount) FROM transactions;  -- seq scan wins
```

Consider materialized views or pre-aggregation instead.

</div>

</div>

<!--
glenn
-->

---

# Partial indexes — index only what you query

You can add a `WHERE` clause to an index to index only a **subset of rows**.

<div class="grid grid-cols-2 gap-8 mt-4">

<div>

```sql
-- 99% of orders are 'completed'
-- You only query 'pending' orders
CREATE INDEX idx_pending_orders
  ON orders(created_at)
  WHERE status = 'pending';
```

<div v-click class="mt-4 p-4 bg-green-900/30 border border-green-500 rounded text-sm">

✅ The index only contains the 1% of rows where `status = 'pending'`.  
**Dramatically smaller**, faster to update, faster to scan.

</div>

</div>

<div v-click>

```sql
-- Unique constraint only on active users
-- (allows multiple deleted users with same email)
CREATE UNIQUE INDEX idx_active_email
  ON users(email)
  WHERE deleted_at IS NULL;

-- Index for unprocessed jobs only
CREATE INDEX idx_jobs_unprocessed
  ON jobs(priority DESC, created_at)
  WHERE processed_at IS NULL;
```

<div class="mt-4 p-3 bg-blue-900/30 border border-blue-400 rounded text-sm">

💡 Partial indexes are one of the most underused Postgres features. They're especially powerful on **soft-delete** patterns and **status queues**.

</div>

</div>

</div>

<!--
aurel

INCLUDE = ajouter un post-it sur la couverture du livre avec "auteur" et "genre" (tu lis le post-it, pas besoin d'ouvrir le livre) ← aide la lecture 

Partial = créer un mini-catalogue avec SEULEMENT les livres "en cours" (~50k livres) au lieu de tous les 1M ← aide l'indexation et la recherche
-->

---
layout: center
class: text-center
---

# Key Takeaways

<div class="grid grid-cols-2 gap-6 mt-8 text-left text-sm max-w-3xl mx-auto">

<div v-click class="p-4 bg-blue-900/30 border border-blue-500 rounded">

**1. PostgreSQL is always non-clustered**  
The index points to the heap. An `INCLUDE` clause avoids the extra heap fetch.

</div>

<div v-click class="p-4 bg-green-900/30 border border-green-500 rounded">

**2. B-Tree is O(log n) with a linked leaf list**  
Range scans, ordering, and prefix matching are all efficient.

</div>

<div v-click class="p-4 bg-purple-900/30 border border-purple-500 rounded">

**3. Right index type for the right data**  
GIN for arrays/JSONB, GiST for geometry/ranges, BRIN for ordered append-only data.

</div>

<div v-click class="p-4 bg-yellow-900/30 border border-yellow-500 rounded">

**4. Leading column rule in composites**  
Most selective column first. The index can only be used if the leading columns are in the query.

</div>

<div v-click class="p-4 bg-orange-900/30 border border-orange-500 rounded">

**5. Every index has a write + storage cost**  
Unused indexes are pure overhead. Audit with `pg_stat_user_indexes`.

</div>

<div v-click class="p-4 bg-red-900/30 border border-red-500 rounded">

**6. Measure first, index second**  
`EXPLAIN ANALYZE` tells you what's actually happening. Never add indexes blindly.

</div>

</div>

<!--
Aurel
-->

---
layout: center
class: text-center
---

# Go further

<div class="grid grid-cols-3 gap-6 mt-8 text-sm max-w-3xl mx-auto">

<div class="p-4 bg-gray-800 rounded-lg border border-gray-600">
  <div class="text-2xl mb-2">📖</div>
  <div class="font-bold">use-the-index-luke.com</div>
  <div class="text-gray-400 mt-2">Free, in-depth guide to SQL indexing across databases</div>
</div>

<div class="p-4 bg-gray-800 rounded-lg border border-gray-600">
  <div class="text-2xl mb-2">📘</div>
  <div class="font-bold">The Art of PostgreSQL</div>
  <div class="text-gray-400 mt-2">Dimitri Fontaine — covers indexing strategy in depth</div>
</div>

<div class="p-4 bg-gray-800 rounded-lg border border-gray-600">
  <div class="text-2xl mb-2">🐘</div>
  <div class="font-bold">PostgreSQL docs</div>
  <div class="text-gray-400 mt-2">Chapter 11: Indexes — the definitive reference</div>
</div>

</div>

<div class="mt-16 text-gray-400">
  Questions?
</div>
