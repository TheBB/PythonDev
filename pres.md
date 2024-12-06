---
marp: true
theme: sintef
---

<!-- _class: title -->

# Python in 2024

Eivind Fonn


---

# The story so far

- Python package development used to suck
- A **lot** of work has been done lately to fix the issue and create better tooling
- Current status: it's not bad!
- Caveat: it's still a moving target
    - but it's slowing down


---

# Scope for today

- I would have loved to talk about all the mistakes of the past
    - but it would take a long time
- Instead, we focus on summarizing the best practices *today*
- The history lesson can wait for another time
- I have decided to not discuss CI (very much)


---

# Table of contents

1. Virtual environments and version management
2. Build backends
3. Project manager tools
4. Testing
5. Type checking
6. Linting
7. Publishing


---

<!-- _class: chapter blue -->

# Virtual environments


---

# Virtual environments (venvs)

**Virtual environment**: an isolated environment consisting of
- an installation of Python
- its built-in modules
- other installed Python packages and tools

Virtual environments ensure that a project runs with consistent set of dependencies.

**Every project** you are working on should have **its own virtual environment.**


---

# Making a venv and peeking into it

```
$ python -m venv testvenv
```

```
$ tree -L 1 testvenv
testvenv
├── bin
├── include
├── lib
├── lib64 -> lib
└── pyvenv.cfg

4 directories, 1 file
```

---

# Peeking in a venv

```
$ tree -L 1 testvenv/bin
testvenv/bin
├── activate
├── activate.csh
├── activate.fish
├── Activate.ps1
├── pip
├── pip3
├── pip3.10
├── python -> /usr/bin/python
├── python3 -> python
└── python3.10 -> python

0 directories, 10 files
```


---

# Peeking in a venv

```
$ tree -L 3 testvenv/lib
testvenv/lib
└── python3.10
    └── site-packages
        ├── _distutils_hack
        ├── distutils-precedence.pth
        ├── pip
        ├── pip-22.0.2.dist-info
        ├── pkg_resources
        ├── setuptools
        └── setuptools-59.6.0.dist-info

8 directories, 1 file
```


---

# Activating and deactivating a venv

```bash
$ source testvenv/bin/activate  # or other activation script
```

```bash
(testvenv) $ which python
./testvenv/bin/python
```

```bash
(testvenv) $ which pip
./testvenv/bin/pip
```

```
(testvenv) $ deactivate
```

```bash
$ which python
/usr/bin/python
```


---

# Venv particulars

- The Python version inside the venv is the same one you used to create it
- There's 10,000 other ways to make venvs with various tools
    - they are always *just a folder*
    - there should always be an activation script in the `bin` folder
- But `pip` may not necessarily be installed
    - it's always good to check `which pip` before `pip install`


---

# Ensuring pip

```bash
$ python -m venv testvenv --without-pip
$ source testvenv/bin/activate
```

```bash
$ python -m ensurepip --default-pip
# ...
Successfully installed pip-22.0.2
```

```bash
$ which pip
./testvenv/bin/pip
```

```bash
$ pip install --upgrade pip
# ...
Successfully installed pip-24.0
```


---

# Pyenv

Pyenv is a tool that manages multiple different Python versions.

```
$ pyenv install 3.11
$ pyenv install 3.12
$ pyenv versions
* system (set by /home/eivind/.pyenv/version)
  3.11.8
  3.12.2
```
```
$ pyenv global 3.12
$ python --version
Python 3.12.2
```


---

# Pyenv

Pyenv can set

- the "global" Python version
- the Python version for the current shell
- a Python version for a specific path, so that it always activates when you're in that location


---

# Pyenv venvs

Pyenv installations are **not** venvs. But you can use them to make venvs:

```
$ pyenv shell 3.12
$ python -m venv my-new-venv
```

Alternatively, pyenv can also create venvs

```
$ pyenv virtualenv 3.12 [name]
$ pyenv activate [name]
```

