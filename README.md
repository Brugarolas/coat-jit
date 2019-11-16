COAT: COdegen Abstract Types
===

COAT is an embedded domain specific language (EDSL) for C++ to make just-in-time (JIT) code generation easier.
It provides abstract types and control flow constructs to express the data flow you want to generate and execute at runtime.
All operations are transformed at compile-time to appropriate API calls of a supported JIT engine, currently AsmJit and LLVM.

Code specialization has a huge impact on performance.
Tailored code takes advantage of the knowledge about the involved data types and operations.
In C++, we can instruct the compiler to generate specialized code at compile-time with the help of template metaprogramming and constant expressions.
However, constant evaluation is limited as it cannot leverage runtime information.
JIT compilations lifts this limitation by enabling programs to generate code at runtime.

More details are explained in a [blog post](https://tetzank.github.io/posts/coat-edsl-for-codegen/).

## Example

The following code snippet is a full example program.
It generates a function which calculates the sum of all elements for a passed array.
This is hardly a good application of JIT compilation, but it shows how easy and readable code generation is with the help of COAT.

```C++
#include <cstdio>
#include <vector>
#include <numeric>

#include <coat/Function.h>
#include <coat/ControlFlow.h>


int main(){
	// generate some data
	std::vector<uint64_t> data(1024);
	std::iota(data.begin(), data.end(), 0);

	// initialize backend, AsmJit in this case
	coat::runtimeasmjit asmrt;
	// signature of the generated function
	using func_t = uint64_t (*)(uint64_t *data, uint64_t size);
	// context object representing the generated function
	coat::Function<coat::runtimeasmjit,func_t> fn(asmrt);
	// start of the EDSL code describing the code of the generated function
	{
		// get function arguments as "meta-variables"
		auto [data,size] = fn.getArguments("data", "size");

		// "meta-variable" for sum
		coat::Value sum(fn, uint64_t(0), "sum");
		// "meta-variable" for past-the-end pointer
		auto end = data + size;
		// loop over all elements
		coat::for_each(fn, data, end, [&](auto &element){
			// add each element to the sum
			sum += element;
		});
		// specify return value
		coat::ret(fn, sum);
	}
	// finalize code generation and get function pointer to the generated function
	func_t foo = fn.finalize();

	// execute the generated function
	uint64_t result = foo(data.data(), data.size());

	// print result
	uint64_t expected = std::accumulate(data.begin(), data.end(), 0);
	printf("result: %lu; expected: %lu\n", result, expected);

	return 0;
}
```

A more comprehensive example is provided in [another repository](https://github.com/tetzank/sigmod18contest), using COAT in the context of query processing.


## Build Instructions

The library is header-only. Small test programs are included to show how to use the library.

Fetch JIT engines and build them (use more or less cores by changing `-j`, LLVM can take a while...):
```
$ ./buildDependencies.sh -j8
```

Then, build with cmake:
```
$ mkdir build
$ cd build
$ cmake ..
$ make
```


## Profiling Instructions

Profiling is currently only supported for the AsmJit backend and perf on Linux.
Support has to be enabled by defining `PROFILING_ASSEMBLY` or `PROFILING_SOURCE`.
For the example programs, set the cmake variable `PROFILING` to either "ASSEMBLY" or "SOURCE".
"ASSEMBLY" instructs COAT to dump the generated code to a special file which enables `perf report` to annotate the generated instructions with profile information.
Hence, you can profile the generated code on the assembly level.
"SOURCE" adds debug information to map instructions to the C++ source using COAT which generated the instruction.
`perf report` will show the source line next to the instruction.

Profiling steps:
1. `perf record -k 1 ./application`
2. `perf inject -j -i perf.data -o perf.data.jitted`
3. `perf report -i perf.data.jitted`

The generated function should appear with the chosen name in the list of functions.
You can zoom into the function and annotate the instructions with profile information by pressing 'a'.
