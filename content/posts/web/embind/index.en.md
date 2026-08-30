---
title: "Using C++ Classes from JavaScript - Embind"
date: 2026-08-19
draft: false
description: "An introduction to Embind for using C++ classes from JavaScript. Also covers binding not only C++ classes but also functions, variables, and enum classes with Embind, and using them from JavaScript."
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
> An introduction to Embind for using C++ classes from JavaScript. Also covers binding not only C++ classes but also functions, variables, and enum classes with Embind, and using them from JavaScript.

## Binding Regular Functions

### Source Code

```C++
// hello_embind.cpp
#include <emscripten.h>
#include <emscripten/bind.h>
#include <string>
#include <iostream>

void hello() {
  std::cout << "Hello, WebAssembly! (From C++)" << std::endl;
}

void hello(const std::string& name) {
  std::cout << "Hello, " << name << "! (From C++)" << std::endl;
}

void hello_with_number(int number) {
  std::cout << "Hello, " << number << "! (From C++)" << std::endl;
}

std::string get_hello_with_name(const std::string& name) {
  std::string message = "Hello, " + name + "! (From C++)";
  return message;
}

EMSCRIPTEN_BINDINGS(my_module) {
  emscripten::function("js_hello", emscripten::select_overload<void()>(&hello));
  emscripten::function("js_hello", emscripten::select_overload<void(const std::string&)>(&hello));
  emscripten::function("js_hello_with_number", &hello_with_number);
  emscripten::function("js_get_hello_with_name", &get_hello_with_name);
}

// template.html
<!doctype html>
<html>
  <head>
    <title>Emscripten Example-7</title>
  </head>
  <body>
    {{{ SCRIPT }}}
  </body>
</html>
```

- You need to include `bind.h`
- The `my_module` that follows `EMSCRIPTEN_BINDINGS` is just an identifier for the binding block — it has nothing to do with the names used to call the bound functions from JavaScript. That said, like a variable name, it may only contain letters, digits, and underscores (`_`), and can't start with a digit. Also, if binding blocks are declared across multiple files, this block name must not be duplicated
- `emscripten::function`
  - First argument: the function name to use in JavaScript. It doesn't need to match the C++ function's name
  - Second argument: the C++ function to bind to JavaScript. Prefix it with `&` to pass it as a function pointer
  - Two functions were exported under the same name `js_hello` — on the JavaScript side, you can only overload functions sharing the same name when they take different numbers of arguments
- `emscripten::select_overload`
  - Used to explicitly specify a function's return type, arguments, and name, in order to pick out the exact overloaded function in C++

### Building

```bash
em++ hello_embind.cpp -o index.html --shell-file template.html -lembind
```

- You need to add `-lembind` to link in the Embind runtime. Adding `--bind` also works, but it's an older approach

### Result

![export-functions_embind](images/export-functions_embind.png)
_Figure 1. Running functions exported via Embind. You can see that even a string is passed through correctly, with no manual memory allocation required._

