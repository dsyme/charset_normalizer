# mypyc Performance Verification

## Summary

This document verifies the actual performance impact of mypyc compilation in charset_normalizer.

**Key Finding:** mypyc compilation provides approximately **2.0x (100%) performance improvement** across all metrics when properly enabled.

## Performance Comparison

### Without mypyc (Pure Python Mode)

Installation: `pip install -e .` (editable install)
Verification: `normalizer --version` shows **"SpeedUp OFF"**

```
Charset-Normalizer / Chardet: Performance Comparison
--> Avg: x3.63
--> 99th: x1.88
--> 95th: x1.06
--> 50th: x0.86
```

### With mypyc (Compiled Mode)

Installation: `CHARSET_NORMALIZER_USE_MYPYC=1 pip install . --no-build-isolation`
Verification: `normalizer --version` shows **"SpeedUp ON"**

```
Charset-Normalizer / Chardet: Performance Comparison
--> Avg: x7.27
--> 99th: x3.35
--> 95th: x1.95
--> 50th: x1.71
```

### mypyc Impact

| Metric | Without mypyc | With mypyc | Improvement |
|--------|---------------|------------|-------------|
| Average | 3.63x | 7.27x | **+2.00x (100%)** |
| 99th percentile | 1.88x | 3.35x | **+1.78x (78%)** |
| 95th percentile | 1.06x | 1.95x | **+1.84x (84%)** |
| 50th percentile | 0.86x | 1.71x | **+1.99x (99%)** |

**Observation:** mypyc compilation approximately **doubles** charset_normalizer's performance across all percentiles.

## Compiled Modules

Currently compiled with mypyc (setup.py):
- `src/charset_normalizer/md.py` - Mess detection
- `src/charset_normalizer/cd.py` - Coherence detection (language detection)
- `src/charset_normalizer/utils.py` - Character classification utilities

## Verification

To check if mypyc is active in your installation:

```bash
normalizer --version
```

**Expected output:**
- With mypyc: `Charset-Normalizer X.X.X - Python X.X.X - Unicode X.X.X - SpeedUp ON`
- Without mypyc: `Charset-Normalizer X.X.X - Python X.X.X - Unicode X.X.X - SpeedUp OFF`

## When mypyc is Active

**Wheel installations (pip install from PyPI):**
- Pre-compiled wheels include mypyc extensions
- SpeedUp ON by default
- This is what production users get

**Source installations:**
- Requires explicit `CHARSET_NORMALIZER_USE_MYPYC=1` environment variable
- Requires mypy package installed
- Used for building from source

**Editable installations (development):**
- `pip install -e .` installs in pure Python mode
- SpeedUp OFF by default (for faster development iteration)
- To enable mypyc in editable mode: `CHARSET_NORMALIZER_USE_MYPYC=1 pip install -e . --no-build-isolation`

## Performance Testing Considerations

When running performance tests or benchmarks:

1. **For production performance measurements:** Build with mypyc enabled
2. **For development/debugging:** Pure Python mode is faster to rebuild
3. **For CI/CD regression detection:** Use mypyc-compiled builds to match production
4. **For profiling:** Pure Python mode provides better introspection

Always document whether benchmarks were conducted with SpeedUp ON or OFF, as the difference is substantial (~2.0x).

## Test Environment

These measurements were conducted on:
- Python 3.12.3
- Ubuntu Linux (GitHub Actions runner)
- charset_normalizer 3.4.4
- Test dataset: 476 files from char-dataset repository
- Benchmark: `bin/performance.py` (comparison vs chardet)

---

**Generated:** 2025-10-23 by Daily Perf Improver
