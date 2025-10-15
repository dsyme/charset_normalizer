# Performance Guide: Measurement and Benchmarking

## Overview

Effective performance engineering requires reliable measurement. This guide covers measurement strategies, benchmarking methodologies, and result interpretation for charset_normalizer.

## Built-in Performance Testing

### Primary Tool: bin/performance.py

**Purpose:** Compare charset_normalizer against chardet on 400+ test files from char-dataset

**Usage:**
```bash
# Run full benchmark
nox -s performance

# Or directly
git clone --depth 1 https://github.com/ousret/char-dataset
pip install chardet
pip install -e .
python bin/performance.py
```

**Output format:**
```
charset_normalizer: 10ms mean, 100 files/sec, 50th: 8ms, 95th: 15ms, 99th: 25ms, coverage: 98.5%
chardet: 63ms mean, 16 files/sec, 50th: 45ms, 95th: 120ms, 99th: 180ms, coverage: 97.2%
```

**Key metrics:**
- **Mean time:** Average detection time per file
- **Files/sec:** Throughput metric
- **Percentiles:** Distribution of performance
  - 50th (median): Typical case
  - 95th: Near worst-case
  - 99th: Worst-case
- **Coverage:** Detection accuracy (% of files correctly identified)

### Interpreting Results

**Good performance improvement:**
- Mean time reduced by > 10%
- Files/sec increased proportionally
- Percentiles improved (especially 95th/99th)
- Coverage maintained or improved (≥97%)

**Warning signs:**
- Coverage drops below 97%
- 99th percentile worsens (even if mean improves)
- High variance between runs (> 15%)

## Measurement Methodologies

### 1. Synthetic Benchmarks (Fast, Reproducible)

**For algorithm optimizations:**

```python
import time
import statistics
from charset_normalizer import from_bytes

# Load test data
with open('char-dataset/Samples/sample-arabic-1.txt', 'rb') as f:
    test_data = f.read()

# Warm-up (avoid cold start bias)
for _ in range(10):
    from_bytes(test_data)

# Measure
times = []
for _ in range(100):
    start = time.perf_counter()
    result = from_bytes(test_data)
    end = time.perf_counter()
    times.append(end - start)

# Statistics
mean_ms = statistics.mean(times) * 1000
stdev_ms = statistics.stdev(times) * 1000
median_ms = statistics.median(times) * 1000

print(f"Mean: {mean_ms:.2f}ms ± {stdev_ms:.2f}ms")
print(f"Median: {median_ms:.2f}ms")
print(f"Min: {min(times) * 1000:.2f}ms, Max: {max(times) * 1000:.2f}ms")
```

**Benefits:**
- Fast (< 10 seconds)
- Immediate feedback
- Good for tight iteration loops

**Limitations:**
- Single file may not be representative
- Doesn't test full encoding diversity

### 2. Focused Benchmarks (Targeted, Controlled)

**For testing specific encoding or file size:**

```python
# Test small files (< 100 bytes)
small_files = [
    'char-dataset/Samples/sample-ascii-short.txt',
    'char-dataset/Samples/sample-utf8-short.txt',
    # ... more small files
]

# Benchmark small file performance
for file_path in small_files:
    with open(file_path, 'rb') as f:
        data = f.read()
    # Time detection...
```

**Use cases:**
- Testing small input fast path optimization
- Comparing specific encoding detection
- Validating worst-case inputs

### 3. Comprehensive Benchmarks (Complete, Accurate)

**Use bin/performance.py for final validation:**

```bash
# Baseline (before optimization)
git checkout main
pip install -e .
nox -s performance | tee baseline.txt

# After optimization
git checkout feature-branch
pip install -e .
nox -s performance | tee optimized.txt

# Compare
grep "mean" baseline.txt optimized.txt
```

**Comparison script example:**
```python
# scripts/compare_perf.py
import sys
import re

def parse_output(filename):
    with open(filename) as f:
        content = f.read()
    # Extract mean, files/sec, percentiles
    # ... parsing logic ...
    return metrics

baseline = parse_output(sys.argv[1])
optimized = parse_output(sys.argv[2])

improvement = (baseline['mean'] - optimized['mean']) / baseline['mean'] * 100
print(f"Mean time improvement: {improvement:.1f}%")
print(f"Throughput change: {optimized['fps'] / baseline['fps'] - 1:.1%}")
```

## Profiling for Bottleneck Identification

### cProfile (Function-Level Profiling)

**Quick profiling:**
```bash
python -m cProfile -s cumulative bin/performance.py 2>&1 | head -30
```

**Save and analyze:**
```bash
python -m cProfile -o profile.stats bin/performance.py

python -c "
import pstats
p = pstats.Stats('profile.stats')
p.strip_dirs()
p.sort_stats('cumulative')
p.print_stats(20)
p.sort_stats('tottime')
p.print_stats(20)
"
```

