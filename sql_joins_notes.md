# SQL Complete Self-Study Notes
## Lecture Segment: `REPLACE` (Part 3) → `REPLACE vs UPDATE` → **JOINS** (Complete Chapter)

> **How to read these notes**
> Everything the instructor said out loud has been converted into full English explanations. Everything shown on screen (MySQL Workbench code, result grids, iPad handwritten notes, PDF notes, schema slides, Venn diagrams) has been reconstructed in text/table form.
>
> Two special markers are used:
> - **▶ Beyond the lecture** — extra material *not* said in the lecture, added because a first-time learner needs it (interview traps, engine internals, complexity). Clearly separated so you always know what came from the instructor and what did not.
> - **⚠ Correction** — places where the instructor's spoken words or handwritten notes contained a small slip. The slip is shown, then corrected, because you are studying from these notes alone and must not learn the mistake.

---

# TABLE OF CONTENTS

**PART A — Finishing the `REPLACE` Statement**
1. The Third `REPLACE` Syntax (`REPLACE ... SELECT`)
2. The `NULL` Side-Effect — the single most important behaviour of `REPLACE`
3. All Three `REPLACE` Syntaxes Consolidated
4. `REPLACE` vs `UPDATE`

**PART B — JOINS: Foundations**
5. Why JOINS Exist (the relational model and foreign keys)
6. Left Table / Right Table / Parent Table / Child Table — terminology
7. The Prerequisite Rule for Applying Any Join
8. Aliasing with the `AS` Keyword

**PART C — The Five Joins**
9. INNER JOIN
10. LEFT JOIN (LEFT OUTER JOIN)
11. RIGHT JOIN (RIGHT OUTER JOIN)
12. FULL JOIN (and why MySQL forces you to emulate it)
13. CROSS JOIN (Cartesian Product)
14. SELF JOIN (and why it too is emulated)

**PART D — Hands-On in MySQL Workbench**
15. The Three-Table Database Schema (`Employee`, `Project`, `Client`)
16. Practical Query 1 — INNER JOIN (employees + their projects)
17. Practical Query 2 — INNER JOIN + `WHERE` (Jaipur employees with Hyderabad clients)
18. Practical Query 3 — LEFT JOIN (the employee with no project)
19. Practical Query 4 — RIGHT JOIN (all projects with employee details)
20. Practical Query 5 — CROSS JOIN (all combinations)

**PART E — Consolidation**
21. Master Comparison Table of All Joins
22. Complexity Discussion
23. Complete List of Common Mistakes
24. Complete Interview Notes
25. The Cliffhanger Question — "Can we use JOIN without the JOIN keyword?"

---
---

# PART A — FINISHING THE `REPLACE` STATEMENT

---

# 1. The Third `REPLACE` Syntax (`REPLACE ... SELECT`)

## Concept

`REPLACE` is a MySQL statement that behaves like a smart `INSERT`. The lecture had already covered two ways of writing it, and this segment begins in the middle of demonstrating the **third** form: instead of typing literal values by hand, you **feed `REPLACE` the output of a `SELECT` query**.

The exact statement typed in MySQL Workbench was:

```sql
REPLACE INTO Customer(id, cname, City)
    SELECT id, cname, City
    FROM CUSTOMER WHERE id = 500;
```

Read it as two halves:

- **Right half (the data producer):** `SELECT id, cname, City FROM CUSTOMER WHERE id = 500` — this goes into the `CUSTOMER` table, finds the single row whose `id` is `500`, and pulls out three of its columns: the id, the customer name, and the city.
- **Left half (the data consumer):** `REPLACE INTO Customer(id, cname, City)` — whatever rows the `SELECT` produced are now pushed into `Customer`, and the three values land in the three named columns in that same order.

## Intuition — Why does this form exist at all?

Sometimes the data you want to insert **already lives inside the database**. Typing it out by hand would be pointless duplication, and worse, error-prone. So SQL lets a `SELECT` act as the source of rows. This is the same idea as `INSERT INTO ... SELECT`, just with `REPLACE`'s overwrite behaviour bolted on.

## The specific point the instructor stressed at the very start

The segment opens with the instructor saying, in effect:

> "The ID also has to be changed. So I will have to supply the ID here as well."

**Why does the ID have to be in the list?**

`REPLACE` decides whether a row already exists by looking at the **PRIMARY KEY** (or a `UNIQUE` key). If you never give it the primary key value, it has nothing to compare against, so it cannot detect a collision and it can only ever insert. The `id` column is therefore **not optional** in a meaningful `REPLACE` — it is the very thing the statement uses to make its decision.

That is why the column list is `(id, cname, City)` and the `SELECT` list is `id, cname, City` — the `id` had to be deliberately included in **both** lists.

## Step-by-step Process (how MySQL executes this statement)

1. Run the inner `SELECT` first. Scan `CUSTOMER`, filter with `WHERE id = 500`, keep only the columns `id`, `cname`, `City`. The result is one row: `(500, 'Rohan Arora', 'Colony')`.
2. Hand that row to `REPLACE`.
3. `REPLACE` checks: *does a row with primary key `500` already exist in `Customer`?* Yes, it does.
4. Because a collision exists, `REPLACE` **deletes the existing row `500` entirely**.
5. `REPLACE` then **inserts a brand-new row** using the values it was given.
6. The new row only has values for `id`, `cname`, and `City`. Every other column of the table was never supplied — so those columns receive `NULL`.

## Code Explanation — every clause

| Clause | Purpose |
|---|---|
| `REPLACE INTO` | The statement keyword. Means "insert, but if the primary/unique key already exists, wipe the old row first." |
| `Customer` | The **target** table — the table being written into. |
| `(id, cname, City)` | The **column list** — tells MySQL which columns the incoming values map to, and in what order. Position matters: the 1st selected value goes to `id`, the 2nd to `cname`, the 3rd to `City`. |
| `SELECT id, cname, City` | The **source column list**. Must produce the same number of columns, in the same order, with compatible data types as the target column list. |
| `FROM CUSTOMER` | The **source** table. Note it is the *same* table being written to — legal here, and it is exactly what makes this a "copy a row onto itself" demonstration. |
| `WHERE id = 500` | The filter that restricts the source to exactly one row. Without it, every row of the table would be replaced onto itself. |

## Dry Run

