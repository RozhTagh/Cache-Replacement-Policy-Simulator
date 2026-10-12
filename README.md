# Cache Replacement Policy Simulator

**A trace-driven Python study of cache replacement policies for storage I/O workloads.**

This project implements and compares six cache-management strategies using real block-level storage access traces. It was developed as an **Operating Systems course project** to investigate how recency, frequency, adaptive replacement, and knowledge of future accesses affect cache hit rates.

The original experiment evaluated **four datasets: `A42.csv`, `A108.csv`, `A129.csv`, and `A669.csv`**. To keep the public repository focused and manageable, **only the A42 trace is shared, as a compressed ZIP archive (`A42.zip`)**. Extract the archive to obtain `A42.csv` before running the simulator. Results from all four original experiments are retained in the report and summarized below.

The accompanying [project report](Document.pdf) (in Persian) describes the methodology, implementation, and detailed results.

## Highlights

- **Six policies:** LRU, N-Hit, N-Hit + LRU, ARC, LARC, and Belady's Optimal.
- **Real trace data:** evaluates ordered block read/write requests rather than synthetic cache accesses.
- **Workload analysis:** compares hit and miss counts for reads, writes, and all requests.
- **Experiment controls:** supports time-window selection, an address-range-based cache-capacity rule, and configurable admission thresholds.
- **Offline benchmark:** includes Belady's Optimal algorithm as a reference upper bound for hit-rate performance under the modeled request stream.

## Algorithms

| Policy | Approach |
| --- | --- |
| **LRU (Least Recently Used)** | Evicts the least recently accessed resident block. |
| **N-Hit** | Tracks access frequency and applies an admission threshold; its eviction order is based on recorded access counts. |
| **N-Hit + LRU** | Applies frequency-based admission with LRU eviction. |
| **ARC (Adaptive Replacement Cache)** | Dynamically balances recency and frequency with resident and ghost lists. |
| **LARC (Lazy Adaptive Replacement Cache)** | Uses an ARC-like structure with deferred maintenance, as implemented for this course project. |
| **Belady's Optimal** | Uses future requests to choose an eviction victim; provides an **offline theoretical benchmark**, not an implementable online policy. |

## Repository structure

```text
cache-replacement-policy-simulator/
├── code.py            # Trace loader and the six policy simulators
├── A42.zip            # Compressed public trace
├── Document.pdf       # Full course report (Persian)
├── requirements.txt   # Python dependencies
└── README.md
```

**Dataset availability:** The original experiment used `A42.csv`, `A108.csv`, `A129.csv`, and `A669.csv`. This repository includes **`A42.zip`**, containing the complete `A42.csv` dataset in compressed form; the other three traces are not redistributed. The example rows below illustrate the format and are not substitutes for the full trace. The extracted `A42.csv` is intentionally excluded from version control to avoid storing duplicate copies.

## Getting started

### Prerequisites

- Python 3.9 or newer (recommended)
- `pip`
- Enough memory to load the selected trace and simulate policies, especially Belady's Optimal

### Installation

```bash
git clone https://github.com/YOUR_USERNAME/cache-replacement-policy-simulator.git
cd cache-replacement-policy-simulator
python -m venv .venv
```

Activate the environment:

**macOS / Linux**

```bash
source .venv/bin/activate
```

**Windows (PowerShell)**

```powershell
.venv\Scripts\Activate.ps1
```

Install dependencies:

```bash
python -m pip install -r requirements.txt
```

On systems that use `python3` or `py` instead of `python`, substitute that command throughout.

### Extract the A42 dataset

The dataset is provided as **`A42.zip`** to reduce the repository size. The simulator reads the **uncompressed** `A42.csv`, not the ZIP file. After cloning the repository and entering its root directory, extract the archive:

**macOS / Linux**

```bash
unzip A42.zip
```

**Windows (PowerShell)**

```powershell
Expand-Archive -Path .\A42.zip -DestinationPath . -Force
```

### Run the A42 experiment

Ensure the **complete, original** `A42.csv` is in the same directory as `code.py`, then run:

```bash
python code.py
```

The script currently selects `A42.csv` automatically, computes its cache capacity, and runs all six policies. It prints the file name, cache capacity, and a statistics tuple for each policy.

No command-line flags are implemented; experiment parameters are changed near the bottom of `code.py`.

## A42 dataset format

`A42.csv` is a **headerless, comma-separated storage trace**. Each record has eight fields. For example, its first rows begin:

```csv
1577808000009424896,-1,4609650688,4096,Read,-1,-1,-1
1577808000009424896,-1,4609654784,4096,Read,-1,-1,-1
1577808000009424896,-1,4609658880,4096,Read,-1,-1,-1
1577808000009424896,-1,4609662976,4096,Read,-1,-1,-1
1577808000009424896,-1,4609667072,4096,Read,-1,-1,-1
1577808000009424896,-1,4609671168,4096,Read,-1,-1,-1
```

