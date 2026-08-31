---
title: JavaScript와 C++로 배열 주고 받기 - JSON, Standard Library, Memory View
date: 2026-08-30
draft: false
description: "JavaScript에서 사용하기 위해 C++ 함수를 내보낼 때, 인자로 배열을 넘기는 방법에 대해 소개한다. 첫 번째로 JSON 문자열을 이용하여 데이터를 전달하고 파싱하여 사용하는 방법, 두 번째 Standard Library를 Embind로 바인딩하여 사용하는 방법, 세 번째로 Memory View를 이용하여 Wasm 메모리에 직접 접근하는 방법에 대해 예제를 통해 설명한다. 마지막으로 세 가지 방법의 장단점을 비교한다."
categories:
  - Web
tags:
  - emscripten
  - json
  - memory-view
ShowToc: true
TocOpen: false
ShowReadingTime: false
---

> [!SUMMARY]
> JavaScript에서 사용하기 위해 C++ 함수를 내보낼 때, 인자로 배열을 넘기는 방법에 대해 소개한다. 첫 번째로 JSON 문자열을 이용하여 데이터를 전달하고 파싱하여 사용하는 방법, 두 번째 Standard Library를 Embind로 바인딩하여 사용하는 방법, 세 번째로 Memory View를 이용하여 Wasm 메모리에 직접 접근하는 방법에 대해 예제를 통해 설명한다. 마지막으로 세 가지 방법의 장단점을 비교한다.

## JSON 문자열 이용하기

- 데이터를 JSON 문자열로 변환하여 전달하고 전달받음. 실질적으로는 `std::string`을 이용하여 데이터를 전달
- 전달 받은 JSON 문자열을 배열이나 object 타입으로 변환하여 사용
- 배열 이외에도 object 타입을 전달할 수 있다는 장점이 있음
- JSON object와 JSON 문자열의 상호 변환 과정에서 오버헤드가 발생하기 때문에, 크기가 큰 데이터의 경우에는 추천하지 않음

### `nlohmann-json`

- C++용 JSON 라이브러리. JavaScript와 달리 C++에서는 기본적으로 JSON을 사용할 수 없음
- Header-only 라이브러리. `single_include/nlohmann/json.hpp`만 포함하여 사용할 수 있음
- https://github.com/nlohmann/json
- Version - 3.12.0

### 소스 코드

```C++
// array_json.cpp
#include "nlohmann/json.hpp"
#include <string>
#include <iostream>
#include <emscripten/bind.h>

using json = nlohmann::json;

std::string GetJsonStr(bool isArray) {
  if (isArray) {
    json j = json::array({"Hello", "WebAssembly", "and", "JSON"});
    return j.dump();
  } else {
    json j = json::object({{"Compiler", "Emscripten"}, {"Age", 11}, {"Language", "C++"}});
    return j.dump();
  }
}

void LoadJsonStr(const std::string& jsonStr) {
  auto j = json::parse(jsonStr);

  if (j.is_array()) {
    for (const auto& item : j) {
      std::cout << item << std::endl;
    }
    return;
  }

  if (j.is_object()) {
    for (const auto& [key, value] : j.items()) {
      std::cout << key << ": " << value << std::endl;
    }
    return;
  }
}

EMSCRIPTEN_BINDINGS(my_module) {
  emscripten::function("getJsonStr", &GetJsonStr);
  emscripten::function("loadJsonStr", &LoadJsonStr);
}

// index.html
<!doctype html>
<html>
  <head>
    <title>Emscripten Example-11</title>
  </head>
  <body>
    <script async type="text/javascript" src="array_json.js"></script>
  </body>
</html>
```

- `GetJsonStr`
  - 인자가 true이면 JSON 배열을 JSON 문자열로 변환하여 리턴
  - 인자가 false이면 JSON object를 JSON 문자열로 변환하여 리턴
- `LoadJsonStr`
  - 전달 받은 JSON 문자열을 파싱하여 배열이나 object를 출력

### 빌드하기

```bash
em++ array_json.cpp -o array_json.js -lembind
```