**What to look for:**
- Top functions by cumulative time (includes callees)
- Top functions by tottime (own time only)
- Surprising hot spots (shouldn't be expensive but are)

### line_profiler (Line-Level Profiling)

**For deep analysis of specific functions:**

1. **Install:**
```bash
pip install line_profiler
```

2. **Add decorator to target function:**
```python
# In src/charset_normalizer/api.py
@profile  # Add this line
def from_bytes(sequences, ...):
    # ... function code ...
```

3. **Run profiler:**
```bash
kernprof -l -v your_test_script.py
```

4. **Analyze output:**
```
Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
   45                                           @profile
   46                                           def from_bytes(sequences, ...):
   47       100          1.2      0.01      0.5      # ... line 1 ...
   48       100         15.3      0.15      6.8      # ... line 2 (hot) ...
```

**Use for:**
- Identifying hot lines within a function
- Understanding loop performance
- Validating optimization impact line-by-line

### py-spy (Sampling Profiler, No Code Changes)

**Real-time profiling without instrumentation:**

```bash
# Install
pip install py-spy

# Profile running process
python bin/performance.py &
PID=$!
py-spy record -o profile.svg --pid $PID

# View profile.svg in browser
```

**Benefits:**
- No code modification needed
- Low overhead
- Visual flamegraph output

## Measurement Best Practices

### 1. Statistical Rigor

**Run multiple iterations:**
```bash
for i in {1..5}; do
    nox -s performance | tee run_$i.txt
done

# Calculate mean and stdev across runs
python scripts/aggregate_results.py run_*.txt
```

**Why:** Performance can vary due to system load, CPU frequency scaling, etc.

### 2. Isolate Variables

**When comparing:**
- Same hardware
- Same Python version
- Same system load (close other apps)
- Same input data
- Only one change at a time

**Bad:**
```bash
# Multiple changes + different Python version
pip install -e . --python 3.11  # Changed Python
# ... make 3 different optimizations ...
nox -s performance
```

**Good:**
```bash
# Isolate single optimization
git checkout baseline
pip install -e .
nox -s performance > baseline.txt

git checkout single-optimization
pip install -e .
nox -s performance > optimized.txt
```

### 3. Warm-up Runs

**Always run warm-up iterations:**
```python
# Warm-up (avoid cold start, populate caches)
for _ in range(10):
    from_bytes(test_data)

# Measure
for _ in range(100):
    # ... timing code ...
```

**Why:** First runs may be slower due to:
- Python bytecode compilation
- Module imports
- CPU frequency scaling
- Disk I/O for first file access

### 4. Reproducible Environments

**Document environment:**
```bash
# Save environment for reproduction
python --version > environment.txt
pip freeze >> environment.txt
uname -a >> environment.txt
```

**Use containers for consistency:**
```bash
# Run benchmarks in Docker for reproducibility
docker run --rm -it python:3.12-slim bash
pip install charset-normalizer
# ... benchmark ...
```

## Common Measurement Pitfalls

### 1. Timing Too Little or Too Much

**Too little (< 1ms):**
- Measurement noise dominates
- Solution: Run many iterations, measure total time

**Too much (> 10 seconds):**
- System noise accumulates
- Solution: Break into multiple shorter measurements

### 2. Not Accounting for Variance

**Bad:**
```
Before: 10ms
After: 9ms
Conclusion: 10% faster ✗
```

**Good:**
```
Before: 10ms ± 2ms (5 runs)
After: 9ms ± 2ms (5 runs)
Conclusion: Improvement within noise, need more data
```

### 3. Cherry-Picking Results

**Bad:** Run benchmark 10 times, report best result

**Good:** Run benchmark 10 times, report mean ± stdev

### 4. Ignoring Worst Case

**Bad:** Optimize mean time, ignore 99th percentile regression

**Good:** Track all percentiles (50th, 95th, 99th) and ensure none regress significantly

### 5. Not Validating Accuracy

**Critical:** Always run coverage check after performance optimization
```bash
nox -s coverage -- --coverage 97
```

## Quick Reference Commands

### Profiling
```bash
# Function-level profiling
python -m cProfile -s cumulative bin/performance.py | head -30

# Line-level profiling (requires @profile decorator)
kernprof -l -v your_script.py

# Sampling profiler (no code changes)
py-spy record -o profile.svg -- python bin/performance.py
```

### Benchmarking
```bash
# Full benchmark
nox -s performance

# Custom micro-benchmark
python your_perf_test.py

# Compare baseline vs optimized
diff <(grep "mean" baseline.txt) <(grep "mean" optimized.txt)
```

### Validation
```bash
# Detection coverage
nox -s coverage -- --coverage 97

# Full test suite
nox -s test

# Backward compatibility
nox -s backward_compatibility
```

## Success Criteria for Performance Changes

Before submitting a performance PR, ensure:

1. **Improvement is measurable**
   - Mean time improved by > 5%
   - OR throughput improved by > 5%
   - With statistical significance (> 2× stdev)

2. **No regression in percentiles**
   - 95th percentile not worse by > 5%
   - 99th percentile not worse by > 10%

3. **Accuracy maintained**
   - Detection coverage ≥97%
   - All tests passing

4. **Results are reproducible**
   - Documented environment
   - Measurement methodology explained
   - Multiple runs show consistent results

5. **Trade-offs documented**
   - Memory usage impact (if any)
   - Code complexity increase (if any)
   - Platform-specific considerations
