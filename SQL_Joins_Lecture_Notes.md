# SQL — REPLACE (contd.) & JOINS — Complete Study Notes
**Covers timestamps `2:18:00 – 2:50:45`**

---

## Table of Contents

1. [Part A — Finishing `REPLACE` (3rd Syntax)](#part-a--finishing-replace-3rd-syntax)
2. [Part B — `REPLACE` vs `UPDATE`](#part-b--replace-vs-update)
3. [Part C — Introduction to JOINS](#part-c--introduction-to-joins)
4. [Part D — INNER JOIN](#part-d--inner-join)
5. [Part E — Aliasing (`AS`)](#part-e--aliasing-as)
6. [Part F — OUTER JOINS: LEFT JOIN](#part-f--outer-joins--left-join)
7. [Part G — RIGHT JOIN](#part-g--right-join)
8. [Part H — FULL JOIN (emulated in MySQL)](#part-h--full-join-emulated-in-mysql)
9. [Part I — CROSS JOIN](#part-i--cross-join)
10. [Part J — SELF JOIN (emulated)](#part-j--self-join-emulated)
11. [Part K — Practical Workbench Session (Schema + 5 Queries)](#part-k--practical-workbench-session)
12. [Part L — Master Comparison Table](#part-l--master-comparison-table-of-all-joins)
13. [Part M — ⚠️ Instructor Errors / Corrections](#part-m--️-instructor-errors--things-stated-incorrectly)
14. [Part N — Exam & Interview Traps](#part-n--exam--interview-traps)
15. [Part O — Quick Revision Sheet](#part-o--quick-revision-sheet)

---

# PART A — Finishing `REPLACE` (3rd Syntax)

The lecture picks up mid-way through the three syntaxes of `REPLACE`. Here are all three, since syntax 3 is only meaningful in contrast with 1 and 2.

## A.1 The three `REPLACE` syntaxes (as shown on screen)

```sql
-- Syntax 1: column-list + VALUES  (most commonly used)
REPLACE INTO Customer(id, City)
    VALUES(1251, 'Colony');

-- Syntax 2: SET style  (also commonly used)
REPLACE INTO Customer SET id = 1300, cname = 'Mac', City = 'Utah';

-- Syntax 3: REPLACE ... SELECT  (rarely used)
REPLACE INTO Customer(id, cname, City)
    SELECT id, cname, City
    FROM CUSTOMER WHERE id = 500;
```

**Instructor's own preference (stated verbatim in the lecture):** he normally uses **syntax 1 and syntax 2**, and says syntax 3 *"is not very useful."* He then adds **"THIS IS VERY IMPORTANT"** — meaning: know that all three exist, but the first two are what you actually write.

## A.2 What syntax 3 actually did — walk-through

The query targeted the row with `id = 500`.

**Before running:** the `CUSTOMER` table had a row with `id = 500`, `cname = 'Rohan Arora'`, and all its other columns populated.

**What the query says:**
- `REPLACE INTO Customer(id, cname, City)` → I am supplying values for **only three columns**: `id`, `cname`, `City`.
- `SELECT id, cname, City FROM CUSTOMER WHERE id = 500` → and the values I'm supplying are pulled out of the row where `id = 500`.

**Result Grid output shown on screen:**

| id | cname | City | (all other columns) |
|---|---|---|---|
| 500 | Rohan Arora | Colony | **`NULL`** |

**Read that carefully.** The row with key `500` was *replaced*. The three columns we listed (`id`, `cname`, `City`) survived with their values. **Every column we did NOT list got wiped to `NULL`.**

## A.3 ⭐ WHY do the unlisted columns become `NULL`? (the reasoning behind it)

> *Added explanation — the instructor states the behaviour ("baaki saari values ko NULL set kar diya") but does not spell out the mechanism. This mechanism is the single most interview-relevant fact about `REPLACE`.*

`REPLACE` is **not** an update. Internally, MySQL executes `REPLACE` as:

```
1. Try to INSERT the new row.
2. If a duplicate-key conflict occurs on a PRIMARY KEY or UNIQUE index:
      a. DELETE the entire old conflicting row.
      b. INSERT the new row fresh.
```

Because step 2(a) **deletes the whole old row**, there is nothing left of the old values. Step 2(b) then inserts a brand-new row built only from the columns you supplied. Any column you did not supply falls back to its **column default**, which is `NULL` for a nullable column with no default.

**This is the difference in one line:**
- `UPDATE` **modifies** a row → untouched columns keep their old values.
- `REPLACE` **destroys and rebuilds** a row → untouched columns are reset to defaults/`NULL`.

⚠️ **Practical danger:** if you `REPLACE` a row while listing only 3 of its 10 columns, you have just silently destroyed 7 columns of real data. This is exactly why the instructor says syntax 3 "is not very useful" — it looks like a targeted edit but behaves like a destructive rewrite.

---

# PART B — `REPLACE` vs `UPDATE`

This comparison is directly from the on-screen handwritten notes and is a classic interview question.

## B.1 On-screen notes (verbatim)

```
Replace :-
1. Data already present, Replace.
2. Data not present, INSERT

Replace VS Update
-> if row is not present, Replace will add a new row while UPDATE will do nothing
```

## B.2 Expanded explanation

**`REPLACE` behaves in two modes depending on whether the row exists:**

| Situation | What `REPLACE` does | What `UPDATE` does |
|---|---|---|
| Row **already present** (key conflict) | Deletes the old row and inserts the new one → effectively replaces it | Modifies the matching row in place, leaving unlisted columns untouched |
| Row **not present** | Performs a plain **`INSERT`** — a brand-new row appears | **Does absolutely nothing.** No error, no row, no effect. Affected rows = 0 |

## B.3 The one-line summary to memorise

> **If the row is not present, `REPLACE` will add a new row while `UPDATE` will do nothing.**

**Intuition:** `REPLACE` is an *"upsert"* — a combination of update-or-insert. `UPDATE` is strictly a *modifier*; it can only work on something that already exists. It has no power to create.

**Second distinction (from Part A.3, equally examinable):** even when the row *does* exist, they are not equivalent — `UPDATE` preserves unlisted columns, `REPLACE` nukes them.

---

# PART C — Introduction to JOINS

> The instructor flags this explicitly: **"kaafi important topic... interviewer ka favorite topic"** — a very important topic and an interviewer's favourite. Learn this thoroughly.

## C.1 Why do JOINS even exist? (the motivation)

The chain of reasoning the instructor builds:

1. **All RDBMS are relational in nature.** "Relational" means one table can *refer to* another table.
2. **How is that reference physically created?** Through a **Foreign Key (FK)**. This is what you already studied under referential constraints — referencing table, referenced table, child table, parent table.
3. So: **Foreign keys establish the relationships.**
4. **Now the new question:** we have established relations… so how do we actually **fetch data across those relations**?
5. **Answer: JOINS.**

Written as the on-screen notes put it:

```
# JOINS
-> F.K -> Relations
-> use -> Data fetch -> JOINS
```

**In one sentence:** *Foreign keys create the relationship; JOINS are how you read data across that relationship.*

## C.2 The standard two-table setup and terminology

The instructor sets up a canonical picture that is reused for every single join type. **Memorise this picture — every join is explained on top of it.**

```
        LEFT TABLE  (Parent Table)              RIGHT TABLE  (Child Table)
        ┌────┬────┬───────────┐                 ┌───────────┬────┬────┐
        │ L1 │ L2 │ Key (P.K) │                 │ Key (F.K) │ R1 │ R2 │
        ├────┼────┼───────────┤                 ├───────────┼────┼────┤
        │    │    │     1     │  ◄───────────►  │     1     │    │    │
        │    │    │     2     │  ◄───────────►  │     2     │    │    │
        │    │    │     3     │  ◄───────────►  │     3     │    │    │
        │    │    │     4     │   (no match)    └───────────┴────┴────┘
        └────┴────┴───────────┘
```

**Terminology you must be fluent in:**

| Term | Meaning in this diagram |
|---|---|
| **Left Table** | The table written **first** in the query (immediately after `FROM`) |
| **Right Table** | The table written **second** (after the `JOIN` keyword) |
| **Key in Left Table** | The **Primary Key** |
| **Key in Right Table** | The **Foreign Key** referencing the left table's PK |
| **Referencing table / Child table** | The table that **contains** the foreign key → here, the Right table |
| **Referenced table / Parent table** | The table being **pointed at** → here, the Left table |

## C.3 ⭐ How to think about what a JOIN is doing (the intuition)

The instructor's key mental model:

> *"These two tables are stored separately in memory. They are two completely different tables. **But** there is a relation between them — and what establishes that relation is the key."*

So when you see key value `1` in the left table:
- the whole left-side row for `1` "belongs to" key 1, **and**
- the whole right-side row for `1` also "belongs to" key 1.

A JOIN uses the key as the glue to say: *"all of this data — from both tables — is the data corresponding to key 1."* The two separate tables are stitched into one.

⚠️ **Note on Left = Parent:** In this diagram the Left table happens to be the parent (PK side) and the Right table the child (FK side). **That is a property of this specific example, not a rule.** "Left" and "Right" are determined *purely by writing order in the query* — nothing more. You are perfectly free to write the child table first, in which case the child becomes the Left table. (Flagged because the diagram can create a false impression.)

## C.4 The prerequisite for applying any JOIN

From the on-screen notes:

```
- Join -> To apply joins, there should be a common attribute between both the tables.
```

**Meaning:** you cannot join two tables arbitrarily. There must be a **common attribute** — practically, the PK–FK relationship — that lets you say "this row here corresponds to that row there."

*(The one exception, covered later, is `CROSS JOIN`, which needs no common attribute at all because it matches everything with everything.)*

---

# PART D — INNER JOIN

## D.1 Definition (verbatim from the PDF notes on screen)

```
JOINING TABLES
1. All RDBMS are relational in nature, we refer to other tables to get meaningful outcomes.
2. FK are used to do reference to other table.
3. INNER JOIN
   1. Returns a resultant table that has matching values from both the tables or all the tables.
   2. SELECT column-list FROM table1 INNER JOIN table2 ON condition1
      INNER JOIN table3 ON condition2
```

> **INNER JOIN returns a resultant table that has matching values from both the tables (or all the tables).**

## D.2 Unpacking the definition, phrase by phrase

| Phrase | What it actually means |
|---|---|
| *"matching values"* | We compare the two tables **column-wise on the join key**. Only where the key values are equal in both tables do we produce output. |
| *"from both the tables"* | The output row contains columns from **both** tables merged together. |
| *"a resultant table"* | A JOIN **generates a brand-new table**. The original tables are untouched. |
| *"or all the tables"* | You are not limited to two tables — you can chain joins across three or more. |

## D.3 Worked example — what gets kept and what gets dropped

**Setup:**
- Left table keys: `1, 2, 3, 4`
- Right table keys: `1, 2, 3`

Key `4` exists in the left (parent) table but has **no corresponding row** in the right (child) table.

**Applying `INNER JOIN`:** compare the keys of both tables and find the matches → `1, 2, 3` match; `4` does not.

**Resultant table (as drawn on screen):**

```
① INNER JOIN
Resultant table:
Key | L1 | L2 | R1 | R2
----|----|----|----|---
 1  |    |    |    |
 2  |    |    |    |
 3  |    |    |    |
```

**Key observation #1 — key `4` vanishes.** *"4 ko humne chhod diya"* — we dropped 4. It had no partner on the right, so INNER JOIN discards it entirely.

**Key observation #2 — the number of COLUMNS grows.** This is the point the instructor stresses hardest and explicitly warns you to remember:

> *"Dhyan se dekho INNER JOIN apply karke number of columns badha liye… Isko dhyan dena kyunki aage jab hum ek aur cheez padhne wale hain to usme confusion create ho sakti hai."*
> — Note that JOIN **increases the number of columns**. Remember this, because when we study another topic later it will cause confusion.

**What "later topic" is he warning about?** He is contrasting `JOIN` with `UNION`:
- **JOIN → grows the table HORIZONTALLY** (columns from both tables side by side).
- **UNION → grows the table VERTICALLY** (rows stacked on top of each other).

Keep this contrast filed away; it becomes directly relevant in Part H (FULL JOIN), where both operations appear in the same query.

## D.4 Venn diagram for INNER JOIN

```
      ┌───────────────┐
      │  Left table   │
      │        ┌──────┼────────────┐
      │        │//////│            │
      │        │//////│  Right     │
      │        │//////│  table     │
      │        └──────┼────────────┘
      └───────────────┘
               ▲
        Shaded = INNER JOIN result
        (the intersection only)
```

Only the **intersection** — rows whose key value appears in *both* tables — is returned.

## D.5 Syntax

**General form:**

```sql
SELECT column-list
FROM table1
INNER JOIN table2 ON condition1;
```

**Multi-table form (on screen, not verbally explained — explained here):**

```sql
SELECT column-list
FROM table1
INNER JOIN table2 ON condition1
INNER JOIN table3 ON condition2;
```

> *Explanation added for the on-screen line the instructor didn't narrate:* joins **chain**. `table1 INNER JOIN table2 ON condition1` produces an intermediate result; that intermediate result is then joined to `table3` using `condition2`. `condition2` may therefore reference columns from `table1`, `table2`, or `table3`. This is how you pull data spanning three or more related tables in one query.

## D.6 The Customer–Orders example (built line by line)

```sql
Select c.*, o.*
FROM Customer AS c
INNER JOIN Orders AS o
ON c.id = o.cust_id;
```

**How the instructor builds it:**

1. **`FROM Customer AS c`** — start from the Customer table, give it the short alias `c`.
2. **`INNER JOIN Orders AS o`** — I want matching values, joined against the Orders table, aliased `o`.
3. **`ON c.id = o.cust_id`** — the condition. This is where you tell the query *which* columns to match on: **Left table's key must equal Right table's key.**
4. **`SELECT c.*, o.*`** — which columns to display. Since the join produced a wider table, we can pull columns from both sides.

**Why `c.id = o.cust_id` and not something else?** Recall the schema design: *"Yaad karo humne Order ke andar customer ki ID embed ki thi"* — we embedded the customer's ID inside the Order table. So `Orders` holds `cust_id` as a **foreign key**, which means **Orders is the child table**, and `Customer` is the parent.

**Selecting specific columns instead of `*`:**

```sql
-- instead of c.*, o.* you can write:
c.id, c.name, o.product_name
```

## D.7 Which table is "Left" and which is "Right"? (critical rule)

> **The Left table always appears leftmost — i.e., first — in the query.**

In `FROM Customer AS c INNER JOIN Orders AS o`:
- `Customer` is written first → **Customer is the Left table**
- `Orders` is written second → **Orders is the Right table**

The instructor's explicit warning:

> *"Jab aap query likhoge to ye cheez dhyan se rakhna ki INNER JOIN ke andar ye Left hota hai, ye Right hota hai, to Left-Right ka dhyan rakhna."*
> — When you write the query, be careful about which is Left and which is Right.

For `INNER JOIN` this ordering doesn't change the result set (matching is symmetric), but for `LEFT JOIN` and `RIGHT JOIN` **it completely changes the answer**. Build the habit now.

## D.8 Importance

> *"Sabse zyada use hone wala join INNER JOIN hi hai."* — **INNER JOIN is the most-used join.** The instructor says to learn it thoroughly.

---

# PART E — Aliasing (`AS`)

## E.1 What it is

`AS` gives a table (or column) a temporary short name **for the duration of that one query**.

```sql
FROM Customer AS c          -- from now on in this query, Customer is 'c'
INNER JOIN Orders AS o      -- and Orders is 'o'
```

## E.2 Why we need it here specifically

Two reasons, both given in the lecture:

**Reason 1 — Brevity.** *"Customer likhne mein bahut bada lag raha tha"* — writing `Customer` everywhere is long, so `c` shortens it.

**Reason 2 — Disambiguation (the real reason).** This is the important one:

> *"Yahan par do table aa gaye. Abhi tak ek hi table ke baare mein baat ho rahi thi humari, ab yahan pe do tables hain. To dono ko distinguish karna padega."*

Until now every query touched **one** table, so an unqualified column name like `id` was unambiguous. Now **two tables are in play**, and both may have a column called `id`. Writing bare `id` would be ambiguous — the engine cannot tell which table you mean. So you qualify it: `c.id` means "the `id` of the Customer table," `o.id` means "the `id` of the Orders table."

## E.3 The instructor's analogy — think of it like C++ objects

> *"Jaise aap C++ mein karte ho kisi bhi object pe — `customer.id`, `customer.kuch` — waise hi yahan pe `c.` ka matlab customer table ko refer kiya ja raha hai."*

Just as in C++ you write `object.member` to access a member of a specific object, in SQL you write `alias.column` to access a column of a specific table. The dot means the same thing conceptually: *"this member, belonging to this thing."*

```
c.id            → the id column of Customer
c.name          → the name column of Customer
o.product_name  → the product_name column of Orders
c.*             → all columns of Customer
o.*             → all columns of Orders
```

## E.4 Scope of the alias

The alias exists **only inside that query**. It is not stored, it does not rename the table in the database, and it disappears the moment the query finishes. *"For this query, for this particular query, Customer ko main `c` se refer kar sakta hu."*

> *Small addition:* the `AS` keyword is optional for table aliases in MySQL — `FROM Customer c` works identically to `FROM Customer AS c`. Writing `AS` is clearer and is what the instructor uses throughout.

---

# PART F — OUTER JOINS — LEFT JOIN

## F.1 What "OUTER JOIN" means conceptually

> *"OUTER JOINs… matlab hum dono ki matching values se thoda bahar jaane ki koshish karenge."*
> — With OUTER JOINS we try to go a little **beyond** just the matching values.

INNER JOIN stays strictly inside the intersection. OUTER JOINS deliberately step **outside** it to also pull in non-matching rows.

The family covered: **LEFT JOIN**, **RIGHT JOIN**, and **FULL JOIN**.

## F.2 LEFT JOIN — definition

> **This returns a resulting table that has all the data from the left table and the matched data from the right table.**

Two clauses, and both matter:
1. **ALL** the data from the Left table — unconditionally, matched or not.
2. **Only the matched** data from the Right table.

## F.3 The obvious question — and its answer

> *"Ab simple sa question aata hai, aap sochoge ki ye to match hi nahi ho rahe hain — 5 ke corresponding kaise data aayega?"*

If the left row (say key `5`) has no partner on the right, the join must still output it (clause 1 demands it) — but what goes in the right-hand columns?

**Answer: `NULL`.** *"5 ke corresponding mujhe Right table mein `NULL` values milengi."*

**Why `NULL` and not blank / 0 / error?** Because `NULL` in SQL means *"no value exists here"* — which is precisely the truth: there genuinely is no matching right-side row. The join preserves the shape of the result table (every row must have all columns) by filling the missing side with `NULL`.

## F.4 Worked example (from the on-screen notes)

**Setup:**

```
② Left Join
Left Table         R. Table
L1 | L2 | Key      Key | R1 | R2
---|----|---       ----|----|---
   |    | 1          1 |    |
   |    | 2          2 |    |
   |    | 3          3 |    |
   |    | 4
   |    | 5
   |    | 6
```

Left has keys `1–6`. Right has only `1, 2, 3`.

**Resultant table:**

```
Key | L1 | L2 | R1   | R2
----|----|----|------|-----
 1  |    |    | Data | Data
 2  |    |    | Data | Data
 3  |    |    | Data | Data
 4  |    |    | Null | Null
 5  |    |    | Null | Null
 6  |    |    | Null | Null
```

**Reading the result against the definition** (the instructor explicitly checks this — *"jaisi definition mein bola tha wahi cheez honor ho rahi hai"*):
- *"all the data from left table"* → keys 1–6 are **all** present. ✔
- *"the matched data from the right table"* → keys 1, 2, 3 carry real right-side data. ✔
- Keys 4, 5, 6 had no match → right-side columns filled with `NULL`. ✔

**Row count note:** LEFT JOIN never loses a left row. Left had 6 rows → result has 6 rows. *(It can produce **more** than 6 if a left key matches multiple right rows — see Part K.)*

## F.5 Venn diagram

```
      ┌───────────────┐
      │///////////////│
      │/////// ┌──────┼────────────┐
      │////////│//////│            │
      │////////│//////│   Right    │
      │/////// └──────┼────────────┘
      └───────────────┘
      Shaded = entire LEFT circle + intersection
```

## F.6 Syntax

```sql
select c.*, o.*
FROM Customer as c
LEFT JOIN Orders as o
ON c.id = o.cust_id;
```

> *"Matlab kuch nahi karna, sab kuch same hai — bas `LEFT JOIN` likha hua hai."*
> — Nothing else changes; only the keyword `INNER JOIN` becomes `LEFT JOIN`.

---

# PART G — RIGHT JOIN

## G.1 Definition and idea

> **RIGHT JOIN is the exact opposite of LEFT JOIN.**
> On-screen note: *"Interested in knowing only the right side info."*

By symmetry with LEFT JOIN, the definition is:

> **Returns a resulting table that has all the data from the RIGHT table and the matched data from the LEFT table.**

Non-matching right rows still appear, with `NULL` filled into the **left-side** columns.

## G.2 Venn diagram

```
      ┌───────────────┐
      │               │
      │        ┌──────┼────────────┐
      │        │//////│////////////│
      │        │//////│   Right    │
      │        └──────┼────────────┘
      └───────────────┘
      Shaded = entire RIGHT circle + intersection
```

## G.3 Worked example (from the on-screen notes)

Right table has keys `1, 2, 3, 4, 5`; Left table has only `1, 2, 3`.

```
③ Right Join
Resultant Table:
Key | L1   | L2   | R1 | R2
----|------|------|----|---
 1  | Data | Data |    |
 2  | Data | Data |    |
 3  | Data | Data |    |
 4  | Null | Null |    |
 5  | Null | Null |    |
```

- All 5 right-side keys survive (that's the whole point of RIGHT JOIN).
- Keys `1, 2, 3` matched → real left-side data.
- Keys `4, 5` had no left-side partner → left columns become `NULL`.

## G.4 The clean one-line contrast (memorise this)

> *"LEFT JOIN aur RIGHT JOIN ek doosre ke opposite hain — LEFT JOIN mein mujhe Right side mein `NULL` mil raha hai, aur RIGHT JOIN mein mujhe Left side mein `NULL` mil raha hai."*

| Join | Which side is fully preserved | Which side can contain `NULL` |
|---|---|---|
| **LEFT JOIN** | Left table (100% of its rows) | **Right** side columns |
| **RIGHT JOIN** | Right table (100% of its rows) | **Left** side columns |

**Memory hook:** *the named side is the side that survives; the other side is where the `NULL`s appear.*

## G.5 ⭐ LEFT and RIGHT JOIN are interchangeable

A point the instructor makes at the end of the practical section — and a frequent interview question:

> *"LEFT JOIN, RIGHT JOIN interchangeably use ho jaate hain, kyunki jo aap Left table maan rahe ho usko agar Right mein likh doge to wo Right Join ki tarah use hoga."*

**Meaning:** every RIGHT JOIN can be rewritten as a LEFT JOIN simply by **swapping the order of the two tables**:

```sql
A LEFT  JOIN B ON cond      -- is exactly equivalent to
B RIGHT JOIN A ON cond
```

**Consequence:** RIGHT JOIN is technically redundant. Many teams and style guides therefore standardise on LEFT JOIN only, because reading a query left-to-right matches the direction of preservation and is less error-prone.

---

# PART H — FULL JOIN (emulated in MySQL)

## H.1 Building up to it — recap of the three joins so far

The instructor deliberately recaps before introducing FULL JOIN. Follow this progression:

| Join | Focus | What you get | Where `NULL`s appear |
|---|---|---|---|
| **INNER** | The common part | Only matching rows | Nowhere (no unmatched rows survive) |
| **LEFT** | The left table | All left rows + matched right data | Right-side columns |
| **RIGHT** | The right table | All right rows + matched left data | Left-side columns |
| **FULL** | Everything | All left rows **and** all right rows | Both sides, wherever unmatched |

## H.2 Definition

> **FULL JOIN is nothing but the UNION of LEFT JOIN and RIGHT JOIN.**

> *"Left table ke corresponding saara data milega — agar wo Right mein nahi hai to bhi milega. Aur Right table ka bhi saara data milega — even agar wo Left mein nahi hai to bhi milega."*

You lose nothing from either table.

## H.3 ⚠️ MySQL does NOT have a `FULL JOIN` keyword

This is a **guaranteed exam/interview point**, stated emphatically:

> *"MySQL ke andar FULL JOIN karke koi keyword available hi nahi hai — **NO, NOT AVAILABLE**. To hum isko **emulate** karte hain."*

*(For context: `FULL OUTER JOIN` is standard SQL and exists in PostgreSQL, Oracle, SQL Server. MySQL is the notable one that omits it.)*

## H.4 How to emulate it

On-screen note:

```
④ Full Join
- Emulated in MySQL using LEFT JOIN U RIGHT JOIN
```

Run a LEFT JOIN, run a RIGHT JOIN, and take the **`UNION`** of the two result sets.

**Syntax (verbatim from screen):**

```sql
select * from lefttable as l
LEFT JOIN Righttable as r ON l.key = r.key
UNION
select * from lefttable as l
RIGHT JOIN Righttable as r ON l.key = r.key;
```

## H.5 ⭐ Why does `UNION` (and not `UNION ALL`) make this work?

> *Added explanation — the instructor gives the recipe but not the reason. This is exactly the follow-up an interviewer asks.*

The matching rows (the intersection) appear in **both** the LEFT JOIN result and the RIGHT JOIN result. If you simply stacked the two result sets, every matched row would be **duplicated**.

`UNION` in SQL performs **set union — it removes duplicate rows automatically.** That deduplication is precisely what collapses the doubled intersection back down to one copy each, producing a correct FULL JOIN.

> Had you written `UNION ALL` (which keeps duplicates), every matching row would appear twice and the result would be wrong.

**Also recall the JOIN-vs-UNION warning from Part D.3:** this query uses both, and they act on different axes —
- the `LEFT JOIN` / `RIGHT JOIN` widen the table **horizontally** (more columns),
- the `UNION` stacks results **vertically** (more rows).

Both halves must therefore produce the **same number of columns in the same order**, which is why both use `select *` on the same pair of tables.

## H.6 Worked example

**Setup:**
- Left table keys: `1, 2, 3` (matching) **plus** `4, 5, 6` (no partner on the right)
- Right table keys: `1, 2, 3` (matching) **plus** `7, 8, 9` (no partner on the left)

**What each join returns from this same data:**

| Join applied | Rows returned |
|---|---|
| **INNER JOIN** | `1, 2, 3` only |
| **LEFT JOIN** | `1, 2, 3` (full data) + `4, 5, 6` (right side `NULL`) |
| **RIGHT JOIN** | `1, 2, 3` (full data) + `7, 8, 9` (left side `NULL`) |
| **FULL JOIN** | `1, 2, 3` + `4, 5, 6` (right `NULL`) + `7, 8, 9` (left `NULL`) |

**Final FULL JOIN result:**

```
Key | L1   | L2   | R1   | R2
----|------|------|------|------
 1  | Data | Data | Data | Data
 2  | Data | Data | Data | Data
 3  | Data | Data | Data | Data
 4  | Data | Data | Null | Null    ← contributed by the LEFT JOIN half
 5  | Data | Data | Null | Null
 6  | Data | Data | Null | Null
 7  | Null | Null | Data | Data    ← contributed by the RIGHT JOIN half
 8  | Null | Null | Data | Data
 9  | Null | Null | Data | Data
```

---

# PART I — CROSS JOIN

## I.1 Definition

> **CROSS JOIN is nothing but the Cartesian Product.**

Every row of the left table is paired with **every** row of the right table.

## I.2 The row-count rule

```
-> Cartesian product
T_L (5 rows)  x  T_R (10 rows)  => Resultant (5 * 10 = 50 rows)
```

> **Rows in result = (rows in Left table) × (rows in Right table)**

## I.3 ⭐ The defining difference from every other join

> *"Jaise Inner Join, Left Join, Right Join hain — hum kisi condition pe karte the… hum keys ko match karne ki koshish kar rahe the. Yahan hum keys-vees ko kuch match nahi karte."*

**CROSS JOIN has NO `ON` condition.** There is no key matching whatsoever. You simply take the left table, take the right table, and generate every possible combination.

This is also why CROSS JOIN is the **one exception** to the "there must be a common attribute" rule from Part C.4 — it needs no relationship at all.

## I.4 Worked example (from screen)

```
Left table:        Right table:
L1 | L2            R1 | R2
---|---            ---|---
 1 | A              3 | C
 1 | B              4 | D
 2 | B
```

Wait — the instructor's drawn left table lists three rows but he then counts **2 rows × 2 rows = 4**. The **resultant table drawn on screen** is built from left rows `(1,A)` and `(2,B)`:

```
Resultant:
L1 | L2 | R1 | R2
---|----|----|---
 1 | A  | 3  | C
 1 | A  | 4  | D
 2 | B  | 3  | C
 2 | B  | 4  | D
```

**How it is generated (the pairing order):**
1. Take left row `(1, A)` → pair it with right row `(3, C)` → output row 1
2. Take left row `(1, A)` → pair it with right row `(4, D)` → output row 2
3. Move to left row `(2, B)` → pair with `(3, C)` → output row 3
4. Left row `(2, B)` → pair with `(4, D)` → output row 4

**Verification:** 2 left rows × 2 right rows = **4 result rows.** ✔

*(⚠️ See Part M for the arithmetic inconsistency in the drawn left table.)*

## I.5 Industrial usefulness

On-screen note is blunt:

```
- Industrial use: X
```

> *"Cartesian product ki agar main baat karu to iska koi industrial use nahi hai, kuch khas kaam nahi aata."*

The only use case he can offer: *"maan lo mujhe har bande ke corresponding saare project ki ek list chahiye"* — i.e., generating **every possible combination** of two sets (e.g., every employee against every project, for building a blank allocation grid). Even then he adds: *"I think ye kuch zyada khas use hota nahi hai."*

⚠️ **Practical danger (worth knowing):** because the output size is the *product* of the inputs, an accidental CROSS JOIN on two large tables explodes catastrophically — 10,000 × 10,000 rows = 100 million rows. A forgotten `ON` clause is a classic production incident.

---

# PART J — SELF JOIN (emulated)

## J.1 The motivation — Unary Relations

Recall from ER-modelling: a **Unary Relation** is a relationship a table has **with itself**.

**Classic example:** `Employee **manages** Employee`.

```
        ┌──────────────┐
        │              │ manages
        │   EMPLOYEE   │◄────┐
        │              │─────┘
        └──────────────┘
```

An employee's manager is also an employee — stored in the *same* table. So to answer *"who manages whom?"* you must join the Employee table **to itself**.

> *"Employee table ke upar hi Employee table baitha hua hai… mujhe nikaalna hai kaun kiska manager hai — is case ke liye main Self Join ka use karta hu."*

## J.2 ⚠️ There is no `SELF JOIN` keyword either

Like FULL JOIN, SELF JOIN is **emulated**:

```
⑤ Self Join
- Unary Relation: Employee manages Employee
- Emulate using 'INNER JOIN' and Alias 'AS'
```

> **Emulated using `INNER JOIN` + aliasing (`AS`).**

## J.3 ⭐ How the trick works — the intuition

This is the most elegant idea in the lecture:

> *"Virtually aisa kuch ho raha hai — **software ko bewaqoof banane ki koshish ho rahi hai**. Same table hai, par uske do reference bana diye humne, do alias bana diye. Ab ye do alag-alag table ki tarah behave kar rahe hain."*

— *We are essentially fooling the software.* It is **one physical table**, but by giving it **two different aliases** we create two independent references to it. As far as the query engine's join logic is concerned, `e1` and `e2` are two separate tables:

- **`e1` acts as the Left table**
- **`e2` acts as the Right table**

Then you apply a completely ordinary `INNER JOIN` between them.

**This is exactly why aliasing (Part E) is mandatory here, not optional.** Without two distinct aliases there would be no way to say "the id in *this* copy" vs "the id in *that* copy" — every column reference would be hopelessly ambiguous.

## J.4 Example shown

```sql
select e1.id, e2.id, e2.name
FROM employee as e1
INNER JOIN employee as e2
ON e1.id = e2.id;
```

**Structure to take away:**
- Same table name (`employee`) appears **twice**
- Each occurrence gets its **own alias** (`e1`, `e2`)
- Ordinary `INNER JOIN` between the two aliases
- `ON` condition compares a column of `e1` against a column of `e2`

⚠️ **This specific query is logically wrong as a manager query — see Part M.2 for the correction.** Learn the *structure* from it, not the `ON` condition.

---

# PART K — Practical Workbench Session

## K.1 The database schema

Three tables: **Project**, **EMPLOYEE**, **CLIENT**.

### Table 1: `Project`

| id | empID | name | startdate | clientID |
|---|---|---|---|---|
| 1 | 1 | A | 2021-04-21 | 3 |
| 2 | 2 | B | 2021-03-12 | 1 |
| 3 | 3 | C | 2021-01-16 | 5 |
| 4 | 3 | D | 2021-04-27 | 2 |
| 5 | 5 | E | 2021-05-01 | 4 |

### Table 2: `EMPLOYEE`

| id | fname | lname | age | emailID | phoneNo | City |
|---|---|---|---|---|---|---|
| 1 | Aman | Proto | 32 | aman@gmail.com | 898 | Delhi |
| 2 | Yagya | Narayan | 44 | yagya@gmail.com | 222 | Palam |
| 3 | Rahul | BD | 22 | rahul@gmail.com | 444 | Kolkata |
| 4 | Jatin | Hermit | 31 | jatin@gmail.com | 666 | Raipur |
| 5 | PK | Pandey | 21 | pk@gmail.com | 555 | Jaipur |

### Table 3: `CLIENT`

| id | first_name | last_name | age | emailID | phoneNo | City | empID |
|---|---|---|---|---|---|---|---|
| 1 | Mac | Rogers | 47 | mac@hotmail.com | 333 | Kolkata | 3 |
| 2 | Max | Poster | 27 | max@gmail.com | 222 | Kolkata | 3 |
| 3 | Peter | Jain | 24 | peter@abc.com | 111 | Delhi | 1 |
| 4 | Sushant | Aggarwal | 23 | sushant@yahoo.com | 45454 | Hyderabad | 5 |
| 5 | Pratap | Singh | 36 | p@xyz.com | 77767 | Mumbai | 2 |

### The relationships (foreign keys)

```
   EMPLOYEE.id ──────┐
                     ├──► Project.empID     (which employee works on which project)
                     └──► CLIENT.empID      (which employee works with which client)

   CLIENT.id ────────────► Project.clientID  (which client the project is for)
```

- **`Project.empID`** is a foreign key → EMPLOYEE
- **`Client.empID`** is a foreign key → EMPLOYEE
- **`Project.clientID`** is a foreign key → CLIENT

**Reading the data (as the instructor walks through it):**
- Employee `1` = Aman. `Project` row 1 has `empID = 1` → **Aman works on project A.**
- `Client` row 3 = Peter Jain, with `empID = 1` → **Aman works with client Peter.**
- Cross-check: `Project` row 1 has `clientID = 3` = Peter. Consistent.

---

## K.2 Query 1 — INNER JOIN

> **Question:** *Enlist all the employees' IDs, names along with the Project allocated to them.*

**Thinking it through:**
1. I need employee ID and name → from `Employee`.
2. I need the project they work on → from `Project`.
3. **What is common between them?** `Employee.id` ↔ `Project.empID`. That's my join condition.
4. I want only employees who *have* a project → **INNER JOIN**.

```sql
select e.id, e.fname, e.lname, p.id, p.name
from employee as e
INNER JOIN project as p on e.id = p.empID;
```

**Dry run — output:**

| e.id | e.fname | e.lname | p.id | p.name |
|---|---|---|---|---|
| 1 | Aman | Proto | 1 | A |
| 2 | Yagya | Narayan | 2 | B |
| 3 | Rahul | BD | 3 | C |
| 3 | Rahul | BD | 4 | D |
| 5 | PK | Pandey | 5 | E |

**Observations the instructor draws out:**
- **Number of columns has increased** — left and right tables attach **column-wise**.
- Employee 1 (Aman) → project A; Employee 2 (Yagya) → project B.
- **Employee 3 (Rahul) appears TWICE** — on projects C and D. ⭐ *One left row matching multiple right rows produces multiple output rows.* This is why a join result can have **more rows than either input table**.
- **Employee 4 (Jatin) is completely absent** — he has no project, so INNER JOIN drops him. (This becomes the whole point of Query 3.)

### K.2a The `select *` demonstration — proving how JOIN works

The instructor deliberately re-runs the same query with `*` instead of a column list, purely to demonstrate the mechanics:

```sql
select * from employee as e INNER JOIN project as p on e.id = p.empID;
```

**Result:** a very wide table containing **every** employee column **and** every project column — including `clientID`, which we never asked for.

**Column arithmetic (done explicitly on the iPad):**

| Table | Columns | List |
|---|---|---|
| `EMPLOYEE` | **7** | id, fname, lname, age, emailID, phoneNo, City |
| `Project` | **5** | id, empID, name, startdate, clientID |
| **Result** | **7 + 5 = 12** | all of both |

> **Rule: the joined table's column count = Left table columns + Right table columns.**

**The lesson:** *"Ye dikhane ke liye ki Join kaise kaam karta hai — Join saare columns ko match kar deta hai."* JOIN glues **all** columns together; the `SELECT` list is what you use to narrow it back down to just the information you actually want.

---

## K.3 Query 2 — INNER JOIN with a `WHERE` filter

> **Question:** *Fetch out all the employee IDs and their contact detail who have been working from **Jaipur** with the clients' names working in **Hyderabad**.*

**Decoding the requirement:**
- The **employee** must be based in **Jaipur**.
- The **client** they work with must be based in **Hyderabad**.
- Output: employee ID + contact details + client's name.

**Step 1 — write the join first:**

```sql
select e.id, e.emailID, e.phoneNo, c.first_name, c.last_name
from Employee as e
INNER JOIN Client as c on e.id = c.empID
```

This gives **complete information** — every employee paired with every client they work with. Too much.

**Step 2 — add the filter:**

```sql
WHERE e.city = 'Jaipur' AND c.city = 'Hyderabad';
```

**Complete query:**

```sql
select e.id, e.emailID, e.phoneNo, c.first_name, c.last_name
from Employee as e
INNER JOIN Client as c on e.id = c.empID
where e.city = 'Jaipur' AND c.city = 'Hyderabad';
```

**Dry run:**
1. Join `Employee.id = Client.empID`:
   - Client 1 (Mac, `empID` 3) → Rahul
   - Client 2 (Max, `empID` 3) → Rahul
   - Client 3 (Peter, `empID` 1) → Aman
   - Client 4 (Sushant, `empID` 5) → PK Pandey
   - Client 5 (Pratap, `empID` 2) → Yagya
2. Apply `e.city = 'Jaipur'` → only **PK Pandey** (employee 5) is in Jaipur.
3. Apply `c.city = 'Hyderabad'` → only **Sushant Aggarwal** (client 4) is in Hyderabad.
4. Both conditions are satisfied by exactly one joined row.

**Output:**

| e.id | e.emailID | e.phoneNo | c.first_name | c.last_name |
|---|---|---|---|---|
| 5 | pk@gmail.com | 555 | Sushant | Aggarwal |

**Verification on the raw tables:** Employee 5 = PK Pandey, city Jaipur ✔. The client whose `empID` = 5 is client id 4 = Sushant Aggarwal, city Hyderabad ✔.

> *"Dekho kitni specific information nikaali hai using Joins."*

### K.3a ⭐ `ON` vs `WHERE` — a crucial distinction

> *Added clarification — the instructor uses both correctly but never names the difference. This is a very common interview question.*

| Clause | Purpose |
|---|---|
| **`ON`** | Defines **how the two tables are linked** — the relationship/matching rule |
| **`WHERE`** | **Filters the rows of the joined result** — an ordinary condition applied afterwards |

Mental model: `ON` builds the combined table; `WHERE` then throws rows out of it.

⚠️ For an **INNER JOIN**, moving a condition between `ON` and `WHERE` happens to give the same answer. For **OUTER JOINS (LEFT/RIGHT) it does not** — a condition in `WHERE` will eliminate the `NULL`-padded rows that the OUTER JOIN just went to the trouble of preserving, silently turning your LEFT JOIN back into an INNER JOIN. Put join logic in `ON`, filters in `WHERE`.

---

## K.4 Query 3 — LEFT JOIN

> **Question:** *Fetch out each project allocated to each employee.*

**Intent:** list **every employee**, along with their project — including employees who have **no project at all**. INNER JOIN would silently hide those employees, which is precisely what we don't want.

```sql
select * from Employee as e
LEFT JOIN Project as p on e.id = p.empID;
```

**Here `Employee` is written first → Employee is the Left table → all employees are preserved.**

**Dry run — output (12 columns; abbreviated):**

| e.id | e.fname | e.lname | … | p.id | p.empID | p.name | p.startdate | p.clientID |
|---|---|---|---|---|---|---|---|---|
| 1 | Aman | Proto | … | 1 | 1 | A | 2021-04-21 | 3 |
| 2 | Yagya | Narayan | … | 2 | 2 | B | 2021-03-12 | 1 |
| 3 | Rahul | BD | … | 3 | 3 | C | 2021-01-16 | 5 |
| 3 | Rahul | BD | … | 4 | 3 | D | 2021-04-27 | 2 |
| **4** | **Jatin** | **Hermit** | … | **NULL** | **NULL** | **NULL** | **NULL** | **NULL** |
| 5 | PK | Pandey | … | 5 | 5 | E | 2021-05-01 | 4 |

**The headline result — the whole reason this query is a LEFT JOIN:**

> *"Jatin, jo employee ID 4 hai, Jatin Hermit — iske corresponding koi project nahi mil raha hai."*

**Jatin Hermit (employee 4) has no project allocated.** He appears in the output with `NULL`s across every Project column. Under an INNER JOIN he would have disappeared entirely and you would never have known he was unassigned.

> *"LEFT JOIN humne kiya tha, to aap dekh rahe ho Right side mujhe `NULL` mil raha hai."*

⭐ **This is the classic real-world use of LEFT JOIN: finding records that have NO match.** "Which employees have no project?", "which customers placed no orders?" — LEFT JOIN and look for the `NULL`s.

⚠️ *The instructor verbally says "focus Left, i.e. the Project table" — that's a slip; the query makes **Employee** the Left table. See Part M.3.*

---

## K.5 Query 4 — RIGHT JOIN

> **Question:** *List out all the projects along with the employee's name and their respective allocated email ID.*

**Intent:** every **project** must appear, whether or not it has an employee attached. Projects are the priority → put Project on the **right** and use RIGHT JOIN.

```sql
select p.id, p.name, e.fname, e.lname, e.emailID
FROM Employee as e
RIGHT JOIN Project as p ON e.id = p.empID;
```

**Dry run — output:**

| p.id | p.name | e.fname | e.lname | e.emailID |
|---|---|---|---|---|
| 1 | A | Aman | Proto | aman@gmail.com |
| 2 | B | Yagya | Narayan | yagya@gmail.com |
| 3 | C | Rahul | BD | rahul@gmail.com |
| 4 | D | Rahul | BD | rahul@gmail.com |
| 5 | E | PK | Pandey | pk@gmail.com |

**Reading the result:** all 5 projects (A, B, C, D, E) are listed, each with the employee working on it. **Rahul BD appears twice — on projects C and D** (he handles two projects).

⭐ **Subtle but important observation:** **there are no `NULL`s in this output.** Every project in this dataset has a valid `empID`, so nothing on the left is missing. **A RIGHT JOIN produces `NULL`s only when the right table actually contains unmatched rows.** Had there been a project with an `empID` not present in EMPLOYEE (or `NULL`), you would see `NULL` in the employee columns. On this data, the RIGHT JOIN result is identical to what an INNER JOIN would give — a good reminder that a join's *behaviour* and its *visible output on one dataset* are different things.

⚠️ *The instructor says "B, C and D are handled by Rahul BD" — B is actually Yagya's. See Part M.4.*

**Closing point (repeated from G.5):** LEFT and RIGHT JOIN are interchangeable by swapping table order. This same query could be written as `FROM Project as p LEFT JOIN Employee as e ON e.id = p.empID`.

---

## K.6 Query 5 — CROSS JOIN

> **Question:** *List out all the combinations possible for the employee's name and projects that can exist.*

```sql
select e.fname, e.lname, p.id, p.name
from Employee as e
CROSS JOIN Project as p;
```

**Note the absence of any `ON` clause** — CROSS JOIN takes no condition.

**Output size:** 5 employees × 5 projects = **25 rows.** Every employee paired with every project.

**Structure of the output:**

| e.fname | e.lname | p.id | p.name |
|---|---|---|---|
| Aman | Proto | 1 | A |
| Aman | Proto | 2 | B |
| Aman | Proto | 3 | C |
| Aman | Proto | 4 | D |
| Aman | Proto | 5 | E |
| Yagya | Narayan | 1 | A |
| … | … | … | … |
| PK | Pandey | 5 | E |

*(Each employee is exhausted against all 5 projects before moving to the next employee.)*

**Usefulness:** *"Iska industrial use kuch khas nahi hai."* The only scenario offered: when you genuinely want *every possible combination* — e.g. "for each person, a list of all projects."

---

# PART L — Master Comparison Table of All Joins

| # | Join | Keyword exists in MySQL? | Needs `ON` condition? | What it returns | Where `NULL`s appear | Venn region |
|---|---|---|---|---|---|---|
| 1 | **INNER JOIN** | ✅ Yes | ✅ Yes | Only rows where the key matches in **both** tables | Nowhere | Intersection only |
| 2 | **LEFT JOIN** | ✅ Yes | ✅ Yes | **All** left rows + matched right data | Right-side columns | Whole left circle |
| 3 | **RIGHT JOIN** | ✅ Yes | ✅ Yes | **All** right rows + matched left data | Left-side columns | Whole right circle |
| 4 | **FULL JOIN** | ❌ **No — emulate** | ✅ Yes | All rows from **both** tables | Both sides | Both circles entirely |
| 5 | **CROSS JOIN** | ✅ Yes | ❌ **No** | Every possible pairing (Cartesian product) | Nowhere | N/A — not a set operation |
| 6 | **SELF JOIN** | ❌ **No — emulate** | ✅ Yes | A table joined to itself | Depends on the join used | N/A |

**Emulation recipes:**

```sql
-- FULL JOIN (MySQL)
SELECT * FROM A LEFT  JOIN B ON A.key = B.key
UNION
SELECT * FROM A RIGHT JOIN B ON A.key = B.key;

-- SELF JOIN (any RDBMS)
SELECT ... FROM tbl AS t1 INNER JOIN tbl AS t2 ON t1.someCol = t2.otherCol;
```

**Row-count behaviour (very frequently tested):**

| Join | Result row count |
|---|---|
| INNER | ≤ matching rows; **can exceed** either table if a key repeats (1-to-many) |
| LEFT | ≥ number of left rows |
| RIGHT | ≥ number of right rows |
| CROSS | **exactly** rows(L) × rows(R) |

**Column-count behaviour:** for *every* join type, result columns = Left columns + Right columns (before you narrow it with `SELECT`).

---

# PART M — ⚠️ Instructor Errors / Things Stated Incorrectly

*Flagged so you learn the correct version, not the spoken one. The concepts are fine — these are execution slips.*

### M.1 — Numbering slip: two "5th" joins
The instructor calls **CROSS JOIN** "5th type" and then immediately calls **SELF JOIN** "5th type" as well. The on-screen notes repeat the mistake: both Cross Join and Self Join are labelled **⑤**.
**Correct numbering:** ① Inner, ② Left, ③ Right, ④ Full, ⑤ Cross, **⑥ Self**.

### M.2 — ⭐ The Self Join example is logically wrong (most important correction)
Shown on screen and read out:

```sql
select e1.id, e2.id, e2.name
FROM employee as e1
INNER JOIN employee as e2
ON e1.id = e2.id;          -- ❌ this does nothing useful
```

**Two problems:**

1. **`ON e1.id = e2.id` matches every row to itself.** Since `e1` and `e2` are the same table and `id` is the primary key, row 1 matches only row 1, row 2 only row 2, and so on. The result is just the original table with its columns duplicated — it reveals **nothing** about who manages whom. A self join is only meaningful when you join **two different columns**.

2. **There is no `name` column** in the `EMPLOYEE` schema shown (it has `fname` and `lname`).

**Corrected version.** Assuming the Employee table carries a `managerID` column (which is what makes `Employee manages Employee` a unary relation in the first place):

```sql
SELECT  e1.id           AS employee_id,
        e1.fname        AS employee_name,
        e2.id           AS manager_id,
        e2.fname        AS manager_name
FROM    employee AS e1
INNER JOIN employee AS e2
    ON  e1.managerID = e2.id;      -- ✔ e1's manager pointer → e2's identity
```

**Read it as:** "for each employee `e1`, find the row `e2` whose `id` equals `e1`'s `managerID` — that row *is* the manager."

*(Use `LEFT JOIN` instead of `INNER JOIN` if you also want to list the CEO, who has `managerID = NULL` and therefore no matching manager row.)*

**Still learn from the original:** the *structure* the instructor demonstrates — same table, two aliases, ordinary INNER JOIN — is exactly right. Only the `ON` condition is wrong.

### M.3 — LEFT JOIN question: "focus on Project" contradicts the query
For *"Fetch out each project allocated to each employee,"* the instructor says: *"mujhe focus karna hai Left, yaani **Project** table pe."* But the query he writes is:

```sql
select * from Employee as e LEFT JOIN Project as p on e.id = p.empID;
```

Here **`Employee` is written first, so Employee is the Left table** — the query preserves all *employees*, not all *projects*. And that is exactly what the demonstrated output shows (Jatin, an employee with no project, appears with `NULL`s). **The query is correct; the spoken sentence is a slip.** If you genuinely wanted all *projects* preserved, you'd write `FROM Project as p LEFT JOIN Employee as e …` (or the RIGHT JOIN of Query 4).

### M.4 — RIGHT JOIN result misread
He says: *"B, C aur D project ye Rahul BD naam ka banda kar raha hai"* — that Rahul handles projects B, C and D.
**Checking the data:** Project B has `empID = 2` → **Yagya Narayan**. Projects C and D have `empID = 3` → Rahul.
**Correct:** **Rahul handles C and D only** (two projects). B belongs to Yagya.

### M.5 — Cross Join example: left table drawn with 3 rows but counted as 2
The drawn left table is `(1,A), (1,B), (2,B)` — **three** rows — yet he concludes `2 × 2 = 4`, and the resultant table on screen only uses `(1,A)` and `(2,B)`.
**Consistent reading:** treat the left table as 2 rows → `2 × 2 = 4` output rows, matching the drawn result.
**If you instead take all 3 left rows,** the correct answer would be `3 × 2 = 6` rows:

| L1 | L2 | R1 | R2 |
|---|---|---|---|
| 1 | A | 3 | C |
| 1 | A | 4 | D |
| 1 | B | 3 | C |
| 1 | B | 4 | D |
| 2 | B | 3 | C |
| 2 | B | 4 | D |

**The formula `rows(L) × rows(R)` is what matters** — apply it to whatever the actual row counts are.

### M.6 — Transcript timestamp jumps
The transcript shows `2:50:22` and `2:50:45` appearing *after* `2:59:38`. These are transcription artefacts, not content issues. Content order is: Inner → Left → Right → Cross → closing question.

---

# PART N — Exam & Interview Traps

The instructor explicitly labels JOINS *"interviewer ka favorite topic."* These are the high-probability questions:

1. **"What's the difference between `REPLACE` and `UPDATE`?"**
   → If the row doesn't exist, REPLACE inserts it; UPDATE does nothing. **Bonus point that separates good answers:** even when the row *does* exist they differ — REPLACE deletes-and-reinserts, so any column you didn't list is reset to `NULL`/default, whereas UPDATE preserves them.

2. **"Does MySQL support `FULL JOIN`?"**
   → **No.** There is no `FULL JOIN` keyword in MySQL. Emulate with `LEFT JOIN` `UNION` `RIGHT JOIN`. (Follow-up: *why `UNION` and not `UNION ALL`?* → UNION deduplicates the intersection rows, which appear in both halves.)

3. **"Is there a `SELF JOIN` keyword?"**
   → **No.** It's an INNER JOIN of a table with itself, made possible by two different **aliases**. Used for **unary relations** (Employee manages Employee).

4. **"Which join is used most in practice?"**
   → **INNER JOIN.**

5. **"What is the prerequisite for applying a join?"**
   → A **common attribute** between the tables (typically the PK–FK relationship). *Exception:* CROSS JOIN needs none.

6. **"How do you find employees who have NO project?"**
   → `LEFT JOIN` and filter for `NULL` on the right side. This is *the* canonical real-world use of an OUTER JOIN.

7. **"LEFT JOIN vs RIGHT JOIN?"**
   → Mirror images; **interchangeable by swapping table order.** The *named* side is fully preserved; the *other* side receives the `NULL`s.

8. **"How many rows does a CROSS JOIN return?"**
   → `rows(Left) × rows(Right)`. Also: CROSS JOIN takes **no `ON` clause**.

9. **"Does a join increase rows or columns?"**
   → **Columns** (horizontal growth) = Left cols + Right cols. Contrast with **`UNION`**, which increases **rows** (vertical growth). *The instructor explicitly warns this confusion is coming.*

10. **"Can a join return more rows than either input table?"**
    → **Yes.** If one left row matches several right rows (1-to-many), it is duplicated in the output — exactly what happens to Rahul (employee 3) in Query 1.

11. **"`ON` vs `WHERE`?"**
    → `ON` defines the linking condition; `WHERE` filters the joined result. For OUTER JOINS the placement matters — a right-table condition in `WHERE` will kill the `NULL` rows and degrade your LEFT JOIN into an INNER JOIN.

12. **"Why do we need aliasing in joins?"**
    → Because two tables are now in scope and column names can collide; `alias.column` disambiguates (the `object.member` analogy from C++). For SELF JOIN it is not merely convenient but **mandatory**.

---

# PART O — Quick Revision Sheet

### REPLACE
- **Row exists →** replace (delete + insert). **Row doesn't exist →** insert.
- **UPDATE** on a non-existent row → does **nothing**.
- REPLACE resets any unlisted column to `NULL`/default; UPDATE preserves them.

### JOINS — one-line each
- **INNER** → only what matches, in both. *(Most used.)*
- **LEFT** → all of left + matches; `NULL` on the **right**.
- **RIGHT** → all of right + matches; `NULL` on the **left**.
- **FULL** → everything from both; **no MySQL keyword** → `LEFT UNION RIGHT`.
- **CROSS** → Cartesian product; **no `ON`**; `n × m` rows; little industrial use.
- **SELF** → table joined to itself via **two aliases**; **no keyword**; for unary relations.

### Syntax skeletons
```sql
-- INNER
SELECT col-list FROM t1 INNER JOIN t2 ON cond;
-- chained
SELECT col-list FROM t1 INNER JOIN t2 ON cond1 INNER JOIN t3 ON cond2;

-- LEFT / RIGHT
SELECT c.*, o.* FROM Customer AS c LEFT  JOIN Orders AS o ON c.id = o.cust_id;
SELECT c.*, o.* FROM Customer AS c RIGHT JOIN Orders AS o ON c.id = o.cust_id;

-- FULL (emulated)
SELECT * FROM l LEFT JOIN r ON l.key = r.key
UNION
SELECT * FROM l RIGHT JOIN r ON l.key = r.key;

-- CROSS
SELECT ... FROM A CROSS JOIN B;       -- no ON

-- SELF (emulated)
SELECT ... FROM emp AS e1 INNER JOIN emp AS e2 ON e1.managerID = e2.id;
```

### Rules to never forget
1. **Left = written first, Right = written second.** Nothing else determines it.
2. **Join → more COLUMNS. Union → more ROWS.**
3. Result columns = Left columns + Right columns.
4. A join needs a **common attribute** (except CROSS JOIN).
5. `alias.column` disambiguates — think `object.member`.
6. Aliases live **only for that one query**.

---

# 🔜 Open Question — Where the Lecture Ends

The segment closes on a cliffhanger slide:

```
Q Can we use JOIN w/o using JOIN key word?
```

> *"Yahan pe INNER JOIN ki baat ho rahi hai — kya hum INNER JOIN ko apply kar sakte hain `INNER JOIN` ka keyword use kare bina?"*

The instructor poses the question but the answer comes in the **next segment**. Keep it in mind — it is a common interview question in its own right.

*(One-line preview, flagged as an addition and not from this lecture: the answer is yes — an older comma-separated syntax, `FROM A, B WHERE A.id = B.a_id`, produces the same result as an INNER JOIN. Wait for the lecture for the full treatment.)*

---
