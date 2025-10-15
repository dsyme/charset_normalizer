# Performance Guide: Build Performance and Development Workflow

## Overview

Fast iteration cycles are crucial for effective performance engineering. This guide covers build optimization, test execution performance, and CI/CD efficiency for charset_normalizer.

## Build System

### Standard Build (Pure Python)

**Fast editable install:**
```bash
pip install -e .
```
- Minimal overhead
- Good for rapid development iteration
- No compilation delay

### mypyc Build (Compiled)

**Enable mypyc compilation:**
```bash
CHARSET_NORMALIZER_USE_MYPYC=1 pip install -e .
```

**Build time characteristics:**
- First build: ~30-60 seconds (compiles md.py)
- Rebuilds: Faster if only non-compiled files changed
- Trade-off: Build time vs runtime performance (2-10x speedup)

**When to use mypyc builds:**
- ✅ Final performance validation
- ✅ Benchmarking for PRs
- ✅ Testing production-like performance
- ❌ Rapid development iterations
- ❌ Debugging (harder to debug compiled code)

### Build Optimization Strategies

**1. Development: Use pure Python by default**
```bash
# Fast rebuilds
pip install -e .

# Quick test
pytest tests/test_api.py -k test_specific
```

**2. Performance work: Selective mypyc usage**
```bash
# Test without mypyc first (faster iteration)
pip install -e .
python bin/performance.py

# Validate with mypyc when ready
CHARSET_NORMALIZER_USE_MYPYC=1 pip install -e .
python bin/performance.py
```

**3. Build caching with uv (future consideration)**
```bash
# uv can significantly speed up dependency resolution
uv pip install -e .
```

## Test Execution Performance

### Quick Testing Strategies

**Run specific tests:**
```bash
# Single test file
pytest tests/test_api.py

# Single test function
pytest tests/test_api.py::test_elementary_unicode_range

# Pattern matching
pytest -k "unicode"
```

**Fast feedback loop:**
```bash
# Skip slow tests during development
pytest -m "not slow"

# Show test durations
pytest --durations=10
```

### Parallel Test Execution

**Current setup:** Tests run sequentially

**Optimization opportunity:**
```bash
# Install pytest-xdist
pip install pytest-xdist

# Run tests in parallel (e.g., 4 workers)
pytest -n 4

# Auto-detect CPU count
pytest -n auto
```

**Caution:** Some tests may have shared state issues

### Focused Performance Testing

**Test specific optimizations without full suite:**
```bash
# Custom perf test script
python <<EOF
import time
from charset_normalizer import from_bytes

with open('char-dataset/Samples/sample-arabic-1.txt', 'rb') as f:
    data = f.read()

start = time.perf_counter()
for _ in range(100):
    from_bytes(data)
end = time.perf_counter()

print(f"Time: {(end - start) / 100 * 1000:.2f}ms per iteration")
EOF
```

**Benefits:**
- Immediate feedback (seconds vs minutes)
- Isolate specific code paths
- Easier to reproduce and debug

## CI/CD Pipeline Optimization

### Current CI Structure

**Job overview (from `.github/workflows/ci.yml`):**
- Lint job: Pre-commit checks
- Tests: 8 Python versions (3.7-3.14)
- mypyc tests: 8 versions × 3 OS = 24 jobs
- Detection coverage, integration tests, performance test
- **Total:** ~35+ jobs per PR

### Optimization Opportunities

**1. Conditional job execution**

Example workflow improvement:
```yaml
# Only run mypyc tests if code changed (not docs-only)
mypyc_test:
  if: |
    !contains(github.event.pull_request.labels.*.name, 'documentation') &&
    (contains(github.event.commits[0].modified, 'src/') ||
     contains(github.event.commits[0].modified, 'setup.py'))
```

**2. Artifact caching**

Current: Limited caching

Improvement opportunities:
- Cache pip dependencies across jobs
- Cache char-dataset clone
- Share compiled mypyc modules between jobs (same Python version)

**3. Matrix optimization**

