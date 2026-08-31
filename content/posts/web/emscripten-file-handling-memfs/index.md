---
title: Emscripten에서 파일 다루기 1 - 파일 올리기와 결과 내려받기 (MEMFS)
date: 2026-08-31
draft: false
description: Emscripten에서 파일을 다루기 위한 Virtual File System을 소개한다. 외부 C++ 라이브러리 tinyobjloader를 내 프로젝트에 포함시키고 Emscripten의 MEMFS를 이용하여 사용자의 Wavefront OBJ 파일을 읽고, 내려받는 과정을 보여준다.
categories:
  - Web
tags:
  - emscripten
  - webassembly
  - javascript
  - cpp
  - memfs
ShowToc: true
TocOpen: false
ShowReadingTime: false
---

> [!SUMMARY]
> Emscripten에서 파일을 다루기 위한 Virtual File System을 소개한다. 외부 C++ 라이브러리 tinyobjloader를 내 프로젝트에 포함시키고 Emscripten의 MEMFS를 이용하여 사용자의 Wavefront OBJ 파일을 읽고, 내려받는 과정을 보여준다.

## Emscripten Virtual File System (VFS)

Emscripten에 의해 빌드된 WebAssembly는 브라우저 환경 내에서 동작하기 때문에, 사용자의 로컬 파일 시스템에 임의로 접근할 수 없다. 네이티브 환경에서는 `std::ifstream ifs("my_data.dat");`와 같이 파일 경로를 지정하여 운영체제의 파일 시스템에 직접 접근할 수 있지만, 브라우저에서는 보안상의 이유로 이러한 방식의 접근이 제한된다. 로컬 파일을 사용하려면 사용자가 파일 선택 등의 방법을 통해 명시적으로 접근을 허용해야 한다.

그렇다고 Emscripten으로 빌드된 C++ 코드에서 fopen()과 std::ifstream을 사용할 수 없는 것은 아니다. Emscripten은 C++ 프로그램이 기존의 파일 입출력 API를 그대로 사용할 수 있도록 Virtual File System (VFS)를 제공한다.

VFS는 C++ 코드에서만 접근할 수 있는 것은 아니다. Emscripten이 JavaScript 측에 제공하는 API인 `FS.*`를 이용하면 JavaScript 쪽에서도 동일한 가상 파일에 접근할 수 있다.

![Emscripten VFS](images/emscripten-vfs.png)
_그림 1. Emscripten Virtual File System(VFS)은 C++ 파일 API와 JavaScript `FS.*` API가 만나는 공통 파일 시스템 추상화 계층으로 볼 수 있다. VFS의 종류로는 MEMFS, NODEFS, IDBFS, WORKERFS, PROXYFS, WasmFS가 있다._

## OBJ 파일을 브라우저에서 읽고, 결과 내려받기 (MEMFS)

