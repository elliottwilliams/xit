# xit, a unit testing framework for Xinu

**xit** ("**xi**nu" + "**t**est") is a C-based unit testing library for [Xinu][xinu], an
embedded operating system developed at Purdue University. xit provides the
following functionality:

- [x] Organization of tests into suites and test cases.
- [x] Assertions for different data types, with useful, customizable, error
  reporting.
- [x] System function mocking through the [fff][fff] library.
- [x] Cleanup and teardown functions per-test.
- [x] Time measurement. xit will keep track of how long tests run, and kill
  tests that take too long.
- [x] Automatic discovery, compilation, and declaration of tests.
- [x] Integration into the Xinu compilation infrastructure. (`make test` is all
  you need!)

Features slated for development:

- [ ] Test preconditions. This enables a way of determining if certain tests
  should run. For CS 636, the class I wrote xit for, I'll be able to use
  preconditions to exclude some non-router tests from router hosts, and vice
  versa.
- [ ] Network-based distributed testing. Using infrastructure of the [xinu
  lab][lab], xit will be able to spin up multiple Xinu backends and run a
  coordinated sequence of tests across them.  This should be useful for
  integration testing, where behavior of the entire networking stack is
  observed.
  
[xinu]: http://www.xinu.cs.purdue.edu
[lab]: http://www.xinu.cs.purdue.edu/#lab


## Examples

See [`examples/`](examples) for an example suite that tests
the behavior of the `gettime` system function.


## Installation

1. Clone this repository into your Xinu source tree. (I use a directory
   `test/`, but anywhere is sufficient.) Use a submodule to keep track of your
   xit version within your Xinu repo:

  ```sh
  git submodule add https://github.com/elliottwilliams/cs636_test.git test
  ```

2. Include xit's makefile extension into Xinu's `Makefile`. Add this to the
   bottom of `compile/Makefile` (substitute the directory xit is cloned
   into):

  ```make
  -include ../test/xinu/compile.mk
  ```

3. Edit your `main.c` code to start a test runner on startup, like this:

  ```c
  #include <xinu.h>

  // Add these lines near the top of the file:
  #ifdef TESTS_ENABLED
  #include <test/test.h>
  #endif

  // ...
  process main(void)
  {
    // Add these lines to main():
  #ifdef TESTS_ENABLED
    resume(create(local_test_runner, INITSTK, INITPRIO, "local_test_runner",
    0));
    return OK;
  #endif

    // ...
  }
  ```

4. Create a `tests/` directory in the root of your source tree, and write tests
   in it. See the section on writing tests below. 


# Running tests

1. Compile Xinu with tests enabled by running `make test` from the `compile/`
   directory. This will discover test functions in `tests/`, compile them,
   and link to the rest of the xinu codebase. Mock functions declared as
   [wraps][wrap] will be discovered automatically by the linker.

2. To run only a subset of your tests, run `make` with a `TESTS` variable 
   containing space-separated paths to match. Paths are matched starting at 
   the root `tests/` directory. For example, given a `tests` directory:
   
      ```
      tests
      ├── config
      │   └── fakes.def.h
      ├── integration
      │   └── net
      │       └── nd_rsol_events.c
      └── unit
          ├── net
          │   ├── colhex2ip.c
          │   ├── common.c
          │   ├── common.h
          │   ├── ...
          │   └── nd_rsol.c
          └── system
              └── defer.c
      ```
     Running `make test TESTS=unit/system` builds Xinu with the one test in `tests/unit/system`.  
     Running `make test TESTS=unit` builds Xinu with all tests in `tests/unit`.  
     Running `make test TESTS=unit/net/nd_rsol` builds Xinu with just `tests/unit/net/nd_rsol.c`

The resulting `xinu.xbin` image can be used to boot up a Xinu backend as
usual, and will run all tests specified.

[wrap]: #mock-functions


# Writing tests

Test suites are C source files that import the following headers:

- `#include <test/test.h>` for test macros and data structures
- `#include <test/assert.h>` for the assertion library
- `#include <test/fake.h>` for access to [defined mock functions][mock]

[mock]: #mock_functions

### Basic tests

