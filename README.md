# Atheris: A Coverage-Guided, Native Python Fuzzer

Atheris is a coverage-guided Python fuzzing engine. It supports fuzzing of Python code, but also native extensions written for CPython. Atheris is based off of libFuzzer. When fuzzing native code, Atheris can be used in combination with Address Sanitizer or Undefined Behavior Sanitizer to catch extra bugs.

## Installation Instructions

Atheris supports Linux (32- and 64-bit) and Mac OS X.

### Linux

Atheris relies on libFuzzer, which is distributed with Clang. If you have a sufficiently new version of `clang` on your path, installation is as simple as:
```
pip install atheris
```

If you don't have `clang` installed or it's too old, you'll need to download and build the latest version of LLVM. Follow the instructions in Installing Against New LLVM below.

### Mac

Atheris relies on libFuzzer, which is distributed with Clang. However, Apple Clang doesn't come with libFuzzer, so you'll need to install a new version of LLVM from head. Follow the instructions in Installing Against New LLVM below.

### Installing Against New LLVM

```
# Building LLVM
git clone https://github.com/llvm/llvm-project.git
cd llvm-project
mkdir build
cd build
cmake -DLLVM_ENABLE_PROJECTS='clang;compiler-rt' -G "Unix Makefiles" ../llvm
make -j 10  # This step is very slow

# Installing Atheris
CLANG_BIN="$(pwd)/bin/clang" pip3 install atheris
```

## Using Atheris

### Example:

```
import atheris
import sys

def TestOneInput(data):
  if data == b"bad":
    raise RuntimeError("Badness!")
    
atheris.Setup(sys.argv, TestOneInput)
atheris.Fuzz()
```

Atheris supports fuzzing Python code, and uses Python code coverage information for this purpose.

### Fuzzing Python Code

While Atheris supports Python 2.7 and Python 3.3+, its Python code coverage support is *significantly better* when used with Python 3.8+, as it supports opcode-by-opcode coverage. If fuzzing Python code, we strongly recommend using Python 3.8+ where possible.

When fuzzing Python, Atheris will report a failure if the Python code under test throws an uncaught exception.

Be sure to pass `enable_python_coverage=True` as an argument to `Setup()`. You can additionally pass `enable_python_opcode_coverage=[True/False]` to turn on and off opcode coverage. Opcode coverage is typically beneficial, but may provide more performance impact than benefit on large Python projects. This option defaults to `True` on Python 3.8+, or `False` otherwise.

Opcode coverage must be enabled to support features like intelligent string comparison fuzzing for Python code.

### Fuzzing Native Extensions

In order for native fuzzing to be effective, such native extensions must be built with Clang, using the argument `-fsanitize=fuzzer-no-link`. They should be built with the same `clang` as was used when building Atheris.

The mechanics of building with Clang depend on your native extension. However, if your library is built with setuptools (e.g. `pip` and setup.py), the following is often sufficient:

```
CC="/usr/bin/clang" CFLAGS="-fsanitize=fuzzer-no-link" CXX="/usr/bin/clang++" CXXFLAGS="-fsanitize=fuzzer-no-link" pip install .
```

#### Using Sanitizers

When fuzzing a native extension, **we strongly recommend you use a sanitizer**, such as Address Sanitizer or Undefined Behavior Sanitizer. However, there are complexities involved in doing this; see [using_sanitizers.md](using_sanitizers.md) for details.

## Integration with OSS-Fuzz

