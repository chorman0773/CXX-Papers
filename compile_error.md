# std::compile_error function
Document number: D0nnnnR0

Date: 2020-03-25

Project: Programming Language C++, Library Working Group

Reply-To: Connor Horman <chorman64@gmail.com>

## I. Table of Contents

## II. Introduction

Neither the standard library nor the core language provides a method for a constant expression to report an error.
 This proposal allows manifestly constant evaluated expressions to do so,
 without complicating implementations.

## III. Motivation and Scope

Constant Expressions, introduced in C++11, allow a subset of the C++ language to be evaluated by programs at compile time. However, a general issue is that constant expressions cannot report errors when failure would occur.
Such expressions would likely throw an exception, which results in the expression being reduced to runtime behavior. In some cases, a constant expression is required, but in others one is not but may be desired by the user. In particular, the initialization of static variables that would otherwise be constant initialized. This proposal seeks to allow these situations to report an error, without adding the complications of allowing exceptions at compile time, by introducing a function to trigger a compile time error in these circumstances.
This is currently not possible to do, without the use of compiler extensions, or by relying on odr-violations being diagnosed when and only when the function call occurs are runtime.
However, many libraries could benefiet from this ability to do this for domain issues. A constexpr-based math library, for example, could call this function when sqrt is called with a negative input.
A reference implementation is being developed in a fork of llvm-project, at <https://github.com/chorman0773/llvm-project>. It would implement the function proposed herein, as well as a compiler intrinsic which would be needed to provide the behavior of the function.

### Motivating Example 1

Example with std::compile_error
```c++
constexpr void throw_domain_error(std::string_view msg){
    if(std::is_constant_evaluated())
	std::compiler_error(msg);
    else
	throw std::domain_error{msg};
}
constexpr double pow_over_fact(double x,int n){
    double accumulator{1};
    for(int i = 0;i<n;i++)
        accumulator *= x/i;
}
constexpr double sqrt(double x){
    if(x<0)
	throw_domain_error("sqrt of negative is undefined");
    else{
	double val{};
	for(int i = 0;i<20;i++)
	    val += pow_over_factorial(x-1,i);
	return val;
    }
}
const double sqrtm2 = sqrt(-2); // Error: Call to std::compiler_error:  "sqrt negative is undefined"
```

Example without std::compiler_error

```c++
void throw_domain_error(std::string_view msg,std::source_location loc){
    throw std::domain_error{msg};
}
constexpr double pow_over_fact(double x,int n){
    double accumulator{1};
    for(int i = 0;i<n;i++)
        accumulator *= x/i;
}
constexpr double sqrt(double x){
    if(x<0)
	throw_domain_error("sqrt of negative is undefined");
    else{
	double val{};
	for(int i = 0;i<20;i++)
	    val += pow_over_factorial(x-1,i);
	return val;
    }
}
const double sqrtm2 = sqrt(-2); // Throws std::domain_error what()="sqrt of negative is undefined" at runtime.
```

Without std::compile_error, the error is only reported at runtime (if possibility evaluated at runtime). std::compile_error allows a meaningful error to be reported at compile time, to reduce runtime errors.
This is a generally trivial example, and can be solved by making sqrtm2 `constexpr` or `constinit` (with a less customizable error message). 

### Motivating Example 2

Custom Error Messages

```c++
// constexpt_format has limited version of std::format, implemented in constexpr
constexpr void throw_domain_error(std::string_view fmt,double val){
    if(std::is_constant_evaluated())
	std::compile_error(constexpr_format(fmt,val));
    else
	throw std::domain_error{std::format(fmt,val)};
}
constexpr double sqrt(double x){
    if(x<0)
	throw_domain_error("sqrt({}) is not real",x);
    else{
	double val{};
	for(int i = 0;i<20;i++)
	    val += pow_over_factorial(x-1,i);
	return val;
    }
}
const double sqrtm2 = sqrt(-2); // Error: Call to std::compile_error: "sqrt(-2.0) is not real"
const double sqrtm3 = sqrt(-3); // Error: Call to std::compile_error: "sqrt(-3.0) is not real"
```

This lends itself better to more complex cases, where multiple values are in play, as it could be easier to differentiate which value is the errorneous one. 
There is currently no way to achieve this dynamically computed error message at compile-time, whatsoever.

## IV. Impact on the Standard

The function here is a purely an extension to the standard. 
As mentioned, it is currently impossible to implement this in any version of the standard, as it requires the use of a compiler intrinsic.

## V. Design Decisions
Several alternatives were considered. In particular, a function std::error was considered with this behavior, as well as defined runtime behavior. However, it was ultimately decided against for a number of reasons, specifically that std::compile_error could be used to implement an error function with both compile time and runtime behavior, but the runtime behavior of std::error would be fixed, both for users and for implementations. With std::compile_error, runtime behavior of such a function could be chosen. For example, an exception could be thrown at runtime, or std::terminate could be called. 

## VI. Technical Specifications

Function Specification std::compiler_error

```c++
Header <diagnostic>
namespace std{
...
    constexpr void compile_error(std::string_view diag) noexcept;
...
}
```

1. Preconditions:
   - Let `E` be an expression that evaluates a call to `std::compile_error`. The behaviour is undefined if E is not *manifestly constant evaluated*.
2. Effects:
   - A program that evaluates a call to `std::compile_error` that is *manfiestly constant evaluated* is ill-formed. The implementation shall issue a diagnostic which, at the very least contains `diag`. If `diag` contains characters which are not part of the extended source character set, this behaviour is conditionally-supported. If not supported, the diagnostic message is unspecified. 

## VII. Acknowledgements

## VIII. References

[N3370](https://open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3370.html): This paper uses the wording for *manfiestly constant evaluated* defined there, and adopted into C++20. 
