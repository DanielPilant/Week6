# Note on Task A5 — why the aggregation uses Spark instead of pure JSONiq

**Short version:** A5 was implemented the canonical JSONiq way first (FLWOR `group by`
with `min`/`max`, as in the lecture slides). On the full **~70 GB** dataset it crashes with
out-of-memory, because RumbleDB materializes each group in memory and our largest group has
**14.3 million rows**. After exhausting the options below, the per-type `min`/`max` aggregation
was delegated to Spark's `groupBy().agg(...)` (which is also covered in the course's Spark
slides). The JSONiq logic — reading the file and projecting the fields — is still done in JSONiq.

## Machine used
| | |
|---|---|
| CPU | Intel Core i7-13650HX, 20 logical cores |
| RAM | 15.7 GB total (only **~4.4 GB free** during the runs; the rest used by Windows/VS Code/etc.) |
| Disk | single C: drive, ~25 GB free |
| OS / Java | Windows 11, Java 17 |
| Engine | `jsoniq` package = RumbleDB 2.1.1 on PySpark 3.5.1, default Spark driver heap = **1 GB** |
| Data | `git-archive-huge.json`, ~70 GB uncompressed JSONL, **28,506,909** events; PushEvent alone = **14,271,557** rows |

## The canonical JSONiq query we wanted to use (per the slides)
```
for $e in json-file("…/git-archive-huge.json")
group by $t := $e.type
return { "type": $t, "earliest": min($e.created_at), "latest": max($e.created_at) }
```
This is correct and was **verified on a small sample**. It does not complete on the full file.

## Options we tried (all failed on the full file)
1. **Pure JSONiq with the `{| … |}` object merge** (to return one object directly).
   → `SparkException: Job cancelled because SparkContext was shut down` — the merge forces
   local materialization on the driver.
2. **Pure JSONiq returning a sequence** (`for … group by … return {type, min, max}`, the slide form).
   → Same `SparkContext was shut down`; the failure is in the `min`/`max` group-by itself, not the merge.
3. **Projection + larger heap:** projected each event to a tiny `{type, ts}` object *before* grouping
   (≈25× less data) **and** raised the driver heap to 5 GB via `PYSPARK_SUBMIT_ARGS`.
   → Clean `java.lang.OutOfMemoryError: Java heap space` in the aggregation task (stage 2, task 0).

## Why it fails (root cause)
RumbleDB evaluates `min`/`max` inside a `group by` by **collecting every row of a group into one
Spark task's heap** (groupByKey-style: no incremental folding, no spill within a key). The PushEvent
group is 14.3M rows; held in memory that is ~37 GB as full objects, and still several GB even when
projected to `{type, ts}` — more than the 1 GB default heap and more than a 5 GB heap. We cannot raise
the heap further because only ~4.4 GB of RAM is physically free; an `-Xmx` larger than available RAM
just gets the JVM killed (which is the "SparkContext shut down" we saw).

Note that **A2's `count` group-by worked fine** on the same data: `count` is computed as a foldable
aggregate (a running integer per group, never holding the rows), so it streams. `min`/`max` are equally
foldable in principle, but RumbleDB's group-by does not fold them — it materializes the group first.

## What we did instead (and why it's legitimate)
JSONiq does the part it does well — read and project:
```python
query = 'for $e in json-file("…") return { "type": $e.type, "ts": $e.created_at }'
df = rumble.jsoniq(query).df()                      # hand the projection to Spark
rows = df.groupBy("type").agg(F.min("ts").alias("earliest"),
                              F.max("ts").alias("latest")).collect()
```
Spark's `groupBy().agg(min, max)` uses hash aggregation with map-side partial aggregation — it keeps only
a running min/max per key and can spill, so it never holds a whole group in memory. It completes on the
full 70 GB file in **~410 s** within the default 1 GB heap. Spark DataFrame aggregation is part of the
course material (Week 7 Spark slides), so the work stays within the taught toolkit; only the physical
aggregation engine changed, not the logic or the result.

All other Part-A tasks (A1–A4) are pure JSONiq exactly as in the slides.
