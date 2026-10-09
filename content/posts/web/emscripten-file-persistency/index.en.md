---
title: Handling Files in Emscripten 2 - Persisting Data in the Browser (Local Storage, IDBFS)
date: 2026-10-09
draft: false
description: MEMFS, one of Emscripten's virtual file systems (VFS), stores files in memory, so its data is lost when the page is reloaded. This article introduces two ways to persist data in the browser using Local Storage and IDBFS.
categories:
  - Web
tags:
  - Emscripten
  - File-Persistency
  - Local-Storage
  - IDBFS
ShowToc: true
TocOpen: false
ShowReadingTime: false
---

> [!SUMMARY]
> MEMFS, one of Emscripten's virtual file systems (VFS), stores files in memory, so its data is lost when the page is reloaded. This article introduces two ways to persist data in the browser using Local Storage and IDBFS.

## Local Storage - Small Settings Without Files

Not every piece of data needs to be stored in a file. Small settings such as the background color, volume, and theme are easier to store directly in the browser's Local Storage than in Emscripten's file system. However, because Local Storage is not part of Emscripten's VFS, C++ code needs to use `EM_ASM` or `EM_JS` to access it.

In the example from [Setting Up an OpenGL and Dear ImGui Development Environment](/en/posts/setup-opengl-imgui/), the background color and theme settings are not retained, so they return to their defaults when the page is reloaded. In the Native build, the application writes these values to a file and reads them again when it restarts. In the Web build, it instead stores and retrieves them using Local Storage because it cannot access the user's file system directly.

### Source Code

> [!NOTE]
> Only the changes from [Setting Up an OpenGL and Dear ImGui Development Environment](/en/posts/setup-opengl-imgui/) are shown below.

```CMake
# CMakeLists.txt
...

target_link_libraries(main PRIVATE imgui)

# nlohmann-json
add_library(nlohmann_json INTERFACE)

target_include_directories(nlohmann_json INTERFACE
  third_party
)

target_link_libraries(main PRIVATE nlohmann_json)
```

- Adds nlohmann-json to the project for working with JSON

