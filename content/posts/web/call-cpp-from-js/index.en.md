---
title: "Calling C++ Functions from JavaScript - ccall, cwrap"
date: "2026-08-09T23:48:06+09:00"
draft: false
description: "An introduction to calling C++ functions from JavaScript. Also covers ccall and cwrap, helper functions that make it easy to pass string arguments."

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
> An introduction to calling C++ functions from JavaScript. Passing a string argument from JS to C++ requires allocating memory in Wasm, copying the data, and freeing it again, but `ccall` and `cwrap` handle this process for you, making it easy to pass strings.

## Exporting C++ Functions

> [!Important]
> To call a Wasm function from JS at runtime, the C++ function you want to call must be exported from the Wasm module using `EMSCRIPTEN_KEEPALIVE` or `EXPORTED_FUNCTIONS`.

The basic process of exporting and calling a function with `EMSCRIPTEN_KEEPALIVE` and `EXPORTED_FUNCTIONS` is the same as what was covered in [Emscripten Installation and Examples - Example 2](/en/posts/emscripten-install/#example-2). Building on that, this post exports several functions with different argument types and looks at the case of passing a string argument.

### Source Code

```C++
// hello_c_api.cpp
#include <iostream>
#include <emscripten.h>

extern "C" {
  EMSCRIPTEN_KEEPALIVE
  void hello() {
    std::cout << "Hello, WebAssembly! (From C++)" << std::endl;
  }

  EMSCRIPTEN_KEEPALIVE
  void hello_with_number(int number) {
    std::cout << "Hello, " << number << "! (From C++)" << std::endl;
  }

  EMSCRIPTEN_KEEPALIVE
  void hello_with_name(const char* name) {
    std::cout << "Hello, " << name << "! (From C++)" << std::endl;
  }
}

// template.html
<!doctype html>
<html>
  <head>
    <title>Emscripten Example-4</title>
  </head>
  <body>
    {{{ SCRIPT }}}
  </body>
</html>
```

- Putting `EMSCRIPTEN_KEEPALIVE` in front of a function tells `Emscripten` to export it so it can be called from JavaScript.
- To minimize the size of the built Wasm, `Emscripten` performs dead code elimination at compile time, but functions marked with `EMSCRIPTEN_KEEPALIVE` are not removed.
- Three functions are exported here — `hello`, `hello_with_number`, and `hello_with_name`. Notice that each one takes different arguments.

### Building

```bash
em++ hello_c_api.cpp -o index.html --shell-file template.html -s EXPORTED_FUNCTIONS="['_hello','_hello_with_number','_hello_with_name']"
```

- `--shell-file`
  - Sets the HTML template file ([Building so that HTML is Generated](/en/posts/emscripten-install/#building-so-that-html-is-generated))
- `EXPORTED_FUNCTIONS`
  - Add the names of the functions to export
  - You must prefix the original function name with an underscore (`_`) when exporting it
  - When exporting multiple functions, write them as an array: `EXPORTED_FUNCTIONS="['_hello','_hello_with_number','_hello_with_name']"`

### Running in the Browser

```bash
python -m http.server 8080
```

![Calling an exported C function](images/call_exported_c.png)
_Figure 1. Calling the `hello` function, written in C++, from the browser._

- If you don't configure anything related to the module name when building WebAssembly, the default name of the WebAssembly module instance is `Module`.
- To call a function exported via `EMSCRIPTEN_KEEPALIVE`, you need to prefix the function name with an underscore (`_`).

> [!CAUTION]
> Calling an exported function before the Wasm module has finished initializing can cause an error. You should either use the `Module.onRuntimeInitialized` callback or wait until initialization completes before calling the function. See the related explanation in [Emscripten Installation and Examples - Example 2](/en/posts/emscripten-install/#example-2) for more details.

> [!NOTE]
> Usually either `EMSCRIPTEN_KEEPALIVE` or `EXPORTED_FUNCTIONS` alone is enough to call a function from JS. The difference is that `EMSCRIPTEN_KEEPALIVE` marks the export inside the source code, while `EXPORTED_FUNCTIONS` specifies it in the build settings.

### Example Code and emsdk Version

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-4
- emsdk 5.0.3

## Passing Strings from JavaScript to C++

![Calling an exported C function with arguments](images/call_exported_c_with_arguments.png)
_Figure 2. Calling the `hello_with_number` and `hello_with_name` functions, written in C++, from the browser. The argument `100` passed to `hello_with_number` prints correctly, but the string argument passed to `hello_with_name` does not print correctly._

> [!NOTE]
> Wasm functions can only take numeric types (i32, i64, f32, f64) as arguments. So to pass a string to a C++ function, you have to use a pointer.

### The Process of Passing a String from JavaScript to C++

![Calling an exported C function with a string argument](images/call_exported_c_with_string.png)
_Figure 3. Passing a string argument to the `hello_with_name` function, written in C++, through several steps. You can see the string argument now printing correctly in `hello_with_name`._

```JavaScript
const str = "WebAssembly";
const len = Module.lengthBytesUTF8(str) + 1;
const ptr = Module._malloc(len);
Module.stringToUTF8(str, ptr, len);
Module._hello_with_name(ptr);
Module._free(ptr);
```

1. Allocate memory on the Wasm heap
   - Get the string's length using the JS runtime helper function `lengthBytesUTF8`
   - Allocate memory using the Wasm module's `_malloc` function. You need to allocate 1 extra byte here to hold the string's null terminator (`\0`)
2. Copy the JS string into the allocated memory
   - Copy it using the JS runtime helper function `stringToUTF8`
3. Pass the pointer to the function
4. Free the allocated memory once you're done using it
   - Free it using the Wasm module's `_free` function

### Building

```bash
em++ hello_c_api.cpp -o index.html --shell-file template.html -s EXPORTED_FUNCTIONS="['_hello','_hello_with_number','_hello_with_name','_malloc','_free']" -s EXPORTED_RUNTIME_METHODS="['lengthBytesUTF8','stringToUTF8']"
```

- `EXPORTED_FUNCTIONS=[...,'_malloc','_free']`
  - Exports C++'s `malloc` and `free` functions for use from JavaScript
- `EXPORTED_RUNTIME_METHODS="['lengthBytesUTF8','stringToUTF8']"`
  - Used to export JS runtime helper functions, as opposed to functions exported from C++
  - These are part of the glue code Emscripten generates when it compiles C/C++, and since they aren't plain JS functions, they can only be accessed as, e.g., `Module.stringToUTF8`.
  - `lengthBytesUTF8(str)`: calculates how many bytes the JS string `str` becomes once encoded as UTF-8
  - `stringToUTF8(str, ptr, len)`: converts the JS string `str` to a UTF-8 string and copies up to `len` bytes into Wasm heap memory starting at the pointer `ptr`, then writes a null terminator at position `len + 1`
- The source code and emsdk version are the same as above

## Passing Strings from JavaScript to C++ Easily - `ccall` and `cwrap`

> [!NOTE]
> Going through those four steps every time you pass a string from JavaScript to C++ is quite tedious. Emscripten provides the JS runtime helper functions `ccall` and `cwrap`, which handle this process for you automatically.

![Calling an exported C function with ccall and cwrap](images/call_exported_c_with_ccall_cwrap.png)
_Figure 4. Calling the `hello_with_name` function, written in C++, from the browser using `ccall` and `cwrap`. `ccall` and `cwrap` make it easy to pass a string from JS to C++._

```JavaScript
Module.ccall('hello_with_name', 'void', ['string'], ['WebAssembly']);
```

- `ccall`
  - Used to directly call an exported C++ function that has a string argument
  - First argument: the function's name in C++
  - Second argument: the return type
  - Third argument: an array of the C++ function's argument types. Use `number` for numeric types.
  - Fourth argument: the arguments to pass to the C++ function

```JavaScript
const helloWithName = Module.cwrap('hello_with_name', 'void', ['string']);
helloWithName("WebAssembly");
```

- `cwrap`
  - Used to wrap an exported C++ function as a JS function
  - First argument: the function's name in C++
  - Second argument: the return type
  - Third argument: an array of the C++ function's argument types
- With `ccall` and `cwrap`, you don't prefix the C++ function name with an underscore (`_`) when calling it.

> [!CAUTION]
> Strings passed through `ccall` and `cwrap` are usually allocated temporarily on the stack and freed automatically once the call returns. So if the Wasm side stores the pointer it received internally and refers back to it later, this will cause a problem. In that case, you need to manage the memory allocation and freeing yourself using `malloc` and `free`.

### Building

```bash
em++ hello_c_api.cpp -o index.html --shell-file template.html -s EXPORTED_FUNCTIONS="['_hello','_hello_with_number','_hello_with_name','_malloc','_free']" -s EXPORTED_RUNTIME_METHODS="['lengthBytesUTF8','stringToUTF8','ccall','cwrap']"
```

- `EXPORTED_RUNTIME_METHODS=[...,'ccall','cwrap']`
  - Builds with `ccall` and `cwrap` included.
- The source code and emsdk version are the same as above

## FAQ

- Is there a way to use C++ classes from JavaScript?
  - Using Embind ([Using C++ Classes from JavaScript](/en/posts/embind/)) or the WebIDL Binder, you can export a C++ class and create instances of it from JavaScript.
- Which should I use, `ccall` or `cwrap`?
  - For a function you'll call repeatedly, it's better to wrap it once with `cwrap`; for a one-off call, `ccall` is more convenient.
- I get an error like "ccall is not defined".
  - Check whether you forgot to add it to `EXPORTED_RUNTIME_METHODS`.
- How do I receive a string that a C++ function returns?
  - If you set `ccall`'s return type to `string`, it internally calls `UTF8ToString` to automatically convert the returned C++ string into a JavaScript string.
  - In the example below, the string lives in read-only data and stays valid until the program exits — but a string built as a local variable on the stack is destroyed the moment the function returns, so returning just a pointer (`const char*`) to one can cause a crash. Note that this is the same problem you'd run into in native C++.

```C++
// C++
extern "C" {
  EMSCRIPTEN_KEEPALIVE
  const char* get_hello() {
    return "Hello from C++!";
  }
}

// JavaScript
const helloStr = Module.ccall('get_hello', 'string', [], []);
console.log(helloStr);  // prints "Hello from C++!"
```

- Can a function that returns a struct or class be called with `ccall`?
  - `ccall`/`cwrap` return types only support `number`, `string`, `boolean`, and `array`, so they can't represent a struct or class made up of multiple fields. Worse, a function that returns a struct by value gets compiled into an entirely different form at the C/Wasm level — one that receives the result through a hidden pointer argument — so calling it with no arguments can even crash.
  - Embind handles this kind of marshaling for you automatically. See [Using C++ Classes from JavaScript](/en/posts/embind/).

## References

- [Connecting C++ and JavaScript](https://emscripten.org/docs/porting/connecting_cpp_and_javascript/index.html)
