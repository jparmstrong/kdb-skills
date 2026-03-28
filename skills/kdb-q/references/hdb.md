# kdb/q HDB and File I/O Reference

## Basic save and load

```q
/ Save any value to disk
`:myfile set 42
`:data/mytable set ([]x:1 2 3; y:10 20 30)

/ Load from disk
get `:myfile              / 42
get `:data/mytable        / table

/ Shorthand: treat file path as a variable
`:data/mytable            / same as get `:data/mytable (read)
```

## Text and CSV I/O

```q
/ Write CSV
`:out.csv 0: csv 0: t           / save table t as CSV

/ Read CSV (specify types)
("SFI";enlist",") 0: `:data.csv / S=sym, F=float, I=int

/ Write plain text
`:out.txt 0: ("line1";"line2")

/ Read lines
read0 `:out.txt            / returns list of strings

/ Read binary
read1 `:out.bin            / returns byte vector
```

Type string for CSV reader: `"BXHIJEFCSPMDZNUVT"` matches type chars. Use space `" "` to skip a column.

## Splayed tables

A **splayed** table stores each column as a separate file in a directory:

```
mytable/
├── .d      ← column order file
├── sym     ← symbol vector (enumerated)
├── price   ← float vector
└── size    ← int vector
```

```q
/ Save splayed
`:hdb/mytable/ set t    / trailing slash = splayed

/ Load splayed (memory-mapped, not loaded until accessed)
\l hdb/
count trade             / loads on demand

/ The sym file in HDB root enumerates all symbols
`:hdb/sym set `symbol$()   / initialize (or it's auto-created)
```

Splayed tables are **memory-mapped** — only the accessed columns are loaded into RAM.

## Partitioned tables (HDB)

A **partitioned** database splits rows across directories by a partition column (usually `date`):

```
hdb/
├── 2024.01.15/
│   └── trade/
│       ├── .d
│       ├── sym
│       ├── price
│       └── size
├── 2024.01.16/
│   └── trade/
│       ├── ...
└── sym          ← global symbol file
```

```q
/ Load HDB
\l /path/to/hdb

/ Query — kdb automatically limits to matching partitions
select from trade where date=2024.01.15
select from trade where date within 2024.01.15 2024.01.20

/ date column is virtual (derived from directory name)
/ Never store `date` as a column in partitioned tables — it's automatic
```

**Always include the partition column in `where`** — omitting it causes a full scan of all partitions.

## Saving to HDB (.Q.dpft)

```q
/ .Q.dpft[dir; partition; parted_col; table]
.Q.dpft[`:hdb; 2024.01.15; `sym; trade]

/ This:
/   1. Saves trade as splayed to hdb/2024.01.15/trade/
/   2. Applies `p# attribute to sym column
/   3. Enumerates sym against hdb/sym
/   4. Writes .d file (column order)
```

After saving multiple days, load the HDB:
```q
\l hdb/
```

## .Q utilities for HDB

```q
.Q.w[]         / workspace memory stats
.Q.chk[`:hdb]  / check HDB integrity
.Q.par[dir;partitions;table]  / get table path for partition

/ Enumerate a symbol vector against the HDB sym file
.Q.en[`:hdb; ([]sym:`AAPL`GOOG)]

/ Load a partition into memory
.Q.ind[trade; 0 1 2]  / rows by index
```

## Partitioned HDB gotchas

- **Never store the partition column** (`date`) as a physical column — it's virtual
- **Partition column must be first in `where`** for query to skip partitions efficiently
- **All partitions must have identical column types** — a type mismatch causes load errors
- **sym file must be consistent** across partitions — always enumerate against the HDB sym, not a local enum
- **`.d` file** lists column order; missing `.d` = undefined column order
- **Compressed columns**: use `` `:path/col set -19!data `` or set compression params before saving

## Column compression

```q
/ Set compression: algorithm, logical block size, compression level
.z.zd:17 2 6   / lz4hc, 128KB blocks, level 6 (good default)
`:hdb/2024.01.15/trade/price set price_vec  / saves compressed

/ Check compression ratio
hcount `:hdb/2024.01.15/trade/price   / compressed size
count get `:hdb/2024.01.15/trade/price  / row count
```

## Appending / maintenance

```q
/ Append rows to an on-disk table
`:hdb/2024.01.15/trade/ upsert new_rows   / for splayed
.[`:hdb/2024.01.15/trade/price; (); ,; new_prices]  / amend column

/ Sort splayed table on disk
`sym`time xasc `:hdb/2024.01.15/trade/   / resort in place (splayed)

/ Re-apply attributes after modifying
@[`:hdb/2024.01.15/trade/sym; `; `p#]    / re-apply parted attr
```

## File handles for streaming writes

```q
/ Open file for appending
fh:hopen `:mylog.q

/ Write lines (appends)
fh enlist "first line"
fh enlist "second line"

/ Flush and close
hclose fh

/ Reading back
read0 `:mylog.q
```

## Typical HDB intraday write workflow

```q
/ RDB (real-time database): holds today's data in memory
trade:([]time:`timestamp$(); sym:`symbol$(); price:`float$(); size:`int$())

/ EOD save
eod:{[date]
  .Q.dpft[`:hdb; date; `sym; trade];  / save to HDB
  delete from `trade;                   / clear RDB
  \l hdb                                / reload HDB
  }
```
