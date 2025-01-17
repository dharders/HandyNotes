#c-cpp/cpp11 #c-cpp/cpp14 #c-cpp/cpp20
## 2023.08.21

> https://hackingcpp.com/cpp/lang/type_system_basics.html

## Constant Expressions: `constexpr` 

- _must_ be _computable_ at compile time if decorates a variable
- _can_ be computed at runtime if not invoked in a `constexpr` context
- all expressions inside a `constexpr` context must be `constexpr` themselves
- constexpr functions may contain
  - **C++11**  nothing but one return statement
  - **C++14**  multiple statements

```cpp
// two simple functions:
constexpr int cxf (int i) { return i * 2; }
          int foo (int i) { return i * 2; }
constexpr int i = 2;       // OK '2' is a literal
          int j = cxf(5);  // OK, cxf is constexpr 
constexpr int k = cxf(i);  // OK, cxf and i are constexpr
          int x = 0;       // not constexpr
          int l = cxf(x);  // OK, not a constexpr context
// constexpr contexts:
constexpr int m = cxf(x);  // error at x
constexpr int n = foo(5);  // error at foo
```

## `consteval`(C++20)

- can only decorate functions
- must be evaluated at compile time

## With function calls

- Every function calls `consteval` functions must be declared as `consteval`
- Functions calls `constexpr` functions are not necessarily declared as `constexpr`

