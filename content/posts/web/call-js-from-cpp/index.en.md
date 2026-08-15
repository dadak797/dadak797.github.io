---
title: "Calling JavaScript Functions from C++ - EM_ASM, EM_JS"
date: "2026-08-15T16:06:30+09:00"
draft: true
description: "An introduction to calling JavaScript functions from C++. This covers how to use EM_ASM and EM_JS, macros provided by Emscripten, to do so."
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
> An introduction to accessing the DOM and calling JavaScript functions from C++. The first example uses `EM_ASM` in C++ to call a JavaScript function and pass a C++ argument to JavaScript. The second example uses `EM_JS` and `EM_ASM` in C++ to print the mouse position in the web browser.

## Printing to the Console from C++ via a JavaScript Function

### Source Code

```C++
// main.cpp
#include <iostream>
#include <emscripten.h>

int main() {
  EM_ASM({
    console.log("Hello, WebAssembly! (From C++ via JavaScript)");
  });

  int c_number = 100;
  EM_ASM({
    const js_number = $0;
    console.log("Hello, " + js_number + "! (From C++ with number via JavaScript)");
  }, c_number);

  const char* c_name = "WebAssembly";
  EM_ASM({
    const js_name_ptr = $0;
    const js_name = UTF8ToString(js_name_ptr);
    console.log("Hello, " + js_name + "! (From C++ with string via JavaScript)");
  }, c_name);

  return 0;
}

// template.html
<!doctype html>
<html>
  <head>
    <title>Emscripten Example-5</title>
  </head>
  <body>
    {{{ SCRIPT }}}
  </body>
</html>
```

