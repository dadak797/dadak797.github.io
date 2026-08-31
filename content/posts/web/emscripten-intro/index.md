---
title: "Emscripten 소개"
date: "2026-07-25T16:14:39+09:00"
lastmod: "2026-07-26T23:28:39+09:00"
draft: false
description: "Emscripten은 C/C++ 코드를 WebAssembly 코드(.wasm)로 변환해주는 컴파일러이며, 이 WebAssembly를 통해 브라우저에서 돌아가는 고성능 어플리케이션을 만들 수 있다."

categories:
  - Web

tags:
  - emscripten
  - webassembly
  - compiler

ShowToc: true
TocOpen: false
ShowReadingTime: false
---

> [!SUMMARY]
> Emscripten은 C/C++ 코드를 WebAssembly 코드(.wasm)로 변환해주는 컴파일러이며, 이 WebAssembly를 통해 브라우저에서 돌아가는 고성능 어플리케이션을 만들 수 있다.

## 웹어셈블리(WebAssembly)

- WebAssembly(Wasm)는 브라우저에서 네이티브에 가까운 속도로 돌아가는 바이너리 포맷
- JavaScript를 대체하는게 아니라 단점을 보완하는 기술
- 사람이 직접 작성하는 것이 아니라 컴파일 타깃이기 때문에, C/C++로 작성된 코드를 Wasm으로 바꿔주는 도구가 필요하고 Emscripten이 그 역할을 함 (Rust로 작성된 코드는 자체 LLVM 백엔드를 이용하여 Wasm으로 변환을 할 수 있음)

## Emscripten

- Emscripten은 C/C++로 작성된 코드를 WebAssembly 바이너리(.wasm)와 JavaScript 접착 코드(.js)로 변환해주는 컴파일러
- 런타임에 Wasm 모듈은 DOM에 직접 접근할 수 없지만, 접착 코드(.js)를 이용하여 웹 브라우저의 DOM과 Web API에 접근할 수 있음
- Node.js 환경에서도 접착 코드를 이용하여 Wasm 모듈의 기능을 활용할 수 있음

![Emscripten_Toolchain_Flow](images/emscripten_toolchain_flow.svg)
_그림 1. Emscripten은 C/C++를 `.wasm` 바이너리와 `.js` glue로 컴파일하며, 이 glue가 WebAssembly와 DOM·Web API를 이어준다._

## 장점

1. 빠름
   - WebAssembly는 미리 컴파일된 바이너리 포맷이라, JavaScript처럼 실행 중에 파싱·최적화하는 단계 없이 곧바로 실행됨
   - 다만 성능 이점은 문제 유형에 따라 달라서, 무거운 연산에서는 큰 차이가 나지만 가벼운 작업에서는 경계 호출 비용 때문에 이점이 작을 수 있음
   - [같은 연산, 다른 실행 환경: JavaScript·WebAssembly·네이티브 C++ 성능 비교](/posts/performance-js-wasm-native/)
2. 재활용성
   - 기존에 작성된 방대한 C++ 기반의 코드 재활용 가능
   - 빌드된 Wasm 모듈을 통째로 가져와서 사용할 수도 있음. Pyodide는 CPython을 Emscripten으로 빌드한 모듈로, 이 모듈을 이용하면 브라우저에서 Python을 돌릴 수 있음
3. 보안성
   - 기본적으로 격리된 실행 모델을 제공함. Wasm 모듈은 선형 메모리 밖으로 나갈 수 없음
   - Wasm 모듈에서는 JS로부터 import 된 기능만 사용할 수 있고, JS에서도 Wasm 모듈에서 export 한 기능만 사용할 수 있음
   - 다만 이 격리의 실제 범위는 호스트(JS)가 무엇을 import로 넘겨주느냐에 따라 정해짐
4. 서버의 부하를 분산
   - 서버에서 처리하던 무거운 작업도 브라우저에서 처리할 수 있음 (PC에서 처리 가능한 범위 내에서)

## 단점

1. 사용자 파일 접근
   - 브라우저 환경 안에서 동작하기 때문에 사용자의 파일에 직접 접근할 수 없음. HTML input 태그의 file 속성으로 파일을 불러온 뒤 WebAssembly 메모리로 복사하는 과정이 필요하여 native 환경에 비해 로딩이 번거롭고 느림
   - 로딩 시점에 필요한 리소스는 미리 Wasm이 읽을 수 있는 형태로 가상 파일시스템에 넣어둬야 하며, 링크 옵션(`--preload-file`)으로 이 과정을 처리함
2. HTML의 DOM에 접근성
   - WebAssembly에서 DOM에 직접 접근할 수는 없고, 별도의 매크로(`EM_JS`, `EM_ASM` 등)를 이용하여 JavaScript를 통해 접근해야 함
3. 모듈 초기 로딩
   - Wasm 모듈(.wasm)과 JS 연동을 위한 glue 코드(.js)를 다운로드해야 해서 첫 로딩이 무거울 수 있음
4. JS <span>&harr;</span> Wasm 경계 호출 오버헤드
   - 경계를 자주 넘나들거나 문자열·큰 데이터를 주고 받는 경우에는 변환·메모리 복사 비용(marshaling)이 커질 수 있음
   - 문자열을 주고받을 때 실제로 어떤 과정을 거치는지는 [JavaScript에서 C++ 함수 호출 하기 - ccall, cwrap](/posts/call-cpp-from-js/) 참고
