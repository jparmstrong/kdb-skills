# kdb/q Data Types Reference

## Type table

| Type | n | char | Size | Null | Infinity | Example |
|------|---|------|------|------|----------|---------|
| Boolean | 1 | b | 1B | — | — | `1b`, `0b`, `101b` |
| Guid | 2 | g | 16B | `0Ng` | — | `"G"$"550e8400-e29b-41d4-a716-446655440000"` |
| Byte | 4 | x | 1B | `0x00` | — | `0x1f` |
| Short | 5 | h | 2B | `0Nh` | `0Wh` | `42h` |
| Int | 6 | i | 4B | `0Ni` | `0Wi` | `42i` |
| Long | 7 | j | 8B | `0N` (`0Nj`) | `0W` (`0Wj`) | `42` (default) |
| Real | 8 | e | 4B | `0Ne` | `0We` | `42e` |
| Float | 9 | f | 8B | `0n` | `0w` | `42.0` |
| Char | 10 | c | 1B | `" "` | — | `"x"` |
| Symbol | 11 | s | var | `` ` `` | — | `` `AAPL `` |
| Timestamp | 12 | p | 8B | `0Np` | `0Wp` | `2024.01.15D09:30:00.000000000` |
| Month | 13 | m | 4B | `0Nm` | `0Wm` | `2024.01m` |
| Date | 14 | d | 4B | `0Nd` | `0Wd` | `2024.01.15` |
| DateTime | 15 | z | 8B | `0Nz` | `0wz` | `2024.01.15T09:30:00` |
| Timespan | 16 | n | 8B | `0Nn` | `0Wn` | `0D01:30:00.000000000` |
| Minute | 17 | u | 4B | `0Nu` | `0Wu` | `09:30u` |
| Second | 18 | v | 4B | `0Nv` | `0Wv` | `09:30:00v` |
| Time | 19 | t | 4B | `0Nt` | `0Wt` | `09:30:00.000` |

Special:
| Type | n | Notes |
|------|---|-------|
| List | 0 | Generic mixed list |
| Dictionary | 99 | `d:a!b` |
| Table | 98 | Flip of column dict |
| Keyed table | 99 | Table of tables |
| Function | 100 | Lambda `{x+y}` |

## Type inspection

```q
type 42         / -7h  (negative = atom, 7 = long)
type 42 43      /  7h  (positive = list/vector)
type ()         /  0h  (generic list)
type (1;`a;"x") /  0h  (mixed = generic)
meta t          / table: col name, type char, foreign key, attribute
```

Type char in `meta` output matches the `char` column above.

## Casting

```q
`long$42.9       / 43  (ROUNDS, use floor/ceiling for truncation)
`int$42          / 42i
`float$42        / 42f
`date$0          / 2000.01.01  (kdb epoch = 2000.01.01)
`symbol$"AAPL"   / `AAPL
string `AAPL     / "AAPL"
`$"AAPL"         / `AAPL  (shorthand for `symbol$)
"J"$"42"         / 42j   (parse string to long)
"F"$"3.14"       / 3.14
"D"$"2024.01.15" / 2024.01.15
```

The cast char `"J"` is uppercase of the type char. For symbols use `` `$ ``.

## Epochs and time arithmetic

kdb+ epoch is **2000.01.01**. Dates are stored as days since epoch.

```q
`int$2000.01.01  / 0
`int$2000.01.02  / 1
2024.01.15 - 2024.01.01  / 14 (days as int)

/ Timestamps: nanoseconds since 2000.01.01D00:00:00
`long$2000.01.01D00:00:00.000000000  / 0

/ Common time arithmetic
.z.d              / today's date
.z.t              / current time (local)
.z.p              / current UTC timestamp
.z.P              / current local timestamp

/ Build timestamp from a date variable + a time value
/ (`timestamp$d) + `timespan$`time$09:30:00.000t  →  d D09:30:00.000000000
/ Step by step: date → timestamp (midnight), time → timespan, then add
(`timestamp$2024.01.15) + `timespan$`time$09:30:45.123t
/ = 2024.01.15D09:30:45.123000000

/ date literal + time literal does NOT work (type mismatch)
/ 2024.01.15 + 09:30:00.000t   → error
```

## Null handling

```q
null 0N      / 1b
null 0Ni     / 1b
null 0n      / 1b (float null)
null `       / 1b (symbol null)
null " "     / 1b (char null)

/ null = null is 1b — unlike SQL, same-type nulls compare as equal
0n >= 0n     / 1b  (float null "equals" float null)
0N = 0N      / 1b
/ Guard aggregated comparisons: always check (not null x) before comparing
/ Integer/long nulls act as minimum value in inequality comparisons
0Ni > 5i     / 0b  (int null is minimum int, smaller than 5)
0N > 5       / 0b  (long null is minimum long)
/ Float null also acts as minimum in inequality
0n > 0.5     / 0b

/ Arithmetic with nulls propagates null
0N + 1       / 0N
0Ni * 5i     / 0Ni

/ Fill nulls
0N^42        / 42  (fill null with 42)
1.5^0n       / 1.5

/ fills (fill null with preceding value)
fills 1 0N 0N 4 0N   / 1 1 1 4 4
```

## Attributes

Attributes boost performance of certain operations:

| Attribute | Symbol | Use case |
|-----------|--------|----------|
| Sorted | `` `s# `` | Binary search on sorted vectors |
| Unique | `` `u# `` | Hash lookup (unique values) |
| Grouped | `` `g# `` | Group index for repeated symbols |
| Parted | `` `p# `` | Contiguous groups (HDB partition key) |

```q
`s#`a`b`c`d     / sorted — enables binary search
`u#1 2 3 4      / unique — enables hash lookup
`g#`a`b`a`c`b   / grouped — index by value
`p#`a`a`b`b`c   / parted — groups must be contiguous
```

**Warning**: Applying `` `s# `` to unsorted data does not error — it silently marks it sorted, causing wrong binary search results. Use `(asc t)~t` or check `attr` before applying.

Attributes are lost after most modifications. Re-apply after `update` or inserts if needed.
