# A Simple Test Framework

A simple test framework for language C, inspired by [Check](https://libcheck.github.io/check/) & [Google Test](https://github.com/google/googletest).

The main features are:

* test framework for C/C++ projects
* a simple framework written in C, with few files to include in your project
* framework developed in C on Linux (based on POSIX standard)
* easy way to integrate your tests, just by adding some *test_\*()* functions
* support test with *argv* arguments
* all tests are executed sequentially (one by one)
* different execution modes: "fork", "thread" or "nofork"

## Compilation

Our test framework is made up of two libraries:

* *libtestfw.a*: all the basic routines to discover, register and run tests (see API in [testfw.h](testfw.h)).
* *libtestfw_main.a*: a *main()* routine to launch tests easily (optionnal).

You can compile it by hand quite easily.

```bash
gcc -std=c99 -Wall -g   -c -o testfw.o testfw.c
ar rcs libtestfw.a testfw.o
gcc -std=c99 -Wall -g   -c -o testfw_main.o testfw_main.c
ar rcs libtestfw_main.a testfw_main.o
```

Or if you prefer, you can use the CMake build system ([CMakeLists.txt](CMakeLists.txt)).

```bash
mkdir build ; cd build
cmake .. && make
```

## Writing a First Test

Adding a test *hello* in a suite *test* (the default one) is really simple, you just need to write a function *test_hello()*
with the following signature ([hello.c](hello.c)).

```c
#include <stdio.h>
#include <stdlib.h>

int test_hello(int argc, char* argv[])
{
    printf("hello world\n");
    return EXIT_SUCCESS;
}
```

A success test should allways return EXIT_SUCCESS. All other cases are considered as a failure. More precisely, running a test returns one of the following status:

* SUCCESS: return EXIT_SUCCESS or 0 (normal exit)
* FAILURE: return EXIT_FAILURE (or any value different of EXIT_SUCCESS)
* KILLED: killed by any signal (SIGSEGV, SIGABRT, ...)
* TIMEOUT: after a time limit, return an exit status of 124 (following the convention used in *timeout* command)

Compile it and run it.

```bash
$ gcc -std=c99 -Wall -g -c hello.c
$ gcc hello.o -o hello -rdynamic -ltestfw_main -ltestfw -ldl -L.
$ ./hello
hello world
[SUCCESS] run test "test.hello" in 0.46 ms (status 0, wstatus 0)
```

And that's all!

## Running Tests

Let's consider the code [sample.c](sample.c). To run all this tests, you need first to compile it and then to link it against our both libraries.

```bash
gcc -std=c99 -Wall -g -c sample.c
gcc sample.o -o sample -rdynamic -ltestfw_main -ltestfw -ldl -L.
```

The '-rdynamic' option is required to load all symbols in the dynamic symbol table (ELF linker).

Then, launching the main routine provide you some helpful commands to run your tests. Usage:

```text
Usage: ./sample [options] [actions] [-- <testargs> ...]
Actions:
  -x: execute all registered tests (one by one, sequentially)
  -l: list all registered tests
Options:
  -r <suite.name>: register a function "suite_name()" as a test
  -R <suite>: register all functions "suite_*()" as a test suite
  -o <logfile>: redirect test output to a log file
  -O: redirect test stdout & stderr to /dev/null
  -t <timeout>: set time limits for each test (in sec.) [default 2]
  -T: no timeout
  -c: return the total number of test failures
  -s: silent test output
  -m <mode>: set execution mode: "fork"|"thread"|"nofork" [default "fork"]
  -S: full silent mode (not only tests)
  -v: print version
  -h: print this help message
```

List all available tests in the default suite (named *test*):

```bash
$ ./sample -l
test.alarm
test.args
test.assert
test.failure
test.infiniteloop
test.segfault
test.sleep
test.success
```

To use another suite, use '-r/-R' options:

```bash
$ ./sample -R othertest -l
othertest.failure
othertest.success
```

Run your tests with some options (timeout = 2 seconds, log file = /dev/null):

```bash
$ ./sample -t 2 -O -x
[KILLED] run test "test.alarm" in 1000.70 ms (signal "Alarm clock", wstatus 14)
[SUCCESS] run test "test.args" in 0.59 ms (status 0, wstatus 0)
[KILLED] run test "test.assert" in 0.58 ms (signal "Aborted", wstatus 6)
[FAILURE] run test "test.failure" in 0.43 ms (status 1, wstatus 256)
[TIMEOUT] run test "test.infiniteloop" in 2000.23 ms (status 124, wstatus 31744)
[KILLED] run test "test.segfault" in 0.17 ms (signal "Segmentation fault", wstatus 11)
[TIMEOUT] run test "test.sleep" in 2000.56 ms (status 124, wstatus 31744)
[SUCCESS] run test "test.success" in 0.57 ms (status 0, wstatus 0)
```

Run a single test in *fork* mode (default):

```bash
$ ./sample -m fork -r test.failure -x
[FAILURE] run test "test.failure" in 0.43 ms (status 1, wstatus 256)
$ echo $?
0
```

In the *fork* mode, each test is runned separately in a child process. The failure of a test will not affect the execution of following tests.

Now, let's run a single test in *nofork* mode:

```bash
$ ./sample -m nofork  -r test.failure -x
[FAILURE] run test "test.failure" in 0.01 ms (status 1, wstatus 256)
$ echo $?
1
```

In the *nofork* mode, each test is runned *directly* as function call (without fork). As a consequence, the first test that fails will interrupt all the following.  It is especially useful when running all tests one by one within another test framework as CTest. See [CMakeLists.txt](CMakeLists.txt).

And running tests.

```bash
cmake . && make && make test
```

You can also pass arguments à la *argv* s follows.

```bash
$ ./sample -r test.args -- a b c
argc: 3
argv: a b c
[SUCCESS] run test "test.args" in 0.45 ms (status 0, wstatus 0)
```

## Main Routine

A *main()* routine is already provided for convenience in the *libtestfw_main.a* library, but it could be useful in certain case to write your own *main()* routine based on the [testfw.h](testfw.h) API. See [sample_main.c](sample_main.c).

```c
#include <stdlib.h>
#include <stdbool.h>
#include "testfw.h"
#include "sample.h"

#define TIMEOUT 2
#define LOGFILE "test.log"
#define SILENT false

int main(int argc, char *argv[])
{
    struct testfw_t *fw = testfw_init(argv[0], TIMEOUT, LOGFILE, SILENT);
    testfw_register_func(fw, "test", "success", test_success);
    testfw_register_symb(fw, "test", "failure");
    testfw_register_suite(fw, "othertest");
    testfw_run_all(fw, argc - 1, argv + 1, TESTFW_FORK);
    testfw_free(fw);
    return EXIT_SUCCESS;
}
```

Compiling and running this test will produce the following results.

```bash
$ gcc -std=c99 -rdynamic -Wall sample.c sample_main.c -o sample_main -ltestfw -ldl -L.
$ ./sample_main
[SUCCESS] run test "sample.success" in 0.41 ms (status 0, wstatus 0)
[FAILURE] run test "sample.failure" in 0.52 ms (status 1, wstatus 256)
[FAILURE] run test "othersample.negret" in 0.52 ms (status 255, wstatus 65280)
[FAILURE] run test "othersample.posret" in 0.52 ms (status 2, wstatus 512)
```

---

aurelien.esnard@u-bordeaux.fr