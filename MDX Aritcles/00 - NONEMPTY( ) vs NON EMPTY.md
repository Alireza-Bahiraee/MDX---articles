# NONEMPTY() vs NON EMPTY in MDX  
### What’s the real difference?

In MDX (for SQL Server Analysis Services and related OLAP engines), `NONEMPTY()` is a **function**, while `NON EMPTY` is a **keyword** used on query axes.  

Both aim to remove “empty” rows/columns, but they behave differently in **when** and **how** they filter data—especially when calculations and multiple axes are involved.

---

## 🔍 Quick overview

| Aspect                  | `NONEMPTY()`                               | `NON EMPTY` keyword                        |
|-------------------------|---------------------------------------------|--------------------------------------------|
| Type                    | Function                                    | Keyword / axis prefix                      |
| Where used              | Inside set expressions, named sets, WITH   | On axes in the `SELECT` clause             |
| Evaluation time         | When the set/axis is built                 | Last, after all axes are computed          |
| Filters based on        | Specific measure(s) you pass               | All measures on the other axes             |
| Handles calculations    | Yes (uses cube calculations)               | Yes, but on final result set               |
| Duplicates              | Preserves duplicate tuples                 | Works on final result (no explicit dupes)  |
| Typical use case        | Fine-grained control, named sets           | Simple, Excel-like queries                 |

---

## 1️⃣ Syntax and role

### `NONEMPTY(set_expression [, set_expression2])`

A function that returns a set of tuples that are not empty when evaluated against another set (usually a measure).

```mdx
NONEMPTY(
  [Customer].[Customer].MEMBERS,
  [Measures].[Sales Amount]
)
```

- Used inside:
  - Set expressions
  - Named sets (`WITH SET ...`)
  - Combined with `SUM`, `FILTER`, etc.

---

### `NON EMPTY` (keyword)

A prefix on an axis in the `SELECT` clause.

```mdx
SELECT
  [Measures].[Sales Amount] ON COLUMNS,
  NON EMPTY [Customer].[Customer].MEMBERS ON ROWS
FROM [Adventure Works]
```

- Applies to the **whole axis**.
- Removes tuples that are empty across the measures on the other axes.

---

## ⏱ 2. When they are evaluated

This is the **core difference**.

### `NON EMPTY` keyword

- Evaluated **last**, after all axes have been computed.
- Conceptual flow:

```text
1. Compute all axes (rows, columns, etc.)
2. Build the full result grid
3. Apply NON EMPTY → remove rows/columns that are empty
   across all measures on the other axes
```

So it filters the **final result set**.

---

### `NONEMPTY()` function

- Evaluated **when the specific axis/set is built**, not at the end.
- Conceptual flow:

```text
1. Build the set using NONEMPTY(...)
   → filter members based on given measure(s)
2. Use that filtered set in the query
3. Combine with other axes
4. Final result is produced
```

So it can change which members exist **before** cross-join with other axes happens.

> The big difference between the `NON EMPTY` keyword and the `NONEMPTY` function is **when** the evaluation occurs in the MDX.  
> `NON EMPTY` is the last thing evaluated; `NONEMPTY()` is evaluated when the specific axis is evaluated. [^3]

---

## 🧮 3. Behavior with calculations and duplicates

### `NONEMPTY()`

- Takes cube **calculations** into account.
- Preserves **duplicate tuples**.
- “Non-empty” is a property of the **cells** referenced by the tuples, not the tuples themselves.

Example:

```mdx
NONEMPTY(
  [Customer].[Customer].MEMBERS,
  [Measures].[Sales Amount]
)
```

- Each customer is checked against `Sales Amount`, including any calculated members.

---

### `NON EMPTY` keyword

- Also removes empty space, but works on the **final result set**.
- More “coarse” and global.
- Typically what you get from Excel-style pivot queries.

> The `NonEmpty` function takes into account calculations and preserves duplicate tuples.  
> Non-empty is a characteristic of the cells referenced by the tuples, not the tuples themselves. [^4]

---

## 🧪 4. Practical differences (examples)

### 4.1 Using `NON EMPTY` on a single axis

