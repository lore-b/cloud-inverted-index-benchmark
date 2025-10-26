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


