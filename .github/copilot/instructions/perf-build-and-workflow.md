# Performance Engineering Guide: Build Performance & Development Workflow

## Overview

This guide covers build system optimization, fast iteration strategies, and developer workflow improvements for performance engineering in charset_normalizer.

## Build System Architecture

### Standard Build (No mypyc)
```bash
# Fast build for development
pip install -e .
```
- **Time**: ~5-10 seconds
- **Use case**: Rapid iteration, debugging
- **Performance**: Pure Python (baseline speed)

### mypyc Build (Production)
```bash
# Build with C compilation
CHARSET_NORMALIZER_USE_MYPYC=1 pip install -e .
```
- **Time**: ~30-90 seconds (depends on system)
- **Use case**: Performance testing, benchmarking
- **Performance**: 2-10x faster on compiled modules

## Fast Iteration Workflow

### Strategy 1: Develop Without mypyc First

1. **Make code changes** in pure Python
2. **Run tests quickly**: `pytest tests/test_api.py -v`
3. **Verify functionality** without waiting for compilation
4. **Only compile when ready** to benchmark

### Strategy 2: Incremental Testing

```bash
# Test specific module quickly
pytest tests/test_specific.py -k "test_name" -v

# Run fast subset of tests
pytest tests/ -m "not slow" -v

# Skip integration tests during development
pytest tests/ --ignore=tests/test_integration.py
```

### Strategy 3: Use Pre-built char-dataset

```bash
# Clone once (done automatically by nox, but can do manually)
git clone --depth 1 https://github.com/ousret/char-dataset

# Reuse for multiple benchmark runs
python bin/performance.py
```

## Performance Development Cycle

### 1. Profile Phase (No Compilation Needed)
```bash
# Install in development mode (fast)
pip install -e .

# Profile to find bottlenecks
python -m cProfile -s cumulative -o profile.stats bin/performance.py

# Analyze results
python -c "import pstats; p = pstats.Stats('profile.stats'); p.sort_stats('cumulative').print_stats(20)"
```

### 2. Optimize Phase (Pure Python)
```bash
# Make algorithm improvements
# Edit source files in src/charset_normalizer/

# Quick validation
pytest tests/ -v

# Functional verification
python -c "from charset_normalizer import from_bytes; print(from_bytes(b'test').best())"
```

### 3. Benchmark Phase (Compile for Accuracy)
```bash
# Rebuild with mypyc for final measurement
CHARSET_NORMALIZER_USE_MYPYC=1 pip install -e .

# Run full benchmark suite
python bin/performance.py

# Or use nox (handles compilation automatically)
nox -s performance
```

## Build Performance Tips

### Reducing Build Time

1. **Use ccache** (on Linux/macOS):
   ```bash
   # Install ccache
   sudo apt-get install ccache  # Ubuntu/Debian
   brew install ccache          # macOS

   # Configure for mypyc builds
   export CC="ccache gcc"
   export CXX="ccache g++"
   ```

2. **Parallel compilation**:
   ```bash
   # Set number of parallel jobs
   export MAX_JOBS=4
   pip install -e .
   ```

3. **Avoid unnecessary rebuilds**:
   ```bash
   # Only rebuild when changing compiled modules
   # If editing api.py (not compiled), no rebuild needed
   ```

### Build Artifacts to Clean

```bash
# Clean build artifacts
rm -rf build/ dist/ *.egg-info
rm -rf src/**/*.so src/**/*.pyd  # Compiled C extensions

# Fresh build
pip install -e .
```

## nox Session Reference

### Performance Engineering Sessions

```bash
# Quick test (one Python version)
nox -s test-3.12

# mypyc test (compile + test)
nox -s test_mypyc-3.12

# Performance benchmark (compile + benchmark)
nox -s performance

# Linting (no compilation)
nox -s lint
```

### nox Performance Tips

1. **Reuse environments**:
   ```bash
   # nox caches virtual environments
   # Subsequent runs are faster
   nox -s test-3.12 --reuse-existing-virtualenvs
   ```

2. **Run single session**:
   ```bash
   # Don't run all sessions if you only need one
   nox -s performance  # NOT: nox (runs all sessions)
   ```

