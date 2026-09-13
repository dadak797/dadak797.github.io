---
title: Emscripten에서 HTTP 요청하기 - Emscripten Fetch API
date: 2026-09-13
draft: false
description: Emscripten에서 HTTP 통신을 수행하는 방법은 크게 세 가지 방법으로 구분할 수 있다. 여기에서는 `emscripten_async_wget*` 계열 함수를 이용하는 방법과 `emscripten_fetch`를 이용하는 방법에 대해 소개한다.
categories:
  - Web
tags:
  - emscripten
  - http
  - fetch
ShowToc: true
TocOpen: false
ShowReadingTime: false
---

> [!SUMMARY]
> Emscripten에서 HTTP 통신을 수행하는 방법은 크게 세 가지 방법으로 구분할 수 있다. 여기에서는 `emscripten_async_wget*` 계열 함수를 이용하는 방법과 `emscripten_fetch`를 이용하는 방법에 대해 소개한다.

## Emscripten에서 HTTP 통신하기

Emscripten으로 C/C++ 코드를 WebAssembly로 빌드하면 프로그램은 브라우저 환경에서 실행된다. 네이티브 C/C++ 프로그램에서는 운영체제의 소켓 API 또는 이를 사용하는 HTTP 라이브러리를 통해 HTTP 통신을 수행할 수 있지만, 브라우저에서 실행되는 WebAssembly는 이러한 소켓 API에 직접 접근할 수 없다. 따라서 Emscripten에서는 브라우저가 제공하는 네트워크 기능을 이용하여 HTTP 통신을 수행한다.

브라우저에서는 일반적으로 JavaScript의 `fetch`를 이용하여 HTTP 요청을 수행한다. Emscripten은 C/C++ 코드에서도 이러한 기능을 사용할 수 있도록 여러 API를 제공한다. 크게 보면 다음과 같은 세 가지 방법으로 구분할 수 있다.

- `emscripten_async_wget*` 계열 함수를 이용하는 방법
- `emscripten_fetch`를 이용하는 방법
- `EM_ASM` 또는 `EM_JS`에서 JavaScript의 `fetch`를 직접 호출하는 방법

`emscripten_async_wget*` 계열 함수는 파일을 간단히 다운로드하는 데 적합한 편의 API이다. `emscripten_fetch`는 GET, POST, HTTP 헤더, 요청 데이터 등을 보다 세밀하게 설정할 수 있어 C/C++ 코드 중심으로 HTTP 통신을 구현할 때 유용하다.

반면 `EM_ASM`이나 `EM_JS`를 이용하면 JavaScript의 `fetch`를 직접 호출할 수 있다. 이 방법은 `Response`, `ArrayBuffer`, `Blob`, `ReadableStream` 등 브라우저의 Fetch API 기능을 그대로 사용할 수 있다는 장점이 있다.

