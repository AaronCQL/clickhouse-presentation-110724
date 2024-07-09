---
title: "ClickHouse: Powering Coinhall's Real-Time Blockchain Data Platform"
# You can also start simply with 'default'
theme: default
# apply unocss classes to the current slide
class: text-2xl text-left
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# https://sli.dev/guide/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations#slide-transitions
transition: fade
# enable MDC Syntax: https://sli.dev/guide/syntax#mdc-syntax
mdc: true
---
# ClickHouse

Powering Coinhall's Real-Time Blockchain Data Platform

---
layout: statement
---

### Coinhall - Trading Terminal for Decentralised Exchanges

<img src="/charts.png" class="m-auto h-110 rounded ring-2 ring-zinc-700 mt-4">

---
layout: image
---

# Candlestick Crash Course

Visualising the open, high, low, and closing prices (OHLC) of an asset within a period

<div class="flex justify-center pt-4">
  <img src="/candlestick.png" class="h-80">
</div>

---
layout: two-cols-header
---

# Candlestick Crash Course

Visualising the open, high, low, and closing prices (OHLC) of an asset within a period

::left::

Assume the following data rows:

```
┌────────┬───────┬──────────┐
| tstamp | price | quantity |
|--------|-------|----------|
| 1      | 2.23  | 4        |
| 3      | 2.14  | 2        |
| 4      | 2.71  | 7        |
| 6      | 2.69  | 1        |
| 9      | 2.42  | 2        |
| 10     | 2.92  | 5        |
| 12     | 2.38  | 1        |
| 15     | 2.72  | 8        |
| 17     | 2.13  | 6        |
| 19     | 2.61  | 3        |
└────────┴───────┴──────────┘
```

<span class="text-zinc-300 text-sm py-1 px-3 bg-zinc-800 border-l-4 border-blue-500">
  Each row is also known as a "tick"
</span>

::right::

<v-click>

Query using ClickHouse SQL:

```sql
SELECT
  ts,
  argMin(price, tstamp) AS open,
  max(price) AS high,
  min(price) AS low,
  argMax(price, tstamp) AS close,
  sum(price * quantity) AS volume
FROM prices
GROUP BY floor(tstamp / 10) * 10 AS ts
--       toStartOfInterval(tstamp, INTERVAL 1 hour)
ORDER BY ts;
```

</v-click>

<v-click>

```
Results
┌─ts─┬─open─┬─high─┬──low─┬─close─┬─volume─┐
│  0 │ 2.23 │ 2.71 │ 2.14 │  2.42 │  39.70 │
│ 10 │ 2.92 │ 2.92 │ 2.13 │  2.61 │  59.35 │
└────┴──────┴──────┴──────┴───────┴────────┘
```

</v-click>

---
layout: section
---
# Coinhall's Data Journey

---
---

<div class="fixed text-3xl text-zinc-500">August 2021</div>

<div class="h-full flex justify-center items-center">
  <img src="/bigquery.png" class="w-90" />
</div>

---
layout: statement
---
# Too "slow"

The most trivial of queries take on average ~2s to run

---
layout: statement
---
# Too expensive

$8.40 SGD per TB scanned, billed at a minimum of <span v-mark.red>10 MB per query</span>

<v-click>

$$
\begin{align}
\text{Cost of 1M queries }&= 10^6 \times 10\text{ MB} \nonumber \\
                          &= 10\text{ TB} \nonumber \\
                          &= 10 \times \$8.40 \nonumber \\
                          &= \$84 \nonumber \\
\text{2M queries per day }&= 2 \times 30 \times \$84 \nonumber \\
                          &= \$5040 \text{ per month} \nonumber \\
\end{align}
$$

</v-click>

---
layout: image
---

<img src="/tsdb.png" class="rounded ring-2 ring-zinc-700" />

---
---

<div class="fixed text-3xl text-zinc-500">October 2021</div>

<div class="h-full flex justify-center items-center">
  <img src="/questdb.svg" class="w-90" />
</div>

---
layout: statement
---
# Too limited

Lack of developer resources and features

<v-click>
<div class="h-full flex justify-center items-center">
  <img src="/quest-delete.png" class="w-100 rounded ring-2 ring-zinc-700" />
</div>
</v-click>

---
layout: statement
---

