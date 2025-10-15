# Performance Guide: Algorithm Optimization and Profiling

## Overview

charset_normalizer is a Python library for charset/encoding detection. Performance is critical as it processes potentially large text files character-by-character. This guide covers algorithm optimization strategies specific to this codebase.

## Key Performance-Critical Components

### 1. Detection Algorithm (`api.py`)
- **from_bytes()**: Main entry point that iterates through IANA encodings
- **Steps parameter**: Controls granularity of chunk testing (default: 5)
- **chunk_size**: Bytes per test chunk (default: 512)
- Performance scales with: file size × encoding candidates × steps

### 2. Mess Detection (`md.py`) - Already mypyc Compiled
- **mess_ratio()**: Character-by-character analysis
- Uses 8 detector plugins (SuccessiveWhitespaceRatio, SymbolsPlugin, etc.)
- Current optimization: Early exit when threshold exceeded
- **Note**: This module is already compiled with mypyc for ~2-10x speedup

### 3. Coherence Detection (`cd.py`)
- **coherence_ratio()**: Language detection using character frequency tables
- Lookups into large FREQUENCIES dict from `constant.py`
- Can be computationally expensive for multi-language text

### 4. Large Data Tables (`constant.py`)
- IANA_SUPPORTED: List of encoding names
- FREQUENCIES: Large dict for language detection
- Character maps and language data (~2,015 lines)

## Profiling Strategies

### Quick Synthetic Benchmarking

**Use the built-in performance script:**
```bash
# Compare against chardet on char-dataset
nox -s performance

# With mypyc compilation enabled
CHARSET_NORMALIZER_USE_MYPYC=1 pip install -e .
nox -s performance
```

**Interpret results:**
- Mean time (ms): Average detection time
- Files/sec: Throughput metric
- Percentiles (50th, 95th, 99th): Distribution of performance
- Compare mypyc vs pure Python builds

### Detailed Profiling with cProfile

**Profile specific detection:**
```bash
python -m cProfile -o profile.stats <<EOF
from charset_normalizer import from_bytes
with open('char-dataset/Samples/sample-arabic-1.txt', 'rb') as f:
    result = from_bytes(f.read())
EOF

# Analyze results
python -c "
import pstats
p = pstats.Stats('profile.stats')
p.sort_stats('cumulative').print_stats(20)
"
```

**Look for:**
- Time spent in mess_ratio(), coherence_ratio()
- Encoding iteration overhead
- Dictionary lookups in FREQUENCIES

### Line-by-Line Profiling

**For deep dive into hot functions:**
```bash
pip install line_profiler

# Add @profile decorator to target function, then:
kernprof -l -v your_test_script.py
```

**Target functions for line profiling:**
- `api.py:from_bytes` - Main detection loop
- `cd.py:coherence_ratio` - Language detection
- Any custom optimization changes

## Optimization Techniques

### 1. Expand mypyc Compilation

**Current state:** Only `md.py` is compiled

**Expansion strategy:**
1. Compile `cd.py` (coherence detection)
   - Heavy dict lookups → good candidate for mypyc
   - Ensure type annotations are complete
2. Compile `utils.py` (helper functions)
   - Frequently called utilities
   - Check for dynamic typing that may need fixing
3. Consider `api.py` (main algorithm)
   - More complex, may need type refactoring
   - Test carefully for correctness

**How to add:**
Edit `setup.py`, add to MYPYC_MODULES list:
```python
MYPYC_MODULES = mypycify([
    "src/charset_normalizer/md.py",
    "src/charset_normalizer/cd.py",  # New
    "src/charset_normalizer/utils.py",  # New
], debug_level="0", opt_level="3")
```

**Validate:** Run full test suite with mypyc enabled:
```bash
CHARSET_NORMALIZER_USE_MYPYC=1 nox -s test
```

### 2. Algorithm-Level Optimizations

**Early Exit Strategies:**
- Quick ASCII/UTF-8 check for small inputs
- Skip language detection for tiny files (< TOO_SMALL_SEQUENCE)
- Limit encoding candidates based on heuristics

**Parameter Tuning:**
- Experiment with `steps` and `chunk_size` defaults
- Profile trade-offs: accuracy vs performance
- Consider adaptive strategies based on input size

**Caching Opportunities:**
- Mess detector plugin results
- Coherence ratio calculations for repeated character sequences
- Encoding probe results

### 3. Data Structure Optimization

**IANA_SUPPORTED list:**
- Currently a list → O(n) iteration
- Consider set for O(1) membership tests where applicable
- Profile impact of reordering (most common encodings first)

**FREQUENCIES dict:**
- Large nested structure for language detection
- Consider: frozen dicts, more compact representations
- Profile memory impact of alternatives

### 4. Small Input Fast Path

**Problem:** charset_normalizer is slower on very small inputs (< 100 bytes)

**Optimization approach:**
```python
def from_bytes(sequences, ...):
    if len(sequences) < SMALL_INPUT_THRESHOLD:
        # Fast path: test only UTF-8, ASCII, Latin-1
        for encoding in ['utf-8', 'ascii', 'latin-1']:
            # Quick test without full mess/coherence analysis
            ...
    # Normal path for larger inputs
    ...
```

**Validate:** Ensure accuracy is maintained for small inputs

## Measurement Best Practices

### Before/After Comparison Protocol

1. **Establish baseline:**
   ```bash
   git checkout main
   pip install -e .
   nox -s performance > baseline.txt
   ```

2. **Apply optimization:**
   ```bash
   git checkout optimization-branch
   pip install -e .  # Rebuild if needed
   nox -s performance > optimized.txt
   ```

3. **Compare results:**
   - Mean time improvement
   - Throughput gain
   - Check 95th/99th percentile (worst case)
   - Verify detection coverage remains ≥97%

4. **Statistical significance:**
   - Run multiple times (5-10 runs)
   - Calculate mean and standard deviation
   - Ensure improvement is > 2× std dev

### Accuracy Validation

**Critical:** Performance must not sacrifice accuracy

```bash
# Check detection coverage
nox -s coverage -- --coverage 97

# Run full test suite
nox -s test

# Backward compatibility check
nox -s backward_compatibility -- --coverage 80
```

## Common Pitfalls

1. **Premature optimization:** Always profile first
2. **Breaking accuracy:** Always validate detection coverage
3. **mypyc type issues:** Ensure all type hints are correct
4. **Over-optimizing cold paths:** Focus on hot paths (detection loop, mess/coherence)
5. **Ignoring memory:** Some optimizations trade memory for speed

## Quick Reference Commands

```bash
# Fast synthetic benchmark
nox -s performance

# Profile specific file
python -m cProfile -s cumulative bin/performance.py

# Test with mypyc
CHARSET_NORMALIZER_USE_MYPYC=1 pip install -e .

# Validate accuracy
nox -s coverage -- --coverage 97

# Full test suite
nox -s test
```

## Success Metrics

- **Primary:** Files/sec throughput improvement
- **Secondary:** Mean/95th/99th percentile latency reduction
- **Constraint:** Detection coverage ≥97%
- **Constraint:** All tests passing
