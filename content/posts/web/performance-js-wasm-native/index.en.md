---
title: "Same Computation, Different Runtimes: JavaScript, WebAssembly, and Native C++ Performance"
date: "2026-08-02T16:27:08+09:00"
draft: false
description: "This article compares the execution time of the same logic implemented in JavaScript and C++, using pure JavaScript, C++ compiled to WebAssembly, and native C++. In the prime-counting example, WebAssembly took 1.002 times as long as native C++ compiled with GCC, while pure JavaScript took 1.523 times as long."

categories:
  - Web

tags:
  - emscripten
  - webassembly
  - javascript
  - cpp

ShowToc: true
TocOpen: false
ShowReadingTime: false
---

> [!SUMMARY]
> This article compares the execution time of the same logic implemented in JavaScript and C++, using pure JavaScript, C++ compiled to WebAssembly, and native C++. In the prime-counting example, WebAssembly took 1.002 times as long as native C++ compiled with GCC, while pure JavaScript took 1.523 times as long.

## What Is Being Compared?

This article compares the execution time of three implementations using the same algorithm and input:

- Pure JavaScript running in a browser
- WebAssembly compiled from C++ with Emscripten
- The same C++ code built as a native executable

The comparison measures the execution time of the core computation rather than the total runtime of the program. Startup costs such as downloading files, loading JavaScript, and initializing the WebAssembly module are therefore excluded.

## Test Environment and Methodology

### Test Environment

- Apple M1 Max
- macOS Sonoma 14.6.1
- Chrome 151.0.7922.47
- Emscripten 5.0.3
- Apple Clang 16.0.0
- GCC 15.1.0

### Test Methodology

- All tests were run on the same machine, with the browser tests performed in Chrome. Both the Wasm and native C++ versions were built with `-O3`. Each implementation was warmed up twice and then measured over 30 runs, from which the mean and standard deviation were calculated. The number of warm-up runs was chosen after checking when execution times stabilized. These results apply only to this test environment and example; they do not represent the general performance of JavaScript, WebAssembly, or native code.
- Execution time was measured with `performance.now()` in the browser and `std::chrono::steady_clock` in the native executable. Both measure monotonically increasing elapsed time in milliseconds, although they are different timing facilities, and the resolution of `performance.now()` may be limited for security reasons. Because each measured operation takes tens of milliseconds, the effect of this resolution difference is negligible. Nevertheless, because the values come from different execution environments, they should be interpreted in terms of relative trends rather than absolute timings.

## Example: Counting Primes

### Source Code — JavaScript

```JavaScript
// js_prime_number.js
function countPrimes(limit) {
  if (limit < 2) return 0;
  let count = 0;
  for (let i = 2; i <= limit; ++i) {
    let isPrime = true;
    for (let j = 2; j * j <= i; ++j) {
      if (i % j === 0) {
        isPrime = false;
        break;
      }
    }
    if (isPrime) ++count;
  }
  return count;
}
```

### Source Code — C++

```C++
// prime_number.h
int CountPrimes(int limit);

// prime_number.cpp
#ifdef __EMSCRIPTEN__
#include <emscripten.h>
extern "C" {
  EMSCRIPTEN_KEEPALIVE
#endif // __EMSCRIPTEN__
  int CountPrimes(int limit) {
    if (limit < 2) return 0;
    int count = 0;
    for (int i = 2; i <= limit; ++i) {
      bool isPrime = true;
      for (int j = 2; j * j <= i; ++j) {
        if (i % j == 0) {
          isPrime = false;
          break;
        }
      }
      if (isPrime) ++count;
    }
    return count;
  }
#ifdef __EMSCRIPTEN__
}
#endif // __EMSCRIPTEN__
```

