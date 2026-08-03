---
title: "같은 연산, 다른 실행 환경: JavaScript·WebAssembly·네이티브 C++ 성능 비교"
date: "2026-08-02T16:27:08+09:00"
draft: false
description: "동일한 로직을 순수 JavaScript, WebAssembly로 컴파일한 C++, 네이티브로 컴파일한 C++을 실행하여 성능 차이를 비교합니다."

categories:
  - Web

tags:
  - emscripten
  - webassembly
  - javascript

ShowToc: true
TocOpen: false
ShowReadingTime: false
---

> [!SUMMARY]
> 동일한 로직의 코드를 JavaScript와 C++로 작성하고 순수 JavaScript 코드, WebAssembly로 빌드된 C++ 코드, Native로 빌드된 C++ 코드의 실행 시간을 비교한다. 소수 찾기 예제를 통해 WebAssembly의 실행 시간은 Native 대비 1.002배, 순수 JavaScript의 실행 시간은 1.523배 느린 것으로 나타났다.

## 무엇을 비교하는가

이 글에서는 동일한 알고리즘과 입력값을 사용해 다음 세 가지 구현의 실행 시간을 비교한다.

- 브라우저에서 실행하는 순수 JavaScript
- C++ 코드를 Emscripten으로 컴파일한 WebAssembly
- 동일한 C++ 코드를 네이티브 실행 파일로 빌드한 버전

비교 대상은 프로그램의 전체 실행 시간이 아니라 핵심 연산 함수의 실행 시간이다. 따라서 파일 다운로드, JavaScript 로딩, WebAssembly 모듈 초기화와 같은 시작 비용은 측정에 포함하지 않는다.

## 테스트 환경과 측정 방법

### 테스트 환경

- Apple M1 Max
- macOS Sonoma 14.6.1
- Chrome 151.0.7922.47
- Emscripten 5.0.3
- Apple Clang 16.0.0
- GCC 15.1.0

### 테스트 방법

- 테스트는 동일한 기기에서 Chrome을 사용해 진행했다. Wasm과 네이티브 C++은 모두 `-O3`로 빌드했으며, 각 구현을 2회 워밍업한 뒤 30회 측정하여 평균과 표준편차를 계산했다. 실행 시간이 안정화 되는 시점을 확인하여 warm-up 횟수를 결정하였다. 결과는 이 테스트 환경과 예제 코드에 한정되며, 일반적인 JavaScript, WebAssembly, 네이티브 성능을 대표하지 않는다.
- 실행 시간은 브라우저에서 `performance.now()`, 네이티브에서 `std::chrono::steady_clock`으로 측정했다. 두 타이머 모두 단조 증가하는 경과 시간을 밀리초 단위로 재지만 측정 도구 자체가 다르고, 브라우저의 `performance.now()`는 보안상 해상도가 제한될 수 있다. 다만 측정 대상이 수십 밀리초 단위의 연산이라 이러한 해상도 차이가 결과에 미치는 영향은 미미하다. 그럼에도 서로 다른 측정 환경에서 얻은 값이므로, 절대적인 수치보다 세 구현 간의 상대적인 경향에 초점을 두고 해석하는 것이 적절하다.

## 예제 - 소수 찾기

### 소스 코드 - JavaScript Version

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

### 소스 코드 - C++ Version

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

