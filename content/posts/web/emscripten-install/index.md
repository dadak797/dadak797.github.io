---
title: "Emscripten 설치 및 예제"
date: "2026-07-28T20:44:36+09:00"
lastmod: "2026-07-29T08:25:55+09:00"
draft: false
description: "Windows·macOS·Linux에서 Emscripten을 설치하고, C++ 코드를 WebAssembly로 빌드해 Node.js와 브라우저에서 실행하는 방법을 정리합니다."

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
> Emscripten을 각 플랫폼에서 설치하는 방법을 알려주고, C++로 작성한 코드를 WebAssembly로 빌드하여 Node.js와 브라우저 환경에서 예제를 실행시킨다.

## Windows에서 설치하기 (명령 프롬프트, CMD)

### 1. Prerequisites 설치

- [Git](https://git-scm.com/install/windows)
- [Python](https://www.python.org/downloads/)

### 2. 저장소 복사

```bash
git clone https://github.com/emscripten-core/emsdk.git
cd emsdk  # 저장소 폴더로 이동
```

### 3. 설치 및 최신 버전 활성화

```bash
emsdk install latest
emsdk activate latest
```

### 4. 환경 변수 추가

```bash
emsdk_env
```

- 위의 명령은 현재 세션에 한해 환경 변수를 잡아주는 것이기 때문에, 영구적으로 사용하려면 PATH에 기본 저장소 폴더 `emsdk`와 `emsdk\upstream\emscripten`, `emsdk\node\{NODE_VERSION}\bin`를 추가해야 함

### 5. emsdk 버전 확인하기

```bash
emsdk list

# 실행 결과
All recent (non-legacy) installable versions are:
         5.0.3    INSTALLED
         5.0.3-asserts
         5.0.2
         5.0.2-asserts
         5.0.1
         5.0.1-asserts
         5.0.0
         5.0.0-asserts
```

- 목록이 잘 나오면 설치가 잘 되었다는 뜻
- 설치 가능한 버전과 현재 설치되어 있는 버전(INSTALLED라고 표기된 것)을 알려줌

### 6. 특정 버전 설치하기

```
emsdk install 5.0.0  # 5.0.0 버전을 설치
```

- 가끔 다른 버전의 emsdk로 빌드된 정적 라이브러리를 가져와서 내 프로젝트에 포함시킬 때 빌드가 안되거나, 실행 중 오류가 발생하는 경우가 있음
- 이 경우에는 위의 명령어로 동일한 버전을 설치하여 사용할 것을 권장

> [!NOTE]
> PowerShell에서 설치하는 경우에는 `emsdk` 대신 `.\emsdk.bat`, `emsdk_env` 대신 `.\emsdk_env.ps1`을 실행하면 된다.

## macOS에서 설치하기

### 1. Prerequisites 설치

- Git

```bash
brew install git
```

- Python

```bash
brew install python
```

### 2. 저장소 복사

- [Windows와 동일](#2-저장소-복사)

### 3. 설치 및 최신 버전 활성화

```bash
./emsdk install latest
./emsdk activate latest
```

### 4. 환경 변수 추가

```bash
source ./emsdk_env.sh
```

- 위의 명령은 현재 세션에 한해 환경 변수를 잡아주는 것이기 때문에, 영구적으로 사용하려면 `~/.zshrc` 파일에 `emsdk_env.sh`의 경로를 추가해 놓아야 함

```bash
# emsdk
source {EMSDK_INSTALL_PATH}/emsdk_env.sh
```

### 5. emsdk 버전 확인하기

- [Windows와 동일](#5-emsdk-버전-확인하기)

### 6. 특정 버전 설치하기

- [Windows와 동일](#6-특정-버전-설치하기)

## Linux(Ubuntu)에서 설치하기

### 1. Prerequisites

- Git

```bash
sudo apt update
sudo apt install git
```

- Python

```bash
sudo apt install python3 python3-pip python3-venv
```

### 2. 저장소 복사

- [Windows와 동일](#2-저장소-복사)

### 3. 설치 및 최신 버전 활성화

- [macOS와 동일](#3-설치-및-최신-버전-활성화-1)

### 4. 환경 변수 추가

```bash
source ./emsdk_env.sh
```

- 위의 명령은 현재 세션에 한해 환경 변수를 잡아주는 것이기 때문에, 영구적으로 사용하려면 `~/.bashrc` 파일에 `emsdk_env.sh`의 경로를 추가해 놓아야 함

```bash
# emsdk
source {EMSDK_INSTALL_PATH}/emsdk_env.sh
```

### 5. emsdk 버전 확인하기

- [Windows와 동일](#5-emsdk-버전-확인하기)

### 6. 특정 버전 설치하기

- [Windows와 동일](#6-특정-버전-설치하기)

---

## 예제 돌려보기 - 1

> [!GOAL]
> 콘솔창에 `"Hello, WebAssembly!"`를 출력하는 코드 작성하기

### 소스 코드

```cpp
// main.cpp
#include <iostream>

int main() {
  std::cout << "Hello, WebAssembly!" << std::endl;
  return 0;
}
```

### 빌드하기

```bash
em++ main.cpp -o main.js
```

- `em++`
  - GCC, Clang의 `g++`과 대응되는 C++용 컴파일러를 실행
  - `emcc`는 `gcc`와 대응됨. `emcc`로 빌드해도 `.cpp`는 동일하게 C++로 빌드됨
- `-o`
  - GCC, Clang에서와 같이 출력 파일의 이름을 지정하는 옵션
  - Emscripten에서는 출력 파일의 확장자에 따라 출력물의 개수가 달라짐
  - `{출력_파일_이름}.wasm`
    - WebAssembly 바이너리(`.wasm`)만 생성됨
    - `.wasm` 로딩 및 접착 코드(`.js`)를 직접 작성해야 함
  - `{출력_파일_이름}.js`
    - `.wasm`과 `.js`를 생성해 줌
    - 가장 많이 사용
  - `{출력_파일_이름}.html`
    - `.wasm`과 `.js`와 테스트용 HTML 파일(`.html`)이 출력됨

### Node.js로 실행하기

```bash
node main.js

# 실행 결과
Hello, WebAssembly!
```

### HTML이 출력되게 빌드하기

```bash
em++ main.cpp -o index.html --shell-file template.html
```

- HTML 템플릿을 사용하면 원하는 위치에 접착 코드를 삽입할 수 있음
- HTML 템플릿 파일에는 `{{{ SCRIPT }}}`가 포함되어 있어야 하고, 이 부분이 접착 코드를 로딩하는 코드로 변경됨
- Python으로 웹 서버를 구동하기 위해 `index.html`로 빌드함

### 브라우저에서 실행하기

```bash
python -m http.server 8080
```

- 파이썬 http.server로 서버를 실행하여 빌드 결과물을 브라우저에서 실행할 수 있다.
- `localhost:8080`으로 접속 <span>&rarr;</span> `index.html` 로딩 <span>&rarr;</span> `index.js` 로딩 <span>&rarr;</span> `index.wasm` 로딩

![Emscripten_Example_1](images/Emscripten_Example_1.png)
_그림 1. 브라우저에서 첫 번째 예제의 wasm 코드를 실행한 모습_

- `main` 함수 안에 작성된 코드는 Wasm 모듈의 초기화가 완료되면 자동으로 실행됨

### 예제 코드 및 emsdk 버전

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-1
- emsdk 5.0.3

## 예제 돌려보기 - 2

> [!GOAL]
> C++로 작성한 함수를 브라우저와 Node.js 런타임을 이용하여 직접 호출하기

### 소스 코드

```c++
// prime_number.cpp
#include <iostream>
#include <emscripten.h>

extern "C" {
  EMSCRIPTEN_KEEPALIVE
  void FindPrimes(int limit) {
    if (limit < 2) {
      std::cout << "No primes" << std::endl;
      return;
    }

    std::cout << "Primes up to " << limit << ": ";

    for (int i = 2; i <= limit; ++i) {
      bool isPrime = true;
      for (int j = 2; j * j <= i; ++j) {
        if (i % j == 0) {
          isPrime = false;
          break;
        }
      }

      if (isPrime) {
        std::cout << i << " ";
      }
    }

    std::cout << std::endl;
  }
}
```

- `extern "C"`
  - 컴파일러가 함수 이름을 바꾸지 못하게 하기 위해서 필요함 (네임 맹글링 방지)
  - `emcc`/`em++` 어느 걸로 빌드하든 `.cpp`이면 맹글링이 일어남
- `EMSCRIPTEN_KEEPALIVE`
  - 함수를 지우지 않게 함
  - Emscripten은 최적화 과정에서 C++ 내부에서 사용하지 않는 함수들을 지워 용량을 최적화 함
  - JavaScript 쪽에서 호출할지 말지 Emscripten 입장에서는 알 방법이 없기 때문에, 코드에 명시적으로 표시하여 컴파일러에게 이 함수를 지우지 말라고 알려줘야 함

> [!NOTE]
> **네임 맹글링 (Name Mangling)** - C++에서는 함수 오버로딩을 지원하기 위해, 함수 이름을 그대로 사용하지 않고 매개변수 타입 등의 정보를 섞어 내부적으로 고유한 이름으로 변경하는 것. 이름이 변경되어 있으면 JavaScript 쪽에서는 이 이름을 알 수가 없기 때문에 맹글링이 일어나지 못하게 막아야 해당 함수를 호출할 수 있다.

### 빌드하기

```bash
em++ prime_number.cpp -o index.js -s EXPORTED_FUNCTIONS="['_FindPrimes']"
```

- `-s`
  - 컴파일 옵션을 설정할 때 사용
  - `-s X=Y` 형태로 옵션을 설정
    - X는 컴파일 옵션의 종류
    - Y는 컴파일 옵션 파라미터
    - `-s`뒤에는 한 칸 띄워도 되고 붙여도 되지만, `X=Y`는 다 붙여서 작성할 것
  - `EXPORTED_FUNCTIONS`
    - `_FindPrimes`: 내보낼 함수의 이름. C++에서의 함수 이름 앞에 underscore(`_`)를 붙여줘야 함

### REPL(Node.js 대화형 콘솔)에서 실행하기

- 터미널에서 `node`를 입력하여 대화형 콘솔로 들어갈 수 있음

```node
> const Module = require('./index.js');
> Module._FindPrimes(20);

// 실행 결과
Primes up to 20: 2 3 5 7 11 13 17 19
```

- Module은 Wasm 모듈을 나타냄
- 내보낸 함수의 이름인 `_FindPrimes`를 사용해야 함

### HTML이 출력되게 빌드하기

```bash
em++ prime_number.cpp -o index.html --shell-file template.html -s EXPORTED_FUNCTIONS="['_FindPrimes']"
```

### 브라우저에서 실행하기

```bash
python -m http.server 8080
```

![Emscripten_Example_2](images/Emscripten_Example_2.png)
_그림 2. 브라우저에서 두 번째 예제의 wasm 코드를 실행한 모습_

> [!CAUTION]
> Wasm의 용량이 큰 경우에는 Module이 초기화되는 데 시간이 걸리기 때문에, 내보낸 함수를 바로 호출하는 경우에 오류가 발생할 수 있음. `Module.onRuntimeInitialized`를 콜백 함수로 정의하여 모듈 초기화가 끝난 뒤 함수가 호출되게 하거나, 초기화가 완료될 때까지 충분히 기다린 후에 함수를 호출해야 함. 지금 예제는 짧기 때문에 대화형 콘솔에서 순차적으로 실행해도 별 문제가 발생하지 않음. 예제 1에서는 `main` 함수가 Wasm 모듈이 초기화되면 자동으로 실행되기 때문에, 타이밍에 대해 걱정할 필요가 없었음. `Module` 객체 자체에 대한 자세한 설명은 [Emscripten Module](/posts/emscripten-module/)에서 다룬다.

### 예제 코드 및 emsdk 버전

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-2
- emsdk 5.0.3

## FAQ

- 여러 OS에서 같은 소스로 빌드하면 결과물이 같나요?
  - 네. `.wasm`, `.js`, `.html` 모두 플랫폼과 무관하게 동일합니다.
- 미리 빌드된 라이브러리를 내 코드와 함께 WebAssembly로 빌드하려면 무엇을 맞춰야 하나요?
  - **빌드 대상**: 라이브러리도 Emscripten으로 빌드된 것이어야 합니다. Native용 바이너리는 링크되지 않습니다.
  - **Emscripten 버전**: 라이브러리와 내 코드의 버전을 맞추세요. 다르면 빌드 실패나 런타임 오류가 날 수 있습니다.
  - **컴파일 옵션**: `-pthread`(멀티스레딩), `-sMEMORY64=1`(메모리 모델)처럼 ABI에 영향을 주는 옵션은 양쪽이 같아야 합니다. 다르면 빌드가 실패합니다.
- 브라우저에서 `index.html`을 더블클릭해서 바로 열면 왜 안 되나요?
  - `file://`로 열면 브라우저 보안 정책 때문에 `.wasm`을 불러오지 못합니다. 그래서 `python -m http.server`로 로컬 서버를 띄운 뒤 `localhost`로 접속하는 것입니다.
- `emsdk` 명령을 찾을 수 없다고 나와요 (`command not found`)
  - 새 터미널에서는 환경 변수가 초기화됩니다. 매번 `emsdk_env`(Windows) 또는 `source ./emsdk_env.sh`(macOS/Linux)를 실행하거나, PATH에 영구적으로 등록하세요. macOS·Linux에서는 `emsdk`가 PATH에 없을 때 `./emsdk`처럼 앞에 `./`를 붙여야 합니다.
- `.js`만 있으면 되지, `.wasm`은 왜 따로 생기나요?
  - 실제 컴파일된 코드는 `.wasm`에 들어 있고, `.js`는 그 `.wasm`을 로드·초기화하고 JS와 이어주는 접착(glue) 코드입니다. 브라우저/Node.js 모두 `.js`를 통해 `.wasm`을 실행하므로 두 파일이 같은 위치에 있어야 합니다.
- `node main.js`는 되는데, 왜 브라우저에서는 서버가 필요한가요?
  - Node.js는 파일 시스템에서 `.wasm`을 직접 읽을 수 있지만, 브라우저는 `file://`에서 `.wasm`을 가져오지 못합니다. 그래서 브라우저 실행에만 로컬 서버가 필요합니다.

## 참고 자료

- [Emscripten - Download and Install](https://emscripten.org/docs/getting_started/downloads.html)
- [Emscripten - Interacting with code](https://emscripten.org/docs/porting/connecting_cpp_and_javascript/Interacting-with-code.html)
