# Performance Engineering Guide: Measurement and Benchmarking

## Overview

This guide covers performance measurement strategies, benchmarking techniques, and how to validate optimization results for charset_normalizer.

## Performance Measurement Philosophy

### Key Principles
1. **Measure, don't guess**: Always profile before optimizing
2. **Reproducible results**: Same input should give consistent measurements
3. **Statistical validity**: Multiple runs to account for variance
4. **Realistic workloads**: Test on representative data
5. **Preserve accuracy**: Never sacrifice correctness for speed

## Primary Benchmarking Tool

### bin/performance.py

The main performance testing script that compares charset_normalizer against chardet.

```bash
# Basic usage
python bin/performance.py

# Via nox (recommended - handles compilation)
nox -s performance
```

### What It Measures
- **Detection time**: Per-file processing time
- **Throughput**: Files processed per second
- **Accuracy**: Detection coverage percentage
- **Percentiles**: 50th, 95th, 99th latency
- **Comparison**: charset_normalizer vs chardet

### Output Interpretation

```
Mean: 10ms (charset_normalizer) vs 63ms (chardet)
Files/sec: 100 (charset_normalizer) vs 16 (chardet)
50th percentile: 8ms vs 45ms
95th percentile: 25ms vs 120ms
99th percentile: 40ms vs 180ms
Coverage: 97% vs 89%
```

### Key Metrics
- **Mean time**: Lower is better
- **Throughput**: Higher is better
- **Percentiles**: Lower is better (tail latency)
- **Coverage**: Higher is better (accuracy)

## Benchmark Methodology

### Before/After Testing

```bash
# 1. Establish baseline
git checkout main
CHARSET_NORMALIZER_USE_MYPYC=1 pip install -e .
python bin/performance.py > /tmp/gh-aw/agent/baseline.txt

# 2. Make your optimization changes
git checkout -b perf/my-optimization
# Edit code...

# 3. Rebuild and measure
CHARSET_NORMALIZER_USE_MYPYC=1 pip install -e .
python bin/performance.py > /tmp/gh-aw/agent/optimized.txt

# 4. Compare results
diff /tmp/gh-aw/agent/baseline.txt /tmp/gh-aw/agent/optimized.txt
```

### Multiple Runs for Statistical Validity

```bash
# Run 5 times and average
for i in {1..5}; do
    python bin/performance.py >> /tmp/gh-aw/agent/results_run_$i.txt
done

# Analyze variance across runs
```

### Test Across Different File Sizes

```bash
# Small files (<100 bytes)
python -c "
from charset_normalizer import from_bytes
import time
data = b'small test'
start = time.time()
for _ in range(1000):
    from_bytes(data).best()
print(f'Small file: {(time.time()-start)*1000:.2f}ms for 1000 iterations')
"

# Medium files (1KB - 1MB)
# Use files from char-dataset

# Large files (>1MB)
# Create or find large test files
```

## Profiling Tools and Techniques

### 1. cProfile (CPU Profiling)

```bash
# Profile performance script
python -m cProfile -s cumulative -o /tmp/gh-aw/agent/profile.stats bin/performance.py

# Analyze top functions by cumulative time
python << 'EOF'
import pstats
p = pstats.Stats('/tmp/gh-aw/agent/profile.stats')
p.sort_stats('cumulative').print_stats(30)
EOF

# Sort by time spent in function itself
python << 'EOF'
import pstats
p = pstats.Stats('/tmp/gh-aw/agent/profile.stats')
p.sort_stats('time').print_stats(30)
EOF
```

### 2. line_profiler (Line-by-Line Profiling)

```bash
# Install
pip install line_profiler

# Add @profile decorator to function of interest
# Example: Edit src/charset_normalizer/api.py
# Add: @profile
# def from_bytes(...):

# Run profiler
kernprof -l -v your_test_script.py

# Output shows time per line
```

### 3. memory_profiler (Memory Usage)

```bash
# Install
pip install memory_profiler

# Profile memory usage
python -m memory_profiler your_script.py

# Line-by-line memory profiling
# Add @profile decorator and run with mprof
mprof run your_script.py
mprof plot  # Visualize memory usage over time
```

### 4. py-spy (Sampling Profiler)

```bash
# Install
pip install py-spy

# Profile running process
py-spy top --pid <PID>

# Generate flamegraph
py-spy record -o /tmp/gh-aw/agent/profile.svg --pid <PID>

# Profile command
py-spy record -o /tmp/gh-aw/agent/profile.svg -- python bin/performance.py
```

## Micro-Benchmarking with timeit

```python
import timeit

# Test specific function
setup = """
from charset_normalizer import from_bytes
data = b'test data' * 100
"""

time = timeit.timeit(
    'from_bytes(data).best()',
    setup=setup,
    number=1000
)
print(f"Average time: {time/1000*1000:.2f}ms")
```

## Measuring Specific Optimizations

### Algorithm Change Impact