Each suite should contain one or more test cases, declared with the `TEST`
macro. For example:

```c
TEST(sky_is_blue) {
  // Set up any values to test
  char * color = getskycolor();

  // Perform assertions
  assert_str_eq(color, "blue");
}
```

Inside the expansion of the `TEST` macro, a `test_t` structure is defined
corresponding to the test case's name, and the test case's function signature
is inserted.

The `assert_*` macros check their arguments, and return a failure result to the
test runner if the assertion failed. By default, failures are printed out to
the xinu console. A list of [all available assertions][assertions] is provided
below.

[assertions]: #assertions

####  Test options

The second parameter of the `TEST` macro specifies options, passed in
[structure initializer syntax][gcc-init].

The currently supported options are:

- **_`before`_**  
  `TEST(test_name, .before = before_fn) { /* ... */ }`  
  Run `before_fn` in the same process, immediately before the test procedure.
- **_`after`_**  
  `TEST(test_name, .after = after_fn) { /* ... */ }`  
  Run `after_fn` in the same process, immediately after the test procedure.

[gcc-init]: https://gcc.gnu.org/onlinedocs/gcc/Designated-Inits.html

### Assertions

The `test/assert.h` header defines the following assertion macros:

- **_`assert(cond)`_**   
  Passes if `cond != 0`.
- **_`assert_eq(lhs, rhs)`_**  
  Passes if `lhs == rhs`.
- **_`assert_eq_fmt(lhs, rhs, fmt)`_**  
  Passes if `lhs == rhs`. Uses format string `fmt` to generate a failure
  message.
- **_`assert_str_eq(lhs, rhs)`_**  
  Passes if `lhs` and `rhs` are determined equivalent by `strcmp`.
- **_`assert_in_range(expected, actual, tolerance)`_**  
  Passes if `expected` = `actual` ± `tolerance`.
- **_`assert_mem_eq(expected, actual, size)`_**  
  Passes if the first `size` bytes of `expected` and `actual` are equivalent. 
    - The failure message for this assertion depends on hex dump functions I've
      not implemented yet.


### Mock functions

xit allows functions called within the OS to be monitored and overridden at
runtime. To see these "fake functions" in action, look at
`examples/test_gettime.c`, and see their listing in `config/fakes.def.h`.

#### Background

xit uses [fff][fff], a C microframework for creating "fake functions", which
record information about calls made to them. It combines this library with ld(8) wraps, which
rewrite all references for a symbol *fn* to *__wrap_fn* at link time. 

xit mocks functions defined in `config/fakes.def.h`. For example, for the line:

    F(__wrap_icmp6in,  __real_icmp6in,  struct netpacket *, struct ip6packet *, struct icmp6msg *); \

xit will:

- Create a structure `__wrap_icmp6in_fake`, which records calls made to
  `__wrap_icmp6in`. The schema of this structure is given in 
  [fff's documentation][fake].
- Create a function `__wrap_icmp6in` that updates the `__wrap_icmp6in_fake`
  struct when called.
- Proxy calls to `__wrap_icmp6in` to `__real_icmp6in`. This allows fakes to be
  defined without affecting OS behavior. To proxy to a different function,
  overwrite `__wrap_icmp6in_fake.custom_fake` in a test.
  - Due to ld(8) wrapping, `__real_NAME` references the original symbol
    wrapped. So if `icmp6in` is wrapped, references in code to `icmp6in` go to
    `__wrap_icmp6in`, but `__real_icmp6in` goes to the original symbol.

[fff]: https://github.com/meekrosoft/fff
[fake]: https://github.com/meekrosoft/fff#hello-fake-world


#### Faking a system function

Add the function and its signature to `config/fakes.def.h`. See the
documentation and example in that file. 


#### Using a faked function

1. Ensure your test suite includes `<test/fake.h>`.

2. In a test case, call a function which calls the function being faked. The
   fake function will be used automatically.

3. Inspect the call history of the fake function by looking at its `_fake`
   struct. For example, the if `__wrap_ip6in` is faked, it's struct is
   `__wrap_ip6in_fake`. See [fff's documentation][fff] for details.
