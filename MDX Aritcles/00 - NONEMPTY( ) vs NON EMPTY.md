# What is the difference between nonempty() and non empty in the mdx query

In MDX (for SQL Server Analysis Services and related OLAP engines), `NONEMPTY()` is a function, while `NON EMPTY` is a keyword (prefix) used on query axes. They are similar in goal (removing empty rows/columns), but they have key differences in when and how they are evaluated, and in their behavior with calculations.

## 1. Syntax and role

- `NONEMPTY(set_expression [, set_expression2])`
    - A function that returns a set of tuples that are not empty when evaluated against another set (usually a measure).
    - Example:

```mdx
NONEMPTY(
  [Customer].[Customer].MEMBERS,
  [Measures].[Sales Amount]
)
```

    - Used inside set expressions, named sets, or in combination with `WITH`, `SUM`, etc.
- `NON EMPTY` (keyword)
    - A prefix on an axis in the `SELECT` clause.
    - Example:

```mdx
SELECT
  [Measures].[Sales Amount] ON COLUMNS,
  NON EMPTY [Customer].[Customer].MEMBERS ON ROWS
FROM [Adventure Works]
```

    - Applies to the whole axis; it removes tuples that are empty across the measures on the other axes.


## 2. When they are evaluated

The core difference is evaluation order:

- `NON EMPTY` keyword
    - Evaluated last, after all axes have been computed.
    - Steps:

1. Compute all axes (rows, columns, etc.).
2. Then remove any tuples that are empty across the measures on those axes.
    - As a result, it filters the final result set: any row that is empty for all measures on the COLUMNS axis is removed.
- `NONEMPTY()` function
    - Evaluated when the specific axis is evaluated, not at the end.
    - It filters the set before it is combined with other axes.
    - This means:
        - It can behave differently when you have multiple axes.
        - It can affect which members are present in the result *before* cross-join with other axes happens.

From the literature:
> The big difference between the NON EMPTY keyword and the NONEMPTY function is when the evaluation occurs in the MDX. The NON EMPTY keyword is the last thing that is evaluated, in other words after all axes have been evaluated then the NON EMPTY keyword is executed to remove any empty space from the final result set. The NONEMPTY function is evaluated when the specific axis is evaluated.[^3]

## 3. Behavior with calculations and duplicates

- `NONEMPTY()`:
    - Takes into account calculations in the cube.
    - Preserves duplicate tuples.
    - Non-empty is a characteristic of the cells referenced by the tuples, not the tuples themselves.
    - Example:

```mdx
NONEMPTY(
  [Customer].[Customer].MEMBERS,
  [Measures].[Sales Amount]
)
```

This checks each customer against the `Sales Amount` measure (including any calculated values).
- `NON EMPTY` keyword:
    - Also removes empty space, but operates on the final result set.
    - It is more “coarse” and works on the top level of the query after all axes are resolved.
    - It is often simpler and more intuitive for typical Excel-style queries.

From Microsoft docs:
> The NonEmpty function takes into account calculations and preserves duplicate tuples.
> Non-empty is a characteristic of the cells referenced by the tuples, not the tuples themselves.[^4]

## 4. Practical differences (examples)

### 4.1. Using `NON EMPTY` on a single axis

```mdx
SELECT
  [Measures].[Sales Amount] ON COLUMNS,
  NON EMPTY [Customer].[Country].MEMBERS ON ROWS
FROM [Adventure Works]
```

- This removes all countries that have no Sales Amount for any year on the COLUMNS axis.
- It filters the final result set across all measures on the COLUMNS axis.


### 4.2. Using `NONEMPTY()` in a named set

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

- Here, the set is filtered before it is cross-joined with the COLUMNS axis.
- If you have multiple measures or complex hierarchies, the behavior can differ from `NON EMPTY` keyword.


## 5. Performance considerations

- There is a common misconception that `NONEMPTY()` always performs better than `NON EMPTY`. This is not true. Performance depends on:
    - The structure of the cube.
    - The size of sets.
    - Whether you’re using block mode vs cell-by-cell evaluation.
- In some cases, `NON EMPTY` is more efficient because it works on the final set; in others, `NONEMPTY()` is better because it reduces sets earlier.


## 6. Which one should you use?

- Use `NON EMPTY` keyword when:
    - You want a simple, typical OLAP query (like from Excel).
    - You want to remove empty rows/columns based on all measures on the other axes.
    - You don’t need fine-grained control over set construction.
- Use `NONEMPTY()` function when:
    - You need to remove empty tuples based on a specific measure or set.
    - You are building named sets or complex expressions.
    - You care about how calculations and duplicates are handled.


## Summary

- `NONEMPTY()`
    - Function.
    - Evaluated when the axis is built.
    - Can filter based on a specific measure/set.
    - Takes calculations into account and preserves duplicates.
- `NON EMPTY`
    - Keyword/prefix on axes.
    - Evaluated last, after all axes are computed.
    - Filters the final result set across all measures on the other axes.
    - More “global” and simpler for typical queries.

## Sources:


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