```python
# Create focused test
import time
from charset_normalizer import from_bytes

# Load test file
with open('char-dataset/test-file.txt', 'rb') as f:
    data = f.read()

# Time original approach
start = time.perf_counter()
for _ in range(100):
    result = from_bytes(data).best()
elapsed = time.perf_counter() - start
print(f"100 iterations: {elapsed*1000:.2f}ms")
print(f"Per iteration: {elapsed/100*1000:.2f}ms")
```

### mypyc Compilation Impact

```bash
# Baseline: Without mypyc
pip uninstall charset-normalizer -y
pip install -e .
python bin/performance.py > /tmp/gh-aw/agent/no_mypyc.txt

# With mypyc
pip uninstall charset-normalizer -y
CHARSET_NORMALIZER_USE_MYPYC=1 pip install -e .
python bin/performance.py > /tmp/gh-aw/agent/with_mypyc.txt

# Compare
diff /tmp/gh-aw/agent/no_mypyc.txt /tmp/gh-aw/agent/with_mypyc.txt
```

## Accuracy Validation

### Detection Coverage Testing

```bash
# Run coverage test via nox
nox -s coverage -- --coverage 97 --with-preemptive

# Manual coverage check
python bin/coverage.py --coverage 97 --with-preemptive
```

### Ensure No Regression

```bash
# Full test suite must pass
pytest tests/ -v

# Specific encoding tests
pytest tests/test_api.py -v

# mypyc compiled tests
nox -s test_mypyc-3.12
```

## Performance Testing Best Practices

### ✅ Do

1. **Establish baseline first**: Always measure before changes
2. **Test multiple times**: Run benchmarks 3-5 times minimum
3. **Test realistic data**: Use char-dataset and real-world files
4. **Test different scenarios**: Small, medium, large files
5. **Verify accuracy**: Run full test suite and coverage tests
6. **Document methodology**: How you measured and what you found
7. **Test on target platforms**: Linux, macOS, Windows

### ❌ Don't

1. **Trust single runs**: Results can vary
2. **Test only micro-benchmarks**: May not reflect real-world usage
3. **Skip accuracy validation**: Speed means nothing if detection breaks
4. **Compare different configurations**: mypyc on/off, debug/release
5. **Optimize without profiling**: May optimize the wrong thing
6. **Forget system variance**: CPU frequency scaling, background processes

## Reporting Performance Results

### Minimum Information Required

1. **Measurement methodology**:
   - Command used: `python bin/performance.py`
   - Number of runs: 5
   - Test data: char-dataset (400+ files)

2. **Environment details**:
   - Python version: 3.12
   - mypyc enabled: Yes/No
   - OS: Ubuntu 22.04 / macOS 14 / Windows 11
   - CPU: Model and clock speed

3. **Results**:
   - Mean time: Before vs After
   - Throughput: Before vs After
   - Percentiles: 50th, 95th, 99th
   - Speedup: X% improvement or X.Xx faster

4. **Accuracy validation**:
   - Test suite: All tests passing ✓
   - Detection coverage: 97%+ maintained ✓

### Example Report Format

```markdown
## Performance Impact

### Measurement Methodology
- Benchmark: `python bin/performance.py`
- Runs: 5 times, reporting median
- Dataset: char-dataset (400+ files)
- Configuration: mypyc enabled

### Environment
- Python: 3.12.0
- OS: Ubuntu 22.04
- CPU: Intel i7-9700K @ 3.6GHz

### Results
| Metric | Before | After | Change |
|--------|--------|-------|--------|
| Mean time | 10ms | 8ms | -20% |
| Throughput | 100 f/s | 125 f/s | +25% |
| 50th %ile | 8ms | 6ms | -25% |
| 95th %ile | 25ms | 20ms | -20% |
| 99th %ile | 40ms | 32ms | -20% |

### Accuracy Validation
- All tests passing: ✓
- Detection coverage: 97.2% (maintained)
- No accuracy regressions detected
```

## Performance Targets and Thresholds

### Meaningful Improvements
- **>10% speedup**: Worth pursuing
- **>25% speedup**: Significant improvement
- **>50% speedup**: Major optimization
- **<5% speedup**: May not be worth complexity cost

### Acceptable Variance
- **<5% variance**: Good reproducibility
- **5-10% variance**: Acceptable
- **>10% variance**: Investigate environmental factors

### Regression Thresholds
- **>5% slowdown**: Investigate cause
- **>10% slowdown**: Likely requires fixing
- **>20% slowdown**: Unacceptable regression

## Quick Reference Commands

```bash
# Full benchmark (recommended)
nox -s performance

# Quick benchmark (manual)
python bin/performance.py

# CPU profiling
python -m cProfile -s cumulative bin/performance.py

# Test accuracy
nox -s coverage -- --coverage 97

# Run all tests
pytest tests/ -v

# Time specific operation
python -m timeit -s "from charset_normalizer import from_bytes; data=b'test'*100" \
    "from_bytes(data).best()"
```

## Resources

- **Benchmark script**: `bin/performance.py`
- **Coverage script**: `bin/coverage.py`
- **Benchmark data**: `char-dataset/` (auto-cloned by nox)
- **Test suite**: `tests/`
- **CI benchmarks**: `.github/workflows/ci.yml` (performance job)
