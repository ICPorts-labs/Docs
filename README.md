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

### Working with Chico

[Chico](https://github.com/ICPorts-labs/chico) contains everything you need to write canisters in C/C++.
* a definition of IC system calls in https://github.com/ICPorts-labs/chico/blob/main/src/ic0.h
* a definition of canister I/O function for basic types defined in https://github.com/ICPorts-labs/chico/blob/main/src/chico.h

A user can write an arbitrary C application where only IC system calls are invoked and compile it into wasm code. The wasm code can then be directly integrated into a canister built with (dfx)[https://internetcomputer.org/docs/current/references/cli-reference/dfx-parent/].

The application code needs to import Chico definitions by including the Chico headers:

```c
#include "ic0.h"
#include "chico.h"
```

### Building a C canister using Chico

Once a C/C++ canister code is determined to be supported, building it is as easy as invoking the `clang` compiler. Clang can target wasm, and can compile individual files into wasm files that can be linked into a single canister wasm code. 

The [Dockerfile](https://github.com/ICPorts-labs/chico/blob/main/examples/HelloWorld/Dockerfile) for the Hello world example shows you the dependencies required to use Chico.

After downloading Chico and its wasi dependency, two env variables need to be set: CHICO_PATH and WASI_PATH. The first indicates where Chico is installed and the second indicates where the C headers and C/C++ system libraries are defined.

Copiling a C applications like the helloworld app is as simple as invoking a `clang` command which generates a corresponding wasm file:

```c
clang -c helloworld.c -o helloworld.o \
      -I ${CHICO_PATH} \
      -O3 --sysroot=${SYSROOT} --target=wasm32-wasi
```

The helloworld app consists of a set of imports (include files) 

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "ic0.h"
#include "chico.h"
```

and a single function called `greet`

```c
void greet() WASM_EXPORT("canister_query greet");
void greet() {
  ic_log_message("Running the Hello world example in Chico!");
  // read text input
   char *val= ic_reads_text();
   int len = strlen(val);
  // build response
  char s1[] = "Chico says hello to you ";
  char s2[] = " !";
  const size_t len1 = strlen(s1);
  const size_t len2 = strlen(s2);
  char *result_buf = malloc(strlen(s1) + len + strlen(s2) + 1);
  memcpy(result_buf, s1, len1);
  memcpy(result_buf + len1, val , len );
  memcpy(result_buf + len1 + len, s2, len2 + 1);
  ic_writes_text(result_buf);
  free(val);
  free(result_buf);
}
```

The function is declared as an exported querry function. The code of the function uses two of the utilitiy functions provided by Chico:
* a function `ic_reads_text` used by a canister to read an input message of type text (corresponding to a C string)
* a function `ic_writes_text` used by a canister to output to it caller a message of type text.
* a function `ic_log_message` which is a logging function that output its string argument.

Notic that the function also includes C library functions such as `memcpy` and `strlen`. These function are able to be translated to wasm because they do not invoke a system call. The corresponding wasm file is described as follows. There are eight functions declared in the file (functions (;0;) to (;7;)).
The file contains imports declarations that correspond to the IC system calls required to run the code. The first unction defined is the `greet` function.

```lisp
(module
  (type (;0;) (func (result i32)))
  (type (;1;) (func (param i32 i32 i32)))
  (type (;2;) (func (param i32 i32)))
  (type (;3;) (func))
  (type (;4;) (func (param i32 i32 i32 i32)))
  (type (;5;) (func (param i32)))
  (type (;6;) (func (param i32) (result i32)))
  (type (;7;) (func (param i32 i32 i32) (result i32)))
  (import "ic0" "msg_arg_data_size" (func $ic0_msg_arg_data_size (type 0)))
  (import "ic0" "msg_arg_data_copy" (func $ic0_msg_arg_data_copy (type 1)))
  (import "ic0" "trap" (func $ic0_trap (type 2)))
  (import "ic0" "msg_reply_data_append" (func $ic0_msg_reply_data_append (type 2)))
  (import "ic0" "msg_reply" (func $ic0_msg_reply (type 3)))
  (func $canister_query_greet (type 3)
    ...)
```

The rest of the file containes the definitions of the remaining seven functions:

```lisp
(func $match_byte (type 4) (param i32 i32 i32 i32)
    ...)
(func $match_magic (type 2) (param i32 i32)
    ...)
(func $ic_writes_text (type 5) (param i32)
    ...)
(func $malloc (type 6) (param i32) (result i32)
    ...)
(func $dlmalloc (type 6) (param i32) (result i32)
    ...)
(func $free (type 5) (param i32)
    ...)
(func $dlfree (type 5) (param i32)
    ...)
(func $abort (type 3)
    ...)
(func $sbrk (type 6) (param i32) (result i32)
    ...)
(func $memcpy (type 7) (param i32 i32 i32) (result i32)
    ...)
(func $strlen (type 6) (param i32) (result i32)
    ...)
```

These functions correspond to C library functions as well as Chico functions. This code is able to run because it contains no OS system calls but only IC system calls. The rest of the wasm file contains an expolicit export of the function `greet` and global variables definitions such as strings:

```lisp
(export "canister_query greet" (func $canister_query_greet))
  (data (;0;) (i32.const 1024) "Chico says hello to you \00 !\00Invalid byte!\00")
  (data (;1;) (i32.const 1066) "DIDLq"))
```

This code represents the behavior of the canister. Only one exported function is defined. That is the only way to interact with the canister. This interface is defined in the helloworld `.did` file defined as follows:
```
service : {
  greet: (text) -> (text);
}  
```


### Porting an existing application to the IC

Porting an existing applications written in C/C++ can be extremely simple or extremely complex, if not impossible. As explained above, a canister written in C/C++ should not depend on any OS system calls. That's because a canister running on the IC does not have access to the underlying OS on which an IC node runs. A canister is completely isolated from that OS, the hardware it runs on, and can only involke calls to the IC system calls or call other canisters on the IC.

As explained above, it is relatively easy to compile a C/C++ application to a canister (or a set of canisters). In principle, it is fine if an OS system call is present in the canister wasm code, as long as it is never invoked. However, the IC will reject the wasm file as the system call will show up as an imported function. The IC can not resolve the import and will therefore reject the wasm file. So if you are in such a situation, you can simply include in your code a file with dummy definitions of all the OS syetem calls that are unreachable and can not be pruned fro the applications. Given that they will never be called, it down not matter what definitions they might have. Here is an example of a system call in the libc system library with a dummy definition:

```c
int gettimeofday(const char *pathname, int mode){
  trap("calling gettimeofday");
  return 0;
}
```

If the application ever reaches this call, the canister will trap. But if it is unreachable, it can have any dummy definition and return any valid value given the type.

In practical terms, porting an existing application to the IC is a matter of editing the Makefile or any build script and wrapping the calls to the CC and LD env variables with a call that generates wasm from a C/C++ file or link multiple wasm files into a single one.

### Porting criteria for a C application

C applications come in two flavors: applications and libraries. 

<mark style="background-color: #f7e98e"> Applications tend to be executables that can be invoked from the command line. They typically include a `main` function that parses arguments and make subsequent calls. Such applications are usually hard to port to the IC. Canisters do not have usually a main function. They contain query and update calls. Canisters act more like libraries than applications. Hpwever, it is possible to port an application with a main function into the IC. This is done by either:
* editting the application code to eliminate `main` and eliminating the code that parses commandline arguments and creating multiple update or query calls corresponding to the different arguments combination, or by eliminating `main` alltogether.
* designate which functions in the application that correspond to update and query calls, link against the application code and let the linker eliminate any functions that do not belong to the call graph starting from an update or a query call. Not all linkers might be clever enough to be able to perform such code elimination though.</mark> 

<mark style="background-color: #f7e98e"> Libraries tend to be compiled into an archive file containing a set of exported functions and their callees. They are much more amenable to being compiled inot IC canisters. After deciding which exported functions should be a query or an update call, compiling toa library archive into a canister is a matter of updating the Makefile as described above.</mark> 

## Sqlite

[IC_sqlite](https://github.com/HassenSaidi/IC_sqlite) is a port of SQLite to the IC. SQLite is a very popular database installed on billions of devices.
SQLite can be viewed as two different products:
* sqlite as a commandline executable
* sqlitelib as a library that can be linked to an application.

As an executable, it might seem impossible to port to the IC. An executable can not be ported as is to the IC. A main function does not even make sense.
SQLIte as a library makes more sense to port to the IC, so that some of the library functions can be turned into IC query and update calls. Taking advantage of static linking, it is possible to prune the library and only include code that supports the defined query and update calls.

[IC_sqlite](https://github.com/HassenSaidi/IC_sqlite) uses only the following SQLIte library calls:
* `sqlite3_open_v2` to create a database
* `sqlite3_exec` to execute a query or an update call
* `sqlite3_str_new`, `sqlite3_str_append`, `sqlite3_str_finish`, and `sqlite3_str_appendall` to create and expand sqlite strings.
* `sqlite3_str_free` to free an allocated space for sqlite strings.
Any other library calls that do not support these calls can be safely pruned, resulting in a smaller wasm canister code.

SQLite code contains a lot of linus system calls. In particular, file manipulation system calls associated with creating and accessing databases. Databases are stored in files. However, this port to the IC uses the in-memory feature of SQLite which stores all databases in memory. If the in-memory feautre is used on a linux platform, all databases are lost and not persisted when exciting the sqlite application. However, on the IC, the orthogonal persistence property of canisters makes sure that after each update call, the in-memory databases are automatically persisted.

To ensure that none of the linux system calls are present in the wasm canister code, we compile SQLite with the following flags:
```bash
 -DSQLITE_OS_OTHER=1 \
      -DSQLITE_OMIT_PROGRESS_CALLBACK \
      -DSQLITE_OMIT_XFER_OPT\
      -DHAVE_READLINE=0 -DHAVE_EDITLINE=0 -DSQLITE_OMIT_LOAD_EXTENSION \
      -DSQLITE_OMIT_SHARED_CACHE=1 \
      -DSQLITE_ENABLE_MEMORY_MANAGEMENT=1 \
      -DSQLITE_ENABLE_MEMSYS3=1 \
      -DSQLITE_TEMP_STORE=3 \
      -DSQLITE_OMIT_DEPRECATED=1 \
      -DSQLITE_MAX_EXPR_DEPTH=0 \
      -DSQLITE_THREADSAFE=0 \
      -DSQLITE_OMIT_TRACE=1 -DSQLITE_OMIT_RANDOMNESS=1 \
      -DNDEBUG \
      -DSQLITE_CORE=1 \
```

## Ports: a list of applications ported to the IC

Here we list known applications and libraries ported to the IC

* SQLite

## TODO

Here is a list of things to do before the official release of Chico:

* release candidc
* support for working with stable memory
* provide a tool for porting much of the linux user space to the IC
