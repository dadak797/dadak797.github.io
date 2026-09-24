---
title: "Setting Up a 3D Graphics Development Environment for Native and Web - OpenGL & ImGui"
date: "2026-09-24T20:25:44+09:00"
draft: false
description: Set up a 3D graphics application development environment that works in both Native and Web environments. The Native build uses OpenGL 3.3 Core, while the Web build uses Emscripten to map the OpenGL ES 3.0 API to WebGL 2. Combine GLFW, GLAD, and Dear ImGui to build the same C++ source code for both platforms.
categories:
  - Graphics
tags:
  - OpenGL
  - GLFW
  - GLAD
  - Dear ImGui
  - cross-platform
ShowToc: true
TocOpen: false
ShowReadingTime: false
---

> [!SUMMARY]
> Set up a 3D graphics application development environment that works in both Native and Web environments. The Native build uses OpenGL 3.3 Core, while the Web build uses Emscripten to map the OpenGL ES 3.0 API to WebGL 2. Combine GLFW, GLAD, and Dear ImGui to build the same C++ source code for both platforms.

## Development Environment Setup

### Prerequisites

- CMake 3.20 or later
- C++ compiler
  - Windows: MSVC (Visual Studio)
  - Linux: GCC
  - macOS: AppleClang
  - Web: [emsdk](/en/posts/emscripten-install/)

> [!NOTE]
> When GLFW 3.5.1 is built through `FetchContent` as in this guide, both the X11 and Wayland backends are enabled by default on Linux. Install the following development packages on Ubuntu to build both backends. If you need only one backend, disable the other with the `GLFW_BUILD_X11` or `GLFW_BUILD_WAYLAND` CMake option and omit its dependencies.
>
> ```bash
> sudo apt update
> sudo apt install libwayland-dev libxkbcommon-dev xorg-dev
> ```

### Dependencies

#### GLFW

- A cross-platform library for creating windows and OpenGL/OpenGL ES contexts, and for handling keyboard and mouse input and events
- Can also create window surfaces for Vulkan
- Native build: v3.5.1
- https://github.com/glfw/glfw

> [!NOTE]
> The Web build does not compile the Native GLFW 3.5.1 library. It uses the GLFW 3-compatible implementation included with Emscripten through the `-sUSE_GLFW=3` link option.

#### GLAD 2

