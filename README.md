# `thinking` framework

[![thinking-modules](https://badgen.net/static/thinking/modules/lightgray?icon=github)](https://github.com/FilipMalczak/thinking-modules)
[![thinking-runtime](https://badgen.net/static/thinking/runtime/lightgray?icon=github)](https://github.com/FilipMalczak/thinking-runtime)
[![thinking-tests](https://badgen.net/static/thinking/tests/lightgray?icon=github)](https://github.com/FilipMalczak/thinking-tests)
[![thinking-injection](https://badgen.net/static/thinking/injection/lightgray?icon=github)](https://github.com/FilipMalczak/thinking-injection)

## Story time

For a while now I'm working on a project I keep calling `thinker`. It's an AI
project that glues together different areas in non-standard way. When working
on it I've written some reusable meta code, like fluent testing, DI, runtime
preconfiguration and utility to write code that can be interrupted and rerun
from latest succesful point with ease.

I've reached the point where the auxiliary codebase was bigger than the code
that implemented the idea behing `thinker` itself. Since it made further writing
harder and less focused (and since it could be useful in the future and not only
to me) I've decided to take a break and extract the pieces into an external
framework.

Since the original project was `thinker`, the framework that stemmed from it
is called `thinking`. And that's where you are right now.

## What projects can this be useful for?

Initially this was written when developing a long-running, multi-stage machine
learning experiment. The python code was running in a single thread in a single
process - but it didn't limit the usage of threading or multiprocessing by
external libraries (straight from the python, or lower, via C extensions or CUDA).

Anything that boils down to "start running, do some stuff, achieve result, finish"
is a suitable candidate for `thinking`. Thus, web apps, servers, etc won't
probably benefit from using it. Its disputable whether its a good tool for
cloud functions like AWS Lambda, since the overhead may be significant in that
case (I expect my program to run for hours, days or even weeks on end, so I can
spare 2-3 seconds of startup; that may not be the case for cloud functions).

In case of ML experiments it should be pretty easy to divide them into 
stages that are ultimately pretty long, but short in context of the whole program.
For example, if you're training new model (NN, embeddings, etc), you usually
do it iteratively, in epochs. Each epoch (or a bunch of them) may take an hour, and 
you may wanna have a hundred of iterations like that. Imagine that after 99h you
have power outage. That would hurt, wouldn't it? Yeah, I've been there, so I wrote
a piece that can track what already happened and skip the executed stages, so that 
in the described case if you'd rerun the program, it would only run for an hour 
or two, starting off from the last "commited" epoch.

If your program can be split like that, you can also benefit from `thinking`.
Mind you, it doesn't have to be ML - processing large quantities of data,
performing actions across multiple throttled APIs, etc can fit these characteristics as well.

> All that I've described is already written, but not all of it is extracted from
> `thinker`. I'm trying to make the features opt-in, so I'm extracting project
> after project. Notably, the "persistent executor" feature is not extracted yet.
> 
> Besides simply extracting the code, I'm enhancing it to make it cleaner, neater
> and generally more useful. At the same time, I'm trying to test it better
> than it was tested when it wasn't the sole focus.

## Day-to-day with `thinking`

This framework makes some assumptions about how you write and organize code. That's
what this section is about.

Your code is organized as follows:

 - standard project files - `README`, `LICENSE`, but also `.gitignore`, `pyproject.toml`, `requirements.txt`, etc
 - `venv` - virtual envs are recommended, though optional; it's gonna make your life easier, `thinking` or not
 - `pkg` - any number of them; most of you code resides in packages; there is no `src` holder directory or anything like that
   - `__init__.py`
   - `something.jpg` - you can include non-code resources in the package structure
   - ...
 - `test` or `tests` - conventionally, but nothing stops you from having both, adding `integration_test`, etc
   - `__init__.py` - your tests are residing within a package as well
   - `run_all.py`
   - `test_something.py`
   - ...
   - `subpkg` - tests can have hierarchies
     - `__init__.py`
     - `test_some_other_thing.py`
     - ...
     - `subsubpkg` - ... of any depth
       - `__init__.py`
     - `run_all.py`
- `app.py` - optional, entrypoint to your program
- `__logs__.py` - where you configure logging
- `__dbs__.py` - where you configure databases
- `__whatever__.py`, `__config__.py`, etc - generally, you keep conigs in the root of the repo

You usually use `python -m ...` to run stuff. The exception is the entrypoint to the program, where it doesn't
really matter if you do `python app.py` or `python -m app`.

If you have more than one entrypoint, you probably omit the `app.py` and get things started with `python -m pkg.mod`.

When you write code, you don't want to glue it together manually. You envy Java/Spring folk that they write classes,
mark their requirements and the tooling puts them together. You expect the program to know about the scope of your code
and be able to report what are the known types and their implementations. You use `typing.Protocol`s and `ABC`s to
easily write against abstractions/interfaces and not implementations and their details.

You run all the tests in your repo with `python -m test.run_all`. To run a subset of tests you run 
`python -m test.subpkg.run_all` which will run the tests from `test.subpkg` package (as well as its subpackages, recursively).
This is essentially the same behaviour as with `test.run_all`, so 
If you want to run only specific test suite, you can run a single module too `python -m test.test_something` and execute
all the tests defined in that module - and nothing else.

You expect the framework to provide features out of the box. You don't wanna configure XML unittest reports manually,
nor do you want to care about coverage configs. You still wanna be able to opt-out of them.

When you distribute your app (be it as package you put on PyPI and configure to have scripts, or as a Docker image, or
whatever you fancy), you may wanna exclude `<repo>/__whatever__.py` config files. The framework should provide usable
defaults for these aspects of your app or at least complain about missing required config files. Of course, you can
include them in your distribution, but that will force consumers to use the configs you provide.

### Full, believable example

This is how an overengineered calculator app may look like. For the sake of the example, let's assume that it has 2 
entrypoints - one that reads the expressions from STDIN ("interactive" mode), the other that takes it as CLI arguments.

> Let's ignore the fact that CLI could be unified for both these modes or that passing values like `1 * 2` via CLI
> can be tricky. This is an example.

Your repo would look like this:

 - `README.md`
 - `LICENSE`
 - `.gitignore`
 - `pyproject.toml`
 - `requirements.txt`
 - `__log__.py`
 - `__test__.py`
 - `calculator`
   - `__init__.py`
   - `__main__.py`
   - `cli`
     - `__init__.py`
     - `parser.py`
     - `start.py`
   - `interactive`
     - `banner.txt`
     - `__init__.py`
     - `start.py`
   - `expression`
     - `__init__.py`
     - `model.py`
     - `parser.py`
     - `evaluator.py`
 - `tests`
   - `__init__.py`
   - `expression`
     - `__init__.py`
     - `run_all.py`
     - `test_parser.py`
     - `test_evaluator.py`
   - `test_cli_parser.py`
   - `run_all.py`

You run the app as `python -m calculator`.

When you're working on expressions evaluator, you probably cycle over `python -m test.expression.test_evaluator`.
Once you're done, but you wanna do some regression, to see if the changes you made didn't break other expression-related
code, so you run `python -m test.expression.run_all` and run all tests related to expressions (but not CLI parsing).
Before commiting you do a full suite with `python -m test.run_all`.

> Similar, but smaller example of RPN calculator already exists as [part of test suite for `thinking-injection`](https://github.com/FilipMalczak/thinking-injection/tree/master/calculator).

## Family of `thinking` projects

 - [![thinking-modules](https://badgen.net/static/thinking/modules/lightgray?icon=github)](https://github.com/FilipMalczak/thinking-modules)
   - [![PyPI version](https://badge.fury.io/py/thinking-modules.svg)](https://badge.fury.io/py/thinking-modules)
   - [![codecov](https://codecov.io/github/FilipMalczak/thinking-modules/graph/badge.svg?token=KFQ1DJQMWF)](https://codecov.io/github/FilipMalczak/thinking-modules)
   - at the bottom of the stack lies the library that models module names and modules themselves, allowing for a scan
     of the codebase, figuring out parent packages of modules and packages, placement in the filesystem, recognizing
     the root directory of the project, etc
 - [![thinking-runtime](https://badgen.net/static/thinking/runtime/lightgray?icon=github)](https://github.com/FilipMalczak/thinking-runtime)
   - [![PyPI version](https://badge.fury.io/py/thinking-runtime.svg)](https://badge.fury.io/py/thinking-runtime)
   - this library provides generic framework for configuring runtime-wide parts of code
   - examples that are provided out-of-the-box are
     - logging configuration
     - recognition of runtime mode (app/tests, but also facets, which in Java/Spring world you'd likely call profiles)
   - this is the part that processes `__logging__`, `__dbs__`, etc files
   - it allows for installing your own bootstrap actions
   - basically, if you want to have stuff happen before everything else, this is the tool for you
 - [![thinking-tests](https://badgen.net/static/thinking/tests/lightgray?icon=github)](https://github.com/FilipMalczak/thinking-tests)
   - [![PyPI version](https://badge.fury.io/py/thinking-tests.svg)](https://badge.fury.io/py/thinking-tests)
   - not a testing framework per se, more like a handy frontend for one
   - uses existing frameworks as the backend (`unittest` out of the box, but allows for developers to integrate others)
   - provides decorator-based API to declare tests, as well as test aspect (interceptor) framework, with defaults
     that track and expose test case metadata
   - uses `thinking-runtime` to configure testing
   - besides exposing the API, provides backend-agnostic utilities for test reporting (in JUnit XML format, as well
     as HTML reports) and coverage measurements and reporting (delegating the work to [`coverage`](https://coverage.readthedocs.io/))
   - this is the part that handles `run_all.py` files
   - ideally, a one-stop shop for common testing practices without the "wrap the test case in a class" or "install this 
     specific set of libraries to have XML reports" ceremony
 - [![thinking-injection](https://badgen.net/static/thinking/injection/lightgray?icon=github)](https://github.com/FilipMalczak/thinking-injection)
   - [![PyPI version](https://badge.fury.io/py/thinking-injection.svg)](https://badge.fury.io/py/thinking-injection)
   - [![codecov](https://codecov.io/github/FilipMalczak/thinking-injection/graph/badge.svg?token=X5HGHMQXAP)](https://codecov.io/github/FilipMalczak/thinking-injection)
   - simply a DI framework
   - as opposed to [other python DI frameworks](https://stackoverflow.com/questions/156230/python-dependency-injection-framework)
     relies on code scanning, typing annotations and auto-recognition of injectable stuff instead of decorators and config
     classes
   - while `thinking-runtime` lets you easily define "how" does the app run, this aims to ease the definition of "what" runs
     in the app
   - uses `thinking-modules` to scan the code structure (as in, modules and packages, and not classes and functions)
   - doesn't only handle injecting stuff into other stuff, but makes sure that the lifecyle of managed objects
     is taken care of
   - is foundational for the future utilities
   - will usually be crucial to your entrypoints like `app.py`
 
### Future projects

These are the projects that wait to be extracted from `thinker`. While they don't have their own repos just yet, the
main portions of their code is already written and waits for prerequisites to be finalized.

These prerequisites are usually `thinking-runtime` (which is pretty stable now) and `thinking-injection` (which still
lacks some more complicated features, notably fallback implementations).

Names and scopes of projects are subject to change, these are rough plans.

- `thinking-embedded-dbs`
  - will provide defaults configs (as well as customizable configs) for several embedded DBs like `tinydb`, `lancedb` and `zodb`
  - will automatically make them available in DI
  - will be able to recognize if given DB is installed and omit any actions and declarations if not
  - may provide transactionality of sorts to the non-transactional DBs, but that part may be extracted in 
    the `thinking-persistent-executor`
- `thinking-persistent-executor`
  - this will be the part that will track executed stages of your program and allow for recovery
  - will probably require `thinking-embedded-dbs`, but should define protocol, so that you can switch the persistence
    backend
  - will be based on the tree of tasks
    - task can be either stage or step
    - stage is a linear container of other tasks and cannot modify any data (to some sane limits, e.g. it won't stop
      you from creating files, but it will be discouraged; it will recognize DB modifications though, and error out
      in such cases)
    - step is an actual action, which is free to modify the data
  - tasks will be declarable as decorated functions, but more imperative API will exist, to allow for iterating and
    parametrization
  - besides being able to recover from interrupted runtime, it should be able to roll back the DB to the past state,
    either in read-only mode (for analysis) or write-again mode (in case you figure that you've made a wrong choice, 
    but instead of starting the whole process again, you wanna start from intermediate point)
    - this may be out of MVP, but the draft of that already exists
- `thinking-injecting-executor`
  - will extend previous project with possibility to invoke stages with arguments injected by the DI context
  - will spare you ceremonial code that only asks the context for the arguments of the stage/step