```C++
#include <iostream>
#include <cassert>

#ifdef __EMSCRIPTEN__
  #include <emscripten.h>
  #include <emscripten/html5.h>
#else
  #include <fstream>
  #include <glad/gl.h>  // GLAD
  #include <nlohmann/json.hpp>
  using json = nlohmann::json;
#endif

...

// Initial background color
float g_BgColor[4] = {0.0f, 0.1f, 0.2f, 1.0f};
int32_t g_Theme = 0;

namespace {

...

const char* g_ThemeNames[] = {"Dark", "Light", "Classic"};

#ifndef __EMSCRIPTEN__
constexpr const char* SETTINGS_FILE = "settings.json";

// Read settings.json as a JSON object (empty object if missing or invalid)
json readSettingsFile() {
  std::ifstream file(SETTINGS_FILE);
  if (!file) {
    return json::object();
  }

  auto settings = json::parse(file, nullptr, false);
  if (!settings.is_object()) {
    return json::object();
  }
  return settings;
}

void writeSettingsFile(const json& settings) {
  std::ofstream file(SETTINGS_FILE);
  if (!file) {
    std::cerr << "Failed to write " << SETTINGS_FILE << std::endl;
    return;
  }
  file << settings.dump(2) << std::endl;
}
#endif

void loadTheme() {
#ifdef __EMSCRIPTEN__
  // Load the theme previously saved in Local Storage
  g_Theme = EM_ASM_INT({
    const themeIndex = localStorage.getItem('theme');
    return parseInt(themeIndex) | 0;
  });
#else
  const auto settings = readSettingsFile();
  g_Theme = settings.value("theme", 0);
#endif
}

void saveTheme() {
#ifdef __EMSCRIPTEN__
  EM_ASM({
    // Save g_Theme in Local Storage
    localStorage.setItem('theme', $0);
  }, g_Theme);
#else
  auto settings = readSettingsFile();
  settings["theme"] = g_Theme;
  writeSettingsFile(settings);
#endif
}

void loadBgColor() {
#ifdef __EMSCRIPTEN__
  EM_ASM({
    // Load bgColor from Local Storage
    const color = JSON.parse(localStorage.getItem('bgColor') || '[0.0, 0.1, 0.2, 1.0]');
    HEAPF32.set(color, $0 >> 2);
  }, g_BgColor);
#else
  const auto settings = readSettingsFile();
  if (settings.contains("bgColor")) {
    settings["bgColor"].get_to(g_BgColor);
  }
#endif
}

void saveBgColor() {
#ifdef __EMSCRIPTEN__
  EM_ASM({
    // Save g_BgColor in Local Storage
    localStorage.setItem('bgColor', JSON.stringify([$0, $1, $2, $3]));
  }, g_BgColor[0], g_BgColor[1], g_BgColor[2], g_BgColor[3]);
#else
  auto settings = readSettingsFile();
  settings["bgColor"] = g_BgColor;
  writeSettingsFile(settings);
#endif
}

bool applyTheme() {
  switch (g_Theme) {
    case 0:
      ImGui::StyleColorsDark();
      break;
    case 1:
      ImGui::StyleColorsLight();
      break;
    case 2:
      ImGui::StyleColorsClassic();
      break;
    default:
      assert(false && "Invalid theme");
      return false;
  }
  return true;
}

void renderFrame(GLFWwindow* window) {
  ...
  ImGui::Begin("Test Window");

  if (ImGui::ColorEdit4("Background Color", g_BgColor)) {
    saveBgColor();
  }

  if (ImGui::Combo("Theme", &g_Theme, g_ThemeNames, IM_ARRAYSIZE(g_ThemeNames))) {
    if (applyTheme()) {
      saveTheme();
    }
  }

  ImGui::End();
  ...
}

...

}  // namespace

int main() {
  ...
  // ImGui initialization
  IMGUI_CHECKVERSION();
  ImGui::CreateContext();
  ImGuiIO& io = ImGui::GetIO();
  io.Fonts->AddFontDefaultVector();

  loadTheme();
  applyTheme();

  ...
  glfwGetFramebufferSize(window, &framebufferWidth, &framebufferHeight);
  framebufferSizeCallback(window, framebufferWidth, framebufferHeight);

  loadBgColor();

  // Main loop
  ...
}
```

- `g_BgColor`: Global variable that stores the background color
- `g_Theme`: Global variable that stores the ImGui theme
- `readSettingsFile`: Reads the background color and theme from `settings.json` in the Native build
- `writeSettingsFile`: Writes the background color and theme to `settings.json` in the Native build
- `loadTheme`: Loads the ImGui theme
- `saveTheme`: Saves the ImGui theme
- `applyTheme`: Applies the ImGui theme
- `loadBgColor`: Loads the background color
- `saveBgColor`: Saves the background color

### Building and Running

#### Native - macOS

```bash
cmake -S. -Bbuild/native
cmake --build build/native
./build/native/main
```

![SaveSettings_Native](images/Native_Save_Settings.png)
_Figure 1. Running in the Native environment on macOS. The background color and ImGui theme persist after the application restarts. Changing either setting creates `settings.json`. The ImGui window's size and position also persist after restarting._

#### Web

```bash
emcmake cmake -S. -Bbuild/web
cmake --build build/web
python -m http.server 8080
```

![SaveSettings_Web](images/Web_Save_Settings.png)
_Figure 2. Running in the Web environment. The background color and ImGui theme persist after the application restarts. Changing either setting creates `theme` and `bgColor` entries in Local Storage, as shown in the developer tools. However, the ImGui window's size and position do not persist after restarting._

### Example Code and Test Environment

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-19
- macOS Sonoma 14.6.1
  - Compiler: AppleClang 16.0.0.16000026