- A loader that finds and connects OpenGL driver function pointers at runtime
- The [GLAD code generator](https://gen.glad.sh/) generates version-specific GLAD code after you select a generator and API
  - Generator: C/C++
  - gl: Version 3.3, Core
- The generated code has the following structure

```
glad/
├── include/
│   ├── glad/
│   │   └── gl.h
│   └── KHR/
│       └── khrplatform.h
└── src/
    └── gl.c
```

> [!NOTE]
> Emscripten provides an implementation that maps OpenGL ES functions to WebGL, so the Web build does not use GLAD.

#### Dear ImGui

- An immediate mode GUI library for easily implementing buttons, menus, windows, and other interfaces in graphics applications
- Like GLFW, it can be used with graphics APIs other than OpenGL
- v1.92.9b
- https://github.com/ocornut/imgui
- Add the following files to the project. If you use a different graphics API, select the appropriate implementation and header from the `backends` directory instead of the OpenGL 3 backend

```
backends/imgui_impl_glfw.cpp
backends/imgui_impl_glfw.h
backends/imgui_impl_opengl3_loader.h
backends/imgui_impl_opengl3.cpp
backends/imgui_impl_opengl3.h
imconfig.h
imgui_demo.cpp
imgui_draw.cpp
imgui_internal.h
imgui_tables.cpp
imgui_widgets.cpp
imgui.cpp
imgui.h
imstb_rectpack.h
imstb_textedit.h
imstb_truetype.h
LICENSE.txt
```

### Source Code

```CMake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)

project(ex-18 LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_executable(main main.cpp)

if(EMSCRIPTEN)
  target_compile_definitions(main PRIVATE
    GLFW_INCLUDE_ES3
  )

  # Use the GLFW implementation included with Emscripten
  target_link_options(main PRIVATE
    "-sUSE_GLFW=3"
    "-sMIN_WEBGL_VERSION=2"  # Set the minimum WebGL version to 2
    "-sMAX_WEBGL_VERSION=2"  # Set the maximum WebGL version to 2
  )

  # Same as specifying "-o main.js"
  set_target_properties(main PROPERTIES SUFFIX ".js")
else()
  target_compile_definitions(main PRIVATE
    GLFW_INCLUDE_NONE
  )

  include(FetchContent)

  # Disable building GLFW documentation, tests, and examples
  set(GLFW_BUILD_DOCS OFF CACHE BOOL "" FORCE)
  set(GLFW_BUILD_TESTS OFF CACHE BOOL "" FORCE)
  set(GLFW_BUILD_EXAMPLES OFF CACHE BOOL "" FORCE)
  set(GLFW_INSTALL OFF CACHE BOOL "" FORCE)

  FetchContent_Declare(
    glfw
    GIT_REPOSITORY https://github.com/glfw/glfw.git
    GIT_TAG        3.5.1
    GIT_SHALLOW    TRUE
  )
  FetchContent_MakeAvailable(glfw)

  # GLAD library for loading OpenGL functions
  add_library(glad STATIC
    third_party/glad/src/gl.c
  )

  target_include_directories(glad PUBLIC
    third_party/glad/include
  )

  target_link_libraries(main PRIVATE glad)
endif()

# IMGUI library for graphical user interfaces(GUI)
add_library(imgui STATIC
  third_party/imgui/imgui_demo.cpp
  third_party/imgui/imgui_draw.cpp
  third_party/imgui/backends/imgui_impl_glfw.cpp
  third_party/imgui/backends/imgui_impl_opengl3.cpp
  third_party/imgui/imgui_tables.cpp
  third_party/imgui/imgui_widgets.cpp
  third_party/imgui/imgui.cpp
)

target_include_directories(imgui PUBLIC
  third_party/imgui
  third_party/imgui/backends
)

if(EMSCRIPTEN)
  target_compile_definitions(imgui PUBLIC
    IMGUI_IMPL_OPENGL_ES3
  )
else()
  target_link_libraries(imgui PUBLIC glfw)
endif()

target_link_libraries(main PRIVATE imgui)
```

- `GLFW_INCLUDE_ES3`: makes GLFW include the OpenGL ES 3 headers instead of the Desktop OpenGL headers
- `-sUSE_GLFW=3`: links Emscripten's GLFW 3-compatible implementation
- `-sMIN_WEBGL_VERSION=2`, `-sMAX_WEBGL_VERSION=2`: restrict the output to WebGL 2
- `IMGUI_IMPL_OPENGL_ES3`: makes Dear ImGui's OpenGL renderer backend use the OpenGL ES 3/WebGL 2 path

```C++
// main.cpp
#include <iostream>

#ifdef __EMSCRIPTEN__
  #include <emscripten.h>
  #include <emscripten/html5.h>
#else
  #include <glad/gl.h>  // GLAD
#endif

// GLFW
#include <GLFW/glfw3.h>

// ImGui
#include <imgui_impl_glfw.h>
#include <imgui_impl_opengl3.h>

// Initial background color
float g_BgColor[4] = {0.0f, 0.1f, 0.2f, 1.0f};

namespace {

#ifdef _WIN32
// Adjust the GUI scale to match the Windows display scale
void applyImGuiScale(float scale) {
  if (scale <= 0.0f) {
    scale = 1.0f;
  }

  ImGuiStyle style;
  ImGui::StyleColorsDark(&style);
  style.ScaleAllSizes(scale);
  style.FontScaleDpi = scale;
  ImGui::GetStyle() = style;
}

void windowContentScaleCallback(GLFWwindow*, float xScale, float yScale) {
  applyImGuiScale(xScale > 0.0f ? xScale : yScale);
}
#endif

// Called every frame to render the application
void renderFrame(GLFWwindow* window) {
  glfwPollEvents();

  // Render ImGui frame
  ImGui_ImplOpenGL3_NewFrame();
  ImGui_ImplGlfw_NewFrame();
  ImGui::NewFrame();

  ImGui::Begin("Test Window");
  ImGui::ColorEdit4("Background Color", g_BgColor);
  ImGui::End();

  ImGui::ShowDemoWindow();

  glClearColor(g_BgColor[0], g_BgColor[1], g_BgColor[2], g_BgColor[3]);
  glClear(GL_COLOR_BUFFER_BIT);

  ImGui::Render();
  ImGui_ImplOpenGL3_RenderDrawData(ImGui::GetDrawData());

  glfwSwapBuffers(window);
}

void framebufferSizeCallback(GLFWwindow*, int width, int height) {
  glViewport(0, 0, width, height);
}

void shutdownGlfw(GLFWwindow* window) {
  if (window) {
    glfwDestroyWindow(window);
  }
  glfwTerminate();
}

void shutdownApplication(GLFWwindow* window) {
#ifdef __EMSCRIPTEN__
  emscripten_set_resize_callback(
    EMSCRIPTEN_EVENT_TARGET_WINDOW, nullptr, false, nullptr);
  emscripten_set_fullscreenchange_callback(
    EMSCRIPTEN_EVENT_TARGET_DOCUMENT, nullptr, false, nullptr);
  emscripten_set_wheel_callback(
    "#canvas", nullptr, false, nullptr);
#endif

  ImGui_ImplOpenGL3_Shutdown();
  ImGui_ImplGlfw_Shutdown();
  ImGui::DestroyContext();

  shutdownGlfw(window);
}

#ifdef __EMSCRIPTEN__
// Called when the browser window is resized
EM_BOOL browserResizeCallback(
  int, const EmscriptenUiEvent* event, void* userData) {
  auto* window = static_cast<GLFWwindow*>(userData);

  if (event->windowInnerWidth > 0 && event->windowInnerHeight > 0) {
    glfwSetWindowSize(
      window, event->windowInnerWidth, event->windowInnerHeight);
  }

  return EM_FALSE;
}

// Called by emscripten_set_main_loop_arg
void browserMainLoop(void* argument) {
  auto* window = static_cast<GLFWwindow*>(argument);

  if (glfwWindowShouldClose(window)) {
    emscripten_cancel_main_loop();
    shutdownApplication(window);
    return;
  }

  renderFrame(window);
}
#endif

}  // namespace

int main() {
  int windowWidth = 800;
  int windowHeight = 600;

  std::cout << "Initialize GLFW" << std::endl;

  if (!glfwInit()) {
    std::cerr << "Failed to initialize GLFW" << std::endl;
    return -1;
  }

#ifdef __EMSCRIPTEN__
  glfwWindowHint(GLFW_SCALE_TO_MONITOR, GLFW_TRUE);
  glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
  glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 0);
  glfwWindowHint(GLFW_CLIENT_API, GLFW_OPENGL_ES_API);

  // Read the canvas CSS size and use it as the initial framebuffer size
  double canvasWidth;
  double canvasHeight;
  if (emscripten_get_element_css_size(
        "#canvas", &canvasWidth, &canvasHeight) == EMSCRIPTEN_RESULT_SUCCESS &&
      canvasWidth > 0.0 && canvasHeight > 0.0) {
    windowWidth = static_cast<int>(canvasWidth);
    windowHeight = static_cast<int>(canvasHeight);
  }
#else
  glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
  glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
  glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
#ifdef __APPLE__
  glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GLFW_TRUE);
#endif
#ifdef _WIN32
  glfwWindowHint(GLFW_SCALE_TO_MONITOR, GLFW_TRUE);
#endif
#endif

  std::cout << "Create GLFW window" << std::endl;

  GLFWwindow* window = glfwCreateWindow(
    windowWidth, windowHeight, "Emscripten Example-18", nullptr, nullptr);
  if (!window) {
    std::cerr << "Failed to create GLFW window" << std::endl;
    glfwTerminate();
    return -1;
  }
  glfwMakeContextCurrent(window);

  std::cout << "GLFW window created successfully" << std::endl;

#ifndef __EMSCRIPTEN__
  int version = gladLoadGL(glfwGetProcAddress);
  if (!version) {
    std::cerr << "Failed to initialize GLAD" << std::endl;
    shutdownGlfw(window);
    return -1;
  }
  std::cout << "GLAD initialized successfully, version: " << version << std::endl;
#endif

  const auto glVersion = glGetString(GL_VERSION);
  if (!glVersion) {
    std::cerr << "Failed to get OpenGL version" << std::endl;
    shutdownGlfw(window);
    return -1;
  }
  std::cout << "OpenGL version: "
            << reinterpret_cast<const char*>(glVersion) << std::endl;

  // ImGui initialization
  IMGUI_CHECKVERSION();
  ImGui::CreateContext();
  ImGuiIO& io = ImGui::GetIO();
  io.Fonts->AddFontDefaultVector();
  ImGui::StyleColorsDark();
  if (!ImGui_ImplGlfw_InitForOpenGL(window, true)) {
    std::cerr << "Failed to initialize ImGui GLFW backend" << std::endl;
    ImGui::DestroyContext();
    shutdownGlfw(window);
    return -1;
  }

#ifdef _WIN32
  applyImGuiScale(ImGui_ImplGlfw_GetContentScaleForWindow(window));
  glfwSetWindowContentScaleCallback(window, windowContentScaleCallback);
#endif

#ifdef __EMSCRIPTEN__
  const char* glslVersion = "#version 300 es";
#else
  const char* glslVersion = "#version 330";
#endif
  if (!ImGui_ImplOpenGL3_Init(glslVersion)) {
    std::cerr << "Failed to initialize ImGui OpenGL backend" << std::endl;
    ImGui_ImplGlfw_Shutdown();
    ImGui::DestroyContext();
    shutdownGlfw(window);
    return -1;
  }

#ifdef __EMSCRIPTEN__
  ImGui_ImplGlfw_InstallEmscriptenCallbacks(window, "#canvas");

  // Keep GLFW's HiDPI framebuffer while matching the canvas to the browser.
  emscripten_set_resize_callback(
    EMSCRIPTEN_EVENT_TARGET_WINDOW,
    window,
    false,
    browserResizeCallback);
#endif

  glfwSetFramebufferSizeCallback(window, framebufferSizeCallback);

  int framebufferWidth;
  int framebufferHeight;
  glfwGetFramebufferSize(window, &framebufferWidth, &framebufferHeight);
  framebufferSizeCallback(window, framebufferWidth, framebufferHeight);

  // Main loop
#ifdef __EMSCRIPTEN__
  emscripten_set_main_loop_arg(browserMainLoop, window, 0, true);
#else
  while (!glfwWindowShouldClose(window)) {
    renderFrame(window);
  }

  shutdownApplication(window);
#endif

  return 0;
}
```

- `browserMainLoop`
  - In an Emscripten build, calling `renderFrame` from a `while` loop prevents Wasm from returning control to the browser event loop, which stops screen updates and input processing
  - `emscripten_set_main_loop_arg` lets the browser retain control and repeatedly invoke the callback passed to it

```HTML
<!-- index.html -->
<!doctype html>
<html>
  <head>
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>Emscripten Example-18</title>
    <style>
      html,
      body {
        width: 100%;
        height: 100%;
        margin: 0;
        overflow: hidden;
      }
      canvas {
        display: block;
        width: 100%;
        height: 100%;
      }
    </style>
  </head>
  <body>
    <canvas id="canvas" width="800" height="600"></canvas>
    <script>
      var Module = {
        canvas: document.getElementById("canvas"),
      };
    </script>
    <script async type="text/javascript" src="build/web/main.js"></script>
  </body>
</html>
```

- When loading the JavaScript output from a custom HTML file as in this example, assign the target `<canvas>` to [`Module.canvas`](/en/posts/emscripten-module/) before loading `main.js`

### Building and Running

#### Windows

```bash
# Configure and Build
cmake -S . -B build/native
cmake --build build/native --config Debug

# Run Executable
.\build\native\Debug\main.exe
```

- When the generator is omitted, CMake selects a default generator that matches the installed Visual Studio version. To select a specific version, find a generator name supported by the current CMake installation with `cmake --help` and pass it through the `-G` option
- `--config Debug`: builds the Debug configuration

![OpenGL_ImGui_Windows](images/OpenGL_ImGui_Windows.png)
_Figure 1. Running on Windows_

#### Ubuntu

```bash
# Configure and Build
cmake -S . -B build/native
cmake --build build/native

# Run Executable
./build/native/main  # Ubuntu & macOS
```

![OpenGL_ImGui_Ubuntu](images/OpenGL_ImGui_Ubuntu.png)
_Figure 2. Running on Ubuntu_

#### macOS

```bash
# Configure and Build
cmake -S . -B build/native
cmake --build build/native

# Run Executable
./build/native/main  # Ubuntu & macOS
```

![OpenGL_ImGui_macOS](images/OpenGL_ImGui_macOS.png)
_Figure 3. Running on macOS_

#### Web

```bash
# Configure and Build
emcmake cmake -S . -B build/web
cmake --build build/web

# Run Server
python -m http.server 8080
```

> [!NOTE]
> To configure a CMake project for a Wasm build, prefix the configure command with `emcmake`. The build step uses the same `cmake --build {build directory}` command as the Native build.

![OpenGL_ImGui_Web](images/OpenGL_ImGui_Web.png)
_Figure 4. Running in a browser_

### Example Code and Test Environments

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-18
- Windows 11 Pro (Parallels Desktop VM)
  - Compiler: MSVC 19.51.36260.0
  - OpenGL 3.3 Metal - 88.1
- Ubuntu 22.04 ARM64 (Parallels Desktop VM)
  - Compiler: GCC-11.4.0
  - OpenGL 4.0 Core Profile, Mesa 23.2.1
- macOS Sonoma 14.6.1
  - Compiler: AppleClang 16.0.0.16000026
  - OpenGL 4.1 Metal - 88.1
- Web
  - Compiler: emsdk 5.0.3
  - OpenGL ES 3.0 (WebGL 2.0 (OpenGL ES 3.0 Chromium))

## FAQ

{{< faq summary="Can I write the same code for the Web build as I do for Native OpenGL?" >}}

Yes. If you stay within the modern API shared by OpenGL 3.3 Core and WebGL 2, most of the C++ code can be reused as-is. This example uses small preprocessor branches only at platform-specific boundaries such as context setup, whether GLAD is needed, the GLSL version, and main-loop integration. The per-frame rendering code and application logic built around Dear ImGui run unchanged on Windows, Linux, macOS, and the Web.

The same approach applies to a custom renderer: most rendering code can be shared when it uses the common VAO, VBO, and shader-based API. WebGL 2 is based on OpenGL ES 3.0 and adds browser security and portability restrictions, so only code that depends on Desktop OpenGL-specific functions or extensions needs a conditional path. Shader logic can also be shared, with the GLSL version and minor Desktop GLSL versus GLSL ES syntax differences handled through a small preprocessing step or separate header.

{{< /faq >}}

{{< faq summary="Why do Dear ImGui window sizes and positions persist in the Native build but reset when I reload the Web page?" >}}

By default, Dear ImGui stores settings such as window sizes and positions in `imgui.ini`. The file remains on disk in a Native build, but files written to Emscripten's default [MEMFS](/en/posts/emscripten-file-handling-memfs/#memfs) filesystem exist only in memory and disappear when the page is reloaded.

To preserve the settings in a browser, mount IDBFS and synchronize it with the browser's IndexedDB using `FS.syncfs()`, or store the result of `ImGui::SaveIniSettingsToMemory()` yourself. See [Persisting Data in the Browser]() for a complete example.

{{< /faq >}}

{{< faq summary="Why doesn't the Web build use GLAD?" >}}

Native OpenGL requires a loader such as GLAD to retrieve function addresses provided by the operating system and driver at runtime. In a Web build, Emscripten provides the headers and implementation that map OpenGL ES function calls to WebGL, so a separate OpenGL function loader is not initialized.

Dear ImGui's `imgui_impl_opengl3.cpp` also handles the functions its backend needs, but it does not load every Native OpenGL function used by the application. The Native build therefore still needs GLAD.

{{< /faq >}}

{{< faq summary="Why can't the Web build use the same `while` rendering loop as the Native build?" >}}

An endless `while` loop on the browser's main thread prevents control from returning to the JavaScript event loop, so the browser cannot update the screen or process input. Passing an FPS value of `0` to `emscripten_set_main_loop_arg()` makes Emscripten repeatedly invoke the rendering callback through `requestAnimationFrame()`, allowing the browser to process events between frames.

This example sets `simulate_infinite_loop` to `true`, so code after the function call does not run. The Web shutdown path therefore calls `emscripten_cancel_main_loop()` and performs cleanup from the main loop callback.

{{< /faq >}}

{{< faq summary="Can I continue using OpenGL on macOS?" >}}

OpenGL can still be used for learning or for checking the portability of an existing codebase as in this example, but Apple has deprecated OpenGL since macOS 10.14. macOS also provides no newer Desktop OpenGL version than 4.1 Core Profile. For a new macOS-specific product intended for long-term development, consider Metal or a renderer that supports Metal.

GLFW 3.4 and later automatically return a forward-compatible context on macOS. This example still specifies the `GLFW_OPENGL_FORWARD_COMPAT` hint to make the intent clear and remain compatible with older GLFW versions.

{{< /faq >}}

## References

- [Compiling GLFW](https://www.glfw.org/docs/latest/compile.html)
- [GLAD](https://github.com/Dav1dde/glad)
- [Dear ImGui - Getting Started](https://github.com/ocornut/imgui/wiki/Getting-Started)
- [OpenGL support in Emscripten](https://emscripten.org/docs/porting/multimedia_and_graphics/OpenGL-support.html)
- [Emscripten main loop API](https://emscripten.org/docs/api_reference/emscripten.h.html#c.emscripten_set_main_loop_arg)
- [Emscripten File System Overview](https://emscripten.org/docs/porting/files/file_systems_overview.html)
- [OpenGL on macOS](https://www.glfw.org/docs/latest/compat_guide.html#compat_osx)
