# std::compile_error function
Document number: D0nnnnR0
Date: 2020-03-25
Project: Programming Language C++, Library Working Group
Reply-To: Connor Horman <chorman64@gmail.com>

## I. Table of Contents

## II. Introduction

Neither the standard library nor the core language provides a method for a constant expression to report an error. This proposal allows constant expressions to do so, without complicating implementations.

## III. Motivation and Scope

Constant Expressions, introduced in C++11, allow a subset of the C++ language to be evaluated by programs at compile time. However, a general issue is that constant expressions cannot report errors when failure would occur.
Such expressions would likely throw an exception, which results in the expression being reduced to runtime behavior. In some cases, a constant expression is required, but in others one is not but may be desired by the user. In particular, the initialization of static variables that would otherwise be constant initialized. This proposal seeks to allow these situations to report an error, without adding the complications of allowing exceptions at compile time, by introducing a function to trigger a compile time error in these circumstances.
This is currently not possible to do, without the use of compiler extensions, or by relying on odr-violations being diagnosed when and only when the function call occurs are runtime.
However, many libraries could benefiet from this ability to do this for domain issues. A constexpr-based math library, for example, could call this function when sqrt is called with a negative input.
A reference implementation is being developed in a fork of llvm-project, at <https://github.com/chorman0773/llvm-project>. It would implement the function proposed herein, as well as a compiler intrinsic which would be needed to provide the behavior of the function.

## IV. Impact on the Standard

The function here is a purely an extension to the standard. 
As mentioned, it is currently impossible to implement this in any version of the standard, as it requires the use of a compiler intrinsic.

## V. Design Decisions
Several alternatives were considered. In particular, a function std::error was considered with this behavior, as well as defined runtime behavior. However, it was ultimately decided against for a number of reasons, specifically that std::compiler_error could be used to implement an error function with both compile time and runtime behavior, but the runtime behavior of std::error would be fixed, both for users and for implementations. With std::compiler_error, runtime behavior of such a function could be chosen. For example, an exception could be thrown at runtime, or std::terminate could be called. 

## VI. Technical Specifications

Function Specification std::compiler_error

```c++
Header <diagnostic>
namespace std{
...
    constexpr void compiler_error(std::string_view diag,std::source_location location=std::source_location::current()) noexcept;
...
}
```

If an expression E evaluates a call to std::compiler_error, where E is *manifestly constant evaluated* the program is ill-formed.
The diagnostic message shall include the string passed as the parameter diag, and the source location indicated by location.

If E is not *manifestly constant evaluated* the behavior is undefined.

## VII. Acknowledgements

## VIII. References

[N3370](https://open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3370.html): This paper uses the wording for *manfiestly constant evaluated* defined there, and adopted into C++20. 