3. **Skip environment recreation**:
   ```bash
   # If dependencies haven't changed
   nox -s test-3.12 --no-install
   ```

## CI/CD Performance Considerations

### Current CI Structure
- **8 Python versions** tested (3.7-3.14)
- **3 OS platforms** for mypyc (Linux, macOS, Windows)
- **Total: ~32 jobs** per PR

### Optimizing CI Time

1. **Conditional execution**:
   - Skip mypyc builds for docs-only changes
   - Use path filters in workflow triggers

2. **Artifact caching**:
   - Cache pip dependencies
   - Cache char-dataset clone
   - Cache build artifacts

3. **Parallel execution**:
   - Jobs run in parallel automatically
   - Reduce dependencies between jobs

### Local CI Emulation

```bash
# Test what CI will run (single Python version)
nox -s test-3.12
nox -s test_mypyc-3.12
nox -s lint
nox -s performance
```

## Measuring Build Performance

### Timing Builds

```bash
# Time standard build
time pip install -e .

# Time mypyc build
time CHARSET_NORMALIZER_USE_MYPYC=1 pip install -e .

# Compare build times
```

### Build Performance Metrics
- **Standard build**: Target <10 seconds
- **mypyc build**: Target <2 minutes
- **Full test suite**: Target <5 minutes
- **Performance benchmark**: Target <2 minutes

## Development Environment Setup

### Minimal Setup (Fast Iteration)
```bash
# Clone repository
git clone https://github.com/Ousret/charset_normalizer.git
cd charset_normalizer

# Install development dependencies
pip install -r dev-requirements.txt

# Install package in development mode
pip install -e .

# Run tests
pytest tests/
```

### Full Setup (Performance Testing)
```bash
# Install CI requirements (includes nox)
pip install -r ci-requirements.txt

# Clone benchmark dataset
git clone --depth 1 https://github.com/ousret/char-dataset

# Build with mypyc
CHARSET_NORMALIZER_USE_MYPYC=1 pip install -e .

# Run performance benchmarks
python bin/performance.py
```

## Troubleshooting Build Issues

### Common Issues

1. **mypyc compilation fails**:
   ```bash
   # Check mypy passes first
   mypy --strict src/charset_normalizer/module.py

   # Fix type errors before compiling
   ```

2. **C compiler not found**:
   ```bash
   # Install build tools
   sudo apt-get install build-essential  # Ubuntu/Debian
   xcode-select --install                # macOS
   # Install Visual Studio Build Tools on Windows
   ```

3. **Import errors after build**:
   ```bash
   # Clean and rebuild
   pip uninstall charset-normalizer
   rm -rf build/ dist/ *.egg-info src/**/*.so
   pip install -e .
   ```

## Quick Reference Commands

```bash
# Fast development build (no mypyc)
pip install -e .

# Production build (with mypyc)
CHARSET_NORMALIZER_USE_MYPYC=1 pip install -e .

# Run specific test
pytest tests/test_api.py::test_function -v

# Quick benchmark (assumes setup done)
python bin/performance.py

# Full performance test via nox
nox -s performance

# Type check before mypyc
mypy --strict src/charset_normalizer/

# Format and lint
nox -s lint
```

## Performance Workflow Optimization Goals

Based on the research:
- **Reduce iteration time**: <10 seconds for code change → test feedback
- **Fast profiling**: <30 seconds to identify bottlenecks
- **Quick benchmarking**: <2 minutes for performance comparison
- **Minimize rebuilds**: Develop in pure Python, compile only for benchmarks

## Best Practices

### ✅ Do
- Develop without mypyc for fast iteration
- Compile with mypyc only for final benchmarking
- Use nox sessions for reproducible testing
- Cache dependencies and build artifacts
- Profile before optimizing

### ❌ Don't
- Compile with mypyc for every test run
- Run full test suite for every code change
- Skip type checking before mypyc compilation
- Forget to clean build artifacts when switching approaches
- Run all nox sessions when you only need one

## Resources

- **Build configuration**: `setup.py`, `pyproject.toml`
- **nox configuration**: `noxfile.py`
- **CI configuration**: `.github/workflows/ci.yml`
- **Requirements**: `ci-requirements.txt`, `dev-requirements.txt`