**Assume `Customer` before the statement** (the columns beyond the three selected ones are shown generically, because the lecture's table had more columns such as phone number, age, etc.):

| id | cname | City | (other columns…) |
|---|---|---|---|
| 500 | Rohan Arora | Colony | filled with real data |
| … | … | … | … |

**Execution trace:**

- **Step 1 —** `SELECT id, cname, City FROM CUSTOMER WHERE id = 500`
  Produces: `500 | Rohan Arora | Colony`
- **Step 2 —** `REPLACE INTO Customer(id, cname, City)` receives that one row.
- **Step 3 —** Primary key `500` found → **existing row deleted**.
- **Step 4 —** New row inserted with only three values populated.

**`Customer` after the statement** (this is exactly what the Result Grid on screen showed for `id = 500`):

| id | cname | City | (other columns…) |
|---|---|---|---|
| 500 | Rohan Arora | Colony | **NULL** |

The instructor read this off the screen as: the `id`, the customer name, and the city survived — because those three were explicitly selected — but *"all the remaining values that we did not give here were set to `NULL`."*

## ⚠ Why this looks harmless but is actually dangerous

Notice the trap. The visible three columns look **unchanged**. `500` is still `500`, `Rohan Arora` is still `Rohan Arora`, `Colony` is still `Colony`. A beginner glancing at the grid would conclude "nothing happened."

But something enormous happened: **every column you did not mention was silently destroyed and replaced with `NULL`.** Phone numbers, ages, addresses — gone.

This is not a bug. It is the definition of `REPLACE`: *delete the old row, insert a new one*. A new row can only contain what you gave it.

## The instructor's own verdict on this syntax

He stated plainly that **this third syntax is not particularly useful**. In his own day-to-day work he uses the first two forms. He then emphasised, in capital-letter tone, **"THIS IS VERY IMPORTANT"** — meaning the *set of three syntaxes as a whole* is exam- and interview-relevant, even though form three is rare in practice.

---

# 2. All Three `REPLACE` Syntaxes Consolidated

All three appeared in the Workbench editor on screen. Here they are together, since you will be asked to distinguish them.

```sql
-- Syntax 1: column list + VALUES
REPLACE INTO Customer(id, City)
    VALUES(1251, 'Colony');

-- Syntax 2: SET style (key = value pairs)
REPLACE INTO Customer SET id = 1300, cname = 'Mac', City = 'Utah';

-- Syntax 3: fed by a SELECT query
REPLACE INTO Customer(id, cname, City)
    SELECT id, cname, City
    FROM CUSTOMER WHERE id = 500;
```

### Syntax 1 — `VALUES` form
You name the columns in parentheses, then supply a matching parenthesised list of literal values. Positional. This is the classic form and mirrors `INSERT INTO ... VALUES`.

### Syntax 2 — `SET` form
You write `column = value` pairs directly. This is more readable when there are many columns, because the column name and its value sit next to each other rather than being separated by half a line. There is no positional matching to get wrong.

### Syntax 3 — `SELECT` form
Covered in detail above. Source rows come from a query rather than from your keyboard.

### Which one does the instructor actually use?

He said he normally uses the first two forms, and that syntax three is the least useful. Regardless of frequency, **all three must be known** — he explicitly flagged the topic as very important.

### ▶ Beyond the lecture — one behaviour worth knowing

Every one of these three forms shares the same `NULL` side-effect described in §1. It is not unique to the `SELECT` form. If you use syntax 1 to `REPLACE INTO Customer(id, City) VALUES(1251, 'Colony')` and row `1251` already existed with a name and a phone number, that name and phone number become `NULL` too. The rule is universal: **`REPLACE` never merges; it always destroys and re-creates.**

---

# 3. `REPLACE` vs `UPDATE`

This comparison was drawn on the iPad. The handwritten notes read:

```
Replace :-
1. Data already present  ->  Replace.
2. Data not present      ->  INSERT

Replace VS Update
-> if row is not present, Replace will add a new row
   while UPDATE will do nothing
```

## Concept

Both statements modify existing data, so beginners confuse them. The difference lives entirely in **what happens when the target row does not exist.**

## The two behaviours of `REPLACE`

1. **If the data is already present** → the old row is replaced by the new data you supplied.
2. **If the data is not present** → the row is simply **inserted** as a new row.

`REPLACE` therefore always leaves *something* behind. It cannot "do nothing."

## The behaviour of `UPDATE`

`UPDATE` operates by finding rows that satisfy a `WHERE` condition and modifying them. The instructor said, with emphasis — *"listen carefully"* —

> If the data is not present in `UPDATE`, then `UPDATE` will do absolutely nothing.

`UPDATE` has no ability to create rows. It only edits rows that already exist. If the `WHERE` clause matches zero rows, zero rows are affected and the database is unchanged. No error is raised — it simply reports "0 rows affected."

## The one-line summary the instructor dictated

> **If a row is not present, `REPLACE` will add a new row, while `UPDATE` will do nothing.**

Memorise this exact sentence. It is the interview answer.

## Comparison Table

| Aspect | `REPLACE` | `UPDATE` |
|---|---|---|
| Row already exists | Deletes old row, inserts new row | Modifies the matching row(s) in place |
| Row does not exist | **Inserts** it (acts like `INSERT`) | **Does nothing** — 0 rows affected |
| Can create data? | Yes | No |
| Columns you did not mention | Become **`NULL`** | Keep their **existing values** |
| Decision is based on | PRIMARY KEY / UNIQUE key collision | The `WHERE` clause |
| Mental model | "Destroy and rebuild the row" | "Edit the row in place" |

## ▶ Beyond the lecture — the practical consequence of row 4 in that table

The "columns you did not mention" row is the single biggest real-world difference and the reason experienced engineers reach for `UPDATE` far more often than `REPLACE`.

- `UPDATE Customer SET City='Colony' WHERE id=500;` → changes **only** the city. Everything else is untouched.
- `REPLACE INTO Customer(id, City) VALUES(500,'Colony');` → changes the city **and nullifies every other column.**

Use `REPLACE` only when you genuinely intend to overwrite the whole row.

## Common Mistakes

- **Assuming `REPLACE` is just "`UPDATE` that also inserts."** It is not. It is "`DELETE` + `INSERT`." The `NULL`-ing of unmentioned columns is the proof.
- **Omitting the primary key from a `REPLACE`.** Without it, `REPLACE` can never detect a collision, so it degenerates into a plain `INSERT` and you quietly get duplicate-looking data.
- **Expecting `UPDATE` to raise an error when nothing matched.** It does not. Silence in `UPDATE` means "no rows matched," which is easy to miss.

## Interview Notes

- **Q: What is the difference between `REPLACE` and `UPDATE`?** → Give the instructor's one-liner first (missing row ⇒ `REPLACE` inserts, `UPDATE` does nothing), then add the `NULL` side-effect as your second point. The second point is what distinguishes a strong candidate.
- **Q: How does `REPLACE` know whether the row exists?** → Through a PRIMARY KEY or UNIQUE index collision.
- **Q: Name the three `REPLACE` syntaxes.** → `VALUES` form, `SET` form, `SELECT` form.

---
---

# PART B — JOINS: FOUNDATIONS

The instructor introduced this section by saying: *"Now we discuss a very important topic — the interviewer's favourite topic — whose name is **JOINS**."*

Treat that sentence as a priority signal. Joins are the highest-yield SQL interview topic.

---

# 4. Why JOINS Exist

## Concept

The PDF notes displayed on screen said:

```
JOINING TABLES
1. All RDBMS are relational in nature, we refer to other tables to get
   meaningful outcomes.
2. FK are used to do reference to other table.
3. INNER JOIN ...
```

The iPad notes reinforced the same chain of reasoning:

```
# JOINS
-> F.K -> Relations
-> use -> Data fetch -> JOINS
```

## The logical chain, spelled out

Read this as a four-link chain. Each link answers "so what?" for the previous one.

**Link 1 — RDBMS are *relational* in nature.**
RDBMS stands for Relational Database Management System. The word "relational" is not decorative. It means: **I can refer from one table to another table.** Data is deliberately *not* stored in one giant table; it is split across many focused tables.

**Link 2 — Foreign keys are the mechanism that creates those references.**
The instructor asked you to recall foreign keys and their surrounding vocabulary: *referential constraints, referencing table, referenced table, child table, parent table*. A foreign key is a column in one table whose values are required to exist as key values in another table. That requirement **is** the relationship.

**Link 3 — So relationships now exist between tables.** Good. But relationships alone are inert. They constrain what data is *allowed* — they do not by themselves retrieve anything.

**Link 4 — To actually *fetch* data across those established relationships, you use JOINS.**

The instructor's own phrasing: *"We established relations with the help of the foreign key. Now, using those relations that we have established, how will I fetch data? To fetch this data, I use JOINS."*

## Intuition — Why is data split across tables in the first place?

Because storing everything in one table causes repetition. If every order row also stored the customer's full name, address, and phone number, then a customer with 50 orders would have their address written 50 times. Change the address once and you must change it in 50 places, and if you miss one, your data is now self-contradictory.

So the relational model splits the data: customers live in a `Customer` table, orders live in an `Orders` table, and each order merely *points* at its customer by storing the customer's id.

That split is what makes the data clean. **JOINS are the price you pay for that cleanliness** — they are the operation that stitches the pieces back together at query time, on demand, without ever duplicating storage.

## What happens if you do not use joins?

You would have to run one query to get customer ids, then loop in your application code running a second query per customer to fetch their orders. This is slow (many round-trips to the database), verbose, and reimplements in application code something the database engine can do far faster internally. Joins let the engine do it in one pass.

---

# 5. Terminology: Left Table, Right Table, Parent, Child

The instructor drew this on the iPad and asked you specifically to *"understand this terminology."*

### The drawing

```
Left table  (Parent Table)              Right Table  (Child Table)
L1 | L2 | Key (P.K)                     Key (F.K) | R1 | R2
---|----|----------                     ----------|----|---
   |    |    1                              1     |    |
   |    |    2                              2     |    |
   |    |    3                              3     |    |
   |    |    4
```

### Reading the diagram

- The **Left Table** has two ordinary data columns, `L1` and `L2`, plus a `Key` column. The instructor stated: *"What is this key here? That same **Primary Key**."*
- The **Right Table** has a `Key` column plus two ordinary data columns, `R1` and `R2`. He stated: *"And what is this key? Everyone knows — I made this a **foreign key**."*

### The four labels and how they map

| Term | Which table | Why |
|---|---|---|
| **Left Table** | The first one written in the query | Positional name only — whatever table appears first / on the left |
| **Right Table** | The second one written in the query | Positional name only |
| **Parent Table** | The one holding the PRIMARY KEY | It owns the original values; the other table depends on it |
| **Child Table** | The one holding the FOREIGN KEY | It borrows/references the parent's key values |
| **Referencing table** | Same as child | It does the referencing |
| **Referenced table** | Same as parent | It gets referenced |

The instructor's exact wording: *"the table inside which the foreign key is added, I also call that the referencing table or child table, and this other table — what is it? The **parent table**."*

## ⚠ A critical clarification you must not miss

In this particular diagram, Left happened to be the Parent and Right happened to be the Child. **That is a coincidence of the drawing, not a rule.**

- "Left" and "Right" describe **position in the query text**.
- "Parent" and "Child" describe **which side holds the primary key vs the foreign key**.

These are two independent classifications. You can absolutely write the child table first, making the child the Left table. In fact, in the Workbench practicals later in this very lecture, `Employee` (the parent, holding the primary key `id`) is written first as the Left table — but you could equally have written `Project` first.

This distinction becomes vital when you get to LEFT JOIN vs RIGHT JOIN, because those two joins care about *position*, not about *parenthood*.

## How the relationship actually works, value by value

The instructor walked through this slowly, and it is worth following just as slowly.

Put values `1, 2, 3` in the Left table's key column. Put values `1, 2, 3` in the Right table's key column.

Now note something he wanted you to consciously register:

> *"Look carefully — these two tables are stored differently in memory. Both are separate, both tables are separate."*

Physically, `Left` and `Right` are two independent structures on disk. There is no wire connecting them. Nothing is physically merged.

> *"But in some way there is a relation between them. What is establishing that relation? **This** — that corresponding to `1`, this is the data here."*

The *only* thing linking them is the shared key **value**. When the Left table's key is `1`, the entire Left row is "the data for key 1." When the Right table's key is also `1`, that entire Right row is *also* "data for key 1." Therefore both rows describe the same real-world entity, and can be legitimately combined.

Same for key `2`: everything on the left with key 2 plus everything on the right with key 2 all belongs to entity 2.

**That is the whole conceptual basis of a join:** matching values in the key columns tell you which rows describe the same thing.

---

# 6. The Prerequisite Rule for Applying Any Join

Written verbatim in the iPad notes:

```
Join -> To apply joins, there should be a common attribute
        between both the tables.
```

## Concept

You cannot join two arbitrary tables meaningfully. You need a **common attribute** — a column that exists in both tables (possibly under different names) and whose values mean the same thing in both.

The instructor's phrasing: *"If I have to apply joins, then I must have common attributes; the foreign key–primary key relation must be established, i.e. I must have a relation."*

## Intuition

Think of it as needing a shared vocabulary. If Table A identifies people by `employee_id` and Table B identifies them by `employee_id` too, you can line them up. If Table B only identifies people by fingerprint scan, there is no shared identifier and no way to match rows.

## What "common attribute" does and does not require

- The columns **do not need the same name.** In the practicals, `Employee.id` is joined to `Project.empID`. Different names, same meaning, join works fine.
- The columns **do need compatible data types and shared semantics.** Joining an integer id to an integer age would run without error and produce complete nonsense — the database will not stop you.
- ▶ **Beyond the lecture:** a declared FOREIGN KEY constraint is not *technically* mandatory for a join to execute. You can join any two columns. But the FK is what *guarantees* the values line up meaningfully, which is why the instructor frames the FK relation as the criterion.

## What happens if you ignore this rule?

If you omit the matching condition entirely, you do not get an error — you get a **CROSS JOIN**, every row paired with every row, an explosion of meaningless rows. This is covered in §13 and is one of the most common accidental bugs in SQL.

---

# 7. Aliasing with the `AS` Keyword

The instructor introduced `AS` here because it is unavoidable once two tables enter the same query. The iPad notes marked it explicitly:

```
Select c.*, o.* FROM Customer AS c INNER JOIN Orders AS o ON c.id = o.cust_id;
                                ^ Aliasing
```

## Concept

**Aliasing means giving a table (or a column) a short temporary nickname that is valid only inside the current query.**

## Why we need it — reason 1: brevity

The instructor's stated motivation: *"Because writing `Customer` looked very long, I simply did `AS c` and gave it the name `c`."*

Once aliased, `Customer` can be referred to as `c` everywhere in that query: `c.name`, `c.id`, `c.*`.

The alias is **scoped to that one query only.** It does not rename the table in the database. The next query knows nothing about `c`.

## Why we need it — reason 2 (the real reason): disambiguation

This is the point the instructor spent the most time on.

> *"Why am I writing `c.` here? Because two tables have arrived here. Until now our discussion was about only one table; now there are two tables. And if there are two tables, we will have to distinguish them, differentiate them."*

When only one table is in play, `SELECT id` is unambiguous — there is only one `id`. But the moment two tables are joined, and both have a column called `id`, the bare word `id` is **ambiguous**. The database cannot guess which one you mean, and will reject the query with an "ambiguous column" error.

Qualifying the column — `c.id` vs `o.id` — removes the ambiguity.

## The analogy the instructor used

> *"`c.` — just like you do in C++ on any object, on a customer object: `customer.id`, `customer.something`. In the same way, here `c.` means the customer table is being referred to."*

So the dot is a **member-access operator for tables**. `c.name` reads as "the `name` belonging to the customer table." `o.product_name` reads as "the `product_name` belonging to the orders table."

## The two things you can put after the dot

```sql
c.*                      -- every column of the customer table
c.id, c.name             -- specific columns of the customer table
o.product_name           -- a specific column of the orders table
```

`c.*, o.*` therefore means "all columns from customer, followed by all columns from orders" — which is the full joined row.

## ▶ Beyond the lecture — practical notes on `AS`

- The `AS` keyword is **optional for table aliases** in MySQL. `FROM Customer c` works identically to `FROM Customer AS c`. Writing `AS` is clearer for learners, so keep it.
- `AS` also aliases **columns**: `SELECT c.cname AS customer_name` renames the column in the output only.
- Aliases are **required**, not optional, for self joins (§14) — that is the one case where you literally cannot write the query without them.
- Once you define an alias, most SQL engines expect you to use it consistently; mixing `Customer.id` and `c.id` in the same query is bad style and in some engines an error.

## Common Mistakes

- Forgetting to qualify a column that exists in both tables → *"Column 'id' in field list is ambiguous."* This is one of the most frequent beginner errors when learning joins.
- Thinking `AS` permanently renames the table. It does not — it lasts exactly as long as the query.

---
---

# PART C — THE FIVE JOINS

The instructor numbered them ①–⑤ on the iPad in this order: **INNER, LEFT, RIGHT, FULL, CROSS**, then added **SELF JOIN** as a further type. Study them in that order — each one builds on the previous.

---

# 8. ① INNER JOIN

## Concept — the official definition

Straight from the PDF notes on screen:

> **INNER JOIN — Returns a resultant table that has matching values from both the tables, or all the tables.**

Break that sentence into its three assertions:

1. **"Returns a resultant table"** — a join does not modify your stored tables. It *generates a brand-new table* as its output. The instructor stressed this: *"the join operation you perform inside SQL — it generates a table for you, a new table is generated."* This output table exists only for the duration of the query result.
2. **"has matching values from both the tables"** — only rows whose key values exist on **both** sides survive.
3. **"or all the tables"** — the phrase *all the tables* is there because you can chain more than two tables into a single join. The PDF's syntax line shows this explicitly.

## Intuition — what "matching" actually means

The instructor's walkthrough:

Suppose the Left table's key column contains `1, 2, 3` and additionally `4`. Suppose the Right table (the child table) contains `1, 2, 3` — but has **no data at all for key `4`**.

> *"When I do an INNER JOIN, then the data that is common between the two — these common keys where they are matching — matching data means exactly this — that data alone will be visible to me. The data corresponding to `4` will not be visible to me."*

Restating it once more, exactly as he did:

> *"When I match these two tables — Left and Right — after matching I will see: which of these keys are matching? You will see `1, 2, 3` and `1, 2, 3` are matching. These are matching, so I will return only these tuples, only these rows."*

So INNER JOIN is fundamentally a **filter plus a merge**: filter to the rows that appear on both sides, then merge those rows side by side.

The instructor's closing summary of the idea: *"This is what I call INNER JOIN — that we have to pick up only the matching values from both tables. We dropped the `4`."*

## Step-by-step Process

1. Take the Left table and the Right table.
2. Compare them on the join condition — normally Left's key column against Right's key column.
3. Identify which key values exist in **both**.
4. For each such matching value, glue the Left row and the Right row together into a single wider row.
5. Discard any Left row whose key has no partner on the Right. Discard any Right row whose key has no partner on the Left.
6. Output the collection of glued rows as a new table.

## The resultant table (from the iPad drawing)

**Inputs:**

```
Left table                    Right Table
L1 | L2 | Key                 Key | R1 | R2
---|----|----                 ----|----|---
   |    |  1                    1 |    |
   |    |  2                    2 |    |
   |    |  3                    3 |    |
   |    |  4       <- no partner on the right
```

**Output of INNER JOIN:**

```
Key | L1 | L2 | R1 | R2
----|----|----|----|---
 1  |    |    |    |
 2  |    |    |    |
 3  |    |    |    |
```

Key `4` is **absent** from the output. It exists on the Left but has no matching partner on the Right, so INNER JOIN throws it away.

## The most important structural observation — columns grow

The instructor flagged this deliberately and told you to pay attention because it will cause confusion later:

> *"Look carefully at what we did — by applying INNER JOIN, we increased the number of columns."*

And later, when introducing LEFT JOIN, he warned again:

> *"Pay attention: inside JOIN we move column-wise, i.e. we increase the columns; inside the new table the columns have increased. Note this, because later when we study one more thing, confusion can be created there."*

**Write this down as a rule:**

| Operation | Direction of growth |
|---|---|
| **JOIN** | **Horizontal** — combines *columns*. Result is **wider**. |
| **UNION** (studied later) | **Vertical** — stacks *rows*. Result is **taller**. |

The "one more thing" he is warning about is `UNION`. Students constantly mix them up. A join widens; a union lengthens. In §12 you will see `UNION` used to emulate FULL JOIN, and the two ideas appear in the same query — so having this distinction clear beforehand is exactly why he warned you now.

**Column arithmetic:** if the Left table has *m* columns and the Right table has *n* columns, `SELECT *` on the join yields **m + n** columns. (This is verified numerically in the practical section, §16, where 7 + 5 = 12.)

## Venn Diagram (drawn on the iPad, also present in the PDF notes)

Two overlapping circles. The left circle is labelled **Left table**; the right circle is labelled **Right table**. **Only the overlapping lens-shaped intersection is hatched/shaded.**

```
        Left table              Right table
          ______                  ______
        /        \              /        \
       /          \ ///////////              \
      |            |/////////|                |
       \          / ///////////              /
        \________/              \________/
                  ↑
            shaded = INNER JOIN result
            (only rows present in BOTH)
```

The shaded intersection = the rows returned. Everything outside the intersection — Left-only rows and Right-only rows — is excluded.

The instructor: *"because inside this we extract matching values, I will say that when the key value matches inside both tables, only then will I answer — i.e. this area inside, these values, this is what will come in my answer."*

## Syntax

From the PDF notes:

```
SELECT column-list
FROM table1
INNER JOIN table2 ON condition1
INNER JOIN table3 ON condition2
```

Note the second `INNER JOIN` line — this is the *"or all the tables"* clause made concrete. You can keep chaining: join table1 to table2, then join that result to table3, each with its own `ON` condition.

## Worked Syntax Example (Customer / Orders)

The instructor rebuilt the earlier Customer–Orders example, writing it piece by piece.

```sql
SELECT c.*, o.*
FROM Customer AS c
INNER JOIN Orders AS o
ON c.id = o.cust_id;
```

### Line-by-line explanation

**`SELECT c.*, o.*`**
Select every column from the customer table followed by every column from the orders table. He explained why the output is wide: *"If I match these two tables, then in the new table I am getting, the columns have increased — that is why here too the columns will appear increased."*
He also noted you may select specific columns instead: `c.id, c.name, o.product_name`.

**`FROM Customer AS c`**
`Customer` is the **first** table named → it is the **Left table**. `AS c` gives it the alias `c`.

**`INNER JOIN Orders AS o`**
`Orders` is the **second** table named → it is the **Right table**. `AS o` gives it the alias `o`. The join type is INNER, so only matching rows survive.

**`ON c.id = o.cust_id`**
This is the **join condition** — the heart of the query. The instructor: *"Now I write next — I need a condition, because I do have to tell the query what to do it on. Just as here the key was the key, so I have to put a condition on the key: that the Left table's key should be equal to the Right table's key."*

- `c.id` — the primary key of `Customer` (the parent table)
- `o.cust_id` — the foreign key inside `Orders`

He then reminded you where `cust_id` came from: *"Remember, we had embedded the customer's ID inside Order. So Order here — Order was the child table."*

That is the FK–PK relationship from §4 being used as the join condition. The relationship you *declared* when designing the schema is now the relationship you *exploit* when querying.

## Left/Right assignment — the rule to memorise

> *"Customer is spoken first, so **Customer is the Left table**, and after that we say Orders, so **Orders is the Right table**."*
>
> *"The Left table will always come at the far left."*
>
> *"When you write the query, keep this thing carefully in mind — inside INNER JOIN, this is Left, this is Right. So keep Left–Right in mind."*

**Rule: the table named *before* the JOIN keyword is Left; the table named *after* it is Right.**

For INNER JOIN this ordering does not change the result set (only column order), because "matching on both sides" is symmetric. **But for LEFT JOIN and RIGHT JOIN it changes everything** — which is precisely why he drills it here, before you get there.

## Dry Run — INNER JOIN on the conceptual tables

**Left (`Customer`):**

| id | cname |
|---|---|
| 1 | Aman |
| 2 | Bela |
| 3 | Chirag |
| 4 | Divya |

**Right (`Orders`):**

| order_id | cust_id | product_name |
|---|---|---|
| 101 | 1 | Laptop |
| 102 | 2 | Mouse |
| 103 | 3 | Monitor |

**Trace of `ON c.id = o.cust_id`:**

| Step | Left row | Search Right for `cust_id = id` | Outcome |
|---|---|---|---|
| 1 | `1, Aman` | found `101, 1, Laptop` | **emit** `1, Aman, 101, 1, Laptop` |
| 2 | `2, Bela` | found `102, 2, Mouse` | **emit** `2, Bela, 102, 2, Mouse` |
| 3 | `3, Chirag` | found `103, 3, Monitor` | **emit** `3, Chirag, 103, 3, Monitor` |
| 4 | `4, Divya` | **no match** | **discard** |

**Result (3 rows, 5 columns):**

| c.id | c.cname | o.order_id | o.cust_id | o.product_name |
|---|---|---|---|---|
| 1 | Aman | 101 | 1 | Laptop |
| 2 | Bela | 102 | 2 | Mouse |
| 3 | Chirag | 103 | 3 | Monitor |

Divya is gone — the exact behaviour the instructor described for key `4`.

## Importance

> *"The most-used join is INNER JOIN itself. So you have to learn INNER JOIN really well."*

INNER JOIN is the default workhorse. Learn it first, learn it completely; the others are variations on it.

## Common Mistakes (INNER JOIN)

- **Forgetting the `ON` clause.** The condition is not optional in intent — omit it and you silently get a Cartesian product.
- **Assuming all Left rows will appear.** They will not. Unmatched rows on either side vanish. If a report is "missing people," an INNER JOIN is usually why.
- **Confusing join (widens) with union (lengthens).**
- **Mixing up Left and Right.** Harmless for INNER, fatal for LEFT/RIGHT.
- **Joining on the wrong columns** — e.g. `ON c.id = o.id` instead of `ON c.id = o.cust_id`. Both compile. Only one is correct.

## Interview Notes (INNER JOIN)

- Give the definition **word-for-word**: *returns a resultant table that has matching values from both the tables.*
- Be able to **draw the Venn diagram** — intersection shaded only.
- State the prerequisite: *to apply joins there must be a common attribute between both the tables.*
- Know that `INNER JOIN` produces `m + n` columns with `SELECT *`.
- Know that a join **produces a new table** and does not alter the originals.
- Be ready to chain three tables using two `INNER JOIN ... ON ...` clauses.

---

# 9. ② LEFT JOIN (LEFT OUTER JOIN)

## Where it sits — the "OUTER JOIN" family

The instructor introduced it like this:

> *"Next come **OUTER JOINs**. You will understand from the name, right? It means we will try to go a little **outside** the matching values."*

That framing is the key to the whole family. INNER JOIN stays strictly *inside* the intersection. OUTER JOINs deliberately step **outside** it to also collect non-matching rows. LEFT, RIGHT, and FULL are the three ways of stepping outside.

## Concept — the official definition

> **LEFT JOIN — This returns a resulting table that has all the data from the left table, and the matched data from the right table.**

Two independent promises in one sentence:

1. **All** the data from the Left table — unconditionally, matched or not.
2. Only the **matched** data from the Right table.

## Intuition

Think of the Left table as the **guest list you must honour completely**. Every left row *will* appear in the output — that is non-negotiable. The Right table is then consulted as a lookup: "do you have anything for this key?" If yes, its columns are attached. If no, the right-hand columns are filled with `NULL`.

The instructor anticipated the obvious student objection and answered it directly:

> *"Now a simple question arises — you will be thinking, but these do not even match. How will data come corresponding to `5`? So inside LEFT JOIN, what happens is: corresponding to `5`, I will get **`NULL` values** in the right table; I will get `NULL` data."*

**`NULL` is the padding used to keep the row rectangular.** A table must have the same number of columns in every row. If the right side has nothing to contribute for that key, SQL must still put *something* in those column positions — and that something is `NULL`, SQL's marker for "no value exists here."

## Venn Diagram

Two overlapping circles. **The entire Left circle is shaded — including the intersection.** The Right-only crescent is unshaded.

```
        Left table              Right table
       //////////                 ______
     ////////////\              /        \
    //////////////|############|          |
     /////////////              \        /
       //////////                 ------
    ↑ all shaded (all left rows + matched right data)
```

The instructor: *"If Left table is this and Right table is this, then I will do the whole Left join, i.e. I will bring all the values of Left."*

## Full worked example (from the iPad drawing)

**Inputs:**

```
Left Table                    R. Table
L1 | L2 | Key                 Key | R1 | R2
---|----|----                 ----|----|---
   |    |  1                    1 |    |
   |    |  2                    2 |    |
   |    |  3                    3 |    |
   |    |  4       <- no partner
   |    |  5       <- no partner
   |    |  6       <- no partner
```

Left has keys `1..6`. Right has only `1, 2, 3`.

**Resultant Table:**

```
Key | L1 | L2 | R1   | R2
----|----|----|------|------
 1  |    |    | Data | Data
 2  |    |    | Data | Data
 3  |    |    | Data | Data
 4  |    |    | Null | Null
 5  |    |    | Null | Null
 6  |    |    | Null | Null
```

The instructor's narration of exactly this table:

> *"Corresponding to `1, 2, 3` I will have matching data, i.e. there will be full data here, full data here, and full data here too. But corresponding to `4, 5, 6`, I will have the Left table's data — data will be there — **but here it will be `NULL`**."*

He then checked the output against the definition:

> *"The same thing the definition said is being honoured, isn't it? It said 'this returns a resulting table that has all the data from left table' — I have all the data of the left table — 'and the matched data from the right table' — the matching data from the right table has also been put in, and corresponding to what is not matching, `NULL` values have come."*

**Notice the row count: 6 rows out.** LEFT JOIN never loses a left row. An INNER JOIN on this same data would have produced only 3.

## Syntax

```sql
SELECT c.*, o.*
FROM Customer AS c
LEFT JOIN Orders AS o
ON c.id = o.cust_id;
```

The instructor's own comment on this:

> *"This is roughly its syntax — simple. There is nothing to do; everything is the same, only `LEFT JOIN` is written here."*

**That is a genuinely important observation.** Syntactically, the only edit is one keyword. `INNER JOIN` → `LEFT JOIN`. Everything else — the `SELECT` list, the aliases, the `ON` condition — is untouched. All the difference is in *semantics*, not in *shape*.

## Dry Run — LEFT JOIN

**Left (`Customer`):**

| id | cname |
|---|---|
| 1 | Aman |
| 2 | Bela |
| 3 | Chirag |
| 4 | Divya |

**Right (`Orders`):**

| order_id | cust_id | product_name |
|---|---|---|
| 101 | 1 | Laptop |
| 102 | 2 | Mouse |
| 103 | 3 | Monitor |

**Trace:**

| Step | Left row | Right match? | Emitted row |
|---|---|---|---|
| 1 | `1, Aman` | yes | `1, Aman, 101, 1, Laptop` |
| 2 | `2, Bela` | yes | `2, Bela, 102, 2, Mouse` |
| 3 | `3, Chirag` | yes | `3, Chirag, 103, 3, Monitor` |
| 4 | `4, Divya` | **no** | `4, Divya, NULL, NULL, NULL` ← **kept, padded** |

**Result (4 rows):**

| c.id | c.cname | o.order_id | o.cust_id | o.product_name |
|---|---|---|---|---|
| 1 | Aman | 101 | 1 | Laptop |
| 2 | Bela | 102 | 2 | Mouse |
| 3 | Chirag | 103 | 3 | Monitor |
| 4 | Divya | NULL | NULL | NULL |

Compare with the INNER JOIN dry run in §8: identical inputs, but Divya survives here. **That single row is the entire difference between INNER and LEFT.**

## ▶ Beyond the lecture — the anti-join pattern (very high interview value)

Because unmatched left rows are padded with `NULL`, you can find *exactly* the rows that have **no** partner:

```sql
SELECT c.*
FROM Customer AS c
LEFT JOIN Orders AS o ON c.id = o.cust_id
WHERE o.cust_id IS NULL;
```

This returns "customers who have never placed an order." It is called an **anti-join**, and it is one of the most frequently asked practical join questions in interviews. The trick: only unmatched rows can have `NULL` in a right-hand column, so testing for `NULL` isolates them.

## ▶ Beyond the lecture — `WHERE` can secretly destroy your LEFT JOIN

```sql
-- Intended: all customers, with orders where they exist
SELECT * FROM Customer c
LEFT JOIN Orders o ON c.id = o.cust_id
WHERE o.product_name = 'Laptop';   -- ⚠ silently becomes an INNER JOIN
```

The padded rows have `o.product_name = NULL`, and `NULL = 'Laptop'` is not true, so `WHERE` filters those rows straight back out. Your LEFT JOIN has degenerated into an INNER JOIN.

**Fix:** put the condition in the `ON` clause instead, where it filters *before* padding:

```sql
LEFT JOIN Orders o ON c.id = o.cust_id AND o.product_name = 'Laptop'
```

**Rule:** for OUTER joins, `ON` and `WHERE` are **not** interchangeable. (For INNER joins they are.) This is a classic interview trap.

## Common Mistakes (LEFT JOIN)

- Believing the Left table means the parent table. It does not — it means whichever table you wrote first.
- Being surprised by `NULL`s. They are the expected, designed behaviour.
- Filtering right-table columns in `WHERE` and unknowingly converting to INNER JOIN.
- Comparing with `= NULL` instead of `IS NULL`. `NULL` is never equal to anything, including itself.

## Interview Notes (LEFT JOIN)

- Definition, word-for-word: *all the data from the left table and the matched data from the right table.*
- Venn diagram: entire left circle shaded.
- Unmatched left rows are padded with `NULL` on the right side.
- Row count ≥ INNER JOIN's row count, always.
- Know the anti-join pattern and the `ON` vs `WHERE` trap.
- `LEFT JOIN` and `LEFT OUTER JOIN` are identical; `OUTER` is optional.

---

# 10. ③ RIGHT JOIN (RIGHT OUTER JOIN)

## Concept

> *"Third, I have **RIGHT JOIN**. RIGHT JOIN is just the opposite of LEFT JOIN."*

The iPad heading captured its purpose in one line:

```
③ Right Join
- Interested in knowing only the right side info
```

**Definition (by symmetry with LEFT JOIN):** RIGHT JOIN returns a resulting table containing **all the data from the right table**, plus the **matched data from the left table**. Where the left side has no match, the left columns are filled with `NULL`.

## Intuition

Mirror everything you learned about LEFT JOIN. The instructor made the symmetry explicit:

> *"Just as the Left table's definition was — in Left we were saying all of Left's data plus the matching data with Right — now in this one we will say: all of Right's data, all of Right's data, and the matching data too."*

Now the **Right table is the guest list you must honour completely.** Every right row appears; the left table is consulted only as a lookup.

## Venn Diagram

Two overlapping circles. **The entire Right circle is shaded — including the intersection.** The Left-only crescent is unshaded.

```
        Left table              Right table
          ______                //////////
        /        \            //////////////
       |          |##########|//////////////|
        \        /             //////////////
          ------                //////////
                                    ↑ all shaded
```

## Full worked example (from the iPad drawing)

Right has keys `1, 2, 3, 4, 5`. Left has only `1, 2, 3`.

**Resultant Table:**

```
Key | L1   | L2   | R1 | R2
----|------|------|----|----
 1  | Data | Data |    |
 2  | Data | Data |    |
 3  | Data | Data |    |
 4  | Null | Null |    |
 5  | Null | Null |    |
```

The instructor's narration:

> *"If we go by the definition — all of the Right table's data: `1, 2, 3, 4, 5` — so I will have all of the Right table's data here. Here I have all the data, and I will have the matching data of both; the matching data of both is this, so we have put the matching data in. **But** the non-matching data — like corresponding to `4, 5` there is no entry in the Left table — that will be replaced by `NULL` here."*

**Note the `NULL`s have moved to the LEFT-hand columns.** That is the entire visual difference from the LEFT JOIN table.

## The summary sentence the instructor gave

> *"So look carefully, you will now understand the difference a little: LEFT JOIN and RIGHT JOIN are opposites of each other. **In LEFT JOIN I get `NULL` in the data on the right side, and in RIGHT JOIN I get `NULL` in the data on the left side.**"*

That sentence is the cleanest possible one-line distinction. Memorise it.

## Syntax

```sql
SELECT c.*, o.*
FROM Customer AS c
RIGHT JOIN Orders AS o
ON c.id = o.cust_id;
```

Again, only the keyword changes.

## Dry Run — RIGHT JOIN

**Left (`Customer`):**

| id | cname |
|---|---|
| 1 | Aman |
| 2 | Bela |

**Right (`Orders`):**

| order_id | cust_id | product_name |
|---|---|---|
| 101 | 1 | Laptop |
| 102 | 2 | Mouse |
| 104 | 9 | Keyboard |

*(Note: `cust_id = 9` does not exist in `Customer` — this is the deliberately orphaned right-hand row.)*

**Trace (driven by the Right table now):**

| Step | Right row | Left match on `id = cust_id`? | Emitted row |
|---|---|---|---|
| 1 | `101, 1, Laptop` | yes → Aman | `1, Aman, 101, 1, Laptop` |
| 2 | `102, 2, Mouse` | yes → Bela | `2, Bela, 102, 2, Mouse` |
| 3 | `104, 9, Keyboard` | **no** | `NULL, NULL, 104, 9, Keyboard` ← padded on the **left** |

**Result (3 rows):**

| c.id | c.cname | o.order_id | o.cust_id | o.product_name |
|---|---|---|---|---|
| 1 | Aman | 101 | 1 | Laptop |
| 2 | Bela | 102 | 2 | Mouse |
| NULL | NULL | 104 | 9 | Keyboard |

## LEFT and RIGHT are interchangeable — the instructor's important remark

He raised this later, during the Workbench session, but it belongs here conceptually:

> *"And if you look carefully, LEFT JOIN and RIGHT JOIN get used interchangeably — because when you are using a LEFT JOIN, the table you are treating as the Left table, if you make it the Right table, if you write it on the right, then it will be used like a RIGHT JOIN. So they can be used interchangeably."*

### What this means concretely

These two queries return **exactly the same rows**:

```sql
SELECT * FROM A LEFT  JOIN B ON A.k = B.k;
SELECT * FROM B RIGHT JOIN A ON A.k = B.k;
```

Swapping the table order *and* flipping the join direction cancel each other out. (The **column order** in the output differs, because the `FROM` table's columns come first — but the set of rows is identical.)

### ▶ Beyond the lecture — the practical consequence

Because of this equivalence, **RIGHT JOIN is technically redundant.** Any RIGHT JOIN can be rewritten as a LEFT JOIN by swapping the tables. Many teams therefore have a style rule: *always use LEFT JOIN, never RIGHT JOIN*, because a query that consistently reads left-to-right ("start from this table, then attach these") is easier to reason about — especially once three or four tables are chained.

You still must **know** RIGHT JOIN for interviews and for reading other people's code.

## Common Mistakes (RIGHT JOIN)

- Expecting `NULL`s on the right side. They are on the **left** side.
- Assuming a RIGHT JOIN must produce `NULL`s. It only produces them if some right row genuinely has no left partner. (You will see exactly this in practical Query 4 — a RIGHT JOIN with *zero* `NULL`s, because every project had a valid employee.)
- Chaining LEFT and RIGHT joins in one multi-table query — the mixture is extremely hard to reason about and a common source of wrong results.

## Interview Notes (RIGHT JOIN)

- One-liner: *LEFT JOIN gives `NULL`s on the right; RIGHT JOIN gives `NULL`s on the left.*
- Venn diagram: entire right circle shaded.
- Know the interchangeability rewrite and be able to demonstrate it.
- `RIGHT JOIN` = `RIGHT OUTER JOIN`.

---

# 11. ④ FULL JOIN (FULL OUTER JOIN)

## Concept

FULL JOIN returns **everything from both tables**: all matching rows, plus all unmatched left rows (padded with `NULL` on the right), plus all unmatched right rows (padded with `NULL` on the left). Nothing is ever discarded.

## The instructor's build-up (a recap of all three joins so far)

Before defining FULL JOIN, he re-derived the previous three, and this recap is worth reading carefully because it is the cleanest summary in the entire lecture:

> *"We saw INNER JOIN, when I want the common values — the common matching values. And I saw LEFT JOIN; in LEFT JOIN, I definitely want all of the Left's data, and along with it the matching data too — but corresponding to Left there will be no data in Right, and that is why I get `NULL`s in Right."*
>
> *"Similarly, in RIGHT JOIN my entire focus is on Right, i.e. I want all the data corresponding to Right; I definitely want the matching data and along with it all of Right's data — so corresponding to Right, where there is no data in Left, I get `NULL`s in Left."*
>
> *"Similarly, what happens in FULL JOIN? **FULL JOIN is nothing but the UNION of LEFT and RIGHT JOIN**, i.e. I merged LEFT and RIGHT JOIN, added them together in a way."*

## Intuition — why "LEFT ∪ RIGHT" equals FULL

Think about what each piece contributes:

- **LEFT JOIN** = (matching rows) + (left-only rows padded right)
- **RIGHT JOIN** = (matching rows) + (right-only rows padded left)

Add them: you get matching rows **twice**, plus left-only rows, plus right-only rows.

`UNION` (as opposed to `UNION ALL`) **removes duplicates**. So the doubled matching rows collapse back into one copy, and you are left with exactly:

- matching rows (once)
- left-only rows
- right-only rows

= **FULL JOIN**. The mathematics works out perfectly, and this is *why* the emulation is valid rather than just an approximation.

The instructor's statement of the outcome:

> *"I do a LEFT JOIN first, and I also do a RIGHT JOIN, and I take the UNION of both, so I get all the values. That is, I will get all the data corresponding to the Left table — even if it is not in Right, I will still get it — and I will also get all the data of the Right table, even if it is not in Left, I will still get it."*

## Venn Diagram

Two overlapping circles with **both circles entirely shaded** — the intersection and both crescents.

## Full worked example (from the iPad)

The instructor set up a deliberately illustrative dataset:

```
Left table                    Right table
L1 | L2 | Key                 Key | R1 | R2
---|----|----                 ----|----|---
   |    |  1                    1 |    |     <- matching
   |    |  2                    2 |    |     <- matching
   |    |  3                    3 |    |     <- matching
   |    |  4                    7 |    |     <- right-only
   |    |  5                    8 |    |     <- right-only
   |    |  6                    9 |    |     <- right-only
      ↑ left-only
```

His description: *"In Left there are matching values, and there are also values here for which there is no corresponding data there, and there are values here for which there is no corresponding data here."*

### What each join would return on this data

He then compared all four on the same input — an excellent revision exercise:

| Join | Rows returned | Which keys |
|---|---|---|
| **INNER JOIN** | 3 | only `1, 2, 3` |
| **LEFT JOIN** | 6 | `1, 2, 3` (full) + `4, 5, 6` (right cols = `NULL`) |
| **RIGHT JOIN** | 6 | `1, 2, 3` (full) + `7, 8, 9` (left cols = `NULL`) |
| **FULL JOIN** | **9** | `1, 2, 3` + `4, 5, 6` + `7, 8, 9` |

His narration for each:
- INNER: *"I will get only this data, i.e. these whole rows."*
- LEFT: *"I will definitely get all this data, and along with it this data too... corresponding to `4` there is no data there, so I will get `NULL` on the right side."*
- RIGHT: *"I will get all of Right's data... along with that, corresponding to `7, 8, 9` I will get `NULL` entries in Left."*
- FULL: *"I will extract the LEFT JOIN, extract the RIGHT JOIN, and what will I do with them? I will take their UNION."*

### The final FULL JOIN result table

```
Key | L1   | L2   | R1   | R2
----|------|------|------|------
 1  | Data | Data | Data | Data
 2  | Data | Data | Data | Data
 3  | Data | Data | Data | Data
 4  | Data | Data | NULL | NULL
 5  | Data | Data | NULL | NULL
 6  | Data | Data | NULL | NULL
 7  | NULL | NULL | Data | Data
 8  | NULL | NULL | Data | Data
 9  | NULL | NULL | Data | Data
```

The instructor: *"I will see all the values corresponding to `1, 2, 3`; `4, 5, 6` will also appear and there I will see `NULL` values; `7, 8, 9` will appear and there I will get `NULL` values."*

## ⚠ THE CRITICAL FACT — MySQL has no FULL JOIN

Said with unusual emphasis:

> *"Inside MySQL there is **no keyword available at all** called FULL JOIN. **NO, NOT AVAILABLE.** So what do we do — we **emulate** it."*

### What "emulate" means

To emulate is to *reproduce a feature's behaviour using other features that do exist.* MySQL does not give you `FULL JOIN` as a keyword, so you build the same result out of parts MySQL does have: `LEFT JOIN`, `RIGHT JOIN`, and `UNION`.

This is the second time in this lecture the word "emulate" appears — SELF JOIN (§14) is also emulated. Notice the pattern: **not every conceptual join has a dedicated keyword.**

### ▶ Beyond the lecture — is this a MySQL-only limitation?

Yes. **PostgreSQL, SQL Server, Oracle, and SQLite (3.39+) all support `FULL OUTER JOIN` natively.** MySQL is the notable holdout. So this is a *MySQL implementation gap*, not a limitation of SQL as a language. If an interviewer asks "how do you do a FULL JOIN?", the complete answer is: *"In most RDBMSs you just write `FULL OUTER JOIN`. In MySQL that keyword does not exist, so you emulate it with `LEFT JOIN UNION RIGHT JOIN`."*

## The Emulation Syntax

From the iPad notes:

```sql
select * from lefttable as l
LEFT JOIN Righttable as r ON l.key = r.key
UNION
select * from lefttable as l
RIGHT JOIN Righttable as r ON l.key = r.key;
```

### Explanation of every part

**First `SELECT`** — a normal LEFT JOIN. Produces: matching rows + left-only rows padded with `NULL` on the right.

**`UNION`** — the vertical stacking operator. It takes two result sets and puts one **below** the other. Critically, plain `UNION` **eliminates duplicate rows**.

**Second `SELECT`** — a normal RIGHT JOIN over the same two tables with the same `ON` condition. Produces: matching rows + right-only rows padded with `NULL` on the left.

**Net effect** — matching rows appear in both halves, `UNION` deduplicates them to one copy, and the two exclusive sets are added. Result = FULL JOIN.

### ▶ Beyond the lecture — three requirements for `UNION` to work

1. **Same number of columns in both `SELECT`s.** Here both are `select *` over the same two tables joined in the same order, so both produce `l`'s columns followed by `r`'s columns. Counts match. ✔
2. **Compatible data types, column by column, in order.** Same reasoning. ✔
3. **`UNION`, not `UNION ALL`.** `UNION ALL` skips deduplication and is faster — but here it would return every matching row **twice**, which is wrong. **You must use plain `UNION`.**

This is exactly the confusion the instructor pre-warned you about back in §8:

> **JOIN grows the table sideways (columns). UNION grows the table downwards (rows).**

In this one query you are using **both at once**: the joins widen, the union lengthens.

## Dry Run — the emulation

**Left (`l`):** keys `1, 2, 4` — **Right (`r`):** keys `1, 2, 7`

**Half 1 — LEFT JOIN:**

| l.key | l.data | r.key | r.data |
|---|---|---|---|
| 1 | L1 | 1 | R1 |
| 2 | L2 | 2 | R2 |
| 4 | L4 | NULL | NULL |

**Half 2 — RIGHT JOIN:**

| l.key | l.data | r.key | r.data |
|---|---|---|---|
| 1 | L1 | 1 | R1 |
| 2 | L2 | 2 | R2 |
| NULL | NULL | 7 | R7 |

**Stack them (6 rows), then deduplicate:** rows `(1,L1,1,R1)` and `(2,L2,2,R2)` each appear twice → each collapses to one.

**Final (4 rows):**

| l.key | l.data | r.key | r.data |
|---|---|---|---|
| 1 | L1 | 1 | R1 |
| 2 | L2 | 2 | R2 |
| 4 | L4 | NULL | NULL |
| NULL | NULL | 7 | R7 |

Exactly a FULL JOIN. ✔

## Common Mistakes (FULL JOIN)

- **Writing `FULL JOIN` in MySQL.** Syntax error. It does not exist.
- **Using `UNION ALL` instead of `UNION`** → matching rows duplicated.
- **Mismatched column lists** between the two halves → error, or worse, silently misaligned data.
- **Forgetting the semicolon placement** — the `;` goes after the *second* `SELECT`, because the two `SELECT`s plus the `UNION` are one single statement.

## Interview Notes (FULL JOIN)

- *"MySQL does not support FULL JOIN; it must be emulated with `LEFT JOIN UNION RIGHT JOIN`."* — this is a very commonly asked question.
- Explain **why** the emulation is mathematically correct (dedup collapses the doubled intersection).
- Be able to state: `UNION` = dedupe, `UNION ALL` = keep duplicates, and why that distinction matters *here specifically*.
- Venn diagram: both circles fully shaded.
- Row-count relationship: `|FULL| = |LEFT| + |RIGHT| − |INNER|`.

---

# 12. ⑤ CROSS JOIN (Cartesian Product)

## Concept

> *"CROSS JOIN is nothing but the **Cartesian Product**."*

A CROSS JOIN pairs **every** row of the left table with **every** row of the right table. There is **no condition** at all.

## Intuition — how it differs from every join so far

This is the sentence that captures it:

> *"Just as INNER JOIN, LEFT JOIN, RIGHT JOIN — we did those on some condition, right? On some condition, as in we were doing `Customer.id = Order.cust_id`, i.e. we were trying to match keys. **Here we do not match keys or anything at all** — simply picked up the Left table, picked up the Right table, and made all the possible values that can be made."*

So CROSS JOIN is the *only* join with **no `ON` clause**. There is nothing to match on, because it is not trying to match anything. It generates every possible pairing.

## The mathematics

If the Left table has *m* rows and the Right table has *n* rows, the result has:

$$\text{rows}_{\text{result}} = m \times n$$

**Explanation of the symbols:**
- **m** — number of rows in the left table (`T_L`)
- **n** — number of rows in the right table (`T_R`)
- **×** — ordinary multiplication, because each of the *m* left rows is paired with each of the *n* right rows independently

**Column count** follows the same rule as every other join: *(left columns) + (right columns)*.

The example from the iPad notes:

```
-> Cartesian product
T_L (5 rows)  x  T_R (10 rows)  =>  Resultant (5 * 10 = 50 rows)
```

The instructor: *"So in this case how many resultant values will be formed? `5 * 10 = 50` rows will be formed."*

## ▶ Beyond the lecture — why "Cartesian product"?

The name comes from set theory (after René Descartes). The Cartesian product of sets A and B, written A × B, is the set of all ordered pairs (a, b) where a ∈ A and b ∈ B. A CROSS JOIN is literally this operation applied to rows. If A has 5 elements and B has 10, A × B has 50 — same arithmetic.

## The small example from the iPad — with a correction

The handwritten notes showed:

```
Example:
L1 | L2          R1 | R2
---|---          ---|---
 1 | A            3 | C
 1 | B            4 | D
 2 | B

Resultant:
L1 | L2 | R1 | R2
---|----|----|---
 1 | A  | 3  | C
 1 | A  | 4  | D
 2 | B  | 3  | C
 2 | B  | 4  | D
```

### ⚠ Correction — the notes are internally inconsistent

Look at the left table: it has **three** rows — `(1,A)`, `(1,B)`, `(2,B)`. The right table has **two** rows.

By the rule, the result must have **3 × 2 = 6 rows**. But the written resultant shows only **4**, and the instructor said *"here there were 2 rows, here there were 2 rows, `2 × 2 = 4`, so 4 rows came in my output."*

He was counting the left table as 2 rows while having drawn 3. **The row `(1, B)` was accidentally skipped.** The two missing output rows are `(1, B, 3, C)` and `(1, B, 4, D)`.

**The correct resultant table (6 rows):**

| L1 | L2 | R1 | R2 |
|---|---|---|---|
| 1 | A | 3 | C |
| 1 | A | 4 | D |
| 1 | B | 3 | C |
| 1 | B | 4 | D |
| 2 | B | 3 | C |
| 2 | B | 4 | D |

**The rule `m × n` is correct and is what you must remember.** The arithmetic in the notes was a slip of counting, not a flaw in the concept. The instructor's *method* — described as *"first we matched these two rows corresponding to `1 A`, then matched these two rows"* — is exactly right; it just needed to be applied a third time.

### How to generate a Cartesian product by hand

Use nested iteration:

```
for each row L in LeftTable:          <- outer loop
    for each row R in RightTable:     <- inner loop
        output (L columns, R columns)
```

Left row 1 pairs with all right rows, then left row 2 pairs with all right rows, and so on. This is why the output shows `1 A` twice consecutively, then `1 B` twice, then `2 B` twice.

## Syntax

```sql
SELECT e.fname, e.lname, p.id, p.name
FROM Employee AS e
CROSS JOIN Project AS p;
```

**Notice: there is no `ON` clause.** Adding one would be meaningless — and in MySQL, adding `ON` to a `CROSS JOIN` actually turns it into an INNER JOIN, since MySQL treats `CROSS JOIN`, `JOIN`, and `INNER JOIN` as syntactic equivalents.

## Industrial usefulness — the instructor's verdict

The iPad notes recorded a blunt one-liner:

```
- Industrial use: X
```

> *"If I talk about the Cartesian product, it has no industrial use; it is not particularly useful."*

And after running it in Workbench:

> *"Its industrial use is nothing special. But suppose I have to do something like: corresponding to every person I need a list of all the projects — all the possible combinations that can be made — then I can use something like this. But I think this does not get used much."*

**So: know it, be able to compute its row count, but do not expect to write one at work.**

## ⚠ Why CROSS JOIN still matters even though nobody wants one

Because **you can produce one by accident.** If you join two tables and *forget the `ON` condition*, or write a condition that is always true, you get a Cartesian product. On two tables of 100,000 rows each, that is **10,000,000,000 rows** — the query hangs, memory fills, the server may fall over.

This is why "always check your join condition" is standard advice. An accidental CROSS JOIN is one of the most destructive beginner mistakes in SQL.

## Interview Notes (CROSS JOIN)

- Definition: Cartesian product — every row of A paired with every row of B.
- Row formula: **m × n**. Be ready to compute it instantly.
- It is the **only join with no `ON` condition**.
- Its danger: accidental Cartesian products from a missing join condition.
- Legitimate niche uses (▶ beyond the lecture): generating all date × store combinations for a report grid, creating test data, building calendar tables.

---

# 13. SELF JOIN

*(The instructor labelled this "5th type" too — after CROSS JOIN — so treat it as a sixth item in the list. The iPad heading reads `⑤ Self Join`.)*

## Concept

A **SELF JOIN** is a join of a table **with itself**.

The iPad notes:

```
⑤ Self Join
- Unary Relation: Employee manages Employee
- Emulate using 'INNER JOIN' and Alias 'AS'
```

## Why would anyone join a table to itself? — the Unary Relation

The instructor connected this back to ER-modelling theory:

> *"Note carefully — we once talked about a **Unary Relation**. What was the unary relation like? That **Employee manages Employee**."*

### What a unary (recursive) relationship is

Most relationships connect **two different** entity types: an Employee works on a Project; a Customer places an Order. A **unary** (also called recursive or self-referencing) relationship connects an entity type **to itself**.

The classic example is management. A manager *is* an employee. So the relationship "manages" runs from `Employee` back to `Employee`.

The instructor: *"So in this case, what was it? That on the Employee table itself, basically an Employee table of this type — such a relation is sitting on it."*

### How it is stored

You do **not** create a separate `Manager` table — that would duplicate every manager's details. Instead, the `Employee` table gets an extra column, typically `manager_id`, which is a foreign key pointing back at `Employee.id` *in the same table*.

| id | name | manager_id |
|---|---|---|
| 1 | Aman | NULL ← the boss; nobody manages them |
| 2 | Yagya | 1 ← managed by Aman |
| 3 | Rahul | 1 ← managed by Aman |
| 4 | Jatin | 2 ← managed by Yagya |

## When do you need it?

> *"Many times, in this same case, if I have to apply Employee on Employee itself and extract things — I have to find out **who is whose manager** — then for this case I use a SELF JOIN."*

The problem: employee 4's row tells you their manager is `2`, but `2` is just a number. You want the manager's *name*. The name lives in a different row of the *same* table. A single-table scan cannot look at two rows simultaneously and combine them into one output row — but a **join** can, if you treat the table as two tables.

## ⚠ SELF JOIN is also emulated — there is no `SELF JOIN` keyword

> *"Now for Self Join too there is no operator; this is also emulated. How do we emulate it? By using **`INNER JOIN`** and by using **Alias**, using the **`AS`** keyword."*

**There is no `SELF JOIN` keyword in SQL.** "Self join" is a *description of what you are doing*, not a syntax element. You write an ordinary `INNER JOIN` (or `LEFT JOIN`) where both sides happen to be the same table, and **aliases** are what make it legal.

## How the trick works — "fooling the software"

The instructor's phrasing is memorable:

> *"Now virtually something like this is happening — an attempt is being made to **fool the software**. It is the same table, but we created two references of it, made two aliases. Now these two are behaving like two different tables. `e1` is the Left table, `e2` is the Right table."*

### Why aliases are mandatory here

Without aliases you would write `FROM employee INNER JOIN employee ON employee.id = employee.id` — and every single column reference is hopelessly ambiguous. Which `employee` does `employee.id` mean? The database cannot tell, and rejects the query.

Aliasing gives the two copies **distinct names**, `e1` and `e2`. From that moment on, `e1.id` and `e2.id` are unambiguous, and the engine treats the two aliases as two independent inputs — even though both read from the same physical file on disk.

**This is the one situation where aliasing is not a convenience but an absolute requirement.**

## The example from the lecture

```sql
select e1.id, e2.id, e2.name
FROM employee as e1
INNER JOIN employee as e2
ON e1.id = e2.id;
```

Line by line:
- `FROM employee as e1` — first copy of the table, aliased `e1`, playing the **Left table**.
- `INNER JOIN employee as e2` — the *same* table again, aliased `e2`, playing the **Right table**.
- `ON e1.id = e2.id` — the join condition linking the two copies.
- `select e1.id, e2.id, e2.name` — pick columns from each copy, distinguished by alias.

The instructor: *"So in this way I can use a Self Join here too — so this too is emulated, there is no separate keyword for it."*

## ⚠ Correction — this example is structurally right but semantically empty

The **mechanism** shown (same table, two aliases, INNER JOIN) is 100% correct and is exactly what you must learn. But the **condition chosen** makes the query do nothing useful, and you should understand why so you do not copy it blindly.

**The problem:** `ON e1.id = e2.id` joins each row to **itself**. Since `id` is the primary key, row 3 in `e1` can only ever match row 3 in `e2`. The output is just the original table with some columns repeated:

| e1.id | e2.id | e2.name |
|---|---|---|
| 1 | 1 | Aman |
| 2 | 2 | Yagya |
| 3 | 3 | Rahul |

No manager information appears anywhere. The join is a no-op.

**Two further notes on this snippet:**
- The `Employee` table used in this lecture's practical schema has **no `name` column** (it has `fname` and `lname`), and **no manager column** at all — so this query would not even run against that schema. It was a whiteboard sketch of the *pattern*, not a runnable query.

**What the manager query should actually look like:**

```sql
SELECT  e1.id        AS employee_id,
        e1.name      AS employee_name,
        e2.id        AS manager_id,
        e2.name      AS manager_name
FROM    employee AS e1
INNER JOIN employee AS e2
        ON e1.manager_id = e2.id;
```

**The key change: `ON e1.manager_id = e2.id`.** You match the *foreign key column* of one copy against the *primary key column* of the other copy. That is what makes the two copies play **different roles** — `e1` is "the employee," `e2` is "that employee's manager."

### Dry Run of the corrected self join

**`employee`:**

| id | name | manager_id |
|---|---|---|
| 1 | Aman | NULL |
| 2 | Yagya | 1 |
| 3 | Rahul | 1 |
| 4 | Jatin | 2 |

Mentally make two copies:

**`e1` (the employee side)** and **`e2` (the manager side)** — both contain all four rows.

| Step | `e1` row | `e1.manager_id` | Find `e2` row with `id` = that | Emit |
|---|---|---|---|---|
| 1 | 1 Aman, mgr NULL | NULL | `NULL = anything` is never true → no match | **dropped** (INNER JOIN) |
| 2 | 2 Yagya, mgr 1 | 1 | `e2` row 1 = Aman | `2, Yagya, 1, Aman` |
| 3 | 3 Rahul, mgr 1 | 1 | `e2` row 1 = Aman | `3, Rahul, 1, Aman` |
| 4 | 4 Jatin, mgr 2 | 2 | `e2` row 2 = Yagya | `4, Jatin, 2, Yagya` |

**Result:**

| employee_id | employee_name | manager_id | manager_name |
|---|---|---|---|
| 2 | Yagya | 1 | Aman |
| 3 | Rahul | 1 | Aman |
| 4 | Jatin | 2 | Yagya |

**Edge case worth noting:** Aman, the top-level boss, **disappears** because their `manager_id` is `NULL` and `NULL` matches nothing under `INNER JOIN`. If you want the boss included with a `NULL` manager name, switch to `LEFT JOIN`. This is a favourite interview follow-up, and it is a perfect illustration of why the INNER-vs-LEFT distinction is not academic.

## Common Mistakes (SELF JOIN)

- Looking for a `SELF JOIN` keyword. **There is none.**
- Omitting aliases → ambiguous column errors.
- Joining `id = id` (as in the lecture sketch) instead of `foreign_key = id` → each row joins to itself and nothing useful is produced.
- Using `INNER JOIN` and then wondering why the top-level manager vanished.
- ▶ Forgetting `e1.id <> e2.id` in "find pairs" style self joins (e.g. "find all pairs of employees in the same city") — without it, every person is paired with themselves.

## Interview Notes (SELF JOIN)

- Definition: a table joined to itself using aliases.
- It is **emulated**, using `INNER JOIN` + `AS` — no dedicated keyword.
- The motivating case is a **unary / recursive relationship**, e.g. *Employee manages Employee*.
- Aliases are **mandatory**, not optional, because both sides share every column name.
- Classic interview question: *"Find every employee along with their manager's name."* Be able to write it from memory, and know that `LEFT JOIN` is needed to retain the top-level manager.
- Other classic uses: find duplicate rows; find pairs of rows sharing an attribute; compare a row against the previous row.

---
---

# PART D — HANDS-ON IN MySQL WORKBENCH

The instructor's transition:

> *"So in this way we have studied Joins. Now let us run it once on the Workbench and see their use cases... But before that, we made a schema just now; let us understand that schema."*

---

# 14. The Three-Table Database Schema

Three tables were shown on a slide: `Project`, `EMPLOYEE`, and `CLIENT`.

## Table 1: `Project`

| id | empID | name | startdate | clientID |
|---|---|---|---|---|
| 1 | 1 | A | 2021-04-21 | 3 |
| 2 | 2 | B | 2021-03-12 | 1 |
| 3 | 3 | C | 2021-01-16 | 5 |
| 4 | 3 | D | 2021-04-27 | 2 |
| 5 | 5 | E | 2021-05-01 | 4 |

- `id` — primary key of the project (5 projects, named A–E)
- `empID` — **foreign key** → `EMPLOYEE.id`. Which employee works on this project.
- `name` — project name
- `startdate` — when the project began
- `clientID` — **foreign key** → `CLIENT.id`. Which client the project is for.
- **5 columns total** (this number is used in the column arithmetic later)

**Read the data:** project A is done by employee 1; project B by employee 2; projects **C and D are both done by employee 3**; project E by employee 5. **Employee 4 appears nowhere** — this is the fact that makes the LEFT JOIN demo interesting.

## Table 2: `EMPLOYEE`

| id | fname | lname | age | emailID | phoneNo | City |
|---|---|---|---|---|---|---|
| 1 | Aman | Proto | 32 | aman@gmail.com | 898 | Delhi |
| 2 | Yagya | Narayan | 44 | yagya@gmail.com | 222 | Palam |
| 3 | Rahul | BD | 22 | rahul@gmail.com | 444 | Kolkata |
| 4 | Jatin | Hermit | 31 | jatin@gmail.com | 666 | Raipur |
| 5 | PK | Pandey | 21 | pk@gmail.com | 555 | Jaipur |

- `id` — **primary key**
- `fname`, `lname`, `age`, `emailID`, `phoneNo`, `City` — attribute columns
- **7 columns total**, **5 rows**

The instructor: *"we have Employees; Employees `1, 2, 3` — these are their employee IDs — `fname`, `lname`, `age`, `emailID`, `phoneNo`, and `city`."*

## Table 3: `CLIENT`

| id | first_name | last_name | age | emailID | phoneNo | City | empID |
|---|---|---|---|---|---|---|---|
| 1 | Mac | Rogers | 47 | mac@hotmail.com | 333 | Kolkata | 3 |
| 2 | Max | Poster | 27 | max@gmail.com | 222 | Kolkata | 3 |
| 3 | Peter | Jain | 24 | peter@abc.com | 111 | Delhi | 1 |
| 4 | Sushant | Aggarwal | 23 | sushant@yahoo.com | 45454 | Hyderabad | 5 |
| 5 | Pratap | Singh | 36 | p@xyz.com | 77767 | Mumbai | 2 |

- `id` — **primary key** of the client
- `empID` — **foreign key** → `EMPLOYEE.id`. Which employee handles this client.
- **8 columns**, **5 rows**

Note the column names differ from `EMPLOYEE`: `first_name`/`last_name` here vs `fname`/`lname` there. Real schemas are inconsistent like this, and you must read carefully.

## The relationships — as walked through in the lecture

The instructor traced them by example:

> *"Now, which projects are these Employees working on? For that, look carefully — in Project the Project ID is `1, 2, 3, 4`. Now this [`empID`] is acting like a foreign key here, to tell us that employee `1` is working on project ID `1` whose name is A. Who is employee `1` here? **Aman** — so Aman is working on this project."*

> *"Similarly, here is Client... Now I am saying that this particular employee — like here employee `1` is Aman — Aman is working with this client. Aman is working with client **Peter**, whose ID is `3`."*

> *"So here `empID` is a foreign key, here `empID` is a foreign key, and inside Project, `clientID` is a foreign key."*

### Relationship map

```
                 EMPLOYEE (id = PK)
                    ▲          ▲
                    │          │
       Project.empID│          │Client.empID
                    │          │
                 PROJECT     CLIENT (id = PK)
                    │          ▲
                    └──────────┘
                    Project.clientID
```

**Three foreign keys in total:**

| FK column | Lives in | Points to | Meaning |
|---|---|---|---|
| `Project.empID` | Project | `Employee.id` | which employee works on the project |
| `Project.clientID` | Project | `Client.id` | which client the project is for |
| `Client.empID` | Client | `Employee.id` | which employee handles the client |

**Notice there are two different paths from Employee to Client:** directly via `Client.empID`, or indirectly via `Project` (`Project.empID` → `Project.clientID`). This matters in Query 2 — see the note there.

## All five queries as typed into the Workbench editor

```sql
-- INNER JOIN
-- Enlist all the employees ID's, names along with the Project allocated to them.
select e.id, e.fname, e.lname, p.id, p.name from employee as e
INNER JOIN project as p on e.id = p.empID;

-- Fetch out all the employee ID's and their contact detail who have been working
-- from Jaipur with the clients name working in Hyderabad.
select e.id, e.emailID, e.phoneNo, c.first_name, c.last_name from Employee as e
INNER JOIN Client as c on e.id = c.empID where e.city = 'Jaipur' AND c.city = 'Hyderabad';

-- LEFT JOIN
-- Fetch out each project allocated to each employee.
select * from Employee as e
LEFT JOIN Project as p on e.id = p.empID;

-- RIGHT JOIN
-- List out all the projects along with the employee's name and their respective allocated email ID.
select p.id, p.name, e.fname, e.lname, e.emailID FROM Employee as e
RIGHT JOIN Project as p ON e.id = p.empID;

-- CROSS JOIN
-- List out all the combinations possible for the employee's name and projects that can exist.
select e.fname, e.lname, p.id, p.name from Employee as e
CROSS JOIN Project as p;
```

---

# 15. Practical Query 1 — INNER JOIN

## The question

> *"Enlist all the employees' IDs, names along with the Project allocated to them."*

Decoded: *"I need the employee ID, the employee's name, and correspondingly which project they are working on."*

## Choosing the tables and the join column

The instructor's reasoning process — follow this, it is the reusable method:

> *"I will have to use two tables: I have to use Employee, and I have to use Project. Now in Employee and Project, look at what is common — in what way can I apply a Join? So here I have `id`, and corresponding to it, here is the foreign key `empID`. So if I apply an INNER JOIN on these, then I will get which employee is working on which project."*

**The method:**
1. Identify which tables hold the data you need. (Names → `Employee`; project names → `Project`.)
2. Find the common attribute linking them. (`Employee.id` ↔ `Project.empID`.)
3. Choose the join type based on whether unmatched rows should appear. (INNER here — we only want employees who *have* projects.)

## The query

```sql
select e.id, e.fname, e.lname, p.id, p.name
from employee as e
INNER JOIN project as p on e.id = p.empID;
```

- `employee AS e` → **Left table**
- `project AS p` → **Right table**
- `ON e.id = p.empID` → parent PK matched against child FK
- Selected: employee id, first name, last name, project id, project name
- **Both `e.id` and `p.id` are selected** — a perfect demonstration of why aliases matter. Written unqualified, `id` would be ambiguous and the query would fail.

## Dry Run (against the real data)

| `e` row | `e.id` | Look for `p.empID = e.id` | Output rows |
|---|---|---|---|
| Aman Proto | 1 | project 1 (A) | `1, Aman, Proto, 1, A` |
| Yagya Narayan | 2 | project 2 (B) | `2, Yagya, Narayan, 2, B` |
| Rahul BD | 3 | project 3 (C) **and** project 4 (D) | `3, Rahul, BD, 3, C` **and** `3, Rahul, BD, 4, D` |
| Jatin Hermit | 4 | **none** | **dropped** |
| PK Pandey | 5 | project 5 (E) | `5, PK, Pandey, 5, E` |

## Output (5 rows)

| e.id | e.fname | e.lname | p.id | p.name |
|---|---|---|---|---|
| 1 | Aman | Proto | 1 | A |
| 2 | Yagya | Narayan | 2 | B |
| 3 | Rahul | BD | 3 | C |
| 3 | Rahul | BD | 4 | D |
| 5 | PK | Pandey | 5 | E |

The instructor read the result off screen:

> *"Employee ID `1`, whose name is Aman, is working on project A. Employee ID `2`, who is Yagya, is working on project B. Employee ID `3`, Rahul — now look at Rahul — **Rahul is working on two projects, C and D**. Rahul is working on both projects."*

## Two crucial observations from this output

### (a) One left row can produce multiple output rows

Rahul appears **twice**. This is not a bug and not duplicate data — Rahul genuinely has two projects, so the join emits one row per (employee, project) pair.

**Generalisation:** in a one-to-many relationship, joining multiplies the "one" side. Row counts do **not** simply equal the left table's row count. This surprises beginners constantly, and it is why joins can unexpectedly inflate aggregates (e.g. `SUM(salary)` counting Rahul's salary twice).

### (b) Jatin is absent

Employee 4 (Jatin Hermit) has no project, so INNER JOIN drops him. **Remember this** — it is precisely the row that reappears in the LEFT JOIN demo in §18.

## The `SELECT *` experiment and the column arithmetic

The instructor then deliberately re-ran the query replacing the column list with `*`:

> *"If you look carefully — if in its place I write `*`, then look how big a table comes. Why am I showing this? To show how Join works: **Join matches all the columns**. Look, it has produced a new table for me, of the kind where I have all the details of the employees, and corresponding to that all the details of the projects — even the `clientID` is there in it."*

Then he counted, on the iPad, out loud:

> *"So how many columns do I have here? `1, 2, 3, 4, 5, 6, 7`. And how many columns here? `1, 2, 3, 4, 5`. So how many columns came here? `1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12`, i.e. **`7 + 5 = 12`** columns came."*

$$\text{columns}_{\text{result}} = \text{columns}_{\text{left}} + \text{columns}_{\text{right}} = 7 + 5 = 12$$

**Where the numbers come from:**
- `EMPLOYEE` has 7 columns: `id, fname, lname, age, emailID, phoneNo, City`
- `Project` has 5 columns: `id, empID, name, startdate, clientID`

With `SELECT *`, the joined row is: `e.id, e.fname, e.lname, e.age, e.emailID, e.phoneNo, e.City, p.id, p.empID, p.name, p.startdate, p.clientID` — 12 columns, left table's columns first.

**This numerically proves the "joins grow horizontally" rule from §8.**

## Why he then went back to the explicit column list

> *"But what did I want? I wanted **selected information** — that is why we wrote a query like this. This is my answer."*

The lesson: `SELECT *` is useful for *understanding* what a join produces, but in real queries you name the columns you actually want. `SELECT *` returns clutter, transfers unnecessary data over the network, and breaks if the table's schema changes later.

## Interview Notes from Query 1

- Be able to explain why one employee appears on multiple rows (one-to-many fan-out).
- Be able to state the column-count formula and derive it.
- Know why `e.id` and `p.id` must be qualified.
- Be able to predict, before running, which rows will be *missing* under INNER JOIN.

---

# 16. Practical Query 2 — INNER JOIN with a `WHERE` Clause

## The question

> *"Fetch out all the employee IDs and their contact detail who have been working from Jaipur with the clients name working in Hyderabad."*

The instructor decoded it phrase by phrase:

> *"I need employee IDs, I need their contact details, of those who work **from Jaipur** — and whom do they work with? They work with those clients who are working **from Hyderabad**. That is, where is the client working from — Hyderabad — and where is the employee working from — Jaipur."*

**Two independent filters on two different tables:**
- Employee's city must be Jaipur → filter on `Employee.City`
- Client's city must be Hyderabad → filter on `Client.City`

This is the query's teaching point: **a join lets you filter on columns from more than one table in a single query.** Without a join you would need two queries and manual cross-referencing.

## Step 1 — write the join first

```sql
select e.id, e.emailID, e.phoneNo, c.first_name, c.last_name
from Employee as e
INNER JOIN Client as c on e.id = c.empID
```

He ran this intermediate version deliberately:

> *"Now let us see what happens with this. Now look carefully — with this I have got all the information, complete information. **But** what do I need?"*

The result at this stage is all 5 employee–client pairings — far too much. This is a genuinely good working habit: **build the join first, verify it, then add filters.** If you write everything at once and it is wrong, you cannot tell whether the join or the filter is at fault.

## Step 2 — add the `WHERE` clause

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

### Explanation of the `WHERE` clause

- `e.city = 'Jaipur'` — keep only rows whose *employee-side* city is Jaipur
- `AND` — **both** conditions must hold simultaneously
- `c.city = 'Hyderabad'` — keep only rows whose *client-side* city is Hyderabad
- Both column references are **alias-qualified**, and here it is unavoidable: both tables have a `City` column, so bare `city` would be ambiguous.
- String literals go in **single quotes**. Note also that MySQL string comparison is case-insensitive by default under standard collations, which is why `e.city` (lowercase in the query) matches the `City` column, and `'Jaipur'` matches the stored `Jaipur`.

### Order of operations

1. `FROM` + `INNER JOIN ... ON` — build the joined result set
2. `WHERE` — filter that result set
3. `SELECT` — project the chosen columns

The `WHERE` runs **after** the join, on the combined rows. That is what makes it possible to reference `e.city` and `c.city` in the same condition.

## Dry Run

**Step A — the join `ON e.id = c.empID`:**

| Client | `c.empID` | matching Employee | joined pair |
|---|---|---|---|
| 1 Mac Rogers (Kolkata) | 3 | Rahul BD (Kolkata) | Rahul ↔ Mac |
| 2 Max Poster (Kolkata) | 3 | Rahul BD (Kolkata) | Rahul ↔ Max |
| 3 Peter Jain (Delhi) | 1 | Aman Proto (Delhi) | Aman ↔ Peter |
| 4 Sushant Aggarwal (Hyderabad) | 5 | PK Pandey (Jaipur) | PK ↔ Sushant |
| 5 Pratap Singh (Mumbai) | 2 | Yagya Narayan (Palam) | Yagya ↔ Pratap |

5 joined rows.

**Step B — apply `WHERE e.city='Jaipur' AND c.city='Hyderabad'`:**

| Joined pair | `e.city` | = Jaipur? | `c.city` | = Hyderabad? | Kept? |
|---|---|---|---|---|---|
| Rahul ↔ Mac | Kolkata | ✗ | Kolkata | ✗ | no |
| Rahul ↔ Max | Kolkata | ✗ | Kolkata | ✗ | no |
| Aman ↔ Peter | Delhi | ✗ | Delhi | ✗ | no |
| **PK ↔ Sushant** | **Jaipur** | **✓** | **Hyderabad** | **✓** | **YES** |
| Yagya ↔ Pratap | Palam | ✗ | Mumbai | ✗ | no |

**Step C — project the selected columns.**

## Output (1 row)

| e.id | e.emailID | e.phoneNo | c.first_name | c.last_name |
|---|---|---|---|---|
| 5 | pk@gmail.com | 555 | Sushant | Aggarwal |

The instructor confirmed it on screen:

> *"OK, so the employee ID is `5`, its email ID has come — PK — and he is working with a client named **Sushant Aggarwal**."*

He then verified it manually against the tables on the iPad:

> *"The employee ID was PK — i.e. Pandey ji, PK Pandey — what is he doing? He is working from **Jaipur**, right? And whom is he working with? If we look at ID `5`, then... he is working with clientID `4`, and clientID `4` is **Sushant**, who is working from **Hyderabad**."*

> *"So by matching things like this — see how much specific information we extracted using Joins."*

## ▶ Beyond the lecture — a subtlety in the verification

When verifying, the instructor traced the relationship through the **`Project`** table (`Project` row 5 has `empID = 5` and `clientID = 4`). But the **query** used a different path: `Client.empID = 5`.

Both paths lead to the same answer here — Sushant is linked to PK both directly (`Client.empID = 5`) and indirectly (`Project` row 5 links employee 5 to client 4). But they are logically **different relationships**:

- `Client.empID` = "which employee is the account handler for this client"
- `Project.empID` + `Project.clientID` = "which employee works on a project *for* this client"

In a larger dataset these could disagree. **Lesson:** when a schema offers more than one path between two tables, be explicit about which relationship your question is actually asking about. Choosing the wrong path produces answers that look plausible and are wrong.

## Common Mistakes from Query 2

- Writing `where city = 'Jaipur'` without an alias → ambiguous column error (both tables have `City`).
- Using `OR` instead of `AND` → returns any Jaipur employee *or* any Hyderabad client, a much larger and incorrect set.
- Putting the filter in the `ON` clause. Here it happens to be harmless (INNER JOIN), but building that habit will break your OUTER joins — see the `ON`-vs-`WHERE` trap in §9.
- Forgetting quotes around string literals, or using double quotes inconsistently.

## Interview Notes from Query 2

- Know that `WHERE` is evaluated **after** the join.
- For **INNER** joins, `ON` and `WHERE` conditions are interchangeable in effect. For **OUTER** joins they are **not**. Interviewers love this.
- Be able to explain the "build the join, verify, then filter" development approach.

---

# 17. Practical Query 3 — LEFT JOIN

## The question

> *"Fetch out each project allocated to each employee."*

## The query

```sql
select *
from Employee as e
LEFT JOIN Project as p on e.id = p.empID;
```

## ⚠ Correction — a slip in the spoken reasoning

While setting this up, the instructor said:

> *"I have to focus on all the projects, I have to list all the projects, whether they are allocated to any employee or not... So I have to focus on Left, i.e. on the **Project table**."*

But the query he actually wrote puts **`Employee`** first — so **`Employee` is the Left table, not `Project`.**

**What the query actually does:** it returns **every employee**, whether or not they have a project. Employees with no project get `NULL` in every project column.

**The output he then showed confirms the query's behaviour, not the spoken framing** — Jatin (an employee with no project) appeared with `NULL`s. If `Project` had truly been the Left table, the output would have listed all projects instead, and since every project has a valid `empID`, there would have been **no `NULL`s at all**.

**Trust the query and the output.** The spoken sentence was a slip of the tongue mid-explanation. This is a useful lesson in itself: **the Left table is whatever you wrote first — never assume, always read the `FROM` clause.**

## Why LEFT JOIN and not INNER JOIN here?

Because we want the employees who have **no** project to still appear. INNER JOIN would silently hide them. This is the entire reason OUTER joins exist: **absence is information.** "Which employees are unassigned?" is a real business question, and only an OUTER join can answer it.

## Note on `SELECT *`

Here `*` is used deliberately: the point of the demo is to *see the `NULL`s appear on the right-hand side*, which requires seeing all the project columns. Output width = 7 + 5 = **12 columns**.

## Dry Run

| `e` row | `e.id` | matching `p` rows | Emitted |
|---|---|---|---|
| 1 Aman Proto, Delhi | 1 | project 1 (A) | full row, all project cols populated |
| 2 Yagya Narayan, Palam | 2 | project 2 (B) | full row |
| 3 Rahul BD, Kolkata | 3 | project 3 (C) | full row |
| 3 Rahul BD, Kolkata | 3 | project 4 (D) | full row (second row for Rahul) |
| **4 Jatin Hermit, Raipur** | 4 | **none** | **employee cols filled, all 5 project cols = `NULL`** |
| 5 PK Pandey, Jaipur | 5 | project 5 (E) | full row |

## Output (6 rows, 12 columns — project columns abbreviated)

| e.id | e.fname | e.lname | e.City | p.id | p.empID | p.name | p.startdate | p.clientID |
|---|---|---|---|---|---|---|---|---|
| 1 | Aman | Proto | Delhi | 1 | 1 | A | 2021-04-21 | 3 |
| 2 | Yagya | Narayan | Palam | 2 | 2 | B | 2021-03-12 | 1 |
| 3 | Rahul | BD | Kolkata | 3 | 3 | C | 2021-01-16 | 5 |
| 3 | Rahul | BD | Kolkata | 4 | 3 | D | 2021-04-27 | 2 |
| **4** | **Jatin** | **Hermit** | **Raipur** | **NULL** | **NULL** | **NULL** | **NULL** | **NULL** |
| 5 | PK | Pandey | Jaipur | 5 | 5 | E | 2021-05-01 | 4 |

## The instructor's reading of the output

> *"Look carefully — inside this, **Jatin**, who is employee ID `4`, Jatin Hermit — corresponding to him I am not getting any project. If we come to the iPad, then if we talk about Jatin, Jatin's employee ID is `4`; corresponding to `4` I do not have any project allocated here at all, i.e. **Jatin Hermit has not been allocated any project**."*
>
> *"So this kind of information has come to me. We did a LEFT JOIN, so **you can see I am getting `NULL` on the Right side.**"*

## Why this is the perfect demonstration

Compare with **Query 1** (INNER JOIN on the *same two tables* with the *same `ON` condition*):

| | Query 1 (INNER) | Query 3 (LEFT) |
|---|---|---|
| Rows returned | 5 | **6** |
| Jatin present? | **No** | **Yes, with `NULL`s** |
| Everything else | identical | identical |

**One keyword changed. One row appeared.** That single row *is* the difference between INNER JOIN and LEFT JOIN, made concrete. If you understand why Jatin is present in one and absent in the other, you understand OUTER joins.

## ▶ Beyond the lecture — turning this into a business report

To list *only* the unassigned employees, add the anti-join filter from §9:

```sql
SELECT e.id, e.fname, e.lname
FROM Employee AS e
LEFT JOIN Project AS p ON e.id = p.empID
WHERE p.empID IS NULL;
```

Returns exactly one row: `4, Jatin, Hermit`. This is the "find the bench" report every services company runs.

---

# 18. Practical Query 4 — RIGHT JOIN

## The question

> *"List out all the projects along with the employee's name and their respective allocated email ID."*

The focus is on **projects** — every project must be listed. So the project table must be the one whose rows are all guaranteed to appear.

## The query

```sql
select p.id, p.name, e.fname, e.lname, e.emailID
FROM Employee as e
RIGHT JOIN Project as p ON e.id = p.empID;
```

**Structure:**
- `Employee AS e` is written first → **Left table**
- `Project AS p` comes after `RIGHT JOIN` → **Right table**
- `RIGHT JOIN` ⇒ **all rows of `Project` are guaranteed**, employee data attached where available
- Selected: project id, project name, employee first name, last name, email

**This is exactly why RIGHT JOIN was chosen.** The instructor wanted `Project` guaranteed but wrote `Employee` first (natural reading order). Flipping the join direction achieves the goal without reordering the tables. Equivalent alternative: `FROM Project AS p LEFT JOIN Employee AS e ON e.id = p.empID`.

## Dry Run

Driven by the **Right** table (`Project`):

| `p` row | `p.empID` | matching `e` row | Emitted |
|---|---|---|---|
| 1, A | 1 | Aman Proto | `1, A, Aman, Proto, aman@gmail.com` |
| 2, B | 2 | Yagya Narayan | `2, B, Yagya, Narayan, yagya@gmail.com` |
| 3, C | 3 | Rahul BD | `3, C, Rahul, BD, rahul@gmail.com` |
| 4, D | 3 | Rahul BD | `4, D, Rahul, BD, rahul@gmail.com` |
| 5, E | 5 | PK Pandey | `5, E, PK, Pandey, pk@gmail.com` |

## Output (5 rows)

| p.id | p.name | e.fname | e.lname | e.emailID |
|---|---|---|---|---|
| 1 | A | Aman | Proto | aman@gmail.com |
| 2 | B | Yagya | Narayan | yagya@gmail.com |
| 3 | C | Rahul | BD | rahul@gmail.com |
| 4 | D | Rahul | BD | rahul@gmail.com |
| 5 | E | PK | Pandey | pk@gmail.com |

## ⚠ IMPORTANT — this RIGHT JOIN contains no `NULL`s. Why?

A RIGHT JOIN only produces `NULL`s when some right-table row has **no** matching left-table row.

Here, every project's `empID` (`1, 2, 3, 3, 5`) corresponds to an employee that genuinely exists. **No orphan projects exist**, so no padding is needed.

**Do not conclude that "RIGHT JOIN always shows `NULL`s."** `NULL`s appear only when there is unmatched data. On this dataset a RIGHT JOIN and an INNER JOIN return identical results — the *difference* would only show up if someone inserted a project with, say, `empID = 99`.

This is an excellent thing to be able to explain in an interview: **the join type determines what *would* happen to unmatched rows; whether any unmatched rows exist depends on the data.**

## ⚠ Correction — a misreading of the output

The instructor said:

> *"Here it is plainly visible that projects **B, C and D** are being done by a person named Rahul BD."*

**That is not what the data says.** Checking `Project`:
- Project **B** has `empID = 2` → **Yagya Narayan**
- Project **C** has `empID = 3` → Rahul BD ✔
- Project **D** has `empID = 3` → Rahul BD ✔

**Rahul does C and D only. Project B belongs to Yagya.** The correct output table is the one above. The likely cause is that "B" was read from `lname = BD` — Rahul's *last name* is literally "BD", sitting adjacent to the project-name column, which is an easy visual slip when scanning a result grid.

**Lesson for you:** always verify a join's output against the base tables before trusting it. This is also a warning about naming columns ambiguously — a last name of "BD" next to a project named "B" or "D" is a readability disaster.

## The interchangeability remark

Immediately after this query, the instructor made the LEFT/RIGHT interchangeability point covered in §10:

> *"LEFT JOIN and RIGHT JOIN get used interchangeably, because when you use a LEFT JOIN, the table you are treating as the Left table — if you make it the Right table, if you write it on the right, then it will be used like a RIGHT JOIN."*

Applied to this exact query:

```sql
-- What was written:
FROM Employee AS e RIGHT JOIN Project AS p ON e.id = p.empID

-- Exactly equivalent:
FROM Project AS p LEFT JOIN Employee AS e ON e.id = p.empID
```

Both guarantee all projects. Same rows. (Default column order would differ under `SELECT *`, but here the columns are named explicitly, so even the output is byte-identical.)

---

# 19. Practical Query 5 — CROSS JOIN

## The question

> *"List out all the combinations possible for the employee's name and projects that can exist."*

The word **"all the combinations"** is the giveaway: no matching, no condition, just every pairing. That is a Cartesian product.

## The query

```sql
select e.fname, e.lname, p.id, p.name
from Employee as e
CROSS JOIN Project as p;
```

**Note what is absent: there is no `ON` clause.** No condition exists because none is wanted.

## Dry Run and row count

- `Employee` has **5** rows
- `Project` has **5** rows
- Result: **5 × 5 = 25 rows**

The generation pattern (outer loop over employees, inner loop over projects):

| # | e.fname | e.lname | p.id | p.name |
|---|---|---|---|---|
| 1 | Aman | Proto | 1 | A |
| 2 | Aman | Proto | 2 | B |
| 3 | Aman | Proto | 3 | C |
| 4 | Aman | Proto | 4 | D |
| 5 | Aman | Proto | 5 | E |
| 6 | Yagya | Narayan | 1 | A |
| 7 | Yagya | Narayan | 2 | B |
| 8 | Yagya | Narayan | 3 | C |
| 9 | Yagya | Narayan | 4 | D |
| 10 | Yagya | Narayan | 5 | E |
| 11 | Rahul | BD | 1 | A |
| 12 | Rahul | BD | 2 | B |
| 13 | Rahul | BD | 3 | C |
| 14 | Rahul | BD | 4 | D |
| 15 | Rahul | BD | 5 | E |
| 16 | Jatin | Hermit | 1 | A |
| 17 | Jatin | Hermit | 2 | B |
| 18 | Jatin | Hermit | 3 | C |
| 19 | Jatin | Hermit | 4 | D |
| 20 | Jatin | Hermit | 5 | E |
| 21 | PK | Pandey | 1 | A |
| 22 | PK | Pandey | 2 | B |
| 23 | PK | Pandey | 3 | C |
| 24 | PK | Pandey | 4 | D |
| 25 | PK | Pandey | 5 | E |

**Note Jatin appears here** (rows 16–20) even though he has no project. Of course he does — CROSS JOIN does not care about relationships. Every employee is paired with every project regardless of reality. Rows 16–20 assert nothing true about the world; they are merely *possible* combinations.

**That is the fundamental difference:** INNER/LEFT/RIGHT report **what is**; CROSS JOIN reports **what could be**.

## The instructor's commentary

> *"Now see, here all your combinations will get printed. So its industrial use is nothing special. But suppose I have to do something like: corresponding to every person, I need a list of all the projects — all the possible combinations that can be made — then I can use something like this. But I think this does not get used much."*

So the one legitimate use-case shape he offered is: **generating an exhaustive grid of possibilities** — e.g. an assignment matrix showing every employee against every project, so you can then mark which combinations are actually staffed.

---
---

# PART E — CONSOLIDATION

---

# 20. Master Comparison Table of All Joins

Using a Left table **L** (m rows, `a` columns) and a Right table **R** (n rows, `b` columns):

| Join | Keeps | `NULL`s appear in | Venn | Row count | Cols | `ON` needed? | MySQL keyword exists? |
|---|---|---|---|---|---|---|---|
| **INNER** | Only matching rows | Nowhere (from the join itself) | Intersection only | ≤ min-ish, depends on matches | a+b | Yes | ✔ `INNER JOIN` |
| **LEFT** | All L + matched R | **Right-side** columns | Whole left circle | ≥ INNER; ≥ m if 1:1 | a+b | Yes | ✔ `LEFT JOIN` |
| **RIGHT** | All R + matched L | **Left-side** columns | Whole right circle | ≥ INNER | a+b | Yes | ✔ `RIGHT JOIN` |
| **FULL** | Everything from both | **Both** sides | Both circles fully | LEFT + RIGHT − INNER | a+b | Yes | ✘ **must emulate** |
| **CROSS** | Every possible pairing | Nowhere | (not applicable) | **m × n** | a+b | **No** | ✔ `CROSS JOIN` |
| **SELF** | (depends on inner type) | (depends) | (depends) | (depends) | a+a | Yes | ✘ **must emulate** |

## Escalation ladder — a mental model for choosing

Order the joins by "how much do they let through":

```
CROSS  >  FULL  >  LEFT / RIGHT  >  INNER
(most permissive)              (most restrictive)
```

- Need **only confirmed relationships**? → INNER
- Need one side **guaranteed complete**? → LEFT (or RIGHT)
- Need **both sides complete**? → FULL (emulate in MySQL)
- Need **every hypothetical pairing**? → CROSS
- Working within **one table**? → SELF (via INNER/LEFT + aliases)

## Which is used most?

> *"The most-used join is INNER JOIN itself. So you have to learn INNER JOIN really well."*

Roughly: INNER dominates, LEFT is a close second in reporting work, RIGHT is rare (rewritable as LEFT), FULL is rare, CROSS is rarer still, SELF appears in specific hierarchical/comparison problems.

---

# 21. Complexity Discussion

> **▶ Beyond the lecture.** The instructor did not discuss algorithmic complexity — SQL is declarative, so you specify *what* you want and the query optimiser chooses *how*. This section is added because complexity is standard interview territory and it explains why join performance varies so much.

## Why complexity is not fixed for a join

In an imperative language, you write the algorithm, so its complexity is determined. In SQL you write only the *goal*. The database's **query optimiser** picks the physical algorithm based on table sizes, available indexes, and statistics. The same `INNER JOIN` can be O(m·n) or nearly O(m+n) depending on that choice.

## The three physical join algorithms

Let m = rows in the left table, n = rows in the right table.

### 1. Nested Loop Join

```
for each row L in LeftTable:        # m iterations
    for each row R in RightTable:   # n iterations
        if condition(L, R): emit
```

- **Time: O(m · n)** — every left row compared against every right row.
- **Space: O(1)** extra (streams output).
- **Used when:** tables are small, or no index/sort is available.
- **With an index** on the right table's join column, the inner scan becomes an index lookup: **O(m · log n)**, a huge improvement. This is why indexing foreign key columns matters so much.

### 2. Hash Join

- **Build phase:** load the smaller table into an in-memory hash table keyed on the join column — **O(n)**.
- **Probe phase:** scan the larger table, hashing each row's key to find matches — **O(m)**.
- **Time: O(m + n)** average.
- **Space: O(n)** — the hash table must fit in memory (if not, it spills to disk and degrades).
- **Used when:** joining on **equality** (`=`) with large unindexed tables. MySQL added hash joins in 8.0.18.

### 3. Sort-Merge Join

- Sort both tables on the join key, then walk them in parallel like merging two sorted lists.
- **Time: O(m log m + n log n)** for sorting, then **O(m + n)** for the merge.
- **Space: O(m + n)** if sorting must be materialised; O(1) extra if inputs are already sorted (e.g. by index order).
- **Used when:** inputs are already sorted, or for range conditions where hashing does not apply.

## CROSS JOIN complexity

CROSS JOIN is the one case with no optimisation available:

- **Time: Θ(m · n)** — not an upper bound but an exact requirement, since m·n rows must actually be produced.
- **Space: O(m · n)** for the materialised result.

This is why an accidental Cartesian product is catastrophic. Two 100,000-row tables → 10¹⁰ rows.

## Practical performance rules

1. **Index your join columns.** A foreign key without an index turns a fast lookup into a full scan per row. Primary keys are indexed automatically; foreign keys often are not, unless the engine creates one.
2. **Filter early.** A `WHERE` that cuts a table from a million rows to a hundred *before* the join makes the join dramatically cheaper — optimisers usually do this automatically ("predicate pushdown"), but writing selective conditions helps.
3. **Select only needed columns.** `SELECT *` on a 12-column join transfers data you will discard.
4. **Join order matters** for multi-table joins; the optimiser normally handles it, but with many tables its search space explodes.
5. **Use `EXPLAIN`** before your query to see which algorithm the optimiser chose and whether indexes are being used. This is the single most useful practical skill in this section.

## Complexity of the FULL JOIN emulation

`LEFT JOIN UNION RIGHT JOIN` costs roughly **two join executions plus a deduplication pass**. Deduplication requires sorting or hashing the combined output — an extra **O(k log k)** where k is the combined row count. This makes the emulation meaningfully more expensive than a native `FULL OUTER JOIN` would be, which is one reason to avoid it when a LEFT JOIN would suffice.

---

# 22. Complete List of Common Mistakes

Consolidated from the whole lecture, plus the additions flagged above.

## On `REPLACE`

1. Treating `REPLACE` as "`UPDATE` that inserts." It is `DELETE` + `INSERT`.
2. **Forgetting that unmentioned columns become `NULL`.** The single biggest `REPLACE` hazard — and the on-screen demo proved it.
3. Omitting the primary key from a `REPLACE`, so it can never detect a collision.
4. Expecting `UPDATE` to error when no rows match. It silently reports 0 rows affected.

## On joins generally

5. **Forgetting the `ON` clause** → accidental Cartesian product; possibly catastrophic on large tables.
6. **Not qualifying ambiguous columns** → "ambiguous column" errors. Guaranteed when both tables have `id` or `City`.
7. **Confusing JOIN with UNION.** JOIN adds **columns** (horizontal); UNION adds **rows** (vertical). The instructor pre-warned about this explicitly.
8. **Mixing up Left and Right.** The Left table is whichever is written **first**. Irrelevant for INNER, decisive for LEFT/RIGHT.
9. **Assuming Left = Parent.** They are independent concepts.
10. **Assuming output row count equals input row count.** One-to-many relationships fan out — Rahul appeared twice.
11. **Joining on the wrong columns** (`c.id = o.id` instead of `c.id = o.cust_id`). Runs fine, answers wrongly.

## On OUTER joins

12. Being surprised by `NULL`s. They are the designed padding mechanism.
13. Getting the `NULL` side backwards. LEFT ⇒ `NULL`s on the **right**; RIGHT ⇒ `NULL`s on the **left**.
14. **Filtering a right-table column in `WHERE` after a LEFT JOIN** → silently converts it to an INNER JOIN. Put such conditions in `ON`.
15. Writing `= NULL` instead of `IS NULL`.
16. Assuming an OUTER join must show `NULL`s — Query 4 showed a RIGHT JOIN with none, because the data had no orphans.

## On FULL and SELF joins

17. **Writing `FULL JOIN` in MySQL.** It does not exist.
18. Using `UNION ALL` in the FULL JOIN emulation → duplicated matching rows.
19. Mismatched column lists across the two halves of a `UNION`.
20. **Looking for a `SELF JOIN` keyword.** There is none — it is INNER JOIN + aliases.
21. Writing a self join as `ON e1.id = e2.id` → every row joins to itself; nothing useful returned.
22. Using INNER JOIN for a manager self join and losing the top-level manager (whose `manager_id` is `NULL`).

## On reading results

23. **Misreading result grids** — the "project B done by Rahul BD" slip is a live example of confusing adjacent columns. Always verify against the base tables.
24. Miscounting rows in a Cartesian product — the notes' `3 × 2` example was written as 4 rows instead of 6. Trust the **formula**, not a hurried count.

---

# 23. Complete Interview Notes

The instructor called JOINS *"the interviewer's favourite topic."* Here is everything from this segment, organised for revision.

## Definitions to memorise word-for-word

- **INNER JOIN** — *Returns a resultant table that has matching values from both the tables, or all the tables.*
- **LEFT JOIN** — *Returns a resulting table that has all the data from the left table and the matched data from the right table.*
- **RIGHT JOIN** — All the data from the right table and the matched data from the left table. *"Interested in knowing only the right side info."*
- **FULL JOIN** — *Nothing but the UNION of LEFT and RIGHT JOIN.*
- **CROSS JOIN** — *Nothing but the Cartesian Product.*
- **SELF JOIN** — A table joined with itself; emulated using INNER JOIN and aliasing.
- **REPLACE vs UPDATE** — *If a row is not present, REPLACE will add a new row while UPDATE will do nothing.*
- **Join prerequisite** — *To apply joins, there should be a common attribute between both the tables.*

## Facts that get asked directly

1. **MySQL does not support `FULL JOIN`.** Emulate with `LEFT JOIN ... UNION ... RIGHT JOIN`. (Most other RDBMSs *do* support it.)
2. **There is no `SELF JOIN` keyword.** It is a pattern, not syntax.
3. **CROSS JOIN needs no `ON` clause** — the only join that does not.
4. **CROSS JOIN row count = m × n.**
5. **Joined column count = left columns + right columns** (with `SELECT *`). Demonstrated as 7 + 5 = 12.
6. **LEFT and RIGHT JOIN are interchangeable** by swapping the tables and flipping the keyword.
7. **INNER JOIN is the most commonly used join.**
8. **`AS` = aliasing**, scoped to a single query, mandatory for self joins and for disambiguating shared column names.
9. **A join produces a new table**; it never modifies the source tables.
10. **JOIN grows columns; UNION grows rows.**
11. **`UNION` deduplicates; `UNION ALL` does not** — and the FULL JOIN emulation depends on that deduplication.

## Diagrams to be able to draw

- INNER JOIN — intersection shaded
- LEFT JOIN — entire left circle shaded
- RIGHT JOIN — entire right circle shaded
- FULL JOIN — both circles fully shaded
- Left/Right table with PK and FK columns, labelled parent and child

## Query patterns to be able to write from memory

```sql
-- Basic inner join
SELECT a.col, b.col FROM A AS a INNER JOIN B AS b ON a.id = b.a_id;

-- Anti-join: rows in A with no match in B
SELECT a.* FROM A AS a LEFT JOIN B AS b ON a.id = b.a_id WHERE b.a_id IS NULL;

-- FULL JOIN emulation for MySQL
SELECT * FROM A AS a LEFT  JOIN B AS b ON a.id = b.a_id
UNION
SELECT * FROM A AS a RIGHT JOIN B AS b ON a.id = b.a_id;

-- Self join: employee with manager name
SELECT e1.name AS employee, e2.name AS manager
FROM Employee AS e1 LEFT JOIN Employee AS e2 ON e1.manager_id = e2.id;

-- Three-table chain
SELECT * FROM A JOIN B ON A.id = B.a_id JOIN C ON B.id = C.b_id;
```

## Trickier follow-ups you should be ready for

- *"What is the difference between `ON` and `WHERE` in a join?"* → Identical in effect for INNER joins. For OUTER joins, `ON` filters **before** `NULL`-padding (preserving unmatched rows), `WHERE` filters **after** (potentially removing them and silently turning your OUTER join into an INNER join).
- *"Can a LEFT JOIN return more rows than the left table has?"* → **Yes**, when the right table has multiple matches per key. Rahul's two projects proved it.
- *"Will a RIGHT JOIN always show `NULL`s?"* → No. Only if right rows exist with no left match. Query 4 had none.
- *"How do you find rows in A with no counterpart in B?"* → LEFT JOIN + `WHERE b.key IS NULL` (anti-join). Alternatives: `NOT EXISTS`, `NOT IN` (beware `NULL` semantics with `NOT IN`).
- *"Why is a Cartesian product dangerous?"* → m × n row explosion; a forgotten `ON` produces one accidentally.
- *"Can you join three or more tables?"* → Yes, chain `JOIN ... ON ...` clauses. The PDF notes showed exactly this two-condition form.
- *"Does the join modify the underlying tables?"* → No. It returns a new result table.

---

# 24. The Cliffhanger Question

The segment ends on a question written on the iPad:

```
Q  Can we use JOIN w/o using JOIN key word?
```

The instructor's framing:

> *"Now let us discuss a very important question: **Can we use JOIN without using the JOIN keyword?** Here we are talking about INNER JOIN — can we apply an inner join without using the `INNER JOIN` keyword?"*

**The lecture cuts off here**, so the answer was given in the next segment. For completeness — and because you will meet this syntax constantly in older codebases — here is the answer.

## ▶ Beyond the lecture — the answer: yes, with implicit join syntax

You can express an INNER JOIN by listing tables comma-separated in the `FROM` clause and putting the join condition in `WHERE`:

```sql
-- Explicit join syntax (ANSI SQL-92, modern):
SELECT e.id, e.fname, p.name
FROM Employee AS e
INNER JOIN Project AS p ON e.id = p.empID;

-- Implicit join syntax (SQL-89 "comma join", older):
SELECT e.id, e.fname, p.name
FROM Employee AS e, Project AS p
WHERE e.id = p.empID;
```

**Both return exactly the same rows.**

### How it works

`FROM Employee, Project` alone is a **CROSS JOIN** — every employee paired with every project (25 rows here). The `WHERE e.id = p.empID` then filters that product down to only the pairs where the keys match — which is precisely the definition of an INNER JOIN.

So conceptually: **INNER JOIN = CROSS JOIN followed by a filter.** The implicit syntax makes that relationship visible; the explicit syntax hides it behind a keyword. (Query engines do not literally build the full product — the optimiser is smarter than that — but logically the result is identical.)

### Which should you use? — explicit, and here is why

1. **Only INNER joins can be written implicitly.** There is no comma-syntax for LEFT, RIGHT, or FULL joins. You need `LEFT JOIN` regardless, so consistency argues for the explicit form everywhere.
2. **Forgetting the `WHERE` gives a silent Cartesian product.** With explicit syntax, forgetting the `ON` is usually a syntax error — the language catches you.
3. **Separation of concerns.** Explicit syntax keeps *join conditions* in `ON` and *filters* in `WHERE`. Implicit syntax jumbles them together in one `WHERE`, which becomes unreadable with four or five tables.
4. **Industry standard.** The explicit form has been standard since SQL-92 and is what modern code and interviewers expect.

**The correct interview answer:** *"Yes — you can write an implicit/comma join by listing the tables in `FROM` and putting the condition in `WHERE`. It is equivalent to an INNER JOIN, because `FROM A, B` produces a Cartesian product that the `WHERE` then filters. However, it only works for INNER joins, and the explicit `JOIN ... ON` syntax is preferred because it is clearer and prevents accidental Cartesian products."*

---
---

# QUICK REVISION SHEET

| # | Item | One-line takeaway |
|---|---|---|
| 1 | `REPLACE` 3rd syntax | `REPLACE INTO t(cols) SELECT ... ` — source rows come from a query |
| 2 | `REPLACE` hazard | Unmentioned columns become **`NULL`** — it deletes and re-inserts |
| 3 | `REPLACE` vs `UPDATE` | Missing row ⇒ `REPLACE` inserts, `UPDATE` does nothing |
| 4 | Why joins exist | FK creates relations → JOIN fetches across them |
| 5 | Left / Right | Position in the query, **not** parent/child |
| 6 | Parent / Child | PK side / FK side |
| 7 | Join prerequisite | Must share a common attribute |
| 8 | `AS` | Query-scoped alias; mandatory for self joins & ambiguity |
| 9 | INNER JOIN | Intersection only; most used |
| 10 | LEFT JOIN | All left + matched right; `NULL`s on the **right** |
| 11 | RIGHT JOIN | All right + matched left; `NULL`s on the **left** |
| 12 | FULL JOIN | Everything; **not in MySQL** — `LEFT UNION RIGHT` |
| 13 | CROSS JOIN | Cartesian product, **m × n**, no `ON`, no industrial use |
| 14 | SELF JOIN | Table with itself; emulated via INNER JOIN + aliases |
| 15 | Growth direction | JOIN → columns; UNION → rows |
| 16 | Column count | left cols + right cols (7 + 5 = 12) |
| 17 | Fan-out | One-to-many multiplies rows (Rahul × 2 projects) |
| 18 | Anti-join | `LEFT JOIN ... WHERE right.key IS NULL` |
| 19 | `ON` vs `WHERE` | Same for INNER; **critically different** for OUTER |
| 20 | Implicit join | `FROM A, B WHERE A.k = B.k` ≡ INNER JOIN only |