```mdx
SELECT
  [Measures].[Sales Amount] ON COLUMNS,
  NON EMPTY [Customer].[Country].MEMBERS ON ROWS
FROM [Adventure Works]
```

- Removes all countries that have **no Sales Amount** for any year on the COLUMNS axis.
- Filters the final result set across all measures on COLUMNS.

---

### 4.2 Using `NONEMPTY()` in a named set

```mdx
WITH
  SET [NonEmptyCountries] AS
    NONEMPTY(
      [Customer].[Country].MEMBERS,
      [Measures].[Sales Amount]
    )
SELECT
  [Measures].[Sales Amount] ON COLUMNS,
  [NonEmptyCountries] ON ROWS
FROM [Adventure Works]
```

- The set is filtered **before** it is cross-joined with the COLUMNS axis.
- With multiple measures or complex hierarchies, behavior can differ from `NON EMPTY`.

---

## ⚡ 5. Performance considerations

There’s a common myth: “`NONEMPTY()` is always faster than `NON EMPTY`.”  
That’s **not** universally true.

Performance depends on:

- Cube structure and partitions
- Size of the sets
- Block mode vs cell-by-cell evaluation
- How many measures and calculated members are involved

Sometimes:

- `NON EMPTY` is more efficient (works on final set).
- Sometimes `NONEMPTY()` is better (reduces sets earlier).

Always test with your real queries and cube.

---

## ✅ 6. Which one should you use?

### Use `NON EMPTY` when:

- You want a simple, typical OLAP query (like from Excel).
- You want to remove empty rows/columns based on **all** measures on the other axes.
- You don’t need fine-grained control over set construction.

### Use `NONEMPTY()` when:

- You need to remove empty tuples based on a **specific measure** or set.
- You are building **named sets** or complex expressions.
- You care about how **calculations** and **duplicates** are handled.

---

## 🧭 Summary

- `NONEMPTY()`
  - Function.
  - Evaluated when the axis/set is built.
  - Can filter based on a specific measure/set.
  - Takes calculations into account and preserves duplicates.

- `NON EMPTY`
  - Keyword/prefix on axes.
  - Evaluated last, after all axes are computed.
  - Filters the final result set across all measures on the other axes.
  - More global and simpler for typical queries.

---

## Sources

<ul>
  <li><a href="https://sqldusty.com/2012/08/10/non-empty-vs-nonempty-to-the-death/">NON EMPTY vs Nonempty(): To the Death!</a></li>
  <li><a href="https://subscription.packtpub.com/book/data/9781849689601/1/ch01lvl1sec12/optimizing-mdx-queries-using-the-nonempty-function">Optimizing MDX queries using the NonEmpty() function</a></li>
  <li><a href="https://mitchellpearson.com/2016/02/09/mdx-non-empty-keyword-vs-nonempty-function/">MDX NON EMPTY KEYWORD VS NONEMPTY FUNCTION</a></li>
  <li><a href="https://learn.microsoft.com/en-us/sql/mdx/nonempty-mdx?view=sql-server-ver17">NonEmpty (MDX) - SQL Server</a></li>
  <li><a href="http://www.ssas-info.com/forum/6-ssas-admindevelopers-lounge/2196-format-option-not-working-for-attribute">Non Empty v/s NonEmpty – SSAS-Info</a></li>
  <li><a href="https://stackoverflow.com/questions/29779880/mdx-query-for-filtering-non-empty-values">MDX Query for Filtering Non Empty Values</a></li>
  <li><a href="https://stackoverflow.com/questions/64758664/usage-of-select-nonempty">Usage of SELECT NONEMPTY</a></li>
  <li><a href="https://learn.microsoft.com/en-us/sql/mdx/working-with-empty-values?view=sql-server-ver17">Working with Empty Values - SQL Server</a></li>
  <li><a href="https://www.cnblogs.com/heitou/archive/2011/03/08/Non_Empty_vs_NonEmpty.html">Non Empty 和 NonEmpty</a></li>
  <li><a href="https://www.tutorialgateway.org/mdx-nonempty/">MDX NONEMPTY Function</a></li>
</ul>
