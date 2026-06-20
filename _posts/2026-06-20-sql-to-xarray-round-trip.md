---
title: "Goal 3 is in: SQL queries round-trip back to xarray"
date: 2026-06-20 18:00:00 +0530
categories: [gsoc, holoviz, xarray-sql]
tags: [xarray-sql, datafusion, arrow, lazy-evaluation, gsoc-2026]
excerpt: "Six weeks, four review rounds, dozens of inline comments, two architectural rewrites. xarray-sql now reconstructs an xr.Dataset from a SQL query result, with coordinates and attrs intact."
header:
  overlay_image: /assets/images/2026-06-20-sql-to-xarray/hero.png
  overlay_filter: 0.45
  caption: "PR #167 closed issue #58. Six weeks, four review rounds, one library."
---

Goal 3 of the proposal was the inverse of xarray-sql's current
direction: take a SQL query result and reconstruct the xarray Dataset
on the other side, with coordinates, dimensions and attributes intact.
[Issue #58](https://github.com/alxmrs/xarray-sql/issues/58) had been
open since 2025. Today it closed.

<figure>
  <img src="{{ '/assets/images/2026-06-20-sql-to-xarray/issue58-closed.png' | relative_url }}"
       alt="GitHub issue #58 'Inverse problem: Read a table or SQL query into an Xarray Dataset' showing the purple Closed badge and link to PR #167.">
  <figcaption>Issue #58 closed. The title says "inverse problem" for a
  reason.</figcaption>
</figure>

This is the post about how that happened. What I built, what I got
wrong, what Alex Merose (the library author and reviewer) pushed back
on, and what the final API looks like.

## What it does, in 8 lines

```python
import xarray as xr
import xarray_sql as xql

ds = xr.tutorial.open_dataset("air_temperature").chunk(time=24)
ctx = xql.XarrayContext()
ctx.from_dataset("air", ds)

out = ctx.sql("SELECT * FROM air").to_dataset()   # lazy, no scan yet
region = out["air"].isel(time=0).values           # only this slice is read
```

`out` is a real `xr.Dataset` with the original dims, coord values,
dtype, and attrs. The data variables are backed by a custom
`BackendArray`; xarray indexers translate to DataFusion `filter`
expressions, so only the requested region is materialized through
Arrow `RecordBatches`. Aggregations follow the same surface but
materialize once eagerly, because their result is small by
construction.

`.compute()` reads the whole thing into memory if you want it that
way.

<figure>
  <img src="{{ '/assets/images/2026-06-20-sql-to-xarray/pr167-merged.png' | relative_url }}"
       alt="GitHub PR #167 header showing 'Add lazy SQL to xarray round-trip via XarrayDataFrame.to_dataset (closes #58)' with the purple Merged badge, merged 20 commits into alxmrs:main from ghostiee-11:feat/lazy-sql-to-xarray.">
  <figcaption>PR #167 merged on 20 Jun 2026 by Alex Merose. 20 commits
  in the end, after several rewrites.</figcaption>
</figure>

## What was hard

The shape of the problem looks easier than it was. xarray-sql's
forward pivot flattens an N-D Dataset to one row per cell, the SQL
engine executes a query over that table, and we get a flat result
back. Inverting it is, in toy form, `set_index(dims).to_xarray()`.
Four lines. So why did this take six weeks and four review rounds?

Because four lines is the part that works on toy data. Everything
else is the part that makes a library. By the end the PR had:

- **20 commits**, 13 of mine plus 7 from Alex helping land
  the PR
- **Dozens of inline review comments** across four rounds
- **Three full rewrites** of the lazy backend before it stabilized
- **Around 700 lines** of new code in `ds.py`, plus a new test
  module covering round-trip identity, sparsity, fill-value
  behavior, and vectorized-indexer fallback
- **One performance regression** I shipped and had to validate the
  fix for on real ERA5

Each review round sent me back to redesign, and the version that
merged was the better answer for it.

### The lazy-eager axis

My first draft was eager. You called `.to_dataset()`, it pulled the
whole result through pandas, indexed it, handed you back a Dataset.
For a tutorial dataset this is fine. For a year of ERA5 it is
several gigabytes of buffer for someone who maybe wanted one slice.

The round-one review made it clear that laziness had to be the
default, with xarray's `BackendArray` contract as the right hook.
The shape: return a Dataset whose variables look real to xarray, but
whose values are not loaded yet. Indexers translate to filters, the
filters push into the SQL plan, and only the matching rows ever
leave the engine.

I rebuilt the path around `BackendArray` plus
`xarray.core.indexing.LazilyIndexedArray`. The new internals are
`SQLBackendArray` (the lazy backend), `_scatter_batches_to_ndarray`
(Arrow positional scatter into a numpy buffer), `_lazy_to_xarray`
(the dispatch that builds the Dataset), and `_apply_template`
(metadata recovery for var attrs, encoding, non-dim coords, and dim
dtype). Slicing now goes through DataFusion's
`df.filter(expr).select(*cols).execute_stream()`, and the returned
Arrow `RecordBatches` scatter directly into numpy, no pandas hop.

<figure>
  <img src="{{ '/assets/images/2026-06-20-sql-to-xarray/architecture.png' | relative_url }}"
       alt="Architecture diagram: xr.Dataset cube on the left, an arrow to a 'DataFusion + Arrow' database in the middle (with a SQL snippet 'SELECT * FROM air WHERE lat > 30' above it), an arrow to a faded xr.Dataset (lazy) cube on the right, and a row of Arrow RecordBatch glyphs flowing along the bottom.">
  <figcaption>The reverse path: indexer to DataFusion filter, filtered
  rows stream back as Arrow batches, scatter into numpy.</figcaption>
</figure>

### Stop parsing SQL with regex

My second mistake was reading the FROM clause with a regex to find
the "default" template. The round-two review was unambiguous: don't
parse SQL strings when the DataFusion DataFrame interfaces already
expose what you need.

That landed. The `XarrayDataFrame` wrapper already held the live
DataFusion DataFrame; I rewrote everything to flow through that
reference instead of re-parsing query strings. Two regexes, a
`_sql_literal` helper, a `_build_cond` helper, and a fallback
materialize path all went into the bin. Net: `+177 / -361` lines on
that round.

### The performance moment

Round four had a clean diff, all tests green, and the PR ready to
merge. Then the README example on real ERA5 surfaced one more
problem:

> `agg_ds = result.to_dataset(dimension_columns=["level"]) # this
> takes a surprisingly long time`

The cause was in my lazy path. For aggregation queries it was
re-executing the whole GROUP BY once per dim just to discover the
distinct coord values. On ERA5 with a WHERE filter pulling chunks
from GCS, every redundant scan was a remote round trip.

Alex helped land the fix on the branch. I built a 115 MB synthetic
ERA5-shape reproducer and benchmarked the fix end to end:

| stage (cold)                      | pre-fix | post-fix |
| --------------------------------- | ------: | -------: |
| `ctx.sql(...)`                    | 374 ms  |   45 ms  |
| `result.to_dataset(dims=["level"])` | **684 ms** | **192 ms** |
| `agg_ds["avg_c"].values`          |  88 ms  |    0 ms  |
| total cold                        | 1146 ms | **237 ms** |

About 4.8x faster end to end on cold cache, and `.values` is free on
the post-fix because the single eager pass already populated it. On
real ERA5 the gap is bigger because each pre-fix re-scan was a remote
read.

<figure>
  <img src="{{ '/assets/images/2026-06-20-sql-to-xarray/perf-bars.png' | relative_url }}"
       alt="Horizontal bar chart titled 'Cold-run to_dataset() on a 115 MB ERA5-shape synthetic': ctx.sql 374ms pre-fix vs 45ms post-fix, result.to_dataset 684ms vs 192ms, agg_ds['avg_c'].values 88ms vs 0ms, total cold 1146ms vs 237ms, captioned '4.8x faster end to end'.">
  <figcaption>4.8x faster on cold cache. Scatter is free on the
  post-fix because the eager pass already populated the buffer.</figcaption>
</figure>

## What I changed my mind about

Three things.

**Public API surface should be smaller, not larger.** I had exported
`XarrayDataFrame` and `apply_template` as public. Alex asked the
cross-project mentor for a second opinion on whether either needed to
be top-level. The answer was no. Both went private, the only public
name from this work is the existing `XarrayContext.sql()` return
value. The smaller the surface, the less rework when the internals
move.

**"Lazy" and "fast" are not the same word.** I had been optimizing
for "no work until first access." The review pushed back: also
optimize for "no surprise when the user actually does access." For
aggregations those pull in opposite directions, and the user-facing
answer wins. The chunks knob that landed later
([`9c48b53`](https://github.com/alxmrs/xarray-sql/commit/9c48b53),
[`f535f3c`](https://github.com/alxmrs/xarray-sql/commit/f535f3c))
gives the caller explicit control rather than leaving it implicit.

**Cut tests too.** I shipped 31 tests; Alex pared them to 28. His
note in the review thread is worth quoting in full:

> Having too many tests puts a high maintenance burden on the project
> (tests require human verification) to the point that many of them
> might not be worth the regressions they are designed to catch.

I had been writing tests to prove the implementation was the way I
wrote it. The valuable tests prove the contract holds for the user,
not the path I happened to take.

<figure>
  <img src="{{ '/assets/images/2026-06-20-sql-to-xarray/all-checks-green.png' | relative_url }}"
       alt="GitHub PR view: 'alxmrs merged commit d5814b5 into alxmrs:main' with 12 checks passed list including python 3.10/3.11/3.12/3.13 tests, lint, Windows/macOS/Linux builds, and Rust Lint and Test.">
  <figcaption>Merge commit d5814b5. All 12 CI checks green.</figcaption>
</figure>

## Thanks, and a credit I did not expect

In the days before merge, Alex added me to the Sponsors & Contributors
section of the README. That is the kind of thing maintainers do not
have to do, and a small line item in a README does not capture how
much of a confidence boost it is when the person whose code you have
been pestering for six weeks decides you belong in the
acknowledgements.

<figure>
  <img src="{{ '/assets/images/2026-06-20-sql-to-xarray/sponsors-attribution.png' | relative_url }}"
       alt="Screenshot of the xarray-sql README's 'Sponsors & Contributors' section, with the final bullet reading 'Aman Kumar for spending a considerable amount of his GSoC internship contributing to this project.'">
  <figcaption>The bottom line of the Sponsors &amp; Contributors
  section. Alex added this himself, in the same week as merge.</figcaption>
</figure>

Six weeks of design, three lazy-backend rewrites, dozens of inline
comments, 13 of my own commits and Alex helping land the PR: the
implementation is mine, and the architectural framings (lazy
default, no SQL parsing, smaller public surface, chunks as the
partitioning knob) came out of Alex's review feedback. Every "no,
do this instead" was a better answer than what I had, and the right
way to thank a maintainer who does that is to be worth the time on
the next PR too.

Thanks also to [Andrew](https://github.com/ahuang11), for assisting
and helping along the way.

## Links

- PR: [alxmrs/xarray-sql#167](https://github.com/alxmrs/xarray-sql/pull/167)
- Merge commit: [`d5814b5`](https://github.com/alxmrs/xarray-sql/commit/d5814b5)
- Closed issue: [alxmrs/xarray-sql#58](https://github.com/alxmrs/xarray-sql/issues/58)

Next post in two weeks. Honest notes only.