Atheris is fully supported by [OSS-Fuzz](https://github.com/google/oss-fuzz), Google's continuous fuzzing service for open source projects. For integrating with OSS-Fuzz, please see [https://google.github.io/oss-fuzz/getting-started/new-project-guide/python-lang](https://google.github.io/oss-fuzz/getting-started/new-project-guide/python-lang).

## API

### Main Interface

The `atheris` module provides two key functions: `Setup()` and `Fuzz()`.

In your source file, define a fuzzer entry point function, and pass it to atheris.Setup(), along with the fuzzer's arguments (typically sys.argv). Finally, call atheris.Fuzz() to start fuzzing. Here's an example:

```
def Setup(args, callback, [optional arguments...]):
```

Configure the Atheris Python Fuzzer. You must call atheris.Setup() before atheris.Fuzz().

Args:
 - `args`: A list of strings: the process arguments to pass to the fuzzer, typically sys.argv. This argument list may be modified in-place, to remove arguments consumed by the fuzzer.
 - `test_one_input`: your fuzzer's entry point. Must take a single `bytes` argument (`str` in Python 2). This will be repeatedly invoked with a single bytes container.

Optional Args:
 - `enable_python_coverage`: boolean. Controls whether to collect coverage information on Python code. Defaults to `True`. If fuzzing a native extension with minimal Python code, set to `False` for a performance increase.
 - `enable_python_opcode_coverage`: boolean. Controls whether to collect Python opcode trace events. You typically want this enabled. Defaults to `True` on Python 3.8+, and `False` otherwise. Ignored if `enable_python_coverage=False`, or if using a version of Python prior to 3.8.

```
def Fuzz():
```

This starts the fuzzer. You must have called Setup() before calling this function. This function does not return.

### FuzzedDataProvider

Often, a `bytes` object is not convenient input to your code being fuzzed. Similar to libFuzzer, we provide a FuzzedDataProvider to translate these bytes into other input forms.

You can construct the FuzzedDataProvider with:

```
fdp = atheris.FuzzedDataProvider(input_bytes)
```

The FuzzedDataProvider then supports the following functions:

```
def ConsumeBytes(count: int)
```
Consume `count` bytes.

  
```
def ConsumeUnicode(count: int)
```

Consume unicode characters. Might contain surrogate pair characters, which according to the specification are invalid in this situation. However, many core software tools (e.g. Windows file paths) support them, so other software often needs to too.

```
def ConsumeUnicodeNoSurrogates(count: int)
```

Consume unicode characters, but never generate surrogate pair characters.

```
def ConsumeString(count: int)
```

Alias for `ConsumeBytes` in Python 2, or `ConsumeUnicode` in Python 3.

```
def ConsumeInt(int: bytes)
```

Consume a signed integer of the specified size (when written in two's complement notation).

```
def ConsumeUInt(int: bytes)
```

Consume an unsigned integer of the specified size.

```
def ConsumeIntInRange(min: int, max: int)
```

Consume an integer in the range [`min`, `max`].

```
def ConsumeIntList(count: int, bytes: int)
```

Consume a list of `count` integers of `size` bytes.

```
def ConsumeIntListInRange(count: int, min: int, max: int)
```

Consume a list of `count` integers in the range [`min`, `max`].

```
def ConsumeFloat()
```

Consume an arbitrary floating-point value. Might produce weird values like `NaN` and `Inf`.

```
def ConsumeRegularFloat()
```

Consume an arbitrary numeric floating-point value; never produces a special type like `NaN` or `Inf`.

```
def ConsumeProbability()
```

Consume a floating-point value in the range [0, 1].

```
def ConsumeFloatInRange(min: float, max: float)
```

Consume a floating-point value in the range [`min`, `max`].

```
def ConsumeFloatList(count: int)
```

Consume a list of `count` arbitrary floating-point values. Might produce weird values like `NaN` and `Inf`.

```
def ConsumeRegularFloatList(count: int)
```

Consume a list of `count` arbitrary numeric floating-point values; never produces special types like `NaN` or `Inf`.

```
def ConsumeProbabilityList(count: int)
```

Consume a list of `count` floats in the range [0, 1].

```
def ConsumeFloatListInRange(count: int, min: float, max: float)
```

Consume a list of `count` floats in the range [`min`, `max`]

```
def PickValueInList(l: list)
```

Given a list, pick a random value

```
def ConsumeBool()
```

Consume either `True` or `False`.


### Use with Hypothesis

The [Hypothesis library for property-based testing](https://hypothesis.readthedocs.io/)
is also useful for writing fuzz harnesses.  As well as a great library of "strategies"
which describe the inputs to generate, using Hypothesis makes it trivial to reproduce
failures found by the fuzzer - including automatically finding a minimal reproducing
input.  For example:

```
import atheris
from hypothesis import given, strategies as st

@given(st.from_regex(r"\w+!?", fullmatch=True))
def test(string):
  assert string != "bad"

atheris.Setup(sys.argv, test.hypothesis.fuzz_one_input)
atheris.Fuzz()
```

[See here for more details](https://hypothesis.readthedocs.io/en/latest/details.html#use-with-external-fuzzers),
or [here for what you can generate](https://hypothesis.readthedocs.io/en/latest/data.html).

