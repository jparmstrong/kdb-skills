---
name: kdb-q
description: Write, debug, and optimize kdb+/q code. Covers q syntax, qSQL queries, tables, joins, IPC, iterators, and HDB architecture. Use when working with kdb+ databases, writing q scripts, building real-time or historical data systems, or any task involving the q language.
license: Apache-2.0
compatibility: Requires a kdb+ installation (q binary). For HDB tasks, requires filesystem access.
metadata:
  author: kdb-skills2
  version: "1.0"
---

# kdb/q Skill

You are working with **kdb+**, a columnar time-series database, and **q**, its integrated programming language. kdb+ is optimized for high-frequency, ordered data — queries over time-series data are extremely fast when data is sorted and attributed correctly.

## Core language rules

**Evaluation is right-to-left** — the most common source of bugs for newcomers:
```q
2 * 3 + 1   / = 2 * 4 = 8  (NOT 7)
```
Use parentheses to override: `(2 * 3) + 1  / = 7`

**Whitespace matters in expressions** — space between a function name and `[` means application; no space means indexing:
```q
f [x]   / f applied to x (same as f[x])
t[`col] / index table t by symbol `col
```

**Atoms vs. lists** — most operators are implicitly vectorized; loops are rarely needed:
```q
1 2 3 + 10       / 11 12 13  (not an error)
1 2 3 = 2        / 010b
```

**Symbols use backtick**:
```q
`aapl`goog`msft  / list of 3 symbols
`price           / single symbol
```

**Strings are char vectors** (type 10h), not a primitive type:
```q
"hello"          / char vector, not a string atom
"hello" ~ "hello" / 1b
```

## Defining functions

```q
f:{[x;y] x + y}     / named args
g:{x * x}           / implicit args: x, y, z (up to 3)
add:{x+y}           / x=first, y=second, z=third arg
```

Lambdas are first-class. Apply with `[]` or as prefix/infix:
```q
f[3;4]              / 7
{x+y}[3;4]          / 7
```

## Tables

