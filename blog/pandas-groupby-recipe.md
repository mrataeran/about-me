[Home](../index.html) | [Blog](index.html)

# Pandas `groupby()` without confusion: a practical recipe

## Introduction
If you have worked with real datasets in Pandas, you have probably run into this moment: you know you need “one row per group” or “a summary by category,” but your `groupby()` output turns into a MultiIndex puzzle, your columns disappear, or you end up with a Series when you wanted a DataFrame. This post is a practical recipe for using `groupby()` confidently.

The core idea is simple: **a `groupby()` splits your data into groups, applies a function to each group, and then combines the results**. Most confusion comes from not being explicit about (1) what you are grouping by, (2) what columns you are summarizing, and (3) what shape you want the result to have.

By the end, you will be able to:
- build clean “summary tables” for reports,
- compute multiple metrics in one pass,
- avoid common index and column surprises, and
- apply the same pattern to your own Stat 386 projects.

## The mental model: split, apply, combine
Think of `groupby()` as a pipeline:

1. **Split**: partition rows by one or more keys (like `month`, `carrier`, `species`, `team`).
2. **Apply**: run an aggregation or transformation on each partition.
3. **Combine**: stack the results back together into a single object.

In code, it looks like this:

```python
summary = (
    df
    .groupby("group_key")        # split
    ["value_column"]             # choose what to summarize
    .agg("mean")                 # apply
)                                # combine happens automatically
```

The biggest improvement you can make is to **choose the output shape on purpose**:
- Do you want the grouping keys to stay as an index, or become normal columns?
- Do you want a single value per group (aggregation), or a same-length column (transform)?
- Do you want one metric or many?

## A small example dataset
We will use a simple “transactions” dataset. The example is small enough to reason about, but the pattern scales to big datasets.

```python
import pandas as pd

df = pd.DataFrame({
    "customer": ["A", "A", "A", "B", "B", "C"],
    "month":    ["Jan", "Jan", "Feb", "Jan", "Feb", "Feb"],
    "amount":   [120, 80, 200, 50, 75, 300],
    "returned": [0, 1, 0, 0, 0, 1],
})
df
```

Visually, you can read this as: each row is an order, and you want summaries like “total spending per customer” or “return rate per month.”

## Recipe 1: one metric per group
Goal: **Total amount per customer**.

```python
spend_by_customer = (
    df.groupby("customer", as_index=False)["amount"]
      .sum()
      .rename(columns={"amount": "total_amount"})
)
spend_by_customer
```

Two details matter:
- `as_index=False` keeps `customer` as a regular column (often easier for merges and plotting).
- `.rename(...)` makes your columns self-explanatory.

If you omit `as_index=False`, you will get `customer` as an index. That is not “wrong,” but it often becomes annoying later when you try to join or export.

## Recipe 2: multiple metrics in one table
Goal: **Per customer: count of orders, total, mean, and max**.

```python
customer_metrics = (
    df.groupby("customer", as_index=False)
      .agg(
          orders=("amount", "size"),
          total_amount=("amount", "sum"),
          avg_amount=("amount", "mean"),
          max_amount=("amount", "max"),
      )
)
customer_metrics
```

This is the most useful modern pattern for Stat 386 work. It is readable because each output column is named, and each one points to:
- a source column (`"amount"`) and
- an aggregation (`"sum"`, `"mean"`, etc.).

### Common aggregation choices
Here is a quick reference for typical project needs:

| Goal | Pattern | Notes |
|---|---|---|
| How many rows in each group? | `("col", "size")` | Works even if values are missing |
| How many non-missing values? | `("col", "count")` | Ignores `NaN` |
| Total | `("col", "sum")` | For numeric columns |
| Average | `("col", "mean")` | Watch out for outliers |
| Minimum or maximum | `("col", "min")`, `("col", "max")` | Useful for date ranges too |

## Recipe 3: rates and percentages (including booleans)
Goal: **Return rate by month**.

Your `returned` column is 0/1. The mean of 0/1 is a proportion.

```python
return_rate_by_month = (
    df.groupby("month", as_index=False)
      .agg(
          orders=("returned", "size"),
          return_rate=("returned", "mean"),
      )
)

return_rate_by_month["return_rate"] = (return_rate_by_month["return_rate"] * 100).round(1)
return_rate_by_month
```

This pattern is common in course datasets too. For example:
- `cancelled` flights (0/1),
- “is_late” flags,
- “above_threshold” indicators,
- any binary label.

## Recipe 4: keep the original row count (use `transform`)
Sometimes you want a per-row feature like “customer average amount,” where the average is computed per customer but attached to each row.

If you use `.agg("mean")`, you get one row per customer, which will not align with the original DataFrame. The fix is `.transform("mean")`.

```python
df["customer_avg_amount"] = df.groupby("customer")["amount"].transform("mean")
df
```

Use this when you are engineering features for modeling or you want to compare each row to its group baseline.

## Recipe 5: top-k within groups
A common workflow is: “for each customer, keep the largest order.” This is not a single aggregation when you want the whole row, so you use sorting plus `head`.

```python
top_order_per_customer = (
    df.sort_values(["customer", "amount"], ascending=[True, False])
      .groupby("customer", as_index=False)
      .head(1)
)
top_order_per_customer
```

This gives you full rows, not just a summary number.

## Debugging checklist for `groupby()` confusion
When your result looks weird, walk through this checklist:

1. **What is the grouping key?**  
   Print `df[group_cols].drop_duplicates().head()` to sanity-check categories.

2. **Are you getting a Series or a DataFrame?**  
   - `df.groupby(...)[col].agg(...)` often returns a Series.
   - `df.groupby(...).agg(...)` usually returns a DataFrame.

3. **Did your group key become an index?**  
   Use `as_index=False` or call `.reset_index()` after the aggregation.

4. **Did you mean aggregation or transform?**  
   - Aggregation shrinks rows (one row per group).
   - Transform keeps rows (same length as original).

5. **Are you mixing missing values?**  
   Remember:
   - `size` counts all rows,
   - `count` ignores `NaN`.

## Conclusion and call to action
`groupby()` becomes much easier when you treat it like a recipe:
- choose your grouping key,
- choose your summarized columns,
- choose your output shape,
- then use either `.agg(...)` for summaries or `.transform(...)` for per-row features.

Next steps:
1. Take a dataset you have used in Stat 386 (flights, climate, or any lab dataset).
2. Write one table that summarizes something meaningful by a category you care about (month, carrier, station, etc.).
3. Add at least two metrics in a single `.agg(...)` call, and rename them to clear column names.
4. If you need the summary back in the original table, try one `.transform(...)` feature too.

If you do those four steps, you will be using `groupby()` like a pro on real assignments.
