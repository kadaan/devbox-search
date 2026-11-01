# Serving query plans

Recorded 2026-11-01 against `ep-purple-river-auzjl04j.c-10.us-east-1.aws.neon.tech` with `node tools/explain-plans.mjs`.
Regenerate after changing apps/web/lib/search.ts or the indexes, and diff.
Warm plans (second run); see the script for what each statement is.

Table sizes: packages=253727, versions=1497814, variants=3909649, search_terms=254320

## What to look for

- **Name lookups** must be index walks: `packages_name_lower_idx` -> `versions` ->
  `variants_identity_key`, unioned with `variants_attr_path_idx`. A `Hash Join` or
  `Seq Scan on variants` here means the name/attr_path predicate has become an OR
  across two tables again (#26: ~4 s per lookup).
- **Batched fetch** must be one `Nested Loop` over `hits` with an index scan on
  `versions (package_id, ...)` per hit — never one query per hit.
- **Latest** (pick latest version, batched fetch latest) sorts one package's variant
  rows with a `SubPlan` per row on `variant_ranges_pkey` (#44). A few hundred index
  probes; a `Seq Scan on variant_ranges` would mean the presence subquery lost its
  `variant_id =` correlation.
- **Ranked terms** is two tiers. The `breadth` probe is an index scan of
  `search_terms_name_lower_idx` stopped by its LIMIT, and it gates the `prefix` arms with
  `One-Time Filter`s. For a narrow phrase (`go`, `hello`) only the first arm runs: a
  `BitmapOr` of `search_terms_name_lower_idx` and `search_terms_attr_path_lower_idx`. For a
  broad one (`python`) that arm shows `(never executed)`, the alias arm scans
  `search_terms_alias_idx` (a few hundred rows), and the two nearest-match arms are
  `Index Scan`s of `search_terms_top_level_name_knn_idx` / `search_terms_nested_name_knn_idx`
  ordered by `<->` under an `Incremental Sort` and a `Limit` of 50. A seq scan or a
  full-arm `Sort` of tens of thousands of rows here is the old cost coming back:
  similarity() over every prefix match was ~0.4 s for `python`, 70k rows. Check
  `python313Packages.` too: at 12k matches it is just over the probe's threshold, and a
  `Bitmap Heap Scan` + `top-N heapsort` in its nested arm means the planner expects fewer
  than 50 rows there — `search_terms_same_name_stats` is missing or was never analyzed
  (`(name = attr_path) IS TRUE` should estimate ~99.8% of the table). `-` has no
  trigrams and takes the scoring path with no probe. The `fuzzy` arm must sit under a
  `One-Time Filter` and show `(never executed)` whenever the prefix tier is full (`go`,
  `python`); it runs for `hello` and `-`. A `%` in the prefix arms, or a fuzzy arm that
  ran for `python`, is a regression: `%` is cheap to index but every GIN candidate is
  rechecked with similarity() — ~1.1 s of CPU for `python` when both tiers were one
  OR'd WHERE.

## Phrase search (/v2/search, /v1/search)

### ranked terms — q=go

Parameters: `["go","go"]` — Execution Time: 2.506 ms

```
Limit  (cost=2448.59..2448.71 rows=50 width=44) (actual time=1.679..1.692 rows=50 loops=1)
  Buffers: shared hit=60
  CTE breadth
    ->  Aggregate  (cost=18.08..18.09 rows=1 width=1) (actual time=0.272..0.272 rows=1 loops=1)
          Buffers: shared hit=34
          ->  Limit  (cost=0.42..17.77 rows=25 width=4) (actual time=0.027..0.242 rows=455 loops=1)
                Buffers: shared hit=34
                ->  Index Scan using search_terms_name_lower_idx on search_terms search_terms_1  (cost=0.42..17.77 rows=25 width=4) (actual time=0.027..0.205 rows=455 loops=1)
                      Index Cond: ((lower(name) >= 'go'::text) AND (lower(name) < 'gp'::text))
                      Filter: (lower(name) ~~ 'go%'::text)
                      Buffers: shared hit=34
  CTE prefix
    ->  Limit  (cost=381.25..381.37 rows=50 width=39) (actual time=1.583..1.594 rows=50 loops=1)
          Buffers: shared hit=60
          ->  Sort  (cost=381.25..381.45 rows=79 width=39) (actual time=1.583..1.590 rows=50 loops=1)
                Sort Key: (max((((CASE WHEN (lower(search_terms_3.name) = 'go'::text) THEN 1000 WHEN (lower(search_terms_3.attr_path) = 'go'::text) THEN 900 WHEN (lower(search_terms_3.name) ~~ 'go%'::text) THEN 800 WHEN (lower(search_terms_3.attr_path) ~~ 'go%'::text) THEN 700 ELSE 0 END)::double precision + ('100'::double precision * CASE WHEN (search_terms_3.name = search_terms_3.attr_path) THEN similarity(search_terms_3.name, 'go'::text) ELSE GREATEST(similarity(search_terms_3.name, 'go'::text), similarity(search_terms_3.attr_path, 'go'::text)) END)) + (CASE WHEN (search_terms_3.top_level_attr IS NOT NULL) THEN 25 ELSE 0 END)::double precision))) DESC, search_terms_3.name
                Sort Method: top-N heapsort  Memory: 30kB
                Buffers: shared hit=60
                ->  GroupAggregate  (cost=373.62..378.76 rows=79 width=39) (actual time=0.647..1.503 rows=440 loops=1)
                      Group Key: search_terms_3.package_id, search_terms_3.name
                      Buffers: shared hit=60
                      ->  Sort  (cost=373.62..373.82 rows=79 width=68) (actual time=0.611..0.638 rows=455 loops=1)
                            Sort Key: search_terms_3.package_id, search_terms_3.name
                            Sort Method: quicksort  Memory: 51kB
                            Buffers: shared hit=60
                            ->  Append  (cost=9.68..371.13 rows=79 width=68) (actual time=0.324..0.532 rows=455 loops=1)
                                  Buffers: shared hit=60
                                  ->  Result  (cost=9.68..285.41 rows=51 width=68) (actual time=0.324..0.482 rows=455 loops=1)
                                        One-Time Filter: (NOT (InitPlan 2).col1)
                                        Buffers: shared hit=60
                                        InitPlan 2
                                          ->  CTE Scan on breadth  (cost=0.00..0.02 rows=1 width=1) (actual time=0.273..0.273 rows=1 loops=1)
                                                Buffers: shared hit=34
                                        ->  Bitmap Heap Scan on search_terms search_terms_3  (cost=9.68..285.41 rows=51 width=68) (actual time=0.048..0.156 rows=455 loops=1)
                                              Recheck Cond: ((lower(name) ~~ 'go%'::text) OR (lower(attr_path) ~~ 'go%'::text))
                                              Filter: ((lower(name) ~~ 'go%'::text) OR (lower(attr_path) ~~ 'go%'::text))
                                              Heap Blocks: exact=16
                                              Buffers: shared hit=26
                                              ->  BitmapOr  (cost=9.66..9.66 rows=79 width=0) (actual time=0.041..0.041 rows=0 loops=1)
                                                    Buffers: shared hit=10
                                                    ->  Bitmap Index Scan on search_terms_name_lower_idx  (cost=0.00..4.81 rows=39 width=0) (actual time=0.020..0.020 rows=455 loops=1)
                                                          Index Cond: ((lower(name) >= 'go'::text) AND (lower(name) < 'gp'::text))
                                                          Buffers: shared hit=5
                                                    ->  Bitmap Index Scan on search_terms_attr_path_lower_idx  (cost=0.00..4.82 rows=40 width=0) (actual time=0.020..0.020 rows=455 loops=1)
                                                          Index Cond: ((lower(attr_path) >= 'go'::text) AND (lower(attr_path) < 'gp'::text))
                                                          Buffers: shared hit=5
                                  ->  Result  (cost=8.88..16.76 rows=2 width=68) (actual time=0.002..0.003 rows=0 loops=1)
                                        One-Time Filter: (InitPlan 3).col1
                                        InitPlan 3
                                          ->  CTE Scan on breadth breadth_1  (cost=0.00..0.02 rows=1 width=1) (actual time=0.001..0.001 rows=1 loops=1)
                                        ->  Bitmap Heap Scan on search_terms search_terms_4  (cost=8.88..16.76 rows=2 width=68) (never executed)
                                              Recheck Cond: ((lower(name) = 'go'::text) OR (lower(attr_path) = 'go'::text))
                                              ->  BitmapOr  (cost=8.86..8.86 rows=2 width=0) (never executed)
                                                    ->  Bitmap Index Scan on search_terms_name_lower_idx  (cost=0.00..4.43 rows=1 width=0) (never executed)
                                                          Index Cond: (lower(name) = 'go'::text)
                                                    ->  Bitmap Index Scan on search_terms_attr_path_lower_idx  (cost=0.00..4.43 rows=1 width=0) (never executed)
                                                          Index Cond: (lower(attr_path) = 'go'::text)
                                  ->  Result  (cost=29.62..33.64 rows=1 width=68) (actual time=0.001..0.002 rows=0 loops=1)
                                        One-Time Filter: (InitPlan 4).col1
                                        InitPlan 4
                                          ->  CTE Scan on breadth breadth_2  (cost=0.00..0.02 rows=1 width=1) (actual time=0.000..0.000 rows=1 loops=1)
                                        ->  Bitmap Heap Scan on search_terms search_terms_5  (cost=29.62..33.64 rows=1 width=68) (never executed)
                                              Recheck Cond: (((lower(name) ~~ 'go%'::text) OR (lower(attr_path) ~~ 'go%'::text)) AND ((name = attr_path) IS FALSE))
                                              Filter: ((lower(name) ~~ 'go%'::text) OR (lower(attr_path) ~~ 'go%'::text))
                                              ->  BitmapAnd  (cost=29.60..29.60 rows=1 width=0) (never executed)
                                                    ->  BitmapOr  (cost=9.63..9.63 rows=79 width=0) (never executed)
                                                          ->  Bitmap Index Scan on search_terms_name_lower_idx  (cost=0.00..4.81 rows=39 width=0) (never executed)
                                                                Index Cond: ((lower(name) >= 'go'::text) AND (lower(name) < 'gp'::text))
                                                          ->  Bitmap Index Scan on search_terms_attr_path_lower_idx  (cost=0.00..4.82 rows=40 width=0) (never executed)
                                                                Index Cond: ((lower(attr_path) >= 'go'::text) AND (lower(attr_path) < 'gp'::text))
                                                    ->  Bitmap Index Scan on search_terms_alias_idx  (cost=0.00..19.71 rows=693 width=0) (never executed)
                                  ->  Subquery Scan on "*SELECT* 4"  (cost=16.13..16.17 rows=3 width=68) (actual time=0.004..0.005 rows=0 loops=1)
                                        ->  Limit  (cost=16.13..16.14 rows=3 width=72) (actual time=0.004..0.005 rows=0 loops=1)
                                              InitPlan 5
                                                ->  CTE Scan on breadth breadth_3  (cost=0.00..0.02 rows=1 width=1) (actual time=0.000..0.000 rows=1 loops=1)
                                              ->  Sort  (cost=16.11..16.12 rows=3 width=72) (actual time=0.004..0.004 rows=0 loops=1)
                                                    Sort Key: ((lower(search_terms_6.name) <-> 'go'::text)), search_terms_6.name
                                                    Sort Method: quicksort  Memory: 25kB
                                                    ->  Result  (cost=4.31..16.09 rows=3 width=72) (actual time=0.001..0.001 rows=0 loops=1)
                                                          One-Time Filter: (InitPlan 5).col1
                                                          ->  Bitmap Heap Scan on search_terms search_terms_6  (cost=4.31..16.08 rows=3 width=68) (never executed)
                                                                Recheck Cond: ((lower(name) ~~ 'go%'::text) AND ((name = attr_path) IS TRUE) AND (top_level_attr IS NOT NULL))
                                                                ->  Bitmap Index Scan on search_terms_top_level_name_knn_idx  (cost=0.00..4.31 rows=3 width=0) (never executed)
                                                                      Index Cond: (lower(name) ~~ 'go%'::text)
                                  ->  Subquery Scan on "*SELECT* 5"  (cost=18.48..18.76 rows=22 width=68) (actual time=0.004..0.004 rows=0 loops=1)
                                        ->  Limit  (cost=18.48..18.54 rows=22 width=72) (actual time=0.003..0.004 rows=0 loops=1)
                                              InitPlan 6
                                                ->  CTE Scan on breadth breadth_4  (cost=0.00..0.02 rows=1 width=1) (actual time=0.000..0.000 rows=1 loops=1)
                                              ->  Sort  (cost=18.46..18.52 rows=22 width=72) (actual time=0.003..0.004 rows=0 loops=1)
                                                    Sort Key: ((lower(search_terms_7.name) <-> 'go'::text)), search_terms_7.name
                                                    Sort Method: quicksort  Memory: 25kB
                                                    ->  Result  (cost=0.42..17.97 rows=22 width=72) (actual time=0.001..0.001 rows=0 loops=1)
                                                          One-Time Filter: (InitPlan 6).col1
                                                          ->  Index Scan using search_terms_name_lower_idx on search_terms search_terms_7  (cost=0.42..17.86 rows=22 width=68) (never executed)
                                                                Index Cond: ((lower(name) >= 'go'::text) AND (lower(name) < 'gp'::text))
                                                                Filter: ((top_level_attr IS NULL) AND ((name = attr_path) IS TRUE) AND (lower(name) ~~ 'go%'::text))
  ->  Sort  (cost=2049.12..2049.37 rows=100 width=44) (actual time=1.678..1.682 rows=50 loops=1)
        Sort Key: (max(prefix.rank)) DESC, (min(prefix.name))
        Sort Method: quicksort  Memory: 28kB
        Buffers: shared hit=60
        ->  GroupAggregate  (cost=2043.80..2045.80 rows=100 width=44) (actual time=1.639..1.660 rows=50 loops=1)
              Group Key: prefix.package_id
              Buffers: shared hit=60
              ->  Sort  (cost=2043.80..2044.05 rows=100 width=42) (actual time=1.637..1.641 rows=50 loops=1)
                    Sort Key: prefix.package_id
                    Sort Method: quicksort  Memory: 27kB
                    Buffers: shared hit=60
                    ->  Append  (cost=0.00..2040.48 rows=100 width=42) (actual time=1.585..1.624 rows=50 loops=1)
                          Buffers: shared hit=60
                          ->  CTE Scan on prefix  (cost=0.00..1.00 rows=50 width=44) (actual time=1.585..1.600 rows=50 loops=1)
                                Buffers: shared hit=60
                          ->  Subquery Scan on fuzzy  (cost=2038.36..2038.98 rows=50 width=39) (actual time=0.019..0.020 rows=0 loops=1)
                                ->  Limit  (cost=2038.36..2038.48 rows=50 width=39) (actual time=0.019..0.020 rows=0 loops=1)
                                      InitPlan 8
                                        ->  Aggregate  (cost=1.12..1.14 rows=1 width=8) (actual time=0.007..0.007 rows=1 loops=1)
                                              ->  CTE Scan on prefix prefix_1  (cost=0.00..1.00 rows=50 width=0) (actual time=0.000..0.003 rows=50 loops=1)
                                      ->  Sort  (cost=2037.22..2037.35 rows=51 width=39) (actual time=0.018..0.019 rows=0 loops=1)
                                            Sort Key: (max((((CASE WHEN (lower(search_terms.name) = 'go'::text) THEN 1000 WHEN (lower(search_terms.attr_path) = 'go'::text) THEN 900 WHEN (lower(search_terms.name) ~~ 'go%'::text) THEN 800 WHEN (lower(search_terms.attr_path) ~~ 'go%'::text) THEN 700 ELSE 0 END)::double precision + ('100'::double precision * CASE WHEN (search_terms.name = search_terms.attr_path) THEN similarity(search_terms.name, 'go'::text) ELSE GREATEST(similarity(search_terms.name, 'go'::text), similarity(search_terms.attr_path, 'go'::text)) END)) + (CASE WHEN (search_terms.top_level_attr IS NOT NULL) THEN 25 ELSE 0 END)::double precision))) DESC, search_terms.name
                                            Sort Method: quicksort  Memory: 25kB
                                            ->  GroupAggregate  (cost=2032.46..2035.77 rows=51 width=39) (actual time=0.015..0.015 rows=0 loops=1)
                                                  Group Key: search_terms.package_id, search_terms.name
                                                  ->  Sort  (cost=2032.46..2032.59 rows=51 width=68) (actual time=0.014..0.015 rows=0 loops=1)
                                                        Sort Key: search_terms.package_id, search_terms.name
                                                        Sort Method: quicksort  Memory: 25kB
                                                        ->  Result  (cost=1845.75..2031.01 rows=51 width=68) (actual time=0.008..0.008 rows=0 loops=1)
                                                              One-Time Filter: ((InitPlan 8).col1 < 50)
                                                              ->  Bitmap Heap Scan on search_terms  (cost=1845.75..2031.01 rows=51 width=68) (never executed)
                                                                    Recheck Cond: ((name % 'go'::text) OR (attr_path % 'go'::text))
                                                                    Filter: ((lower(name) !~~ 'go%'::text) AND (lower(attr_path) !~~ 'go%'::text))
                                                                    ->  BitmapOr  (cost=1845.74..1845.74 rows=51 width=0) (never executed)
                                                                          ->  Bitmap Index Scan on search_terms_name_trgm_idx  (cost=0.00..922.86 rows=25 width=0) (never executed)
                                                                                Index Cond: (name % 'go'::text)
                                                                          ->  Bitmap Index Scan on search_terms_attr_path_trgm_idx  (cost=0.00..922.86 rows=25 width=0) (never executed)
                                                                                Index Cond: (attr_path % 'go'::text)
Planning:
  Buffers: shared hit=4
Planning Time: 1.349 ms
Execution Time: 2.506 ms
```

### batched fetch, latest — q=go (50 hits)

Parameters: `["35171,35225,35172,35184,35203,35208,35209,35233,35234,35238,35241,35252,35263,35409,35430,35547,35585,35589,35177,35178,35183,35191,35196,272526,35201,35202,35206,35210,35240,35245,35248,35250,35174,35185,35213,35231,35251,35260,35261,35266,35273,35299,35304,35376,35378,35384,35390,35392,35400,35401"]` — Execution Time: 14.875 ms

```
Limit  (cost=4241.35..107618.59 rows=50 width=1378) (actual time=6.270..14.764 rows=50 loops=1)
  Buffers: shared hit=23414
  ->  Unique  (cost=4241.35..107618.59 rows=50 width=1378) (actual time=6.269..14.758 rows=50 loops=1)
        Buffers: shared hit=23414
        ->  Incremental Sort  (cost=4241.35..107616.83 rows=706 width=1378) (actual time=6.268..14.720 rows=171 loops=1)
              Sort Key: hits.ord, variants.system, variants.attr_path
              Presorted Key: hits.ord
              Full-sort Groups: 6  Sort Method: quicksort  Average Memory: 55kB  Peak Memory: 55kB
              Buffers: shared hit=23414
              ->  Nested Loop  (cost=2131.84..107593.52 rows=706 width=1378) (actual time=4.910..14.570 rows=174 loops=1)
                    Buffers: shared hit=23414
                    ->  Nested Loop  (cost=2131.56..107383.45 rows=706 width=1323) (actual time=4.904..14.325 rows=174 loops=1)
                          Buffers: shared hit=22892
                          ->  Nested Loop  (cost=2131.14..107018.11 rows=706 width=340) (actual time=4.896..13.970 rows=174 loops=1)
                                Join Filter: (variants.version_id = versions_1.id)
                                Buffers: shared hit=22196
                                ->  Nested Loop  (cost=2130.71..106939.17 rows=50 width=53) (actual time=4.889..13.767 rows=50 loops=1)
                                      Buffers: shared hit=21890
                                      ->  Nested Loop  (cost=2130.29..106916.94 rows=50 width=30) (actual time=4.881..13.660 rows=50 loops=1)
                                            Buffers: shared hit=21690
                                            ->  Nested Loop  (cost=2129.86..106494.69 rows=50 width=12) (actual time=4.872..13.521 rows=50 loops=1)
                                                  Buffers: shared hit=21490
                                                  ->  Function Scan on unnest hits  (cost=0.00..0.50 rows=50 width=12) (actual time=0.008..0.015 rows=50 loops=1)
                                                  ->  Limit  (cost=2129.86..2129.86 rows=1 width=24) (actual time=0.270..0.270 rows=1 loops=50)
                                                        Buffers: shared hit=21490
                                                        ->  Sort  (cost=2129.86..2130.06 rows=79 width=24) (actual time=0.269..0.269 rows=1 loops=50)
                                                              Sort Key: (EXISTS(SubPlan 1)) DESC, ((SubPlan 2)) DESC, versions_1.sort_key DESC
                                                              Sort Method: top-N heapsort  Memory: 25kB
                                                              Buffers: shared hit=21490
                                                              ->  Nested Loop  (cost=0.86..2129.47 rows=79 width=24) (actual time=0.013..0.260 rows=46 loops=50)
                                                                    Buffers: shared hit=21490
                                                                    ->  Index Scan using versions_latest_idx on versions versions_1  (cost=0.43..66.26 rows=32 width=19) (actual time=0.003..0.007 rows=13 loops=50)
                                                                          Index Cond: ((package_id = hits.package_id) AND (prerelease = false))
                                                                          Buffers: shared hit=250
                                                                    ->  Index Scan using variants_identity_key on variants variants_1  (cost=0.43..36.09 rows=14 width=8) (actual time=0.002..0.003 rows=4 loops=629)
                                                                          Index Cond: (version_id = versions_1.id)
                                                                          Buffers: shared hit=2815
                                                                    SubPlan 1
                                                                      ->  Index Scan using variants_identity_key on variants b  (cost=0.43..36.09 rows=14 width=0) (actual time=0.001..0.001 rows=1 loops=2286)
                                                                            Index Cond: (version_id = versions_1.id)
                                                                            Filter: (NOT broken)
                                                                            Buffers: shared hit=9144
                                                                    SubPlan 2
                                                                      ->  Aggregate  (cost=8.45..8.46 rows=1 width=4) (actual time=0.003..0.003 rows=1 loops=2286)
                                                                            Buffers: shared hit=9281
                                                                            ->  Index Scan using variant_ranges_variant_id_first_seq_pk on variant_ranges r  (cost=0.43..8.45 rows=1 width=4) (actual time=0.002..0.002 rows=0 loops=2286)
                                                                                  Index Cond: (variant_id = variants_1.id)
                                                                                  Filter: (NOT seeded)
                                                                                  Rows Removed by Filter: 1
                                                                                  Buffers: shared hit=9281
                                            ->  Index Scan using versions_pkey on versions  (cost=0.43..8.45 rows=1 width=18) (actual time=0.002..0.002 rows=1 loops=50)
                                                  Index Cond: (id = versions_1.id)
                                                  Buffers: shared hit=200
                                      ->  Index Scan using packages_pkey on packages  (cost=0.42..0.44 rows=1 width=31) (actual time=0.002..0.002 rows=1 loops=50)
                                            Index Cond: (id = versions.package_id)
                                            Buffers: shared hit=200
                                ->  Index Scan using variants_identity_key on variants  (cost=0.43..1.40 rows=14 width=299) (actual time=0.002..0.003 rows=3 loops=50)
                                      Index Cond: (version_id = versions.id)
                                      Buffers: shared hit=306
                          ->  Index Scan using meta_pkey on meta  (cost=0.42..0.52 rows=1 width=991) (actual time=0.002..0.002 rows=1 loops=174)
                                Index Cond: (id = variants.meta_id)
                                Buffers: shared hit=696
                    ->  Index Scan using commits_pkey on commits commit_hash  (cost=0.28..0.30 rows=1 width=53) (actual time=0.001..0.001 rows=1 loops=174)
                          Index Cond: (seq = variants.commit_seq)
                          Buffers: shared hit=522
Planning:
  Buffers: shared hit=115
Planning Time: 33.238 ms
Execution Time: 14.875 ms
```

### batched fetch, all versions — q=go (50 hits)

Parameters: `["35171,35225,35172,35184,35203,35208,35209,35233,35234,35238,35241,35252,35263,35409,35430,35547,35585,35589,35177,35178,35183,35191,35196,272526,35201,35202,35206,35210,35240,35245,35248,35250,35174,35185,35213,35231,35251,35260,35261,35266,35273,35299,35304,35376,35378,35384,35390,35392,35400,35401"]` — Execution Time: 12.561 ms

```
Limit  (cost=119.91..1568.91 rows=1000 width=1393) (actual time=3.529..12.364 rows=649 loops=1)
  Buffers: shared hit=13537
  ->  Unique  (cost=119.91..5979.69 rows=4044 width=1393) (actual time=3.527..12.311 rows=649 loops=1)
        Buffers: shared hit=13537
        ->  Incremental Sort  (cost=119.91..5949.36 rows=4044 width=1393) (actual time=3.527..11.773 rows=2361 loops=1)
              Sort Key: hits.ord, versions.sort_key DESC, versions.version, variants.system, variants.attr_path
              Presorted Key: hits.ord
              Full-sort Groups: 21  Sort Method: quicksort  Average Memory: 70kB  Peak Memory: 75kB
              Pre-sorted Groups: 27  Sort Method: quicksort  Average Memory: 581kB  Peak Memory: 581kB
              Buffers: shared hit=13537
              ->  Nested Loop  (cost=1.99..5769.66 rows=4044 width=1393) (actual time=0.052..8.236 rows=2361 loops=1)
                    Buffers: shared hit=13537
                    ->  Nested Loop  (cost=1.70..5121.54 rows=4044 width=1338) (actual time=0.044..6.803 rows=2361 loops=1)
                          Buffers: shared hit=12811
                          ->  Nested Loop  (cost=1.28..3028.84 rows=4044 width=355) (actual time=0.038..3.169 rows=2361 loops=1)
                                Buffers: shared hit=3367
                                ->  Nested Loop  (cost=0.85..507.87 rows=1633 width=64) (actual time=0.031..0.574 rows=649 loops=1)
                                      Join Filter: (versions.package_id = hits.package_id)
                                      Buffers: shared hit=457
                                      ->  Nested Loop  (cost=0.42..418.38 rows=50 width=43) (actual time=0.019..0.124 rows=50 loops=1)
                                            Buffers: shared hit=200
                                            ->  Function Scan on unnest hits  (cost=0.00..0.50 rows=50 width=12) (actual time=0.008..0.015 rows=50 loops=1)
                                            ->  Index Scan using packages_pkey on packages  (cost=0.42..8.36 rows=1 width=31) (actual time=0.002..0.002 rows=1 loops=50)
                                                  Index Cond: (id = hits.package_id)
                                                  Buffers: shared hit=200
                                      ->  Index Scan using versions_package_version_key on versions  (cost=0.43..1.38 rows=33 width=33) (actual time=0.003..0.007 rows=13 loops=50)
                                            Index Cond: (package_id = packages.id)
                                            Buffers: shared hit=257
                                ->  Index Scan using variants_identity_key on variants  (cost=0.43..1.40 rows=14 width=299) (actual time=0.002..0.003 rows=4 loops=649)
                                      Index Cond: (version_id = versions.id)
                                      Buffers: shared hit=2910
                          ->  Index Scan using meta_pkey on meta  (cost=0.42..0.52 rows=1 width=991) (actual time=0.001..0.001 rows=1 loops=2361)
                                Index Cond: (id = variants.meta_id)
                                Buffers: shared hit=9444
                    ->  Memoize  (cost=0.29..0.31 rows=1 width=53) (actual time=0.000..0.000 rows=1 loops=2361)
                          Cache Key: variants.commit_seq
                          Cache Mode: logical
                          Hits: 2119  Misses: 242  Evictions: 0  Overflows: 0  Memory Usage: 38kB
                          Buffers: shared hit=726
                          ->  Index Scan using commits_pkey on commits commit_hash  (cost=0.28..0.30 rows=1 width=53) (actual time=0.001..0.001 rows=1 loops=242)
                                Index Cond: (seq = variants.commit_seq)
                                Buffers: shared hit=726
Planning:
  Buffers: shared hit=69
Planning Time: 10.386 ms
Execution Time: 12.561 ms
```

### ranked terms — q=python

Parameters: `["python","python"]` — Execution Time: 36.537 ms

```
Limit  (cost=22077.05..22077.18 rows=50 width=44) (actual time=36.303..36.325 rows=50 loops=1)
  Buffers: shared hit=3692
  CTE breadth
    ->  Aggregate  (cost=789.01..789.02 rows=1 width=1) (actual time=5.548..5.549 rows=1 loops=1)
          Buffers: shared hit=398
          ->  Limit  (cost=0.42..664.01 rows=10000 width=4) (actual time=0.026..4.921 rows=10000 loops=1)
                Buffers: shared hit=398
                ->  Index Scan using search_terms_name_lower_idx on search_terms search_terms_1  (cost=0.42..4754.63 rows=71644 width=4) (actual time=0.025..4.125 rows=10000 loops=1)
                      Index Cond: ((lower(name) >= 'python'::text) AND (lower(name) < 'pythoo'::text))
                      Filter: (lower(name) ~~ 'python%'::text)
                      Buffers: shared hit=398
  CTE prefix
    ->  Limit  (cost=15845.47..15845.60 rows=50 width=39) (actual time=36.197..36.213 rows=50 loops=1)
          Buffers: shared hit=3692
          ->  Sort  (cost=15845.47..15876.35 rows=12349 width=39) (actual time=36.196..36.208 rows=50 loops=1)
                Sort Key: (max((((CASE WHEN (lower(search_terms_3.name) = 'python'::text) THEN 1000 WHEN (lower(search_terms_3.attr_path) = 'python'::text) THEN 900 WHEN (lower(search_terms_3.name) ~~ 'python%'::text) THEN 800 WHEN (lower(search_terms_3.attr_path) ~~ 'python%'::text) THEN 700 ELSE 0 END)::double precision + ('100'::double precision * CASE WHEN (search_terms_3.name = search_terms_3.attr_path) THEN similarity(search_terms_3.name, 'python'::text) ELSE GREATEST(similarity(search_terms_3.name, 'python'::text), similarity(search_terms_3.attr_path, 'python'::text)) END)) + (CASE WHEN (search_terms_3.top_level_attr IS NOT NULL) THEN 25 ELSE 0 END)::double precision))) DESC, search_terms_3.name
                Sort Method: top-N heapsort  Memory: 30kB
                Buffers: shared hit=3692
                ->  HashAggregate  (cost=15311.76..15435.25 rows=12349 width=39) (actual time=36.069..36.156 rows=203 loops=1)
                      Group Key: search_terms_3.package_id, search_terms_3.name
                      Batches: 1  Memory Usage: 433kB
                      Buffers: shared hit=3692
                      ->  Append  (cost=0.02..8828.69 rows=123487 width=68) (actual time=5.590..35.132 rows=247 loops=1)
                            Buffers: shared hit=3692
                            ->  Result  (cost=0.02..8049.98 rows=123049 width=68) (actual time=5.553..5.554 rows=0 loops=1)
                                  One-Time Filter: (NOT (InitPlan 2).col1)
                                  Buffers: shared hit=398
                                  InitPlan 2
                                    ->  CTE Scan on breadth  (cost=0.00..0.02 rows=1 width=1) (actual time=5.550..5.550 rows=1 loops=1)
                                          Buffers: shared hit=398
                                  ->  Seq Scan on search_terms search_terms_3  (cost=0.02..8049.98 rows=123049 width=68) (never executed)
                                        Filter: ((lower(name) ~~ 'python%'::text) OR (lower(attr_path) ~~ 'python%'::text))
                            ->  Result  (cost=8.88..16.76 rows=2 width=68) (actual time=0.035..0.042 rows=16 loops=1)
                                  One-Time Filter: (InitPlan 3).col1
                                  Buffers: shared hit=8
                                  InitPlan 3
                                    ->  CTE Scan on breadth breadth_1  (cost=0.00..0.02 rows=1 width=1) (actual time=0.000..0.001 rows=1 loops=1)
                                  ->  Bitmap Heap Scan on search_terms search_terms_4  (cost=8.88..16.76 rows=2 width=68) (actual time=0.033..0.036 rows=16 loops=1)
                                        Recheck Cond: ((lower(name) = 'python'::text) OR (lower(attr_path) = 'python'::text))
                                        Heap Blocks: exact=2
                                        Buffers: shared hit=8
                                        ->  BitmapOr  (cost=8.86..8.86 rows=2 width=0) (actual time=0.025..0.026 rows=0 loops=1)
                                              Buffers: shared hit=6
                                              ->  Bitmap Index Scan on search_terms_name_lower_idx  (cost=0.00..4.43 rows=1 width=0) (actual time=0.018..0.019 rows=16 loops=1)
                                                    Index Cond: (lower(name) = 'python'::text)
                                                    Buffers: shared hit=3
                                              ->  Bitmap Index Scan on search_terms_attr_path_lower_idx  (cost=0.00..4.43 rows=1 width=0) (actual time=0.006..0.006 rows=1 loops=1)
                                                    Index Cond: (lower(attr_path) = 'python'::text)
                                                    Buffers: shared hit=3
                            ->  Result  (cost=0.30..49.71 rows=336 width=68) (actual time=0.257..0.405 rows=167 loops=1)
                                  One-Time Filter: (InitPlan 4).col1
                                  Buffers: shared hit=160
                                  InitPlan 4
                                    ->  CTE Scan on breadth breadth_2  (cost=0.00..0.02 rows=1 width=1) (actual time=0.000..0.001 rows=1 loops=1)
                                  ->  Index Scan using search_terms_alias_idx on search_terms search_terms_5  (cost=0.30..49.71 rows=336 width=68) (actual time=0.255..0.384 rows=167 loops=1)
                                        Filter: ((lower(name) ~~ 'python%'::text) OR (lower(attr_path) ~~ 'python%'::text))
                                        Rows Removed by Filter: 431
                                        Buffers: shared hit=160
                            ->  Subquery Scan on "*SELECT* 4"  (cost=1.76..76.99 rows=50 width=68) (actual time=0.981..0.987 rows=14 loops=1)
                                  Buffers: shared hit=198
                                  ->  Limit  (cost=1.76..76.49 rows=50 width=72) (actual time=0.979..0.983 rows=14 loops=1)
                                        Buffers: shared hit=198
                                        InitPlan 5
                                          ->  CTE Scan on breadth breadth_3  (cost=0.00..0.02 rows=1 width=1) (actual time=0.000..0.001 rows=1 loops=1)
                                        ->  Incremental Sort  (cost=1.74..12870.94 rows=8610 width=72) (actual time=0.978..0.979 rows=14 loops=1)
                                              Sort Key: ((lower(search_terms_6.name) <-> 'python'::text)), search_terms_6.name
                                              Presorted Key: ((lower(search_terms_6.name) <-> 'python'::text))
                                              Full-sort Groups: 1  Sort Method: quicksort  Average Memory: 26kB  Peak Memory: 26kB
                                              Buffers: shared hit=198
                                              ->  Result  (cost=0.28..12483.53 rows=8610 width=72) (actual time=0.152..0.964 rows=14 loops=1)
                                                    One-Time Filter: (InitPlan 5).col1
                                                    Buffers: shared hit=198
                                                    ->  Index Scan using search_terms_top_level_name_knn_idx on search_terms search_terms_6  (cost=0.28..12440.48 rows=8610 width=68) (actual time=0.148..0.933 rows=14 loops=1)
                                                          Index Cond: (lower(name) ~~ 'python%'::text)
                                                          Rows Removed by Index Recheck: 13
                                                          Order By: (lower(name) <-> 'python'::text)
                                                          Buffers: shared hit=198
                            ->  Subquery Scan on "*SELECT* 5"  (cost=0.60..17.81 rows=50 width=68) (actual time=27.505..28.118 rows=50 loops=1)
                                  Buffers: shared hit=2928
                                  ->  Limit  (cost=0.60..17.31 rows=50 width=72) (actual time=27.504..28.110 rows=50 loops=1)
                                        Buffers: shared hit=2928
                                        InitPlan 6
                                          ->  CTE Scan on breadth breadth_4  (cost=0.00..0.02 rows=1 width=1) (actual time=0.000..0.001 rows=1 loops=1)
                                        ->  Incremental Sort  (cost=0.58..21004.35 rows=62839 width=72) (actual time=27.503..28.104 rows=50 loops=1)
                                              Sort Key: ((lower(search_terms_7.name) <-> 'python'::text)), search_terms_7.name
                                              Presorted Key: ((lower(search_terms_7.name) <-> 'python'::text))
                                              Full-sort Groups: 2  Sort Methods: top-N heapsort, quicksort  Average Memory: 29kB  Peak Memory: 29kB
                                              Pre-sorted Groups: 1  Sort Method: top-N heapsort  Average Memory: 27kB  Peak Memory: 27kB
                                              Buffers: shared hit=2928
                                              ->  Result  (cost=0.28..18179.26 rows=62839 width=72) (actual time=27.313..28.024 rows=170 loops=1)
                                                    One-Time Filter: (InitPlan 6).col1
                                                    Buffers: shared hit=2928
                                                    ->  Index Scan using search_terms_nested_name_knn_idx on search_terms search_terms_7  (cost=0.28..17865.06 rows=62839 width=68) (actual time=27.298..27.640 rows=170 loops=1)
                                                          Index Cond: (lower(name) ~~ 'python%'::text)
                                                          Rows Removed by Index Recheck: 12
                                                          Order By: (lower(name) <-> 'python'::text)
                                                          Buffers: shared hit=2928
  ->  Sort  (cost=5442.43..5442.68 rows=100 width=44) (actual time=36.301..36.307 rows=50 loops=1)
        Sort Key: (max(prefix.rank)) DESC, (min(prefix.name))
        Sort Method: quicksort  Memory: 28kB
        Buffers: shared hit=3692
        ->  GroupAggregate  (cost=5437.11..5439.11 rows=100 width=44) (actual time=36.267..36.289 rows=50 loops=1)
              Group Key: prefix.package_id
              Buffers: shared hit=3692
              ->  Sort  (cost=5437.11..5437.36 rows=100 width=42) (actual time=36.262..36.268 rows=50 loops=1)
                    Sort Key: prefix.package_id
                    Sort Method: quicksort  Memory: 27kB
                    Buffers: shared hit=3692
                    ->  Append  (cost=0.00..5433.79 rows=100 width=42) (actual time=36.200..36.252 rows=50 loops=1)
                          Buffers: shared hit=3692
                          ->  CTE Scan on prefix  (cost=0.00..1.00 rows=50 width=44) (actual time=36.199..36.214 rows=50 loops=1)
                                Buffers: shared hit=3692
                          ->  Subquery Scan on fuzzy  (cost=5431.66..5432.29 rows=50 width=39) (actual time=0.030..0.032 rows=0 loops=1)
                                ->  Limit  (cost=5431.66..5431.79 rows=50 width=39) (actual time=0.029..0.031 rows=0 loops=1)
                                      InitPlan 8
                                        ->  Aggregate  (cost=1.12..1.14 rows=1 width=8) (actual time=0.009..0.010 rows=1 loops=1)
                                              ->  CTE Scan on prefix prefix_1  (cost=0.00..1.00 rows=50 width=0) (actual time=0.000..0.004 rows=50 loops=1)
                                      ->  Sort  (cost=5430.53..5437.08 rows=2620 width=39) (actual time=0.028..0.030 rows=0 loops=1)
                                            Sort Key: (max((((CASE WHEN (lower(search_terms.name) = 'python'::text) THEN 1000 WHEN (lower(search_terms.attr_path) = 'python'::text) THEN 900 WHEN (lower(search_terms.name) ~~ 'python%'::text) THEN 800 WHEN (lower(search_terms.attr_path) ~~ 'python%'::text) THEN 700 ELSE 0 END)::double precision + ('100'::double precision * CASE WHEN (search_terms.name = search_terms.attr_path) THEN similarity(search_terms.name, 'python'::text) ELSE GREATEST(similarity(search_terms.name, 'python'::text), similarity(search_terms.attr_path, 'python'::text)) END)) + (CASE WHEN (search_terms.top_level_attr IS NOT NULL) THEN 25 ELSE 0 END)::double precision))) DESC, search_terms.name
                                            Sort Method: quicksort  Memory: 25kB
                                            ->  HashAggregate  (cost=5317.30..5343.50 rows=2620 width=39) (actual time=0.022..0.024 rows=0 loops=1)
                                                  Group Key: search_terms.package_id, search_terms.name
                                                  Batches: 1  Memory Usage: 121kB
                                                  ->  Result  (cost=1975.99..5179.75 rows=2620 width=68) (actual time=0.011..0.013 rows=0 loops=1)
                                                        One-Time Filter: ((InitPlan 8).col1 < 50)
                                                        ->  Bitmap Heap Scan on search_terms  (cost=1975.99..5179.75 rows=2620 width=68) (never executed)
                                                              Recheck Cond: ((name % 'python'::text) OR (attr_path % 'python'::text))
                                                              Filter: ((lower(name) !~~ 'python%'::text) AND (lower(attr_path) !~~ 'python%'::text))
                                                              ->  BitmapOr  (cost=1975.99..1975.99 rows=5118 width=0) (never executed)
                                                                    ->  Bitmap Index Scan on search_terms_name_trgm_idx  (cost=0.00..987.34 rows=2559 width=0) (never executed)
                                                                          Index Cond: (name % 'python'::text)
                                                                    ->  Bitmap Index Scan on search_terms_attr_path_trgm_idx  (cost=0.00..987.34 rows=2560 width=0) (never executed)
                                                                          Index Cond: (attr_path % 'python'::text)
Planning:
  Buffers: shared hit=4
Planning Time: 1.428 ms
Execution Time: 36.537 ms
```

### batched fetch, latest — q=python (50 hits)

Parameters: `["113125,113133,113126,113128,115769,113132,113130,182211,113134,113127,113131,113129,144106,155721,167017,275383,114993,169824,174570,180462,121545,130664,140883,152209,163594,113437,113601,114071,115023,115037,167503,167540,167741,168535,169306,169865,169884,171515,171561,171824,172863,173875,174633,174660,176746,176800,177119,178411,179650,180538"]` — Execution Time: 10.464 ms

```
Limit  (cost=4241.35..107618.59 rows=50 width=1378) (actual time=6.524..10.355 rows=50 loops=1)
  Buffers: shared hit=18037
  ->  Unique  (cost=4241.35..107618.59 rows=50 width=1378) (actual time=6.523..10.349 rows=50 loops=1)
        Buffers: shared hit=18037
        ->  Incremental Sort  (cost=4241.35..107616.83 rows=706 width=1378) (actual time=6.522..10.317 rows=123 loops=1)
              Sort Key: hits.ord, variants.system, variants.attr_path
              Presorted Key: hits.ord
              Full-sort Groups: 4  Sort Method: quicksort  Average Memory: 65kB  Peak Memory: 65kB
              Buffers: shared hit=18037
              ->  Nested Loop  (cost=2131.84..107593.52 rows=706 width=1378) (actual time=2.319..10.201 rows=126 loops=1)
                    Buffers: shared hit=18037
                    ->  Nested Loop  (cost=2131.56..107383.45 rows=706 width=1323) (actual time=2.314..9.976 rows=126 loops=1)
                          Buffers: shared hit=17659
                          ->  Nested Loop  (cost=2131.14..107018.11 rows=706 width=340) (actual time=2.306..9.680 rows=126 loops=1)
                                Join Filter: (variants.version_id = versions_1.id)
                                Buffers: shared hit=17155
                                ->  Nested Loop  (cost=2130.71..106939.17 rows=50 width=53) (actual time=2.301..9.516 rows=50 loops=1)
                                      Buffers: shared hit=16928
                                      ->  Nested Loop  (cost=2130.29..106916.94 rows=50 width=30) (actual time=2.295..9.356 rows=50 loops=1)
                                            Buffers: shared hit=16728
                                            ->  Nested Loop  (cost=2129.86..106494.69 rows=50 width=12) (actual time=2.288..9.195 rows=50 loops=1)
                                                  Buffers: shared hit=16528
                                                  ->  Function Scan on unnest hits  (cost=0.00..0.50 rows=50 width=12) (actual time=0.008..0.015 rows=50 loops=1)
                                                  ->  Limit  (cost=2129.86..2129.86 rows=1 width=24) (actual time=0.183..0.183 rows=1 loops=50)
                                                        Buffers: shared hit=16528
                                                        ->  Sort  (cost=2129.86..2130.06 rows=79 width=24) (actual time=0.183..0.183 rows=1 loops=50)
                                                              Sort Key: (EXISTS(SubPlan 1)) DESC, ((SubPlan 2)) DESC, versions_1.sort_key DESC
                                                              Sort Method: top-N heapsort  Memory: 25kB
                                                              Buffers: shared hit=16528
                                                              ->  Nested Loop  (cost=0.86..2129.47 rows=79 width=24) (actual time=0.014..0.175 rows=35 loops=50)
                                                                    Buffers: shared hit=16528
                                                                    ->  Index Scan using versions_latest_idx on versions versions_1  (cost=0.43..66.26 rows=32 width=19) (actual time=0.004..0.007 rows=10 loops=50)
                                                                          Index Cond: ((package_id = hits.package_id) AND (prerelease = false))
                                                                          Buffers: shared hit=235
                                                                    ->  Index Scan using variants_identity_key on variants variants_1  (cost=0.43..36.09 rows=14 width=8) (actual time=0.002..0.003 rows=3 loops=523)
                                                                          Index Cond: (version_id = versions_1.id)
                                                                          Buffers: shared hit=2213
                                                                    SubPlan 1
                                                                      ->  Index Scan using variants_identity_key on variants b  (cost=0.43..36.09 rows=14 width=0) (actual time=0.001..0.001 rows=1 loops=1756)
                                                                            Index Cond: (version_id = versions_1.id)
                                                                            Filter: (NOT broken)
                                                                            Buffers: shared hit=7024
                                                                    SubPlan 2
                                                                      ->  Aggregate  (cost=8.45..8.46 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=1756)
                                                                            Buffers: shared hit=7056
                                                                            ->  Index Scan using variant_ranges_variant_id_first_seq_pk on variant_ranges r  (cost=0.43..8.45 rows=1 width=4) (actual time=0.002..0.002 rows=0 loops=1756)
                                                                                  Index Cond: (variant_id = variants_1.id)
                                                                                  Filter: (NOT seeded)
                                                                                  Rows Removed by Filter: 1
                                                                                  Buffers: shared hit=7056
                                            ->  Index Scan using versions_pkey on versions  (cost=0.43..8.45 rows=1 width=18) (actual time=0.003..0.003 rows=1 loops=50)
                                                  Index Cond: (id = versions_1.id)
                                                  Buffers: shared hit=200
                                      ->  Index Scan using packages_pkey on packages  (cost=0.42..0.44 rows=1 width=31) (actual time=0.003..0.003 rows=1 loops=50)
                                            Index Cond: (id = versions.package_id)
                                            Buffers: shared hit=200
                                ->  Index Scan using variants_identity_key on variants  (cost=0.43..1.40 rows=14 width=299) (actual time=0.002..0.002 rows=3 loops=50)
                                      Index Cond: (version_id = versions.id)
                                      Buffers: shared hit=227
                          ->  Index Scan using meta_pkey on meta  (cost=0.42..0.52 rows=1 width=991) (actual time=0.002..0.002 rows=1 loops=126)
                                Index Cond: (id = variants.meta_id)
                                Buffers: shared hit=504
                    ->  Index Scan using commits_pkey on commits commit_hash  (cost=0.28..0.30 rows=1 width=53) (actual time=0.001..0.001 rows=1 loops=126)
                          Index Cond: (seq = variants.commit_seq)
                          Buffers: shared hit=378
Planning:
  Buffers: shared hit=115
Planning Time: 17.985 ms
Execution Time: 10.464 ms
```

### batched fetch, all versions — q=python (50 hits)

Parameters: `["113125,113133,113126,113128,115769,113132,113130,182211,113134,113127,113131,113129,144106,155721,167017,275383,114993,169824,174570,180462,121545,130664,140883,152209,163594,113437,113601,114071,115023,115037,167503,167540,167741,168535,169306,169865,169884,171515,171561,171824,172863,173875,174633,174660,176746,176800,177119,178411,179650,180538"]` — Execution Time: 12.023 ms

```
Limit  (cost=119.91..1568.91 rows=1000 width=1393) (actual time=3.303..11.825 rows=647 loops=1)
  Buffers: shared hit=12489
  ->  Unique  (cost=119.91..5979.69 rows=4044 width=1393) (actual time=3.302..11.771 rows=647 loops=1)
        Buffers: shared hit=12489
        ->  Incremental Sort  (cost=119.91..5949.36 rows=4044 width=1393) (actual time=3.301..11.217 rows=2202 loops=1)
              Sort Key: hits.ord, versions.sort_key DESC, versions.version, variants.system, variants.attr_path
              Presorted Key: hits.ord
              Full-sort Groups: 19  Sort Method: quicksort  Average Memory: 126kB  Peak Memory: 126kB
              Pre-sorted Groups: 13  Sort Method: quicksort  Average Memory: 912kB  Peak Memory: 912kB
              Buffers: shared hit=12489
              ->  Nested Loop  (cost=1.99..5769.66 rows=4044 width=1393) (actual time=0.060..7.899 rows=2202 loops=1)
                    Buffers: shared hit=12489
                    ->  Nested Loop  (cost=1.70..5121.54 rows=4044 width=1338) (actual time=0.050..6.567 rows=2202 loops=1)
                          Buffers: shared hit=11973
                          ->  Nested Loop  (cost=1.28..3028.84 rows=4044 width=355) (actual time=0.043..3.178 rows=2202 loops=1)
                                Buffers: shared hit=3165
                                ->  Nested Loop  (cost=0.85..507.87 rows=1633 width=64) (actual time=0.035..0.680 rows=647 loops=1)
                                      Join Filter: (versions.package_id = hits.package_id)
                                      Buffers: shared hit=436
                                      ->  Nested Loop  (cost=0.42..418.38 rows=50 width=43) (actual time=0.021..0.190 rows=50 loops=1)
                                            Buffers: shared hit=200
                                            ->  Function Scan on unnest hits  (cost=0.00..0.50 rows=50 width=12) (actual time=0.008..0.017 rows=50 loops=1)
                                            ->  Index Scan using packages_pkey on packages  (cost=0.42..8.36 rows=1 width=31) (actual time=0.003..0.003 rows=1 loops=50)
                                                  Index Cond: (id = hits.package_id)
                                                  Buffers: shared hit=200
                                      ->  Index Scan using versions_package_version_key on versions  (cost=0.43..1.38 rows=33 width=33) (actual time=0.004..0.007 rows=13 loops=50)
                                            Index Cond: (package_id = packages.id)
                                            Buffers: shared hit=236
                                ->  Index Scan using variants_identity_key on variants  (cost=0.43..1.40 rows=14 width=299) (actual time=0.002..0.003 rows=3 loops=647)
                                      Index Cond: (version_id = versions.id)
                                      Buffers: shared hit=2729
                          ->  Index Scan using meta_pkey on meta  (cost=0.42..0.52 rows=1 width=991) (actual time=0.001..0.001 rows=1 loops=2202)
                                Index Cond: (id = variants.meta_id)
                                Buffers: shared hit=8808
                    ->  Memoize  (cost=0.29..0.31 rows=1 width=53) (actual time=0.000..0.000 rows=1 loops=2202)
                          Cache Key: variants.commit_seq
                          Cache Mode: logical
                          Hits: 2030  Misses: 172  Evictions: 0  Overflows: 0  Memory Usage: 27kB
                          Buffers: shared hit=516
                          ->  Index Scan using commits_pkey on commits commit_hash  (cost=0.28..0.30 rows=1 width=53) (actual time=0.002..0.002 rows=1 loops=172)
                                Index Cond: (seq = variants.commit_seq)
                                Buffers: shared hit=516
Planning:
  Buffers: shared hit=69
Planning Time: 10.504 ms
Execution Time: 12.023 ms
```

### ranked terms — q=python313Packages.

Parameters: `["python313Packages.","python313Packages."]` — Execution Time: 28.890 ms

```
Limit  (cost=39158.14..39158.27 rows=50 width=44) (actual time=26.417..28.711 rows=50 loops=1)
  Buffers: shared hit=3018
  CTE breadth
    ->  Aggregate  (cost=1118.37..1118.38 rows=1 width=1) (actual time=9.391..9.392 rows=1 loops=1)
          Buffers: shared hit=682
          ->  Limit  (cost=0.42..993.37 rows=10000 width=4) (actual time=0.039..8.791 rows=10000 loops=1)
                Buffers: shared hit=682
                ->  Index Scan using search_terms_name_lower_idx on search_terms search_terms_1  (cost=0.42..1270.80 rows=12794 width=4) (actual time=0.039..8.002 rows=10000 loops=1)
                      Index Cond: ((lower(name) >= 'python313packages.'::text) AND (lower(name) < 'python313packages/'::text))
                      Filter: (lower(name) ~~ 'python313packages.%'::text)
                      Buffers: shared hit=682
  CTE prefix
    ->  Limit  (cost=5941.55..5941.68 rows=50 width=39) (actual time=21.843..21.860 rows=50 loops=1)
          Buffers: shared hit=2974
          ->  Sort  (cost=5941.55..5947.83 rows=2512 width=39) (actual time=21.843..21.856 rows=50 loops=1)
                Sort Key: (max((((CASE WHEN (lower(search_terms_3.name) = 'python313packages.'::text) THEN 1000 WHEN (lower(search_terms_3.attr_path) = 'python313packages.'::text) THEN 900 WHEN (lower(search_terms_3.name) ~~ 'python313packages.%'::text) THEN 800 WHEN (lower(search_terms_3.attr_path) ~~ 'python313packages.%'::text) THEN 700 ELSE 0 END)::double precision + ('100'::double precision * CASE WHEN (search_terms_3.name = search_terms_3.attr_path) THEN similarity(search_terms_3.name, 'python313Packages.'::text) ELSE GREATEST(similarity(search_terms_3.name, 'python313Packages.'::text), similarity(search_terms_3.attr_path, 'python313Packages.'::text)) END)) + (CASE WHEN (search_terms_3.top_level_attr IS NOT NULL) THEN 25 ELSE 0 END)::double precision))) DESC, search_terms_3.name
                Sort Method: quicksort  Memory: 28kB
                Buffers: shared hit=2974
                ->  HashAggregate  (cost=5832.98..5858.10 rows=2512 width=39) (actual time=21.810..21.836 rows=50 loops=1)
                      Group Key: search_terms_3.package_id, search_terms_3.name
                      Batches: 1  Memory Usage: 121kB
                      Buffers: shared hit=2974
                      ->  Append  (cost=704.02..4514.45 rows=25115 width=68) (actual time=21.652..21.679 rows=50 loops=1)
                            Buffers: shared hit=2974
                            ->  Result  (cost=704.02..4091.38 rows=24945 width=68) (actual time=9.395..9.396 rows=0 loops=1)
                                  One-Time Filter: (NOT (InitPlan 2).col1)
                                  Buffers: shared hit=682
                                  InitPlan 2
                                    ->  CTE Scan on breadth  (cost=0.00..0.02 rows=1 width=1) (actual time=9.392..9.393 rows=1 loops=1)
                                          Buffers: shared hit=682
                                  ->  Bitmap Heap Scan on search_terms search_terms_3  (cost=704.02..4091.38 rows=24945 width=68) (never executed)
                                        Recheck Cond: ((lower(name) ~~ 'python313packages.%'::text) OR (lower(attr_path) ~~ 'python313packages.%'::text))
                                        Filter: ((lower(name) ~~ 'python313packages.%'::text) OR (lower(attr_path) ~~ 'python313packages.%'::text))
                                        ->  BitmapOr  (cost=704.00..704.00 rows=20268 width=0) (never executed)
                                              ->  Bitmap Index Scan on search_terms_name_lower_idx  (cost=0.00..345.75 rows=10133 width=0) (never executed)
                                                    Index Cond: ((lower(name) >= 'python313packages.'::text) AND (lower(name) < 'python313packages/'::text))
                                              ->  Bitmap Index Scan on search_terms_attr_path_lower_idx  (cost=0.00..345.78 rows=10136 width=0) (never executed)
                                                    Index Cond: ((lower(attr_path) >= 'python313packages.'::text) AND (lower(attr_path) < 'python313packages/'::text))
                            ->  Result  (cost=8.88..16.76 rows=2 width=68) (actual time=0.020..0.022 rows=0 loops=1)
                                  One-Time Filter: (InitPlan 3).col1
                                  Buffers: shared hit=6
                                  InitPlan 3
                                    ->  CTE Scan on breadth breadth_1  (cost=0.00..0.02 rows=1 width=1) (actual time=0.000..0.001 rows=1 loops=1)
                                  ->  Bitmap Heap Scan on search_terms search_terms_4  (cost=8.88..16.76 rows=2 width=68) (actual time=0.018..0.019 rows=0 loops=1)
                                        Recheck Cond: ((lower(name) = 'python313packages.'::text) OR (lower(attr_path) = 'python313packages.'::text))
                                        Buffers: shared hit=6
                                        ->  BitmapOr  (cost=8.86..8.86 rows=2 width=0) (actual time=0.014..0.015 rows=0 loops=1)
                                              Buffers: shared hit=6
                                              ->  Bitmap Index Scan on search_terms_name_lower_idx  (cost=0.00..4.43 rows=1 width=0) (actual time=0.008..0.009 rows=0 loops=1)
                                                    Index Cond: (lower(name) = 'python313packages.'::text)
                                                    Buffers: shared hit=3
                                              ->  Bitmap Index Scan on search_terms_attr_path_lower_idx  (cost=0.00..4.43 rows=1 width=0) (actual time=0.005..0.005 rows=0 loops=1)
                                                    Index Cond: (lower(attr_path) = 'python313packages.'::text)
                                                    Buffers: shared hit=3
                            ->  Result  (cost=0.30..49.71 rows=68 width=68) (actual time=0.380..0.381 rows=0 loops=1)
                                  One-Time Filter: (InitPlan 4).col1
                                  Buffers: shared hit=160
                                  InitPlan 4
                                    ->  CTE Scan on breadth breadth_2  (cost=0.00..0.02 rows=1 width=1) (actual time=0.000..0.000 rows=1 loops=1)
                                  ->  Index Scan using search_terms_alias_idx on search_terms search_terms_5  (cost=0.30..49.71 rows=68 width=68) (actual time=0.378..0.379 rows=0 loops=1)
                                        Filter: ((lower(name) ~~ 'python313packages.%'::text) OR (lower(attr_path) ~~ 'python313packages.%'::text))
                                        Rows Removed by Filter: 598
                                        Buffers: shared hit=160
                            ->  Subquery Scan on "*SELECT* 4"  (cost=3.56..168.67 rows=50 width=68) (actual time=0.302..0.304 rows=0 loops=1)
                                  Buffers: shared hit=47
                                  ->  Limit  (cost=3.56..168.17 rows=50 width=72) (actual time=0.302..0.303 rows=0 loops=1)
                                        Buffers: shared hit=47
                                        InitPlan 5
                                          ->  CTE Scan on breadth breadth_3  (cost=0.00..0.02 rows=1 width=1) (actual time=0.001..0.001 rows=1 loops=1)
                                        ->  Incremental Sort  (cost=3.54..5063.87 rows=1537 width=72) (actual time=0.301..0.302 rows=0 loops=1)
                                              Sort Key: ((lower(search_terms_6.name) <-> 'python313packages.'::text)), search_terms_6.name
                                              Presorted Key: ((lower(search_terms_6.name) <-> 'python313packages.'::text))
                                              Full-sort Groups: 1  Sort Method: quicksort  Average Memory: 25kB  Peak Memory: 25kB
                                              Buffers: shared hit=47
                                              ->  Result  (cost=0.28..4994.70 rows=1537 width=72) (actual time=0.288..0.289 rows=0 loops=1)
                                                    One-Time Filter: (InitPlan 5).col1
                                                    Buffers: shared hit=47
                                                    ->  Index Scan using search_terms_top_level_name_knn_idx on search_terms search_terms_6  (cost=0.28..4987.02 rows=1537 width=68) (actual time=0.286..0.286 rows=0 loops=1)
                                                          Index Cond: (lower(name) ~~ 'python313packages.%'::text)
                                                          Order By: (lower(name) <-> 'python313packages.'::text)
                                                          Buffers: shared hit=47
                            ->  Subquery Scan on "*SELECT* 5"  (cost=1.48..62.35 rows=50 width=68) (actual time=11.553..11.568 rows=50 loops=1)
                                  Buffers: shared hit=2079
                                  ->  Limit  (cost=1.48..61.85 rows=50 width=72) (actual time=11.552..11.560 rows=50 loops=1)
                                        Buffers: shared hit=2079
                                        InitPlan 6
                                          ->  CTE Scan on breadth breadth_4  (cost=0.00..0.02 rows=1 width=1) (actual time=0.000..0.001 rows=1 loops=1)
                                        ->  Incremental Sort  (cost=1.46..13549.68 rows=11221 width=72) (actual time=11.551..11.554 rows=50 loops=1)
                                              Sort Key: ((lower(search_terms_7.name) <-> 'python313packages.'::text)), search_terms_7.name
                                              Presorted Key: ((lower(search_terms_7.name) <-> 'python313packages.'::text))
                                              Full-sort Groups: 1  Sort Method: quicksort  Average Memory: 32kB  Peak Memory: 32kB
                                              Buffers: shared hit=2079
                                              ->  Result  (cost=0.28..13044.81 rows=11221 width=72) (actual time=11.272..11.520 rows=61 loops=1)
                                                    One-Time Filter: (InitPlan 6).col1
                                                    Buffers: shared hit=2079
                                                    ->  Index Scan using search_terms_nested_name_knn_idx on search_terms search_terms_7  (cost=0.28..12988.70 rows=11221 width=68) (actual time=11.260..11.363 rows=61 loops=1)
                                                          Index Cond: (lower(name) ~~ 'python313packages.%'::text)
                                                          Order By: (lower(name) <-> 'python313packages.'::text)
                                                          Buffers: shared hit=2079
  ->  Sort  (cost=32098.09..32098.34 rows=100 width=44) (actual time=26.416..28.693 rows=50 loops=1)
        Sort Key: (max(prefix.rank)) DESC, (min(prefix.name))
        Sort Method: quicksort  Memory: 28kB
        Buffers: shared hit=3018
        ->  GroupAggregate  (cost=32092.77..32094.77 rows=100 width=44) (actual time=26.379..28.673 rows=50 loops=1)
              Group Key: prefix.package_id
              Buffers: shared hit=3018
              ->  Sort  (cost=32092.77..32093.02 rows=100 width=42) (actual time=26.372..28.649 rows=50 loops=1)
                    Sort Key: prefix.package_id
                    Sort Method: quicksort  Memory: 27kB
                    Buffers: shared hit=3018
                    ->  Append  (cost=0.00..32089.44 rows=100 width=42) (actual time=21.846..28.636 rows=50 loops=1)
                          Buffers: shared hit=3018
                          ->  CTE Scan on prefix  (cost=0.00..1.00 rows=50 width=44) (actual time=21.846..21.860 rows=50 loops=1)
                                Buffers: shared hit=2974
                          ->  Subquery Scan on fuzzy  (cost=32087.32..32087.94 rows=50 width=39) (actual time=4.496..6.770 rows=0 loops=1)
                                Buffers: shared hit=44
                                ->  Limit  (cost=32087.32..32087.44 rows=50 width=39) (actual time=4.496..6.769 rows=0 loops=1)
                                      Buffers: shared hit=44
                                      InitPlan 8
                                        ->  Aggregate  (cost=1.12..1.14 rows=1 width=8) (actual time=0.009..0.009 rows=1 loops=1)
                                              ->  CTE Scan on prefix prefix_1  (cost=0.00..1.00 rows=50 width=0) (actual time=0.000..0.004 rows=50 loops=1)
                                      ->  Sort  (cost=32086.18..32358.77 rows=109035 width=39) (actual time=4.495..6.767 rows=0 loops=1)
                                            Sort Key: (max((((CASE WHEN (lower(search_terms.name) = 'python313packages.'::text) THEN 1000 WHEN (lower(search_terms.attr_path) = 'python313packages.'::text) THEN 900 WHEN (lower(search_terms.name) ~~ 'python313packages.%'::text) THEN 800 WHEN (lower(search_terms.attr_path) ~~ 'python313packages.%'::text) THEN 700 ELSE 0 END)::double precision + ('100'::double precision * CASE WHEN (search_terms.name = search_terms.attr_path) THEN similarity(search_terms.name, 'python313Packages.'::text) ELSE GREATEST(similarity(search_terms.name, 'python313Packages.'::text), similarity(search_terms.attr_path, 'python313Packages.'::text)) END)) + (CASE WHEN (search_terms.top_level_attr IS NOT NULL) THEN 25 ELSE 0 END)::double precision))) DESC, search_terms.name
                                            Sort Method: quicksort  Memory: 25kB
                                            Buffers: shared hit=44
                                            ->  Finalize GroupAggregate  (cost=15481.23..28464.12 rows=109035 width=39) (actual time=4.489..6.761 rows=0 loops=1)
                                                  Group Key: search_terms.package_id, search_terms.name
                                                  Buffers: shared hit=44
                                                  ->  Gather Merge  (cost=15481.23..26891.64 rows=64284 width=39) (actual time=4.488..6.759 rows=0 loops=1)
                                                        Workers Planned: 1
                                                        Workers Launched: 1
                                                        Buffers: shared hit=44
                                                        ->  Partial GroupAggregate  (cost=14481.22..18659.68 rows=64284 width=39) (actual time=0.044..0.044 rows=0 loops=2)
                                                              Group Key: search_terms.package_id, search_terms.name
                                                              Buffers: shared hit=44
                                                              ->  Sort  (cost=14481.22..14641.93 rows=64284 width=68) (actual time=0.043..0.043 rows=0 loops=2)
                                                                    Sort Key: search_terms.package_id, search_terms.name
                                                                    Sort Method: quicksort  Memory: 25kB
                                                                    Buffers: shared hit=44
                                                                    Worker 0:  Sort Method: quicksort  Memory: 25kB
                                                                    ->  Result  (cost=0.00..6708.44 rows=64284 width=68) (actual time=0.001..0.001 rows=0 loops=2)
                                                                          One-Time Filter: ((InitPlan 8).col1 < 50)
                                                                          ->  Parallel Seq Scan on search_terms  (cost=0.00..6708.44 rows=64284 width=68) (never executed)
                                                                                Filter: (((name % 'python313Packages.'::text) OR (attr_path % 'python313Packages.'::text)) AND (lower(name) !~~ 'python313packages.%'::text) AND (lower(attr_path) !~~ 'python313packages.%'::text))
Planning:
  Buffers: shared hit=4
Planning Time: 1.418 ms
Execution Time: 28.890 ms
```

### batched fetch, latest — q=python313Packages. (50 hits)

Parameters: `["150986,152209,150834,152035,152503,152537,152607,144825,145013,147350,147764,147765,147997,148046,148297,148334,150150,150536,150586,150612,150629,150655,150684,150771,150780,150816,150824,274938,150979,151062,151063,151070,151177,151212,151252,151267,151284,151298,151520,151586,151651,151873,272730,152489,152502,152504,152533,152553,152573,152630"]` — Execution Time: 5.453 ms

```
Limit  (cost=4241.35..107618.59 rows=50 width=1378) (actual time=1.178..5.339 rows=50 loops=1)
  Buffers: shared hit=8221
  ->  Unique  (cost=4241.35..107618.59 rows=50 width=1378) (actual time=1.177..5.333 rows=50 loops=1)
        Buffers: shared hit=8221
        ->  Incremental Sort  (cost=4241.35..107616.83 rows=706 width=1378) (actual time=1.176..5.296 rows=173 loops=1)
              Sort Key: hits.ord, variants.system, variants.attr_path
              Presorted Key: hits.ord
              Full-sort Groups: 6  Sort Method: quicksort  Average Memory: 61kB  Peak Memory: 61kB
              Buffers: shared hit=8221
              ->  Nested Loop  (cost=2131.84..107593.52 rows=706 width=1378) (actual time=0.110..5.143 rows=175 loops=1)
                    Buffers: shared hit=8221
                    ->  Nested Loop  (cost=2131.56..107383.45 rows=706 width=1323) (actual time=0.105..4.896 rows=175 loops=1)
                          Buffers: shared hit=7696
                          ->  Nested Loop  (cost=2131.14..107018.11 rows=706 width=340) (actual time=0.098..4.394 rows=175 loops=1)
                                Join Filter: (variants.version_id = versions_1.id)
                                Buffers: shared hit=6996
                                ->  Nested Loop  (cost=2130.71..106939.17 rows=50 width=53) (actual time=0.096..4.190 rows=50 loops=1)
                                      Buffers: shared hit=6691
                                      ->  Nested Loop  (cost=2130.29..106916.94 rows=50 width=30) (actual time=0.089..4.058 rows=50 loops=1)
                                            Buffers: shared hit=6491
                                            ->  Nested Loop  (cost=2129.86..106494.69 rows=50 width=12) (actual time=0.083..3.917 rows=50 loops=1)
                                                  Buffers: shared hit=6291
                                                  ->  Function Scan on unnest hits  (cost=0.00..0.50 rows=50 width=12) (actual time=0.009..0.015 rows=50 loops=1)
                                                  ->  Limit  (cost=2129.86..2129.86 rows=1 width=24) (actual time=0.078..0.078 rows=1 loops=50)
                                                        Buffers: shared hit=6291
                                                        ->  Sort  (cost=2129.86..2130.06 rows=79 width=24) (actual time=0.077..0.077 rows=1 loops=50)
                                                              Sort Key: (EXISTS(SubPlan 1)) DESC, ((SubPlan 2)) DESC, versions_1.sort_key DESC
                                                              Sort Method: top-N heapsort  Memory: 25kB
                                                              Buffers: shared hit=6291
                                                              ->  Nested Loop  (cost=0.86..2129.47 rows=79 width=24) (actual time=0.014..0.074 rows=13 loops=50)
                                                                    Buffers: shared hit=6291
                                                                    ->  Index Scan using versions_latest_idx on versions versions_1  (cost=0.43..66.26 rows=32 width=19) (actual time=0.003..0.004 rows=3 loops=50)
                                                                          Index Cond: ((package_id = hits.package_id) AND (prerelease = false))
                                                                          Buffers: shared hit=217
                                                                    ->  Index Scan using variants_identity_key on variants variants_1  (cost=0.43..36.09 rows=14 width=8) (actual time=0.002..0.004 rows=4 loops=173)
                                                                          Index Cond: (version_id = versions_1.id)
                                                                          Buffers: shared hit=847
                                                                    SubPlan 1
                                                                      ->  Index Scan using variants_identity_key on variants b  (cost=0.43..36.09 rows=14 width=0) (actual time=0.001..0.001 rows=1 loops=637)
                                                                            Index Cond: (version_id = versions_1.id)
                                                                            Filter: (NOT broken)
                                                                            Buffers: shared hit=2548
                                                                    SubPlan 2
                                                                      ->  Aggregate  (cost=8.45..8.46 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=637)
                                                                            Buffers: shared hit=2679
                                                                            ->  Index Scan using variant_ranges_variant_id_first_seq_pk on variant_ranges r  (cost=0.43..8.45 rows=1 width=4) (actual time=0.002..0.002 rows=0 loops=637)
                                                                                  Index Cond: (variant_id = variants_1.id)
                                                                                  Filter: (NOT seeded)
                                                                                  Rows Removed by Filter: 1
                                                                                  Buffers: shared hit=2679
                                            ->  Index Scan using versions_pkey on versions  (cost=0.43..8.45 rows=1 width=18) (actual time=0.002..0.002 rows=1 loops=50)
                                                  Index Cond: (id = versions_1.id)
                                                  Buffers: shared hit=200
                                      ->  Index Scan using packages_pkey on packages  (cost=0.42..0.44 rows=1 width=31) (actual time=0.002..0.002 rows=1 loops=50)
                                            Index Cond: (id = versions.package_id)
                                            Buffers: shared hit=200
                                ->  Index Scan using variants_identity_key on variants  (cost=0.43..1.40 rows=14 width=299) (actual time=0.001..0.003 rows=4 loops=50)
                                      Index Cond: (version_id = versions.id)
                                      Buffers: shared hit=305
                          ->  Index Scan using meta_pkey on meta  (cost=0.42..0.52 rows=1 width=991) (actual time=0.002..0.002 rows=1 loops=175)
                                Index Cond: (id = variants.meta_id)
                                Buffers: shared hit=700
                    ->  Index Scan using commits_pkey on commits commit_hash  (cost=0.28..0.30 rows=1 width=53) (actual time=0.001..0.001 rows=1 loops=175)
                          Index Cond: (seq = variants.commit_seq)
                          Buffers: shared hit=525
Planning:
  Buffers: shared hit=115
Planning Time: 68.699 ms
Execution Time: 5.453 ms
```

### batched fetch, all versions — q=python313Packages. (50 hits)

Parameters: `["150986,152209,150834,152035,152503,152537,152607,144825,145013,147350,147764,147765,147997,148046,148297,148334,150150,150536,150586,150612,150629,150655,150684,150771,150780,150816,150824,274938,150979,151062,151063,151070,151177,151212,151252,151267,151284,151298,151520,151586,151651,151873,272730,152489,152502,152504,152533,152553,152573,152630"]` — Execution Time: 4.306 ms

```
Limit  (cost=119.91..1568.91 rows=1000 width=1393) (actual time=0.547..4.177 rows=173 loops=1)
  Buffers: shared hit=4003
  ->  Unique  (cost=119.91..5979.69 rows=4044 width=1393) (actual time=0.546..4.161 rows=173 loops=1)
        Buffers: shared hit=4003
        ->  Incremental Sort  (cost=119.91..5949.36 rows=4044 width=1393) (actual time=0.545..4.012 rows=637 loops=1)
              Sort Key: hits.ord, versions.sort_key DESC, versions.version, variants.system, variants.attr_path
              Presorted Key: hits.ord
              Full-sort Groups: 14  Sort Method: quicksort  Average Memory: 105kB  Peak Memory: 105kB
              Pre-sorted Groups: 2  Sort Method: quicksort  Average Memory: 65kB  Peak Memory: 101kB
              Buffers: shared hit=4003
              ->  Nested Loop  (cost=1.99..5769.66 rows=4044 width=1393) (actual time=0.055..2.800 rows=637 loops=1)
                    Buffers: shared hit=4003
                    ->  Nested Loop  (cost=1.70..5121.54 rows=4044 width=1338) (actual time=0.047..2.412 rows=637 loops=1)
                          Buffers: shared hit=3814
                          ->  Nested Loop  (cost=1.28..3028.84 rows=4044 width=355) (actual time=0.039..1.287 rows=637 loops=1)
                                Buffers: shared hit=1266
                                ->  Nested Loop  (cost=0.85..507.87 rows=1633 width=64) (actual time=0.032..0.398 rows=173 loops=1)
                                      Join Filter: (versions.package_id = hits.package_id)
                                      Buffers: shared hit=419
                                      ->  Nested Loop  (cost=0.42..418.38 rows=50 width=43) (actual time=0.023..0.155 rows=50 loops=1)
                                            Buffers: shared hit=200
                                            ->  Function Scan on unnest hits  (cost=0.00..0.50 rows=50 width=12) (actual time=0.008..0.014 rows=50 loops=1)
                                            ->  Index Scan using packages_pkey on packages  (cost=0.42..8.36 rows=1 width=31) (actual time=0.002..0.002 rows=1 loops=50)
                                                  Index Cond: (id = hits.package_id)
                                                  Buffers: shared hit=200
                                      ->  Index Scan using versions_package_version_key on versions  (cost=0.43..1.38 rows=33 width=33) (actual time=0.003..0.004 rows=3 loops=50)
                                            Index Cond: (package_id = packages.id)
                                            Buffers: shared hit=219
                                ->  Index Scan using variants_identity_key on variants  (cost=0.43..1.40 rows=14 width=299) (actual time=0.002..0.004 rows=4 loops=173)
                                      Index Cond: (version_id = versions.id)
                                      Buffers: shared hit=847
                          ->  Index Scan using meta_pkey on meta  (cost=0.42..0.52 rows=1 width=991) (actual time=0.001..0.001 rows=1 loops=637)
                                Index Cond: (id = variants.meta_id)
                                Buffers: shared hit=2548
                    ->  Memoize  (cost=0.29..0.31 rows=1 width=53) (actual time=0.000..0.000 rows=1 loops=637)
                          Cache Key: variants.commit_seq
                          Cache Mode: logical
                          Hits: 574  Misses: 63  Evictions: 0  Overflows: 0  Memory Usage: 10kB
                          Buffers: shared hit=189
                          ->  Index Scan using commits_pkey on commits commit_hash  (cost=0.28..0.30 rows=1 width=53) (actual time=0.001..0.001 rows=1 loops=63)
                                Index Cond: (seq = variants.commit_seq)
                                Buffers: shared hit=189
Planning:
  Buffers: shared hit=69
Planning Time: 10.220 ms
Execution Time: 4.306 ms
```

### ranked terms — q=hello

Parameters: `["hello","hello"]` — Execution Time: 147.360 ms

```
Limit  (cost=2204.03..2204.16 rows=50 width=44) (actual time=147.134..147.156 rows=15 loops=1)
  Buffers: shared hit=1538
  CTE breadth
    ->  Aggregate  (cost=8.76..8.77 rows=1 width=1) (actual time=0.027..0.027 rows=1 loops=1)
          Buffers: shared hit=4
          ->  Limit  (cost=0.42..8.45 rows=25 width=4) (actual time=0.019..0.023 rows=5 loops=1)
                Buffers: shared hit=4
                ->  Index Scan using search_terms_name_lower_idx on search_terms search_terms_1  (cost=0.42..8.45 rows=25 width=4) (actual time=0.018..0.022 rows=5 loops=1)
                      Index Cond: ((lower(name) >= 'hello'::text) AND (lower(name) < 'hellp'::text))
                      Filter: (lower(name) ~~ 'hello%'::text)
                      Buffers: shared hit=4
  CTE prefix
    ->  Limit  (cost=68.30..68.42 rows=50 width=39) (actual time=0.079..0.094 rows=5 loops=1)
          Buffers: shared hit=11
          ->  Sort  (cost=68.30..68.50 rows=79 width=39) (actual time=0.079..0.093 rows=5 loops=1)
                Sort Key: (max((((CASE WHEN (lower(search_terms_3.name) = 'hello'::text) THEN 1000 WHEN (lower(search_terms_3.attr_path) = 'hello'::text) THEN 900 WHEN (lower(search_terms_3.name) ~~ 'hello%'::text) THEN 800 WHEN (lower(search_terms_3.attr_path) ~~ 'hello%'::text) THEN 700 ELSE 0 END)::double precision + ('100'::double precision * CASE WHEN (search_terms_3.name = search_terms_3.attr_path) THEN similarity(search_terms_3.name, 'hello'::text) ELSE GREATEST(similarity(search_terms_3.name, 'hello'::text), similarity(search_terms_3.attr_path, 'hello'::text)) END)) + (CASE WHEN (search_terms_3.top_level_attr IS NOT NULL) THEN 25 ELSE 0 END)::double precision))) DESC, search_terms_3.name
                Sort Method: quicksort  Memory: 25kB
                Buffers: shared hit=11
                ->  HashAggregate  (cost=65.02..65.81 rows=79 width=39) (actual time=0.074..0.089 rows=5 loops=1)
                      Group Key: search_terms_3.package_id, search_terms_3.name
                      Batches: 1  Memory Usage: 24kB
                      Buffers: shared hit=11
                      ->  Append  (cost=8.91..60.87 rows=79 width=68) (actual time=0.044..0.067 rows=5 loops=1)
                            Buffers: shared hit=11
                            ->  Result  (cost=8.91..12.93 rows=51 width=68) (actual time=0.044..0.050 rows=5 loops=1)
                                  One-Time Filter: (NOT (InitPlan 2).col1)
                                  Buffers: shared hit=11
                                  InitPlan 2
                                    ->  CTE Scan on breadth  (cost=0.00..0.02 rows=1 width=1) (actual time=0.027..0.028 rows=1 loops=1)
                                          Buffers: shared hit=4
                                  ->  Bitmap Heap Scan on search_terms search_terms_3  (cost=8.91..12.93 rows=51 width=68) (actual time=0.014..0.018 rows=5 loops=1)
                                        Recheck Cond: ((lower(name) ~~ 'hello%'::text) OR (lower(attr_path) ~~ 'hello%'::text))
                                        Filter: ((lower(name) ~~ 'hello%'::text) OR (lower(attr_path) ~~ 'hello%'::text))
                                        Heap Blocks: exact=1
                                        Buffers: shared hit=7
                                        ->  BitmapOr  (cost=8.89..8.89 rows=1 width=0) (actual time=0.009..0.011 rows=0 loops=1)
                                              Buffers: shared hit=6
                                              ->  Bitmap Index Scan on search_terms_name_lower_idx  (cost=0.00..4.43 rows=1 width=0) (actual time=0.003..0.003 rows=5 loops=1)
                                                    Index Cond: ((lower(name) >= 'hello'::text) AND (lower(name) < 'hellp'::text))
                                                    Buffers: shared hit=3
                                              ->  Bitmap Index Scan on search_terms_attr_path_lower_idx  (cost=0.00..4.43 rows=1 width=0) (actual time=0.006..0.006 rows=5 loops=1)
                                                    Index Cond: ((lower(attr_path) >= 'hello'::text) AND (lower(attr_path) < 'hellp'::text))
                                                    Buffers: shared hit=3
                            ->  Result  (cost=8.88..16.76 rows=2 width=68) (actual time=0.001..0.003 rows=0 loops=1)
                                  One-Time Filter: (InitPlan 3).col1
                                  InitPlan 3
                                    ->  CTE Scan on breadth breadth_1  (cost=0.00..0.02 rows=1 width=1) (actual time=0.000..0.001 rows=1 loops=1)
                                  ->  Bitmap Heap Scan on search_terms search_terms_4  (cost=8.88..16.76 rows=2 width=68) (never executed)
                                        Recheck Cond: ((lower(name) = 'hello'::text) OR (lower(attr_path) = 'hello'::text))
                                        ->  BitmapOr  (cost=8.86..8.86 rows=2 width=0) (never executed)
                                              ->  Bitmap Index Scan on search_terms_name_lower_idx  (cost=0.00..4.43 rows=1 width=0) (never executed)
                                                    Index Cond: (lower(name) = 'hello'::text)
                                              ->  Bitmap Index Scan on search_terms_attr_path_lower_idx  (cost=0.00..4.43 rows=1 width=0) (never executed)
                                                    Index Cond: (lower(attr_path) = 'hello'::text)
                            ->  Result  (cost=8.88..12.90 rows=1 width=68) (actual time=0.001..0.002 rows=0 loops=1)
                                  One-Time Filter: (InitPlan 4).col1
                                  InitPlan 4
                                    ->  CTE Scan on breadth breadth_2  (cost=0.00..0.02 rows=1 width=1) (actual time=0.000..0.000 rows=1 loops=1)
                                  ->  Bitmap Heap Scan on search_terms search_terms_5  (cost=8.88..12.90 rows=1 width=68) (never executed)
                                        Recheck Cond: ((lower(name) ~~ 'hello%'::text) OR (lower(attr_path) ~~ 'hello%'::text))
                                        Filter: (((name = attr_path) IS FALSE) AND ((lower(name) ~~ 'hello%'::text) OR (lower(attr_path) ~~ 'hello%'::text)))
                                        ->  BitmapOr  (cost=8.86..8.86 rows=1 width=0) (never executed)
                                              ->  Bitmap Index Scan on search_terms_name_lower_idx  (cost=0.00..4.43 rows=1 width=0) (never executed)
                                                    Index Cond: ((lower(name) >= 'hello'::text) AND (lower(name) < 'hellp'::text))
                                              ->  Bitmap Index Scan on search_terms_attr_path_lower_idx  (cost=0.00..4.43 rows=1 width=0) (never executed)
                                                    Index Cond: ((lower(attr_path) >= 'hello'::text) AND (lower(attr_path) < 'hellp'::text))
                            ->  Subquery Scan on "*SELECT* 4"  (cost=8.51..8.54 rows=3 width=68) (actual time=0.004..0.005 rows=0 loops=1)
                                  ->  Limit  (cost=8.51..8.51 rows=3 width=72) (actual time=0.003..0.005 rows=0 loops=1)
                                        InitPlan 5
                                          ->  CTE Scan on breadth breadth_3  (cost=0.00..0.02 rows=1 width=1) (actual time=0.000..0.000 rows=1 loops=1)
                                        ->  Sort  (cost=8.49..8.49 rows=3 width=72) (actual time=0.003..0.003 rows=0 loops=1)
                                              Sort Key: ((lower(search_terms_6.name) <-> 'hello'::text)), search_terms_6.name
                                              Sort Method: quicksort  Memory: 25kB
                                              ->  Result  (cost=0.42..8.46 rows=3 width=72) (actual time=0.001..0.001 rows=0 loops=1)
                                                    One-Time Filter: (InitPlan 5).col1
                                                    ->  Index Scan using search_terms_name_lower_idx on search_terms search_terms_6  (cost=0.42..8.45 rows=3 width=68) (never executed)
                                                          Index Cond: ((lower(name) >= 'hello'::text) AND (lower(name) < 'hellp'::text))
                                                          Filter: ((top_level_attr IS NOT NULL) AND ((name = attr_path) IS TRUE) AND (lower(name) ~~ 'hello%'::text))
                            ->  Subquery Scan on "*SELECT* 5"  (cost=9.07..9.34 rows=22 width=68) (actual time=0.002..0.003 rows=0 loops=1)
                                  ->  Limit  (cost=9.07..9.12 rows=22 width=72) (actual time=0.002..0.003 rows=0 loops=1)
                                        InitPlan 6
                                          ->  CTE Scan on breadth breadth_4  (cost=0.00..0.02 rows=1 width=1) (actual time=0.000..0.000 rows=1 loops=1)
                                        ->  Sort  (cost=9.05..9.10 rows=22 width=72) (actual time=0.002..0.002 rows=0 loops=1)
                                              Sort Key: ((lower(search_terms_7.name) <-> 'hello'::text)), search_terms_7.name
                                              Sort Method: quicksort  Memory: 25kB
                                              ->  Result  (cost=0.42..8.56 rows=22 width=72) (actual time=0.001..0.001 rows=0 loops=1)
                                                    One-Time Filter: (InitPlan 6).col1
                                                    ->  Index Scan using search_terms_name_lower_idx on search_terms search_terms_7  (cost=0.42..8.45 rows=22 width=68) (never executed)
                                                          Index Cond: ((lower(name) >= 'hello'::text) AND (lower(name) < 'hellp'::text))
                                                          Filter: ((top_level_attr IS NULL) AND ((name = attr_path) IS TRUE) AND (lower(name) ~~ 'hello%'::text))
  ->  Sort  (cost=2126.84..2127.09 rows=100 width=44) (actual time=147.133..147.138 rows=15 loops=1)
        Sort Key: (max(prefix.rank)) DESC, (min(prefix.name))
        Sort Method: quicksort  Memory: 26kB
        Buffers: shared hit=1538
        ->  GroupAggregate  (cost=2121.52..2123.52 rows=100 width=44) (actual time=147.119..147.129 rows=15 loops=1)
              Group Key: prefix.package_id
              Buffers: shared hit=1538
              ->  Sort  (cost=2121.52..2121.77 rows=100 width=42) (actual time=147.115..147.120 rows=15 loops=1)
                    Sort Key: prefix.package_id
                    Sort Method: quicksort  Memory: 25kB
                    Buffers: shared hit=1538
                    ->  Append  (cost=0.00..2118.20 rows=100 width=42) (actual time=0.081..147.113 rows=15 loops=1)
                          Buffers: shared hit=1538
                          ->  CTE Scan on prefix  (cost=0.00..1.00 rows=50 width=44) (actual time=0.081..0.082 rows=5 loops=1)
                                Buffers: shared hit=11
                          ->  Subquery Scan on fuzzy  (cost=2116.07..2116.70 rows=50 width=39) (actual time=147.021..147.027 rows=10 loops=1)
                                Buffers: shared hit=1527
                                ->  Limit  (cost=2116.07..2116.20 rows=50 width=39) (actual time=147.020..147.024 rows=10 loops=1)
                                      Buffers: shared hit=1527
                                      InitPlan 8
                                        ->  Aggregate  (cost=1.12..1.14 rows=1 width=8) (actual time=0.002..0.002 rows=1 loops=1)
                                              ->  CTE Scan on prefix prefix_1  (cost=0.00..1.00 rows=50 width=0) (actual time=0.000..0.001 rows=5 loops=1)
                                      ->  Sort  (cost=2114.94..2115.06 rows=51 width=39) (actual time=147.019..147.022 rows=10 loops=1)
                                            Sort Key: (max((((CASE WHEN (lower(search_terms.name) = 'hello'::text) THEN 1000 WHEN (lower(search_terms.attr_path) = 'hello'::text) THEN 900 WHEN (lower(search_terms.name) ~~ 'hello%'::text) THEN 800 WHEN (lower(search_terms.attr_path) ~~ 'hello%'::text) THEN 700 ELSE 0 END)::double precision + ('100'::double precision * CASE WHEN (search_terms.name = search_terms.attr_path) THEN similarity(search_terms.name, 'hello'::text) ELSE GREATEST(similarity(search_terms.name, 'hello'::text), similarity(search_terms.attr_path, 'hello'::text)) END)) + (CASE WHEN (search_terms.top_level_attr IS NOT NULL) THEN 25 ELSE 0 END)::double precision))) DESC, search_terms.name
                                            Sort Method: quicksort  Memory: 25kB
                                            Buffers: shared hit=1527
                                            ->  GroupAggregate  (cost=2110.17..2113.49 rows=51 width=39) (actual time=146.994..147.015 rows=10 loops=1)
                                                  Group Key: search_terms.package_id, search_terms.name
                                                  Buffers: shared hit=1527
                                                  ->  Sort  (cost=2110.17..2110.30 rows=51 width=68) (actual time=146.980..146.982 rows=10 loops=1)
                                                        Sort Key: search_terms.package_id, search_terms.name
                                                        Sort Method: quicksort  Memory: 25kB
                                                        Buffers: shared hit=1527
                                                        ->  Result  (cost=1923.46..2108.73 rows=51 width=68) (actual time=71.044..146.969 rows=10 loops=1)
                                                              One-Time Filter: ((InitPlan 8).col1 < 50)
                                                              Buffers: shared hit=1527
                                                              ->  Bitmap Heap Scan on search_terms  (cost=1923.46..2108.73 rows=51 width=68) (actual time=71.040..146.960 rows=10 loops=1)
                                                                    Recheck Cond: ((name % 'hello'::text) OR (attr_path % 'hello'::text))
                                                                    Rows Removed by Index Recheck: 17553
                                                                    Filter: ((lower(name) !~~ 'hello%'::text) AND (lower(attr_path) !~~ 'hello%'::text))
                                                                    Rows Removed by Filter: 5
                                                                    Heap Blocks: exact=1049
                                                                    Buffers: shared hit=1527
                                                                    ->  BitmapOr  (cost=1923.46..1923.46 rows=51 width=0) (actual time=7.703..7.704 rows=0 loops=1)
                                                                          Buffers: shared hit=478
                                                                          ->  Bitmap Index Scan on search_terms_name_trgm_idx  (cost=0.00..961.72 rows=25 width=0) (actual time=3.752..3.752 rows=17568 loops=1)
                                                                                Index Cond: (name % 'hello'::text)
                                                                                Buffers: shared hit=239
                                                                          ->  Bitmap Index Scan on search_terms_attr_path_trgm_idx  (cost=0.00..961.72 rows=25 width=0) (actual time=3.950..3.950 rows=17568 loops=1)
                                                                                Index Cond: (attr_path % 'hello'::text)
                                                                                Buffers: shared hit=239
Planning:
  Buffers: shared hit=4
Planning Time: 1.328 ms
Execution Time: 147.360 ms
```

### batched fetch, latest — q=hello (15 hits)

Parameters: `["51324,51326,51325,51327,51328,87526,51329,51330,51319,51323,54432,275101,51341,242138,42681"]` — Execution Time: 1.959 ms

```
Limit  (cost=4141.87..32285.75 rows=15 width=1378) (actual time=1.404..1.855 rows=15 loops=1)
  Buffers: shared hit=2839
  ->  Unique  (cost=4141.87..32285.75 rows=15 width=1378) (actual time=1.403..1.852 rows=15 loops=1)
        Buffers: shared hit=2839
        ->  Incremental Sort  (cost=4141.87..32285.22 rows=212 width=1378) (actual time=1.402..1.838 rows=47 loops=1)
              Sort Key: hits.ord, variants.system, variants.attr_path
              Presorted Key: hits.ord
              Full-sort Groups: 2  Sort Method: quicksort  Average Memory: 64kB  Peak Memory: 64kB
              Buffers: shared hit=2839
              ->  Nested Loop  (cost=2131.84..32278.22 rows=212 width=1378) (actual time=0.177..1.779 rows=47 loops=1)
                    Buffers: shared hit=2839
                    ->  Nested Loop  (cost=2131.56..32215.14 rows=212 width=1323) (actual time=0.172..1.708 rows=47 loops=1)
                          Buffers: shared hit=2698
                          ->  Nested Loop  (cost=2131.14..32105.43 rows=212 width=340) (actual time=0.164..1.601 rows=47 loops=1)
                                Join Filter: (variants.version_id = versions_1.id)
                                Buffers: shared hit=2510
                                ->  Nested Loop  (cost=2130.71..32081.75 rows=15 width=53) (actual time=0.161..1.545 rows=15 loops=1)
                                      Buffers: shared hit=2424
                                      ->  Nested Loop  (cost=2130.29..32075.08 rows=15 width=30) (actual time=0.156..1.506 rows=15 loops=1)
                                            Buffers: shared hit=2364
                                            ->  Nested Loop  (cost=2129.86..31948.41 rows=15 width=12) (actual time=0.150..1.396 rows=15 loops=1)
                                                  Buffers: shared hit=2304
                                                  ->  Function Scan on unnest hits  (cost=0.00..0.15 rows=15 width=12) (actual time=0.007..0.009 rows=15 loops=1)
                                                  ->  Limit  (cost=2129.86..2129.86 rows=1 width=24) (actual time=0.092..0.092 rows=1 loops=15)
                                                        Buffers: shared hit=2304
                                                        ->  Sort  (cost=2129.86..2130.06 rows=79 width=24) (actual time=0.092..0.092 rows=1 loops=15)
                                                              Sort Key: (EXISTS(SubPlan 1)) DESC, ((SubPlan 2)) DESC, versions_1.sort_key DESC
                                                              Sort Method: top-N heapsort  Memory: 25kB
                                                              Buffers: shared hit=2304
                                                              ->  Nested Loop  (cost=0.86..2129.47 rows=79 width=24) (actual time=0.015..0.087 rows=15 loops=15)
                                                                    Buffers: shared hit=2304
                                                                    ->  Index Scan using versions_latest_idx on versions versions_1  (cost=0.43..66.26 rows=32 width=19) (actual time=0.004..0.005 rows=5 loops=15)
                                                                          Index Cond: ((package_id = hits.package_id) AND (prerelease = false))
                                                                          Buffers: shared hit=64
                                                                    ->  Index Scan using variants_identity_key on variants variants_1  (cost=0.43..36.09 rows=14 width=8) (actual time=0.002..0.003 rows=3 loops=80)
                                                                          Index Cond: (version_id = versions_1.id)
                                                                          Buffers: shared hit=390
                                                                    SubPlan 1
                                                                      ->  Index Scan using variants_identity_key on variants b  (cost=0.43..36.09 rows=14 width=0) (actual time=0.001..0.001 rows=1 loops=227)
                                                                            Index Cond: (version_id = versions_1.id)
                                                                            Filter: (NOT broken)
                                                                            Buffers: shared hit=908
                                                                    SubPlan 2
                                                                      ->  Aggregate  (cost=8.45..8.46 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=227)
                                                                            Buffers: shared hit=942
                                                                            ->  Index Scan using variant_ranges_variant_id_first_seq_pk on variant_ranges r  (cost=0.43..8.45 rows=1 width=4) (actual time=0.002..0.002 rows=0 loops=227)
                                                                                  Index Cond: (variant_id = variants_1.id)
                                                                                  Filter: (NOT seeded)
                                                                                  Rows Removed by Filter: 1
                                                                                  Buffers: shared hit=942
                                            ->  Index Scan using versions_pkey on versions  (cost=0.43..8.45 rows=1 width=18) (actual time=0.002..0.002 rows=1 loops=15)
                                                  Index Cond: (id = versions_1.id)
                                                  Buffers: shared hit=60
                                      ->  Index Scan using packages_pkey on packages  (cost=0.42..0.44 rows=1 width=31) (actual time=0.002..0.002 rows=1 loops=15)
                                            Index Cond: (id = versions.package_id)
                                            Buffers: shared hit=60
                                ->  Index Scan using variants_identity_key on variants  (cost=0.43..1.40 rows=14 width=299) (actual time=0.001..0.003 rows=3 loops=15)
                                      Index Cond: (version_id = versions.id)
                                      Buffers: shared hit=86
                          ->  Index Scan using meta_pkey on meta  (cost=0.42..0.52 rows=1 width=991) (actual time=0.002..0.002 rows=1 loops=47)
                                Index Cond: (id = variants.meta_id)
                                Buffers: shared hit=188
                    ->  Index Scan using commits_pkey on commits commit_hash  (cost=0.28..0.30 rows=1 width=53) (actual time=0.001..0.001 rows=1 loops=47)
                          Index Cond: (seq = variants.commit_seq)
                          Buffers: shared hit=141
Planning:
  Buffers: shared hit=115
Planning Time: 27.098 ms
Execution Time: 1.959 ms
```

### batched fetch, all versions — q=hello (15 hits)

Parameters: `["51324,51326,51325,51327,51328,87526,51329,51330,51319,51323,54432,275101,51341,242138,42681"]` — Execution Time: 1.871 ms

```
Limit  (cost=130.99..1640.18 rows=1000 width=1393) (actual time=0.313..1.786 rows=80 loops=1)
  Buffers: shared hit=2103
  ->  Unique  (cost=130.99..1961.63 rows=1213 width=1393) (actual time=0.312..1.777 rows=80 loops=1)
        Buffers: shared hit=2103
        ->  Incremental Sort  (cost=130.99..1952.54 rows=1213 width=1393) (actual time=0.312..1.721 rows=227 loops=1)
              Sort Key: hits.ord, versions.sort_key DESC, versions.version, variants.system, variants.attr_path
              Presorted Key: hits.ord
              Full-sort Groups: 5  Sort Method: quicksort  Average Memory: 88kB  Peak Memory: 88kB
              Pre-sorted Groups: 3  Sort Method: quicksort  Average Memory: 52kB  Peak Memory: 54kB
              Buffers: shared hit=2103
              ->  Nested Loop  (cost=1.98..1898.64 rows=1213 width=1393) (actual time=0.048..1.267 rows=227 loops=1)
                    Buffers: shared hit=2103
                    ->  Nested Loop  (cost=1.70..1537.71 rows=1213 width=1338) (actual time=0.043..0.938 rows=227 loops=1)
                          Buffers: shared hit=1422
                          ->  Nested Loop  (cost=1.28..910.01 rows=1213 width=355) (actual time=0.036..0.560 rows=227 loops=1)
                                Buffers: shared hit=514
                                ->  Nested Loop  (cost=0.85..153.56 rows=490 width=64) (actual time=0.028..0.147 rows=80 loops=1)
                                      Join Filter: (versions.package_id = hits.package_id)
                                      Buffers: shared hit=124
                                      ->  Nested Loop  (cost=0.42..126.72 rows=15 width=43) (actual time=0.019..0.059 rows=15 loops=1)
                                            Buffers: shared hit=60
                                            ->  Function Scan on unnest hits  (cost=0.00..0.15 rows=15 width=12) (actual time=0.007..0.009 rows=15 loops=1)
                                            ->  Index Scan using packages_pkey on packages  (cost=0.42..8.44 rows=1 width=31) (actual time=0.003..0.003 rows=1 loops=15)
                                                  Index Cond: (id = hits.package_id)
                                                  Buffers: shared hit=60
                                      ->  Index Scan using versions_package_version_key on versions  (cost=0.43..1.38 rows=33 width=33) (actual time=0.003..0.005 rows=5 loops=15)
                                            Index Cond: (package_id = packages.id)
                                            Buffers: shared hit=64
                                ->  Index Scan using variants_identity_key on variants  (cost=0.43..1.40 rows=14 width=299) (actual time=0.003..0.004 rows=3 loops=80)
                                      Index Cond: (version_id = versions.id)
                                      Buffers: shared hit=390
                          ->  Index Scan using meta_pkey on meta  (cost=0.42..0.52 rows=1 width=991) (actual time=0.001..0.001 rows=1 loops=227)
                                Index Cond: (id = variants.meta_id)
                                Buffers: shared hit=908
                    ->  Index Scan using commits_pkey on commits commit_hash  (cost=0.28..0.30 rows=1 width=53) (actual time=0.001..0.001 rows=1 loops=227)
                          Index Cond: (seq = variants.commit_seq)
                          Buffers: shared hit=681
Planning:
  Buffers: shared hit=69
Planning Time: 10.049 ms
Execution Time: 1.871 ms
```

### ranked terms — q=-

Parameters: `["-","-"]` — Execution Time: 904.123 ms

```
Limit  (cost=7746.26..7746.39 rows=50 width=44) (actual time=903.761..903.993 rows=0 loops=1)
  Buffers: shared hit=3003
  CTE prefix
    ->  Limit  (cost=17.54..17.66 rows=50 width=39) (actual time=0.018..0.022 rows=0 loops=1)
          Buffers: shared hit=6
          ->  Sort  (cost=17.54..17.67 rows=51 width=39) (actual time=0.018..0.021 rows=0 loops=1)
                Sort Key: (max((((CASE WHEN (lower(search_terms_1.name) = '-'::text) THEN 1000 WHEN (lower(search_terms_1.attr_path) = '-'::text) THEN 900 WHEN (lower(search_terms_1.name) ~~ '-%'::text) THEN 800 WHEN (lower(search_terms_1.attr_path) ~~ '-%'::text) THEN 700 ELSE 0 END)::double precision + ('100'::double precision * CASE WHEN (search_terms_1.name = search_terms_1.attr_path) THEN similarity(search_terms_1.name, '-'::text) ELSE GREATEST(similarity(search_terms_1.name, '-'::text), similarity(search_terms_1.attr_path, '-'::text)) END)) + (CASE WHEN (search_terms_1.top_level_attr IS NOT NULL) THEN 25 ELSE 0 END)::double precision))) DESC, search_terms_1.name
                Sort Method: quicksort  Memory: 25kB
                Buffers: shared hit=6
                ->  HashAggregate  (cost=15.58..16.09 rows=51 width=39) (actual time=0.016..0.019 rows=0 loops=1)
                      Group Key: search_terms_1.package_id, search_terms_1.name
                      Batches: 1  Memory Usage: 24kB
                      Buffers: shared hit=6
                      ->  Bitmap Heap Scan on search_terms search_terms_1  (cost=8.89..12.91 rows=51 width=68) (actual time=0.015..0.017 rows=0 loops=1)
                            Recheck Cond: ((lower(name) ~~ '-%'::text) OR (lower(attr_path) ~~ '-%'::text))
                            Filter: ((lower(name) ~~ '-%'::text) OR (lower(attr_path) ~~ '-%'::text))
                            Buffers: shared hit=6
                            ->  BitmapOr  (cost=8.89..8.89 rows=1 width=0) (actual time=0.013..0.014 rows=0 loops=1)
                                  Buffers: shared hit=6
                                  ->  Bitmap Index Scan on search_terms_name_lower_idx  (cost=0.00..4.43 rows=1 width=0) (actual time=0.008..0.009 rows=0 loops=1)
                                        Index Cond: ((lower(name) >= '-'::text) AND (lower(name) < '.'::text))
                                        Buffers: shared hit=3
                                  ->  Bitmap Index Scan on search_terms_attr_path_lower_idx  (cost=0.00..4.43 rows=1 width=0) (actual time=0.004..0.004 rows=0 loops=1)
                                        Index Cond: ((lower(attr_path) >= '-'::text) AND (lower(attr_path) < '.'::text))
                                        Buffers: shared hit=3
  ->  Sort  (cost=7728.60..7728.85 rows=100 width=44) (actual time=903.760..903.988 rows=0 loops=1)
        Sort Key: (max(prefix.rank)) DESC, (min(prefix.name))
        Sort Method: quicksort  Memory: 25kB
        Buffers: shared hit=3003
        ->  GroupAggregate  (cost=7723.28..7725.28 rows=100 width=44) (actual time=903.758..903.985 rows=0 loops=1)
              Group Key: prefix.package_id
              Buffers: shared hit=3003
              ->  Sort  (cost=7723.28..7723.53 rows=100 width=42) (actual time=903.756..903.984 rows=0 loops=1)
                    Sort Key: prefix.package_id
                    Sort Method: quicksort  Memory: 25kB
                    Buffers: shared hit=3003
                    ->  Append  (cost=0.00..7719.96 rows=100 width=42) (actual time=903.755..903.981 rows=0 loops=1)
                          Buffers: shared hit=3003
                          ->  CTE Scan on prefix  (cost=0.00..1.00 rows=50 width=44) (actual time=0.019..0.019 rows=0 loops=1)
                                Buffers: shared hit=6
                          ->  Subquery Scan on fuzzy  (cost=7717.83..7718.46 rows=50 width=39) (actual time=903.734..903.960 rows=0 loops=1)
                                Buffers: shared hit=2997
                                ->  Limit  (cost=7717.83..7717.96 rows=50 width=39) (actual time=903.733..903.959 rows=0 loops=1)
                                      Buffers: shared hit=2997
                                      InitPlan 2
                                        ->  Aggregate  (cost=1.12..1.14 rows=1 width=8) (actual time=0.002..0.003 rows=1 loops=1)
                                              ->  CTE Scan on prefix prefix_1  (cost=0.00..1.00 rows=50 width=0) (actual time=0.000..0.000 rows=0 loops=1)
                                      ->  Sort  (cost=7716.70..7716.82 rows=51 width=39) (actual time=903.732..903.956 rows=0 loops=1)
                                            Sort Key: (max((((CASE WHEN (lower(search_terms.name) = '-'::text) THEN 1000 WHEN (lower(search_terms.attr_path) = '-'::text) THEN 900 WHEN (lower(search_terms.name) ~~ '-%'::text) THEN 800 WHEN (lower(search_terms.attr_path) ~~ '-%'::text) THEN 700 ELSE 0 END)::double precision + ('100'::double precision * CASE WHEN (search_terms.name = search_terms.attr_path) THEN similarity(search_terms.name, '-'::text) ELSE GREATEST(similarity(search_terms.name, '-'::text), similarity(search_terms.attr_path, '-'::text)) END)) + (CASE WHEN (search_terms.top_level_attr IS NOT NULL) THEN 25 ELSE 0 END)::double precision))) DESC, search_terms.name
                                            Sort Method: quicksort  Memory: 25kB
                                            Buffers: shared hit=2997
                                            ->  Finalize GroupAggregate  (cost=7709.19..7715.25 rows=51 width=39) (actual time=903.729..903.953 rows=0 loops=1)
                                                  Group Key: search_terms.package_id, search_terms.name
                                                  Buffers: shared hit=2997
                                                  ->  Gather Merge  (cost=7709.19..7714.51 rows=30 width=39) (actual time=903.728..903.952 rows=0 loops=1)
                                                        Workers Planned: 1
                                                        Workers Launched: 1
                                                        Buffers: shared hit=2997
                                                        ->  Partial GroupAggregate  (cost=6709.18..6711.13 rows=30 width=39) (actual time=894.473..894.475 rows=0 loops=2)
                                                              Group Key: search_terms.package_id, search_terms.name
                                                              Buffers: shared hit=2997
                                                              ->  Sort  (cost=6709.18..6709.25 rows=30 width=68) (actual time=894.472..894.473 rows=0 loops=2)
                                                                    Sort Key: search_terms.package_id, search_terms.name
                                                                    Sort Method: quicksort  Memory: 25kB
                                                                    Buffers: shared hit=2997
                                                                    Worker 0:  Sort Method: quicksort  Memory: 25kB
                                                                    ->  Result  (cost=0.00..6708.44 rows=30 width=68) (actual time=894.457..894.457 rows=0 loops=2)
                                                                          One-Time Filter: ((InitPlan 2).col1 < 50)
                                                                          Buffers: shared hit=2982
                                                                          ->  Parallel Seq Scan on search_terms  (cost=0.00..6708.44 rows=30 width=68) (actual time=894.455..894.455 rows=0 loops=2)
                                                                                Filter: (((name % '-'::text) OR (attr_path % '-'::text)) AND (lower(name) !~~ '-%'::text) AND (lower(attr_path) !~~ '-%'::text))
                                                                                Rows Removed by Filter: 127160
                                                                                Buffers: shared hit=2982
Planning:
  Buffers: shared hit=30
Planning Time: 8.496 ms
Execution Time: 904.123 ms
```

### batched fetch, latest — q=- (0 hits)

Parameters: `[""]` — Execution Time: 0.106 ms

```
Limit  (cost=2152.04..2152.27 rows=1 width=1378) (actual time=0.012..0.013 rows=0 loops=1)
  ->  Unique  (cost=2152.04..2152.27 rows=1 width=1378) (actual time=0.011..0.012 rows=0 loops=1)
        ->  Incremental Sort  (cost=2152.04..2152.24 rows=14 width=1378) (actual time=0.011..0.012 rows=0 loops=1)
              Sort Key: hits.ord, variants.system, variants.attr_path
              Presorted Key: hits.ord
              Full-sort Groups: 1  Sort Method: quicksort  Average Memory: 25kB  Peak Memory: 25kB
              ->  Nested Loop  (cost=2131.84..2151.78 rows=14 width=1378) (actual time=0.005..0.005 rows=0 loops=1)
                    ->  Nested Loop  (cost=2131.56..2147.61 rows=14 width=1323) (actual time=0.004..0.005 rows=0 loops=1)
                          ->  Nested Loop  (cost=2131.14..2140.36 rows=14 width=340) (actual time=0.004..0.005 rows=0 loops=1)
                                Join Filter: (variants.version_id = versions_1.id)
                                ->  Nested Loop  (cost=2130.71..2138.79 rows=1 width=53) (actual time=0.004..0.005 rows=0 loops=1)
                                      ->  Nested Loop  (cost=2130.29..2138.34 rows=1 width=30) (actual time=0.004..0.004 rows=0 loops=1)
                                            ->  Nested Loop  (cost=2129.86..2129.90 rows=1 width=12) (actual time=0.004..0.004 rows=0 loops=1)
                                                  ->  Function Scan on unnest hits  (cost=0.00..0.01 rows=1 width=12) (actual time=0.004..0.004 rows=0 loops=1)
                                                  ->  Limit  (cost=2129.86..2129.86 rows=1 width=24) (never executed)
                                                        ->  Sort  (cost=2129.86..2130.06 rows=79 width=24) (never executed)
                                                              Sort Key: (EXISTS(SubPlan 1)) DESC, ((SubPlan 2)) DESC, versions_1.sort_key DESC
                                                              ->  Nested Loop  (cost=0.86..2129.47 rows=79 width=24) (never executed)
                                                                    ->  Index Scan using versions_latest_idx on versions versions_1  (cost=0.43..66.26 rows=32 width=19) (never executed)
                                                                          Index Cond: ((package_id = hits.package_id) AND (prerelease = false))
                                                                    ->  Index Scan using variants_identity_key on variants variants_1  (cost=0.43..36.09 rows=14 width=8) (never executed)
                                                                          Index Cond: (version_id = versions_1.id)
                                                                    SubPlan 1
                                                                      ->  Index Scan using variants_identity_key on variants b  (cost=0.43..36.09 rows=14 width=0) (never executed)
                                                                            Index Cond: (version_id = versions_1.id)
                                                                            Filter: (NOT broken)
                                                                    SubPlan 2
                                                                      ->  Aggregate  (cost=8.45..8.46 rows=1 width=4) (never executed)
                                                                            ->  Index Scan using variant_ranges_variant_id_first_seq_pk on variant_ranges r  (cost=0.43..8.45 rows=1 width=4) (never executed)
                                                                                  Index Cond: (variant_id = variants_1.id)
                                                                                  Filter: (NOT seeded)
                                            ->  Index Scan using versions_pkey on versions  (cost=0.43..8.45 rows=1 width=18) (never executed)
                                                  Index Cond: (id = versions_1.id)
                                      ->  Index Scan using packages_pkey on packages  (cost=0.42..0.44 rows=1 width=31) (never executed)
                                            Index Cond: (id = versions.package_id)
                                ->  Index Scan using variants_identity_key on variants  (cost=0.43..1.40 rows=14 width=299) (never executed)
                                      Index Cond: (version_id = versions.id)
                          ->  Index Scan using meta_pkey on meta  (cost=0.42..0.52 rows=1 width=991) (never executed)
                                Index Cond: (id = variants.meta_id)
                    ->  Index Scan using commits_pkey on commits commit_hash  (cost=0.28..0.30 rows=1 width=53) (never executed)
                          Index Cond: (seq = variants.commit_seq)
Planning:
  Buffers: shared hit=115
Planning Time: 18.195 ms
Execution Time: 0.106 ms
```

### batched fetch, all versions — q=- (0 hits)

Parameters: `[""]` — Execution Time: 0.079 ms

```
Limit  (cost=130.62..132.28 rows=82 width=1393) (actual time=0.014..0.015 rows=0 loops=1)
  ->  Unique  (cost=130.62..132.28 rows=82 width=1393) (actual time=0.013..0.014 rows=0 loops=1)
        ->  Incremental Sort  (cost=130.62..131.67 rows=82 width=1393) (actual time=0.013..0.014 rows=0 loops=1)
              Sort Key: hits.ord, versions.sort_key DESC, versions.version, variants.system, variants.attr_path
              Presorted Key: hits.ord
              Full-sort Groups: 1  Sort Method: quicksort  Average Memory: 25kB  Peak Memory: 25kB
              ->  Nested Loop  (cost=1.98..128.02 rows=82 width=1393) (actual time=0.004..0.005 rows=0 loops=1)
                    ->  Nested Loop  (cost=1.70..103.62 rows=82 width=1338) (actual time=0.004..0.004 rows=0 loops=1)
                          ->  Nested Loop  (cost=1.28..61.18 rows=82 width=355) (actual time=0.004..0.004 rows=0 loops=1)
                                ->  Nested Loop  (cost=0.85..10.24 rows=33 width=64) (actual time=0.004..0.004 rows=0 loops=1)
                                      Join Filter: (versions.package_id = hits.package_id)
                                      ->  Nested Loop  (cost=0.42..8.45 rows=1 width=43) (actual time=0.003..0.003 rows=0 loops=1)
                                            ->  Function Scan on unnest hits  (cost=0.00..0.01 rows=1 width=12) (actual time=0.003..0.003 rows=0 loops=1)
                                            ->  Index Scan using packages_pkey on packages  (cost=0.42..8.44 rows=1 width=31) (never executed)
                                                  Index Cond: (id = hits.package_id)
                                      ->  Index Scan using versions_package_version_key on versions  (cost=0.43..1.38 rows=33 width=33) (never executed)
                                            Index Cond: (package_id = packages.id)
                                ->  Index Scan using variants_identity_key on variants  (cost=0.43..1.40 rows=14 width=299) (never executed)
                                      Index Cond: (version_id = versions.id)
                          ->  Index Scan using meta_pkey on meta  (cost=0.42..0.52 rows=1 width=991) (never executed)
                                Index Cond: (id = variants.meta_id)
                    ->  Index Scan using commits_pkey on commits commit_hash  (cost=0.28..0.30 rows=1 width=53) (never executed)
                          Index Cond: (seq = variants.commit_seq)
Planning:
  Buffers: shared hit=69
Planning Time: 10.085 ms
Execution Time: 0.079 ms
```

## Name lookups (/v2/resolve, /v1/resolve, /v2/pkg, /v1/pkg)

### pick latest version — name=go

Parameters: `["go"]` — Execution Time: 8.210 ms

```
Limit  (cost=2821.09..2821.09 rows=1 width=24) (actual time=8.121..8.125 rows=1 loops=1)
  Buffers: shared hit=15706
  ->  Sort  (cost=2821.09..2821.40 rows=125 width=24) (actual time=8.119..8.123 rows=1 loops=1)
        Sort Key: (EXISTS(SubPlan 1)) DESC, ((SubPlan 2)) DESC, versions.sort_key DESC
        Sort Method: top-N heapsort  Memory: 25kB
        Buffers: shared hit=15706
        ->  Nested Loop  (cost=211.57..2820.47 rows=125 width=24) (actual time=1.200..7.982 rows=671 loops=1)
              Buffers: shared hit=15706
              ->  Nested Loop  (cost=211.15..1334.90 rows=125 width=27) (actual time=1.180..3.838 rows=671 loops=1)
                    Buffers: shared hit=6972
                    ->  Nested Loop  (cost=210.72..1275.94 rows=126 width=8) (actual time=1.173..2.617 rows=746 loops=1)
                          Buffers: shared hit=3988
                          ->  HashAggregate  (cost=210.29..211.55 rows=126 width=4) (actual time=1.167..1.279 rows=746 loops=1)
                                Group Key: va.id
                                Batches: 1  Memory Usage: 89kB
                                Buffers: shared hit=1004
                                ->  Append  (cost=1.28..209.98 rows=126 width=4) (actual time=0.031..0.984 rows=1012 loops=1)
                                      Buffers: shared hit=1004
                                      ->  Nested Loop  (cost=1.28..86.07 rows=15 width=4) (actual time=0.031..0.823 rows=746 loops=1)
                                            Buffers: shared hit=934
                                            ->  Nested Loop  (cost=0.85..76.81 rows=6 width=4) (actual time=0.025..0.101 rows=196 loops=1)
                                                  Buffers: shared hit=28
                                                  ->  Index Scan using packages_name_lower_idx on packages p  (cost=0.42..8.44 rows=1 width=4) (actual time=0.013..0.013 rows=1 loops=1)
                                                        Index Cond: (lower(name) = 'go'::text)
                                                        Buffers: shared hit=4
                                                  ->  Index Scan using versions_semver_idx on versions ve  (cost=0.43..68.04 rows=33 width=8) (actual time=0.009..0.063 rows=196 loops=1)
                                                        Index Cond: (package_id = p.id)
                                                        Buffers: shared hit=24
                                            ->  Index Scan using variants_identity_key on variants va  (cost=0.43..1.40 rows=14 width=8) (actual time=0.002..0.003 rows=4 loops=196)
                                                  Index Cond: (version_id = ve.id)
                                                  Buffers: shared hit=906
                                      ->  Index Scan using variants_attr_path_idx on variants va_1  (cost=0.43..123.28 rows=111 width=4) (actual time=0.007..0.081 rows=266 loops=1)
                                            Index Cond: (attr_path = 'go'::text)
                                            Buffers: shared hit=70
                          ->  Index Scan using variants_pkey on variants  (cost=0.43..8.45 rows=1 width=8) (actual time=0.001..0.001 rows=1 loops=746)
                                Index Cond: (id = va.id)
                                Buffers: shared hit=2984
                    ->  Index Scan using versions_pkey on versions  (cost=0.43..0.47 rows=1 width=23) (actual time=0.001..0.001 rows=1 loops=746)
                          Index Cond: (id = variants.version_id)
                          Filter: (NOT prerelease)
                          Rows Removed by Filter: 0
                          Buffers: shared hit=2984
              ->  Index Only Scan using packages_pkey on packages  (cost=0.42..0.44 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=671)
                    Index Cond: (id = versions.package_id)
                    Heap Fetches: 671
                    Buffers: shared hit=3356
              SubPlan 1
                ->  Index Scan using variants_identity_key on variants b  (cost=0.43..36.09 rows=14 width=0) (actual time=0.002..0.002 rows=1 loops=671)
                      Index Cond: (version_id = versions.id)
                      Filter: (NOT broken)
                      Buffers: shared hit=2684
              SubPlan 2
                ->  Aggregate  (cost=8.45..8.46 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=671)
                      Buffers: shared hit=2694
                      ->  Index Scan using variant_ranges_variant_id_first_seq_pk on variant_ranges r  (cost=0.43..8.45 rows=1 width=4) (actual time=0.002..0.002 rows=0 loops=671)
                            Index Cond: (variant_id = variants.id)
                            Filter: (NOT seeded)
                            Rows Removed by Filter: 1
                            Buffers: shared hit=2694
Planning:
  Buffers: shared hit=94
Planning Time: 14.788 ms
Execution Time: 8.210 ms
```

### pick latest version — name=python

Parameters: `["python"]` — Execution Time: 6.205 ms

```
Limit  (cost=2821.09..2821.09 rows=1 width=24) (actual time=6.112..6.116 rows=1 loops=1)
  Buffers: shared hit=11805
  ->  Sort  (cost=2821.09..2821.40 rows=125 width=24) (actual time=6.110..6.114 rows=1 loops=1)
        Sort Key: (EXISTS(SubPlan 1)) DESC, ((SubPlan 2)) DESC, versions.sort_key DESC
        Sort Method: top-N heapsort  Memory: 25kB
        Buffers: shared hit=11805
        ->  Nested Loop  (cost=211.57..2820.47 rows=125 width=24) (actual time=0.895..6.023 rows=433 loops=1)
              Buffers: shared hit=11805
              ->  Nested Loop  (cost=211.15..1334.90 rows=125 width=27) (actual time=0.874..3.357 rows=433 loops=1)
                    Buffers: shared hit=6166
                    ->  Nested Loop  (cost=210.72..1275.94 rows=126 width=8) (actual time=0.867..2.185 rows=669 loops=1)
                          Buffers: shared hit=3490
                          ->  HashAggregate  (cost=210.29..211.55 rows=126 width=4) (actual time=0.861..0.968 rows=669 loops=1)
                                Group Key: va.id
                                Batches: 1  Memory Usage: 89kB
                                Buffers: shared hit=814
                                ->  Append  (cost=1.28..209.98 rows=126 width=4) (actual time=0.029..0.728 rows=672 loops=1)
                                      Buffers: shared hit=814
                                      ->  Nested Loop  (cost=1.28..86.07 rows=15 width=4) (actual time=0.029..0.666 rows=669 loops=1)
                                            Buffers: shared hit=809
                                            ->  Nested Loop  (cost=0.85..76.81 rows=6 width=4) (actual time=0.021..0.096 rows=183 loops=1)
                                                  Buffers: shared hit=30
                                                  ->  Index Scan using packages_name_lower_idx on packages p  (cost=0.42..8.44 rows=1 width=4) (actual time=0.012..0.013 rows=1 loops=1)
                                                        Index Cond: (lower(name) = 'python'::text)
                                                        Buffers: shared hit=4
                                                  ->  Index Scan using versions_semver_idx on versions ve  (cost=0.43..68.04 rows=33 width=8) (actual time=0.007..0.059 rows=183 loops=1)
                                                        Index Cond: (package_id = p.id)
                                                        Buffers: shared hit=26
                                            ->  Index Scan using variants_identity_key on variants va  (cost=0.43..1.40 rows=14 width=8) (actual time=0.002..0.002 rows=4 loops=183)
                                                  Index Cond: (version_id = ve.id)
                                                  Buffers: shared hit=779
                                      ->  Index Scan using variants_attr_path_idx on variants va_1  (cost=0.43..123.28 rows=111 width=4) (actual time=0.007..0.008 rows=3 loops=1)
                                            Index Cond: (attr_path = 'python'::text)
                                            Buffers: shared hit=5
                          ->  Index Scan using variants_pkey on variants  (cost=0.43..8.45 rows=1 width=8) (actual time=0.001..0.001 rows=1 loops=669)
                                Index Cond: (id = va.id)
                                Buffers: shared hit=2676
                    ->  Index Scan using versions_pkey on versions  (cost=0.43..0.47 rows=1 width=23) (actual time=0.001..0.001 rows=1 loops=669)
                          Index Cond: (id = variants.version_id)
                          Filter: (NOT prerelease)
                          Rows Removed by Filter: 0
                          Buffers: shared hit=2676
              ->  Index Only Scan using packages_pkey on packages  (cost=0.42..0.44 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=433)
                    Index Cond: (id = versions.package_id)
                    Heap Fetches: 433
                    Buffers: shared hit=2166
              SubPlan 1
                ->  Index Scan using variants_identity_key on variants b  (cost=0.43..36.09 rows=14 width=0) (actual time=0.002..0.002 rows=1 loops=433)
                      Index Cond: (version_id = versions.id)
                      Filter: (NOT broken)
                      Buffers: shared hit=1732
              SubPlan 2
                ->  Aggregate  (cost=8.45..8.46 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=433)
                      Buffers: shared hit=1741
                      ->  Index Scan using variant_ranges_variant_id_first_seq_pk on variant_ranges r  (cost=0.43..8.45 rows=1 width=4) (actual time=0.002..0.002 rows=0 loops=433)
                            Index Cond: (variant_id = variants.id)
                            Filter: (NOT seeded)
                            Rows Removed by Filter: 1
                            Buffers: shared hit=1741
Planning:
  Buffers: shared hit=94
Planning Time: 14.453 ms
Execution Time: 6.205 ms
```

### pick latest version — name=python311

Parameters: `["python311"]` — Execution Time: 0.888 ms

```
Limit  (cost=2821.09..2821.09 rows=1 width=24) (actual time=0.801..0.803 rows=1 loops=1)
  Buffers: shared hit=1444
  ->  Sort  (cost=2821.09..2821.40 rows=125 width=24) (actual time=0.800..0.802 rows=1 loops=1)
        Sort Key: (EXISTS(SubPlan 1)) DESC, ((SubPlan 2)) DESC, versions.sort_key DESC
        Sort Method: top-N heapsort  Memory: 25kB
        Buffers: shared hit=1444
        ->  Nested Loop  (cost=211.57..2820.47 rows=125 width=24) (actual time=0.122..0.783 rows=54 loops=1)
              Buffers: shared hit=1444
              ->  Nested Loop  (cost=211.15..1334.90 rows=125 width=27) (actual time=0.097..0.420 rows=54 loops=1)
                    Buffers: shared hit=738
                    ->  Nested Loop  (cost=210.72..1275.94 rows=126 width=8) (actual time=0.090..0.264 rows=90 loops=1)
                          Buffers: shared hit=378
                          ->  HashAggregate  (cost=210.29..211.55 rows=126 width=4) (actual time=0.081..0.094 rows=90 loops=1)
                                Group Key: va.id
                                Batches: 1  Memory Usage: 40kB
                                Buffers: shared hit=18
                                ->  Append  (cost=1.28..209.98 rows=126 width=4) (actual time=0.021..0.064 rows=90 loops=1)
                                      Buffers: shared hit=18
                                      ->  Nested Loop  (cost=1.28..86.07 rows=15 width=4) (actual time=0.011..0.012 rows=0 loops=1)
                                            Buffers: shared hit=3
                                            ->  Nested Loop  (cost=0.85..76.81 rows=6 width=4) (actual time=0.011..0.011 rows=0 loops=1)
                                                  Buffers: shared hit=3
                                                  ->  Index Scan using packages_name_lower_idx on packages p  (cost=0.42..8.44 rows=1 width=4) (actual time=0.011..0.011 rows=0 loops=1)
                                                        Index Cond: (lower(name) = 'python311'::text)
                                                        Buffers: shared hit=3
                                                  ->  Index Scan using versions_semver_idx on versions ve  (cost=0.43..68.04 rows=33 width=8) (never executed)
                                                        Index Cond: (package_id = p.id)
                                            ->  Index Scan using variants_identity_key on variants va  (cost=0.43..1.40 rows=14 width=8) (never executed)
                                                  Index Cond: (version_id = ve.id)
                                      ->  Index Scan using variants_attr_path_idx on variants va_1  (cost=0.43..123.28 rows=111 width=4) (actual time=0.010..0.045 rows=90 loops=1)
                                            Index Cond: (attr_path = 'python311'::text)
                                            Buffers: shared hit=15
                          ->  Index Scan using variants_pkey on variants  (cost=0.43..8.45 rows=1 width=8) (actual time=0.002..0.002 rows=1 loops=90)
                                Index Cond: (id = va.id)
                                Buffers: shared hit=360
                    ->  Index Scan using versions_pkey on versions  (cost=0.43..0.47 rows=1 width=23) (actual time=0.001..0.001 rows=1 loops=90)
                          Index Cond: (id = variants.version_id)
                          Filter: (NOT prerelease)
                          Rows Removed by Filter: 0
                          Buffers: shared hit=360
              ->  Index Only Scan using packages_pkey on packages  (cost=0.42..0.44 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=54)
                    Index Cond: (id = versions.package_id)
                    Heap Fetches: 54
                    Buffers: shared hit=271
              SubPlan 1
                ->  Index Scan using variants_identity_key on variants b  (cost=0.43..36.09 rows=14 width=0) (actual time=0.002..0.002 rows=1 loops=54)
                      Index Cond: (version_id = versions.id)
                      Filter: (NOT broken)
                      Buffers: shared hit=216
              SubPlan 2
                ->  Aggregate  (cost=8.45..8.46 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=54)
                      Buffers: shared hit=219
                      ->  Index Scan using variant_ranges_variant_id_first_seq_pk on variant_ranges r  (cost=0.43..8.45 rows=1 width=4) (actual time=0.002..0.002 rows=0 loops=54)
                            Index Cond: (variant_id = variants.id)
                            Filter: (NOT seeded)
                            Rows Removed by Filter: 1
                            Buffers: shared hit=219
Planning:
  Buffers: shared hit=94
Planning Time: 15.652 ms
Execution Time: 0.888 ms
```

### pick latest version — name=hello

Parameters: `["hello"]` — Execution Time: 0.381 ms

```
Limit  (cost=2821.09..2821.09 rows=1 width=24) (actual time=0.299..0.301 rows=1 loops=1)
  Buffers: shared hit=444
  ->  Sort  (cost=2821.09..2821.40 rows=125 width=24) (actual time=0.298..0.300 rows=1 loops=1)
        Sort Key: (EXISTS(SubPlan 1)) DESC, ((SubPlan 2)) DESC, versions.sort_key DESC
        Sort Method: top-N heapsort  Memory: 25kB
        Buffers: shared hit=444
        ->  Nested Loop  (cost=211.57..2820.47 rows=125 width=24) (actual time=0.112..0.288 rows=19 loops=1)
              Buffers: shared hit=444
              ->  Nested Loop  (cost=211.15..1334.90 rows=125 width=27) (actual time=0.090..0.155 rows=19 loops=1)
                    Buffers: shared hit=193
                    ->  Nested Loop  (cost=210.72..1275.94 rows=126 width=8) (actual time=0.083..0.118 rows=19 loops=1)
                          Buffers: shared hit=117
                          ->  HashAggregate  (cost=210.29..211.55 rows=126 width=4) (actual time=0.077..0.081 rows=19 loops=1)
                                Group Key: va.id
                                Batches: 1  Memory Usage: 40kB
                                Buffers: shared hit=41
                                ->  Append  (cost=1.28..209.98 rows=126 width=4) (actual time=0.029..0.067 rows=38 loops=1)
                                      Buffers: shared hit=41
                                      ->  Nested Loop  (cost=1.28..86.07 rows=15 width=4) (actual time=0.029..0.052 rows=19 loops=1)
                                            Buffers: shared hit=33
                                            ->  Nested Loop  (cost=0.85..76.81 rows=6 width=4) (actual time=0.023..0.025 rows=5 loops=1)
                                                  Buffers: shared hit=8
                                                  ->  Index Scan using packages_name_lower_idx on packages p  (cost=0.42..8.44 rows=1 width=4) (actual time=0.013..0.013 rows=1 loops=1)
                                                        Index Cond: (lower(name) = 'hello'::text)
                                                        Buffers: shared hit=4
                                                  ->  Index Scan using versions_semver_idx on versions ve  (cost=0.43..68.04 rows=33 width=8) (actual time=0.007..0.008 rows=5 loops=1)
                                                        Index Cond: (package_id = p.id)
                                                        Buffers: shared hit=4
                                            ->  Index Scan using variants_identity_key on variants va  (cost=0.43..1.40 rows=14 width=8) (actual time=0.003..0.005 rows=4 loops=5)
                                                  Index Cond: (version_id = ve.id)
                                                  Buffers: shared hit=25
                                      ->  Index Scan using variants_attr_path_idx on variants va_1  (cost=0.43..123.28 rows=111 width=4) (actual time=0.006..0.011 rows=19 loops=1)
                                            Index Cond: (attr_path = 'hello'::text)
                                            Buffers: shared hit=8
                          ->  Index Scan using variants_pkey on variants  (cost=0.43..8.45 rows=1 width=8) (actual time=0.002..0.002 rows=1 loops=19)
                                Index Cond: (id = va.id)
                                Buffers: shared hit=76
                    ->  Index Scan using versions_pkey on versions  (cost=0.43..0.47 rows=1 width=23) (actual time=0.002..0.002 rows=1 loops=19)
                          Index Cond: (id = variants.version_id)
                          Filter: (NOT prerelease)
                          Buffers: shared hit=76
              ->  Index Only Scan using packages_pkey on packages  (cost=0.42..0.44 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=19)
                    Index Cond: (id = versions.package_id)
                    Heap Fetches: 19
                    Buffers: shared hit=96
              SubPlan 1
                ->  Index Scan using variants_identity_key on variants b  (cost=0.43..36.09 rows=14 width=0) (actual time=0.002..0.002 rows=1 loops=19)
                      Index Cond: (version_id = versions.id)
                      Filter: (NOT broken)
                      Buffers: shared hit=76
              SubPlan 2
                ->  Aggregate  (cost=8.45..8.46 rows=1 width=4) (actual time=0.002..0.002 rows=1 loops=19)
                      Buffers: shared hit=79
                      ->  Index Scan using variant_ranges_variant_id_first_seq_pk on variant_ranges r  (cost=0.43..8.45 rows=1 width=4) (actual time=0.002..0.002 rows=0 loops=19)
                            Index Cond: (variant_id = variants.id)
                            Filter: (NOT seeded)
                            Rows Removed by Filter: 1
                            Buffers: shared hit=79
Planning:
  Buffers: shared hit=94
Planning Time: 15.132 ms
Execution Time: 0.381 ms
```

### every version — name=go

Parameters: `["go"]` — Execution Time: 8.677 ms

```
Sort  (cost=1498.01..1498.33 rows=126 width=1389) (actual time=8.420..8.460 rows=746 loops=1)
  Sort Key: versions.sort_key DESC, versions.version, versions.id, variants.system, variants.attr_path
  Sort Method: quicksort  Memory: 576kB
  Buffers: shared hit=15178
  ->  Nested Loop  (cost=212.27..1493.61 rows=126 width=1389) (actual time=1.162..7.443 rows=746 loops=1)
        Buffers: shared hit=15178
        ->  Nested Loop  (cost=211.99..1456.12 rows=126 width=1334) (actual time=1.155..6.267 rows=746 loops=1)
              Buffers: shared hit=12940
              ->  Nested Loop  (cost=211.57..1390.92 rows=126 width=351) (actual time=1.148..5.111 rows=746 loops=1)
                    Buffers: shared hit=9956
                    ->  Nested Loop  (cost=211.15..1334.90 rows=126 width=328) (actual time=1.143..3.994 rows=746 loops=1)
                          Buffers: shared hit=6972
                          ->  Nested Loop  (cost=210.72..1275.94 rows=126 width=299) (actual time=1.137..2.736 rows=746 loops=1)
                                Buffers: shared hit=3988
                                ->  HashAggregate  (cost=210.29..211.55 rows=126 width=4) (actual time=1.129..1.271 rows=746 loops=1)
                                      Group Key: va.id
                                      Batches: 1  Memory Usage: 89kB
                                      Buffers: shared hit=1004
                                      ->  Append  (cost=1.28..209.98 rows=126 width=4) (actual time=0.033..0.949 rows=1012 loops=1)
                                            Buffers: shared hit=1004
                                            ->  Nested Loop  (cost=1.28..86.07 rows=15 width=4) (actual time=0.033..0.787 rows=746 loops=1)
                                                  Buffers: shared hit=934
                                                  ->  Nested Loop  (cost=0.85..76.81 rows=6 width=4) (actual time=0.025..0.103 rows=196 loops=1)
                                                        Buffers: shared hit=28
                                                        ->  Index Scan using packages_name_lower_idx on packages p  (cost=0.42..8.44 rows=1 width=4) (actual time=0.013..0.014 rows=1 loops=1)
                                                              Index Cond: (lower(name) = 'go'::text)
                                                              Buffers: shared hit=4
                                                        ->  Index Scan using versions_semver_idx on versions ve  (cost=0.43..68.04 rows=33 width=8) (actual time=0.009..0.063 rows=196 loops=1)
                                                              Index Cond: (package_id = p.id)
                                                              Buffers: shared hit=24
                                                  ->  Index Scan using variants_identity_key on variants va  (cost=0.43..1.40 rows=14 width=8) (actual time=0.002..0.003 rows=4 loops=196)
                                                        Index Cond: (version_id = ve.id)
                                                        Buffers: shared hit=906
                                            ->  Index Scan using variants_attr_path_idx on variants va_1  (cost=0.43..123.28 rows=111 width=4) (actual time=0.007..0.084 rows=266 loops=1)
                                                  Index Cond: (attr_path = 'go'::text)
                                                  Buffers: shared hit=70
                                ->  Index Scan using variants_pkey on variants  (cost=0.43..8.45 rows=1 width=303) (actual time=0.002..0.002 rows=1 loops=746)
                                      Index Cond: (id = va.id)
                                      Buffers: shared hit=2984
                          ->  Index Scan using versions_pkey on versions  (cost=0.43..0.47 rows=1 width=33) (actual time=0.001..0.001 rows=1 loops=746)
                                Index Cond: (id = variants.version_id)
                                Buffers: shared hit=2984
                    ->  Index Scan using packages_pkey on packages  (cost=0.42..0.44 rows=1 width=31) (actual time=0.001..0.001 rows=1 loops=746)
                          Index Cond: (id = versions.package_id)
                          Buffers: shared hit=2984
              ->  Index Scan using meta_pkey on meta  (cost=0.42..0.52 rows=1 width=991) (actual time=0.001..0.001 rows=1 loops=746)
                    Index Cond: (id = variants.meta_id)
                    Buffers: shared hit=2984
        ->  Index Scan using commits_pkey on commits commit_hash  (cost=0.28..0.30 rows=1 width=53) (actual time=0.001..0.001 rows=1 loops=746)
              Index Cond: (seq = variants.commit_seq)
              Buffers: shared hit=2238
Planning:
  Buffers: shared hit=116
Planning Time: 18.189 ms
Execution Time: 8.677 ms
```

### every version — name=python

Parameters: `["python"]` — Execution Time: 7.771 ms

```
Sort  (cost=1498.01..1498.33 rows=126 width=1389) (actual time=7.509..7.546 rows=669 loops=1)
  Sort Key: versions.sort_key DESC, versions.version, versions.id, variants.system, variants.attr_path
  Sort Method: quicksort  Memory: 906kB
  Buffers: shared hit=13525
  ->  Nested Loop  (cost=212.27..1493.61 rows=126 width=1389) (actual time=0.898..6.540 rows=669 loops=1)
        Buffers: shared hit=13525
        ->  Nested Loop  (cost=211.99..1456.12 rows=126 width=1334) (actual time=0.892..5.574 rows=669 loops=1)
              Buffers: shared hit=11518
              ->  Nested Loop  (cost=211.57..1390.92 rows=126 width=351) (actual time=0.884..4.459 rows=669 loops=1)
                    Buffers: shared hit=8842
                    ->  Nested Loop  (cost=211.15..1334.90 rows=126 width=328) (actual time=0.878..3.466 rows=669 loops=1)
                          Buffers: shared hit=6166
                          ->  Nested Loop  (cost=210.72..1275.94 rows=126 width=299) (actual time=0.873..2.367 rows=669 loops=1)
                                Buffers: shared hit=3490
                                ->  HashAggregate  (cost=210.29..211.55 rows=126 width=4) (actual time=0.865..0.987 rows=669 loops=1)
                                      Group Key: va.id
                                      Batches: 1  Memory Usage: 89kB
                                      Buffers: shared hit=814
                                      ->  Append  (cost=1.28..209.98 rows=126 width=4) (actual time=0.033..0.732 rows=672 loops=1)
                                            Buffers: shared hit=814
                                            ->  Nested Loop  (cost=1.28..86.07 rows=15 width=4) (actual time=0.032..0.672 rows=669 loops=1)
                                                  Buffers: shared hit=809
                                                  ->  Nested Loop  (cost=0.85..76.81 rows=6 width=4) (actual time=0.024..0.104 rows=183 loops=1)
                                                        Buffers: shared hit=30
                                                        ->  Index Scan using packages_name_lower_idx on packages p  (cost=0.42..8.44 rows=1 width=4) (actual time=0.014..0.015 rows=1 loops=1)
                                                              Index Cond: (lower(name) = 'python'::text)
                                                              Buffers: shared hit=4
                                                        ->  Index Scan using versions_semver_idx on versions ve  (cost=0.43..68.04 rows=33 width=8) (actual time=0.007..0.063 rows=183 loops=1)
                                                              Index Cond: (package_id = p.id)
                                                              Buffers: shared hit=26
                                                  ->  Index Scan using variants_identity_key on variants va  (cost=0.43..1.40 rows=14 width=8) (actual time=0.002..0.003 rows=4 loops=183)
                                                        Index Cond: (version_id = ve.id)
                                                        Buffers: shared hit=779
                                            ->  Index Scan using variants_attr_path_idx on variants va_1  (cost=0.43..123.28 rows=111 width=4) (actual time=0.007..0.008 rows=3 loops=1)
                                                  Index Cond: (attr_path = 'python'::text)
                                                  Buffers: shared hit=5
                                ->  Index Scan using variants_pkey on variants  (cost=0.43..8.45 rows=1 width=303) (actual time=0.002..0.002 rows=1 loops=669)
                                      Index Cond: (id = va.id)
                                      Buffers: shared hit=2676
                          ->  Index Scan using versions_pkey on versions  (cost=0.43..0.47 rows=1 width=33) (actual time=0.001..0.001 rows=1 loops=669)
                                Index Cond: (id = variants.version_id)
                                Buffers: shared hit=2676
                    ->  Index Scan using packages_pkey on packages  (cost=0.42..0.44 rows=1 width=31) (actual time=0.001..0.001 rows=1 loops=669)
                          Index Cond: (id = versions.package_id)
                          Buffers: shared hit=2676
              ->  Index Scan using meta_pkey on meta  (cost=0.42..0.52 rows=1 width=991) (actual time=0.001..0.001 rows=1 loops=669)
                    Index Cond: (id = variants.meta_id)
                    Buffers: shared hit=2676
        ->  Index Scan using commits_pkey on commits commit_hash  (cost=0.28..0.30 rows=1 width=53) (actual time=0.001..0.001 rows=1 loops=669)
              Index Cond: (seq = variants.commit_seq)
              Buffers: shared hit=2007
Planning:
  Buffers: shared hit=116
Planning Time: 19.051 ms
Execution Time: 7.771 ms
```

### every version — name=python311

Parameters: `["python311"]` — Execution Time: 1.214 ms

```
Sort  (cost=1498.01..1498.33 rows=126 width=1389) (actual time=1.107..1.113 rows=90 loops=1)
  Sort Key: versions.sort_key DESC, versions.version, versions.id, variants.system, variants.attr_path
  Sort Method: quicksort  Memory: 140kB
  Buffers: shared hit=1728
  ->  Nested Loop  (cost=212.27..1493.61 rows=126 width=1389) (actual time=0.120..0.872 rows=90 loops=1)
        Buffers: shared hit=1728
        ->  Nested Loop  (cost=211.99..1456.12 rows=126 width=1334) (actual time=0.114..0.728 rows=90 loops=1)
              Buffers: shared hit=1458
              ->  Nested Loop  (cost=211.57..1390.92 rows=126 width=351) (actual time=0.107..0.574 rows=90 loops=1)
                    Buffers: shared hit=1098
                    ->  Nested Loop  (cost=211.15..1334.90 rows=126 width=328) (actual time=0.101..0.436 rows=90 loops=1)
                          Buffers: shared hit=738
                          ->  Nested Loop  (cost=210.72..1275.94 rows=126 width=299) (actual time=0.094..0.281 rows=90 loops=1)
                                Buffers: shared hit=378
                                ->  HashAggregate  (cost=210.29..211.55 rows=126 width=4) (actual time=0.085..0.099 rows=90 loops=1)
                                      Group Key: va.id
                                      Batches: 1  Memory Usage: 40kB
                                      Buffers: shared hit=18
                                      ->  Append  (cost=1.28..209.98 rows=126 width=4) (actual time=0.023..0.067 rows=90 loops=1)
                                            Buffers: shared hit=18
                                            ->  Nested Loop  (cost=1.28..86.07 rows=15 width=4) (actual time=0.012..0.013 rows=0 loops=1)
                                                  Buffers: shared hit=3
                                                  ->  Nested Loop  (cost=0.85..76.81 rows=6 width=4) (actual time=0.012..0.013 rows=0 loops=1)
                                                        Buffers: shared hit=3
                                                        ->  Index Scan using packages_name_lower_idx on packages p  (cost=0.42..8.44 rows=1 width=4) (actual time=0.012..0.012 rows=0 loops=1)
                                                              Index Cond: (lower(name) = 'python311'::text)
                                                              Buffers: shared hit=3
                                                        ->  Index Scan using versions_semver_idx on versions ve  (cost=0.43..68.04 rows=33 width=8) (never executed)
                                                              Index Cond: (package_id = p.id)
                                                  ->  Index Scan using variants_identity_key on variants va  (cost=0.43..1.40 rows=14 width=8) (never executed)
                                                        Index Cond: (version_id = ve.id)
                                            ->  Index Scan using variants_attr_path_idx on variants va_1  (cost=0.43..123.28 rows=111 width=4) (actual time=0.010..0.046 rows=90 loops=1)
                                                  Index Cond: (attr_path = 'python311'::text)
                                                  Buffers: shared hit=15
                                ->  Index Scan using variants_pkey on variants  (cost=0.43..8.45 rows=1 width=303) (actual time=0.002..0.002 rows=1 loops=90)
                                      Index Cond: (id = va.id)
                                      Buffers: shared hit=360
                          ->  Index Scan using versions_pkey on versions  (cost=0.43..0.47 rows=1 width=33) (actual time=0.001..0.001 rows=1 loops=90)
                                Index Cond: (id = variants.version_id)
                                Buffers: shared hit=360
                    ->  Index Scan using packages_pkey on packages  (cost=0.42..0.44 rows=1 width=31) (actual time=0.001..0.001 rows=1 loops=90)
                          Index Cond: (id = versions.package_id)
                          Buffers: shared hit=360
              ->  Index Scan using meta_pkey on meta  (cost=0.42..0.52 rows=1 width=991) (actual time=0.001..0.001 rows=1 loops=90)
                    Index Cond: (id = variants.meta_id)
                    Buffers: shared hit=360
        ->  Index Scan using commits_pkey on commits commit_hash  (cost=0.28..0.30 rows=1 width=53) (actual time=0.001..0.001 rows=1 loops=90)
              Index Cond: (seq = variants.commit_seq)
              Buffers: shared hit=270
Planning:
  Buffers: shared hit=116
Planning Time: 17.031 ms
Execution Time: 1.214 ms
```

### every version — name=hello

Parameters: `["hello"]` — Execution Time: 0.405 ms

```
Sort  (cost=1498.01..1498.33 rows=126 width=1389) (actual time=0.314..0.317 rows=19 loops=1)
  Sort Key: versions.sort_key DESC, versions.version, versions.id, variants.system, variants.attr_path
  Sort Method: quicksort  Memory: 56kB
  Buffers: shared hit=402
  ->  Nested Loop  (cost=212.27..1493.61 rows=126 width=1389) (actual time=0.106..0.265 rows=19 loops=1)
        Buffers: shared hit=402
        ->  Nested Loop  (cost=211.99..1456.12 rows=126 width=1334) (actual time=0.101..0.227 rows=19 loops=1)
              Buffers: shared hit=345
              ->  Nested Loop  (cost=211.57..1390.92 rows=126 width=351) (actual time=0.093..0.189 rows=19 loops=1)
                    Buffers: shared hit=269
                    ->  Nested Loop  (cost=211.15..1334.90 rows=126 width=328) (actual time=0.089..0.155 rows=19 loops=1)
                          Buffers: shared hit=193
                          ->  Nested Loop  (cost=210.72..1275.94 rows=126 width=299) (actual time=0.083..0.119 rows=19 loops=1)
                                Buffers: shared hit=117
                                ->  HashAggregate  (cost=210.29..211.55 rows=126 width=4) (actual time=0.077..0.081 rows=19 loops=1)
                                      Group Key: va.id
                                      Batches: 1  Memory Usage: 40kB
                                      Buffers: shared hit=41
                                      ->  Append  (cost=1.28..209.98 rows=126 width=4) (actual time=0.028..0.066 rows=38 loops=1)
                                            Buffers: shared hit=41
                                            ->  Nested Loop  (cost=1.28..86.07 rows=15 width=4) (actual time=0.028..0.052 rows=19 loops=1)
                                                  Buffers: shared hit=33
                                                  ->  Nested Loop  (cost=0.85..76.81 rows=6 width=4) (actual time=0.022..0.024 rows=5 loops=1)
                                                        Buffers: shared hit=8
                                                        ->  Index Scan using packages_name_lower_idx on packages p  (cost=0.42..8.44 rows=1 width=4) (actual time=0.012..0.013 rows=1 loops=1)
                                                              Index Cond: (lower(name) = 'hello'::text)
                                                              Buffers: shared hit=4
                                                        ->  Index Scan using versions_semver_idx on versions ve  (cost=0.43..68.04 rows=33 width=8) (actual time=0.007..0.008 rows=5 loops=1)
                                                              Index Cond: (package_id = p.id)
                                                              Buffers: shared hit=4
                                                  ->  Index Scan using variants_identity_key on variants va  (cost=0.43..1.40 rows=14 width=8) (actual time=0.003..0.005 rows=4 loops=5)
                                                        Index Cond: (version_id = ve.id)
                                                        Buffers: shared hit=25
                                            ->  Index Scan using variants_attr_path_idx on variants va_1  (cost=0.43..123.28 rows=111 width=4) (actual time=0.006..0.011 rows=19 loops=1)
                                                  Index Cond: (attr_path = 'hello'::text)
                                                  Buffers: shared hit=8
                                ->  Index Scan using variants_pkey on variants  (cost=0.43..8.45 rows=1 width=303) (actual time=0.001..0.001 rows=1 loops=19)
                                      Index Cond: (id = va.id)
                                      Buffers: shared hit=76
                          ->  Index Scan using versions_pkey on versions  (cost=0.43..0.47 rows=1 width=33) (actual time=0.001..0.001 rows=1 loops=19)
                                Index Cond: (id = variants.version_id)
                                Buffers: shared hit=76
                    ->  Index Scan using packages_pkey on packages  (cost=0.42..0.44 rows=1 width=31) (actual time=0.001..0.001 rows=1 loops=19)
                          Index Cond: (id = versions.package_id)
                          Buffers: shared hit=76
              ->  Index Scan using meta_pkey on meta  (cost=0.42..0.52 rows=1 width=991) (actual time=0.002..0.002 rows=1 loops=19)
                    Index Cond: (id = variants.meta_id)
                    Buffers: shared hit=76
        ->  Index Scan using commits_pkey on commits commit_hash  (cost=0.28..0.30 rows=1 width=53) (actual time=0.002..0.002 rows=1 loops=19)
              Index Cond: (seq = variants.commit_seq)
              Buffers: shared hit=57
Planning:
  Buffers: shared hit=116
Planning Time: 17.566 ms
Execution Time: 0.405 ms
```