<div class="fixed top-10 text-center w-full left-0 text-zinc-300">A (Pretty Subjective) Comparison</div>

<div class="h-full flex justify-center items-center">
  <img src="/comparison.png" class="rounded w-full" />
</div>

<div class="fixed bottom-12 text-zinc-500 text-xs">Data as of June 2022</div>
<div class="fixed bottom-8 text-zinc-500 text-xs">* Inferred from online benchmarks and other resources</div>


---
layout: statement
---

A (Pretty Naive) Benchmark

<div class="h-full flex justify-center items-center">
  <img src="/benchmark.png" class="rounded w-full" />
</div>

<div class="text-left mt-2 text-zinc-500 text-xs">All tables contain 1M rows</div>

---
---

<div class="fixed text-3xl text-zinc-500">July 2022</div>

<div class="h-full flex justify-center items-center">
  <img src="/clickhouse.svg" class="w-90" />
</div>

---
layout: section
---
# Pitfalls & Optimisations

---
layout: statement
---
# A database is only as fast as its slowest query

---
---
# Optimisation: RTFM

https://clickhouse.com/docs/en/optimize/sparse-primary-indexes

<div class="flex justify-center py-8">
  <img src="/ch-docs.png" class="rounded ring-2 ring-zinc-700 -mt-8" >
</div>

---
layout: two-cols-header
---
# The `LIMIT BY` Clause

To select the first `n` rows for each distinct value (eg. latest prices of each asset)

::left::

```
┌──────────┬────────┬───────┬──────────┐
| asset_id | tstamp | price | quantity |
|----------|--------|-------|----------|
| a        | 1      | 2.23  | 4        |
| a        | 3      | 2.14  | 2        |
| a        | 4      | 2.71  | 7        |
| a        | 6      | 2.69  | 1        |
| a        | 9      | 2.42  | 2        |
| b        | 1      | 2.92  | 5        |
| b        | 2      | 2.38  | 1        |
| b        | 5      | 2.72  | 8        |
| b        | 7      | 2.13  | 6        |
| b        | 9      | 2.61  | 3        |
└──────────┴────────┴───────┴──────────┘
```

::right::

<v-click>

```sql
-- Table definition
CREATE TABLE prices (
  `asset_id` String,
  `tstamp`   UInt64,
  `price`    Float64,
  `quantity` UInt64,
)
ORDER BY (asset_id, tstamp);

-- Query
SELECT * FROM prices
ORDER BY tstamp DESC
LIMIT 1 BY asset_id;
```

```
Results
┌─asset_id─┬─tstamp─┬─price─┬─quantity─┐
│ a        │      9 │  2.42 │        2 │
│ b        │      9 │  2.61 │        3 │
└──────────┴────────┴───────┴──────────┘
```

</v-click>


---
layout: center
---
<div class="flex justify-center py-8">
  <img src="/limit-by.png" class="rounded ring-2 ring-zinc-700 w-75%" >
</div>

---
---
### Optimisation: `AggregatingMergeTree` + Materialized View

<v-click>

```sql
CREATE TABLE prices_latest (
  `asset_id` String,
  `tstamp`   AggregateFunction(max, UInt64),
  `price`    AggregateFunction(argMax, Float64, UInt64),
)
ENGINE = AggregatingMergeTree ORDER BY asset_id;
```

</v-click>

<v-click>

```sql
CREATE MATERIALIZED VIEW prices_latest_mv
TO prices_latest
AS SELECT
  asset_id,
  maxState(prices.tstamp) AS tstamp,
  argMaxState(prices.price, prices.tstamp) AS price
FROM prices
GROUP BY asset_id;
```

</v-click>

<v-click>

```sql
SELECT
  asset_id,
  maxMerge(tstamp),
  argMaxMerge(price)
FROM prices_latest
GROUP BY asset_id;
```

</v-click>

---
---
# Sort-Merge Joins

Join algorithm where tables are individually sorted, then merged

<div class="flex justify-center pt-2">
  <img src="/smj.png" class="h-90" >
</div>

---
layout: center
---

<div class="flex justify-center pt-2">
  <img src="/smj-issue.png" class="h-110 rounded ring-2 ring-zinc-700" >
</div>

---
layout: center
---

# Optimisation: just don't join?

---
layout: statement
---

# Thank You!
