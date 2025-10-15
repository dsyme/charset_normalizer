# Performance Engineering Guide: Algorithm Optimization

## Overview

This guide covers algorithm-level performance optimization for charset_normalizer, including profiling techniques, hot path optimization, and mypyc compilation strategies.

## Performance-Critical Code Paths

### 1. Detection Algorithm (api.py)
- **Main function**: `from_bytes()` - Entry point for charset detection
- **Hot path**: Iterates through `IANA_SUPPORTED` encodings, tests chunks
- **Key parameters**:
  - `steps` (default 5): Number of incremental analysis steps
  - `chunk_size` (default 512 bytes): Size of data chunks analyzed
- **Location**: `src/charset_normalizer/api.py:from_bytes`

### 2. Mess Detection (md.py)
- **Already mypyc compiled** for ~2x speedup
- **Function**: `mess_ratio()` - Character-by-character analysis
- **Optimization**: Uses 8 detector plugins with early exit on threshold
- **Location**: `src/charset_normalizer/md.py:mess_ratio`

### 3. Coherence Detection (cd.py)
- **Not yet mypyc compiled** - High-priority target
- **Function**: `coherence_ratio()` - Language frequency analysis
- **Hot operations**: Character frequency lookups in large tables
- **Location**: `src/charset_normalizer/cd.py:coherence_ratio`

### 4. Utility Functions (utils.py)
- **Not yet mypyc compiled** - Medium-priority target
- **Called frequently** during detection process
- **Location**: `src/charset_normalizer/utils.py`

## Profiling Strategy

### Quick Profiling with cProfile

```bash
# Profile detection on a specific file
python -m cProfile -s cumulative -o profile.stats bin/performance.py

# Analyze profile interactively
python -c "import pstats; p = pstats.Stats('profile.stats'); p.sort_stats('cumulative').print_stats(20)"
```

### Focused Profiling with line_profiler

```bash
# Install line_profiler
pip install line_profiler

# Add @profile decorator to function of interest
# Run with kernprof
kernprof -l -v your_script.py
```

### Memory Profiling

```bash
# Install memory_profiler
pip install memory_profiler

# Profile memory usage
python -m memory_profiler your_script.py
```

## mypyc Compilation Strategy

### Current Status
- **Compiled**: `md.py` (mess detection) - 2x speedup achieved
- **Not compiled**: `cd.py`, `utils.py`, `api.py`

### Adding New Modules to mypyc

1. **Edit setup.py** to add module to `MYPYC_MODULES`:
   ```python
   MYPYC_MODULES = mypycify(
       [
           "src/charset_normalizer/md.py",
           "src/charset_normalizer/cd.py",  # Add new module here
       ],
       debug_level="0",
       opt_level="3",
   )
   ```

2. **Ensure type compatibility**:
   - Module must have complete type annotations
   - Check with: `mypy --strict src/charset_normalizer/cd.py`
   - Fix any type errors before compiling

3. **Build and test**:
   ```bash
   # Build with mypyc
   CHARSET_NORMALIZER_USE_MYPYC=1 pip install -e .

   # Run tests to ensure correctness
   pytest tests/

   # Benchmark performance impact
   python bin/performance.py
   ```

### mypyc Limitations
- Cannot compile modules with certain Python features (e.g., exec, eval)
- Debugging compiled modules is harder
- Build time increases significantly
- May not speed up I/O-bound code

## Optimization Techniques

### 1. Algorithm Selection
- **Profile first**: Measure before optimizing
- **Focus on hot paths**: 80/20 rule applies
- **Consider trade-offs**: Speed vs accuracy vs memory

### 2. Data Structure Optimization
- **Use appropriate containers**: list vs tuple vs set
- **Pre-compute when possible**: Avoid repeated calculations
- **Cache results**: For expensive operations with repeated inputs

### 3. Loop Optimization
- **Minimize work inside loops**: Move invariants outside
- **Use list comprehensions**: Often faster than explicit loops
- **Consider generator expressions**: For large datasets

### 4. Early Exit Optimization
- **Example in md.py**: Exits early when mess ratio exceeds threshold
- **Check expensive conditions last**: Short-circuit evaluation
- **Fail fast**: Return early for invalid inputs

## Measurement Guidelines

### Before/After Benchmarking

```bash
# Baseline: Current performance
python bin/performance.py > baseline.txt

# Make your optimization changes

# Rebuild if needed
pip install -e .

# Measure new performance
python bin/performance.py > optimized.txt

# Compare results
diff baseline.txt optimized.txt
```

### Key Metrics to Track
1. **Mean execution time**: Overall performance
2. **Percentiles** (50th, 95th, 99th): Tail latency
3. **Throughput**: Files per second
4. **Detection accuracy**: Must maintain ≥97% coverage

### Statistical Significance
- Run benchmarks **multiple times** (at least 3-5 runs)
- Look for **consistent improvements** (≥10% to be meaningful)
- Watch for **performance regressions** on specific file types
- Test across **different file sizes** (small, medium, large)

## Common Performance Pitfalls

### ❌ Don't
- Optimize without profiling first
- Break existing functionality for marginal gains
- Sacrifice accuracy for speed
- Ignore cross-platform compatibility
- Skip type checking before mypyc compilation

### ✅ Do
- Profile to identify actual bottlenecks
- Measure performance impact quantitatively
- Maintain or improve detection accuracy
- Test on Linux, macOS, and Windows
- Run full test suite after changes

## Quick Reference: Performance Testing Commands

```bash
# Run performance benchmark (with mypyc)
nox -s performance

# Run tests with coverage
nox -s test-3.12

# Run mypyc tests
nox -s test_mypyc-3.12

# Run type checking
mypy src/charset_normalizer/

# Run pre-commit checks (includes formatting and linting)
nox -s lint
```

## Performance Targets

Based on the research in the performance plan:

- **Current throughput**: ~100 files/sec on char-dataset
- **Current vs chardet**: 10x faster (10ms vs 63ms mean)
- **Optimization goal**: Maintain accuracy while improving speed
- **Small file target**: Competitive with cchardet on <100 byte inputs
- **Large file target**: Maintain or improve current performance on MB+ files

## Resources

- **Performance script**: `bin/performance.py`
- **Test files**: `char-dataset/` (cloned via nox sessions)
- **CI performance test**: `.github/workflows/ci.yml` (performance job)
- **Issue tracker**: Check for issues tagged with "performance"
