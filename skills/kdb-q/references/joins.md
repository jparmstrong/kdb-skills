# kdb/q Joins Reference

## Join overview

| Join | Type | Requires keyed? | Result rows | Notes |
|------|------|-----------------|-------------|-------|
| `lj` / `ljf` | Left | Yes (right) | All left rows | Fills unmatched with nulls |
| `ij` / `ijf` | Inner | Yes (right) | Matched rows only | Drops unmatched left |
| `ej` | Equi | No | Matched rows | Like ij but specify join cols explicitly |
| `uj` / `ujf` | Union | No | All rows, both tables | Concatenates; fills missing cols with null |
| `pj` | Plus | Yes (right) | All left rows | Adds values instead of replacing |
| `^` | Coalesce | Yes | All left rows | Merges, preserving non-null from right |
| `aj` / `aj0` | As-of | No | All left rows | Match by time (most recent prior) |
| `asof` | Simple as-of | No | All left rows | `aj` with all right cols |
| `wj` / `wj1` | Window | No | All left rows | Aggregate over time window |

## Keyed joins (lj, ij, ej, uj, pj)

The **right table must be keyed** for `lj`, `ij`, `pj`. Match is on key columns.

```q
trade:([]sym:`AAPL`GOOG`AAPL`AMZN; price:150.0 200.0 151.0 100.0)
ref:([sym:`AAPL`GOOG] sector:`Tech`Tech; exchange:`NASDAQ`NASDAQ)

/ lj: keep all left rows, fill unmatched with null
trade lj ref
/ sym  price sector exchange
/ AAPL 150   Tech   NASDAQ
/ GOOG 200   Tech   NASDAQ
/ AAPL 151   Tech   NASDAQ
/ AMZN 100              ← nulls for AMZN (not in ref)

/ ij: only rows with a match in ref
trade ij ref
/ sym  price sector exchange
/ AAPL 150   Tech   NASDAQ
/ GOOG 200   Tech   NASDAQ
/ AAPL 151   Tech   NASDAQ  ← AMZN dropped
```

**`ljf` / `ijf`**: "foreign" variants — tolerate mismatched schemas (fill with type-specific nulls). Use when join might fail due to schema differences.

### Equi join (ej)

```q
/ ej[join_cols; left_table; right_table]  — neither needs to be keyed
ej[`sym; trade; ([]sym:`AAPL`GOOG; sector:`Tech`Tech)]
```

### Plus join (pj)

```q
/ Values from matching rows are ADDED, not replaced
a:([]x:1 2 3; y:10 20 30)
b:([x:1 2] y:1000 2000)
a pj b
/ x y
/ 1 1010   ← 10 + 1000
/ 2 2020   ← 20 + 2000
/ 3 30     ← no match, unchanged
```

Useful for accumulating adjustments.

### Union join (uj)

```q
/ Concatenates all rows; columns must be compatible (or use ujf)
a:([]x:1 2 3; y:10 20 30)
b:([]x:2 3 4; y:20 30 40)
a uj b
/ Returns 6 rows (all from both)
```

**Note**: `uj` does NOT deduplicate. It is a union in the sense of "combine all", not set union.

## Coalesce (^)

```q
/ Merge two keyed tables, preferring non-null values from right
a:([sym:`AAPL`GOOG] price:100.0 0n; sector:`Tech`)
b:([sym:`AAPL`GOOG] price:0n 200.0; mktcap:1e12 1e11)
a ^ b   / fills null price in a with price from b
```

## As-of join (aj)

`aj` is the most important join for time-series work. It finds the **most recent prior record** in the right table for each row of the left table.

```q
/ aj[join_cols; left; right]
/ Last element of join_cols is the time column

trade:([]sym:`AAPL`GOOG`AAPL;
       time:09:30:00.000 09:30:01.000 09:30:02.000t;
       price:150.0 200.0 151.0)

quote:([]sym:`AAPL`AAPL`GOOG;
       time:09:29:59.000 09:30:00.500 09:30:00.000t;
       bid:149.5 149.8 199.5;
       ask:150.5 150.2 200.5)

aj[`sym`time; trade; quote]
/ For each trade row: finds the quote with same sym AND largest time <= trade time
/ AAPL @ 09:30:00 → quote @ 09:29:59 (bid=149.5)
/ GOOG @ 09:30:01 → quote @ 09:30:00 (bid=199.5)
/ AAPL @ 09:30:02 → quote @ 09:30:00.5 (bid=149.8)
```

**aj vs aj0**: `aj` returns the prevailing value (last at or before); `aj0` excludes exact time matches (strictly before).

**Requirements**:
- Right table must be **sorted** by the time column (apply `` `s# ``)
- Right table should have a `` `p# `` or `` `g# `` attribute on the first join column (sym) for performance

```q
/ Optimized quote table for aj
quote:`sym`time xasc quote
update `p#sym from `quote    / parted attribute on sym
aj[`sym`time; trade; quote]
```

## Window join (wj / wj1)

More powerful than `aj` — aggregates over an explicit time window.

```q
/ wj[windows; join_cols; left; (right; agg1; agg2; ...)]
/ windows: pair of (start_times; end_times) as list

trade:([]sym:`AAPL`GOOG`AAPL;
       time:09:30:00.000 09:30:01.000 09:30:02.000t;
       price:150.0 200.0 151.0)

quote:([]sym:`AAPL`AAPL`AAPL`GOOG;
       time:09:29:58.000 09:30:00.000 09:30:01.500 09:30:00.500t;
       bid:149.0 149.5 149.8 199.5)

/ For each trade, look 2 seconds before up to 0 seconds after
windows:(-00:00:02t+trade`time; trade`time)
wj[windows; `sym`time; trade; (quote; (avg;`bid); (max;`bid))]
```

**`wj` vs `wj1`**:
- `wj`: includes prevailing value (last value before window start), then aggregates window
- `wj1`: only aggregates values strictly within the window

Common aggregation functions: `avg`, `max`, `min`, `sum`, `count`, `last`, `first`.

## Join gotchas

- **`lj` loses right-table columns not in key if names clash** — rename conflicting cols before joining
- **`aj` right table must be sorted** — unsorted data returns wrong matches silently
- **`uj` does not dedup** — use `distinct` if needed
- **Time column in `aj` must be the last element** of the join column list
- **Keyed table `!` syntax**: `([keycol] valcol:...)` or `` `keycol xkey table ``
- **`lj` with multi-key**: `` ([sym:...; date:...] price:...) `` — key can be compound
- **Type mismatch in join columns causes error** — ensure both sides have same type (e.g., both `symbol`, not one `symbol` and one `string`)