Note that pyenv maintains its venvs in its own data folder and doesn't let you place them wherever you like.


---

# Editor support

- Every serious Python editor has support for choosing which venv to use for a given project
- E.g. in VSCode: Ctrl+Shift+P and *Select Python Interpreter*
    - The VSCode Python extension will remember this setting per project
    - It even auto-detects the most common places for venvs (pyenv, conda, `.venv`, etc.)


---

<!-- _class: chapter red -->

# Build backends


---

# Building

- A Python package is typically (and preferably) distributed as a *wheel* file.
- A wheel is "built" in the sense that it contains
    - the Python source files of the package
    - compiled modules, if present
    - any data files the package may need to run
- Wheels that don't contain compiled components tend to be universal
    - `Grevling-2.0.4-py3-none-any.whl`
- Otherwise not
    - `lrsplines-1.13.0-cp310-win_amd64.whl`


---

# Building

- When you `pip install` something, *pip* tries to find a wheel file.
- Otherwise, *pip* will download the source code and try to build a wheel before installing it.
- Likewise, when you want to install projects from source (like your own), *pip* also builds a wheel first.
- The wheel is build in an isolated environment, where your normally-installed packages are *not available*.

---

# Build backends

Considering the vast variety of Python projects and all their gnarly details,
*pip* delegates the building of the wheel to a *build backend*: a Python module
that satisfies a well-defined interface.

Enter: **pyproject.toml**

```toml
[build-system]
requires = ["..."]     # packages that must be installed for the build to work
build-backend = "..."  # the actual build backend
```


---

# Build backend: setuptools

If you have a project that uses setuptools (it has a `setup.py` file):

```toml
[build-system]
requires = ["setuptools", "wheel"]
build-backend = "setuptools.build_meta"
```

It is perfectly fine to use setuptools in 2024, but you need to have a `pyproject.toml` that directs tools to use it.


---

# The TOML file

The file `pyproject.toml` has become the standardized source of project metadata in Python.

- Package name, version, maintainer, description, etc
- List of required and optional dependencies and their version bounds
- Configuration for any tool you want to use

Everything here is technically optional, *except* `build-system`

When a wheel is built, the only source of truth is the metadata files in the
wheel, and the build backend isn't *required* to honor whatever is in
`pyproject.toml`


---

<!-- _class: chapter yellow -->

# Project manager tools


---

# Project managers

A project manager is a tool that shepherds your package throughout its lifetime.
For example, it will

- build your package and publish it
- manage your dependencies
- create a virtual enviroment
- maintain a *lockfile*

You don't *need* it - everything can be done manually.

Confusingly: many project manager tools are also build backends.


---

# Project managers

- **Poetry** is popular, but
    - lacks support for some of the latest standards
    - *only* works with its own build backend
    - is very opinionated
- **PDM** is my personal recommendation
    - has none of the above issues


---

# PDM: pyproject.toml