- The difference between `EMSCRIPTEN_KEEPALIVE` and Embind ([Exporting C++ Functions](/en/posts/call-cpp-from-js/#exporting-c-functions))
  - When calling an exported function directly, you need to prefix it with an underscore (`_`) ([Running in the Browser](/en/posts/call-cpp-from-js/#running-in-the-browser)). When calling it through `ccall`/`cwrap`, or when it's exported via Embind, you don't need the underscore (`_`)
  - When exporting with Embind, you don't need to wrap the function in `extern "C"`
  - Embind only allows registered types as function arguments, and since the `char` type isn't registered, you can't use `const char*` as an argument
  - Embind lets you use the `std::string` class instead of `const char*` to pass and return strings. This avoids the tedious process you'd otherwise have to go through when calling a function exported with `EMSCRIPTEN_KEEPALIVE` directly ([Passing Strings from JavaScript to C++](/en/posts/call-cpp-from-js/#the-process-of-passing-a-string-from-javascript-to-c))
  - Even when a function returns `std::string`, it's automatically converted into a JavaScript string

> [!NOTE]
> The most convenient way to export C++ functions to JavaScript: **Embind**
>
> 1. It's convenient to use `std::string` when passing a string argument or receiving one as a return value
> 2. You can use the function name directly, without prefixing it with an underscore (`_`)
> 3. You don't need to wrap the function in `extern "C"`
> 4. You can export and use C++ classes
> 5. The downside is an overall increase in overhead — larger Wasm size, runtime overhead, and longer compile times ([Emscripten Introduction - Disadvantages](/en/posts/emscripten-intro/#disadvantages))

### Example Code and emsdk Version

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-7
- emsdk 5.0.3

## Binding a Class

### Source Code

```C++
// class_embind.cpp
#include <emscripten/bind.h>
#include <iostream>

constexpr double PI = 3.14159265358979323846;

enum class ShapeType {
  Circle,
  Rectangle,
};

class Shape {
 public:
  Shape(ShapeType type) : m_Type(type) {}

  virtual ~Shape() {
    std::cout << "Shape destructor" << std::endl;
  }

  virtual double GetArea() const = 0;
  ShapeType GetType() const { return m_Type; }

 private:
  ShapeType m_Type;
};

class Circle : public Shape {
 public:
  static Shape* Create(double radius) {
    return new Circle(radius);
  }

  virtual ~Circle() {
    std::cout << "Circle destructor" << std::endl;
  }

  double GetArea() const override { return PI * m_Radius * m_Radius; }

 private:
  double m_Radius;

  Circle(double radius)
    : Shape(ShapeType::Circle), m_Radius(radius) {}
};

class Rectangle : public Shape {
 public:
  static Shape* Create(double width, double height) {
    return new Rectangle(width, height);
  }

  virtual ~Rectangle() {
    std::cout << "Rectangle destructor" << std::endl;
  }

  double GetArea() const override { return m_Width * m_Height; }

 private:
  double m_Width;
  double m_Height;

  Rectangle(double width, double height)
    : Shape(ShapeType::Rectangle), m_Width(width), m_Height(height) {}
};

EMSCRIPTEN_BINDINGS(shape_module) {
  emscripten::enum_<ShapeType>("ShapeType")
    .value("Circle", ShapeType::Circle)
    .value("Rectangle", ShapeType::Rectangle);
  emscripten::class_<Shape>("Shape")
    .function("getArea", &Shape::GetArea)
    .function("getType", &Shape::GetType);
  emscripten::function("createCircle", &Circle::Create, emscripten::allow_raw_pointers());
  emscripten::function("createRectangle", &Rectangle::Create, emscripten::allow_raw_pointers());
}

// template.html
<!doctype html>
<html>
  <head>
    <title>Emscripten Example-8</title>
  </head>
  <body>
    {{{ SCRIPT }}}
  </body>
</html>
```

- Defines the abstract class `Shape`, along with the derived classes `Circle` and `Rectangle`
- Exports the C++ enum class via `emscripten::enum_<>()`
  - Embind maps a C++ enum class to a JavaScript object
  - `<>`: the name of the C++ enum class to export
  - `()`: the name to use for the object in JavaScript
  - `.value()`: adds a value from the enum class that you want to export
  - You can use it like `Module.ShapeType.Circle`
- Exports the C++ class via `emscripten::class_<>()`
  - `<>`: the name of the C++ class to export
  - `()`: the name to use for the class in JavaScript
  - `.function()`: adds a member function of the class that you want to export
- `emscripten::function`
  - A static member function is independent of any particular class instance, so it can be exported as a standalone function
  - `emscripten::allow_raw_pointers()`: required because `createCircle` and `createRectangle` create a `Circle` or `Rectangle` instance and then return a pointer (`Shape*`). This won't cause an error only because the `Shape` class is already registered with Embind
- `Circle` and `Rectangle` are never registered with `emscripten::class_` on their own, yet polymorphism through `Shape`'s virtual function (`GetArea`) is enough to use each derived class's behavior as-is. In other words, exposing just a single base class at minimum — without writing a separate binding for every derived class — is enough to implement the functionality you need

### Result

![export-class_embind](images/export-class_embind.png)
_Figure 2. Creating and destroying a class exported via Embind, and calling its member functions._

- Calling the `Shape` class's virtual function `getArea` correctly calls the derived class's `GetArea` function
- The `enum class ShapeType` is correctly exported as the `Module.ShapeType` object
- Calling the `delete` function correctly invokes each class's destructor

> [!CAUTION]
> Once you're done using an instance, you must call the `delete` function so that the class's destructor is invoked. Otherwise, a memory leak can occur

### Script to Run

```JavaScript
const circle = Module.createCircle(5.0);
console.log(circle.getArea());
console.log(circle.getType() === Module.ShapeType.Circle);
circle.delete();
const rect = Module.createRectangle(3.0, 4.0);
console.log(rect.getArea());
console.log(rect.getType() === Module.ShapeType.Rectangle);
rect.delete();
```

### Example Code and emsdk Version

- https://github.com/dadak797/blog-examples/tree/master/examples/ex-8
- emsdk 5.0.3

## FAQ

- What happens if you use `const char*` as a function argument in Embind?
  - Say you bind a function that takes a `const char*` argument, like this:

    ```C++
    void test(const char* str) {}

    EMSCRIPTEN_BINDINGS(my_module) {
      emscripten::function("js_test", &test);
    }
    ```

  - With no policy specified, it gets blocked right at compile time:

    ```
    error: static assertion failed due to requirement '!std::is_pointer<const char *>::value':
    Implicitly binding raw pointers is illegal.  Specify allow_raw_pointer<arg<?>>
    ```

  - Adding `emscripten::allow_raw_pointers()` gets you past compilation, but calling that function from JavaScript then throws a runtime error:

    ```
    Uncaught UnboundTypeError: Cannot call js_test due to unbound types: PKc
    ```

    - `PKc` is the (Itanium C++ ABI) mangled type name for `const char*`. `allow_raw_pointers()` only "allows raw pointers to be taken as arguments" — it doesn't register the `PKc` type itself with Embind, so there's no way for the JavaScript side to handle that type, which is why this error occurs

  - In other words, even though `allow_raw_pointers()` lets you get past the compiler, the conclusion is the same either way: you can't actually use it. Using `std::string` instead of `const char*` is the only practical solution
  - For the same reason, you can't pass basic-type pointers like `int*` or `float*` as arguments either. For a workaround that casts the pointer to an integer type instead, see [Exchanging Arrays between JavaScript and C++ - Using Memory Views](/en/posts/emscripten-exchange-array/#using-memory-views)

- Do `Circle` and `Rectangle` also need to be registered with `emscripten::class_`?
  - If you only use `createCircle`/`createRectangle` returning a `Shape*`, as in the example above, this works fine without registering them. On the JavaScript side, both instances just look like the `Shape` type, and you can only call functions registered on `Shape`, such as `getArea`/`getType`
  - If you want to call a function that only exists on `Circle` (e.g. `getRadius`) from JavaScript, or distinguish the actual type with something like `circle instanceof Module.Circle`, you need to register the inheritance relationship by specifying `emscripten::base<Shape>`

    ```C++
    emscripten::class_<Circle, emscripten::base<Shape>>("Circle")
      .function("getRadius", &Circle::GetRadius);
    ```

  - Once registered this way, Embind recognizes that the `Shape*` returned by `createCircle` is actually a `Circle` instance, so `getRadius` can be called from JavaScript as well, and `circle instanceof Module.Circle` becomes `true`
  - `getRadius` isn't a virtual function, yet it can still be called through a `Shape*` — this differs from C++, where you can't call a non-virtual child function through a base pointer; it's behavior unique to Embind

- Can you construct an instance directly, like `new Module.Circle(5.0)`, without a factory function?
  - In the current example, `Circle` and `Rectangle` have `private` constructors, so they can't be registered directly with `.constructor<>()`. That's why the `static Create()` factory function is exported separately via `emscripten::function`
  - If you make the constructor `public`, you can attach `.constructor<double>()` so it can be constructed directly from JavaScript

    ```C++
    emscripten::class_<Circle, emscripten::base<Shape>>("Circle")
      .constructor<double>()
      .function("getRadius", &Circle::GetRadius);
    ```

    ```JavaScript
    const circle = new Module.Circle(5.0);  // constructed directly, without createCircle
    ```

  - Note, however, that an abstract class like `Shape` can't have any instances at all, so you can't register `.constructor<>()` on it — only on a derived class

- What if `delete()` doesn't free memory even after you call it, or, conversely, you get an error from using an object that's already been deleted?
  - If a function returns a class instance by value, or takes one as an argument, an extra temporary copy gets created along the way. It's easy to call `delete()` on the instance you actually need while missing this temporary copy
  - It's also easy to miss a `delete()` call along a code path where an exception causes the function to exit early. This is a problem that RAII naturally solves in C++, but that guarantee doesn't carry over to a JS object handed over through Embind, so you need to explicitly guarantee the `delete()` call with `try`/`finally`
  - Conversely, if you keep the same instance in multiple variables and call `delete()` through only one of them, calling a method through any of the other variables throws a "using deleted object"-style error. This is actually Embind's safeguard for catching use of an already-freed object — so if you hit this error, treat it as a sign that some reference somewhere already had `delete()` called on it

## References

- [Emscripten - Embind](https://emscripten.org/docs/porting/connecting_cpp_and_javascript/embind.html)
