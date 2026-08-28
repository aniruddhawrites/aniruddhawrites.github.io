---
layout: post
title: "When Fast Ingestion Creates Slow Analytics"
seo_title: "When Fast Ingestion Creates Slow Analytics | Iceberg, S3 & Redshift Spectrum"
subtitle: "Why can a fast streaming pipeline create slow analytics? Explore the S3 small-file problem and how Parquet, compaction, Apache Iceberg, and Redshift Spectrum improve analytical data layouts."
cover-img: /assets/img/architecture-diagram.png
thumbnail-img: /assets/img/architecture-diagram.png
share-img: /assets/img/architecture-diagram.png
tags: [Amazon Redshift, Redshift Spectrum, Amazon S3, Apache Iceberg, Parquet, Data Engineering, Data Architecture, Performance Engineering]
---

# When Fast Ingestion Creates Slow Analytics

> Solving the S3 Small-File Problem with Apache Iceberg, Compaction, and Amazon Redshift Spectrum
>
> *A reference architecture case study*

---
## Author's note

> This is a reference architecture, not a specific customer case study. Workload characteristics and file sizes are illustrative; performance claims should be validated through controlled benchmarking. The article deliberately separates the roles of Iceberg, Parquet, and compaction rather than treating Iceberg as a universal performance solution.

---

### The Pipeline Was Fast. The Queries Were Slow.

Every metric on the ingestion side looked healthy. Events from IoT devices, mobile apps, and web clients streamed into an Amazon S3 data lake continuously, durably, with low write latency. The pipeline was doing exactly what it was built to do.

The dashboards built on top of that data told a different story. Queries that used to return in seconds were taking minutes. Analysts filed tickets. Someone suggested adding more Redshift compute. It didn't help much.

The two halves of this system were never in conflict about correctness — the data was all there, intact and durable. They were in conflict about *shape*. A streaming system optimizes for low-latency, continuous, durable writes. An analytical engine optimizes for the opposite: fewer, larger, well-organized files it can plan around efficiently. Nobody had designed for the second half of that sentence, because nothing about the ingestion pipeline was wrong on its own terms.

This is a reference architecture, not a specific customer engagement — the workload, file counts, and query patterns below are illustrative, built to be representative of a common streaming-ingestion pattern. Where a claim is backed by AWS or Apache documentation, it's cited. Where a number is illustrative rather than measured, that's stated explicitly. No benchmark result in this article is fabricated; where we don't have a measured number, we say so and describe how you'd get one.

---

### 1. The Architecture at a Glance

The system has three stages, and the story of this article is what happens to the data's physical shape as it moves through them.

**Ingestion.** IoT devices, mobile apps, and web applications emit events continuously. A streaming layer writes them to an Amazon S3 landing zone as JSON objects, favoring low write latency over any downstream analytical concern.

**The bottleneck.** Over hours and days, that landing zone accumulates an enormous number of small JSON objects — illustratively, tens or hundreds of thousands of files averaging around 10 KB each. Amazon Redshift Spectrum, querying this layout as external tables, absorbs a large and largely avoidable planning and scanning overhead before it ever reaches a useful byte of data.

**The optimization.** Apache Iceberg is introduced as a table format and metadata layer over the same S3 storage. AWS Glue or Spark jobs perform compaction — rewriting many small JSON files into fewer, larger Parquet files — and commit the result as a new Iceberg table snapshot. Redshift Spectrum then queries a physically consolidated, columnar, metadata-described table instead of a sprawling directory of small text files.

Three stages, three different concerns: how data arrives, how data is stored for analysis, and how a query engine plans and executes against it. Keeping those three concerns distinct is the throughline of everything that follows.

---

### 2. The Incident: Fast Ingestion, Slow Analytics

The presenting symptom was simple: dashboard queries against recent event data were getting progressively slower, while the ingestion pipeline showed no degradation at all. Write throughput was steady. S3 PUT success rates were normal. No errors, no backpressure, no obvious infrastructure fault.

That combination — a healthy write path and a degrading read path over the *same* data — is the signature of a physical layout problem, not a capacity problem. If the cluster were simply undersized, you'd expect both ingestion and query performance to degrade together as volume grew, or you'd expect the problem to be fixable by scaling compute. Here, scaling compute produced only marginal improvement, which is itself a diagnostic clue worth taking seriously: throwing more compute at a query engine doesn't reduce the number of small objects it has to enumerate, open, and parse before it reaches useful data. It just processes the overhead in parallel, faster, up to a point.

