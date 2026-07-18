




# Properties() in MDX  
### What does the `Properties()` function really do?

In MDX, `Properties()` is a **function** used to retrieve the value of a **member property**. It is one of the most practical functions when you need metadata from a member, not just the member itself.

In simple terms, if a member represents a product, date, customer, or geography item, `Properties()` lets you read details attached to that member, such as its name, key, caption, or a custom attribute.

---

## 🔍 Quick overview

| Aspect | `Properties()` |
|---|---|
| Type | Function |
| Where used | On a member expression |
| Main purpose | Return a member property value |
| Returns | Usually a string, or a typed value with `TYPED` |
| Best for | Metadata, captions, keys, custom attributes |
| Typical pattern | `CurrentMember.Properties(...)` |

---

## 1️⃣ Syntax and role

### `Member_Expression.Properties(Property_Name [, TYPED])`

`Properties()` returns the value of the specified property for a member. The property can be an intrinsic property such as `Name`, `Caption`, `Key`, or `ID`, or a user-defined property created in the cube design .

```mdx
[Date].[Calendar].CurrentMember.Properties("Caption")
```

- `Member_Expression` must evaluate to a member.
- `Property_Name` is the property you want to read.
- `TYPED` is optional and returns the native type when possible .

---

## 2️⃣ What it returns

By default, `Properties()` returns the property value as a string .
If you add `TYPED`, MDX tries to return the original data type instead of converting it to text .

That means:

- without `TYPED`, the result is usually text,
- with `TYPED`, the result can behave like a number, date, or other native type .

This matters when you want to sort, compare, or calculate with the value correctly.

---

## 3️⃣ Intrinsic and user-defined properties

MDX supports two broad kinds of member properties :

- **Intrinsic properties**: built-in properties such as `NAME`, `ID`, `KEY`, and `CAPTION`.
- **User-defined properties**: custom attributes added in the cube design, such as `Day Name` or `List Price` .

The `Properties()` function can read both kinds .

---

## 4️⃣ Adventure Works examples

### 4.1 Read intrinsic properties

The following query returns several intrinsic properties for a date member in Adventure Works .

```mdx
WITH
MEMBER Measures.MemberName AS
    [Order Date].[Date].&[20050101].Properties( "Name" )
MEMBER Measures.MemberKey AS
    [Order Date].[Date].&[20050101].Properties("Key")
MEMBER Measures.MemberID AS
    [Order Date].[Date].&[20050101].Properties("ID")
MEMBER Measures.MemberCaption AS
    [Order Date].[Date].&[20050101].Properties("Caption")
SELECT
{
    Measures.MemberName,
    Measures.MemberKey,
    Measures.MemberID,
    Measures.MemberCaption
} ON 0
FROM [Internet Sales]
```

This is useful when you want to inspect how a member is identified and displayed in the cube.

---

### 4.2 Read user-defined properties

Adventure Works also includes custom date-related properties that can be read from the member itself .

```mdx
WITH
MEMBER Measures.DayName AS
    [Order Date].[Date].&[20050101].Properties("WeekDay" )
MEMBER Measures.Month AS
    [Order Date].[Date].&[20050101].Properties("MOnth")
MEMBER Measures.DayOfMonth AS
    [Order Date].[Date].&[20050101].Properties("Day Number Of Month" )
MEMBER Measures.DayOfYear AS
   [Order Date].[Date].&[20050101].Properties("Day Number Of Year")
SELECT
{
    Measures.DayName,
    Measures.Month,
    Measures.DayOfMonth,
    Measures.DayOfYear
} ON 0
FROM [Internet Sales]
```

This pattern is common when a dimension contains useful descriptive attributes that are not measures .

---

### 4.3 Use `TYPED`

If you need the native value instead of a string, use `TYPED` .

```mdx
WITH
MEMBER Measures.DayNumberTyped AS
    [Date].[Calendar].[Date].&.Properties("Day Number Of Month", TYPED)
SELECT
{
    Measures.DayNumberTyped
} ON 0
FROM [Adventure Works]
```

