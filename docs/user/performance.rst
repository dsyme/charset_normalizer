Performance Optimization Guide
===============================

This guide explains how to optimize charset_normalizer for your specific use case, covering performance features, trade-offs, and best practices.

Overview
--------

charset_normalizer has been extensively optimized to provide excellent performance while maintaining high detection accuracy. The library achieves **~7x faster** performance than chardet on average through several optimization techniques.

Key Performance Features
------------------------

1. **mypyc Compilation** (automatic in installed wheels)
2. **Small Input Fast Path** (automatic for inputs < 100 bytes)
3. **Optional Language Detection** (user-controlled via parameter)
4. **Early Stop Logic** (automatic when confident match found)

Quick Performance Tips
----------------------

**For maximum speed:**

.. code-block:: python

    from charset_normalizer import from_bytes

    # Disable language detection when you don't need it
    result = from_bytes(data, enable_language_detection=False)

    # Use appropriate parameters for your use case
    result = from_bytes(
        data,
        steps=3,  # Reduce from default 5 for faster detection
        threshold=0.25,  # Slightly higher than default 0.20
        enable_language_detection=False
    )

**For maximum accuracy:**

.. code-block:: python

    # Use defaults (optimized for accuracy)
    result = from_bytes(data)

    # Or increase granularity
    result = from_bytes(
        data,
        steps=7,  # More chunks tested
        threshold=0.15  # Lower threshold = more strict
    )

Performance Features in Detail
-------------------------------

Optional Language Detection
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``enable_language_detection`` parameter allows you to skip expensive language detection (coherence ratio calculation) when you don't need language information.

**When to disable:**

- You only need encoding, not language identification
- Processing large batches where language isn't relevant
- Known language contexts (e.g., English-only system)
- Binary detection scenarios
- Real-time/streaming applications requiring minimal latency

**Performance impact:**

- **18% faster** on average (35.0 → 41.2 files/sec)
- **32x faster** on very small inputs (< 100 bytes)
- **4.3ms saved** per file on typical workloads

**Example:**

.. code-block:: python

    from charset_normalizer import from_bytes

    # With language detection (default)
    result = from_bytes(data)
    print(result.best().encoding)  # e.g., 'utf-8'
    print(result.best().languages)  # e.g., ['English']

    # Without language detection (faster)
    result = from_bytes(data, enable_language_detection=False)
    print(result.best().encoding)  # e.g., 'utf-8'
    print(result.best().languages)  # [] (empty)

**Trade-offs:**

- ✅ Significant speed improvement
- ✅ Zero overhead when not needed
- ❌ No language information in results
- ❌ Slightly lower accuracy for ambiguous encodings (99% vs 100%)

Small Input Optimization
~~~~~~~~~~~~~~~~~~~~~~~~~

For inputs < 100 bytes, charset_normalizer automatically uses a fast path that tests only the most common ~40 encodings instead of all 100+ IANA-supported encodings.

**Automatic behavior:**

- Triggered for sequences < 100 bytes without BOM
- Tests Western European, Cyrillic, CJK, and other common encodings
- Bypassed when ``cp_isolation`` parameter is used
- Bypassed when BOM/SIG is detected

**Performance impact:**

- **2.44x faster** on small inputs (144% speedup)
- **3.2x faster** for Latin-1 text
- **2.6x faster** for Japanese text
- **2.4x faster** for Cyrillic text

**No configuration needed** - this optimization is automatic.

.. code-block:: python

    # Automatically optimized for small inputs
    short_text = b"Hello World"
    result = from_bytes(short_text)  # Fast path automatically used

    # Normal path for larger inputs
    long_text = b"x" * 200
    result = from_bytes(long_text)  # Full encoding list tested

**Covered encodings:**

- Western European: latin_1, windows_1252, iso8859_1, cp1252
- Central European: cp1250, windows_1250, iso8859_2
- Cyrillic: cp1251, windows_1251, iso8859_5, koi8_r, koi8_u, cp866
- Japanese: cp932, shift_jis, euc_jp, iso2022_jp
- Chinese: gb2312, gb18030, big5, hz
- Korean: euc_kr, cp949, johab
- Greek: iso8859_7, cp1253
- Turkish: iso8859_9, cp1254
- Hebrew: iso8859_8, cp1255
- Arabic: iso8859_6, cp1256

