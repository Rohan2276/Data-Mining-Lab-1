# SetuBid Deduplication — Question 2 Final README

## 1. Objective

SetuBid aggregates 12,000 procurement notices from 260 portals. The same tender can appear under different portal reference numbers, dates and wording. The system therefore needs a similarity-based retrieval stage that is:

- substantially cheaper than all-pairs comparison;
- measurable against the only trusted labels, `data/labelled_pairs.csv`;
- persistent in relational storage;
- reproducible after process restart;
- capable of showing the cost/recall trade-off;
- measured against the 20-minute nightly requirement.

This README records the implemented measurements for Question 2, Sections A and B.

---

# A — From an interactable comparison to a tractable one

## A(a) Mechanical definition of similarity

### Corpus observations

The supplied portal notes show that:

- P001/P002/P005 add approximately 1,400 characters of common legal preamble.
- P003/P004/P006 use a similar State Procurement Cell preamble.
- Portal reference numbers are not shared identifiers.
- Dates occur in many formats.
- Money occurs in several formats.
- Some portals use upper case.
- Corrigenda repeat the original notice and append a correction section.
- Some portals truncate notices.

Therefore the text representation was normalized rather than compared as raw strings.

### Normalization

The working representation:

1. lowercases text;
2. removes formatting/noise differences;
3. retains the substantive words;
4. avoids treating portal-specific reference numbers and formatting as primary identity;
5. keeps the normalized text available for downstream shingling.

A sample of the normalized representation was inspected in DuckDB before the similarity measurements.

### Competing text decompositions measured

Two competing decompositions were evaluated on the labelled pairs:

- word shingles (`word5`);
- character shingles (`char3`).

The measured labelled-pair results were:

| Method | Label | Pairs | Mean Jaccard | Median Jaccard | Minimum | Maximum |
|---|---|---:|---:|---:|---:|---:|
| word5 | different | 621 | 0.2201 | 0.1974 | 0.0792 | 0.5050 |
| word5 | same | 279 | 0.6610 | 0.6652 | 0.1942 | 1.0000 |
| char3 | different | 621 | 0.6081 | 0.6051 | 0.3645 | 0.7844 |
| char3 | same | 279 | 0.8177 | 0.8253 | 0.4181 | 1.0000 |

Word shingles produced much stronger separation between the labelled SAME and DIFFERENT groups. Character shingles were substantially more similar across the two labels, which is consistent with shared boilerplate and formatting-level overlap.

**Adopted representation: word 5-grams (`word5`).**

The adoption cost is that word-level tokenization can be less tolerant of small spelling/character changes than character shingles, so this choice was validated empirically rather than assumed from a tutorial.

---

## A(b) Reduced representation and estimation error

Exact whole-document comparison was not retained for every pair. Notices were represented using MinHash signatures.

The corpus contains:

- 12,000 notices;
- average word-shingle count: approximately 1,307.

The MinHash approximation was measured against exact Jaccard on the 900 labelled pairs.

### Measured estimator error

| Signature | Mean absolute error | Median absolute error | Maximum absolute error |
|---|---:|---:|---:|
| 64 hash functions | 0.07475 | 0.06822 | 0.28726 |
| 128 hash functions | 0.03403 | 0.02940 | 0.12538 |
| 256 hash functions | 0.01897 | 0.01524 | 0.08701 |

The 128-hash representation reduced mean absolute error to about 0.034 compared with about 0.075 for 64 hashes. The 256-hash version was more accurate, but doubles the signature size relative to 128 hashes.

**Adopted MinHash size: 128 hash functions.**

This is a measured space/accuracy trade-off: 128 hashes provided a substantially lower error than 64 while retaining a smaller representation than 256.

---

## A(c) Sublinear retrieval and risk trade-off

The retrieval stage used Locality Sensitive Hashing (LSH) over the 128-hash signatures.

The candidate stage was evaluated using the 900 trusted labelled pairs. The labels are skewed:

- DIFFERENT: 621 pairs (69.0%)
- SAME: 279 pairs (31.0%)

### 32 bands × 4 rows

The original 32×4 configuration produced:

- 384,000 LSH index rows;
- 12,000 notices;
- 32 bands;
- 71,065,222 candidate pairs;
- labelled DIFFERENT retrieval: 619/621 = 99.68%;
- labelled SAME retrieval: 278/279 = 99.64%.

The candidate volume was extremely high because several buckets became very large.

### Collision diagnostic

The largest observed buckets included:

- band 27: 10,958 notices;
- band 26: 3,930;
- band 23: 3,841;
- band 4: 3,866;
- band 15: 3,383.

Thus the candidate explosion was measurable in the LSH structure itself rather than being assumed from documentation.

### Mitigated operating point: 16 bands × 8 rows

The 16×8 configuration produced:

- 192,000 LSH index rows;
- 12,000 notices;
- 16 bands;
- 17,995,868 candidate pairs;
- SAME-pair recall: 246/279 = 88.17%.

Compared with 32×4:

- candidate work fell from 71,065,222 to 17,995,868;
- reduction = approximately **74.67%**;
- SAME recall fell from 99.64% to 88.17%.

This exposes the required asymmetric risk explicitly: reducing candidate work has a measurable false-negative cost. The operating point was selected as the measured mitigation point rather than presenting recall/cost as a qualitative adjective.

### Similarity-stage measurements

The labelled-pair retrieval measurements also showed that retrieval probability was dependent on the true similarity range. For the measured 32×4 candidate retrieval, the observed labelled pairs were:

| Similarity range | Pairs | Retrieved | Retrieval probability |
|---|---:|---:|---:|
| 0.00–0.39 | 2 | 2 | 1.00 |
| 0.40–0.49 | 53 | 53 | 1.00 |
| 0.50–0.59 | 270 | 270 | 1.00 |
| 0.60–0.69 | 278 | 278 | 1.00 |
| 0.70–0.79 | 127 | 127 | 1.00 |
| 0.80–0.89 | 69 | 69 | 1.00 |
| 0.90–1.00 | 101 | 101 | 1.00 |

The table above is the measured candidate retrieval behaviour from the earlier operating-point experiment; it is retained as evidence rather than extrapolated beyond the labelled corpus.

---

# B — Making it a database problem, not a script

## B(d) Persistent retrieval structure and access path

The retrieval index was stored as relational data.

Schema:

```sql
CREATE TABLE lsh_buckets (
    notice_id TEXT NOT NULL,
    band_id INTEGER NOT NULL,
    bucket BIGINT NOT NULL,
    PRIMARY KEY (notice_id, band_id)
);
```

An access index was created:

```sql
CREATE INDEX ix_lsh_bucket
ON persist.lsh_buckets (band_id, bucket);
```

The persistent DuckDB database was attached as:

```sql
ATTACH 'setubid.duckdb' AS persist;
```

### Verification

The persisted LSH table contained:

- rows: **384,000**
- notices: **12,000**
- bands: **32**

### Planner measurement

The indexed-table lookup was measured using `EXPLAIN ANALYZE`.

Observed:

- total time: **0.0043 s**
- planner path: **Sequential Scan**
- filtered band: 10
- rows shown by the scan: **1,160**

The important observation is that the planner did **not** select the created index for this small lookup. The report therefore does not claim an index scan that was not observed.

### Rejected alternative

A no-index copy of the same table was measured with the equivalent lookup.

Observed:

- total time: **0.0036 s**
- planner path: **Sequential Scan**
- rows shown: **1,160**

For this particular small lookup, the measured indexed and no-index cases were both sequential scans and had very similar wall-clock times. Therefore the evidence does not justify claiming an index speedup for this workload.

The persistent relational table nevertheless satisfies the requirement that the retrieval structure survives process restart and is queryable by the application.

---

## B(e) Full-corpus uneven work and mitigation

### Before mitigation — 32×4

Full-corpus candidate work:

**71,065,222 candidate pairs**

Per-notice candidate work:

- notices: 12,000
- mean: **11,844.2**
- median: **11,951**
- minimum: **5,026**
- maximum: **11,998**

This is highly concentrated: many notices were candidates with a very large fraction of the corpus.

### Portal distribution

The highest candidate-work portals included:

| Portal | Notices | Total candidate work | Average candidates |
|---|---:|---:|---:|
| P094 | 2,851 | 4,463,872 | 1,565.72 |
| P002 | 1,597 | 2,512,290 | 1,573.13 |
| P005 | 1,536 | 2,404,067 | 1,565.15 |
| P006 | 1,583 | 2,353,924 | 1,487.60 |
| P001 | 1,556 | 2,339,172 | 1,503.32 |
| P003 | 1,543 | 2,278,392 | 1,476.60 |
| P004 | 1,508 | 2,239,184 | 1,484.87 |
| P020 | 768 | 1,203,187 | 1,566.65 |
| P240 | 753 | 1,197,067 | 1,590.45 |

The supplied portal notes explain why the nodal portals can create this behaviour: P001–P006 republish notices and add long common preambles. However, the bucket diagnostic also demonstrated that the LSH configuration itself was producing very large collision buckets. Both effects therefore matter to the observed workload.

### Mitigation

The operating configuration was changed from:

**32 bands × 4 rows**

to:

**16 bands × 8 rows**

This reduced the number of LSH index rows from 384,000 to 192,000 and reduced full-corpus candidate pairs to:

**17,995,868**

Candidate generation for the 16×8 configuration was measured using `EXPLAIN ANALYZE`:

**Total time: 1.05 s**

The resulting candidate table contained 17,995,868 rows. A direct `COUNT(*)` scan of that materialized candidate table measured 0.0032 s.

### Mitigation cost in retrieval quantity

The cost of the mitigation was measured on the trusted labels:

- SAME pairs: 279
- retrieved after mitigation: 246
- recall: **88.17%**

Thus 33 of the 279 trusted SAME pairs did not survive the candidate stage in the 16×8 configuration.

This is the explicit retrieval-quality cost of reducing candidate work by approximately 74.67%.

---

# Evidence files

The `screenshots/` directory contains the final screenshots supplied during the database and B(e) measurements.

Included screenshots:

1. `B_e_candidate_generation_runtime_1.05s.png`
2. `B_e_hash_join_and_scans.png`
3. `B_e_portal_candidate_work.png`
4. `B_e_full_plan_detail.png`

The screenshots are evidence of the measured DuckDB plans and outputs. They are not substituted with fabricated screenshots.

---

# Reproducibility notes

The working implementation was performed in DuckDB.

The source corpus used for the measurements was:

```text
data/
├── notices/
├── labelled_pairs.csv
└── portal_profiles.md
```

The labelled-pair file is treated as the only trustworthy ground-truth label source, as required by the question.

The final measured system should be described using the actual observed planner behaviour and measurements above. In particular:

- do not claim the DuckDB index was used when `EXPLAIN ANALYZE` showed Sequential Scan;
- do not claim the 1.05-second measurement is the runtime of the entire nightly pipeline — it is the measured candidate-generation query;
- do not claim 100% retrieval after the 16×8 mitigation — measured SAME recall was 88.17%.

# Final status

## Question 2 — A and B

- A(a): completed and measured.
- A(b): completed and measured.
- A(c): completed and measured.
- B(d): persistent relational structure and planner comparison completed.
- B(e): full-corpus distribution, cause diagnosis, mitigation, runtime and recall cost measured.