This is helpful when the property should behave like a typed value in later calculations or client tools .

---

### 4.4 Use `Properties()` in a calculated member

A common pattern is to create a measure from a member property .

```mdx
WITH
MEMBER [Measures].[Product List Price] AS
    [Product].[Product].CurrentMember.Properties("List Price")
SELECT
{
    [Measures].[Product List Price]
} ON COLUMNS,
[Product].[Product].Members ON ROWS
FROM [Adventure Works]
```

This is a practical technique when you want to display a property like a measure in the result set .

---

### 4.5 Read properties from `CurrentMember`

The real power of `Properties()` appears when it is paired with `CurrentMember` .

```mdx
WITH
MEMBER [Measures].[Current Product Caption] AS
    [Product].[Product].CurrentMember.Properties("Caption")
SELECT
{
    [Measures].[Current Product Caption]
} ON 0,
[Product].[Product].Members ON 1
FROM [Adventure Works]
```

This lets you return a different property value for each row in the set, which is useful in dynamic reports .

---

## 5️⃣ Composite key warning

There is one important caveat: `Properties("Key")` does not always behave the way people expect for composite keys . For composite keys, `Properties("Key")` returns null, and you should use `Key0`, `Key1`, `Key2`, and so on instead .

```mdx
WITH
MEMBER Measures.MemberKey AS
    [Order Date].[Calendar].[Month of Year].&[2005]&[1].Properties("Key")
MEMBER Measures.MemberKey0 AS
    [Order Date].[Calendar].[Month of Year].&[2005]&[1].Properties("Key0")
MEMBER Measures.MemberKey1 AS
    [Order Date].[Calendar].[Month of Year].&[2005]&[1].Properties("Key1")

SELECT
{
    Measures.MemberKey,
    Measures.MemberKey0,
	Measures.MemberKey1
} ON 0
FROM [Internet Sales]
```

This distinction is important in real cubes, especially when dimensions use composite keys.

---

## 6️⃣ Common use cases

Use `Properties()` when you need to:

- display descriptive labels in reports,
- debug or inspect cube metadata,
- retrieve custom dimension attributes,
- build calculated measures from member attributes,
- make query output more user-friendly .

For example, if a business user wants to see product list price alongside product name, `Properties("List Price")` is often the simplest solution .

---

## 7️⃣ When not to use it

`Properties()` is not the right tool when you actually need a measure from fact data. It reads member metadata, not transactional or aggregated measure values .

Also, if the property you need is already available through a dimension property in the axis clause, you may not need a calculated member at all. In that case, `DIMENSION PROPERTIES` can be a better fit for presentation-only scenarios .

---

## 8️⃣ Practical rule

A good rule is:

- use `Properties()` when you want a value attached to a member,
- use measures when you want business facts,
- use `DIMENSION PROPERTIES` when you want metadata displayed directly in the result set .

That mental model makes MDX easier to design and debug.

---

## Conclusion

The `Properties()` function is one of the most useful MDX functions for working with cube metadata. In Adventure Works, it is especially helpful for reading date, product, and customer attributes in a clean and reusable way .

If you understand `CurrentMember.Properties(...)`, you can write more readable queries, create smarter calculated members, and build richer analytical output with less effort.

---





## Sources

- [Properties (MDX) - SQL Server | Microsoft Learn](https://learn.microsoft.com/en-us/sql/mdx/properties-mdx?view=sql-server-ver17)
- [User-Defined Member Properties (MDX) | Microsoft Learn](https://learn.microsoft.com/en-us/analysis-services/multidimensional-models/mdx/mdx-member-properties-user-defined-member-properties?view=sql-analysis-services-ver17)
- [Intrinsic Member Properties (MDX) | Microsoft Learn](https://learn.microsoft.com/en-us/analysis-services/multidimensional-models/mdx/mdx-member-properties-intrinsic-member-properties?view=sql-analysis-services-ver17)