- Web
  - Compiler: emsdk 5.0.3

## IDBFS - Persisting Files

The previous example showed how to persist the background color and theme. However, when the application restarts, the ImGui window's size and position persist in the Native build but return to their defaults in the Web build. This is because the Native build automatically stores these settings in a file named `imgui.ini`. The Web build therefore needs a way to save and restore this automatically generated file. This section uses IDBFS to store `imgui.ini` in the browser's IndexedDB and load it again later.

### Mounting IDBFS

```JavaScript
FS.mkdir('/data');
FS.mount(IDBFS, {}, '/data');
```

- To persist MEMFS files in IndexedDB, mount an IDBFS instance at an existing VFS directory
- After creating `/data`, mount IDBFS at that path. Files under `/data` are read from and written to the in-memory VFS, then synchronized asynchronously with IndexedDB by calling `FS.syncfs()` as described below
- [IDBFS](https://emscripten.org/docs/api_reference/Filesystem-API.html#filesystem-api-idbfs)

### Synchronizing Files Between MEMFS and IDBFS

```
FS.syncfs(populate, callback);
```

- Emscripten provides an interface for synchronizing MEMFS with the browser's IndexedDB
- `populate`
  - `false`: Synchronizes files from MEMFS to IndexedDB
  - `true`: Synchronizes files from IndexedDB to MEMFS
- `callback`
  - Called after synchronization completes
- [FS.syncfs](https://emscripten.org/docs/api_reference/Filesystem-API.html#FS.syncfs)

### Source Code

> [!NOTE]
> Only the changes from the [Local Storage example](/en/posts/emscripten-file-persistency/#local-storage---small-settings-without-files) are shown below.

```CMake
# CMakeLists.txt
...

target_link_options(main PRIVATE
  "-sUSE_GLFW=3"
  "-sMIN_WEBGL_VERSION=2"  # Set the minimum WebGL version to 2
  "-sMAX_WEBGL_VERSION=2"  # Set the maximum WebGL version to 2
  "-lidbfs.js"             # Use IDBFS to persist imgui.ini
)

...
```

- `-lidbfs.js`: Links the library required to use IDBFS

```C++
// main.cpp
#include <iostream>
#include <cassert>
#include <fstream>  // Moved out of the Native-only block
#include <string>

...

#ifdef __EMSCRIPTEN__
...

constexpr const char* IMGUI_INI_PATH = "/settings/imgui.ini";
bool g_IniLoaded = false;

// Mount IDBFS and load imgui.ini once the browser storage is synced
void loadImGuiIni() {
  EM_ASM({
    FS.mkdir('/settings');
    FS.mount(IDBFS, {}, '/settings');  // Mount IDBFS in the browser at /settings
    // true: synchronize from the browser storage to the in-memory filesystem
    FS.syncfs(true, (err) => {
      if (err) console.error(err);
      _onImGuiIniSynced();
    });
  });
}

// Write imgui.ini to IDBFS and flush it to the browser storage
void saveImGuiIni() {
  size_t dataSize = 0;
  const char* data = ImGui::SaveIniSettingsToMemory(&dataSize);

  std::ofstream file(IMGUI_INI_PATH, std::ios::binary);
  file.write(data, dataSize);
  file.close();

  EM_ASM({
    // false: synchronize from the in-memory filesystem to the browser storage
    FS.syncfs(false, (err) => {
      if (err) console.error(err);
    });
  });
  ImGui::GetIO().WantSaveIniSettings = false;
}

// Called by emscripten_set_main_loop_arg
void browserMainLoop(void* argument) {
  ...
  // Wait until imgui.ini is loaded from IDBFS
  if (!g_IniLoaded) {
    return;
  }

  renderFrame(window);

  // Save the ini settings, when WantSaveIniSettings becomes true
  if (ImGui::GetIO().WantSaveIniSettings) {
    saveImGuiIni();
  }
}
#endif

}  // namespace

#ifdef __EMSCRIPTEN__
// Called from JavaScript when FS.syncfs(true) finishes
extern "C" EMSCRIPTEN_KEEPALIVE void onImGuiIniSynced() {
  std::ifstream file(IMGUI_INI_PATH, std::ios::binary);
  if (file) {
    std::string fileContents((std::istreambuf_iterator<char>(file)),
                             std::istreambuf_iterator<char>());
    ImGui::LoadIniSettingsFromMemory(fileContents.c_str(), fileContents.size());
  }
  g_IniLoaded = true;
}
#endif

int main() {
  ...
  ImGui::CreateContext();
  ImGuiIO& io = ImGui::GetIO();
  io.Fonts->AddFontDefaultVector();
#ifdef __EMSCRIPTEN__
  io.IniFilename = nullptr;  // Save imgui.ini manually to IDBFS
  loadImGuiIni();
#endif
  ...
}
```

- `loadImGuiIni`: Loads `imgui.ini` from IndexedDB during startup and applies the stored ImGui window size and position
- `saveImGuiIni`: Saves `imgui.ini` from MEMFS to IndexedDB when `ImGui::GetIO().WantSaveIniSettings` becomes `true` in `browserMainLoop`
- `onImGuiIniSynced`
  - Called by the callback in `loadImGuiIni`
  - Exposed to JavaScript with `extern "C" EMSCRIPTEN_KEEPALIVE`
  - Called from JavaScript as `_onImGuiIniSynced`, with an underscore (`_`) prefix

### Building and Running

```bash
emcmake cmake -S. -Bbuild/web
cmake --build build/web
python -m http.server 8080
```

![Save_Ini_Web](images/Web_Save_Ini.png)
_Figure 3. Running in the Web environment. The ImGui window's size and position persist after the application restarts. The `/settings/imgui.ini` entry is visible in IndexedDB through the developer tools._

### Example Code and Test Environment

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-20
- Web
  - Compiler: emsdk 5.0.3

## FAQ

{{< faq summary="Couldn't I store the data in a server-side database and load it from there?" >}}

Yes. However, this article focuses on approaches that work entirely in the frontend without requiring a backend.

{{< /faq >}}

{{< faq summary="If Local Storage is the simplest option, why would I use anything else?" >}}

Local Storage is simple, but it only supports key-value storage, and both keys and values must be strings. JSON data therefore needs to be serialized and deserialized, and keys must be designed carefully to avoid collisions when storing multiple fields.

Its capacity is also generally limited to around 5 MB, so it is best suited to small amounts of data. Because it operates synchronously, reading or writing large values can also block the main thread.

{{< /faq >}}

{{< faq summary="Why does the state still reset when I reload the page after enabling IDBFS?" >}}

The default value of `io.IniSavingRate` is five seconds, so `WantSaveIniSettings` may not have been set immediately after a change. IndexedDB is updated only after the callback for `FS.syncfs(false)` runs. If you close the tab before that callback, the settings may not be saved. If necessary, you can reduce `io.IniSavingRate` to one second.

{{< /faq >}}

{{< faq summary="Is data stored in IDBFS guaranteed to persist permanently?" >}}

No. It can be deleted when the user clears site data, ends a private browsing session, or when the browser runs out of storage space.

{{< /faq >}}

{{< faq summary="Why did my saved data disappear after I changed the deployment URL?" >}}

Browser storage is separated by `origin`. A change to the protocol, host, or port creates a different storage area.

{{< /faq >}}

{{< faq summary="Does writing a file automatically save it from IDBFS to IndexedDB?" >}}

By default, you need to call `FS.syncfs(false)`. You can instead use the `autoPersist: true` mount option to persist changes automatically.

{{< /faq >}}

## References

- [Emscripten File System API](https://emscripten.org/docs/api_reference/Filesystem-API.html)
