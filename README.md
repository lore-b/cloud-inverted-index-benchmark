# cloud-inverted-index-benchmark
Cloud-scale evaluation of inverted index construction with Hadoop MapReduce (incl. in-mapper), Apache Spark, and non-parallel Python across 5 MB–2 GB inputs with time/shuffle/CPU/memory metrics.

# Evaluating Distributed Processing on Cloud: Hadoop vs Spark for Inverted Index

Cloud-scale benchmark of **inverted index** construction with:
- **Hadoop MapReduce** (standard and **in-mapper combining**)
- **Apache Spark** (Scala or PySpark)
- **Non-parallel Python** baseline

We evaluate **wall-clock time**, **shuffle I/O**, **CPU**, and **memory** across inputs from **5 MB → 2 GB** on a **3-node cluster** (2 vCPU, 6.8 GB RAM per node). The repo includes scripts and a notebook to reproduce and query results.

---

## Contents

- [Project structure](#project-structure)
- [Requirements](#requirements)
- [Dataset](#dataset)
- [Implementations](#implementations)
- [Quick start](#quick-start)
- [Running queries](#running-queries)
- [Experiments & metrics](#experiments--metrics)
- [Result discussion (to replicate)](#result-discussion-to-replicate)
- [Reproducibility](#reproducibility)
- [Configuration notes](#configuration-notes)
- [License](#license)

---


---

## Requirements

- **Java 8+** and **Hadoop 3.x** for MapReduce
- **Spark 3.x** (Scala SBT/Maven or PySpark)
- **Python 3.10+** with packages in `requirements.txt`
- (Optional) **Docker/Compose** for a local pseudo-cluster

---

## Dataset

Texts sourced from **Project Gutenberg**. One large file (~2.1 GB) plus additional books are **split** into blocks of **500 KB, 1 MB, 5 MB, 25 MB, 50 MB** (total ≈ 61 files) to vary input sizes.

- Use `scripts/scraping/*.py` to download data.
- Use `scripts/split_file.py` to create the input blocks.
- Keep only tiny samples in `data/sample/` under version control; place full data under `data/raw/` or `data/splits/` (git-ignored).

---

## Implementations

### Hadoop MapReduce (standard)
- **Mapper**: normalize, tokenize, remove stopwords; emit `word@filename → 1`.
- **Combiner**: local sum per mapper.
- **Reducer**: aggregate to `filename:count` lists per word.

### Hadoop MapReduce (in-mapper combining)
- Accumulate counts **inside** the mapper; emit at `cleanup()` to reduce shuffle volume.

### Apache Spark
- Load as `(filename, text)` (`wholeTextFiles` or equivalent).
- Cleaning + tokenization; **local combining** via `mapPartitions`.
- Global `reduceByKey` and grouping to inverted index format.

### Python baseline (single node)
- Same preprocessing pipeline for comparability; suited for small inputs.

---
## Quick start

### 1) Prepare the data
```bash
# Download a small sample (or the full corpus)
python scripts/scraping/download_from_dir.py --out data/raw/

# Split a large file into blocks of varying sizes
python scripts/split_file.py --input data/raw/big.txt --sizes 512K 1M 5M 25M 50M --out data/splits/
```

### 2) Hadoop MapReduce
```bash
# Build (Maven)
cd hadoop && mvn -q package && cd -

# Run (standard)
hadoop jar hadoop/target/inverted-index.jar \
  org.invertedindex.Driver \
  /path/input /path/output --reducers 2

# Run (in-mapper combining)
hadoop jar hadoop/target/inverted-index.jar \
  org.invertedindex.Driver \
  /path/input /path/output_inmapper --reducers 2 --inmapper
```

### 3) Spark (Scala)
```bash
# If you packaged a fat jar with your main class:
spark-submit --class org.invertedindex.Main \
  spark/target/inv-index-spark.jar /path/input /path/output_spark
```

### 3.1) Spark (PySpark)
```
spark-submit spark/inv_index.py /path/input /path/output_spark
# Use mapPartitions for local combining, then reduceByKey
```

### Python baseline
```
pip install -r requirements.txt
python python/inverted_index.py --input data/sample --output data/output_py
```

## Runnign queries
Search documents containing a term or all terms in a phrase:
```
python python/query_script.py \
  --index /path/to/index_output \
  --query "architecture medicine"
```

---
## Experiments & metrics

**Recommended grid:**

- **Input sizes:** 5 MB, 50 MB, 200 MB, 1 GB, 2 GB  
- **Hadoop:** vary number of reducers; compare *standard* vs *in-mapper*  
- **Spark vs Hadoop vs Python:** align resources (executors/cores/memory vs reducers)

**Metrics:**
- **Wall-clock time** (end-to-end)
- **Shuffle read/write** (Spark/Hadoop counters)
- **CPU% / memory** (per executor/task; export to CSV)
- **# tasks / stages** (for overhead insight)

> Aggregate each configuration over **≥ 3 runs** and report **mean ± std**.

---

## Result discussion (to replicate)

- In-mapper combining typically **reduces shuffle** and improves total time, especially on larger inputs.  
- For **small inputs** (≤ ~1 GB), single-node Python may compete due to lower framework overhead; at **2 GB** the benefits of distributed engines emerge and **Spark** often outperforms Hadoop—*provided resources are tuned*.  
- Sensitivity to **partitioning**, **compression**, and **serializer** settings can dominate algorithmic differences; document configs alongside results.

> ⚖️ **Fair comparison:** if you change Hadoop reducers, also tune Spark executors/cores/memory accordingly.

---

## Reproducibility

- All plotting and table generation lives in `notebooks/CloudPerfAnalysis.ipynb`.  
- Seed random components where applicable.  
- Persist raw counters (Spark/Hadoop) to CSV/Parquet and **version the scripts**, not large outputs.

---

## Configuration notes

Keep environment settings explicit in the README or a `configs/` folder:

- **Hadoop:** `mapreduce.job.reduces`, compression, block size  
- **Spark:** `--num-executors`, `--executor-cores`, `--executor-memory`, serializer, shuffle service

> Avoid committing large datasets; keep small `data/sample/` and scripts to regenerate.