- JS 버전과 동일한 로직으로 작성되었음
- `extern "C"`와 `EMSCRIPTEN_KEEPALIVE`는 각각 네임맹글링과 함수 삭제 방지를 위해 추가됨
  - [Emscripten 설치 및 예제](/posts/emscripten-install/#소스-코드) 참고
- `__EMSCRIPTEN__`
  - `emcc`나 `em++`로 빌드하면 활성화 됨

### Wasm 빌드하기

```bash
em++ prime_number.cpp -O3 -o wasm_prime_number.js \
  -s EXPORTED_FUNCTIONS='["_CountPrimes"]' \
  -s MODULARIZE=1 \
  -s EXPORT_NAME='createPrimeModule'
```

- 성능 평가를 위해 가장 높은 수준의 최적화 옵션(`-O3`)을 사용
- `wasm_prime_number.js`와 `wasm_prime_number.wasm`이 생성됨
- `CountPrimes`함수를 export
- `MODULARIZE`
  - 전역 Module 객체 대신 비동기 팩토리 함수가 생성됨
  - 이 팩토리 함수를 호출하면 초기화된 Module 인스턴스로 resolve 되는 Promise가 반환됨
  - Wasm 모듈의 로딩 시간을 테스트 시간에서 제외하기 위해 사용하는 옵션
- `EXPORT_NAME`
  - 모듈 팩토리 함수의 이름
  - `const module = await createPrimeModule();`과 같은 형태로 module의 인스턴스를 생성할 수 있음

### 성능 측정을 위한 코드 - 브라우저

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
  - 여러 번 반복한 테스트의 실행 시간에 대한 통계를 계산
  - 평균, 표준편차를 계산
- `runBenchmark`
  - V8 엔진이 충분히 최적화를 할 수 있도록 warm-up을 수행
  - 그 후 timedRuns 횟수 만큼 `fn`함수를 실행하여 결과를 times에 저장
  - 측정된 실행 시간을 times에 모아 반환
- `waitForPaint`
  - 브라우저의 화면 갱신을 위한 함수

### JS와 Wasm 실행하기 - 브라우저

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

- 버튼을 클릭하면 JavaScript 버전과 Wasm 버전의 함수를 실행
- 함수의 실행이 완료되면 각 버전의 실행 시간의 평균과 표준편차를 보여줌

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

- runBrowserBenchmark
  - 1,000,000까지의 소수를 계산
- 2번의 warm-up 후 30번의 테스트를 수행하여 결과를 얻음
- 실행 시간이 안정화 되는 시점을 확인하여 warm-up 횟수를 정함

### 성능 측정을 위한 코드 - Native

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

### Native 빌드 하기 및 실행

```bash
# Apple Clang
clang++ -O3 -std=c++17 main.cpp prime_number.cpp -o native_prime
./native_prime

# GCC
g++-15 -O3 -std=c++17 main.cpp prime_number.cpp -o native_prime_gcc
```

- Native의 경우 Apple Clang 16.0.0, GCC 15.1.0 두 가지 컴파일러를 이용하여 빌드하였고, 최적화 옵션은 모두 동일하게 `-O3`를 사용하였다.

### 예제 코드

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-3

## 실행 시간 정리 및 결론

| Platform             | Mean (ms) | Stddev (ms) | Relative to Native (GCC) |
| -------------------- | --------- | ----------- | ------------------------ |
| JS                   | 91.203    | 0.916       | 1.523× slower            |
| Wasm                 | 60.007    | 0.085       | 1.002× slower            |
| Native (Apple Clang) | 61.709    | 0.505       | 1.030× slower            |
| Native (GCC)         | 59.889    | 0.379       | -                        |

- 네이티브 C++(GCC)와 비교했을 때 실행 시간은 Wasm이 약 1.002배로 0.2% 길었고, JavaScript는 약 1.523배로 52.3% 길었다.
- 네이티브 C++(Apple Clang)과 Wasm을 비교해보면 Wasm이 약간 더 빠르게 수행되는 것을 확인할 수 있고, 이는 백엔드 최적화의 차이일 수 있음
- 실행할 때마다 약간의 오차는 발생함
- JavaScript와 Wasm의 실행 순서를 변경해도 실행 시간의 유의미한 차이는 발생하지 않았음

## FAQ

- Wasm과 Native의 성능 차이가 없다고 보면 되나요?
  - 위의 예제는 Wasm 함수를 돌리기 위해 정수 인자 하나만 전달하면 되는 형태로 `JS 접착 코드` ↔ `Wasm` 상호작용이 거의 없어 Wasm의 성능을 극대화할 수 있는 형태입니다.
  - 다른 문제에서 `사용자` ↔ `JS 접착 코드` ↔ `Wasm` 상호작용이 자주 발생하는 경우, 경계를 넘는 호출 자체의 오버헤드가 쌓입니다. 특히 문자열이나 배열처럼 큰 데이터를 주고받을 때는 해당 데이터를 Wasm의 선형 메모리로 복사해 넣고 결과를 다시 꺼내는 과정이 필요해, 그 비용 때문에 Native만큼의 성능이 나오지 않을 수 있습니다.
  - 따라서 Wasm이 동작할 때는 JS 접착 코드와의 상호작용을 최소화하도록 설계하는 것이 성능을 높일 수 있습니다.
- 왜 JavaScript가 1.5배밖에 안 느린가요?
  - 흔히 JS는 몇 배씩 느릴 거라 예상하는데, 이 예제는 정수 산술과 단순 루프처럼 타입이 안정적인 코드라 V8의 JIT(TurboFan)가 거의 네이티브에 가까운 기계어로 컴파일합니다. Warm-up을 두는 이유도 이 JIT 최적화가 걸릴 시간을 주기 위해서입니다.
  - 반대로 객체·문자열·동적 타입이 많은 코드였다면 격차가 더 커질 수 있습니다.
- 어떤 작업에 Wasm이 유리한가요?
  - JS와 Wasm의 경계를 거의 넘지 않는 순수 계산(암호화, 이미지·비디오 처리, 물리 시뮬레이션, 압축 등 CPU 집약적 작업)에서 이점이 크고, DOM 조작이나 잦은 문자열 처리처럼 경계를 자주 넘나드는 작업에는 오히려 불리할 수 있습니다.
- 왜 로딩 시간을 측정에서 뺐나요?
  - Wasm은 `.wasm` 파일 다운로드와 컴파일·인스턴스화 비용이 있어서, 짧게 한 번 실행하는 작업이라면 이 시작 비용 때문에 순수 JS보다 전체적으로 느릴 수도 있습니다.
  - 이 글의 목적은 시작 비용이 아니라 순수 연산 성능을 비교하는 것이라, 세 구현 모두 동일하게 핵심 연산 함수의 실행 시간만 측정했습니다. 실제 서비스에서 Wasm 도입을 판단할 때는 이 시작 비용까지 함께 고려해야 합니다.