`EM_ASM`이나 `EM_JS` 매크로 내부의 코드는 JavaScript 코드이기 때문에, 이 글에서는 별도로 소개하지 않는다. `EM_ASM`이나 `EM_JS`에 C++의 인자를 전달하는 방법은 [C++에서 JavaScript의 함수를 호출하기](/posts/call-js-from-cpp/)와 [JavaScript에서 C++ 클래스 사용하기](/posts/embind/)에 소개하였고, `EM_ASM`이나 `EM_JS`에 C++에서 작성한 파일을 전달하는 방법은 [Emscripten Virtual File System](/posts/emscripten-file-handling-memfs/#emscripten-virtual-file-system-vfs)을 참고하면 된다.

이 글에서는 가장 간단한 `emscripten_async_wget*` 계열부터 살펴본 뒤, `emscripten_fetch`를 이용하는 방법을 차례대로 알아본다.

> [!NOTE]
> 아래의 Fetch API 요청 테스트에서는 [테스트용 웹 서버 구축하기](/posts/building-test-web-server/)에서 구성한 웹 서버를 사용한다. 모든 예제는 이 테스트 서버가 동작 중이라고 가정한다.

## `emscripten_async_wget*` - 간단한 파일 다운로드

### 소스 코드

```C++
// em_wget.cpp
#include <emscripten.h>
#include <emscripten/bind.h>
#include <iostream>
#include <string>
#include <fstream>
#include <cstring>

// emscripten_async_wget이 성공했을 때의 callback 함수
void onLoadWget(const char* filename) {
  std::cout << "File downloaded: " << filename << std::endl;
  std::ifstream ifs(filename);
  std::cout << "File content:" << std::endl;
  std::string line;
  while (std::getline(ifs, line)) {
    std::cout << line << std::endl;
  }
  ifs.close();
}

// emscripten_async_wget이 실패했을 때의 callback 함수
void onErrorWget(const char* filename) {
  std::cerr << "Error downloading file: " << filename << std::endl;
}

// emscripten_async_wget을 이용하여 파일 다운로드
void GetFile(const std::string& url, const std::string& filename) {
  emscripten_async_wget(url.c_str(), filename.c_str(), onLoadWget, onErrorWget);
}

// emscripten_async_wget_data가 성공했을 때의 callback 함수
void onLoadWgetData(void* userdata, void* data, int size) {
  char* filename = static_cast<char*>(userdata);
  std::cout << "Data downloaded to memory for file: " << filename << std::endl;
  char* buffer = static_cast<char*>(data);
  std::string content(buffer, size);
  std::cout << "Data content:" << std::endl;
  std::cout << content << std::endl;
  delete[] filename;  // userdata 메모리 정리
}

// emscripten_async_wget_data가 실패했을 때의 callback 함수
void onErrorWgetData(void* userdata) {
  char* filename = static_cast<char*>(userdata);
  std::cerr << "Error downloading data: " << filename << std::endl;
  delete[] filename;  // userdata 메모리 정리
}

// emscripten_async_wget_data를 이용하여 파일을 메모리 버퍼로 받음
void GetFileData(const std::string& url, const std::string& filename) {
  char* userData = new char[filename.size() + 1];  // 힙 메모리 동적할당
  std::strcpy(userData, filename.c_str());  // userData에 filename 복사
  emscripten_async_wget_data(url.c_str(), static_cast<void*>(userData), onLoadWgetData, onErrorWgetData);
}

EMSCRIPTEN_BINDINGS(my_module) {
  emscripten::function("getFile", &GetFile);
  emscripten::function("getFileData", &GetFileData);
}
```

```
<!-- index.html -->
<!doctype html>
<html>
  <head>
    <title>Emscripten Example-16</title>
  </head>
  <body>
    <script async type="text/javascript" src="em_wget.js"></script>
  </body>
</html>
```

- `GetFile(const std::string& url, const std::string& filename)`
  - `url`과 `filename`을 받아 `emscripten_async_wget`를 호출하는 함수
  - JavaScript에서 사용할 수 있도록 바인딩
- `void emscripten_async_wget(const char* url, const char* filename, em_str_callback_func onload, em_str_callback_func onerror)`
  - `url`에 있는 파일을 GET으로 다운로드
  - MEMFS에 `filename`이라는 이름으로 저장
  - 비동기 함수로 다운로드에 성공하면 `onload`, 실패하면 `onerror`를 호출함
  - `em_str_callback_func`는 `void`를 리턴하고 `const char* filename`를 인자로 받는 함수 포인터
- `onLoadWget`
  - `emscripten_async_wget`이 성공했을 때의 콜백 함수
  - `const char* filename`을 인자로 넘겨 받음
  - 함수에 진입했을 때는 이미 MEMFS에 파일이 생성된 상태이기 때문에 `std::ifstream`으로 파일을 열어볼 수 있음
- `onErrorWget`
  - `emscripten_async_wget`이 실패했을 때의 콜백 함수
- `GetFileData(const std::string& url, const std::string& filename)`
  - `url`과 `filename`을 받아 `emscripten_async_wget_data`를 호출하는 함수
  - `emscripten_async_wget_data`은 `void*` 타입의 포인터를 통해 콜백함수에 `userData`를 전달
  - 콜백함수가 호출될 때까지 `userData`가 유효해야 하기 때문에 힙에 메모리를 할당하여 `filename`을 복사한 뒤에 `void*`로 캐스팅하여 전달
  - 동적할당한 메모리는 콜백함수에서 정리해야 함
- `void emscripten_async_wget_data(const char* url, void *userdata, em_async_wget_onload_func onload, em_arg_callback_func onerror)`
  - `url`에 있는 파일을 GET으로 다운로드
  - `userdata`로 콜백함수에서 사용할 데이터를 전달
  - 비동기 함수로 다운로드에 성공하면 `onload`, 실패하면 `onerror`를 호출함
  - `em_async_wget_onload_func`는 `void`를 리턴하고 `void* userdata`, `void* data`, `int size`을 인자로 받는 함수 포인터
  - `em_arg_callback_func`는 `void* userdata`를 인자로 받는 함수 포인터
  - MEMFS 쓰고 읽는 과정이 없기 때문에 `emscripten_async_wget`에 비해서는 오버헤드가 적음
- `onLoadWgetData`
  - `emscripten_async_wget_data`가 성공했을 때의 콜백 함수
  - `void* userdata`, `void* data`, `int size`를 인자로 받음. `data`는 데이터 버퍼의 주소를 나타내고 `size`는 데이터 버퍼의 크기를 나타냄. 이 두 값을 이용하여 문자열을 생성할 수 있음
  - `data` 버퍼는 콜백이 실행되는 동안에만 유효하므로, 콜백 안에서 사용하거나 필요한 데이터를 다른 메모리로 복사해야 함. 예제에서는 `std::string content`에 복사함
  - 힙에 생성했던 `userdata`의 메모리를 정리해야 함
- `onErrorWgetData`
  - `emscripten_async_wget_data`가 실패했을 때의 콜백 함수
  - 실패한 경우에도 `userdata`의 메모리는 정리해야 함

### 빌드하기

```bash
em++ em_wget.cpp -o em_wget.js -lembind
```

### 실행 스크립트

```JavaScript
Module.getFile("https://raw.githubusercontent.com/dadak797/blog-examples/master/examples/ex-14/models/Cube.obj", "Cube.obj");
Module.getFileData("https://raw.githubusercontent.com/dadak797/blog-examples/master/examples/ex-14/models/Cube.obj", "Cube.obj");
```

- [Emscripten에서 파일 다루기](/posts/emscripten-file-handling-memfs/)의 OBJ 파일을 다운로드 받음
- 두 경우 모두 파일의 내용을 콘솔창에 잘 출력하는 것을 확인할 수 있지만, 첫 번째 예제는 MEMFS를 거치기 때문에 추가로 한 번의 복사 과정이 더 필요하다는 차이가 있음

![emscripten-async-wget-comparison](images/emscripten-async-wget-comparison.png)
_그림 1. emscripten_async_wget과 emscripten_async_wget_data의 데이터 흐름 비교_

### 예제 코드 및 emsdk 버전

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-16
- emsdk 5.0.3

## Emscripten Fetch API - C++에서 HTTP 요청하기

Emscripten Fetch API를 사용하면 네이티브 코드에서 XMLHttpRequest(XHR)을 통해 원격 서버와 파일을 전송할 수 있다. 또한 HTTP GET, POST, PUT 등을 지원하므로 일반적인 HTTP 통신에도 사용할 수 있지만, 원격 파일의 다운로드 및 업로드와 같이 C/C++ 프로그램에서 데이터를 전송하는 용도를 주요 사용 사례로 제공한다.

![emscripten fetch comparison](images/emscripten-fetch-comparison.png)
_그림 2. `Emscripten Fetch API`와 `EM_ASM/EM_JS + Javascript fetch`를 활용한 방법의 비교_

### Emscripten Fetch API 함수와 인자

**`emscripten_fetch_t* emscripten_fetch(emscripten_fetch_attr_t* fetch_attr, const char* url)`**

- emscripten을 이용하여 fetch를 할 때 사용하는 함수
- `emscripten_fetch_attr_t`과 url을 인자로 넘김

**`emscripten_fetch_attr_t`**

- Fetch 요청의 동작 방식을 설정하는 구조체. 주요 인자는 아래와 같음
- `char requestMethod[32]`: GET, POST와 같은 요청 방식을 문자열로 설정
- `void *userData`: 콜백 함수에서 사용하고 싶은 데이터를 전달하기 위한 포인터
- `void (*onsuccess)(struct emscripten_fetch_t *fetch)`: 요청에 성공했을 때 호출되는 콜백함수
- `void (*onerror)(struct emscripten_fetch_t *fetch)`: 요청에 실패했을 때 호출되는 콜백함수
- `void (*onprogress)(struct emscripten_fetch_t *fetch)`: 요청이 진행 중일 때 호출되는 콜백함수. 큰 파일을 다운 받는 경우에 진행 상황을 파악하기 위해 사용
- `uint32_t attributes`: EMSCRIPTEN_FETCH_LOAD_TO_MEMORY, EMSCRIPTEN_FETCH_REPLACE 등 fetch 데이터를 어디에 둘 것인지, 동기/비동기로 할 것인지 등을 결정
- `const char * const *requestHeaders`: HTTP 헤더를 추가. 마지막은 nullptr로 끝내야 함

```C++
const char* headers[] = {
  "Header-1", "Value-1",
  "Header-2", "Value-2",
  nullptr
};
```

- `const char *requestData`, `size_t requestDataSize`: POST 요청으로 서버에 데이터를 전송할 때 사용. Fetch가 진행되는 동안 데이터가 유효해야 하기 때문에 스택 메모리에 저장한 데이터를 넘기는 것은 위험하고, 힙 메모리를 통해 데이터를 넘긴 후 fetch 요청이 끝났을 때 메모리 해제를 해야 함
- `uint32_t timeoutMSecs`: 밀리초 단위로 지정한 이 값 이내에 요청이 완료되지 않으면 timeout 처리하도록 지정
- `bool withCredentials`: 로그인된 웹 서비스에서 쿠키 기반 인증을 사용하는 API에 cross-origin 요청을 할 때 설정
  `emscripten_fetch_t`
- `emscripten_fetch_attr_t`의 콜백 함수에 전달되는 인자
- `const char *data`: 다운로드된 데이터에 접근하는 포인터. `emscripten_fetch_attr_t.attributes`를 `EMSCRIPTEN_FETCH_LOAD_TO_MEMORY`로 지정하지 않으면 `nullptr`이 됨
- `uint64_t numBytes`: `data`의 크기. `data`와 한 쌍으로 쓰이는 경우가 많음
- `uint64_t totalBytes`: 응답 body의 전체 크기. `onprogress` 콜백 함수에서 진행률 계산할 때 유용함
- `unsigned short status`: HTTP 응답 코드
- `char statusText[64]`: `status`의 문자열
- `const char *url`: 요청한 URL
- `void *userData`: 요청 시에 `emscripten_fetch_attr_t`를 통해 전달 받은 데이터. 전달 시에 `void*`로 형 변환하여 전달하기 때문에, 콜백 함수 내에서는 원래의 데이터 타입으로 변환하여 사용

```C++
void onSuccess(emscripten_fetch_t* fetch)
{
    MyObject* obj =
        static_cast<MyObject*>(fetch->userData);
    // ...
}
```

### 소스 코드

```C++
// em_fetch.cpp
#include <emscripten/bind.h>
#include <emscripten/fetch.h>
#include <emscripten.h>

#include <cstring>
#include <fstream>
#include <iostream>
#include <string>

// 1. GET request example
void get_request_example() {
  // Implementation code goes here...
}

// 2. GET request with authorization header example
// Use withAuth to test requests both with and without an Authorization header
void get_request_with_auth_example(bool withAuth) {
  // Implementation code goes here...
}

// 3. POST request with body example
void post_request_with_body_example(const std::string& body) {
  // Implementation code goes here...
}

// 4. File upload example
void post_request_upload_file_example(
    const std::string& fileName,
    const std::string& fileContent) {
  // Implementation code goes here...
}

// 5. File download example
// Call the download API using the filename returned by the upload response
void get_request_download_file_example(const std::string& fileName) {
  // Implementation code goes here...
}

EMSCRIPTEN_BINDINGS(my_module) {
  emscripten::function("get_request_example", &get_request_example);
  emscripten::function("get_request_with_auth_example", &get_request_with_auth_example);
  emscripten::function("post_request_with_body_example", &post_request_with_body_example);
  emscripten::function("post_request_upload_file_example",
    &post_request_upload_file_example);
  emscripten::function("get_request_download_file_example",
    &get_request_download_file_example);
}
```

- 5가지 요청에 대한 테스트를 위한 함수를 Embind로 내보냄
- 각 함수의 구현은 아래 쪽에 있음

```HTML
<!-- index.html -->
<!doctype html>
<html>
  <head>
    <title>Emscripten Example-17</title>
  </head>
  <body>
    <script async type="text/javascript" src="em_fetch.js"></script>
  </body>
</html>
```

### 빌드하기

```bash
em++ em_fetch.cpp -o em_fetch.js -lembind -s FETCH=1 -s EXPORTED_RUNTIME_METHODS="['FS']"
```

- `emscripten_fetch`를 사용하기 위해 `-s FETCH=1`를 컴파일 옵션으로 추가해야 함
- JavaScript에서 파일 시스템을 사용하기 위해 `EXPORTED_RUNTIME_METHODS`에 `FS`를 추가함

### 예제 코드 및 emsdk 버전

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-17
- emsdk 5.0.3

## 요청 테스트 하기

> [!NOTE]
> [테스트용 웹 서버 구축하기](/posts/building-test-web-server/)와 동일한 테스트 과정을 `emscripten_fetch`를 이용하여 진행한다. 해당 예제에서 구축한 서버를 실행시킨 후(`npm run dev`), Embind로 내보낸 함수들을 이용하여 각 요청들을 테스트 한다. 현재 예제는 `python -m http.server 8080`으로 구동 시킨다.

### GET 요청 테스트

```C++
// em_fetch.cpp
void get_request_example() {
  emscripten_fetch_attr_t attr;
  emscripten_fetch_attr_init(&attr);  // attr을 초기화 하기

  strcpy(attr.requestMethod, "GET");  // 요청 방법을 GET으로 설정
  attr.attributes =
      EMSCRIPTEN_FETCH_LOAD_TO_MEMORY |  // 응답 데이터를 Wasm 메모리에 로드
      EMSCRIPTEN_FETCH_REPLACE;          // IndexedDB에 저장된 기존 데이터를 사용하지 않음

  // 성공한 요청에 대한 콜백 함수 설정
  attr.onsuccess = [](emscripten_fetch_t* fetch) {
    // fetch에 전달된 data와 numBytes를 이용하여 문자열 생성
    std::string response(fetch->data, fetch->numBytes);

    std::cout << "Status: " << fetch->status << std::endl;  // Fetch 상태 확인
    std::cout << "Response: " << response << std::endl;     // 전달 받은 문자열 출력

    emscripten_fetch_close(fetch);  // Fetch 요청에 사용된 리소스를 정리하고 해제
  };

  // 실패한 요청에 대한 콜백 함수 설정
  attr.onerror = [](emscripten_fetch_t* fetch) {
    std::string response =
        fetch->data
            ? std::string(fetch->data, fetch->numBytes)
            : std::string{};

    std::cerr << "Status: " << fetch->status << std::endl;
    std::cerr << "Response: " << response << std::endl;

    emscripten_fetch_close(fetch);
  };

  // 생성한 attr 구조체와 url을 이용하여 fetch 실행
  emscripten_fetch_t* fetch
    = emscripten_fetch(&attr, "http://localhost:3000/hello");
  if (!fetch) {  // Fetch 작업 자체를 정상적으로 생성 및 시작 했는지 확인
    std::cerr << "Failed to start request" << std::endl;
  }
}
```

- `attr`에 `requestMethod`, `attributes`, `onsuccess`, `onerror`를 설정하여 `emscripten_fetch` 함수에 전달
- `emscripten_fetch`는 기본적으로 비동기로 동작하기 때문에 콜백 함수(`onsuccess`, `onerror`)를 전달하여 요청 이후의 동작을 전달해야 함

```JavaScript
Module.get_request_example();
```

- [테스트용 웹 서버 구축하기 - GET 요청 테스트](/posts/building-test-web-server/#get-요청-테스트)와 동일한 요청

![GET test](images/GET_test.png)
_그림 3. GET 요청 결과 - 서버로부터 정상적으로 응답을 받음_

### 인증 헤더가 포함된 GET 요청 테스트

```C++
// em_fetch.cpp
void get_request_with_auth_example(bool withAuth) {
  emscripten_fetch_attr_t attr;
  emscripten_fetch_attr_init(&attr);

  strcpy(attr.requestMethod, "GET");  // 요청 방법을 GET으로 설정
  attr.attributes =
      EMSCRIPTEN_FETCH_LOAD_TO_MEMORY |
      EMSCRIPTEN_FETCH_REPLACE;

  // Authorization 헤더를 설정
  if (withAuth) {
    const char* headers[] = {
      "Authorization", "Bearer test-token",
      nullptr
    };
    attr.requestHeaders = headers;
  }

  // 성공한 요청에 대한 콜백 함수 설정
  attr.onsuccess = [](emscripten_fetch_t* fetch) {
    std::string response(fetch->data, fetch->numBytes);

    std::cout << "Status: " << fetch->status << std::endl;
    std::cout << "Response: " << response << std::endl;

    emscripten_fetch_close(fetch);
  };

  // 실패한 요청에 대한 콜백 함수 설정
  attr.onerror = [](emscripten_fetch_t* fetch) {
    std::string response =
        fetch->data
            ? std::string(fetch->data, fetch->numBytes)
            : std::string{};

    std::cerr << "Status: " << fetch->status << std::endl;
    std::cerr << "Response: " << response << std::endl;

    emscripten_fetch_close(fetch);
  };

  emscripten_fetch_t* fetch
    = emscripten_fetch(&attr, "http://localhost:3000/auth");
  if (!fetch) {
    std::cerr << "Failed to start request" << std::endl;
  }
}
```

- `withAuth`가 `true`인 경우에는 `attr.requestHeaders`를 추가하여 요청을 보냄

```JavaScript
Module.get_request_with_auth_example(true);
Module.get_request_with_auth_example(false);
```

- [테스트용 웹 서버 구축하기 - 인증 헤더가 포함된 GET 요청 테스트](/posts/building-test-web-server/#인증-헤더가-포함된-get-요청-테스트)와 동일한 요청

![GET with auth](images/GET_with_auth.png)
_그림 4. 인증 헤더를 포함한 GET 요청 결과 - 인증 헤더가 포함되지 않은 경우에는 오류가 발생함_

### POST 요청 테스트

```C++
// em_fetch.cpp
void post_request_with_body_example(const std::string& body) {
  emscripten_fetch_attr_t attr;
  emscripten_fetch_attr_init(&attr);

  strcpy(attr.requestMethod, "POST");  // 요청 방법을 POST로 설정
  attr.attributes =
      EMSCRIPTEN_FETCH_LOAD_TO_MEMORY |
      EMSCRIPTEN_FETCH_REPLACE;

  // 헤더에서 Content-Type을 설정
  const char* headers[] = {
    "Content-Type", "application/json",
    nullptr
  };
  attr.requestHeaders = headers;

  // 요청 body 설정
  // 요청이 완료될 때까지 requestBody가 유효해야 하기 때문에 Wasm 힙 메모리에 requestBody를 생성
  auto* requestBody = new std::string(body);
  attr.requestData = requestBody->data();
  attr.requestDataSize = requestBody->size();

  // 콜백 함수에서 requestBody의 메모리 해제를 수행하기 위해 requestBody를 userData에 전달
  // 콜백 함수에서는 이 userData를 std::string*으로 변환하여 사용
  attr.userData = requestBody;

  // 성공 콜백 함수
  attr.onsuccess = [](emscripten_fetch_t* fetch) {
    std::string response(fetch->data, fetch->numBytes);

    std::cout << "Status: " << fetch->status << std::endl;
    std::cout << "Response: " << response << std::endl;

    // requestBody의 사용이 완료되었기 때문에 메모리 해제를 수행
    // attr.userData를 통해 전달한 포인터가 fetch->userData로 전달됨
    // userData의 데이터 타입은 void* 이기 때문에 원래 타입(std::string*)으로 변환하여 해제
    delete static_cast<std::string*>(fetch->userData);
    emscripten_fetch_close(fetch);
  };

  // 실패 콜백 함수
  attr.onerror = [](emscripten_fetch_t* fetch) {
    std::string response =
        fetch->data
            ? std::string(fetch->data, fetch->numBytes)
            : std::string{};

    std::cerr << "Status: " << fetch->status << std::endl;
    std::cerr << "Response: " << response << std::endl;

    // requestBody의 사용이 완료되었기 때문에 메모리 해제를 수행
    delete static_cast<std::string*>(fetch->userData);
    emscripten_fetch_close(fetch);
  };

  emscripten_fetch_t* fetch
    = emscripten_fetch(&attr, "http://localhost:3000/echo");
  if (!fetch) {
    // 요청 자체가 제대로 생성되지 않은 경우에는 콜백함수가 호출되지 않기 때문에, 여기서 메모리 해제를 수행
    delete requestBody;
    std::cerr << "Failed to start request" << std::endl;
  }
}
```

- `attr.requestData`, `attr.requestDataSize`를 추가하여 POST 요청의 `requestBody`를 전달
- `requestBody`는 요청이 완료될 때까지 유효해야 하기 때문에 데이터를 힙 메모리에 생성 후 포인터를 통해 전달해야 함
- 힙 메모리에 할당된 `requestBody` 데이터는 요청 완료 후에 삭제해야 하기 때문에, `userData`에 `void*`로 형 변환하여 전달한 후 콜백 함수에서 원래의 데이터 타입(`std::string*`)으로 형 변환하여 메모리 해제를 수행해야 함

```JavaScript
Module.post_request_with_body_example(JSON.stringify({
  'name': 'Emscripten',
  'language': 'C++',
}));
```

- [테스트용 웹 서버 구축하기 - POST 요청 테스트](/posts/building-test-web-server/#post-요청-테스트)와 동일한 요청

![POST test](images/POST_test.png)
_그림 5. POST 요청 결과 - 전달한 JSON body를 그대로 돌려 받음_

### 파일 업로드 테스트

```C++
// em_fetch.cpp
void post_request_upload_file_example(
    const std::string& fileName,
    const std::string& fileContent) {
  // multipart/form-data의 각 part를 구분하기 위한 boundary 정의
  const std::string boundary =
      "----EmscriptenBoundary7MA4YWxkTrZu0gW";

  auto* requestBody = new std::string();

  // multipart part 시작
  *requestBody += "--" + boundary + "\r\n";

  // form field 이름(name)과 업로드 파일 이름(filename) 지정
  *requestBody +=
      "Content-Disposition: form-data; "
      "name=\"uploadFile\"; "
      "filename=\"" + fileName + "\"\r\n";

  // 업로드 파일의 MIME type 지정
  *requestBody += "Content-Type: text/plain\r\n";

  // part header와 실제 파일 데이터를 구분하는 빈 줄
  *requestBody += "\r\n";

  // 실제 파일 데이터 추가
  requestBody->append(fileContent.data(), fileContent.size());
  *requestBody += "\r\n";

  // multipart body 종료
  *requestBody += "--" + boundary + "--\r\n";

  emscripten_fetch_attr_t attr;
  emscripten_fetch_attr_init(&attr);

  strcpy(attr.requestMethod, "POST");
  attr.attributes =
      EMSCRIPTEN_FETCH_LOAD_TO_MEMORY |
      EMSCRIPTEN_FETCH_REPLACE;

  // Content-Type 정의
  const std::string contentType =
      "multipart/form-data; boundary=" + boundary;

  // 헤더 설정
  const char* headers[] = {
      "Content-Type", contentType.c_str(),
      nullptr
  };

  attr.requestHeaders = headers;
  attr.requestData = requestBody->data();
  attr.requestDataSize = requestBody->size();
  attr.userData = requestBody;

  // 성공 콜백 함수 설정
  attr.onsuccess = [](emscripten_fetch_t* fetch) {
    std::string response(fetch->data, fetch->numBytes);

    std::cout << "Status: " << fetch->status << std::endl;
    std::cout << "Response: " << response << std::endl;

    delete static_cast<std::string*>(fetch->userData);
    emscripten_fetch_close(fetch);
  };

  // 실패 콜백 함수 설정
  attr.onerror = [](emscripten_fetch_t* fetch) {
    std::string response =
        fetch->data
            ? std::string(fetch->data, fetch->numBytes)
            : std::string{};

    std::cerr << "Status: " << fetch->status << std::endl;
    std::cerr << "Response: " << response << std::endl;

    delete static_cast<std::string*>(fetch->userData);
    emscripten_fetch_close(fetch);
  };

  emscripten_fetch_t* fetch
    = emscripten_fetch(&attr, "http://localhost:3000/upload");
  if (!fetch) {
    delete requestBody;
    std::cerr << "Failed to start upload" << std::endl;
  }
}
```

- Emscripten Fetch에서는 `multipart/form-data` 형식의 request body를 직접 구성해야 함. 반면, JavaScript Fetch에서 `FormData` 객체를 사용하면 브라우저가 boundary와 각 part의 header를 자동으로 구성하므로 별도의 작업이 필요하지 않음

```JavaScript
Module.post_request_upload_file_example(
  "test_file.txt",
  "Hello from JavaScript",
);
```

- [테스트용 웹 서버 구축하기 - 파일 업로드 테스트](/posts/building-test-web-server/#파일-업로드-테스트)와 동일한 요청

![POST file upload](images/POST_file_upload.png)
_그림 6. POST 파일 업로드 요청 결과 - `uploads` 폴더에 `test_file-{timestamp}.txt` 파일이 생성됨_

### 파일 다운로드 테스트

```C++
// em_fetch.cpp
void get_request_download_file_example(const std::string& fileName) {
  emscripten_fetch_attr_t attr;
  emscripten_fetch_attr_init(&attr);

  strcpy(attr.requestMethod, "GET");
  attr.attributes =
      EMSCRIPTEN_FETCH_LOAD_TO_MEMORY |
      EMSCRIPTEN_FETCH_REPLACE;

  std::string* fileNamePtr = new std::string(fileName);
  attr.userData = static_cast<void*>(fileNamePtr);

  attr.onsuccess = [](emscripten_fetch_t* fetch) {
    std::string response(fetch->data, fetch->numBytes);

    std::cout << "Status: " << fetch->status << std::endl;
    std::cout << "Response: " << response << std::endl;

    // 파일 다운 받기 시작
    // fetch->data, fetch->numBytes를 이용하여 MEMFS에 파일을 생성
    std::string* pFileName = static_cast<std::string*>(fetch->userData);
    std::ofstream outFile(*pFileName, std::ios::binary);
    outFile.write(fetch->data, fetch->numBytes);
    outFile.close();

    EM_ASM({
      const fileName = UTF8ToString($0);

      // MEMFS의 파일에 접근하여 Uint8Array의 TypedArray로 파일의 content를 받아옴
      const content = FS.readFile(fileName);
      const blob = new Blob([content], {
        type: 'application/octet-stream'
      });

      // 가상의 링크 태그를 만들어 파일을 다운로드
      const url = URL.createObjectURL(blob);
      const link = document.createElement('a');
      link.href = url;
      link.download = fileName;
      link.click();
      setTimeout(() => {
        URL.revokeObjectURL(url);
      }, 0);

      // MEMFS에 생성했던 임시 파일을 삭제
      FS.unlink(fileName);

      // 파일 다운 받기 완료
    }, (*pFileName).c_str());

    delete pFileName;
    emscripten_fetch_close(fetch);
  };

  attr.onerror = [](emscripten_fetch_t* fetch) {
    std::string response =
        fetch->data
            ? std::string(fetch->data, fetch->numBytes)
            : std::string{};

    std::cerr << "Status: " << fetch->status << std::endl;
    std::cerr << "Response: " << response << std::endl;

    delete static_cast<std::string*>(fetch->userData);
    emscripten_fetch_close(fetch);
  };

  emscripten_fetch_t* fetch =
    emscripten_fetch(&attr, ("http://localhost:3000/download/" + fileName).c_str());
  if (!fetch) {
    delete fileNamePtr;
    std::cerr << "Failed to start download" << std::endl;
  }
}
```

- 파일 데이터를 읽어 C++에서 처리하는 경우에는 `// 파일 다운 받기 시작`, `// 파일 다운 받기 완료` 부분의 코드가 필요하지 않지만, [테스트용 웹 서버 구축하기 - 파일 다운로드 테스트](/posts/building-test-web-server/#파일-다운로드-테스트)와 동일한 처리를 위해 `EM_ASM`을 이용하여 파일을 다운 받는 코드를 추가함
- 하나의 요청에 파일을 읽어 출력하는 것과 다운로드 하는 것을 동시에 수행함
- 파일 데이터(content)를 이용하여 MEMFS에 파일을 저장하고, 이를 JavaScript에서 읽고 (`FS.readFile(fileName)`), 이를 Blob 객체로 만들어 파일을 다운로드 함

```JavaScript
Module.get_request_download_file_example("test_file-{timestamp}.txt");
```

- [테스트용 웹 서버 구축하기 - 파일 다운로드 테스트](/posts/building-test-web-server/#파일-다운로드-테스트)와 동일한 요청
- [파일 업로드 테스트](#파일-업로드-테스트)에서 저장된 파일의 이름 가져와 테스트에서 사용함

![GET file download](images/GET_file_download.png)
_그림 7. GET 파일 다운로드 요청 결과 - 응답 본문을 문자열로 읽어 파일 내용을 출력하고 동시에 요청한 파일을 내 컴퓨터에 다운로드 함_

## FAQ

{{< faq summary="`EM_ASM`/`EM_JS`를 사용하면 JavaScript 그대로 사용할 수 있어서 편할 것 같은데, Emscripten Fetch API를 사용하면 어떤 장점이 있나요?" >}}

- Emscripten Fetch API를 사용하면 요청 설정, 콜백, 응답 처리를 C/C++ 코드 안에서 일관되게 관리할 수 있습니다. `EMSCRIPTEN_FETCH_LOAD_TO_MEMORY`를 사용하면 응답을 Wasm 메모리에서 바로 처리할 수 있고, `userData`를 통해 C++ 객체나 요청별 상태를 콜백에 전달하기도 편리합니다. 또한 진행률 확인, timeout, IndexedDB 저장과 같은 기능을 `emscripten_fetch_attr_t`의 옵션으로 설정할 수 있습니다.
- 반면 DOM 조작이 필요하거나 `Response`, `Blob`, `ReadableStream` 등 브라우저 Fetch API의 기능을 직접 사용하려면 `EM_ASM`/`EM_JS`에서 JavaScript의 `fetch`를 호출하는 편이 더 단순할 수 있습니다. 어느 방법이 항상 더 좋은 것은 아니며, 요청과 응답을 주로 C++에서 처리한다면 Emscripten Fetch API가, 브라우저 기능과의 연동이 중심이라면 JavaScript Fetch API가 더 자연스럽습니다.
  {{< /faq >}}

{{< faq summary="서버는 정상적으로 동작하는데 Emscripten Fetch 요청에서 CORS 오류가 발생해요." >}}

- 브라우저에서 실행되는 Emscripten Fetch 요청도 일반적인 XHR과 동일하게 CORS 정책의 적용을 받습니다. 이 글처럼 클라이언트가 `http://localhost:8080`, API 서버가 `http://localhost:3000`에서 실행되면 포트가 다르므로 서로 다른 origin입니다. 서버가 클라이언트의 origin을 `Access-Control-Allow-Origin`으로 허용해야 합니다.
- `Authorization` 헤더나 일부 `Content-Type`을 포함한 요청은 실제 요청 전에 preflight 요청(`OPTIONS`)을 보낼 수 있습니다. 이때 서버가 요청 메서드와 `Authorization`, `Content-Type` 등의 헤더를 허용하지 않으면 실제 요청은 전송되지 않습니다. 개발자 도구의 Network 탭에서 실패한 `OPTIONS` 요청과 응답 헤더를 먼저 확인하는 것이 좋습니다.
- 쿠키 기반 인증을 위해 `withCredentials`를 사용한다면 서버도 `Access-Control-Allow-Credentials: true`를 응답해야 합니다. 이 경우 `Access-Control-Allow-Origin`에는 `*`를 사용할 수 없고 허용할 origin을 명시해야 합니다.
  {{< /faq >}}

{{< faq summary="비동기 Fetch 요청에서 `requestData`, `userData`, `fetch->data`는 언제까지 사용할 수 있나요?" >}}

- `requestData`가 가리키는 메모리는 요청이 끝날 때까지 유효해야 합니다. 따라서 지역 변수처럼 요청 함수가 반환될 때 사라지는 메모리를 전달하면 안 되며, 이 글의 POST 예제처럼 힙에 할당한 뒤 성공 또는 실패 콜백에서 해제하는 방법을 사용할 수 있습니다. `emscripten_fetch()`가 요청을 시작하지 못해 `nullptr`을 반환한 경우에는 콜백이 호출되지 않으므로 호출한 쪽에서 직접 해제해야 합니다.
- `userData`는 포인터 값을 콜백에 전달할 뿐, Emscripten이 해당 객체의 수명을 관리해주지는 않습니다. 성공과 실패 경로에서 같은 메모리를 중복 해제하거나 한쪽 경로에서 해제를 빠뜨리지 않도록 주의해야 합니다.
- `fetch->data`는 `EMSCRIPTEN_FETCH_LOAD_TO_MEMORY`를 지정한 경우에 응답 데이터를 가리키며, `emscripten_fetch_close(fetch)`를 호출하면 함께 해제됩니다. 콜백 이후에도 데이터가 필요하다면 `close`를 호출하기 전에 다른 버퍼나 객체로 복사해야 합니다. 요청이 끝난 뒤에는 성공과 실패 여부에 관계없이 `emscripten_fetch_close()`를 호출해야 합니다.
  {{< /faq >}}

{{< faq summary="큰 파일을 받을 때 Wasm 메모리가 부족해지거나 브라우저가 느려져요." >}}

- `EMSCRIPTEN_FETCH_LOAD_TO_MEMORY`는 응답 전체를 Wasm 힙에 올리므로 파일이 크면 메모리 사용량이 크게 증가합니다. 내려받은 데이터를 즉시 처리할 필요 없이 나중에 사용할 목적이라면 `LOAD_TO_MEMORY`를 빼고 `EMSCRIPTEN_FETCH_PERSIST_FILE`을 사용하여 IndexedDB에 저장하는 방법을 고려할 수 있습니다. 이 경우 성공 콜백의 `fetch->data`에는 파일 데이터가 제공되지 않습니다.
- 서버가 Range 요청을 지원하고 부분 데이터만 필요하다면 HTTP Range 요청을 사용할 수 있습니다. Emscripten의 `EMSCRIPTEN_FETCH_STREAM_DATA`도 있지만 브라우저 지원에 제약이 있으므로, 여러 브라우저에서 스트리밍 처리가 필요하다면 JavaScript의 Fetch API와 `ReadableStream`을 사용하는 방법도 함께 검토하는 것이 좋습니다. 자세한 내용은 [Emscripten 공식 문서의 대용량 파일 관리](https://emscripten.org/docs/api_reference/fetch.html#managing-large-files)를 참고합니다.
  {{< /faq >}}

{{< faq summary="`EMSCRIPTEN_FETCH_REPLACE`를 사용하면 브라우저의 HTTP 캐시를 무시하나요?" >}}

- 아니요. `EMSCRIPTEN_FETCH_REPLACE`는 Emscripten Fetch가 사용하는 IndexedDB의 기존 데이터를 다시 사용할지 결정하는 옵션입니다. `EMSCRIPTEN_FETCH_PERSIST_FILE` 없이 사용하면 IndexedDB를 읽거나 쓰지 않고 네트워크 요청을 수행하지만, 브라우저의 일반적인 HTTP 캐시 동작은 그대로 적용될 수 있습니다.
- HTTP 캐시를 제어하려면 요청의 `Cache-Control` 헤더나 서버가 보내는 캐시 관련 응답 헤더를 별도로 설정해야 합니다.
  {{< /faq >}}

## 참고 자료

- [Emscripten Fetch API](https://emscripten.org/docs/api_reference/fetch.html)
