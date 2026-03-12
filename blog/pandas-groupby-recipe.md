[Home](../index.html) | [Blog](index.html) | [Pandas Groupby Recipe](pandas-groupby-recipe.html) | [Macroeconomic Data Project](macroeconomic-data-project.html)

# Pandas `groupby()` without confusion: a practical recipe

## Introduction
If you have worked with real datasets in Pandas, you have probably run into this moment: you know you need “one row per group” or “a summary by category,” but your `groupby()` output turns into a MultiIndex puzzle, your columns disappear, or you end up with a Series when you wanted a DataFrame. This post is a practical recipe for using `groupby()` confidently.

The core idea is simple: **a `groupby()` splits your data into groups, applies a function to each group, and then combines the results**. Most confusion comes from not being explicit about (1) what you are grouping by, (2) what columns you are summarizing, and (3) what shape you want the result to have.

By the end, you will be able to:
- build clean summary tables for reports,
- compute multiple metrics in one pass,
- avoid common index and column surprises,
- and apply the same pattern to your own Stat 386 projects.

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