- `EM_ASM`
  - A macro function that lets you use JavaScript code from within C++ code
  - You need to include `emscripten.h` to use it
  - It's called in three forms: with no arguments, with a numeric argument, and with a string argument
  - You can write JavaScript code inside the `{}` block within `EM_ASM()`
  - You can pass C++ arguments sequentially after the `{}` block, and receive them inside the block as `$0, $1, ...`. Argument indices start at 0
  - Since the only thing you can effectively pass from C++ to JavaScript is a numeric type (i32, i64, f32, f64) (the same is true in the other direction, from JavaScript to C++ — see [Passing Strings from JavaScript to C++](/en/posts/call-cpp-from-js/#passing-strings-from-javascript-to-c)), you need to pass a pointer and then convert it into a JavaScript string using the `UTF8ToString` function
- `UTF8ToString(_ptr_[, _maxBytesToRead_][, _ignoreNul_])`
  - A function that takes a pointer to a C++ UTF-8 string and converts it into a JavaScript string
  - First argument: the pointer to the string
  - Second argument: the maximum number of bytes to read. If omitted, the string is converted by scanning until a null character is found
  - Third argument: if set to true, the string length isn't determined by scanning for a null character — the value of the second argument is used instead

### Building

```bash
em++ main.cpp -o index.html --shell-file template.html -s EXPORTED_RUNTIME_METHODS="UTF8ToString"
```

- Add `UTF8ToString` to the `EXPORTED_RUNTIME_METHODS` list to use it

### Result

![example-em_asm](images/example-em_asm.png)
_Figure 1. You can see the strings being printed in sequence via JavaScript's console.log function, called from C++._

### Example Code and emsdk Version

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-5
- emsdk 5.0.3

## Getting the Mouse Position in C++ via a JavaScript Function

### Source Code

```C++
#include <emscripten.h>
#include <iostream>

// Get the mouse's X-coordinate using a JavaScript global variable
EM_JS(int, jsGetMouseX, (), {
  return window.mouseX;
});

// Get the mouse's Y-coordinate using a JavaScript global variable
EM_JS(int, jsGetMouseY, (), {
  return window.mouseY;
});

int g_MouseX = 0;
int g_MouseY = 0;

void MainLoop() {
  const int mouseX = jsGetMouseX();
  const int mouseY = jsGetMouseY();

  // Update the global variables with the new position
  const bool changed = mouseX != g_MouseX || mouseY != g_MouseY;
  if (!changed) {
    return;
  }
  g_MouseX = mouseX;
  g_MouseY = mouseY;

  // Print the new position to the console
  std::cout << "Mouse position: (" << g_MouseX << ", " << g_MouseY << ")" << std::endl;

  // Manipulate the HTML DOM to render the new position
  EM_ASM({
    const mousePosElem = document.getElementById('mouse-pos');
    mousePosElem.innerText = 'Mouse position: (' + $0 + ', ' + $1 + ')';
  }, g_MouseX, g_MouseY);
}

int main() {
  EM_ASM({
    window.mouseX = 0;
    window.mouseY = 0;

    // Register a mousemove event handler
    window.addEventListener('mousemove', function(event) {
      // Store the mouse position in the JavaScript global variables mouseX, mouseY
      window.mouseX = event.clientX;
      window.mouseY = event.clientY;
    });
  });

  emscripten_set_main_loop(MainLoop, 0, true);

  return 0;
}

// template.html
<!doctype html>
<html>
  <head>
    <title>Emscripten Example-6</title>
  </head>
  <body>
    <h1 id="mouse-pos">Mouse position: (0, 0)</h1>
    {{{ SCRIPT }}}
  </body>
</html>
```

- `EM_JS`
  - A macro function that wraps a function written in JavaScript so it can be called from C++
  - First argument: the function's return type
  - Second argument: the name to use for the function in C++
  - Third argument: the function's argument list
  - Write the function body as JavaScript code inside `{}`
  - Creates the `jsGetMouseX`/`jsGetMouseY` functions, which pass the mouse position to C++ using the JavaScript global variables `window.mouseX`/`mouseY`
- `MainLoop` function
  - The function called from the main loop
  - Every frame, it retrieves the mouse position using the `jsGetMouseX`/`jsGetMouseY` functions, and only prints the updated position to the console and HTML when the position has changed
- `main` function
  - Uses the `EM_ASM` macro to register a `mousemove` event handler, and declares the JavaScript global variables `mouseX`, `mouseY`
- `emscripten_set_main_loop`
  - In native C++ you'd create an infinite loop with a while statement, but doing that in WebAssembly never returns the main thread to the browser, leaving the browser stuck on an unresponsive page
  - To get around this, the main thread is left under the browser's control, and the browser periodically calls the function passed in as an argument
  - First argument: the function the browser should call periodically. It must return void and take no arguments
  - Second argument: sets the fps. If set to 0 or a negative value, the browser's `requestAnimationFrame` mechanism is used to call the main loop function
  - Third argument: when set, this throws an exception to stop the execution of the calling function. As a result, the code after the call to `emscripten_set_main_loop()` doesn't run, and the main loop begins
  - [Emscripten API Reference](https://emscripten.org/docs/api_reference/emscripten.h.html#id3)

### Building

```bash
em++ main.cpp -o index.html --shell-file template.html
```

### Result

![example-em_js](images/example-em_js.png)
_Figure 2. You can see the mouse position being printed to both the HTML and the console._

### Example Code and emsdk Version

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-6
- emsdk 5.0.3

## FAQ

- Which should I use, `EM_ASM` or `EM_JS`?
  - As with `ccall` and `cwrap`, for a function you'll call repeatedly it's better to wrap it once with `EM_JS`; for a one-off call, `EM_ASM` is more convenient ([Calling C++ Functions from JavaScript](/en/posts/call-cpp-from-js/#faq))
- Looking at the examples, it seems like this could just be handled in JavaScript — is it really necessary to call JavaScript from C++?
  - You might not feel a strong need to in the examples above. But there are cases where you're required to use JavaScript code from within C++, such as the following:
  - When your C++ code needs to open a file browser dialog and read a file ([Working with Files in WebAssembly]())
  - When your C++ code needs to use the JavaScript Fetch API ([Fetching Data in WebAssembly]())
  - When doing graphics programming in C++ (WebGL, WebGPU) and you need to resize the frame buffer according to the browser's size
- What happens if I call an asynchronous function from `EM_ASM` or `EM_JS`?
  - Both `EM_ASM` and `EM_JS` are synchronous calls by default, so you can't get a Promise back directly. To make an asynchronous call, you need to use `EM_ASYNC_JS`, inside which you can use `await` to receive a value. To use `EM_ASYNC_JS`, you also need to add the `-s ASYNCIFY` build option. ([Asyncify](https://emscripten.org/docs/porting/asyncify.html#making-async-web-apis-behave-as-if-they-were-synchronous))
- How do I debug JavaScript code written with `EM_ASM`/`EM_JS`?
  - The code inside `EM_ASM`/`EM_JS` is carried over as-is, as a string, into the build output (e.g., `index.js`) — `em++` doesn't check its JavaScript syntax. So even if there's a typo or syntax error, the build still succeeds, and the problem only shows up once you run it in the browser, as something like an `Uncaught SyntaxError` in the DevTools Console tab.
  - The error location points to a specific line in the compiled glue code file (`index.js`), not to `main.cpp`, so if you have multiple `EM_ASM`/`EM_JS` blocks it can be hard to tell right away which one caused the problem.
  - If you need to set a breakpoint, open that glue code file in the DevTools Sources tab and set it at the actual JavaScript code location. In an optimized build, though, the code will be minified and hard to find, so while debugging it's worth building with the `-g` option or temporarily adding a `console.log` to pinpoint the location.

## References

- [Inline assembly/JavaScript](https://emscripten.org/docs/api_reference/emscripten.h.html#inline-assembly-javascript)
