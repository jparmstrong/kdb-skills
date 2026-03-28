# kdb/q Iterators Reference

## Key principle: prefer implicit iteration

Most q operators are already **atomic** (automatically vectorized). Iterators are needed only when explicit repetition is required.

```q
/ Implicit — no iterator needed:
1 2 3 + 10        / 11 12 13
1 2 3 = 2         / 010b
upper "hello"     / "HELLO" (applies to each char)

/ Wrong — adding unnecessary iterator:
upper each "hello"  / works but wasteful; upper is already atomic
```

## Map iterators

### Each (`'`)

Apply a function to each element of one or more lists:

```q
count each ("hello";"world";"!")     / 5 5 1
reverse each (1 2 3; 4 5 6)         / (3 2 1; 6 5 4)
{x*x} each 1 2 3 4 5                 / 1 4 9 16 25

/ Each with two args (zips two lists)
(+') [1 2 3; 10 20 30]               / 11 22 33  (same as +' applied)
```

### Each-left (`\:`) and Each-right (`/:`)

Apply a binary function fixing one argument:

```q
/ Each-left: fix left arg, iterate right
1 2 3 +\: 10 20
/ 11 21
/ 12 22
/ 13 23
/ (each left element paired with whole right list)

/ Each-right: fix right arg, iterate left
1 2 3 +/: 10 20
/ 11 12 13
/ 21 22 23
/ (whole left list paired with each right element)
```

Mnemonic: `\:` leans left (left is the "anchor"), `/:` leans right.

### Each-prior (`':`)

Apply a binary function to each consecutive pair:

```q
(-':) 1 3 6 10      / 1 2 3 4  (differences)
(*':) 1 2 6 24      / 1 2 3 4  (ratios)

/ With a seed (initial prior value)
10 -': 1 3 6 10     / -9 2 3 4  (seed is 10)

/ Practical: compute returns
price:100 102 105 103f
(price - prev price) % prev price   / pct change (using prev keyword)
(-':) price % first price            / cumulative returns
```

### Parallel each (peach)

Like `each` but runs in parallel across threads:

```q
/ Requires q started with multiple threads: q -s 4
{heavyCalc x} peach big_list
```

Only beneficial for CPU-bound work on large datasets. Data must be thread-safe (no global writes inside the function).

## Accumulator iterators

### Over (`/`) — fold/reduce

Applies a function cumulatively, returning only the **final result**:

```q
(+/) 1 2 3 4 5      / 15  (sum)
(*/) 1 2 3 4 5      / 120 (product)
(,/) (1 2;3 4;5 6)  / 1 2 3 4 5 6 (flatten one level)
(max/) 3 1 4 1 5 9  / 9

/ With seed (left arg = initial value)
100 (+/) 1 2 3 4 5  / 115  (start from 100)

/ Converge (no count): iterate until result stops changing
({x+1} /) 0         / infinite loop! — use do/while for convergence
({abs[x-y]<0.001} /) 1.0  / converge form: stop when f[prev;curr] is true
```

### Scan (`\`) — cumulative/running result

Like Over but returns **all intermediate results**:

```q
(+\) 1 2 3 4 5      / 1 3 6 10 15  (running sum = sums)
(*\) 1 2 3 4 5      / 1 2 6 24 120 (running product)
(max\) 3 1 4 1 5 9  / 3 3 4 4 5 9  (running max)

/ Convenient shorthand keywords:
sums  1 2 3 4 5     / 1 3 6 10 15
prds  1 2 3 4 5     / 1 2 6 24 120
maxs  3 1 4 1 5 9   / 3 3 4 4 5 9
mins  3 1 4 1 5 9   / 3 1 1 1 1 1
```

### Do (n-times form)

Apply a function exactly N times:

```q
/ Over N times
5 (+/) 1 2 3 4 5     / NOT n-times — that's the seed form

/ N-times with do
10 {x+1}/ 0          / 10 (apply 10 times starting from 0)
10 {x,x}/ "a"        / 1024 copies of "a"

/ Scan N times
5 {x+1}\ 0           / 0 1 2 3 4 5 (intermediate results)
```

### While (convergence form)

Apply until a condition is met:

```q
/ While: f[x] is truthy → keep applying g
{x<100} {x*2}/ 1     / 128 (double until >= 100)

/ Two-arg while: stop when f[prev;curr] is true
{y-x<0.001} {(x+y)*0.5}/ (0f; 1f)  / converge to midpoint
```

## Iterator notation summary

| Syntax | Name | Description |
|--------|------|-------------|
| `f'` | Each | Map f over each element |
| `f\:` | Each-left | Fix left arg, map over right |
| `f/:` | Each-right | Fix right arg, map over left |
| `f':` | Each-prior | Apply to consecutive pairs |
| `f peach` | Parallel each | Like each, multi-threaded |
| `f/` | Over | Fold (final result only) |
| `f\` | Scan | Fold (all intermediate results) |
| `n f/` | Do (over) | Apply f exactly n times |
| `n f\` | Do (scan) | Apply f n times, keep all |
| `p f/` | While (over) | Apply until predicate false |
| `p f\` | While (scan) | Apply until predicate false, keep all |

## Practical patterns

```q
/ Running average (window)
mavg[5; 1 2 3 4 5 6 7 8 9 10]   / 5-period moving avg

/ Running correlation
mcorr[5; price1; price2]

/ Running sum from scratch
(+\) price                        / cumulative sum

/ Apply function to each column
{avg x} each flip t               / avg of each column

/ Apply multiple aggregations
(sum;avg;min;max) @\: 1 2 3 4 5   / each-right across functions

/ Nested each for matrix ops
{x+\:y} . (1 2 3; 10 20 30)      / outer addition

/ Flatten nested lists
raze (1 2;3 4;5)                  / 1 2 3 4 5 (same as (,/)...)
```

## Iterator vs explicit loop

Iterators are almost always preferred over `do`/`while` control words:

```q
/ Avoid this:
result:();
i:0;
while[i<count t;
  result,:someFunc t[i];
  i+:1];

/ Prefer this:
result:someFunc each t
```

Explicit loops have higher overhead and are harder to read. Use iterators.
