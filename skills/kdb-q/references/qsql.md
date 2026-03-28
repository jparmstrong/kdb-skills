# qSQL Reference

## Statement forms

```q
select  [cols]       from t [where c] [by g]
exec    [cols]       from t [where c] [by g]
update  col:expr     from t [where c]
delete  [cols | rows] from t [where c]
```

`select` returns a **table**. `exec` returns a **vector** (or dict/table with `by`).

## Execution order

`FROM → WHERE → BY → SELECT → LIMIT`

The where clause runs before column expressions are evaluated. You cannot reference a computed column in `where` — filter on the raw source columns.

## select

```q
t:([]sym:`AAPL`GOOG`MSFT`AAPL; price:150.0 200.0 50.0 151.0; vol:1000 500 2000 800i)

/ All rows
select from t

/ Filter
select from t where sym=`AAPL

/ Specific columns
select sym, price from t

/ Computed column
select sym, ret:price%first price from t

/ Multiple conditions (AND) — comma = successive filter, most restrictive first
select from t where sym=`AAPL, price>150

/ Aggregation with grouping
select cnt:count i, avg_price:avg price by sym from t

/ top-N per group using fby
select from t where price=(max;price) fby sym

/ Distinct
select distinct sym from t

/ Limit rows (last N)
select[-3] from t     / last 3 rows
select[3] from t      / first 3 rows
```

## exec

```q
/ Returns vector, not table
exec price from t              / 150 200 50 151f
type exec price from t         / 9h (float vector)
type select price from t       / 98h (table)

/ With by: returns dict
exec price by sym from t
/ `AAPL`GOOG`MSFT!(150 151f;,200f;,50f)

/ Distinct values
exec distinct sym from t
```

## update

```q
/ In-memory update (does not modify t, returns new table)
t2:update price:price*1.1 from t

/ Conditional update using vector conditional
update adj_price:?[sym=`AAPL;price*1.1;price] from t

/ Update matching rows only
update price:price*1.1 from t where sym=`AAPL

/ Add new column
update sector:`Tech from t

/ In-place update on global table
update price:price*1.1 from `t     / backtick before table name!
```

## delete

```q
/ Delete rows
delete from t where sym=`MSFT

/ Delete columns
delete vol from t

/ In-place delete on global
delete from `t where price<100
```

## Where clause patterns

```q
/ Equality
where sym=`AAPL

/ Range
where price within 100 200     / 100 <= price <= 200

/ List membership
where sym in `AAPL`GOOG

/ Pattern match (symbols)
where sym like "A*"

/ Compound: comma = short-circuit AND (faster than 'and')
where date=2024.01.15, sym=`AAPL, price>100

/ NOT
where not sym=`AAPL
where sym<>`AAPL

/ Null check
where not null price
where null price

/ Using fby for within-group filters
where vol=(max;vol) fby sym            / rows with max vol per sym
where vol>(avg;vol) fby sym            / rows above average per sym
```

**Performance**: List conditions most restrictive first. The comma form stops evaluating later conditions for rows already filtered out.

## By clause and aggregation

```q
/ Count rows per group
select count i by sym from t

/ Multiple aggregations
select cnt:count i, avg_price:avg price, ttl_vol:sum vol by sym from t

/ Multi-column key
select avg price by sym, (`date$time) from trades

/ Aggregation functions
/ count, sum, avg, min, max, first, last, med, dev, var, prd
/ all return scalar per group

/ Custom aggregation
select custom:{sum x*x} price by sym from t
```

## Vector conditional (use instead of $[] in qSQL)

```q
/ $[cond;a;b] does NOT work in qSQL column expressions — use ?[;;]:
select adjprice:?[sym=`AAPL; price*1.1; price] from t

/ Nested conditions
select tier:?[price>150;`high;?[price>100;`mid;`low]] from t

/ Multi-arg version with lists
?[100 0 200 0b; 1 2 3 4; 10 20 30 40]  / 1 20 3 40
```

## Functional qSQL

Use when column names are determined at runtime:

```q
/ ?[table; where_list; by_dict; select_dict]
/ ![table; where_list; by_dict; update_dict]

t:([]sym:`a`b`c; price:1.1 2.2 3.3; vol:100 200 300)

/ select sym, price from t where price > 2.0
?[t; enlist(>;`price;2.0); 0b; `sym`price!`sym`price]

/ select avg price by sym from t
?[t; (); (enlist `sym)!(enlist `sym); (enlist `avg_price)!(enlist (avg;`price))]

/ Dynamic column name
col:`price
?[t; (); 0b; (enlist col)!(enlist col)]

/ update newcol:price+0.5 from t
![t; (); 0b; (enlist `newcol)!(enlist (+;`price;0.5))]

/ delete vol from t
![t; (); 0b; enlist `vol]

/ delete rows where price<2
![t; enlist(<;`price;2.0); 0b; `symbol$()]
```

Where list format: empty `()` = no filter; each condition is a parse tree: `(op;lhs;rhs)`.
By dict: `0b` = no grouping; `cols!cols` for grouping.
Select dict: `cols!cols` for passthrough; `newname!(func;col)` for aggregation.

## Common qSQL patterns

```q
/ Time-bucketed OHLCV — approach depends on the type of the time column:
/ timestamp (12h): use timespan xbar — PRESERVES date, safe for multi-day data
select open:first price,high:max price,low:min price,close:last price,vol:sum size
  by sym,0D00:05:00 xbar time from trades

/ time (19h): xbar with timespan FAILS — use `minute$ instead (loses date)
select open:first price,high:max price,low:min price,close:last price,vol:sum size
  by sym,`minute$time from trades            / 1-min bars

/ time (19h) N-minute bars: bucket via int xbar on minute
select open:first price,high:max price,low:min price,close:last price,vol:sum size
  by sym,`minute$5 xbar`int$`minute$time from trades  / 5-min bars

/ WARNING: `minute$timestamp loses the date — both dates give 09:32
/ `minute$2024.01.15D09:32:00  →  09:32
/ `minute$2024.01.16D09:32:00  →  09:32
/ Use xbar on timestamp columns when data spans multiple dates.

/ Rolling window (last 5 rows per sym)
select[-5] from trades where sym=`AAPL

/ Rank within group
update rnk:rank price by sym from t

/ Cumulative sum of column
select sym, cumsz:sums size from t

/ Previous row value
select sym, price, prev_price:prev price from t

/ Join and aggregate in one
select avg price by sym from trade lj ref

/ Coalesce nulls across joined tables
select sym, price^0.0 from t   / replace null price with 0
```

## System \t timing

```q
\t select from trade where sym=`AAPL   / time in ms
\ts select from trade where sym=`AAPL  / time + bytes allocated
```