5. 멀티스레딩
   - 멀티스레딩을 사용하려면 SharedArrayBuffer가 필요하고, 이를 위해 COOP/COEP 헤더 설정이 강제됨
   - 설정 자체는 간단하지만, COEP는 페이지 전체에 적용되는 전역 정책이라 외부 CDN·서드파티 위젯 등 교차 출처 리소스가 함께 대응돼 있어야 함. 자체 호스팅 중심의 앱에서는 부담이 적지만, 외부 의존이 많은 사이트에서는 호환성 점검이 필요함

## 어떤 경우에 사용하는게 좋을까?

- 서버 왕복 없이 클라이언트에서 처리를 끝내고 싶을 때 (프라이버시 이점도 있음)
  - 예) FFmpeg.wasm (브라우저 내 미디어 변환), sql.js (브라우저 내 SQL DB)
- 대규모·고성능 연산이나 저지연 실시간 처리가 필요한 웹 앱
  - 예) Figma, Photoshop Web
- 기존 대형 C/C++ 앱을 웹으로 이식하고 싶을 때 - 예) AutoCAD Web, Google Earth
  ![네이티브와 웹에서 실행 중인 C++ 3D 뷰어](images/NativeToWeb.webp)
  _그림 2. 네이티브(왼쪽)로 작성한 C++ 3D 뷰어를 Emscripten으로 컴파일해 웹 브라우저(오른쪽)에서 동일하게 실행한 모습. 같은 코드가 두 환경에서 그대로 동작한다._
- 여러 플랫폼에서 동일한 핵심 로직을 공유하고 싶을 때 (C++ 코어 하나로 네이티브 앱 + 웹)
- 성숙한 C/C++ 라이브러리를 재작성 없이 웹에 그대로 가져오고 싶을 때
  - 예) OpenCASCADE (CAD 커널), VTK (과학·의료 시각화)

> [!NOTE]
> 위 제품들의 Emscripten/Wasm 활용 여부는 2026년 7월 25일 기준으로 확인한 정보입니다. 기술 구성은 변경될 수 있으므로, 인용할 때는 각 사의 최신 엔지니어링 자료를 확인하는 것이 좋습니다.

## FAQ

- C++ 코드를 하나도 수정하지 않아도, 모두 Wasm으로 빌드할 수 있나요?
  - Windows 환경에서 코딩한 C++ 코드를 아무 수정 없이 Linux에서 사용할 수 없듯이, Wasm으로 빌드하는 경우에도 약간의 수정이 필요합니다.
  - 아무 수정 없이 빌드되지 않는 대표적인 경우:
    - Emscripten으로 포팅(Porting)되지 않은 라이브러리를 사용하는 경우
    - 파일 접근, 스레드, 네트워크 등 OS/브라우저에 의존적인 기능을 사용하는 경우
- 그래픽 프로그래밍도 가능한가요?
  - 네. 가능합니다. 데스크톱 OpenGL 전체가 아니라 OpenGL ES 서브셋 기준으로, 이 스펙에 맞게 코딩된 C++ 코드는 Wasm으로 컴파일 시 내부적으로 WebGL로 변환되어 HTML canvas 태그를 이용해 브라우저에서 렌더링할 수 있습니다. (대략 OpenGL ES 2.0 ↔ WebGL 1, OpenGL ES 3.0 ↔ WebGL 2로 대응됩니다.)
  - WebGPU 스펙에 맞게 C++로 코딩된 코드도 Wasm으로 컴파일하여 canvas 태그로 렌더링할 수 있습니다. 다만 WebGPU는 아직 확산 단계이므로 대상 브라우저의 지원 상황을 확인하는 것이 좋습니다.
- 메모리는 제한 없이 사용할 수 있나요?
  - 아니요. CPU 메모리는 기본 메모리 모델(wasm32)에서 주소 공간이 4GB로 제한됩니다. 다만 2025년 9월 17일 기준, Chrome 133, Firefox 134 및 Node.js 24.0 이상에서는 Memory64를 지원하므로 4GB를 넘는 메모리를 사용할 수 있습니다. 단, 웹 환경에서 64비트 메모리의 최대 크기는 16GB로 제한됩니다. 자세한 내용은 [WebAssembly 3.0 발표 자료](https://webassembly.org/news/2025-09-17-wasm-3.0/)를 참고하세요.
  - GPU 메모리는 WebGL/WebGPU를 통해 브라우저가 중개하므로, 그래픽 카드의 전체 용량을 그대로 쓸 수 있는 것은 아니며 브라우저가 컨텍스트에 거는 제한과 다른 앱과의 공유 범위 안에서 사용할 수 있습니다.
- 디버깅은 어떻게 하나요?
  - 소스맵(DWARF) 정보를 함께 빌드하면, 브라우저 개발자 도구에서 원본 C++ 소스 레벨로 중단점을 걸고 디버깅할 수 있습니다.
- 빌드 결과물 크기가 큰가요? 줄일 수 있나요?
  - 최적화 옵션(`-Os`, `-Oz`)과 링크 타임 최적화, 그리고 gzip/brotli 압축 등을 이용하여 크기를 줄일 수 있습니다.
- 기존 CMake/Make 프로젝트를 그대로 쓸 수 있나요?
  - `emcmake`, `emmake`로 기존 빌드 스크립트를 감싸서 Emscripten 툴체인으로 빌드할 수 있습니다.
- 어떻게 시작하나요?
  - [Emscripten 설치 및 예제](/posts/emscripten-install/)

## 참고 자료

- [Emscripten 공식 문서](https://emscripten.org/docs/)
- [WebAssembly 공식 홈페이지](https://webassembly.org/)