Early Stop Logic
~~~~~~~~~~~~~~~~

charset_normalizer automatically stops testing encodings when it finds a high-confidence match. This is particularly effective for UTF-8 and ASCII content.

**Automatic behavior:**

- Stops immediately when UTF-8/ASCII detected with 0% mess ratio
- Stops after testing UTF-8/ASCII when specified encoding has low mess ratio
- No user configuration required

**Performance impact:**

- **Significant speedup** for clean UTF-8/ASCII files
- **Minimal overhead** when detection requires more investigation
- Most effective for Western European and ASCII-compatible content

Parameter Tuning
~~~~~~~~~~~~~~~~

The ``steps`` and ``chunk_size`` parameters control detection granularity.

**Default values (optimized for accuracy):**

- ``steps=5``: Tests 5 chunks per encoding
- ``chunk_size=512``: 512 bytes per chunk

**Trade-off guidance:**

.. code-block:: python

    # Faster detection (less accurate)
    result = from_bytes(data, steps=3, chunk_size=256)

    # More accurate detection (slower)
    result = from_bytes(data, steps=7, chunk_size=1024)

    # Very fast detection (use with caution)
    result = from_bytes(data, steps=1, chunk_size=512)

**Recommendations:**

- **Default (steps=5, chunk_size=512)**: Best for most use cases
- **Fast (steps=3)**: When speed is critical and data is relatively clean
- **Accurate (steps=7)**: When processing ambiguous or corrupted data
- **Steps=1**: Only for very clean, unambiguous data

Installation Considerations
----------------------------

mypyc Compiled Wheels
~~~~~~~~~~~~~~~~~~~~~

charset_normalizer provides pre-compiled wheels with mypyc optimization for major platforms (Linux, macOS, Windows) and Python versions (3.7-3.13).

**To use compiled wheels (recommended):**

.. code-block:: bash

    pip install charset-normalizer

**Performance impact:**

- **~2-3x faster** than pure Python (cumulative from multiple modules)
- **Zero code changes** required
- **Automatic** when installing from PyPI

**Verify mypyc is active:**

.. code-block:: bash

    normalizer --version
    # Look for "SpeedUp ON" in output

**To disable mypyc (for debugging):**

.. code-block:: bash

    pip install charset-normalizer --no-binary charset-normalizer

Use Case Examples
-----------------

Batch Processing Large Files
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Scenario:** Processing thousands of large text files for encoding normalization.

**Recommendation:**

.. code-block:: python

    from charset_normalizer import from_bytes

    def process_files(file_paths):
        for path in file_paths:
            with open(path, 'rb') as f:
                data = f.read()

            # Use defaults (already optimized)
            result = from_bytes(data)
            encoding = result.best().encoding

            # Process with detected encoding
            with open(path, 'r', encoding=encoding) as f:
                content = f.read()
                # ... process content

**Why:** Default parameters are already optimized for large files. mypyc compilation and early stop logic provide excellent performance.

Real-time Log Processing
~~~~~~~~~~~~~~~~~~~~~~~~

**Scenario:** Detecting encoding of log lines in real-time with minimal latency.

**Recommendation:**

.. code-block:: python

    from charset_normalizer import from_bytes

    def process_log_line(line_bytes):
        # Disable language detection for speed
        result = from_bytes(
            line_bytes,
            enable_language_detection=False,
            steps=3,  # Reduce steps for speed
            threshold=0.25  # Slightly higher threshold
        )
        return result.best().encoding

    # Process incoming log lines
    for line in log_stream:
        encoding = process_log_line(line)
        decoded_line = line.decode(encoding)
        # ... process log line

**Why:** Language detection not needed for logs. Reduced steps and higher threshold trade some accuracy for speed. Small input fast path automatically helps.

API Response Processing
~~~~~~~~~~~~~~~~~~~~~~~

**Scenario:** Detecting encoding of HTTP response bodies in a web scraper.

**Recommendation:**