The current loader uses these columns (zero-based indices):

| Index | Example value | Meaning in this simulator |
| --- | --- | --- |
| `0` | `1577808000009424896` | Timestamp, used for optional time-window filtering. |
| `1` | `-1` | Unused metadata field. |
| `2` | `4609650688` | Block **offset**, used directly as the cache key. |
| `3` | `4096` | Request size in bytes (4 KiB in this example); not read by the simulator. |
| `4` | `Read` | Operation (`Read` or `Write`) used for hit/miss accounting. |
| `5`, `6`, `7` | `-1` | Remaining metadata; ignored by the simulator. |

The loader does **not** infer a header, group multiple offsets into a single block, or parse extra metadata. In particular, an offset is treated as one cacheable identifier, and the `4096`-byte size appears in the trace but is not used to split requests.

The six example lines are **not** a usable substitute for the full dataset: the cache-sizing formula can produce a zero-sized cache for very small samples.

## Experiment configuration

The default experiment is specified at the end of [`code.py`](code.py):

```python
filename = "A42.csv"
sequence, cache_size = get_sequence(filename, 0, np.inf)

lru_results = sim_lru(sequence, cache_size)
nhit_results = sim_nhit(sequence, cache_size, 5, 0.5)
nhit_lru_results = sim_nhit_lru(sequence, cache_size, 5, 0.5)
arc_results = sim_arc(sequence, cache_size)
larc_results = sim_larc(sequence, cache_size)
oracle_belady_results = sim_oracle_belady(sequence, cache_size)
```

- **Time selection:** `get_sequence(filename, t1, t2)` includes requests with `t1 <= timestamp <= t2`; `0` to `np.inf` covers the entire trace.
- **Cache capacity:** `int((max_offset - min_offset) / (4096 * 100))`, a project-specific approximation corresponding to 1% of the observed address span in 4 KiB units. For the A42 experiment the report records a capacity of **84,807 blocks**.
- **N-Hit threshold:** `5` accesses and a `0.5` trigger ratio in the default configuration.
- **Optional sampling helpers:** `get_random_sequence` and `get_random_sequences` are available for selecting contiguous time regions of a trace.

The trace reader loads the selected requests into memory. For a large file, running the complete experiment—especially the offline optimal baseline—can be memory-intensive.

## Output and metrics

All six simulation functions return a nine-element tuple, in this order:

```text
(
  read_hits, read_misses,
  write_hits, write_misses,
  read_requests, write_requests,
  total_requests, total_cache_hits, total_cache_misses
)
```

The overall cache hit rate is calculated as:

```text
hit_rate (%) = 100 * total_cache_hits / total_requests
```

The hit counts and rates below are **historical results transcribed from the accompanying project report**; they are not newly generated from the six-line example.

## Experimental results

### A42 — the publicly shared trace

The report evaluates **5,087,932 requests** with a cache capacity of **84,807 blocks**.

| Policy | Cache hits | Cache misses | Hit rate |
| --- | ---: | ---: | ---: |
| LRU | 2,924,075 | 2,163,857 | 57.47% |
| **N-Hit** | **3,548,563** | **1,539,369** | **69.74%** |
| N-Hit + LRU | 2,966,954 | 2,120,978 | 58.31% |
| ARC | 3,330,441 | 1,757,491 | 65.46% |
| LARC | 2,948,341 | 2,139,591 | 57.95% |
| **Belady's Optimal (offline)** | **4,431,755** | **656,177** | **87.10%** |

N-Hit achieved the highest reported hit rate **among the online policies** on A42; Belady's Optimal provides the offline benchmark.

### Comparison across all four datasets

| Policy | A42 | A108 | A129 | A669 |
| --- | ---: | ---: | ---: | ---: |
| LRU | 57.47% | 25.03% | 62.73% | 96.70% |
| N-Hit | **69.74%** | 32.94% | 58.84% | 96.16% |
| N-Hit + LRU | 58.31% | 23.61% | 61.73% | 96.58% |
| ARC | 65.46% | 28.53% | 63.51% | 96.64% |
| LARC | 57.95% | **34.03%** | **64.17%** | **96.91%** |
| Belady's Optimal (offline) | 87.10% | 47.29% | 75.91% | 97.95% |

**Interpretation:** The relative ranking changes with the workload. N-Hit leads the practical policies on A42, while LARC leads them on A108, A129, and A669. Belady's Optimal performs best in all four traces, as expected given its access to future requests.

**Reproducibility:** Only A42 is intended to be downloadable from this repository. The values for A108, A129, and A669 document the original experiments; reproducing those runs requires the corresponding trace files, which are not included. For complete read/write breakdowns, see [the project report](Document.pdf).
