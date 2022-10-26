# Documentation



## Chico: a C/C++ CDK for the Internet Computer

Chico allows C/C++ developers to build [Internet Computer](https://internetcomputer.org/) (IC) applications (canisters) using C and C++.
This is a description of how to use Chico to either write a new canister in C/C++ or to port an existing C/C++ application into the IC.

### Working with C/C++ on the IC

Canisters are the basic unit of computation and storage on the IC, and can be thought of as microservices implemented as containers or serverless functions in the cloud. Canisters can be built using Motoko, Rust, Typescript, python, and more. Any language that targets wasm can be used to develop a canister.
Unlike containers and serverless functions, canisters do not have access to any system services, and can only communicate with other canisters and the outside world through query and update calls. Canisters can also invoke a set of IC "system calls" for accessing time and debug printing to name a few.

Working with C code to build a canister means that no system calls are allowed. A canister can't invoke the function `printf` for instance. The IC wouldn't know what to do with such a call. Similarely, any traditional system call that gives access to the network or any perferals is not allowed. To detect that your code is not supported by the IC and cant produce a canister that is not going to trap, you can try the following:
* Try compiling your code using [wasi](https://wasi.dev/). If that fails, you know you can't produce a working canister. Wasi splits the system library `libc` into two groups. The first one is the group of supported system calls that can be translated to wasm. The second consists of the system calls that can not, and therefore can not be invoked by a canister wasm code.
* Inspect your code and see if it invokes a system call that is not supported by wasi.

Once a working canister can be built, a wrapper code needs to be used to invoke the canister code. The wrapper code implements the I/O operations of the canister. That is, operations to get data into and out of the canister. The data is usually specified by a `.did` file. This is effectively the implementation of any query or update call supported by the canister. Chico provide a number of function for supporting basic types, and vectors of basic types. A custom implementation of the canidid interface supporting arbitrary types is provided in a tool to be released called `candidc` (for Candid C).

### Building a C canister using Chico

Once a C/C++ canister code is determined to be supported, building it is as easy as invoking the `clang` compiler. Clang can target wasm, and can compile individual files into wasm files that can be linked into a single canister wasm code. 

The [Dockerfile](https://github.com/ICPorts-labs/chico/blob/main/examples/HelloWorld/Dockerfile) for the Hello world example shows you the dependencies required to use Chico.

### porting an existing application to the IC

## Hello world example

## Sqlite

## Ports: a list of applications ported to the IC

## TODO