.. code-block:: python

    from charset_normalizer import from_bytes
    import requests

    def fetch_and_detect(url):
        response = requests.get(url)

        # Fast detection for typical web content
        result = from_bytes(
            response.content,
            enable_language_detection=False  # Optional: disable if not needed
        )

        encoding = result.best().encoding
        return response.content.decode(encoding)

    # Use in scraping workflow
    for url in urls:
        content = fetch_and_detect(url)
        # ... extract data from content

**Why:** Web content is typically UTF-8 or Latin-1, which early stop logic handles efficiently. Language detection optional depending on use case.

CSV/Database Field Processing
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Scenario:** Detecting encoding of database text fields or CSV columns with mixed encodings.

**Recommendation:**

.. code-block:: python

    from charset_normalizer import from_bytes

    def detect_field_encoding(field_data):
        if isinstance(field_data, str):
            field_data = field_data.encode('utf-8')

        # Very fast detection for short fields
        result = from_bytes(
            field_data,
            enable_language_detection=False,
            steps=3
        )
        return result.best().encoding

    # Process database records
    for record in database.query(\"SELECT * FROM table\"):
        for field in record:
            if is_text_field(field):
                encoding = detect_field_encoding(field)
                # ... normalize to UTF-8

**Why:** Database fields are typically short, so small input fast path helps. Language detection not needed. Reduced steps speeds up processing.

File Upload Validation
~~~~~~~~~~~~~~~~~~~~~~

**Scenario:** Validating that uploaded files are text (not binary) and detecting their encoding.

**Recommendation:**

.. code-block:: python

    from charset_normalizer import from_bytes, is_binary

    def validate_text_upload(file_data):
        # Check if binary first
        if is_binary(file_data):
            raise ValueError(\"Binary file not allowed\")

        # Detect encoding with full accuracy
        result = from_bytes(file_data)

        if not result:
            raise ValueError(\"Unable to detect text encoding\")

        encoding = result.best().encoding
        confidence = 1.0 - result.best().mess

        if confidence < 0.8:
            raise ValueError(f\"Low confidence encoding detection: {confidence:.1%}\")

        return encoding

    # Use in upload handler
    uploaded_file = request.files['document']
    encoding = validate_text_upload(uploaded_file.read())

**Why:** Accuracy more important than speed for validation. Use defaults for maximum reliability.

Troubleshooting Performance Issues
-----------------------------------

Slow Detection on Large Files
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Problem:** Detection takes too long on files > 1MB.

**Solutions:**

1. **Reduce steps:**

   .. code-block:: python

       result = from_bytes(data, steps=3)

2. **Disable language detection:**

   .. code-block:: python

       result = from_bytes(data, enable_language_detection=False)

3. **Process only a sample:**

   .. code-block:: python

       # Detect from first 50KB only
       result = from_bytes(data[:50000])

High CPU Usage in Batch Processing
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**Problem:** CPU usage too high when processing many files.

**Solutions:**

1. **Use multiprocessing (for CPU-bound workloads):**

   .. code-block:: python

       from multiprocessing import Pool
       from charset_normalizer import from_path

       def detect_file(path):
           result = from_path(path, enable_language_detection=False)
           return (path, result.best().encoding)

       with Pool(processes=4) as pool:
           results = pool.map(detect_file, file_paths)

2. **Add delay between detections:**

   .. code-block:: python

       import time

       for path in file_paths:
           result = from_path(path)
           # ... process result
           time.sleep(0.01)  # 10ms delay

3. **Filter files before detection:**

   .. code-block:: python

       # Skip small files that are likely ASCII
       for path in file_paths:
           if os.path.getsize(path) < 1024:
               # Assume UTF-8 for very small files
               continue
           result = from_path(path)

Memory Usage Concerns
~~~~~~~~~~~~~~~~~~~~~~

**Problem:** High memory usage when processing many large files.

**Solutions:**

1. **Process files one at a time:**

   .. code-block:: python

       # Bad: loads all files into memory
       results = [from_bytes(open(f, 'rb').read()) for f in files]

       # Good: processes one file at a time
       for file_path in files:
           result = from_path(file_path)
           encoding = result.best().encoding
           # ... process immediately

2. **Use streaming for very large files:**

   .. code-block:: python

       # Detect from first chunk only
       with open(large_file, 'rb') as f:
           chunk = f.read(100000)  # Read first 100KB
           result = from_bytes(chunk)

Performance Benchmarking
------------------------

Built-in Performance Testing
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

charset_normalizer includes a built-in performance benchmark script.

**Run performance tests:**

.. code-block:: bash

    # Install test dataset
    git clone --depth 1 https://github.com/ousret/char-dataset

    # Run performance comparison vs chardet
    nox -s performance

**Interpret results:**

- **Avg speedup**: Overall performance multiplier vs chardet
- **50th percentile**: Typical file performance
- **95th/99th percentile**: Worst-case performance
- **Files/sec**: Throughput metric

**Example output:**

.. code-block:: text

    --> Charset-Normalizer / Chardet: Performance Сomparison
       --> Avg: x6.81
       --> 99th: x3.18
       --> 95th: x1.88
       --> 50th: x1.71

This means charset_normalizer is 6.81x faster on average, with consistently better performance across all percentiles.

Custom Benchmarking
~~~~~~~~~~~~~~~~~~~~

For your specific use case, create custom benchmarks:

.. code-block:: python

    import time
    from charset_normalizer import from_bytes

    def benchmark_detection(test_files, iterations=100):
        total_time = 0

        for _ in range(iterations):
            for file_path in test_files:
                with open(file_path, 'rb') as f:
                    data = f.read()

                start = time.perf_counter()
                result = from_bytes(data)
                end = time.perf_counter()

                total_time += (end - start)

        avg_time = (total_time / (len(test_files) * iterations)) * 1000
        throughput = (len(test_files) * iterations) / total_time

        print(f\"Average detection time: {avg_time:.2f}ms\")
        print(f\"Throughput: {throughput:.1f} files/sec\")

Performance Regression Detection
---------------------------------

charset_normalizer includes automated performance regression detection in CI to protect performance gains.

**For contributors:**

The CI pipeline automatically checks for performance regressions on every PR. If performance degrades by > 10%, the CI will fail with a clear message indicating which metrics regressed.

**Local regression testing:**

.. code-block:: bash

    # Create performance baseline
    python bin/performance_regression.py --save-baseline

    # Make changes to code
    # ...

    # Check for regressions
    python bin/performance_regression.py --check --threshold 10.0

See ``docs/PERFORMANCE_REGRESSION_DETECTION.md`` for full details.

Best Practices Summary
----------------------

**Always:**

- Use pre-compiled wheels from PyPI (automatic with ``pip install``)
- Verify mypyc is active with ``normalizer --version``
- Use defaults unless you have specific requirements

**Consider disabling language detection when:**

- You only need encoding information
- Processing structured data (logs, CSVs, databases)
- Language is already known from context
- Maximizing throughput is critical

**Consider reducing steps when:**

- Processing very large batches of files
- Real-time/streaming applications
- Data is relatively clean and unambiguous
- You can tolerate slightly lower accuracy (e.g., 95% vs 97%)

**Keep defaults when:**

- Accuracy is more important than speed
- Processing user uploads or arbitrary data
- Detection results used for critical decisions
- Ambiguous or corrupted data expected

**Avoid:**

- Processing entire multi-MB files when a sample would suffice
- Using ``steps=1`` on ambiguous data
- Disabling mypyc without good reason
- Over-optimizing before profiling

Further Reading
---------------

- :doc:`getstarted` - Basic usage examples
- :doc:`advanced_search` - Encoding detection customization
- :doc:`support` - Supported encodings and languages
- `Performance Regression Detection Guide <../PERFORMANCE_REGRESSION_DETECTION.html>`_ - For contributors

Contributing Performance Improvements
--------------------------------------

See the ``.github/copilot/instructions/perf-*.md`` files for detailed performance engineering guides covering:

- Algorithm optimization strategies
- Build and workflow performance
- Measurement and benchmarking methodologies

Performance issues and optimization ideas are welcome on the `GitHub issue tracker <https://github.com/Ousret/charset_normalizer/issues>`_.