Consider:
- Test most common Python versions first (3.10, 3.11, 3.12)
- Run less common versions (3.7, 3.14) only on main/release branches
- Reduce OS matrix for older Python versions

### Performance Engineering CI

**Dedicated perf workflow (future consideration):**
```yaml
name: Performance Regression Detection

on: [pull_request]

jobs:
  perf-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - name: Benchmark baseline
        run: |
          git checkout ${{ github.base_ref }}
          pip install -e .
          nox -s performance > baseline.txt
      - name: Benchmark PR
        run: |
          git checkout ${{ github.sha }}
          pip install -e .
          nox -s performance > pr.txt
      - name: Compare results
        run: python scripts/compare_perf.py baseline.txt pr.txt
```

## Efficient Performance Engineering Workflow

### Recommended Iteration Cycle

**Phase 1: Hypothesis and Quick Test**
```bash
# 1. Profile to identify bottleneck (once)
python -m cProfile -s cumulative bin/performance.py

# 2. Make targeted change to hot path

# 3. Quick synthetic test (< 10 seconds)
python scripts/quick_perf_test.py

# 4. Iterate on step 2-3 until promising
```

**Phase 2: Validation**
```bash
# 5. Run focused tests
pytest tests/test_api.py tests/test_models.py

# 6. Full performance benchmark (pure Python)
nox -s performance

# 7. If good, rebuild with mypyc and retest
CHARSET_NORMALIZER_USE_MYPYC=1 pip install -e .
nox -s performance
```

**Phase 3: Verification**
```bash
# 8. Full test suite
nox -s test

# 9. Detection coverage check
nox -s coverage -- --coverage 97

# 10. Backward compatibility
nox -s backward_compatibility
```

### Maximally Efficient Measurements

**For algorithm changes:**
1. Create micro-benchmark with specific input
2. Use `timeit` for statistical reliability
3. Test both typical and worst-case inputs
4. Only run full `bin/performance.py` for final validation

**For mypyc compilation changes:**
1. Test pure Python first (faster to iterate)
2. Only compile when ready to validate
3. Use `nox -s test_mypyc` for correctness
4. Use `nox -s performance` with MYPYC=1 for performance

## Quick Reference Commands

### Fast Development
```bash
# Quick install
pip install -e .

# Run specific test
pytest tests/test_api.py -k test_name

# Quick perf check (custom script)
python your_perf_test.py
```

### Build Performance
```bash
# Pure Python (fast)
pip install -e .

# With mypyc (slow build, fast runtime)
CHARSET_NORMALIZER_USE_MYPYC=1 pip install -e .

# Clean rebuild
pip uninstall charset-normalizer -y && pip install -e .
```

### Test Performance
```bash
# Show slowest tests
pytest --durations=10

# Run tests in parallel
pytest -n auto

# Skip slow tests
pytest -m "not slow"
```

### Nox Sessions
```bash
# Full performance benchmark
nox -s performance

# Test single Python version
nox -s test-3.12

# Test with mypyc
nox -s test_mypyc-3.12

# Lint/format
nox -s lint
```

## Common Time Sinks to Avoid

1. **Running full test suite too often**
   - Use focused tests during development
   - Reserve full suite for final validation

2. **Rebuilding with mypyc unnecessarily**
   - Develop with pure Python
   - Only enable mypyc for final performance checks

3. **Testing on too many Python versions locally**
   - Test on your primary version (3.11/3.12)
   - Let CI handle matrix testing

4. **Cloning char-dataset repeatedly**
   - Clone once, reuse
   - nox handles this automatically

5. **Not using profiling to identify bottlenecks**
   - Always profile before optimizing
   - Focus on hot paths (top 5 functions)

## Success Metrics

- **Build time:** < 5 seconds for editable pure Python install
- **Test iteration:** < 30 seconds for focused test subset
- **Full test suite:** < 5 minutes locally (with parallel execution)
- **Performance benchmark:** < 2 minutes (nox -s performance)
- **CI pipeline:** < 15 minutes for critical path