### 실행 결과

![pass json](images/array_with_json.png)
_그림 1. C++과 JavaScript에서 JSON 문자열을 이용하여 데이터를 상호 주고 받고 있는 모습_

### 실행 스크립트

```JavaScript
// Array data exchange
const jsonArrayStr = Module.getJsonStr(true);  // C++ → JS로 문자열 전달
const array = JSON.parse(jsonArrayStr);        // JS 문자열 → JS array
console.log(array);                            // JS array 출력
Module.loadJsonStr(JSON.stringify(array));     // JS array → JS 문자열 → C++에서 출력

// Object data exchange
const jsonObjStr = Module.getJsonStr(false);  // C++ → JS로 문자열 전달
const obj = JSON.parse(jsonObjStr);           // JS 문자열 → JS object
console.log(obj);                             // JS object 출력
Module.loadJsonStr(JSON.stringify(obj));      // JS object → JS 문자열 → C++에서 출력
```

### 예제 코드 및 emsdk 버전

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-11
- emsdk 5.0.3

## C++ Standard Library 이용하기

- Embind를 이용하여 `std::vector`, `std::map`을 JavaScript에서 사용할 수 있도록 내보낼 수 있음. `std::string`은 [Embind가 기본적으로 지원함](/posts/embind/#일반-함수-바인딩-하기)
- 단, 템플릿 클래스 전체를 내보낼 수 없고, 구체화 된 클래스로만 내보낼 수 있음
- Emscripten에서 구체화 된 템플릿 클래스를 내보내기 위한 헬퍼 함수를 지원함
  - `register_vector<T>()`: `std::vector<T>`를 내보내기 위한 함수
  - `register_map<K, V>()`: `std::map<K, T>`를 내보내기 위한 함수
- 명시적으로 내보낸 클래스만을 이용할 수 있기 때문에, JSON에 비해 유연성은 떨어지지만 JSON object와 JSON 문자열의 상호 변환 과정이 없다는 장점이 있다.

### 소스 코드

```C++
// array_std.cpp
#include <vector>
#include <string>
#include <map>
#include <iostream>
#include <emscripten/bind.h>

std::vector<std::string> GetStrVector() {
  std::vector<std::string> arr = {"Hello", "WebAssembly", "and", "JSON"};
  return arr;
}

std::map<std::string, std::string> GetStrMap() {
  std::map<std::string, std::string> map = {
    {"Compiler", "Emscripten"},
    {"Age", "11"},
    {"Language", "C++"}
  };
  return map;
}

void LoadStrVector(const std::vector<std::string>& arr) {
  for (const auto& item : arr) {
    std::cout << item << std::endl;
  }
}

void LoadStrMap(const std::map<std::string, std::string>& map) {
  for (const auto& [key, value] : map) {
    std::cout << key << ": " << value << std::endl;
  }
}

EMSCRIPTEN_BINDINGS(my_module) {
  emscripten::register_vector<std::string>("StrVector");
  emscripten::register_map<std::string, std::string>("StrMap");
  emscripten::function("GetStrVector", &GetStrVector);
  emscripten::function("GetStrMap", &GetStrMap);
  emscripten::function("LoadStrVector", &LoadStrVector);
  emscripten::function("LoadStrMap", &LoadStrMap);
}

// index.html
<!doctype html>
<html>
  <head>
    <title>Emscripten Example-12</title>
  </head>
  <body>
    <script async type="text/javascript" src="array_std.js"></script>
  </body>
</html>
```

- `register_vector<std::string>("StrVector")`
  - `std::vector<std::string>`을 `StrVector`라는 이름으로 JavaScript에 노출
  - https://emscripten.org/docs/api_reference/bind.h.html#vectors
- `register_map<std::string, std::string>("StrMap")`
  - `std::map<std::string, std::string>`을 `StrMap`이라는 이름으로 JavaScript에 노출
  - https://emscripten.org/docs/api_reference/bind.h.html#maps

### 빌드하기

```bash
em++ array_std.cpp -o array_std.js -lembind
```

### 실행 결과

![array with standard library](images/array_with_std.png)
_그림 2. C++과 JavaScript에서 `std::vector<std::string>`과 `std::map<std::string, std::string>`타입 데이터를 상호 주고 받고 있는 모습_

- 내보낸 `std::vector<std::string>`과 `std::map<std::string, std::string>`의 요소에 접근하기 위해 `get` 함수를 이용함. C++에서는 요소에 접근하기 위해 `[]`이나 `at`을 사용했던과는 차이가 있음
- JavaScript Array를 C++로 직접 전달할 수 있는 것은 아니고, 내보낸 `std::vector`를 이용하여 데이터를 전달해야 함
- JS에서는 내보낸 구체화 클래스 이름(`StrVector`, `StrMap`)을 이용하여 직접 생성할 수 있음

```JavaScript
const strVector = new Module.StrVector();
strVector.push_back("Hello");
strVector.push_back("WebAssembly!");
Module.LoadStrVector(strVector);
```

- `std::vector`나 `std::map`의 모든 함수가 JavaScript로 노출되는 것은 아니고, 아래와 같이 일부의 함수만 노출됨

| `C++ std::vector v`  | Embind JS Wrapper v  |
| -------------------- | -------------------- |
| `v.size()`           | `v.size()`           |
| `v[i] / v.at(i)`     | `v.get(i)`           |
| `v[i] = value`       | `v.set(i, value)`    |
| `v.push_back(value)` | `v.push_back(value)` |

| `C++ std::map m`         | Embind JS Wrapper m        |
| ------------------------ | -------------------------- |
| `m.size()`               | `m.size()`                 |
| `m[key] / m.at(key)`     | `m.get(key)`               |
| `m[key] = value`         | `m.set(key, value)`        |
| `m.find(key) != m.end()` | `m.get(key) !== undefined` |
| `key iteration`          | `m.keys()`                 |

### 실행 스크립트

```JavaScript
// std::vector<std::string> exchange
const strVector = Module.GetStrVector();  // C++ → JS로 문자열 배열 전달
for (let i = 0; i < strVector.size(); ++i) {
  console.log(strVector.get(i));          // 배열 요소 출력
}
strVector.push_back("and"); strVector.push_back("JSON"); // "and", "JSON" 요소 추가
Module.LoadStrVector(strVector);          // 문자열 배열을 C++로 전달 → C++에서 출력
strVector.delete();                       // 사용이 끝난 strVector 메모리 해제

// std::map<std::string, std::string> exchange
const strMap = Module.GetStrMap();    // C++ → JS로 map 전달
const keys = strMap.keys();           // map의 key 배열
for (let i = 0; i < keys.size(); ++i) {
    const key = keys.get(i);          // index를 이용하여 key 얻기
    const value = strMap.get(key);    // key를 이용하여 value 얻기
    console.log(`${key}: ${value}`);  // map의 key와 value를 출력
}
keys.delete();                        // 사용이 끝난 keys(std::vector<std::string>) 메모리 해제
strMap.set("Language", "C++");        // map에 새로운 item 추가
Module.LoadStrMap(strMap);            // map을 C++로 전달 → C++에서 출력
strMap.delete();                      // 사용이 끝난 strMap 메모리 해제
```

> [!CAUTION]
> strVector, strMap, keys와 같이 C++로 부터 export된 객체의 경우에 실제 데이터는 Wasm heap 메모리에 저장되어 있다. 따라서, 해당 변수를 더 이상 사용하지 않을때 명시적으로 `delete()` 함수를 통해 소멸자를 호출해야 한다. 하지만, `key`와 `value`와 같은 `std::string` 문자열은 JavaScript 문자열로 자동 변환하기 때문에, 직접 삭제하지 않아도 JS Garbage Collector가 알아서 삭제 처리한다.

### 예제 코드 및 emsdk 버전

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-12
- emsdk 5.0.3

## Memory View 이용하기

### Memory View

- C++ 힙에 있는 메모리 구간을 복사 없이 JavaScript의 TypedArray로 바라보게 해주는 것
- WebAsssembly 메모리 모델의 TypedArray View, C++에서의 포인터, 배열의 개수를 이용하여 생성할 수 있음
- WebAssembly의 메모리는 하나의 거대한 ArrayBuffer이고, 이것을 서로 다른 데이터 타입으로 접근할 수 있도록 다양한 TypedArray View를 제공
- TypedArray View에는 `HEAP8`, `HEAP16`, `HEAP32`, `HEAPU8`, `HEAPU16`, `HEAPU32`, `HEAPF32`, `HEAPF64` 등이 있고, 숫자는 bit의 크기, `U`는 unsigned, `F`는 float를 나타냄 (https://emscripten.org/docs/api_reference/preamble.js.html#type-accessors-for-the-memory-model)

> [!CAUTION]
> Typed Memory View는 Wasm 메모리가 resize 되면 무효화 되어, 기존에 TypedArray가 유효하지 않을 수 있음

### 소스 코드

```C++
#include <cstddef>
#include <vector>
#include <iostream>
#include <emscripten/bind.h>

std::vector<float> g_Vertices = {
  1.0f, 2.0f, 3.0f, 1.0f, 0.0f, 0.0f,  // Vertex 1: position (1, 2, 3), color (1, 0, 0)
  4.0f, 5.0f, 6.0f, 0.0f, 1.0f, 0.0f,  // Vertex 2: position (4, 5, 6), color (0, 1, 0)
  7.0f, 8.0f, 9.0f, 0.0f, 0.0f, 1.0f   // Vertex 3: position (7, 8, 9), color (0, 0, 1)
};

uintptr_t GetVertexData() {
  return reinterpret_cast<uintptr_t>(g_Vertices.data());
}

std::size_t GetVertexDataCount() {
  return g_Vertices.size();
}

void LoadVertexData(uintptr_t ptr, std::size_t size) {
  float* data = reinterpret_cast<float*>(ptr);
  for (std::size_t i = 0; i < size; ++i) {
    std::cout << data[i] << " ";
    if ((i + 1) % 6 == 0) { // Print a newline after every 6 floats (one vertex)
      std::cout << std::endl;
    }
  }
}

EMSCRIPTEN_BINDINGS(my_module) {
  emscripten::function("getVertexData", &GetVertexData);
  emscripten::function("getVertexDataCount", &GetVertexDataCount);
  emscripten::function("loadVertexData", &LoadVertexData);
}
```

- `int`, `float`, `double`과 같은 C++의 기본 데이터 타입의 포인터를 binding 하는 경우에는 `emscripten::allow_raw_pointers()` 옵션을 사용하여 컴파일 오류는 막을 수 있지만 런타임 오류가 발생함 (raw pointer 바인딩의 한계에 대한 자세한 내용은 [Embind FAQ](/posts/embind/#faq) 참고). 포인터 전달하는 것 보다는 정수형 타입으로 캐스팅하여 전달하면 오류 없이 사용할 수 있음

```C++
reinterpret_cast<uintptr_t>(g_Vertices.data())
float* data = reinterpret_cast<float*>(ptr);
```

### 빌드하기

```bash
em++ memory_view.cpp -o memory_view.js -lembind -s EXPORTED_FUNCTIONS="['_malloc','_free']" -s EXPORTED_RUNTIME_METHODS="['HEAPF32']"
```

- JS쪽에서 Wasm 메모리의 동적 할당 및 해제를 위해 `_malloc`, `_free` 함수를 내보냄
- JS쪽에서 사용할 TypedArray View(`HEAPF32`)를 내보냄 (Memory View의 종류와 원리는 [Emscripten Module - FAQ](/posts/emscripten-module/#faq) 참고)

### 실행 결과

![array_memory_view](images/array_with_memoryview.png)
_그림 3. C++과 JavaScript에서 Memory View를 통해 TypedArray 데이터를 상호 주고 받고 있는 모습_

### 실행 스크립트

```JavaScript
// C++ 메모리를 복사 없이 읽기 (Memory view)
const ptr = Module.getVertexData();
const count = Module.getVertexDataCount();

const vertices = new Float32Array(Module.HEAPF32.buffer, ptr, count);  // Wasm 힙의 g_Vertices를 들여다보는 TypedArray를 생성 (복사 없음)
console.log(vertices);  // vertices 정보를 출력

// JS에서 데이터를 생성
const jsVertices = new Float32Array([
  3.0, 2.0, 1.0, 0.0, 0.0, 1.0,
  6.0, 5.0, 4.0, 0.0, 1.0, 0.0,
  9.0, 8.0, 7.0, 1.0, 0.0, 0.0,
]);

// Wasm 메모리에 jsVertices를 복사할 공간을 바이트 단위로 할당
const vPtr = Module._malloc(jsVertices.byteLength);

// Wasm 힙을 들여다보는 TypedArray를 생성
const heapView = new Float32Array(Module.HEAPF32.buffer, vPtr, jsVertices.length);

// JS의 데이터를 Wasm으로 복사
heapView.set(jsVertices);

// C++에서 데이터를 읽음
Module.loadVertexData(vPtr, jsVertices.length);

// 사용이 끝난후 Wasm 메모리 해제
Module._free(vPtr);
```

### 예제 코드 및 emsdk 버전

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-13
- emsdk 5.0.3

## 세 가지 방법의 비교

| 구분                        | JSON 문자열                                             | C++ Standard Library                                                            | Memory View                                             |
| --------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------- |
| **사용 방법**               | 데이터를 JSON 문자열로 직렬화하여 전달                  | `std::vector`, `std::map` 등을 Embind로 바인딩하여 전달                         | Wasm 메모리를 TypedArray View로 직접 접근               |
| **구현 난이도**             | 낮음                                                    | 보통                                                                            | 높음                                                    |
| **데이터 표현의 유연성**    | 높음                                                    | 보통                                                                            | 낮음                                                    |
| **데이터 변환**             | JSON 직렬화/역직렬화 필요                               | Embind를 통한 타입 변환 및 Wrapper 사용                                         | 숫자 배열은 별도 변환 없이 접근 가능                    |
| **메모리 복사**             | 많음                                                    | Embind 처리 과정에서 발생 가능                                                  | C++ → JS는 복사 없이 접근 가능                          |
| **대용량 데이터 효율**      | 낮음                                                    | 보통                                                                            | 높음                                                    |
| **여러 타입의 복합 데이터** | 매우 적합                                               | 타입별 바인딩 필요                                                              | 직접 구조를 정의하여 처리해야 함                        |
| **추가 바인딩**             | `std::string`은 기본 지원                               | 구체화된 STL 타입별 등록 필요                                                   | 포인터와 데이터 크기 등을 JS에 노출해야 함              |
| **메모리/Lifetime 관리**    | 대부분 자동                                             | Embind 객체의 `delete()` 관리 필요                                              | 포인터, 메모리 할당/해제 및 View 유효성 관리 필요       |
| **주요 장점**               | 코드가 직관적이고 다양한 구조의 데이터를 쉽게 전달 가능 | C++ STL을 활용하면서 JSON 변환 과정 없이 데이터 전달 가능                       | 가장 높은 성능과 메모리 효율, 대용량 연속 데이터에 적합 |
| **주요 단점**               | 직렬화/역직렬화 비용과 문자열 크기 증가                 | 타입별 바인딩이 필요하고 C++ STL과 다른 Embind Wrapper 인터페이스를 사용해야 함 | 코드가 복잡하고 Wasm 메모리 구조에 대한 이해 필요       |
| **적합한 데이터**           | 설정값, 메타데이터, 작은 배열 및 객체                   | 중간 규모의 배열, Map 및 C++ 객체                                               | Mesh, 좌표, 해석 결과 등 대용량 수치 배열               |

> [!NOTE]
> 작고 구조가 복잡한 데이터는 JSON, C++ STL 컨테이너를 활용하는 데이터는 Standard Library Binding, Mesh나 해석 결과와 같은 대용량 연속 수치 데이터는 Memory View를 사용하는 것이 적합하다.

## FAQ

- Standard Library의 모든 데이터 구조를 binding 할 수 있나요?
  - 아니요. 현재(emsdk 6.0.8 기준) 제공되는 데이터 구조는 std::vector(구체 타입 등록 필요), std::map(구체 타입 등록 필요)이고 등록을 위한 전용 helper 함수가 존재합니다.
  - `std::array`와 같은 고정 크기의 데이터는 `emscripten::value_array`를 통해 JavaScript 객체로 binding 할 수 있고, 데이터만 담겨있는 구조체의 경우에는 `emscripten::value_object`를 통해 JavaScript 객체로 binding 할 수 있습니다. 이렇게 바인딩된 JavaScript 객체의 데이터는 JavaScript 메모리에 생성되기 때문에 `delete` 함수를 통해 메모리 관리를 하지 않아도 됩니다.

- `strMap`에 특정 key가 있는지 확인하고 싶은데, `m.has(key)` 같은 함수는 없나요?
  - 없습니다. `register_map`이 실제로 노출하는 함수는 `size`, `get`, `set`, `keys` 뿐이고, C++의 `std::map::find`에 대응하는 존재 여부 확인 함수는 별도로 제공되지 않습니다.
  - `get(key)`는 내부적으로 `std::optional`을 이용해 구현되어 있어서, 찾는 key가 없으면 `undefined`를 리턴합니다. 따라서 `strMap.get(key) !== undefined`로 존재 여부를 확인하면 됩니다.

- 등록한 `std::vector`를 JavaScript의 `for...of` 문으로 순회할 수 있나요?
  - 네, 가능합니다. `register_vector`는 `size`와 `get` 함수를 기반으로 하는 JS iterable 프로토콜을 함께 등록하기 때문에, 인덱스를 직접 다루지 않고도 아래처럼 순회할 수 있습니다.

    ```JavaScript
    for (const item of strVector) {
      console.log(item);
    }
    ```

  - 단, `std::map`으로 등록한 객체(`register_map`)에는 이런 iterable이 등록되어 있지 않으므로, 위 예제 스크립트처럼 `keys()`로 key 목록을 받아 순회해야 합니다.

- JavaScript의 배열(Array)을 `Module.StrVector`로 변환하지 않고 C++ 함수에 바로 전달할 수는 없나요?
  - 안 됩니다. `LoadStrVector`처럼 `std::vector<std::string>`을 인자로 받는 함수는 Embind가 등록한 `StrVector` 타입만 받을 수 있고, 일반 JavaScript `Array`는 대응되는 타입이 아니라서 그대로 전달하면 `BindingError`가 발생합니다.
  - JavaScript `Array`를 넘기려면 먼저 `StrVector` 인스턴스로 변환해야 합니다.

    ```JavaScript
    const jsArray = ["Hello", "WebAssembly"];
    const strVector = new Module.StrVector();
    jsArray.forEach((item) => strVector.push_back(item));
    Module.LoadStrVector(strVector);
    strVector.delete();
    ```

- JSON, Standard Library, Memory View 말고 `emscripten::val`을 쓰는 방법은 없나요?
  - 있습니다. `emscripten::val`은 임의의 JavaScript 값(배열, 객체, 함수 등)을 C++에서 동적으로 다룰 수 있게 해주는 타입으로, `std::vector`/`std::map`처럼 타입별로 미리 등록하지 않아도 되고 JSON 직렬화 비용도 없습니다.
  - 다만 컴파일 타임에 타입이 고정되지 않아 오류를 미리 잡기 어렵고, 접근할 때마다 JS ↔ C++ 경계를 오가는 오버헤드가 있어서, 위 세 가지 방법 중 하나가 적합하지 않은 예외적인 경우(예: 콜백 함수를 그대로 저장해야 하는 경우)에 주로 사용됩니다. 자세한 내용은 [Emscripten 공식 문서의 val 가이드](https://emscripten.org/docs/api_reference/val.h.html)를 참고하세요.

## 참고 자료

- https://emscripten.org/docs/api_reference/bind.h.html