See [writing your pyproject.toml](https://packaging.python.org/en/latest/guides/writing-pyproject-toml/).

```toml
[project]
authors = [{name = "...", email = "..."}]
requires-python = "<4,>=3.9"
dependencies = [
    "numpy<2,>=1.24",
    "pydantic<3,>=2",
]
name = "mypackage"
version = "1.2.1"
description = "Short description"
readme = "README.md"
```


---

# Optional dependencies

```toml
[project.optional-dependencies]
coolfeature = [
    "package-needed-for-cool-feature",
]
coolerfeature = [
    "package-needed-for-cooler-feature",
]
```

Then people can install those packages using

```bash
$ pip install 'mypackage[coolfeature]'
```


---

# Development dependencies

Dev dependencies aren't standardized, so they are listed under PDM's own
configuration, which falls under the `tool.pdm` namespace.

Other tools do the same thing (examples to come).

```toml
[tool.pdm.dev-dependencies]
dev = [
    "pytest<8,>=7.4.3",
    "mypy<2,>=1.7.0",
    "ruff<1,>=0.1.5",
]
```


---

# Scripts / executables

```toml
[scripts]
myprogram = "mypackage.submodule:main"
```

Then you can add a function to `mypackage/submodule.py`

```python
def func():
    print("Hello World!")
```

Result:

```
$ pip install mypackage
$ myprogram
Hello World!
```


---

# Creating the venv

```
$ pdm install --dev
```

Will create a venv in the `.venv` folder by default, including all the
development dependencies.

Fair warning: PDM does **not** install pip by default.

```
$ pdm config -g venv.with_pip true
```

Ok, now it does.


---

# Running in the venv

The `pdm run <cmd>` command runs a command *in the project venv*, even if the
venv is not currently activated in the shell. This is extremely useful for
development tools.

- Make sure all the dev tools you want are listed as dev dependencies
- Then `pdm install --dev` will install them in the venv
- Then you can `pdm run pytest`, `pdm run mypy` without worrying about venvs
    - I like to invoke things like this from a makefile
- You can even `pdm run myprogram`


---

<!-- _class: chapter green -->

# Testing


---

# Pytest

The Python industry standard for testing is *pytest*. It is very easy to
integrate: any function prefixed by `test_` in any file called `test_*.py` or
`*_test.py` is a test.

E.g. **tests/test_stuff.py**:

```python
def test_passes():
    assert 1 + 1 == 2

def test_fails():
    assert 1 + 1 == 5
```


---

# Pytest

Pytest usually requires no configuration.

```bash
# Run all tests
$ pytest

# Run a specific file
$ pytest tests/test_stuff.py

# Run a specific test
$ pytest tests/test_stuff.py::test_passes
```


---

# Pytest fixtures (setup)

```python
import pytest

@pytest.fixture
def thing():
    return setup_complicated_thing()

def test_thing_1(thing):
    assert thing.works_as_expected()

def test_thing_2(thing):
    assert thing.also_works_as_expected()
```

Pytest magically injects `thing` into the test functions as arguments.


---

# Pytest fixtures (setup and teardown)

```python
import pytest

@pytest.fixture
def thing():
    thing = setup_complicated_thing()
    yield thing
    thing.destroy_cleanly()

def test_thing(thing):
    assert thing.works_as_expected()
```


---

# Parametrized tests

```python
import pytest

CASES = [
    (1, 1, 2),
    (1, 3, 4),
    (5, 7, 12),
    (3, -1, 2),
]

@pytest.mark.parametrize("x,y,total", CASES)
def test_add(x, y, total):
    assert x + y == total
```


---

# Marking tests

```python
import pytest, os

# Skip a test
@pytest.mark.skip(reason="...")
def test_fails(): ...

# Conditionally skip a test
@pytest.mark.skipif(os.name == "Darwin", reason="Doesn't work on Apple")
def test_fails_on_apple(): ...

# Run a test but expect a fail
@pytest.mark.xfail(os.name == "Darwin", reason="Shouldn't work on Apple")
def test_probably_fails_on_apple(): ...
```


---

# Tox

Tox can run your tests under many different Python versions. This ought to do it:

```toml
[tool.tox]
legacy_tox_ini = """
[tox]
env_list = py{39,310,311,312}

[testenv]
set_env = VIRTUALENV_DISCOVERY=pyenv
deps = pdm
commands =
    pdm install --dev
    pdm run pytest
"""
```


---

# Tox

- I prefer using CI instead (then you can run with different OSes too)
- `pip install virtualenv-pyenv` to support discovering pyenv versions


---

<!-- _class: chapter purple -->

# Typing


---

# Typing in Python

Type hints in Python is possibly the most exciting recent development.

```python
a: int = 1

def mean(x: int, y: int) -> float:
    return (x + y) / 2
```

- Python is still dynamically typed: type hints are *hints* and are *ignored* by the Python interepret at runtime
- Third-party type checkers can find errors in your codebase, just like a compiler for a statically-typed language


---

# Drawbacks

- Python is a highly dynamic language
- Not every object that can be coded can have its type expressed
- But you can use it as a guideline: if you can't express the type, maybe you should simplify your code


---

# Type checkers

Use `mypy` or `pyright` to type check your code.

- `mypy` - the de-facto standard type checker sanctioned by the Python foundation
    - has permissible default settings
- `pyright` - Microsoft product
    - very strict by default
    - more capable of understanding complex structures

I ususally use Mypy.


---

# Mypy 101

- Mypy is usually smart enough to figure out the type of variables
- You need to tell it the types of function parameters and return values
- Anything not given a type will be assumed `Any` - a special type that "magically" works anywhere
- The fewer `Any` you have, the more helpful Mypy will be

```python
from typing import Any

x: Any = 1
z = x + ""    # no error
```


---

# Mypy 101

- Mypy will **not** inspect third-party imported code for correctness
- It will only use its **declared** interface to check *your* code for correctness
- Third-party modules must explicitly opt in to this
    - Just because a package uses type hints to check their own code,
      doesn't mean mypy will use those type hints as a public interface when checking *your* code
- Unfortunately, many packages are lagging behind in this regard
- This is bad because any time you import something from such packages, Mypy will treat the imported object as `Any`


---

# Stubs

A *stub* is a Python-like file with a `.pyi` extension that only contains type information.  Stubs can be found:

- alongside `.py`-files (unusual - why not just add the type hints to the `.py`-file)
- alongside compiled modules to describe their interface
- in dedicated stub packages that are explicitly for type checking
- in your Mypy stub directory

Writing your own stubs for third-party packages sounds tiresome, but is much more feasible than you think

- Usually you use only a very limited part of the full interface of a package


---

# Stubs

Example: `scipy.spatial.transform`

```python
import numpy as np
import numpy.typing as npt

class Rotation:
    @classmethod
    def from_euler(
        cls,
        seq: str,
        angles: float | npt.ArrayLike,
        degrees: bool = ...,
    ) -> Rotation: ...
    def inv(self) -> Rotation: ...
    def apply(self, vectors: npt.ArrayLike, inverse: bool = ...) -> np.ndarray: ...
```


---

# Mypy configuration

```toml
[tool.mypy]
plugins = ["numpy.typing.mypy_plugin"]  # if you use numpy
files = ["mypackage/**/*.py", "tests/**/*.py"]
mypy_path = "$MYPY_CONFIG_FILE_DIR/stubs"
disallow_untyped_defs = true
disallow_any_unimported = true
check_untyped_defs = true
warn_return_any = true
show_error_codes = true
warn_unused_ignores = true
warn_redundant_casts = true
```


---

# Mypy configuration

```toml
disallow_untyped_defs = true
```

Treats any function definition without type hints as an error.

```toml
disallow_any_unimported = true
```

Don't fall back to `Any` when importing from packages without a declared public interface. (Error instead.)


---

# Mypy configuration

```toml
check_untyped_defs = true
```

Without this, Mypy will not type-check the interior of untyped functions.

```toml
warn_return_any = true
```

Fail if the `return` expression in a function is determined to be of type `Any`.


---

# More exotic settings

```toml
disallow_any_expr = true
```

Prevent *any expression in your entire codebase* from having type `Any`.

This is **really difficult**.

```toml
disallow_any_decorated = true
disallow_any_explicit = true
disallow_any_generics = true
disallow_subclassing_any = true
```

Buy into the anti-`Any` movement to various degrees.


---

# How to write type hints

- Writing type hints is a skill that takes time to learn
    - These are just a few tips
- Be general in what you accept - be specific in what you promise
- Understand the difference between invariant, covariant and contravariant
    - T/F? `Sequence[int] <: Sequence[int | str]`
    - T/F? `list[int] <: list[int | str]`
    - T/F? `Callable[[], int] <: Callable[[], int | str]`
    - T/F? `Callable[[int], None] <: Callable[[int | str], None]`


---

# Casting and ignoring errors

- Use `cast` if you can't get Mypy to comprehend

    ```python
    from typing import cast

    cast(int, this_really_is_an_int_i_promise)
    ```

- Use ignore comments if you *really* can't get Mypy to comprehend

    ```python
    def myfunc():  # type: ignore[no-untyped-def]
        ...
    ```


---

# Overloading

If you have a function with many different calling conventions, use overload.

```python
from typing import overload

@overload
def myfunc(x: int, y: str) -> list[int]: ...

@overload
def myfunc(x: str, y: str) -> list[bool]: ...

def myfunc(x, y):  # type: ignore[no-untyped-def]
    ...
```


---

# Protocols

If you need to describe a type *structurally*, use `Protocol`

```python
from typing import Protocol

class QuacksLikeADuck(Protocol):
    def quack(self) -> str: ...

def myfunc(duckish: QuacksLikeADuck) -> None:
    duckish.quack()
```

Now `myfunc` will work with any object that has a `.quack() -> str` no matter
its actual type.


---

# Protocols as callables

Protocols are *great* for describing function-like objects with sophisticated
parameter specifications.

```python
from typing import Protocol

class Function(Protocol):
    def __call__(self, x: int, *y: str, /, p: str, z: bool = ...) -> None: ...

def myfunc(duckish: Function) -> None:
    duckish(1, "a", "b", p="lol")
```

The type `Function` cannot be described any other way.


---

# Unpack

Used for describing splatted arg/kwarg types.

```python
from typing import TypedDict, Unpack

class Kwargs(TypedDict, total=False):
    kw1: int
    kw2: str

def wrapper_function(arg1: int, arg2: int, **kwargs: Unpack[Kwargs]) -> None:
    call_inner_function(arg1 + arg2, **kwargs)
```


---

# Self

Useful for typing inheritance interfaces.

```python
from typing import Self

class SuperCls:
    def tweak(self) -> Self:
        ...

class SubCls(SuperCls):
    ...

x = SubCls().tweak()  # ==> SubCls
```


---

# Literal

Use to denote a limited set of literal values that a variable can take. It's
useful for e.g. enum-like strings. E.g. from Splipy:

```python
from typing import Literal

def circle(..., type: Literal["p2C0", "p4C1"], ...) -> Curve:
    ...
```


---

# Generics

```python
from typing import Generic, TypeVar

T = TypeVar("T", covariant=True)

class SinglyLinkedList(Generic[T]):
    value: T
    tail: SinglyLinkedList[T]

def head(node: SinglyLinkedList[T]) -> T:
    return node.value

def tail(node: SinglyLinkedList[T]) -> SinglyLinkedList[T]:
    return node.tail
```


---

# Generics (in Python 3.12)

```python
class SinglyLinkedList[T]:
    value: T
    tail: SinglyLinkedList[T]

def head[T](node: SinglyLinkedList[T]) -> T:
    return node.value

def tail[T](node: SinglyLinkedList[T]) -> SinglyLinkedList[T]:
    return node.tail
```


---

# TYPE_CHECKING

- During runtime, type annotations are disregarded and (as a rule) not even evaluated.
- Use `typing.TYPE_CHECKING` to conditionally import only when type checking.
- Can be used to avoid e.g. circular imports
    - if you have two classes whose *types* are circularly dependent but the *implementations* aren't
- Avoid doing this in the few cases where you use runtime type information!


---

# Pydantic

Extremely useful package for input validation!

```python
from pydantic import BaseModel

class MyModel(BaseModel):
    x: int
    y: str
    submodel: Other

class Other(BaseModel):
    ...

my_model = MyModel.model_validate(untrusted_input_data)
```


---

# Backward compatibility

- In general, updates to typing is the most common way to break backward compatibility
- *Most* things in `typing` can be imported from a backwards-compatible `typing_extensions` package on older Python versions.
- In particular:
    - `a | b` should be `Union[a, b]` on Python 3.9 or older
    - `a | None` should be `Optional[a]` on Python 3.9 or older
    - that means you need to `from typing import Union, Optional`
    - `Self` and `Unpack` must be imported from `typing_extensions` until you drop support for Python 3.10


---

<!-- _class: chapter dark-green -->

# Linting


---

# Linting

- A linter (like a type checker) is a static code analysis tool
- A linter aims to find *common issues* that are not necessarily *objective
  errors* but more or less likely to be problematic
- *Ruff* is a lightning-fast linter written in Rust that has re-implemented all
  the rules from all other linters known (or unknown) to humanity.


---

# Ruff configuration

```toml
[tool.ruff]
include = ["mypackage/**/*.py", "tests/**/*.py", "stubs/**/*.pyi"]

[tool.ruff.lint]
select = [
    "F", "W", "E", "I", "UP", "C4", "FA", "ISC", "ICN",
    "RET", "SIM", "TID", "TCH", "PTH", "TD", "NPY",
]
ignore = ["E741", "SIM115", "ISC001", "TD003"]
```

This is a good starting point, but [pick a set of rules best suited for your project](https://docs.astral.sh/ruff/rules/).


---

# Using ruff

Check all files

```
$ ruff check
```

Fix all auto-fixable problems

```
$ ruff check --fix
```

Fix all problems (even if risky)

```
$ ruff check --fix --unsafe-fixes
```


---

# Code style

More and more, Python projects tend to adopt the *Black* code style.

- *Black* is a Python auto-formatter that is by design not (very) configurable
- Has the benefit of freeing the programmer from
    - worrying about aesthetics
    - fighting over code style with others
- *Perfect* for no-one, but *good enough* for everyone


---

# Ruff, again

Ruff includes a formatter (which is separate from the linter) that is nearly completely Black compatible.

```toml
[tool.ruff]
line-length = 110
```

Then just...

```
$ ruff format
```


---

<!-- _class: chapter light-green -->

# Publishing


---

# Pure Python packages

Publishing pure Python packages is very easy. Make an account on [TestPyPI](https://test.pypi.org), then

```
$ pdm publish -r testpypi
```

Then make an account on [PyPI](https://pypi.org) and

```
$ pdm publish -r pypi
```

*Always publish to TestPyPI at least once before you publish to PyPI.*


---

# Compiled packages

- It's a huge challenge to build a system-specific wheel for all possible
  combinations of hardware, OS and Python version on your own computer(s)
- Use your CI runner to build them
    - only on new version tag, and only if the tests pass first
- Helpful tools:
    - [cibuildwheel](https://cibuildwheel.readthedocs.io/en/stable/) - a Python package for building Python packages in Docker containers, one for each relevant combination
    - [maturin](https://www.maturin.rs/) - the standard build backend for Rust-based Python packages is a dream to work with


---

# CI

- As a rule, I always publish from CI
- That way, I never push by mistake if tests fail
- Configure your CI to:
    - Build and publish on new tags starting with 'v'
    - Build and publish to *testpypi* when triggered manually
    - Otherwise it's hard to test your build/publish CI workflow


---

<!-- _class: chapter blue -->

# Appendix


---

# Useful links

- [Python Packaging User Guide](https://packaging.python.org/en/latest/)
- [Typing Best Practices](https://typing.readthedocs.io/en/latest/source/best_practices.html)
- [The Black Code Style](https://black.readthedocs.io/en/stable/the_black_code_style/current_style.html)


---

# Tools

- [PDM](https://pdm-project.org)
- [Pytest](https://pytest.org)
- [Tox](https://tox.wiki)
- [Mypy](https://mypy-lang.org/)
- [Pyright](https://microsoft.github.io/pyright/#/)
- [Ruff](https://docs.astral.sh/ruff/)
- [Cibuildwheel](https://cibuildwheel.readthedocs.io/en/stable/)


---

# Compiled Python modules

- [PyO3](https://pyo3.rs/)
- [Maturin](https://www.maturin.rs/)
- [Cython](https://cython.org/)