- The implementation uses the same logic as the JavaScript version
- `extern "C"` prevents C++ name mangling, while `EMSCRIPTEN_KEEPALIVE` prevents the function from being removed
  - See [Emscripten Installation and Examples](/en/posts/emscripten-install/#source-code)
- `__EMSCRIPTEN__`
  - Defined when the code is built with `emcc` or `em++`

### Building the Wasm Module

```bash
em++ prime_number.cpp -O3 -o wasm_prime_number.js \
  -s EXPORTED_FUNCTIONS='["_CountPrimes"]' \
  -s MODULARIZE=1 \
  -s EXPORT_NAME='createPrimeModule'
```

- Uses the performance-oriented `-O3` optimization level
- Generates `wasm_prime_number.js` and `wasm_prime_number.wasm`
- Exports the `CountPrimes` function
- `MODULARIZE`
  - Generates an asynchronous factory function instead of a global `Module` object
  - Calling the factory function returns a Promise that resolves to an initialized `Module` instance
  - This option makes it possible to exclude Wasm module loading time from the benchmark
  - For a detailed look at `MODULARIZE`, `EXPORT_NAME`, and Module creation options in general, see [Emscripten Module](/en/posts/emscripten-module/)
- `EXPORT_NAME`
  - Specifies the name of the module factory function
  - A module instance can be created with `const module = await createPrimeModule();`

### Browser Benchmarking Code

```JavaScript
// common.js
function computeStats(times) {
  if (times.length < 2) {
    throw new Error("At least two timed runs are required.");
  }

  const n = times.length;

  // Mean
  let sum = 0;
  for (const time of times) {
    sum += time;
  }
  const mean = sum / n;

  // Variance
  let squaredDiffSum = 0;
  for (const time of times) {
    const diff = time - mean;
    squaredDiffSum += diff * diff;
  }
  const variance = squaredDiffSum / n;
  const stddev = Math.sqrt(variance);

  return { mean, stddev };
}

function runBenchmark(label, warmupRuns, timedRuns, expectedResult, fn) {
  const warmupTimes = [];
  for (let i = 0; i < warmupRuns; ++i) {
    const start = performance.now();
    const result = fn();
    const end = performance.now();
    if (result !== expectedResult) {
      throw new Error(
        `${label} returned ${result}, expected ${expectedResult}`,
      );
    }
    const elapsed = end - start;
    warmupTimes.push(elapsed);
  }

  const times = [];
  for (let i = 0; i < timedRuns; ++i) {
    const start = performance.now();
    const result = fn();
    const end = performance.now();
    if (result !== expectedResult) {
      throw new Error(
        `${label} returned ${result}, expected ${expectedResult}`,
      );
    }
    const elapsed = end - start;
    times.push(elapsed);
  }

  return { label, warmupTimes, times };
}

function waitForPaint() {
  return new Promise((resolve) => {
    requestAnimationFrame(() => setTimeout(resolve, 0));
  });
}
```

- `computeStats`
  - Calculates statistics for the execution times collected over multiple runs
  - Calculates the mean and standard deviation
- `runBenchmark`
  - Performs warm-up runs so the V8 engine has time to optimize the code
  - Executes `fn` for the specified number of timed runs and stores each execution time in `times`
  - Returns the collected execution times
- `waitForPaint`
  - Allows the browser to update the page before the benchmark starts

### Running JavaScript and Wasm in the Browser

```HTML
<!doctype html>
<html>
  <head>
    <title>Emscripten Example-3</title>
    <style>
      table {
        border-collapse: collapse;
      }
      table,
      th,
      td {
        border: 1px solid black;
      }
      th,
      td {
        padding: 8px;
      }
    </style>
  </head>
  <body>
    <h1>JavaScript vs. WebAssembly</h1>
    <p>
      Count the prime numbers up to 1,000,000 after 2 warm-up runs, then measure
      30 runs. Close DevTools before starting the benchmark. Refresh the page to
      run the benchmark again.
    </p>
    <button id="run-benchmark" type="button" disabled>Run Benchmark</button>
    <section id="results" hidden>
      <h2>Results</h2>
      <table>
        <thead>
          <tr>
            <th scope="col">Runtime</th>
            <th scope="col">Mean</th>
            <th scope="col">Stddev</th>
          </tr>
        </thead>
        <tbody id="results-body"></tbody>
      </table>
    </section>
    <script type="text/javascript" src="common.js"></script>
    <script type="text/javascript" src="js_prime_number.js"></script>
    <script type="text/javascript" src="wasm_prime_number.js"></script>
    <script type="text/javascript" src="benchmark.js"></script>
  </body>
</html>
```

- Clicking the button runs both the JavaScript and Wasm implementations
- When both functions finish, the page displays the mean and standard deviation of their execution times

```JavaScript
// benchmark.js
"use strict";

const LAST_NUMBER = 1000000;
const PRIME_COUNT = 78498;
const WARMUP_RUNS = 2;
const TIMED_RUNS = 30;

const btnBenchmark = document.querySelector("#run-benchmark");
const resultsSection = document.querySelector("#results");
const resultsBody = document.querySelector("#results-body");

let wasmModule;

function formatMilliseconds(value) {
  return `${value.toFixed(3)} ms`;
}

function renderResults(results) {
  resultsBody.replaceChildren();

  for (const result of results) {
    const row = document.createElement("tr");
    const values = [
      result.label,
      formatMilliseconds(result.mean),
      formatMilliseconds(result.stddev),
    ];

    for (const value of values) {
      const cell = document.createElement("td");
      cell.textContent = value;
      row.append(cell);
    }

    resultsBody.append(row);
  }

  resultsSection.hidden = false;
}

async function runBrowserBenchmark() {
  btnBenchmark.disabled = true;
  resultsSection.hidden = true;

  await waitForPaint();

  const jsResult = runBenchmark(
    "JavaScript",
    WARMUP_RUNS,
    TIMED_RUNS,
    PRIME_COUNT,
    () => countPrimes(LAST_NUMBER),
  );
  const wasmResult = runBenchmark(
    "WebAssembly",
    WARMUP_RUNS,
    TIMED_RUNS,
    PRIME_COUNT,
    () => wasmModule._CountPrimes(LAST_NUMBER),
  );
  const jsStat = computeStats(jsResult.times);
  const wasmStat = computeStats(wasmResult.times);
  const results = [
    {
      label: "JavaScript",
      mean: jsStat.mean,
      stddev: jsStat.stddev,
    },
    {
      label: "WebAssembly",
      mean: wasmStat.mean,
      stddev: wasmStat.stddev,
    },
  ];
  renderResults(results);

  btnBenchmark.disabled = false;
}

async function initializeBenchmark() {
  wasmModule = await createPrimeModule();
  btnBenchmark.disabled = false;
}

btnBenchmark.addEventListener("click", runBrowserBenchmark);
initializeBenchmark();
```

- `runBrowserBenchmark`
  - Counts all prime numbers up to 1,000,000
- Runs two warm-up iterations followed by 30 measured iterations
- The number of warm-up runs was chosen after checking when execution times stabilized

### Native Benchmarking Code

```C++
// main.cpp
#include "prime_number.h"

#include <chrono>
#include <cmath>
#include <iostream>
#include <stdexcept>
#include <vector>

namespace {

const int LAST_NUMBER = 1000000;
const int PRIME_COUNT = 78498;
const int WARMUP_RUNS = 2;
const int TIMED_RUNS = 30;

double ElapsedMs(std::chrono::steady_clock::time_point start,
                 std::chrono::steady_clock::time_point end) {
  return std::chrono::duration<double, std::milli>(end - start).count();
}

struct Stats {
  double mean;
  double stddev;
};

Stats ComputeStats(const std::vector<double>& times) {
  if (times.size() < 2) {
    throw std::invalid_argument("At least two timed runs are required.");
  }

  const size_t n = times.size();

  // Mean
  double sum = 0.0;
  for (const double time : times) {
    sum += time;
  }
  const double mean = sum / n;

  // Variance
  double squaredDiffSum = 0.0;
  for (const double time : times) {
    const double diff = time - mean;
    squaredDiffSum += diff * diff;
  }
  const double variance = squaredDiffSum / n;
  const double stddev = std::sqrt(variance);

  return { mean, stddev };
}

}  // namespace

int main() {
  std::vector<double> warmupTimes;
  warmupTimes.reserve(WARMUP_RUNS);
  std::vector<double> times;
  times.reserve(TIMED_RUNS);

  for (int i = 0; i < WARMUP_RUNS; ++i) {
    const auto start = std::chrono::steady_clock::now();
    const int result = CountPrimes(LAST_NUMBER);
    const auto end = std::chrono::steady_clock::now();
    if (result != PRIME_COUNT) {
      throw std::runtime_error(
          "Native returned an unexpected result during warmup.");
    }
    const double elapsed = ElapsedMs(start, end);
    warmupTimes.push_back(elapsed);
  }

  for (int i = 0; i < TIMED_RUNS; ++i) {
    const auto start = std::chrono::steady_clock::now();
    const int result = CountPrimes(LAST_NUMBER);
    const auto end = std::chrono::steady_clock::now();
    if (result != PRIME_COUNT) {
      throw std::runtime_error(
          "Native returned an unexpected result during timed run.");
    }
    const double elapsed = ElapsedMs(start, end);
    times.push_back(elapsed);
  }

  Stats s = ComputeStats(times);
  std::cout << "--- Native stats (n=" << TIMED_RUNS << ") ---" << std::endl;
  std::cout << "mean=" << s.mean << " ms, stddev=" << s.stddev << " ms"
            << std::endl;

  return 0;
}
```

### Building and Running the Native Version

```bash
# Apple Clang
clang++ -O3 -std=c++17 main.cpp prime_number.cpp -o native_prime
./native_prime

# GCC
g++-15 -O3 -std=c++17 main.cpp prime_number.cpp -o native_prime_gcc
```

- The native version was built with both Apple Clang 16.0.0 and GCC 15.1.0. Both builds used the same `-O3` optimization level.

### Example Code

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-3

## Results and Conclusion

| Platform             | Mean (ms) | Stddev (ms) | Relative to Native (GCC) |
| -------------------- | --------- | ----------- | ------------------------ |
| JavaScript           | 91.203    | 0.916       | 1.523× as long           |
| Wasm                 | 60.007    | 0.085       | 1.002× as long           |
| Native (Apple Clang) | 61.709    | 0.505       | 1.030× as long           |
| Native (GCC)         | 59.889    | 0.379       | -                        |

- Compared with native C++ compiled by GCC, Wasm took about 1.002 times as long, an increase of 0.2%, while JavaScript took about 1.523 times as long, an increase of 52.3%.
- Wasm was slightly faster than the native C++ build produced by Apple Clang, possibly because of differences in backend optimization.
- Results varied slightly between runs.
- Reversing the execution order of the JavaScript and Wasm benchmarks did not produce a meaningful difference in execution time.

## FAQ

- Does this mean there is no performance difference between Wasm and native code?
  - This example needs to pass only one integer argument to invoke the Wasm function, so there is very little interaction across the `JavaScript glue code` ↔ `Wasm` boundary. This structure allows Wasm to perform close to its best case.
  - In other workloads, frequent interaction across the `user code` ↔ `JavaScript glue code` ↔ `Wasm` boundary adds call overhead. Passing large values such as strings or arrays can also require copying data into Wasm linear memory and copying results back out, which may prevent Wasm from reaching native performance. See [Calling C++ Functions from JavaScript - ccall, cwrap](/en/posts/call-cpp-from-js/) for a detailed walkthrough of this process.
  - Wasm code should therefore be designed to minimize interaction with JavaScript glue code when performance matters.
- Why is JavaScript only about 1.5 times slower?
  - JavaScript is often expected to be several times slower, but this example consists of type-stable integer arithmetic and simple loops. V8's JIT compiler can optimize this into machine code that performs relatively close to native code. The warm-up runs give the JIT time to apply these optimizations.
  - The gap could be larger for code involving many objects, strings, or dynamically changing types.
- What kinds of workloads benefit from Wasm?
  - Wasm is most effective for CPU-intensive computation that rarely crosses the JavaScript–Wasm boundary, such as cryptography, image and video processing, physics simulation, and compression. It can be less effective for workloads that frequently cross the boundary, such as DOM manipulation or repeated string processing.
- Why was loading time excluded?
  - Wasm has startup costs for downloading, compiling, and instantiating the `.wasm` file. For a short task that runs only once, these costs can make the overall operation slower than pure JavaScript.
  - This article focuses on computation rather than startup cost, so all three implementations measure only the execution time of the core computation. When deciding whether to use Wasm in a real service, startup costs should be considered as well.