이 예제에서는 여러 가지 VFS 중에서 MEMFS를 사용하고 tinyobjloader라는 외부 라이브러리를 활용하여 두 개의 OBJ 파일을 읽고, 파싱 결과(정점의 개수, 면의 개수 등)를 확인하고, 두 오브젝트를 하나로 병합하여 파일로 내려받는 과정을 보여 준다. 두 오브젝트와 병합된 오브젝트는 [Blender](https://www.blender.org/)를 이용하여 가시화 한다.

### MEMFS

- 빌드 옵션을 별도로 설정하지 않아도 기본적으로 포함되는 VFS
- Wasm 런타임이 초기화 되는 시점에 `/`에 마운트 됨
- 모든 파일은 메모리에 존재함
  - 페이지가 새로고침 될 때 사라짐
  - 파일의 사용이 끝나면 삭제하여 메모리 공간을 확보하는 것이 좋음

### tinyobjloader

- Wavefront (`.obj`/`.mtl`) 형식의 파일을 읽어 파싱해주는 header-only 라이브러리
- Wavefront 형식은 3D model을 저장하는 포맷 중 하나로 무료 오픈소스 모델링 소프트웨어인 [Blender](https://www.blender.org/)를 이용하여 렌더링 할 수 있음
- `tiny_obj_loader.h`만 포함하여 사용할 수 있음
- https://github.com/tinyobjloader/tinyobjloader
- Version - 2.0.0

### 소스 코드 - memfs.cpp

```C++
#define TINYOBJLOADER_IMPLEMENTATION
#include "tiny_obj_loader/tiny_obj_loader.h"
#include "nlohmann/json.hpp"

#include <optional>
#include <string>
#include <vector>
#include <iostream>
#include <fstream>
#include <filesystem>
#include <iomanip>
#include <limits>
#include <utility>
#include <emscripten.h>
#include <emscripten/bind.h>

using json = nlohmann::json;
namespace fs = std::filesystem;

// OBJ 파일로부터 읽어들인 데이터를 저장하는 구조체
struct ObjFile {
  std::string filename;
  tinyobj::attrib_t attrib;
  std::vector<tinyobj::shape_t> shapes;
};

std::vector<ObjFile> g_ObjFiles;

// 특정 MEMFS 경로에 있는 파일과 폴더 목록을 출력
void PrintFiles(const std::string& path) {
  for (const auto& entry : fs::directory_iterator(path)) {
    if (entry.is_directory()) {
      std::cout << "[DIR ] ";
    }
    else if (entry.is_regular_file()) {
      std::cout << "[FILE] ";
    }
    std::cout << entry.path() << std::endl;
  }
}

// 파일 이름들이 담긴 JSON 문자열을 받아 OBJ 파일들을 읽고 요약 정보를 리턴
std::optional<std::string> LoadObjFiles(const std::string& filenames) {
  std::cout << "=== Before loading OBJ files ===" << std::endl;
  PrintFiles("/");  // MEMFS root의 파일과 폴더 목록 확인

  g_ObjFiles.clear();

  json files = json::parse(filenames);
  if (!files.is_array()) {
    std::cerr << "Input must be a JSON array of file names." << std::endl;
    return std::nullopt;
  }

  json summary = json::array();  // 요약 정보를 담을 JSON array

  for (const auto& file : files) {
    if (!file.is_string()) {
      std::cerr << "Each file name must be a string." << std::endl;
      return std::nullopt;
    }
    std::string filename = file.get<std::string>();

    tinyobj::attrib_t attrib;
    std::vector<tinyobj::shape_t> shapes;
    std::vector<tinyobj::material_t> materials;
    std::string warn, err;

    // OBJ 파일로부터 vertex 속성, shape, material을 읽음
    bool ok = tinyobj::LoadObj(
      &attrib,
      &shapes,
      &materials,
      &warn,
      &err,
      filename.c_str(),
      nullptr,
      false  // true로 설정하면 모든 face를 triangulate 함
    );

    fs::remove(filename.c_str());  // 성공/실패 여부와 관계없이 읽은 OBJ 파일을 MEMFS에서 삭제

    if (!warn.empty()) {
      std::cout << "Warning: " << warn << std::endl;
    }
    if (!err.empty()) {
      std::cerr << "Error: " << err << std::endl;
      continue;
    }
    if (!ok) {
      std::cerr << "Failed to load/parse .obj file." << std::endl;
      continue;
    }

    for (size_t i = 0; i < shapes.size(); ++i) {
      uint32_t triangleFaceCount = 0;    // 삼각형 face 개수
      uint32_t quadFaceCount = 0;        // 사각형 face 개수
      uint32_t ngonFaceCount = 0;        // n각형(n-gon) face 개수
      uint32_t renderTriangleCount = 0;  // triangulation 이후의 삼각형 개수

      for (const auto vertexCount : shapes[i].mesh.num_face_vertices) {
        if (vertexCount == 3) {
          ++triangleFaceCount;
        } else if (vertexCount == 4) {
          ++quadFaceCount;
        } else if (vertexCount > 4) {
          ++ngonFaceCount;
        }

        if (vertexCount >= 3) {
          renderTriangleCount += vertexCount - 2;
        }
      }

      summary.push_back({
        {"name", shapes[i].name},
        {"vertices", attrib.vertices.size() / 3},
        {"faces", shapes[i].mesh.num_face_vertices.size()},
        {"triangleFaces", triangleFaceCount},
        {"quadFaces", quadFaceCount},
        {"ngonFaces", ngonFaceCount},
        {"renderTriangles", renderTriangleCount}
      });
    }

    // 파일 이름, vertex 속성, shape 정보를 g_ObjFiles에 저장
    g_ObjFiles.push_back({
      filename,
      std::move(attrib),
      std::move(shapes)
    });
  }

  std::cout << "=== After loading OBJ files ===" << std::endl;
  PrintFiles("/");  // 임시 OBJ 파일들이 MEMFS에서 삭제되었는지 확인

  return summary.dump();  // 요약 정보를 JSON 문자열로 직렬화하여 리턴
}

// g_ObjFiles에 담긴 오브젝트들을 병합하여 OBJ 파일로 작성하고 다운로드
void MergeAndDownloadObjFiles() {
  constexpr const char* MERGED_OBJ_FILE = "/merged.obj";  // MEMFS에 생성할 출력 파일 경로
  std::ofstream output(MERGED_OBJ_FILE);
  if (!output) {
    std::cerr << "Failed to create merged OBJ file." << std::endl;
    return;
  }

  output << std::setprecision(
    std::numeric_limits<tinyobj::real_t>::max_digits10
  );
  output << "# Merged OBJ file\n";

  std::size_t vertexOffset = 0;
  std::size_t normalOffset = 0;
  std::size_t texcoordOffset = 0;

  for (std::size_t fileIndex = 0;
       fileIndex < g_ObjFiles.size();
       ++fileIndex) {
    const auto& loadedObj = g_ObjFiles[fileIndex];
    const auto& attrib = loadedObj.attrib;

    output << "\n# Source: " << loadedObj.filename << '\n';

    // Vertex 좌표 작성
    for (std::size_t i = 0; i + 2 < attrib.vertices.size(); i += 3) {
      output << "v "
             << attrib.vertices[i] << ' '
             << attrib.vertices[i + 1] << ' '
             << attrib.vertices[i + 2] << '\n';
    }

    // Vertex normal 작성
    for (std::size_t i = 0; i + 2 < attrib.normals.size(); i += 3) {
      output << "vn "
             << attrib.normals[i] << ' '
             << attrib.normals[i + 1] << ' '
             << attrib.normals[i + 2] << '\n';
    }

    // Vertex texture 좌표 작성
    for (std::size_t i = 0; i + 1 < attrib.texcoords.size(); i += 2) {
      output << "vt "
             << attrib.texcoords[i] << ' '
             << attrib.texcoords[i + 1] << '\n';
    }

    // 각 face의 vertex 인덱스 작성
    for (std::size_t shapeIndex = 0;
         shapeIndex < loadedObj.shapes.size();
         ++shapeIndex) {
      const auto& shape = loadedObj.shapes[shapeIndex];
      output << "o " << fileIndex + 1 << '_' << shapeIndex + 1;
      if (!shape.name.empty()) {
        output << '_' << shape.name;
      }
      output << '\n';

      // 여러 오브젝트의 속성이 전역 배열을 공유하기 때문에 오프셋을 적용
      std::size_t indexOffset = 0;
      for (const auto faceVertexCount : shape.mesh.num_face_vertices) {
        if (indexOffset + faceVertexCount > shape.mesh.indices.size()) {
          std::cerr << "Invalid face indices in " << loadedObj.filename
                    << std::endl;
          output.close();
          fs::remove(MERGED_OBJ_FILE);
          return;
        }

        output << 'f';
        for (std::size_t i = 0; i < faceVertexCount; ++i) {
          const auto& index = shape.mesh.indices[indexOffset + i];
          if (index.vertex_index < 0) {
            std::cerr << "Missing vertex index in " << loadedObj.filename
                      << std::endl;
            output.close();
            fs::remove(MERGED_OBJ_FILE);
            return;
          }

          const auto vertexIndex =
            static_cast<std::size_t>(index.vertex_index) + vertexOffset + 1;

          output << ' ' << vertexIndex;

          const bool hasTexcoord = index.texcoord_index >= 0;
          const bool hasNormal = index.normal_index >= 0;
          if (hasTexcoord || hasNormal) {
            output << '/';
            if (hasTexcoord) {
              output << static_cast<std::size_t>(index.texcoord_index)
                        + texcoordOffset + 1;
            }
            if (hasNormal) {
              output << '/'
                     << static_cast<std::size_t>(index.normal_index)
                          + normalOffset + 1;
            }
          }
        }
        output << '\n';
        indexOffset += faceVertexCount;
      }
    }

    vertexOffset += attrib.vertices.size() / 3;
    normalOffset += attrib.normals.size() / 3;
    texcoordOffset += attrib.texcoords.size() / 2;
  }

  output.close();
  if (!output) {
    std::cerr << "Failed to write merged OBJ file." << std::endl;
    fs::remove(MERGED_OBJ_FILE);
    return;
  }

  std::cout << "Merged " << g_ObjFiles.size()
            << " OBJ files into " << MERGED_OBJ_FILE << std::endl;

  // MEMFS 파일을 읽어 DOM을 이용하여 다운로드
  EM_ASM({
    // MERGED_OBJ_FILE 포인터를 JavaScript 문자열로 변환
    const path = UTF8ToString($0);

    // MEMFS 파일을 Uint8Array로 읽음
    const data = Module.FS.readFile(path);

    // 파일 데이터를 담은 Blob 생성
    const blob = new Blob([data], { type: "text/plain" });

    // Blob을 가리키는 임시 URL 생성
    const url = URL.createObjectURL(blob);

    // 다운로드를 위한 anchor 엘리먼트 생성
    const anchor = document.createElement("a");
    anchor.href = url;
    anchor.download = "merged.obj";
    document.body.appendChild(anchor);

    // 다운로드 시작
    anchor.click();
    anchor.remove();

    // 임시 Blob URL 해제
    setTimeout(() => URL.revokeObjectURL(url), 0);
  }, MERGED_OBJ_FILE);

  // 다운로드를 시작한 후 병합된 파일을 MEMFS에서 삭제
  fs::remove(MERGED_OBJ_FILE);
}

EMSCRIPTEN_BINDINGS(my_module) {
  emscripten::function("loadObjFile", &LoadObjFiles);
  emscripten::register_optional<std::string>();
  emscripten::function("mergeAndDownloadObjFiles", &MergeAndDownloadObjFiles);
}
```

- `ObjFile`
  - OBJ 파일을 담기 위한 구조체
  - 현재 파일은 material(`.mtl`)을 읽고 있지 않아 material은 포함하지 않음
- `PrintFiles`
  - 인자로 주어진 경로에 속한 폴더와 파일 목록을 출력
  - JS 쪽에서는 `Module.FS.readdir(path)`로 같은 동작을 수행할 수 있음
- `LoadObjFiles`
  - 리턴값 `std::optional<std::string>`: `register_optional` 헬퍼 함수를 이용하여 바인딩. `std::nullopt`를 리턴하면 JS 쪽에서는 `undefined`를 리턴 받음
  - 함수의 시작과 끝에서 `PrintFiles("/")`를 호출하여 MEMFS에 생성된 파일 목록을 확인
  - 여러 OBJ 파일을 읽어 `g_ObjFiles`에 데이터를 저장하고 요약 정보(Vertex, Face 개수 등)는 `summary`에 저장하여 JSON 문자열로 리턴
  - 파일 이름 배열과 요약 정보를 리턴하기 위해 JSON 문자열로 변환하여 전달하는 방법을 활용 ([JSON 문자열 이용하기](/posts/emscripten-exchange-array/#json-문자열-이용하기))
- `MergeAndDownloadObjFiles`
  - `g_ObjFiles`에 저장된 여러 개의 object를 하나의 파일로 합친 후 다운로드 하는 함수
  - Vertex 좌표, normal, texture 좌표, face 정보 순으로 작성
  - Face 정보를 작성할 때는 vertex ID에 오프셋을 적용하여 정확한 vertex를 가리키게 함
  - `EM_ASM` 부분: DOM과 JavaScript를 이용하여 파일을 다운로드
  - 지금은 다운로드 로직을 C++ 코드 안의 EM_ASM에 넣었지만, MergeAndDownloadObjFiles가 다운로드 대신 병합된 파일의 경로(MERGED_OBJ_FILE)만 리턴하도록 바꿀 수도 있음. 이 경우 JavaScript 쪽에서 그 경로로 Module.FS.readFile을 호출해 파일을 읽고 Blob을 만들어 다운로드하는 코드를 실행하면 동일한 결과를 얻을 수 있음

### 소스 코드 - index.html

```HTML
<!doctype html>
<html>
  <head>
    <title>Emscripten Example-14</title>
  </head>
  <body>
    <!-- 여러 개의 OBJ 파일 선택 -->
    <div>
      <label for="fileInput">Select OBJ Files</label>
      <input type="file" accept=".obj" id="fileInput" disabled multiple />
    </div>
    <!-- OBJ 파일 요약 정보 표시 -->
    <h3 id="shape-info" style="white-space: pre-wrap"></h3>
    <!-- OBJ 파일 병합 및 다운로드 버튼 -->
    <button
      id="download-obj"
      onclick="Module.mergeAndDownloadObjFiles()"
      disabled
    >
      Merge and Download OBJ
    </button>
    <!-- OBJ 파일을 여는 버튼을 Wasm 모듈 초기화 후에 활성화 -->
    <script>
      Module = {
        onRuntimeInitialized: () => {
          document.getElementById("fileInput").disabled = false;
        },
      };
    </script>
    <script async type="text/javascript" src="memfs.js"></script>
    <script>
      const fileInput = document.getElementById("fileInput");
      fileInput.addEventListener("change", async (event) => {
        let filenames = [];
        for (const file of event.target.files) {
          const buffer = await file.arrayBuffer();
          const typedArray = new Uint8Array(buffer);
          // MEMFS에 파일 쓰기
          Module.FS.writeFile(file.name, typedArray);
          // 파일 이름을 배열에 추가
          filenames.push(file.name);
        }
        // MEMFS에 저장된 OBJ 파일을 로드하고 요약 정보 가져오기
        const summary = Module.loadObjFile(JSON.stringify(filenames));
        if (summary === undefined) {
          console.error("Failed to load OBJ files.");
          return;
        }
        // 요약 정보를 JSON 객체로 변환하여 화면에 표시
        const shapes = JSON.parse(summary);
        const shapeInfoDiv = document.getElementById("shape-info");
        shapeInfoDiv.textContent = JSON.stringify(shapes, null, 2);
        // 파일을 다 읽은 후 다운로드 버튼 활성화
        document.getElementById("download-obj").disabled = false;
      });
    </script>
  </body>
</html>
```

- 브라우저에 파일 Dialog를 열어 여러 개의 `.obj` 파일을 전달
- 전달 받은 OBJ 파일들을 이용하여 MEMFS에 파일을 씀. 이 과정에서 데이터 복사가 일어남
- 파일들을 읽어 요약 정보를 화면에 표시. 표시가 완료되면 다운로드 버튼 활성화
- 다운로드 버튼을 클릭하여 `Module.mergeAndDownloadObjFiles` 함수를 호출

### 빌드 하기

```bash
em++ memfs.cpp -o memfs.js -lembind -s EXPORTED_RUNTIME_METHODS="['FS']"
```

- JS 쪽에서 파일 시스템에 접근하기 위해 `EXPORTED_RUNTIME_METHODS`에 `FS`를 추가해야 함
- C++ 코드에서 파일 시스템을 전혀 사용하지 않으면, Emscripten이 빌드 최적화 과정에서 파일 시스템 관련 코드를 제거할 수 있음. 이것을 막기 위해서는 빌드 옵션 `-s FORCE_FILESYSTEM`을 추가해야 JS 쪽에서 `Module.FS.writeFile` 같은 함수를 이용하여 파일에 접근할 수 있음. 현재 예제에서는 C++ 코드에서도 파일 시스템에 접근하고 있기 때문에 빌드 옵션을 추가하지 않아도 파일 시스템이 삭제 되지 않음

### 실행 결과

![load two obj files](images/load-two-objs.png)
_그림 2. Cube.obj 파일과 Isosphere.obj 파일을 읽은 후 요약 정보를 화면에 잘 보여주고 있다._

- 두 파일(`Cube.obj`, `Isosphere.obj`)을 읽은 후 아래의 Merge and Download OBJ를 클릭하여 `merged.obj` 파일을 다운로드 할 수 있음
- `LoadObjFiles` 함수의 시작 부분(`=== Before loading OBJ files ===`)에서 `Root('/')` 폴더를 확인하면 두 파일(`Cube.obj`, `Isosphere.obj`)이 존재하는 것을 확인 할 수 있음
- `LoadObjFiles` 함수의 마지막 부분(`=== After loading OBJ files ===`)에서는 두 파일이 삭제 되어 MEMFS의 기본 폴더들(`/tmp`, `/home`, `/dev`, `/proc`)만 있는 것을 확인 할 수 있음

### Blender를 이용한 OBJ 파일 확인

![Cube](images/Blender_Cube.png)
_그림 3. Cube.obj 파일을 Blender에서 렌더링 한 모습_

![Isosphere](images/Blender_Isosphere.png)
_그림 4. Isosphere.obj 파일을 Blender에서 렌더링 한 모습_

![merged](images/Blender_merged.png)
_그림 5. 두 파일(Cube.obj, Isosphere.obj)을 병합한 merged.obj 파일을 Blender에서 렌더링 한 모습_

### 예제 코드 및 emsdk 버전

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-14
- emsdk 5.0.3

## FAQ

- OBJ 파일을 다 읽자마자 MEMFS에서 지우는 이유는 무엇인가요?
  - MEMFS는 모든 파일이 메모리에 존재하기 때문에, 사용이 끝난 파일을 계속 남겨두면 브라우저 탭이 열려 있는 동안 그만큼의 메모리를 계속 점유하게 됩니다. 특히 사용자가 큰 OBJ 파일을 여러 번 반복해서 업로드하는 경우라면 차이가 더 커집니다.
  - `LoadObjFiles`에서는 `tinyobj::LoadObj` 호출 직후, 성공/실패 여부와 관계없이 바로 `fs::remove`를 호출합니다. 파싱에 실패한 파일까지 지우는 이유는, 실패한 파일을 MEMFS에 남겨두어 봐야 재사용할 수 없고 이후 같은 이름으로 다시 업로드할 때 방해만 되기 때문입니다.
  - 다운로드용으로 새로 만든 `/merged.obj` 역시 같은 이유로 `EM_ASM`에서 다운로드를 트리거한 직후 `fs::remove`로 정리합니다.

- 네이티브에서 파일을 읽을 때와 비교하면, MEMFS에 파일을 쓰는 과정에서 데이터 복사가 한 번 더 필요한 게 맞나요?
  - 네, 맞습니다. 브라우저는 보안상 JS가 파일 원본에 직접 접근하지 못하게 하고, MEMFS는 Wasm 선형 메모리와는 분리된 별도의 JS 쪽 저장소이기 때문에, 그 사이를 잇기 위한 복사가 하나 더 필요합니다.

    | 단계                                                | 네이티브 (`std::ifstream`)            | 브라우저 + Emscripten MEMFS                                                                       |
    | --------------------------------------------------- | ------------------------------------- | ------------------------------------------------------------------------------------------------- |
    | 1                                                   | 디스크 → 커널 페이지 캐시             | 디스크 → 브라우저가 읽어 JS `ArrayBuffer`에 저장 (`file.arrayBuffer()`)                           |
    | 2 (Emscripten 전용 추가 복사)                       | —                                     | JS 힙(`ArrayBuffer`) → MEMFS 내부 저장소로 복사 (`Module.FS.writeFile`)                           |
    | 3                                                   | 커널 버퍼 → 앱 버퍼로 복사 (`read()`) | MEMFS 저장소 → Wasm 선형 메모리로 복사 (`tinyobj::LoadObj`가 내부적으로 사용하는 `std::ifstream`) |
    | 앱 코드가 데이터를 실제로 읽기까지 필요한 복사 횟수 | 1회                                   | 2회                                                                                               |

  - 1단계(디스크 → 임시 저장)와 3단계(임시 저장 → 앱이 실제로 읽는 버퍼)는 네이티브와 브라우저 양쪽에 똑같이 존재하는, 어차피 필요한 복사입니다. 반면 2단계(`Module.FS.writeFile`)는 MEMFS라는 별도의 가상 파일시스템에 데이터를 옮겨 담기 위해서만 필요한 복사로, 네이티브에는 대응되는 단계가 없습니다.

- OBJ와 함께 있는 `.mtl`(material) 파일도 읽게 하려면 어떻게 해야 하나요?
  - `tinyobj::LoadObj`는 `.obj` 파일뿐 아니라 그 안에서 참조하는 `.mtl` 파일도 함께 읽으려고 시도합니다. 기본적으로는 `.obj`와 같은 디렉토리에서 `.mtl` 파일을 찾고, 마지막 인자(`mtl_search_path`)로 별도의 검색 경로를 지정할 수도 있습니다.
  - 즉, MEMFS 위에서 이 기능을 쓰려면 `.obj` 파일만이 아니라 관련된 `.mtl` 파일도 미리 `Module.FS.writeFile`로 같은 경로에 써 두어야 합니다. 지금 `index.html`은 사용자가 선택한 파일을 모두 그대로 쓰기만 하므로, `.mtl`을 함께 선택하도록 안내하거나 `input`의 `accept` 속성을 `.obj,.mtl`로 넓혀야 합니다.
  - 현재 코드는 `LoadObj`의 네 번째 인자로 `materials`를 넘기고 있지만, 읽어들인 내용을 `ObjFile`에 저장하지 않고 그냥 버리고 있습니다. 실제로 material 정보를 활용하려면 `ObjFile`에 `std::vector<tinyobj::material_t> materials` 필드를 추가하고, 병합 시 `mtllib`/`usemtl` 지시자도 함께 기록해야 합니다.

- 빌드할 때 `-s EXPORTED_RUNTIME_METHODS="['FS']"`를 빠뜨리면 어떻게 되나요?
  - Emscripten은 기본적으로 `Module` 객체에 노출하는 심볼을 최소화하기 때문에, `FS` 네임스페이스를 명시적으로 export하지 않으면 JS 쪽에서 `Module.FS`가 `undefined`가 됩니다.
  - 결과적으로 `index.html`의 `Module.FS.writeFile(...)` 호출에서 `Cannot read properties of undefined (reading 'writeFile')` 같은 런타임 에러가 발생합니다.
  - 반면 C++ 쪽에서 파일 시스템에 접근하는 코드(`tinyobj::LoadObj`, `std::ofstream` 등)는 이 옵션과 무관하게 그대로 동작합니다. 문제는 JS에서 사용자 파일을 MEMFS로 전달할 방법이 없어진다는 점입니다.

- 같은 이름의 파일을 두 번 선택하면 어떻게 되나요? (예: 서로 다른 폴더에 있는 두 개의 `Cube.obj`)
  - `Module.FS.writeFile(file.name, typedArray)`는 파일명만을 MEMFS 경로로 사용하므로, 이름이 같은 파일을 여러 개 선택하면 나중에 쓴 파일이 먼저 쓴 파일을 덮어씁니다.
  - 이후 `filenames` 배열에는 같은 이름이 두 번 들어가지만, `LoadObjFiles`가 첫 번째 항목을 읽으면서 해당 파일을 MEMFS에서 이미 삭제했기 때문에 두 번째 항목을 읽을 때는 파일을 찾지 못해 `err`가 채워지고 건너뛰게 됩니다.
  - 이 문제를 피하려면 업로드 시점에 파일마다 고유한 접두사를 붙여 쓰거나(예: `${index}_${file.name}`), 이름이 겹치는 파일이 있을 때 사용자에게 알려주는 처리가 필요합니다.

- 다운로드 로직을 C++ 안의 `EM_ASM` 대신 JS 쪽 코드로 옮기려면 어떻게 하나요?
  - `MergeAndDownloadObjFiles`가 직접 다운로드까지 처리하는 대신, 병합된 파일의 경로(`MERGED_OBJ_FILE`)만 리턴하도록 바꾸면 됩니다.

    ```C++
    std::optional<std::string> MergeAndDownloadObjFiles() {
      // ... 병합 로직은 동일 ...

      output.close();
      if (!output) {
        std::cerr << "Failed to write merged OBJ file." << std::endl;
        fs::remove(MERGED_OBJ_FILE);
        return std::nullopt;
      }

      return std::string(MERGED_OBJ_FILE);  // 다운로드 대신 경로만 리턴
    }
    ```

  - JS 쪽에서는 이 경로로 `Module.FS.readFile`을 호출해 파일을 읽고, `Blob`을 만들어 다운로드한 뒤 MEMFS에서 정리하면 동일한 결과를 얻을 수 있습니다.

    ```JavaScript
    document.getElementById("download-obj").addEventListener("click", () => {
      const path = Module.mergeAndDownloadObjFiles();
      if (path === undefined) {
        console.error("Failed to merge OBJ files.");
        return;
      }

      const data = Module.FS.readFile(path);
      const blob = new Blob([data], { type: "text/plain" });
      const url = URL.createObjectURL(blob);

      const anchor = document.createElement("a");
      anchor.href = url;
      anchor.download = "merged.obj";
      document.body.appendChild(anchor);
      anchor.click();
      anchor.remove();

      setTimeout(() => URL.revokeObjectURL(url), 0);
      Module.FS.unlink(path);  // 다운로드 후 MEMFS에서 정리
    });
    ```

  - 이렇게 바꾸면 `EM_ASM`을 쓰지 않고도 동일한 사용자 경험을 얻을 수 있고, C++ 코드는 파일 병합 로직에만 집중할 수 있습니다. 다만 `index.html`의 버튼도 `onclick="Module.mergeAndDownloadObjFiles()"` 대신 위 이벤트 리스너 방식으로 함께 바꿔야 합니다.
