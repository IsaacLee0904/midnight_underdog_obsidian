---
title: "[DE] Building an End-to-End Data Platform (3): The Marts Layer, and a Data Contract Grown Bottom-Up"
source: "https://joshua-data.medium.com/building-an-end-to-end-data-platform-3-the-marts-layer-and-a-data-contract-grown-bottom-up-9d580a0a3bf4"
author:
  - "[[Joshua Kim]]"
published: 2026-07-03
created: 2026-07-04
description: "More"
tags:
  - "clippings"
---
![](https://miro.medium.com/v2/resize:fit:1400/format:webp/0*06UoAUFw7h7wYjyV.png)

In [Part (1)](https://joshua-data.medium.com/building-an-end-to-end-data-platform-1-the-lakehouse-before-the-first-dbt-model-5034b1446246), I built everything that sits under dbt: hourly ingest, GCS as the raw layer, the BigQuery external table, the `dw` / `dw_dev` split, and Workload Identity Federation instead of service account keys. In [Part (2)](https://joshua-data.medium.com/building-an-end-to-end-data-platform-2-the-first-dbt-models-on-a-polymorphic-event-stream-f736d387a2e2), I put on the analytics engineer hat and wrote the first models: one staging fact over a polymorphic event stream, 20+ core models (per-event-type transactional facts, periodic snapshots, and latest-state dimensions), and the two macros (`batch_filter` and `get_where_subquery`) that gate every model and every test to the batch window.

Part (2) stopped at core layer, on purpose. I wrote, out loud, that I would not build a marts layer until I had a real product question to build it around, because marts encode product questions and guessing at them is how an analytics engineer ends up maintaining a `mart_user_activity_v2_FINAL_clean` table that nobody queries.

This post is what happened once I had questions worth answering. It adds a `03-marts` layer with two fact archetypes, two more macros, and the part I care most about: a data contract that I grew from the bottom up instead of writing top-down.

Everything here is a modeling decision. I will show the actual SQL, the trade-offs I weighted, and the two or three places where I deliberately picked the more complex option and can tell you exactly why.

Repo:

## [GitHub - joshua-data/project-gharchive-elt: ELT pipeline - GitHub Archive Data](https://github.com/joshua-data/project-gharchive-elt?source=post_page-----9d580a0a3bf4---------------------------------------)

### ELT pipeline - GitHub Archive Data. Contribute to joshua-data/project-gharchive-elt development by creating an account…

github.com

Live dbt Docs (updated every merge to `main`):

## [dbt Docs](https://joshua-data.github.io/project-gharchive-elt/dbt-docs/?source=post_page-----9d580a0a3bf4---------------------------------------#!/overview)

### documentation for dbt

joshua-data.github.io

## Table of contents

```c
1. Where Part (2) left off
2. The marts layer, and why it exists now
  - core is raw material, marts are product questions
  - the two archetypes I built first
3. Fan out, then fan back in: the unify_events macro
  - FULL OUTER UNION ALL BY NAME
  - why a macro, not nine hand-written unions
  - the predicate pushdown I checked before trusting it
4. Periodic-snapshot facts: what kind of activity, not just whether
  - the 3x3 grain-by-period matrix
  - why weekly and monthly re-read core instead of rolling up daily
  - the counters, and the PR-close split by merge outcome
5. Accumulating-snapshot facts: one row per PR, milestones that fill in over time
  - the PR lifecycle as a set of milestone timestamps
  - merge, not insert_overwrite, and why the partition key is immutable
  - reverse_batch_filter: the complement of batch_filter
  - reconciling latest against existing
  - the honest gate: PRs whose opened event was never captured
  - mart_snp_fact__pr_reviews, the simpler sibling
6. The data contract, grown bottom-up
  - Part (2): tests lived only on staging
  - what a mart actually demanded
  - the contract, column by column
  - why upstream, and why not_null_proportion at 0.9999
  - placeholder tokens keep tests scoped to the batch
7. The lineage graph gets four more colors
8. What's still on my plate
9. Coming up in Part (4)
```

## 1\. Where Part (2) left off

The project graph at the end of Part (2) was three layers deep and stopped there:

```c
raw__gharchive.ext__events
        │
        ▼
01-stg / stg_fact__events        ← flatten the envelope, keep payload as JSON, dedupe
        │
        ▼
02-core /
   ├── transactional-fact/       ← 16 tables, one per event type, payload extracted
   ├── periodic-snapshot-fact/   ← 9 tables, daily / weekly / monthly active users, repos, orgs
   └── scd1/                     ← 3 latest-state dimensions (users, repos, orgs)
```

That is 1 staging fact and 28 core models. Every core transactional fact is a thin, typed, documented projection of one event type out of the staging table. Every core snapshot answers a whether question (“was this user active on this day?”). None of them answers a shaped business question, and none of them joins two event types together.

This post adds the layer that does both. Here is the graph as it stands today, with the two new mart families at the bottom:

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*htM_UVIuB6wDQOPHC7yEhw.jpeg)

The colors are not decorative. They are the `node_color` values I set per folder in `dbt_project.yml`, so this diagram and the published [dbt-docs lineage graph](https://joshua-data.github.io/project-gharchive-elt/dbt-docs/#!/overview?g_v=1) tell the same story. More on that in section 7.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*hCLfhchoLQOb-EbeQc4LRQ.png)

## 2\. The marts layer, and why it exists now

### core is raw material, marts are product questions

The mental split I settled on is simple to state and load-bearing in practice. Core is raw material. A `core_fact__pull_request_events` row is a faithful, typed record of one thing that happened. It has no opinion about what you want to know. Marts are where opinions live. A mart takes a question (“how active was this user last week, broken down by what they actually did”) and shapes core into the exact grain that answers it.

I did not want to build marts before I could name the questions, so I named two:

- 1\. What is the full lifecycle of a single pull request, from the moment it was opened to the moment it was last touched?
- 2\. How much of each kind of development activity did a given user, repo, or org produce over a day, a week, or a month?

Those two questions do not share a grain, and that turned out to be the whole point.

### the two archetypes I built first

The two questions map cleanly onto two classic dimensional modeling fact types, and I built one folder for each:

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*gtR8eohBihGAIx2lBWSCRA.png)

The rest of this post is a walk through both columns of that table. I will start with the periodic-snapshot side, because it introduces the macro that the whole marts layer leans on.

## 3\. Fan out, then fan back in: the \`unify\_events\` macro

Part (2) was one long argument for fanning the polymorphic stream out. One wide staging table with `payload` as JSON, then 16 thin per-event-type facts, each one typed and documented, each one able to absorb a GitHub schema change without touching its neighbors. That was the right call and I still believe it.

The marts layer needs the opposite move. To answer “how much of each activity did this user produce,” I have to look across every event type at once. A push, a review, an issue comment, and a fork all count toward the same user’s daily activity row. So the periodic-snapshot marts fan the stream back in.

The naive way to do that is a hand-written `UNION ALL` of 15 `SELECT` blocks, repeated in all 9 periodic-snapshot models. That is roughly 135 near-identical `SELECT` blocks to keep in sync. I was not going to do that. Instead the union is defined once, in a macro [unify\_events.sql](https://github.com/joshua-data/project-gharchive-elt/blob/main/dbt/macros/query_utils/unify_events.sql):

```c
{% macro unify_events(shared_columns, branches) %}

    {% for branch in branches %}
        select
            {% for col in shared_columns %} {{ col }}, {% endfor %}
            '{{ branch.event_name }}' as event_name,
            {% for col in branch.get('extra_columns', []) %} {{ col }}, {% endfor %}
        from
            {{ ref(branch.table) }}
        {% if not loop.last %} full outer union all by name {% endif %}
    {% endfor %}

{% endmacro %}
```

Each caller passes the columns every branch shares, plus a per-branch list of extras. The daily user mart calls it like this [mart\_snp\_fact\_\_daily\_user\_dev\_activities.sql](https://github.com/joshua-data/project-gharchive-elt/blob/main/dbt/models/03-marts/periodic-snapshot-fact/mart_snp_fact__daily_user_dev_activities.sql):

```c
with

fact__events as (
    {{
        unify_events(
            shared_columns=['user_id', 'repo_id', 'org_id', 'created_date'],
            branches=[
                {'table': 'core_fact__commit_comment_events',              'event_name': 'commit_comment_event',               'extra_columns': ['action']                },
                {'table': 'core_fact__create_events',                      'event_name': 'create_event',                       'extra_columns': ['ref_type']              },
                {'table': 'core_fact__delete_events',                      'event_name': 'delete_event',                       'extra_columns': ['ref_type']              },
                {'table': 'core_fact__discussion_events',                  'event_name': 'discussion_event',                   'extra_columns': ['action']                },
                {'table': 'core_fact__fork_events',                        'event_name': 'fork_event',                         'extra_columns': ['action']                },
                {'table': 'core_fact__gollum_events',                      'event_name': 'gollum_event'                                                                   },
                {'table': 'core_fact__issue_comment_events',               'event_name': 'issue_comment_event',                'extra_columns': ['action']                },
                {'table': 'core_fact__issues_events',                      'event_name': 'issues_event',                       'extra_columns': ['action']                },
                {'table': 'core_fact__member_events',                      'event_name': 'member_event',                       'extra_columns': ['action']                },
                {'table': 'core_fact__pull_request_events',                'event_name': 'pull_request_event',                 'extra_columns': ['action']                },
                {'table': 'core_fact__pull_request_review_comment_events', 'event_name': 'pull_request_review_comment_event'                                              },
                {'table': 'core_fact__pull_request_review_events',         'event_name': 'pull_request_review_event',          'extra_columns': ['review_state as action']},
                {'table': 'core_fact__push_events',                        'event_name': 'push_event'                                                                     },
                {'table': 'core_fact__release_events',                     'event_name': 'release_event',                      'extra_columns': ['action']                },
                {'table': 'core_fact__watch_events',                       'event_name': 'watch_event',                        'extra_columns': ['action']                },
            ]
        )
    }}
)

...
```

Three details in there earn their place.

- `**FULL OUTER UNION ALL BY NAME**`**, not a positional union.** BigQuery’s `by name` variant aligns columns by name and pads any missing column with `NULL`. That is what lets each branch project only the columns it actually has. A positional `UNION ALL` would force every branch to select every column in the same order, which means writing `null as col1` in every branch that lacks one. The `by name` operator makes the sparseness the union’s problem instead of mine, and a counter like `countif(event_name = ‘pull_request_review_event’ and action = ‘approved’)` stays safe even on the branches that never expose `action`.
- **Column aliasing normalizes heterogeneous fields into one name.** The review fact does not have an `action` column; its discriminator is `review_state`. Passing `review_state as action` as an extra column lets a review’s state land in the same `action` slot that PRs and issues use, so downstream `countif` s can treat them uniformly. That is a small thing that removes a whole class of “which column do I check for this event type” bugs.
- **PublicEvent is deliberately not in the list.** There are 16 core transactional facts but `unify_events` stitches only 15. `core_fact__public_events` (a private repo being made public) is not developer activity in the sense these marts measure, so it is left out. One event type excluded, on purpose, documented in the mart’s YAML.

> **Decision**: the fan-out in Part (2) buys typing and documentation. The fan-in here buys cross-type aggregation. The macro is the seam between the two, and it lives in exactly one file so the union has exactly one definition.

### the predicate pushdown I checked before trusting it

There is one thing about a macro like this that would quietly wreck the project if it were wrong, and I did not want to assume it. Every source table(`core_fact__*_events`) is partitioned on `created_date` with `require_partition_filter: true`. The consuming part puts its `batch_filter(date_col='created_date')` in the outer query, on top of the union. The question is whether that outer predicate pushes down through the `FULL OUTER UNION ALL BY NAME` into each branch, or whether BigQuery materializes the full union of tables first and filters afterward.

If it did not push down, the mart would either fail the partition filter requirement or scan everything, and the whole cost model of the project would fall apart. I checked it against the actual partitioned tables before I trusted it, confirmed the predicate reaches each branch, and then wrote that finding into the macro’s own documentation so the next person (me, in six months) does not have to rediscover it:

```c
Outer predicates on shared_columns (e.g. a batch_filter(created_date)
placed in the consuming model's WHERE) push down through the union to
each source table. Verified against the partitioned, partition-filter-
required core_fact__*_events source tables.
```

A macro that silently defeats partition pruning is worse than no macro. Writing down that I verified it, and against what, is the difference between a clever trick and something I would put in production.

## 4\. Periodic-snapshot facts: what kind of activity, not just whether

### the 3x3 grain-by-period matrix

The periodic-snapshot family is 9 tables: 3 grains times 3 periods.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*Dt_qmJTylB_V8YvO6g5VWQ.png)

Each row is one `(entity, period)` where the entity did at least one thing in that period. These are the natural counterpart to the core `…_active_users` snapshots from Part (2), but where those answered whether an entity was active, these answer what the entity actually did. The core snapshot carries `events_count` and `active_days`. These carry 25+ typed counters, one family per event type: `pushes_count`, `prs_opened_count`, `pr_reviews_count`, `issue_comments_count`, `forks_count`, and so on.

The daily user grain is the representative shape ([mart\_snp\_fact\_\_daily\_user\_dev\_activities.sql](https://github.com/joshua-data/project-gharchive-elt/blob/main/dbt/models/03-marts/periodic-snapshot-fact/mart_snp_fact__daily_user_dev_activities.sql)):

```c
select
    -- grain
    user_id,
    created_date,
    -- volume
    count(1) as all_events_count,
    count(distinct org_id) as unique_orgs_count,
    count(distinct repo_id) as unique_repos_count,
    -- pushes
    countif(event_name = 'push_event') as pushes_count,
    -- pull requests
    countif(event_name = 'pull_request_event' and action = 'opened') as prs_opened_count,
    countif(event_name = 'pull_request_event' and action = 'merged') as prs_merged_closed_count,
    countif(event_name = 'pull_request_event' and action = 'closed') as prs_unmerged_closed_count,
    countif(event_name = 'pull_request_event' and action in ('merged', 'closed')) as prs_total_closed_count,
    -- reviews, comments, issues, refs, releases, ... (25+ counters total)
from
    fact__events
where true
    and {{ batch_filter(date_col='created_date') }}
group by
    all
```

### why weekly and monthly re-read core instead of rolling up daily

This is the decision I want to be precise about, because it looks like an inefficiency until you notice why it is not.

In Part (2), the core weekly snapshot rolled up the core daily snapshot: weekly read from daily and summed. That works there because every core-snapshot measure is additive. You can sum `events_count` and `active_days` across days and get the correct week.

The mart snapshots cannot do that, because they carry non-additive measures. `unique_repos_count` is a `count(distinct repo_id)`. You cannot sum 7 daily distinct-counts and get the weekly distinct-count, because a repo the user touched on both Monday and Tuesday would be counted twice. Distinct is not additive, full stop.

So each grain-period table re-reads core through `unify_events` and re-aggregates from the event rows, using the `interval` argument on `batch_filter` to snap the window to the period ([mart\_snp\_fact\_\_weekly\_user\_dev\_activities.sql](https://github.com/joshua-data/project-gharchive-elt/blob/main/dbt/models/03-marts/periodic-snapshot-fact/mart_snp_fact__weekly_user_dev_activities.sql)):

```c
{% set interval = 'week(sunday)' %}

select
    user_id,
    date_trunc(created_date, {{ interval }}) as created_date,
    count(distinct repo_id) as unique_repos_count,
    -- ... same 25+ counters ...
from
    fact__events
where true
    and {{ batch_filter(date_col='created_date', interval=interval) }}
group by
    all
```
![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*OksgrNN3ffvp86tD9Ozj7w.jpeg)

The `interval` argument is the same period-to-date mechanism from Part (2): `batch_filter` snaps the lower bound of the window back to the start of the enclosing period with `date_trunc`, and leaves the upper bound alone. On a Wednesday run for `batch_date = Tuesday`, the weekly model sees Sunday through Tuesday and rewrites the in-progress Sunday-week partition. The next day rewrites it again with one more day. By the time the week closes, the partition is full and stable.

> **Decision**: I traded a slightly cheaper roll-up for correctness. Re-reading core three times per grain costs more compute than summing the daily table would, but it is the only way to keep distinct counts honest. When correctness and cost point in different directions on a metric this central, I pick correctness and pay the compute.

### the counters, and the PR-close split by merge outcome

The counter I am most deliberate about is PR closes. GitHub’s raw webhook fires a single `closed` action and hangs a `pull_request.merged = true` flag off the payload when the close was a merge. GH Archive normalizes that into a distinct `merged` action in the public feed, so in this project `merged` and `closed` are disjoint: a merged PR shows up as `merged`, an abandoned one as `closed`.

That is a real subtlety, and I did not want every analyst who touches this data to rediscover it. So the mart pre-splits closes into three columns:

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*s2YZoYzHEG0-XSnqx8oS1A.png)

Surfacing all three, with the total pre-computed, means nobody has to remember that `closed` excludes merges in this dataset. The domain knowledge lives in the model and in the [column docs](https://github.com/joshua-data/project-gharchive-elt/blob/main/dbt/docs/columns_dev_activity.md), once, instead of in every query.

The same care shows up in how the same counter means different things at different grains. From the user grain, `forks_count` and `watches_count` mean “this user forked N repos” and “this user starred N repos.” From the repo and org grains, the identical columns mean “this repo was forked N times” and “this repo was starred N times.” The SQL is nearly identical; the interpretation flips with the grain, and the docs say so explicitly so the meaning is never ambiguous.

## 5\. Accumulating-snapshot facts: one row per PR, milestones that fill in over time

### the PR lifecycle as a set of milestone timestamps

A pull request is not an event. It is a thing with a lifecycle. It gets opened, maybe reviewed, maybe labeled and reopened a few times, and eventually merged or closed. The transactional fact `core_fact__pull_request_events` records each of those touches as its own row. But nobody wants to answer “how long did this PR take to merge” by scanning a pile of event rows and reconstructing the timeline every time.

That reconstruction is exactly what an accumulating-snapshot fact is for. One row per pull request, with a column for each milestone, and the row fills in over time as the PR moves through its life:

![](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*wFI3FVxqGkTzPJREpJjJog.jpeg)

`mart_snp_fact__pull_requests` turns that lifecycle into columns:

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*r8ibwv3NSQjDJQTWWukcEQ.png)

The `first_` and `last_` pairs are there because a PR can be reopened and re-merged. For the overwhelming majority of PRs the two are equal, but modeling them as a pair means the rare reopened-then-remerged PR is represented honestly instead of being flattened into a single wrong timestamp.

### merge, not insert\_overwrite, and why the partition key is immutable

Every other incremental model in this project uses `insert_overwrite`, because their unit of work is a partition: a day’s worth or rows, recomputed and swapped in. An accumulating snapshot breaks that assumption. Its unit of work is the entity, not the partition. A PR opened in January and merged in July is one row whose lifespan crosses 6 monthly partitions, and I need to update that one row in place when the July merge lands, not rewrite a partition.

So this is the one family in the project that materializes with `merge` ([mart\_snp\_fact\_\_pull\_requests.yml](https://github.com/joshua-data/project-gharchive-elt/blob/main/dbt/models/03-marts/accumulating-snapshot-fact/mart_snp_fact__pull_requests.yml)):

```c
materialized: incremental
incremental_strategy: merge
unique_key: pull_request_id
partition_by:
  field: opened_date
  data_type: date
  granularity: day
require_partition_filter: true
cluster_by:
  - base_repo_id
  - author_user_id
```

The partition key is `opened_date`, and the choice of that column specifically matters. It is immutable: a PR’s opened date never changes no matter how many later events it collects. If I had partitioned on something mutable like `last_action_at`, a row would have to migrate partitions every time the PR was touched, which `merge` cannot do cleanly. Partitioning on an immutable date means a PR’s row is born into one partition and stays there for life, and `merge` only over updates its contents.

### \`reverse\_batch\_filter\`: the complement of \`batch\_filter\`

Here is the problem that `merge` + `require_partition_filter` creates, and the macro I wrote to solve it.

To fold this batch’s new events into the accumulated history, I need two things: the events from the current window, and the existing rows those events belong to. The current window is easy, that is what `batch_filter` has always done. The existing rows are the hard part, because the target table has `require_partition_filter: true`, so I cannot just `select * from {{ this }}`. BigQuery will refuse a query with no partition predicate.

I also do not want to read the whole target table even if I could. The only existing rows I need are the ones for PRs that were opened before the current window. A PR opened inside the window is fully recomputed from this batch’s events anyway, so its old row (if any, on a re-run) is redundant.

That is a clean, and importantly a complementary, partition predicate: give me everything opened strictly before the window starts. So I wrote `reverse_batch_filter`, which lives right next to `batch_filter` in the same file ([batch\_filter.sql](https://github.com/joshua-data/project-gharchive-elt/blob/main/dbt/macros/batch_filter.sql)):

```c
{% macro reverse_batch_filter(date_col='dt', start_date=none, interval='day') %}
    {# ... resolve sdt from args, batch_start_date, or batch_date-1 ... #}
    {{ date_col }} < date_trunc(date '{{ sdt }}', {{ interval }})
{% endmacro %}
```

Where `batch_filter` emits `date_col between date_trunc(start) and end`, `reverse_batch_filter` emits `date_col < date_trunc(start)`. The two are complements. `batch_filter` selects the window from the source; `reverse_batch_filter` selects everything before the window from the target. Together they tile the entire timeline with no overlap and no gap:

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*TZHvbrpa0g755nSuFZtoLQ.png)

And it satisfies `require_partition_filter` for free, because `opened_date` is the partition key and `< date_trunc(start)` is a valid predicate on it that also prunes to exactly the partitions that could hold a PR needing reconciliation.

### reconciling latest against existing

With those two sets in hand, the model reads like the tiling diagram it implements ([mart\_snp\_fact\_\_pull\_requests.sql](https://github.com/joshua-data/project-gharchive-elt/blob/main/dbt/models/03-marts/accumulating-snapshot-fact/mart_snp_fact__pull_requests.sql)):

```c
-- Per-PR view computed from THIS batch's events
latest as (
    select ... from dim__pr_milestones
    left join dim__prs        using (pull_request_id)   -- author from first 'opened'
    left join scd1__prs       using (pull_request_id)   -- latest header dims
    left join scd1__pr_labels using (pull_request_id)   -- latest label snapshot
),

-- Per-PR view from the EXISTING target, everything opened before the window
existing as (
    select *
    {% if is_incremental() %}
    from {{ this }}
    where true and {{ reverse_batch_filter(date_col='opened_date') }}
    {% else %}
    from latest where false
    {% endif %}
)

select
    latest.pull_request_id,
    coalesce(latest.opened_at, existing.opened_at) as opened_at,
    coalesce(least(latest.first_merged_at,  existing.first_merged_at),
             latest.first_merged_at, existing.first_merged_at) as first_merged_at,
    coalesce(greatest(latest.last_merged_at, existing.last_merged_at),
             latest.last_merged_at, existing.last_merged_at) as last_merged_at,
    -- ... same least/greatest pattern for reopened / closed / last_action ...
from latest
left join existing using (pull_request_id)
where true
    and coalesce(latest.opened_date, existing.opened_date) is not null
```

Three things make the reconciliation correct across batches and safe to re-run:

- `**least**` **and** `**greatest**` **for the extremes,** `**coalesce**` **for the dimensions.** A milestone like `first_merged_at` should be the earliest merge ever seen, whether that was in this batch or a prior one, so it is `least(latest, existing)`. `last_merged_at` is `greatest`. Header dimensions like `base_repo_name` just take whichever value is present, so they are `coalesce`.
- **The** `**least**` **/** `**greatest**` **calls are themselves wrapped in** `**coalesce**`**.** BigQuery’s `least` and `greatest` return `NULL` if any argument is `NULL`, and one side is almost always `NULL` (a PR is usually touched in one batch, not both). So `least(a, b)` alone would erase a perfectly good value the moment the other side was missing. Wrapping it as `coalesce(least(a, b), a, b)` means “take the smaller when both exist, otherwise take whichever one does.” I left that reasoning as a comment in the model, because it is the kind of line that looks redundant until you remember how `least` treats `NULL`.
- **The whole thing is idempotent by construction.** A PR opened in the window is fully recomputed by `latest` and excluded from `existing`, so re-running the batch reproduces it exactly. A PR opened before the window and touched inside it is reconciled every run to the same converged values, because `least` and `greatest` are order-independent. And a PR opened before the window and not touched inside it never appears in the query at all, so `merge` leaves its existing row untouched. Batch order does not matter and re-runs are free.
![](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*cXFnd8Q12_6lysHI5m0SeA.jpeg)

### the honest gate: PRs whose opened event was never captured

The source does not reach back to the beginning of time. When I started ingesting, PRs that had been opened years earlier were already mid-lifecycle, so the feed shows me their merges and closes but never their `opened` event. Those PRs would produce a row with a `NULL` `opened_date`, which is both wrong (I do not actually know when they opened) and dangerous (the partition key would be null).

Rather than carry incomplete rows, I gate them out with the last line of the model:

```c
where true
    and coalesce(latest.opened_date, existing.opened_date) is not null
```

If I have never seen a PR’s `opened` event, it does not enter the mart. This is a deliberate choice to prefer a smaller, honest table over a larger one padded with rows I cannot stand behind. It is documented on the `author_user_id` column too, which is derived from the first `opened` event and is therefore effectively not-null inside the mart precisely because of this gate.

### \`mart\_snp\_fact\_\_pr\_reviews\`, the simpler sibling

Not every accumulating snapshot needs the full reconciliation dance. A PR review has a much flatter lifecycle: it is submitted, and occasionally edited or dismissed, but there are no `first_` / `last_` extremes to track across batches. So `mart_snp_fact__pr_reviews` is the same archetype at a lower complexity ([mart\_snp\_fact\_\_pr\_reviews.sql](https://github.com/joshua-data/project-gharchive-elt/blob/main/dbt/models/03-marts/accumulating-snapshot-fact/mart_snp_fact__pr_reviews.sql)):

```c
select
    review_id,
    pull_request_id,
    review_state,
    review_submitted_at,
    date(review_submitted_at) as review_submitted_date,
    -- ...
from
    {{ ref('core_fact__pull_request_review_events') }}
where
    {{ batch_filter(date_col='created_date') }}
qualify
    row_number() over (partition by review_id order by event_id desc) = 1
```

It still materializes `merge` on `review_id` and partitions on the immutable `review_submitted_date`, so a later edit to a review upserts in place. But within a batch it just keeps the latest event per `review_id` with a `qualify`, and lets `merge` handle the across-batch upsert. No `reverse_batch_filter`, no `least` / `greatest`, because there is nothing to accumulate beyond “the most recent version of this review wins.” Same archetype, right amount of machinery. I did not reach for the heavier pattern where the lighter one is correct.

## 6\. The data contract, grown bottom-up

### Part (2): tests lived only on staging

At the end of Part (2) I was honest about a gap. Every test in the project (`not_null`, `accepted_values`, `dbt_utils.expression_is_true`, `dbt_utils.not_null_proportion`) hung off `stg_fact__events.yml` and nothing else. The reasoning was that core models are pure projections of staging, so if staging holds its contract, core mostly inherits it. I also said, in the same breath, that this was only an approximation of a data contract, and that the real version would come when I sat down and wrote, per model, what the grain is, what the key is, what is not-nullable, and what is referentially intact.

Building the marts is what forced me to actually write it. And it taught me that the right order to write a data contract is not the order I expected.

### what a mart actually demanded

The instinct with data contracts is to go top-down: sit with a blank page, enumerate every column of every model, and assert its properties in the abstract. I tried a little of that and it felt like busywork. I was writing assertions for columns nobody had a concrete reason to depend on yet, and I had no way to tell a load-bearing guarantee from a decorative one.

So I flipped it. I let the marts tell me what they needed, and I only wrote the contract where a real downstream consumer would break without it. Concretely, building the two PR marts surfaced a specific list of demands:

- `mart_snp_fact__pull_requests` does `merge` on `pull_request_id`. A null or non-unique key silently corrupts the upsert. So `pull_request_id` must be not-null upstream.
- Its milestone logic keys entirely off `action` (`min(if(action = ‘opened’, …))` and friends). If GitHub introduced a new action value I did not know about, it would vanish from every `min` / `max` and no error would fire. So `action` must be not-null and must be one of a known set.
- It partitions on `review_submitted_date`, derived from `review_submitted_at`. A null there is a null partition key. So `review_submitted_at` must be not-null.
- The dev-activity marts count `review_state = 'approved'`. A drifted state value would undercount silently. So `review_state` must be constrained to its known set.

Each of those is a guarantee a mart genuinely relies on. So each of those, and only those, is what I encoded, at the place upstream where the column is first given its meaning.

### the contract, column by column

Here is the contract I added, and for each assertion, the downstream thing that would break if it were false. This is the whole point of doing it bottom-up: every row in this table exists because something concrete needed it, not because a checklist said to write it.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*Bvv47M1UQiNw2UdfpSPdgQ.png)

The `accepted_values` sets are the ones I am happiest about, because they turn a silent failure into a loud one. The `action` set is `opened, closed, merged, reopened, labeled, unlabeled, assigned, unassigned`. The `review_state` set is `approved, changes_requested, commented, dismissed`. The day GitHub adds a new value, or GH Archive changes how it normalizes one, the build fails on the exact model where the assumption lives, instead of quietly producing a mart with a counter that is subtly too low.

![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*12Ebi1hwafH2dEoceDZPVw.png)

GitHub Actions log

### why upstream, and why \`not\_null\_proportion\` at 0.9999

Two smaller decisions inside the contract are worth pulling out.

- **Push the assertion as far upstream as it is true.** `pull_request_id` is consumed by three different marts. I could have tested it on each of them. Instead I test it once, on `core_fact__pull_request_events`, where the column is first extracted from JSON and first given its meaning. If the guarantee holds at the point of definition, every downstream consumer inherits it and I never write the same assertion twice. The enforcement point is singular, the fix location is singular, and there is no drift between three copies of “is this column null.”
![](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*rmc0N1BbijhhfUxrLsFwlw.jpeg)

- `**not_null_proportion >= 0.9999**`**, not** `**not_null**`**, where reality is messy.** A PR’s head repository can legitimately be null: if the fork the PR came from has since been deleted, GitHub emits a null `head.repo`. That is not a data quality bug, it is a real state of the world. A strict `not_null` there would fail the build on legitimate data, which is the fastest way to teach yourself to ignore a failing test. So the contract says “at least 99.99% not-null,” which catches a real regression (a sudden wave of nulls) while tolerating the handful of deleted-fork rows that are supposed to be null. The threshold encodes the reality of the source, not an idealized version of it.

### placeholder tokens keep tests scoped to the batch

One mechanical note, because it ties back to Part (2). Every test above carries a `where` clause written with placeholder tokens:

```c
- name: pull_request_id
  data_tests:
    - not_null:
        config:
          where: "created_date between __batch_start_date__ and __batch_end_date__"
```

`__batch_start_date__` and `__batch_end_date__` are not real SQL. They are rewritten at compile time by the `get_where_subquery` override I built in Part (2), which swaps them for the resolved batch dates. Without that scoping, every `not_null` test would scan the full multi-year history on every run, which is prohibitive on a side-project budget. With it, the contract is checked only over the slice the build actually produced. The contract and the batch window stay in lockstep, and adding a new assertion is mostly copy-pasting the same `where` line that every other test already uses.

## 7\. The lineage graph gets four more colors

The four-color palette from Part (2) had a slot per core folder. Adding the marts meant adding two more colors, and the `node_color` config in ([dbt\_project.yml](https://github.com/joshua-data/project-gharchive-elt/blob/main/dbt/dbt_project.yml)) now covers all 6 layers:

```c
models:
  dw:
    +materialized: view
    01-stg:
      +docs: { node_color: "#5DADE2" }   # blue
    02-core:
      transactional-fact:
        +docs: { node_color: "#27AE60" }  # green
      periodic-snapshot-fact:
        +docs: { node_color: "#E67E22" }  # orange
      scd1:
        +docs: { node_color: "#8E44AD" }  # purple
    03-marts:
      accumulating-snapshot-fact:
        +docs: { node_color: "#D35400" }  # dark orange
      periodic-snapshot-fact:
        +docs: { node_color: "#BA4A00" }  # burnt orange
```

The two mart colors are deliberately in the same warm family as the core snapshot orange, so the lineage graph reads as “the snapshot-shaped things live over here” at a glance, while still being distinct enough to tell a core snapshot from a mart snapshot. It is a small touch, and it pays off every single time I open the docs to trace where a number came from.

## 8\. What’s still on my plate

Keeping with the tradition of the first two posts, here is what this one does not solve.

- `**dbt build --select state:modified+**` **is still not wired up.** This was on the list at the end of Part (2) and it is still on the list. Every run today rebuilds the full project, now 40 models instead of 29. At my data volume that is still fast enough not to hurt, but the correct way to scale is to compare the current `manifest.json` against the previous one and rebuild only what changed. That means persisting the prior manifest somewhere durable (probably the same GCS bucket the docs site reads from), wiring `dbt build --defer --state` into the daily workflow, and adding a PR-time job that runs `dbt build --select state:modified+` against a CI dataset so a pull request only builds and tests the slice it touched. I have not done it yet, but it is the next infrastructure piece I plan to pick up, and the growing model count is what will finally justify it.
- **The contract is deep on PRs and shallow everywhere else.** I grew it bottom-up, which means it is thorough exactly where a mart pulled on it (pull requests and reviews) and still thin on the other 14 event types. That is the honest state of a bottom-up contract: it is as complete as the questions I have actually asked. As I build more marts, the same process will extend it, one real demand at a time.
- **Marts are still narrow.** Two archetypes, 11 tables, all centered on developer activity and pull requests. There are whole questions I have not modeled yet (release cadence, issue resolution time, cross-repo contribution graphs). Each one will get its mart when I can name the question it answers, not before.

## 9\. Coming up in Part (4)

The dbt project now runs three layers deep: 1 staging fact, 28 core models, 11 marts, 4 macros, and a data contract that finally deserves the name. The data is shaped. What it is not yet is visible to anyone who does not read SQL.

That is Part (4). I want to put a BI layer on top of the marts and let the numbers speak without a query editor. I am leaning toward a code-defined, statically-built dashboard in the spirit of [Evidence](https://evidence.dev/), so the BI layer lives in git and deploys like the rest of the platform, rather than a clicked-together SaaS dashboard that drifts out of version control. That is still undecided, and I will reason through the trade-off in the open the way I have with every other choice in this series.

The through-line of the whole project holds here too: pick the boring, cheap, version-controlled option unless there is a concrete reason to pay for more, and be able to explain the reason either way.

See you in Part (4).