![From Fast Ingestion to Slow Analytics](https://aniruddhawrites.github.io/assets/img/from-fast-ingestion-to-slow-analytics.png)

> * dashboard queries against recent event data were getting progressively slower
 

---

### 3. The Investigation

The investigation deliberately avoided jumping straight to "we need Iceberg." It started with measurement.

The team looked at:

- **Query latency**, broken into planning time and execution time separately, not as one combined number.
- **Files scanned per query**, compared against the amount of data a query's predicates should have actually required.
- **Bytes scanned**, to distinguish "we're touching too many objects" from "we're touching too much data."
- **S3 object layout** — file count per partition, average object size, and the distribution around that average.
- **File format** — JSON, uncompressed, record-oriented.
- **Partition structure** — how data was organized under the landing prefix.
- **Query plans** — what Redshift Spectrum's planner was actually doing before scan work began.

This separation matters because file-count reduction and query-latency reduction are not the same claim, and conflating them is one of the more common mistakes in how this class of problem gets diagnosed and communicated afterward. A compaction job can drastically cut the number of files a query touches while only moderately reducing the bytes scanned, particularly for queries that aren't highly selective. AWS's own Spectrum performance guidance ties query performance to file size, file format, and partition design as separate, interacting levers — not one lever with one dial. [Amazon Redshift Spectrum query performance, AWS Documentation]

---

### 4. Root Cause #1: The Small-File Explosion

Small files, on their own, aren't the problem. A single 10 KB file is nothing. The problem is what accumulates when a streaming pipeline optimized for write latency runs continuously against the same landing zone for hours and days without any downstream consolidation step.

![The Small-File Tax](https://aniruddhawrites.github.io/assets/img/The-Small-File-Tax.png)

> * continuous streaming creates small-file bottlenecks

It's worth being precise about *why* a large number of small objects is expensive for an analytical engine, because "more files means more requests, therefore slow" is an oversimplification that doesn't hold up to scrutiny and, more importantly, isn't necessary to make the argument. The actual mechanism has several contributing layers:

- **Object and file enumeration.** Before any scanning happens, the query engine has to discover which objects exist under the relevant prefixes or partitions.
- **Metadata discovery and planning overhead.** Building an execution plan against a directory structure with an extremely high object count takes measurably longer than planning against a smaller number of well-described files.
- **Object access overhead.** Each object that participates in a scan carries its own fixed cost — opening it, reading whatever header or footer metadata the format requires, and closing it — independent of how much useful data it contains.
- **Task scheduling.** A distributed query engine has to schedule work across many small units instead of fewer, larger, more efficient ones, and scheduling overhead doesn't scale to zero.
- **Parsing cost.** Record-oriented text formats like JSON have to be parsed row by row, with no ability to skip irrelevant fields during the read.
- **Reduced parallelism efficiency.** Extremely small files can create more scheduling overhead than benefit, because the fixed cost per file competes with the actual work being done.

The governing principle is this: **the smaller the object, the larger the fixed overhead becomes relative to the amount of useful data it contains.** At 10 KB per object and tens of thousands of objects, an analytical engine can spend a disproportionate share of total query time on overhead that has nothing to do with the actual predicates in the query.

It's also worth being explicit about what *isn't* the bottleneck here: Amazon S3 itself is not slow, and it is not inherently poorly suited to storing many small objects — S3 is designed to handle massive object counts at scale. The bottleneck is the interaction between object granularity, file format, and how an analytical query engine has to plan and execute a scan across that layout. That distinction matters, because "S3 is slow" and "this physical layout is expensive to scan" point to very different fixes.

---

### 5. Root Cause #2: JSON Is the Wrong Physical Shape for Analytics

JSON is not a bad format. It's an excellent format for what it's usually used for: representing individual records in application and integration contexts where flexibility and human-readability matter more than analytical scan efficiency. The problem isn't that JSON is deficient — it's that JSON's design center and an analytical engine's access pattern point in different directions.

JSON is text-based and record-oriented: to read any single field from any single record, the parser generally has to read and interpret the whole record. It isn't organized around columnar access, which means an analytical query that only needs three columns out of forty still pays the cost of parsing all forty for every record it touches. It's important not to overstate this — JSON isn't incompressible, and general-purpose compression can and does reduce its size on disk. But compression on a record-oriented text format doesn't grant the ability to skip columns during a scan, which is where a large share of analytical efficiency actually comes from.

Parquet addresses this directly. It's columnar, so a query that touches three columns can, subject to how the file was written and how the engine plans the scan, avoid reading data for the other thirty-seven. It supports efficient compression schemes suited to columnar layouts. And Parquet files carry embedded statistics — min/max values, null counts — that a query engine's planner can use to skip entire row groups without reading them at all.

It's worth stating the underlying architectural principle plainly, because the two problems are frequently conflated in less careful treatments of this topic: **reducing file count addresses file-level overhead; moving to a columnar format addresses per-byte scan efficiency.** They are related, additive improvements, not the same improvement described twice. A workload that fixes file count but stays in JSON still pays the columnar-access penalty. A workload that moves to Parquet but keeps producing thousands of tiny files still pays the object-overhead penalty. Both matter, independently.

---

### 6. Root Cause #3: The Need for Metadata-Aware Table Management

The third root cause is less visible than file size or file format, and it's the one most often skipped: even after you decide to compact files and move to Parquet, someone has to safely manage the *process* of rewriting a live, continuously growing dataset without breaking readers, losing data, or leaving the table in an inconsistent state mid-rewrite.

This is a genuinely different problem from "what format should the files be" or "how many files should there be." It's a table-management problem: how do you track which files currently constitute the authoritative state of a table, how do you let a rewrite job replace a large batch of files atomically, and how do you let readers see a consistent view of the table even while a compaction job is actively running against it. A raw S3 prefix with no table format has no native answer to any of these questions. This is the gap Apache Iceberg is designed to fill, and it's a distinct concern from file size or file format — conflating it with either one is where a lot of otherwise-accurate explanations of this problem start to blur.

---

### 7. The Architectural Intervention

The fix is not "adopt Iceberg" as a single action. It's the deliberate combination of three things that each solve a different part of the problem described above:

**Apache Iceberg** provides the table abstraction — the metadata layer that tracks table state, snapshots, and schema, and that makes atomic, consistent rewrites of the underlying files possible.

![How Iceberg Tracks the Physical Table](https://aniruddhawrites.github.io/assets/img/How-Iceberg-tracks-the-Physical-Table.png align="center")

> * How Iceberg Tracks the Physical Table 

**Parquet** provides the physical file format — columnar storage, efficient compression, and embedded statistics that a query planner can use.

**Compaction**, run as a Spark job on AWS Glue or Amazon EMR, is the process that actually rewrites many small files into fewer, larger ones and commits the result through Iceberg.

These are three separable technologies performing three separable jobs. Iceberg is not a file format — it doesn't itself make files larger, smaller, columnar, or compressed. Parquet is not a table format — it has no concept of snapshots, schema evolution, or atomic multi-file commits on its own. Compaction is not free, automatic, or something either Iceberg or Parquet does by itself — it's compute work that has to be scheduled, run, and paid for. Keeping these boundaries distinct is what separates an accurate explanation of this fix from a marketing summary of it.

---

### 8. How Iceberg Organizes the Table

Iceberg's contribution is best understood as a metadata hierarchy sitting above the physical Parquet files in S3:

```text
Iceberg Table
      ↓
Snapshot
      ↓
Manifest List
      ↓
Manifests
      ↓
Data Files
      ↓
Parquet Objects in S3
```

A **snapshot** represents the complete, consistent state of the table at a point in time. Each snapshot points to a **manifest list**, which in turn references one or more **manifests** — files that track metadata about groups of data files, including partition values and, when generated, column-level statistics. The manifests point to the actual **data files**: the Parquet objects physically stored in S3.

This hierarchy is a conceptual and logical model for how Iceberg organizes table state and enables metadata-driven planning — it is not a literal claim about the exact sequence or count of network operations Redshift Spectrum performs for any given query. What it gives a query engine is the ability to plan from structured metadata about which files are relevant to a query, rather than having to enumerate an entire object listing from S3 directly. AWS's documentation on querying Iceberg tables from Redshift describes this metadata-driven planning model, and separately notes that AWS Glue-generated statistics for Iceberg tables help Redshift's optimizer reduce the number of files it needs to scan. [Using Apache Iceberg tables with Amazon Redshift, AWS Documentation; Accelerate query performance with Apache Iceberg statistics on the AWS Glue Data Catalog, AWS Big Data Blog]

Iceberg also provides atomic commits and snapshot isolation: a compaction job can replace a large batch of small files with a smaller batch of large ones as a single atomic operation, and readers querying the table mid-rewrite see either the old consistent state or the new one — never a partial, broken mix of both. That property is what makes it *safe* to continuously reorganize a live analytical dataset, which is a genuinely different contribution from "makes files bigger" or "makes queries faster."

---

### 9. How Compaction Changes the Physical Layout

Compaction is the mechanical process that actually does the file rewriting. Its lifecycle looks like this:

```text
Small Files
     ↓
Compaction Job (Glue / Spark)
     ↓
Read Existing Data Files
     ↓
Rewrite / Merge into Larger Parquet Files
     ↓
Atomic Iceberg Commit
     ↓
New Table Snapshot
```

The job reads a set of existing small files, rewrites their combined contents into a smaller number of larger Parquet files, and commits that rewrite as a new Iceberg snapshot. The old files aren't deleted the instant the new ones are written — they remain part of prior snapshots until retention and expiration policies clean them up, which is deliberate: it's what allows time-travel queries and safe rollback if something in the rewrite went wrong.

Compaction is recurring, not one-time, because ingestion doesn't stop. A landing zone under continuous write load will keep accumulating small files after every compaction run, which raises an operational question that's easy to get wrong: how often should compaction actually run?

![How Compaction Changes the Physical Layout](https://aniruddhawrites.github.io/assets/img/How-Compaction-Changes-the-Physical-Layout.png align="center")

> * Snapshots preserve history for rollbacks

"Run it daily" is not a universal answer, and treating it as one is one of the more common mistakes in production deployments of this pattern. The right cadence depends on the rate of file accumulation, the query workload's freshness requirements, the cost of running the compaction job itself, and how partitions are behaving under load. A workload accumulating files faster than a daily job can absorb needs more frequent, threshold-based triggering — compact when a partition crosses a file-count or fragmentation threshold, not on a fixed calendar regardless of actual need. A workload with a healthy, slowly-growing partition doesn't need the compute cost of a daily rewrite it doesn't yet require. AWS's guidance on Iceberg optimization on AWS specifically frames compaction as an ongoing maintenance operation to be tuned to the workload, not a one-time fix or a fixed-schedule chore. [Apache Iceberg optimization: Solving the small files problem in Amazon EMR, AWS Big Data Blog]

---

### 10. What Changes at Query Time

With the intervention in place, Redshift Spectrum is querying a table with a materially different physical and metadata profile.

It's important to state precisely what's actually happening, since this is exactly the point where oversimplified claims tend to creep in. Redshift Spectrum does not perform "a single metadata lookup" and then instantly access all relevant data — that phrase overstates how metadata-driven planning works and obscures the real mechanism, which is a structured, hierarchical process of using Iceberg's snapshot and manifest metadata to identify a relevant, pruned set of data files before scan work begins. Nor is it accurate to describe the underlying mechanism as Spectrum issuing "one HTTP GET request for every file" as the defining explanation of the small-file problem — that's a plausible-sounding simplification, but stating it as a precise fact about Spectrum's execution path isn't something the available documentation directly confirms, and the actual overhead mechanism is broader than any single network-call framing: enumeration, planning, parsing, and scheduling overhead all contribute, as described in Root Cause #1.

What can be stated with confidence, grounded in AWS's own guidance: Spectrum performance benefits from columnar file formats, from file sizes generally at or above roughly 64 MB rather than KB-scale objects, from well-designed partitioning that supports pruning, and — specifically for Iceberg tables — from Glue-generated statistics that help the cost-based optimizer reduce the number of files it needs to scan. [Amazon Redshift Spectrum query performance, AWS Documentation; Data files for queries in Amazon Redshift Spectrum, AWS Documentation]

---

### 11. Before vs. After

**Before**

```text
Applications (IoT / mobile / web)
     ↓
Streaming ingestion
     ↓
Amazon S3 landing zone
     ↓
Many small JSON objects (~10 KB, illustrative)
     ↓
Amazon Redshift Spectrum
     ↓
High enumeration, planning, and parsing overhead
```

**After**

```text
Applications (IoT / mobile / web)
     ↓
Streaming ingestion
     ↓
Amazon S3 landing zone
     ↓
Apache Iceberg table
     ↓
AWS Glue / Spark compaction (threshold-based)
     ↓
Fewer, larger Parquet files, committed as new snapshots
     ↓
Amazon Redshift Spectrum
```
| Aspect          | Before          | After                |
| --------------- | --------------- | -------------------- |
| Compression     | Text-oriented   | Columnar compression |
| Scan efficiency | Lower           | Higher               |
| Column access   | Record-oriented | Columnar             |

Ingestion behavior is unchanged in this picture — that's deliberate. The applications still write the way they were always going to write. What changed is entirely downstream, in how that data is subsequently organized for the workload that consumes it.


![Same Data. Different Physical Layout.](https://aniruddhawrites.github.io/assets/img/Same-Data-Different-Physical-Layout.png align="center")

> * Ingestion remains unchanged; optimization happens downstream

---

### 12. Benchmarking the Improvement

Here is the point where this article deliberately declines to hand you a headline number.

No specific production benchmark result — no percentage, no "Nx faster," no before/after latency figure — is presented as measured fact in this article, because none was independently verified for this reference scenario. What follows instead is a reproducible methodology, along with the hypotheses that AWS's own documentation supports, clearly separated from anything that would require an actual measured run to state as fact.

**Datasets.** Run the comparison at more than one scale — for example 100 GB, 1 TB, and 10 TB — since overhead effects at small scale don't always predict behavior at larger scale.

**Variants to compare:**

| Variant | Format | Avg file size | Table layer |
|---|---|---|---|
| Baseline | JSON | ~10 KB | Raw external table, no Iceberg |
| Optimized (Parquet only) | Parquet | 64–128 MB | No Iceberg / minimal metadata |
| Optimized (Parquet + Iceberg) | Parquet | 64–128 MB | Iceberg table + Glue-generated statistics |

Including the middle variant matters — it's what actually lets you separate the contribution of file format and file size from the contribution of Iceberg's metadata layer, rather than crediting one intervention for the combined effect of two.

**Representative query workload:** a time-range filter, a highly selective predicate, an aggregation, a multi-column projection, and a partition-pruned query — covering the query shapes that behave differently under fragmented versus consolidated layouts.

**Metrics, measured separately, not as one combined number:** total query latency, planning latency, execution latency, bytes scanned, files scanned, observable S3 request patterns where available, relevant Spectrum-side query metrics, compute consumption, and cost per query.

**Controlled between runs:** identical SQL text, identical Redshift cluster configuration and concurrency, identical S3 region, identical partition scheme, identical underlying row content, identical compression settings.

**Reporting discipline:** report medians and p95s across repeated runs, not single best-case numbers, and report each metric layer separately rather than collapsing everything into one latency figure.

**What the documentation supports as a reasonable hypothesis, not a guaranteed result:** that JSON at small file sizes will show the highest planning and scan overhead of the three variants; that moving to Parquet will materially reduce per-byte scan cost due to columnar access and compression; and that adding Iceberg with Glue-generated statistics will further reduce files scanned for selective queries, because the optimizer has structured statistics to prune against rather than relying on broader file discovery. [Amazon Redshift Spectrum query performance, AWS Documentation; Accelerate query performance with Apache Iceberg statistics on the AWS Glue Data Catalog, AWS Big Data Blog]

If you run this methodology against your own workload and observe a specific speedup — 10x, 50x, 100x, whatever the number turns out to be — that number is a property of your workload, your query selectivity, and your data shape. It is not a property of Iceberg as a technology, and it should be reported alongside the conditions that produced it, not detached from them.

---

### 13. The Cost of Optimization

It would be dishonest to present this as a pure win. Compaction is not free, and pretending otherwise undermines the credibility of everything else in this article.

**Costs introduced:**
- Glue or Spark compute to run the compaction jobs themselves.
- The rewrite operation's own I/O — reading the small files being consolidated, writing the new Parquet output.
- Temporary storage overhead during the rewrite.
- Operational complexity: something new to monitor, alert on, and operate.
- A freshness trade-off — data isn't reflected in the optimized table until it's been through a compaction cycle, so there's an inherent lag between arrival and full analytical optimization, sized by whatever cadence you choose.

**Costs potentially reduced:**
- Query-side compute spent on unnecessary scanning and parsing.
- The operational and business cost of slow dashboards — which is real but harder to put a precise number on than compute spend.
- Reduced request and object-access overhead per query.

The governing economic principle, stated plainly: **compaction is worthwhile when the recurring analytical cost and latency caused by a fragmented physical layout exceed the cost of maintaining an optimized one.** For a landing zone that's rarely queried directly, that calculus might not favor aggressive compaction at all. For a table backing high-frequency dashboards under continuous analytical load, it usually will. That's a workload-specific judgment, not a default answer.

---

### 14. Production Guardrails

A physical-layout optimization introduces its own failure modes. None of these are exotic — they're the ordinary operational reality of running compaction as a continuous process against a continuously growing dataset.

| Failure mode | Why it happens | Detection | Mitigation |
|---|---|---|---|
| Compaction lag | Files accumulate faster than compaction jobs can process them | File count and freshness-lag trending upward over time | Threshold-based triggers instead of fixed schedules; sufficient rewrite capacity |
| Compaction storms | Multiple large rewrites triggered simultaneously, competing for compute | Sudden spikes in job concurrency and compute cost | Scheduling controls, rate limiting, partition-scoped optimization |
| Over-partitioning | Partition design is too fine-grained, recreating tiny files inside every partition | High file count per partition despite low file count globally | Coarser partition granularity aligned to actual query patterns |
| Wrong target file size | Files end up too small (overhead persists) or too large (parallelism and rewrite cost suffer) | File-size distribution and p95 tracked against target | Tune compaction thresholds against workload behavior, not a fixed constant |
| Snapshot accumulation | Old snapshots retained indefinitely, growing storage and metadata overhead | Snapshot count and age trending upward | Retention and expiration policies |
| Orphaned files | Files left behind by failed or partial rewrites, no longer referenced by any snapshot | Mismatch between catalog references and actual S3 objects | Scheduled cleanup / orphan-file removal jobs |
| Failed compaction jobs | Job failures leave optimization state stale without anyone noticing | Job failure alerting, freshness-lag monitoring | Retries with backoff, alerting tied to lag thresholds, not just job status |
| Raw-zone querying | Analysts or downstream tools query the unoptimized landing zone directly, bypassing the curated table | Query source/access-pattern auditing | Access controls and catalog separation between raw and curated layers |

None of these are reasons to avoid the architecture. They're the reason it needs monitoring rather than being treated as a one-time migration.

---

### 15. When You Don't Actually Need Iceberg

This is worth stating directly, because it's the section most often missing from articles that are ultimately trying to sell a technology rather than solve a problem: **Iceberg is not the first thing to try.**

The decision order, roughly in ascending order of complexity:

**Fix ingestion batching or windowing first.** If the root problem is that the producer is flushing far more frequently than necessary, adjusting write batching or windowing at the source can eliminate the small-file problem before it ever reaches S3. This is the cheapest fix and the one most often skipped in favor of a downstream architectural solution.

**Write directly to Parquet, without a table format**, if the dataset is relatively simple, doesn't need concurrent readers and writers reasoning about consistent snapshots, and doesn't need to evolve its schema or partitioning over time.

**Run scheduled or threshold-based compaction without introducing a table format**, where the operational need is genuinely just "consolidate files periodically" and the added benefits of snapshot isolation, atomic commits, and schema evolution aren't required.

**Adopt Iceberg with compaction** when the dataset genuinely needs table abstraction: concurrent readers and writers operating safely against an evolving dataset, schema evolution, partition evolution, atomic multi-file commits, time travel, and long-lived governance over an analytical table's lifecycle.

The conclusion is not "everyone should use Iceberg." It's **use the simplest architecture that solves the actual problem, and introduce a table format when the dataset's lifecycle actually justifies the complexity it adds.** A batch job that compacts JSON into Parquet on a schedule, with no table format at all, may be entirely sufficient for a dataset that doesn't need concurrent writers or schema evolution. Iceberg earns its place when the table's operational lifecycle demands more than compaction alone can provide.

---

### 16. Architecture Decision Framework

| Approach | Complexity | Solves Small Files? | Adds Table Semantics? | Best Fit |
|---|---|---|---|---|
| Better ingestion batching | Low | Yes | No | The root cause is upstream write behavior, not downstream table management |
| JSON → Parquet (no table format) | Low–Medium | Partially — improves scan efficiency, doesn't inherently limit file count | No | Simple, relatively static analytical datasets without concurrent writers |
| Standalone scheduled/threshold compaction | Medium | Yes | No | Files already exist and need periodic consolidation, without needing snapshot isolation |
| Iceberg + compaction | Medium–High | Yes | Yes | Datasets needing schema/partition evolution, concurrent readers/writers, and long-lived table governance |
| Alternative table format (e.g., Hudi, Delta Lake) | Medium–High | Yes | Yes | Workload patterns favoring a different table format's specific transactional or update-heavy semantics |

This table isn't meant to declare a winner. It's meant to help you place your own workload against the actual axes that matter — how complex is the fix you're willing to operate, and does the dataset genuinely need table-level semantics or just a cleaner physical layout.

---

### 17. Conclusion

The problem this article describes was never really that the data lake contained too much data. It was that the data had been physically organized around the needs of the producer — low-latency, continuous, durable writes — rather than the needs of the analytical consumer, which wants the opposite: fewer, larger, well-described files it can plan around efficiently.

The fix is not "use Iceberg." It's the deliberate combination of appropriate ingestion behavior, an appropriate physical file format, sensible file sizing tuned to the actual workload, a compaction process that runs on a cadence matched to real accumulation and freshness needs, and a metadata-aware table management layer introduced when — and only when — the dataset's lifecycle actually justifies it.

Design the way data arrives for ingestion. Design the way data is stored for analytics. And reach for a table format when the lifecycle of that analytical data genuinely demands one — not because it's the technology attached to the fastest-sounding benchmark number you've seen.

---

## Sources

- Amazon Redshift Spectrum query performance — AWS Documentation: https://docs.aws.amazon.com/redshift/latest/dg/c-spectrum-external-performance.html
- Data files for queries in Amazon Redshift Spectrum — AWS Documentation: https://docs.aws.amazon.com/redshift/latest/dg/c-spectrum-data-files.html
- Using Apache Iceberg tables with Amazon Redshift — AWS Documentation: https://docs.aws.amazon.com/redshift/latest/dg/querying-iceberg.html
- Accelerate query performance with Apache Iceberg statistics on the AWS Glue Data Catalog — AWS Big Data Blog: https://aws.amazon.com/blogs/big-data/accelerate-query-performance-with-apache-iceberg-statistics-on-the-aws-glue-data-catalog/
- Introducing AWS Glue Data Catalog automation for table statistics collection — AWS Big Data Blog: https://aws.amazon.com/blogs/big-data/introducing-aws-glue-data-catalog-automation-for-table-statistics-collection-for-improved-query-performance-on-amazon-redshift-and-amazon-athena/
- Apache Iceberg optimization: Solving the small files problem in Amazon EMR — AWS Big Data Blog: https://aws.amazon.com/blogs/big-data/apache-iceberg-optimization-solving-the-small-files-problem-in-amazon-emr/
- Optimizing read performance — AWS Prescriptive Guidance for Apache Iceberg on AWS: https://docs.aws.amazon.com/prescriptive-guidance/latest/apache-iceberg-on-aws/best-practices-read.html
- Using Iceberg workloads in Amazon S3 — AWS Prescriptive Guidance: https://docs.aws.amazon.com/prescriptive-guidance/latest/apache-iceberg-on-aws/best-practices-workloads.html
- Best practices for using Amazon Redshift Spectrum — AWS Prescriptive Guidance: https://docs.aws.amazon.com/prescriptive-guidance/latest/query-best-practices-redshift/best-practices-redshift-spectrum.html
- 10 Best Practices for Amazon Redshift Spectrum — AWS Big Data Blog: https://aws.amazon.com/blogs/big-data/10-best-practices-for-amazon-redshift-spectrum/
- Apache Iceberg Table Specification: https://iceberg.apache.org/spec/

---

## About the Author

**Aniruddha Banerjee** is a Data Platform Architect, BI and Analytics leader, and technology writer with nearly two decades of experience delivering enterprise data and analytics solutions across global organizations.

His writing explores the intersection of data, AI, digital transformation, architecture, and technology leadership.

Through **Aniruddha Writes**, he shares practical insights, long-form essays, and reflections on how technology shapes organizations, industries, and society.

[LinkedIn](https://www.linkedin.com/in/ruddhani/) · [GitHub](https://github.com/aniruddhawrites) · [Medium](https://aniruddhawrites.medium.com/)

#DataEngineering #AWS #ApacheIceberg #AmazonS3 #RedshiftSpectrum #DataArchitecture
