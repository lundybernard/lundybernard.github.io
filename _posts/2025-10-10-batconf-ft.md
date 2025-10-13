---
layout: post
title: "Supporting free-threading Python"
date: 2025-10-10 12:00:00 -0700
categories: python open-source nogil free-threading
---

How we made [BatConf](https://batconf.readthedocs.io/en/latest/#) ready for free-threading/nogil

> **TLDR**:
> BatConf is thread-safe, well tested, and ready for your free-threaded applications

## Threadsafe by design
Fortunately for us, configuration management is conceptually thread-safe.
We want to load configuration values from their source(s) once, and read them many times.
We do not expect configuration values to be written or updated at runtime.

BatConf's architecture is essentially thread-safe:
* Configuration Sources are treated as read-only
* Python itself protects us against read/write contention on variables


## Some areas for concern
"You're far too trusting," -Grand Moff Tarkin

While BatConf is conceptually and architecturally thread-safe,
it is not aggressively thread-safe.
Users are prohibited from creating attributes on a Configuration object,
updating Environment variables at runtime, 
or creating new config sources that are unsafe.

I don't think we need to be overly concerned with users creating their own
concurrency problems. There is no hidden concurrency inside our package,
so any thread safety issues should be apparent in user code.

But what if we missed something in the design, and there's some unexpected concurrency issue?


## Reasonable effort to ensure thread safety
Simply saying "It Should™ be fine" is not very satisfying.
I would much rather put some tests in place.


### 1. Run the test suite with free-threading enabled
Since we already have an excellent set of test suites, 
running them with the GIL disabled 
gives us reasonable certainty that there are no major issues.

I was able to create a new Conda environment, install `python-freethreading`,
and run our test suite without any errors!

Then we added 3.14t to our CI testing matrix on github, everything passes.
Nice!


### 2. Real threading tests
Next we added a new suite of free-threading test cases.
These tests utilize BatConf Configuration objects in multiple threads.

They demonstrate the expected and officially supported use-cases,
provide examples, and guarantee that these uses-cases work both with and without the GIL.


### 3. Running tests in parallel
Digging into the [Python Free-Threading Guide](https://py-free-threading.github.io/),
I was further assured that our pure-python package should be relatively safe, as
much more care needs to be taken with packages that include extensions in other languages.

There are several plugins for PyTest which allow us to run test cases in parallel,
including [pytest-freethreaded](https://github.com/tonybaloney/pytest-freethreaded)
and [pytest-run-parallel](https://github.com/Quansight-Labs/pytest-run-parallel).

These failed spectacularly! Generating some surprising errors, and even locking up the pytest process!

```
FAILED tests/integration/configuration_test.py::FreeFormConfigTreeTests::test_sub_configs_respect_environment_variables - KeyError: 'SHELL'
```
Wait... why is it looking for `SHELL`?  `git grep SHELL` confirms that doesn't even appear in our code base...

```
FAILED tests/example/example_test.py::CLITests::test_configuration_override_from_cli_args - AttributeError: '_patch' object has no attribute 'temp_original'
```
Errors bubbling up from the `_patch` object... uh oh.

While the package its self is thread-safe, the test suites are not.
We make extensive use of `unittest.mock.patch` to isolate objects under test from side effects,
and it is not thread-safe. 
Different threads compete to patch/unpatch the same objects.
Tests can run with patches from other tests in place.

Rewriting all our tests to be thread-safe is not on the roadmap.
Running all of our test suites together takes about 1.06s, so performance is not a concern.
Isolating test cases with mutex locks,
and refactoring the code to use more dependency injection for the sake of testing,
does not provide much value, and is unlikely to reveal thread-safety issues in the code itself.


## Conclusion
BatConf is ready for use in your free-threaded Python applications!

Free-threading in Python is the best thing since f-strings.
I can't wait to build more truly parallel Python code, 
and develop new testing techniques to guarantee the safety and reliability of that code.

~ℒ
