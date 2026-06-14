# Big Data Week 6 — GitHub Archive: JSONiq → Columnar Store

## Group members

| Name | ID |
|------|----|
|  Daniel Palient  |  214631426  |
|  Elyassaf Okanin |  319028064  |

---

## How to run

```bash
pip install jsoniq pyarrow pandas matplotlib seaborn
```

| Component | Command |
|-----------|---------|
| שאילתות JSONiq (Part A) | `jupyter nbconvert --to notebook --execute 01_explore.ipynb --output 01_explore.ipynb` |
| שיטוח הנתונים + יצירת טבלת עמודות (Part B) | `jupyter nbconvert --to notebook --execute 02_to_parquet.ipynb --output 02_to_parquet.ipynb` |
| חישוב התוצאות (Part C) | `jupyter nbconvert --to notebook --execute 03_analysis.ipynb --output 03_analysis.ipynb` |

> **Data file:** `git-archive-huge.json` — 71,528 MB uncompressed JSONL.  
> Update `DATA_PATH` / `SRC` in the notebooks to point to the file on your machine.

---

## Required results per step

### A1 — Total event count
```
28,506,909
```

### A2 — Event type frequencies

```
PushEvent                      14,271,557
CreateEvent                     3,328,235
IssueCommentEvent               2,782,424
WatchEvent                      2,552,211
IssuesEvent                     1,463,914
PullRequestEvent                1,393,766
ForkEvent                         971,107
DeleteEvent                       548,218
PullRequestReviewCommentEvent     441,408
GollumEvent                       294,042
CommitCommentEvent                193,894
MemberEvent                       147,596
ReleaseEvent                       89,280
PublicEvent                        29,257
```

### A3 — First 5 PushEvents

```
actor_login        repo_name                          created_at
davidcarlsonberg   PubWlkr/PubWlkr                    2015-02-20T01:00:01Z
loomchild          loomchild/reload                   2015-02-20T01:00:01Z
lsqshr             lsqshr/nipype                      2015-02-20T01:00:01Z
PhancyCat          PhancyCat/HTMLClock                2015-02-20T01:00:01Z
saramartinez       saramartinez/tv-or-not-tv          2015-02-20T01:00:01Z
```

### A4 — PushEvents with non-empty commits array
```
PushEvents with non-empty commits :  14,173,075  (99.31%)
PushEvents with empty/missing     :      98,482   (0.69%)
Total PushEvents                  :  14,271,557
```

### A5 — Timestamp range per event type
```
CommitCommentEvent            2015-01-01T00:00:55Z  →  2015-02-28T23:58:56Z
CreateEvent                   2015-01-01T00:00:01Z  →  2015-02-28T23:59:59Z
DeleteEvent                   2015-01-01T00:00:30Z  →  2015-02-28T23:59:46Z
ForkEvent                     2015-01-01T00:00:16Z  →  2015-02-28T23:59:34Z
GollumEvent                   2015-01-01T00:01:10Z  →  2015-02-28T23:59:44Z
IssueCommentEvent             2015-01-01T00:00:06Z  →  2015-02-28T23:59:59Z
IssuesEvent                   2015-01-01T00:00:30Z  →  2015-02-28T23:59:59Z
MemberEvent                   2015-01-01T00:04:11Z  →  2015-02-28T23:58:56Z
PublicEvent                   2015-01-01T00:09:13Z  →  2015-02-28T23:57:28Z
PullRequestEvent              2015-01-01T00:00:11Z  →  2015-02-28T23:59:58Z
PullRequestReviewCommentEvent 2015-01-01T00:00:08Z  →  2015-02-28T23:59:26Z
PushEvent                     2015-01-01T00:00:00Z  →  2015-02-28T23:59:59Z
ReleaseEvent                  2015-01-01T00:02:19Z  →  2015-02-28T23:59:06Z
WatchEvent                    2015-01-01T00:00:18Z  →  2015-02-28T23:59:58Z
```

### B1 — Parquet schema

```
event_id:     string
event_type:   dictionary<values=string, indices=int8, ordered=0>
actor_login:  string
repo_name:    string
created_at:   timestamp[us, tz=UTC]
commit_count: int32
```

### B4 — Output validation
```
events.parquet :    829.7 MB
Source JSON    : 71,528.4 MB
Ratio          :    86.2× smaller
DataFrame mem  :  2,109.4 MB  (28,506,909 rows)
```

### C2 — Busiest repository

```
REPO_BUSIEST = KenanSulayman/heartbeat  (105,335 events)
```

### C3 — Peak activity for KenanSulayman/heartbeat

```
Peak hour (UTC): 01:00
```

---

## Timing tables

### Part A

| Query | Wall-clock time (s) |
|-------|---------------------|
| A1 — total event count | 127.49 |
| A2 — event type frequencies | 245.80 |
| A3 — first 5 PushEvents | 303.96 |
| A4 — non-empty commits count | 187.90 |
| A5 — timestamp range | 409.93 |

### Part B

| Step | Wall-clock time (s) | Throughput (rows/s) |
|------|---------------------|---------------------|
| B3 — full JSON → Parquet conversion | 406.8 | 70,074 |
| B4 — reload Parquet into DataFrame  |  12.9 | 2,208,199 |

### Part C

| Step | Wall-clock time (s) |
|------|---------------------|
| Parquet load + timestamp parse | 58.92 |
| C1 — top users by event count | 7.480 |
| C1 — top users by commit count | 7.906 |
| C2 — top repos by event count | 21.080 |
| C2 — top repos by commit count | 10.679 |
| C3 — hourly aggregation | 0.012 |
| C3 — day-of-week aggregation | 0.019 |
