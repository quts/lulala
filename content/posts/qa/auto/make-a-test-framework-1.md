---
date: "2025-05-01T12:34:56+08:00"
draft: false
title: "Make a Test Framework [1/1]"
---
# Background
There was a side project within our team focused on designing a new test framework that would allow others to easily add new test cases without needing to understand the underlying workings of the framework. The framework was also intended to be deployable across various platforms without requiring any toolchain setup in advance.

Some compiled languages came to mind—Go in particular. Thanks to its static compilation and straightforward build process, we could deploy binaries to almost any platform in our company. It also happened to be my first project using Go, which made the experience even more exciting.

While there are existing options like `testing` and `testify`—both popular in Go projects—we found it somewhat difficult to get `go test` running reliably across different platforms. In the end, we decided to build our own solution. Since it was just a side project with no users and an uncertain future, we figured that keeping things flexible wouldn’t hurt. Who knows—maybe it could even evolve into a new product for our company someday.

In this article, I’m going to revisit the key lessons I learned while designing that test framework. Although the original project was written in Go, for the purposes of this article, I’ll be using Python instead.

# How Does It Work in Other Projects?
As a software engineer who focuses on testing with Python—like me—you’ve probably come across `pytest`, one of the most widely used testing frameworks in the Python world.

Let’s say you wrote a simple test case like this:
```py
# test_math.py

def add(a, b):
    return a + b

def test_add_success():
    # This test should pass
    assert add(2, 3) == 5

def test_add_fail():
    # This test is expected to fail
    assert add(2, 2) == 5
```
When you run this test, you’ll see output similar to the following:
```sh
============================= test session starts =============================
collected 2 items

test_math.py .F                                                        [100%]

================================== FAILURES ===================================
_______________________________ test_add_fail ________________________________

    def test_add_fail():
>       assert add(2, 2) == 5
E       assert 4 == 5
E        +  where 4 = add(2, 2)

test_math.py:10: AssertionError
=========================== short test summary info ============================
FAILED test_math.py::test_add_fail - assert 4 == 5
```
Pretty neat, right? But it also raises a question: How does `pytest` know all this?
- How does it know something went wrong and display such detailed error information?
- How does it count how many test cases passed or failed?
- Is there anything better than `assert`?

## How Does It Detect Failures?
Using `assert` to detect a failed test case in `pytest` is a simple and straightforward approach. pytest handles `AssertionError` by printing the corresponding call stack. Modern languages like Python and Go provide rich libraries to retrieve the call stack during exception handling.

In other words, as long as you follow the pattern below, you’ve already implemented the essence of a basic test framework:
```py
import traceback
import inspect
import sys

def add(a, b):
    return a + b

def test_add_success():
    assert add(2, 3) == 5

def test_add_fail():
    assert add(2, 2) == 5

def runner():
    failed_tests = []
    test_cases = [test_add_success, test_add_fail]

    for test_case in test_cases:
        try:
            test_case()
            print(f"✅ {test_case.__name__} passed")
        except AssertionError as e:
            filename = inspect.getsourcefile(test_case)
            _, lineno = inspect.getsourcelines(test_case)

            print(f"\n{'_' * 30} {test_case.__name__} {'_' * 30}\n")
            tb = traceback.TracebackException(*sys.exc_info())
            for line in tb.stack.format():
                if filename in line:
                    print(line, end="")
            print(f"E       {e.__class__.__name__}: {e}")
            print(f"\n{filename}:{lineno + 1}: {e.__class__.__name__}")
            failed_tests.append(test_case.__name__)

    if failed_tests:
        print("\n=========================== short test summary info ===========================")
        for name in failed_tests:
            print(f"FAILED {inspect.getsourcefile(globals()[name])}::{name} - test failed")

runner()
```

## How to count?
Once you understand how a test framework handles `assert`, counting passed or failed cases becomes straightforward. Just maintain counters outside the loop and increment them accordingly:
```py
def runner():
    n_fails = 0
    n_passes = 0
    n_total = 0

    for test_case in test_cases:
        n_total += 1
        try:
            test_case()
            n_passes += 1
        except AssertionError as e:
            # ...
            # assertion handle
            # ...
            n_fails += 1

    print(f"{n_total} executed, {n_passes} passed, {n_fails} failed")
```

## Anything Better Than `assert`?
To be honest, I’m not a big fan of using `assert` in test cases. Under the hood, `assert` simply raises an exception and leaves the function—leaving `pytest` to handle the rest.

But this has downsides: `assert 4 == 5` behaves the same as `assert False`. Worse yet, I have to mentally evaluate whether 4 equals 5 while reading the code. That’s not ideal.

As a test engineer, I’d prefer something more explicit and readable—like `check.equal(2 + 2, 5)` from the `pytest-check` package. It’s clearer and lets me quickly see what is being tested without mentally parsing boolean logic.

In this article, we’ll use a simple wrapper function called `pass_on_equal()` around `assert` to improve readability. It makes each test case easier to understand at a glance. In the next article, we’ll explore how `pytest-check` actually implements similar behavior.
```py
def add(a, b):
    return a + b

def test_add_success():
    pass_on_equal(add(2, 3), 5)

def test_add_fail():
    pass_on_equal(add(2, 2), 5)

def pass_on_equal(a, b):
    assert a == b
```

# Conclusion
Congratulations — we’ve finished building a very simple test framework! You now know how to test the `add()` function using structured test functions.
```py
import traceback
import inspect
import sys

def add(a, b):
    return a + b

def test_add_success():
    pass_on_equal(add(2, 3), 5)

def test_add_fail():
    pass_on_equal(add(2, 2), 5)

def pass_on_equal(a, b):
    assert a == b

def runner():
    n_fails = 0
    n_passes = 0
    n_total = 0
    failed_tests = []
    test_cases = [test_add_success, test_add_fail]

    for test_case in test_cases:
        n_total += 1
        try:
            test_case()
            print(f"✅ {test_case.__name__} passed")
            n_passes += 1
        except AssertionError as e:
            filename = inspect.getsourcefile(test_case)
            _, lineno = inspect.getsourcelines(test_case)

            print(f"\n{'_' * 30} {test_case.__name__} {'_' * 30}\n")
            tb = traceback.TracebackException(*sys.exc_info())
            for line in tb.stack.format():
                if filename in line:
                    print(line, end="")
            print(f"E       {e.__class__.__name__}: {e}")
            print(f"\n{filename}:{lineno + 1}: {e.__class__.__name__}")
            failed_tests.append(test_case.__name__)
            n_fails += 1

    if failed_tests:
        print("\n=========================== short test summary info ===========================")
        for name in failed_tests:
            print(f"FAILED {inspect.getsourcefile(globals()[name])}::{name} - test failed")

    print(f"{n_total} executed, {n_passes} passed, {n_fails} failed")

runner()
```
Wait — so if a user wants to add a new test case, they have to manually modify the `test_cases` list inside the `runner()` function?
If that’s the case, can we really call this a framework?
That’s a great question — and we’re going to explore the answer in the next article.
