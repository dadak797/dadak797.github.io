---
title: "Emscripten Intro"
date: "2026-07-25T16:14:39+09:00"
draft: false
description: ""

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
> Emscripten is a compiler that turns C/C++ code into WebAssembly (.wasm), letting you build high-performance applications that run in the browser.

## WebAssembly

- WebAssembly (Wasm) is a binary format that runs in the browser at near-native speed
- It doesn't replace JavaScript — it complements JavaScript's weaknesses
- Because Wasm is a compilation target rather than something written by hand, you need a tool to turn C/C++ into Wasm, and Emscripten fills that role (Rust, by contrast, compiles to Wasm through its own LLVM backend)

## Emscripten

- Emscripten is a compiler that turns C/C++ code into a WebAssembly binary (.wasm) plus JavaScript glue code (.js)
- At runtime a Wasm module can't touch the DOM directly, but through the glue code (.js) it can reach the browser's DOM and Web APIs
- The glue code also lets you use a Wasm module's functionality from within Node.js

![Emscripten|525](images/emscripten_toolchain_flow_en_v2.svg)
_Figure 1. Emscripten compiles C/C++ into a `.wasm` binary plus `.js` glue; the glue bridges WebAssembly to the DOM and Web APIs._

## Advantages

1. Fast
   - WebAssembly is a pre-compiled binary format, so it runs directly without the parse-and-optimize step JavaScript goes through at runtime
   - That said, the benefit depends on the workload — heavy computations see a big gap, while lightweight tasks may gain little due to boundary-call overhead
2. Reusability
   - Lets you reuse the vast body of existing C++ code
   - You can also pull in an entire pre-built Wasm module. Pyodide is CPython built with Emscripten — using it, you can run Python in the browser
3. Security
   - Provides an isolated execution model by default. A Wasm module cannot escape its linear memory
   - A Wasm module can only call functions imported from JS, and JS can only call functions the Wasm module exports
   - That said, the actual scope of this isolation depends on what the host (JS) chooses to hand over via imports
4. Offloading the server
   - Heavy work that used to run on the server can run in the browser instead (within the limits of what the user's machine can handle)

## Disadvantages

1. User file access
   - Because it runs inside the browser sandbox, it can't access the user's files directly. A file has to be loaded via an HTML `<input type="file">` element and then copied into WebAssembly memory, which makes loading more cumbersome and slower than in a native environment
   - Resources needed at load time must be placed into a virtual filesystem in a form Wasm can read ahead of time; a link option (`--preload-file`) handles this step
2. DOM accessibility
   - WebAssembly can't access the DOM directly; it must go through JavaScript using dedicated macros (`EM_JS`, `EM_ASM`, etc.)
3. Initial module load
   - The Wasm module (.wasm) and the glue code (.js) that connects it to JS both have to be downloaded, so the first load can be heavy
4. JS <span>&harr;</span> Wasm boundary-call overhead
   - Crossing the boundary frequently, or passing strings and large data, can incur significant conversion and memory-copy (marshaling) costs
5. Multithreading
   - Multithreading requires SharedArrayBuffer, which in turn forces you to set COOP/COEP headers
   - The setup itself is simple, but COEP is a page-wide policy, so cross-origin resources such as external CDNs and third-party widgets must all comply. The burden is small for self-hosted apps, but sites with many external dependencies need to check for compatibility

## When is it a good fit?

- When you want to finish processing on the client without round-tripping to the server (with a privacy benefit as well)
  - e.g. FFmpeg.wasm (in-browser media conversion), sql.js (in-browser SQL DB)
- Web apps that need large-scale, high-performance computation or low-latency real-time processing
  - e.g. Figma, Photoshop Web
- When you want to port an existing large C/C++ application to the web - e.g. AutoCAD Web, Google Earth
  ![A C++ 3D viewer running natively and in the web browser](images/NativeToWeb.png)
  _Figure 2. A C++ 3D viewer written natively (left), compiled with Emscripten and running identically in a web browser (right). The same code works unchanged in both environments._
- When you want to share the same core logic across multiple platforms (one C++ core for both a native app and the web)
- When you want to bring a mature C/C++ library to the web without rewriting it
  - e.g. OpenCASCADE (CAD kernel), VTK (scientific & medical visualization)

> [!NOTE]
> Whether these products use Emscripten/Wasm can change over time, so verify against each company's engineering material before citing them.

## References

- [Emscripten official documentation](https://emscripten.org/docs/)
- [WebAssembly official homepage](https://webassembly.org/)