Tables are lists of dictionaries, or equivalently, column-oriented:
```q
t:([]sym:`a`b`c; price:1.1 2.2 3.3; size:100 200 300)
```

**Keyed table** (like a primary key):
```q
kt:([sym:`a`b`c] price:1.1 2.2 3.3)
```

Access columns as vectors: `t`price` or `t[`price]`

## qSQL — core syntax

```q
select [cols] from t [where cond] [by groups]
update col:expr from t [where cond]
delete [cols] from t [where cond]
exec [cols] from t [where cond]   / returns vector, not table
```

**Critical gotcha**: `cond` inside `$[...]` (ternary/cond) does NOT work in qSQL — wrap in a lambda:
```q
/ WRONG:
select x:$[a>0;a;0] from t

/ RIGHT:
select x:{$[x>0;x;0]}each a from t
/ or:
select x:?[a>0;a;0i] from t   / using vector conditional
```

**Vector conditional** (correct form for qSQL computed columns):
```q
?[condition_bool_vec; true_vec; false_vec]
```

**Where clause performance** — list multiple constraints as separate phrases, most restrictive first:
```q
/ Good — stops early:
select from t where date=2024.01.15, sym=`AAPL

/ Less efficient — evaluates all:
select from t where (date=2024.01.15) and sym=`AAPL
```

See [references/qsql.md](references/qsql.md) for advanced patterns.

## Common joins

| Join | Use case |
|------|----------|
| `lj` | Left join on keyed table — preserves all left rows |
| `ij` | Inner join on keyed table — only matching rows |
| `aj` | As-of join — match on time column (most recent prior) |
| `wj` | Window join — aggregate over time windows |
| `uj` | Union join — combine all rows from both tables |

```q
/ Left join: add ref data to trade
trade lj ([sym:`AAPL`GOOG] sector:`Tech`Tech)

/ As-of join: get prevailing quote at trade time
aj[`sym`time; trade; quote]
```

See [references/joins.md](references/joins.md) for all join types and gotchas.

## Iterators (map/reduce)

Prefer implicit iteration — most q operators already vectorize. Use iterators only when needed:

```q
count each ("hello";"world")   / 5 5 (each = map)
sum over 1 2 3 4 5             / 15  (over = fold/reduce)
sums 1 2 3 4 5                 / 1 3 6 10 15 (scan = cumulative)
{x+y}prior 1 2 3 4             / differences/running ops
```

See [references/iterators.md](references/iterators.md) for full iterator syntax.

## IPC — connecting processes

```q
h:hopen `::5001           / connect to localhost:5001
h"select from trade"      / sync query (blocks)
neg[h]"a:10"              / async (non-blocking)
hclose h                  / always close when done
```

See [references/ipc.md](references/ipc.md) for .z callbacks and production patterns.

## HDB — historical database

A typical HDB on disk:
```
hdb/
├── 2024.01.15/
│   └── trade/
│       ├── sym
│       ├── price
│       └── size
├── 2024.01.16/
│   └── trade/
│       ├── ...
└── sym
```

Load with: `\l hdb/`

Partition queries automatically skip irrelevant dates. **Always** include the partition column (`date`) in your where clause when possible.

See [references/hdb.md](references/hdb.md) for saving/loading, splayed tables, and HDB maintenance.

## Gotchas

- **Top-level `/ ` opens a multi-line comment block** — a line starting with `/ ` (slash space) at the top level of a script consumes everything until a `\` terminator; the entire script can silently disappear into a comment. Use inline comments (`code / note`) or confine `/ comment` lines inside function bodies `{}`
- **Integer division**: `7 % 2` = `3.5` (q uses `%` for division, returns float); use `div` for integer division: `7 div 2` = `3`
- **Float-to-long cast rounds, not truncates**: `` `long$42.9 `` = `43`, not `42`; use `floor`/`ceiling` for explicit control
- **Null values per type**: long=`0N`, int=`0Ni`, float=`0n` (displayed), date=`0Nd`, symbol=`` ` ``; `null x` tests for null; arithmetic propagates nulls
- **`mod` argument order matters**: `a mod b` when `a < b` returns `a` unchanged (e.g., `100 mod 200` = `100`, not `0`); always write `value mod divisor`, never `divisor mod value`
- **null = null is `1b`**: comparing two nulls of the same type returns true (`0n>=0n` = `1b`), unlike SQL; guard aggregated comparisons with `not null` before testing (e.g., `(not null x), x>0`)
- **`` `minute$timestamp `` loses the date** — both `2024.01.15D09:32:00` and `2024.01.16D09:32:00` cast to `09:32`; for multi-day data use `xbar` on the timestamp instead: `0D00:01:00 xbar time`
- **`xbar` on `time` type (19h) requires `int` conversion** — `0D00:05:00 xbar 09:30:00.000t` fails; use `` `minute$5 xbar`int$`minute$time `` for N-minute bars on time columns
- **`where` needs a boolean vector** (type `1h`): `where 10101b` works; `where 1 0 1 0 1` throws a type error — use `where (list)=value` to produce booleans
- **`=` is not assignment**: `:` assigns (`x:5`); `=` tests equality (`x=5` returns boolean)
- **Empty list type**: `()` is generic empty (type 0h); typed: `0#0i` (empty int), `0#0.0` (empty float), `` `$() `` (empty symbol)
- **`#` is take, not count**: `3#1 2 3 4 5` = `1 2 3`; use `count` for length
- **`!` is overloaded**: `a!b` creates a dict; `` `k xkey t `` keys a table; `key d` returns keys; `!t` also keys a table
- **`til` generates 0..n-1**: `til 5` = `0 1 2 3 4`
- **Global assignment inside a function** requires `::`: `f:{a::10}` sets global `a`; `a:10` inside `{}` is local
- **Attribute `s#` on unsorted data causes silently wrong query results** — always verify sort before applying sorted attribute
- **`select` returns a table; `exec` returns a vector/dict** — `exec price from t` returns a float vector (type `9h`), not a table
- **`fby` for within-group filtering**: `select from t where price=(max;price) fby sym` — gets rows with max price per sym
- **Functional qSQL** required for dynamic column names: `?[t;where_list;by_dict;select_dict]`; avoid `value` string eval in production

## Data types quick reference

See [references/datatypes.md](references/datatypes.md) for the full type table.

Most common:
```q
42          / long (7h)
42i         / int  (6h)
42.0        / float (9h)
2024.01.15  / date (14h)
09:30:00.000 / time (19h)
2024.01.15D09:30:00.000000000 / timestamp (12h)
`AAPL       / symbol (11h)
"x"         / char (10h)
```

Check type with: `type x`
Cast with: `` `long$x ``, `` `date$x ``, or `"J"$"42"`

## Coding style

- **Less code means fewer bugs** — no padding, no redundant comments, no speculative abstraction
- **User-defined function calls use square brackets**: `f[x;y]` not `(f x)` or `(f[x;y])`
- **Built-in keywords without brackets where readable**: `count t`, `first x`, `type t`, `show t`
- **Single-line functions are fine** if not too long: `.f:{[x] x*x}`
- **Multi-line functions: closing `}` on its own line with one space before**:
```q
f:{[x;y]
  z:x+y;
  z*z
  }
```
- **Inline `/ comment`** after code is the preferred comment style; block comments (`/ ... \`) only when absolutely necessary and never at the top level of a script
- **Whitespace only when it aids clarity** — no decorative blank lines between related statements
