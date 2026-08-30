---
title: Emscripten Module
date: 2026-08-21
draft: false
description: An introduction to the Emscripten Module.
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
> Explains the role of the Emscripten Module, how to configure its initialization, and how to use it.

## What Is the Emscripten Module?

The Emscripten Module is a global JavaScript object used to configure and control the behavior of the JavaScript glue code that Emscripten generates from outside of it. Its main roles are:

1. Setting properties on the Module to control runtime behavior
2. Providing access to exported functions and runtime methods
3. Providing access to heap memory views

![Emscripten Module Diagram](images/Module_Diagram.png)
_Figure 1. Emscripten Module Diagram_

## Creating the Module

If you set the output file option (`-o`) to `{output_file_name}.js` or `{output_file_name}.html` among the WebAssembly build options ([Emscripten Installation - Building the Example](/en/posts/emscripten-install/#building)), JavaScript glue code is generated alongside the Wasm, and the Module is automatically created inside that glue code. At this point, functions and classes exported via [Embind](/en/posts/embind/), functions exported via [`EXPORTED_FUNCTIONS`](/en/posts/call-cpp-from-js/#exporting-c-functions), and helper functions exported via [`EXPORTED_RUNTIME_METHODS`](/en/posts/call-cpp-from-js/#passing-strings-from-javascript-to-c) are all registered on the Module. You can print out the Module's properties and function list that Emscripten generates automatically to check them directly.

![Module Properties](images/Module_Properties.png)
_Figure 2. The Module's properties and function list_

> [!NOTE]
> If you build with the output file option (`-o`) set to `{output_file_name}.wasm`, only the Wasm file is generated and no Module object is created automatically. This gives you more direct control and may allow for more optimization, but writing your own JS glue code that matches your Emscripten version is very difficult, so it isn't recommended.

### Source Code

```C++
// module_a.cpp
#include <emscripten/bind.h>
#include <string>
#include <iostream>

void hello() {
  std::cout << "Hello, WebAssembly! (From C++)" << std::endl;
}

EMSCRIPTEN_BINDINGS(moduleA) {
  emscripten::function("js_hello", &hello);
}

// module_b.cpp
#include <emscripten/bind.h>
#include <string>
#include <iostream>

void hello_with_name(const std::string& name) {
  std::cout << "Hello, " << name << "! (From C++)" << std::endl;
}

EMSCRIPTEN_BINDINGS(moduleB) {
  emscripten::function("js_hello_with_name", &hello_with_name);
}

// index.html
<!doctype html>
<html>
  <head>
    <title>Emscripten Example-9</title>
  </head>
  <body>
    <script async type="text/javascript" src="module_ab.js"></script>
  </body>
</html>
```

### Module Creation Options

```bash
em++ module_a.cpp module_b.cpp -o module_ab.js -lembind {Module_creation_option}
```

- Without adding any option (Non-MODULARIZE)
  - A Module object is created under the default name (`Module`)
  - Once the module's initialization completes, you can use the exported functions right away: `Module.js_hello();`
  - If you use `Module` before initialization completes, you get a `Module is not defined` error
  - To use `Module` reliably after initialization, you need to set the function you want to call through the `onRuntimeInitialized` callback, one of Module's properties. If you're calling the function directly after giving it enough time to initialize, you don't need to set this up via `onRuntimeInitialized`

```HTML
<script>
  var Module = {
    onRuntimeInitialized: function () {
      Module.js_hello();
    },
  };
</script>
<script async type="text/javascript" src="module_a.js"></script>
```

- `-s MODULARIZE`
  - Exports a factory function under the default name (`Module`)
  - You can use this factory function to create the module
  - `const myModule = await Module(); myModule.js_hello();`
  - Here, `Module` is the name of the factory function exported globally, and `myModule` is just a variable holding the instance you get by calling that factory — it doesn't need to be global or use any fixed name, you can name it whatever you like
- `-s EXPORT_NAME="createModule"`
  - Used together with `-s MODULARIZE` to set the name of the exported factory function. Using this option on its own has no meaningful effect
  - `const myModule = await createModule(); myModule.js_hello();`
  - When calling the factory function, you can pass in predefined Module properties as an argument
  - Note that inside `onRuntimeInitialized`, you should call the function as `this.js_hello();` rather than `Module.js_hello();`
  - Once initialization by `createModule` completes, `js_hello` gets called

```JavaScript
var moduleConfig = {
  onRuntimeInitialized: function () {
    this.js_hello();
  },
};
const myModule = await createModule(moduleConfig);
```

> [!NOTE]
> The difference in how Module properties are specified
>
> - In the Non-MODULARIZE approach, you predefine a global `Module` object and set its properties before the JS glue code loads. When the JS glue code finds an existing global `Module`, it layers its runtime content on top of it, so the properties you set beforehand take effect.
> - In the MODULARIZE approach, there is no global `Module`, so you build an object holding your configuration properties and pass it as an argument to the factory function. The factory initializes the instance based on that object.

> [!CAUTION]
> Using the same `EMSCRIPTEN_BINDINGS` name (`moduleA`, `moduleB`) and building into a single Wasm will succeed at build time, but causes an error when the Module object is created. When declaring multiple binding blocks, you must use independent names ([Binding Regular Functions - Embind](/en/posts/embind/#binding-regular-functions))
>
> ```
> Uncaught (in promise) BindingError: Cannot register multiple overloads of a function with the same number of arguments (0)!
> ```

## Building Two Modules and Calling Their Exported Functions

You can build two source files into separate JS glue code and Wasm, and load each as a separate Module.

- Advantages
  - If you only need `module_a` at first and `module_b` later, the initial load gets lighter and `module_b` can be lazy-loaded
  - If a Module is modified, you only need to rebuild/redeploy that one Module
  - Parallel fetching of Wasm becomes possible
- Disadvantages
  - Since separate Wasm memories can't be shared, if the Modules are tightly coupled, you end up with the inefficiency of copying data back and forth through JS
  - Each Wasm/JS glue code bundle includes its own copy of the Emscripten runtime, so duplicated overhead can grow ([Emscripten Introduction - Disadvantages](/en/posts/emscripten-intro/#disadvantages))

### Building

```bash
em++ module_a.cpp -o module_a.js -lembind
em++ module_b.cpp -o module_b.js -lembind
python -m http.server 8080
```

- If you call the glue code built as above (`module_a.js`, `module_b.js`), both use the default Module name, so the second Wasm to load fails to create its Module object and an error occurs
- Therefore, each module needs to be initialized under a separate name

```bash
em++ module_a.cpp -o module_a.js -lembind -s MODULARIZE -s EXPORT_NAME="createModuleA"
em++ module_b.cpp -o module_b.js -lembind -s MODULARIZE -s EXPORT_NAME="createModuleB"
python -m http.server 8080
```

- Exports each Module's factory function as `createModuleA` and `createModuleB`

### Running It

```HTML
<script async type="text/javascript" src="module_a.js"></script>
<script async type="text/javascript" src="module_b.js"></script>
```

- Modify the part that calls the JS glue code

![Module Properties](images/Multiple_Modules.png)
_Figure 3. Creating two modules and calling the function exported from each_

### Script to Run

```JavaScript
const ModuleA = await createModuleA();
const ModuleB = await createModuleB();
ModuleA.js_hello();
ModuleB.js_hello_with_name("WebAssembly");
```

### Example Code and emsdk Version

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-9
- emsdk 5.0.3

## Controlling Behavior with Module Properties - Changing How Output Is Handled

The Module contains not only exported functions but also several properties for controlling behavior. `Module.onRuntimeInitialized`, seen in the earlier example, is one such property that lets you configure what happens after the Module has been initialized. There are properties like `Module.print` and `Module.printErr` that the JS glue code provides by default, and you can think of using them as overriding those properties.

- `Module.print`: lets you configure how stdout, delivered through `printf` or `std::cout` in C++ code, gets used. With the default behavior provided by the JS glue code, strings sent to stdout are printed to the console via the `console.log` function.
- `Module.printErr`: configures how stderr, delivered through `fprintf(stderr, ...)` or `std::cerr` in C++ code, gets used

### Source Code

```C++
#include <emscripten/bind.h>
#include <string>
#include <iostream>

void print_to_stdout(const std::string& message) {
  std::cout << message << std::endl;
}

void print_to_stderr(const std::string& message) {
  std::cerr << message << std::endl;
}

EMSCRIPTEN_BINDINGS(my_module) {
  emscripten::function("js_print_to_stdout", &print_to_stdout);
  emscripten::function("js_print_to_stderr", &print_to_stderr);
}

// index.html
<!doctype html>
<html>
  <head>
    <title>Emscripten Example-10</title>
  </head>
  <body>
    <div id="stdout" style="color: green"></div>
    <div id="stderr" style="color: red"></div>
    <script async type="text/javascript" src="stdout_stderr.js"></script>
  </body>
</html>
```

### Building

```bash
em++ stdout_stderr.cpp -o stdout_stderr.js -lembind -s MODULARIZE -s EXPORT_NAME="createModule"
```

### Running It

![Module_stdout_stderr](images/Module_stdout_stderr.png)
_Figure 4. `js_print_to_stdout` is delivered via `console.log` and `js_print_to_stderr` via `console.error`._

### Capturing stdout/stderr Output to Update the HTML

```JavaScript
const moduleConfig = {
  print: (text) => {
    const out = document.getElementById("stdout");
    out.innerText = text;
  },
  printErr: (text) => {
    const err = document.getElementById("stderr");
    err.innerText = text;
  }
};
const module = await createModule(moduleConfig);
```

- Overrides the `print` and `printErr` callback functions so they manipulate the DOM instead

![stdout_stderr_to_html](images/stdout_stderr_to_html.png)
_Figure 5. Calling `js_print_to_stdout` and `js_print_to_stderr` no longer prints to the console — instead, you can see the output rendered into the HTML._

### Example Code and emsdk Version

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-10
- emsdk 5.0.3

## Other Module Creation Options

- `--pre-js`
  - An option used when building Wasm
  - `--pre-js config.js`: the given JavaScript file (config.js) gets placed right before the part of the JS glue code where the Module is created
  - Creates the Module by overriding it with the object passed in below

```JavaScript
// config.js
const Module = {
  print: (text) => {
    const out = document.getElementById("stdout");
    out.innerText = text;
  },
  printErr: (text) => {
    const err = document.getElementById("stderr");
    err.innerText = text;
  }
};
```

- `--post-js`
  - An option used when building Wasm
  - `--post-js config.js`: the given JavaScript file (config.js) gets placed at the very end of the JS glue code
  - Note that the JavaScript code included here is not guaranteed to run after the Module has been created. To run code at a point in time after the Module is created, add it to `onRuntimeInitialized` (Non-MODULARIZE) or to `.then()` on the Promise returned by the factory function (MODULARIZE)

## Other Useful Properties

- `Module.arguments`
  - The specified values get passed to `main`'s `argc`/`argv`

```C++
// main.cpp
#include <iostream>

int main(int argc, char **argv) {
  for (int i = 0; i < argc; ++i) {
    std::cout << "Argument " << i << ": " << argv[i] << std::endl;
  }
  return 0;
}

// Build
em++ main.cpp -o main.js -s MODULARIZE -s EXPORT_NAME="createModule"

// Run
const moduleConfig = {
  arguments: ["arg1", "arg2", "arg3"],
};
const module = await createModule(moduleConfig);

// Output
Argument 0: ./this.program
Argument 1: arg1
Argument 2: arg2
Argument 3: arg3
```

- `Module.locateFile`
  - When loading the JS glue code, it usually also needs to load the Wasm file (`.wasm`) or a data file (`.data`) alongside it. Use this when you want the Wasm or data file to be fetched from a different URL than the JS glue code

```JavaScript
// Example where the JS glue code is at the root, and Wasm is under a /wasm/ folder
const Module = {
  locateFile: function(path, prefix) {
    if (path.endsWith(".wasm")) {
      return "/wasm/" + path;
    }
    return prefix + path;
  }
};

// Example where Wasm is served from a CDN
const Module = {
  locateFile: function(path) {
    return "https://cdn.example.com/wasm/" + path;
  }
};
```

- `Module.onAbort`
  - Specifies a callback function to call when the program terminates abnormally

```JavaScript
const Module = {
  onAbort: function(reason) {
    console.error('WASM aborted:', reason);
  }
};
```

- `Module.onRuntimeInitialized`
  - Briefly introduced earlier already, and one of the most commonly used properties
  - The Emscripten runtime's initialization has fully completed, and it's safe to run compiled code

```JavaScript
const Module = {
  onRuntimeInitialized: function() {
    // Write the code you want to run after initialization here
  }
};
```

- `Module.noExitRuntime`
  - Setting this to `true` lets you keep using C++ code even after C++'s `main` function has returned
  - When it's obvious that the Wasm runtime needs to stay alive, as with `emscripten_set_main_loop(...)`, Emscripten sometimes handles this setting automatically

```JavaScript
const Module = {
  noExitRuntime: true
};
```

- `Module.noInitialRun`
  - Setting this to `true` prevents `main` from running automatically
  - You can decide when to call `main` yourself using `Module.callMain()`. Note that `callMain` is also not exported on Module by default, so you need to add it to `EXPORTED_RUNTIME_METHODS`

```JavaScript
const Module = {
  noInitialRun: true
};
```

- `Module.preInit`, `Module.preRun`, `Module.postRun`
  - You can define the work to be done at each of these considering the initialization order below

```
JS runtime basic initializer → preInit → preRun → C/C++ global initializer → main() → postRun
```

- `Module.instantiateWasm`
  - An advanced but important API
  - Normally Emscripten automatically handles downloading the `.wasm` file, compiling it as WebAssembly, and creating the WebAssembly instance; use this when you want to control that process directly
  - Useful when you want to parallelize asynchronous work to speed up initialization

```JavaScript
Module = {
  instantiateWasm: async (imports, successCallback) => {
    // Locate/fetch the Wasm binary.
    const result = await /* custom wasm instantiation */;
    successCallback(
      result.instance,
      result.module
    );
  };
};
```

## FAQ

- I have C++ code that calls JS via [`EM_ASM`/`EM_JS`](/en/posts/call-js-from-cpp/), and I'm building with `MODULARIZE` and `EXPORT_NAME` — do I need to know the module name when writing the C++ code?
  - No. Even if you build with `-sMODULARIZE -sEXPORT_NAME=createModuleA`, you can generally just use `Module` inside `EM_JS`/`EM_ASM`.
  - In a typical `-sMODULARIZE` build, the name you specify with `EXPORT_NAME` (e.g. `createModuleA`) is the name of the factory function used from external JavaScript to create a module instance.
  - The `Module` used inside `EM_ASM`/`EM_JS`, on the other hand, is a name that refers to the current module instance within the private scope of the generated glue code. Its role is therefore different from `EXPORT_NAME`, and your C++ code doesn't need to know the `EXPORT_NAME` value.
  - However, in the experimental `-sMODULARIZE=instance` mode, using `Module` inside `EM_JS` or JS library code is currently unsupported. This should be distinguished from the ordinary `MODULARIZE + EXPORT_ES6` combination, which doesn't have this limitation. (As of emsdk 6.0.8)

- How do you access Wasm's heap memory through the Module?
  - The Module provides typed array views into Wasm's linear memory. Notable ones include `Module.HEAP8`/`Module.HEAPU8` (8-bit), `Module.HEAP32`/`Module.HEAPU32` (32-bit integers), and `Module.HEAPF64` (64-bit floating point), letting you read and write memory directly at whatever unit size you need.
  - These views, however, are not exported on the Module by default. Per Emscripten's own documentation, it used to export quite a few runtime elements by default, but all of those have since been removed — anything you need must be added explicitly to `EXPORTED_RUNTIME_METHODS`. For example, you'd need a build option like `-s EXPORTED_RUNTIME_METHODS=HEAP32,getValue,setValue` for the example below to work.
  - For example, to write and read an integer in a 4-byte buffer allocated with `malloc` in C++, you could do the following.

    ```JavaScript
    const ptr = Module._malloc(4);        // ptr is a byte offset into the Wasm heap
    Module.HEAP32[ptr / 4] = 42;           // ptr / 4: HEAP32 is a view accessed in 4-byte (32-bit) units, so dividing the byte offset by 4 converts it into an index that says "which 4-byte element is this"
    console.log(Module.HEAP32[ptr / 4]);   // 42
    Module._free(ptr);
    ```

    - `Module.HEAP32` is an `Int32Array` that views the entire Wasm memory buffer in 4-byte units. You access it by index like a normal JavaScript array, but that index doesn't mean a byte offset — it means "which 4-byte slot". Since the `ptr` returned by `_malloc` is a byte address, you need to divide it by the element size (4 bytes) to use it as an index into `HEAP32`. (With a 1-byte view like `HEAP8`, there's no need to divide — you can use `ptr` directly as the index.)

  - Instead of manually dividing the index by the element size, you can use `Module.getValue(ptr, 'i32')` and `Module.setValue(ptr, 42, 'i32')`, which let you skip worrying about the type size. These two functions likewise need to be added to `EXPORTED_RUNTIME_METHODS` before you can use them.
  - For a hands-on example of exchanging a whole array between C++ and JS with no copying, using a memory view, see [Exchanging Arrays between JavaScript and C++ - Using Memory Views](/en/posts/emscripten-exchange-array/#using-memory-views)

- What if the `onRuntimeInitialized` callback you registered never gets called?
  - `onRuntimeInitialized` only gets called if it's already included in the Module configuration object while initialization is in progress. If you assign this property after calling the factory function, or after initialization has already finished in Non-MODULARIZE mode, that moment has already passed, so the callback won't be called.
  - In the MODULARIZE approach, the Promise returned by the factory function itself doesn't resolve until runtime initialization — which internally includes calling `onRuntimeInitialized` — has finished. So you don't need to register `onRuntimeInitialized` separately; it's safe to just write your code right after `await createModule()`.

- What happens if you call the same factory function multiple times? Can one module be instantiated more than once?
  - Yes — this is one of the key advantages of `MODULARIZE`. Each call to the factory function creates a new, isolated instance, and instances don't share Wasm memory or internal state with each other.

    ```JavaScript
    const instance1 = await createModule();
    const instance2 = await createModule();
    // instance1 and instance2 each have their own independent Wasm memory
    ```

  - The "two modules" case covered earlier was about instantiating two different source files (`module_a.cpp`, `module_b.cpp`) once each, whereas this is about instantiating a single module — built from one source — multiple times. This is useful, for example, when each web worker needs its own independent computation instance.

- I built with `MODULARIZE`, but I don't see `Module` in the browser console — why?
  - The whole point of `MODULARIZE` is to encapsulate the internal code and symbols inside the factory function's private scope, so the global namespace doesn't get polluted. The default Non-MODULARIZE output exposes not just `Module` but its internal symbols to the global scope too, whereas `MODULARIZE` deliberately prevents that — so it's expected that typing `Module` into the console shows nothing.
  - If you want to access an instance for debugging, you can just assign the object returned by the factory function to whatever global variable you like.

    ```JavaScript
    window.debugModule = await createModule();
    ```

## References

- [Emscripten - Module object](https://emscripten.org/docs/api_reference/module.html#module-object)
