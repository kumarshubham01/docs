# Custom Graders

# Custom Grader Behaviour - `grader`

An `init.yml` object can contain a top-level `custom_judge` node, which contains a path to a Python file to be executed as a grader for the problem. The grader has access to the archive specified in `archive`.

In most use cases, either using one of the built-in graders, or a custom checker will suffice. A custom grader is only truly necessary if the normal interaction between the judge and the submission is insufficient.

```python
class Grader(BaseGrader):
  def grade(self, case):
    pass
```

## Parameters

`case` is a `TestCase` object.

- `case.position` is an integer, the current test case with a zero-based index.
- `case.input_data()` is a buffer containing the contents of the `in` file specified for the current case in `init.yml`. May be `b''`, if no case input file was specified.
- `case.output_data()` is a buffer containing the contents of the `out` file specified for the current case in `init.yml`. May be `b''`, if no case output file was specified.
- `case.points` is an integer, the max points that can be awarded for the current test case.

## Returns

A `Result` object (`from dmoj.result import Result`). Some notable fields include:

- `result_flag`: stores a mask defining the current test case result code.
- `proc_output`: contains the string that will be displayed in the partial output pane. However, if `result_flag` is `Result.AC`, then the partial output pane will not be shown.
- `feedback`: contains the feedback given by the judge.
- `extended_feedback`: contains any extended feedback that would not fit into the shorter `feedback` field, and is displayed as a separate pane beside the partial output on the site.

To illustrate, in a problem where the process must echo a line of input, an interactive approach would look like this:

```python
import subprocess

from dmoj.graders.standard import StandardGrader
from dmoj.result import Result


class Grader(StandardGrader):
  def grade(self, case):
    result = Result(case)
    case_input = b'Hello, World!\n'

    self._current_proc = self.binary.launch(
      time=self.problem.time_limit,
      memory=self.problem.memory_limit,
      stdin=subprocess.PIPE,
      stdout=subprocess.PIPE,
      stderr=subprocess.PIPE,
      wall_time=case.config.wall_time_factor * self.problem.time_limit,
    )
    output, error = self._current_proc.communicate(case_input)
    self.binary.populate_result(error, result, self._current_proc)

    if output == case_input:
      result.extended_feedback = 'Correct answer! This will be displayed in the output pane.'
      if result.result_flag == Result.AC:
        result.points = case.points
    else:
      result.result_flag |= Result.WA
      result.feedback = 'Wrong answer! :('

    return result
```

A simple solution to this problem is `print(input())`.

The associated `init.yml` for this problem would look like this:

```yaml
custom_judge: interactor.py
unbuffered: true
test_cases:
- points: 100
```

Since we use no input or output files (our test case is hardcoded), we do not need to specify the `archive` or related `in` and `out` fields.

In this example, it's important to note the `unbuffered` node. If set to `true`, the judge will use a pseudoterminal device for a submission's input and output pipes. Since ptys are not buffered by design, setting `unbuffered` to `true` removes the need for user submissions to `flush()` their output stream to guarantee that the `grader` receives their response. **The `unbuffered` node is not exclusive to interactive grading: it may be specified regardless of judging mode.**

# Interactive Grading

Interactive grading is used for problems where users should implement an online algorithm or where the grader must generate input or compute a score based on the user's previous output.
Using an interactive grader is similar to using a custom grader: the `custom_judge` node also needs to be set. Rewriting the previous custom judge using an interactive grader would result in:

```python
from dmoj.graders.interactive import InteractiveGrader
from dmoj.utils.unicode import utf8text


class Grader(InteractiveGrader):
  def interact(self, case, interactor):
    # The line to print
    case_input = 'Hello, World!'

    # Print the line, using the interactor
    interactor.writeln(case_input)

    # interact can return either a boolean, or a Result
    return case_input == utf8text(interactor.readln())
```

## Parameters

`case` is a `TestCase` object, and is identical to the one in the `Grader` section.
`interactor` is an `Interactor` object.

- `interactor.read()` reads all of the submission's output available.
- `interactor.readln(strip_newline=True)` reads the next line of the submission's output. If `strip_newline` is true, the trailing newline is stripped, otherwise it is retained.
- `interactor.readtoken(delim=None)` reads the next available token of the submission's output, as determined by `string.split(delim)`.
- `interactor.readint(lo=float('-inf'), hi=float('inf'), delim=None)` reads the next token of the submission's output, as determined by `string.split(delim)`. Additionally, the checker will automatically generate a wrong answer verdict if either the token cannot be converted to an integer, or if it is not in the range [*lo*, *hi*].
- `interactor.readfloat(lo=float('-inf'), hi=float('inf'), delim=None)` reads the next token of the submission's output, as determined by `string.split(delim)`. Additionally, the checker will automatically generate a wrong answer verdict if either the token cannot be converted to a float, or if it is not in the range [*lo*, *hi*].
- `interactor.write(val)` writes `val`, cast to a string, to the submission's standard input.
- `interactor.writeln(val)` writes `val`, cast to a string, to the submission's standard input, followed by a newline.
- `interactor.close()` closes the submission's `stdin` stream.

## Returns

Either a boolean or a `Result` (`from dmoj.result import Result`) object. The boolean is true if the submission should score full points, and `false` otherwise. The `Result` object is handled the same way as custom graders.

## Native Interactive Grading
Sometimes, an interactive grader will be very computationally expensive.
In these cases, one can use the `bridged` grader.
To invoke the `bridged` grader, `interactive` should be a top level node that contains `files` and `lang`.
`files` is either a single filename, or a list of filenames, corresponding to the interactor.
`lang` is the language of the interactor.

Optional arguments are:

- `flags`: flags to pass to the compiler.
- `compiler_time_limit`: the time limit allocated to compiling the interactor.
- `preprocessing_time`: the interactor's time limit is equal to this value plus the time limit of the problem, in seconds.
- `memory_limit`: the memory limit allocated to the interactor. It defaults to `env['generator_memory_limit']`.
- `type`: specifies the arguments to pass the checker and how to interpret the checker's return code and output.
  - The `default` type passes the arguments in the order `input_file judge_file`. A return code of `0` is an AC, `1` is a WA, and anything else results in an internal error.
  - The `testlib` type passes the arguments in the order `input_file output_file judge_file`.
  Note that `output_file` will always be `/dev/null`, and is passed to maintain compatibility with `testlib.h`. A return code of `0` is an AC, `1` is a WA, `2` is a presentation error, `3` corresponds to an assertion failing,
  and `7`, along with an output to `stderr` of the format `points X` for an integral ~X~ awards ~X~ points. Anything else results in an internal error.
  - The `coci` type passes the arguments in the order `input_file judge_file`. Its parsing of return codes is the same as the `testlib` type, but has partial format `partial X/Y`, which awards ~\frac X Y~ of the points.
  - The `peg` type exists for compatibility with the WCIPEG judge, and is not meant to be used here.

The interactor's standard input is connected to the submission's standard output, and vice versa.
After the interactor prints, it is required to flush.

To specify a correct answer, the interactor should return 0.
To specify an incorrect answer, the interactor should return 1.
All other return values will be considered internal errors.

To override this behaviour, you can change `type` to a valid `contrib` module, such as `testlib`.


An example `init.yml` would be as follows:
```yaml
unbuffered: true
archive: seed2.zip
interactive: {files: interactor.cpp, type: testlib}
test_cases:
- {in: seed2.1.in, points: 20}
- {in: seed2.2.in, points: 20}
- {in: seed2.3.in, points: 20}
- {in: seed2.4.in, points: 20}
- {in: seed2.5.in, points: 20}
- ```

An example of interactor is as follows. Note that it is not necessary to flush,
even if `unbuffered` is not true.
```cpp
#include <cstdio>
#include <cstdlib>

inline void read(long long *i) {
    if(scanf("%lld", i) != 1 || *i < 1 || *i > 2000000000)
        exit(2);
}

int main(int argc, char *argv[]) {
  FILE *input_file = fopen(argv[1], "r");
  int N, guesses = 0;
  long long guess;
  fscanf(input_file, "%d", &N);
  while (guess != N) {
    read(&guess);
    if (guess == N) {
      puts("OK");
    } else if (guess > N) {
      puts("FLOATS");
    } else {
      puts("SINKS");
    }
    guesses++;
  }
  if (guesses <= 31)
    return 0; // AC
  else
    return 1; // WA
}
```

# Function Signature Grading (IOI-style)
Signature grading is used for problems where users should implement an online algorithm or interact with the grader directly without the need for traditional input and output routines. This is commonly seen in competitions such as the IOI, where all input is passed through function arguments and output is replaced with return values or directly modifying specifically allocated memory for the computed answer. Currently, C and C++ are supported for this mode. `signature_grader` should be a top-level node that contains `entry` and `header`. `entry` is a C or C++ file that contains the `main` function. It should read input from `stdin`, call the user's implemented functions specified in `header`, and write output to `stdout`. You may specify a custom `checker` to interpret the `entry`'s output. If no custom checker is specified, it will be compared to the output file using the default checker.

The user's submission will be automatically modified to include the file `header`, and the symbol `main` is redefined as `main_GUID` where `GUID` is a randomly generated GUID. This is so users testing their program do not have to manually remove their `main` function before submissions; it does not protect against the preprocessor directive `#undef main`.

The global variables in the `entry` should be declared static to prevent name collisions. Optimally, `header` should have an include guard, in case it contains something other than function prototypes.

Note that both the `entry` and the user's code will be compiled with C++14 regardless of whether C, C++, C++03, C++11, or C++14 is selected as the language. Issues that may arise from this are naming variables like `class` in a C program, which otherwise would be allowed.

An example of the `init.yml`:

```yaml
signature_grader: {entry: handler.c, header: header.h}
test_cases:
- {in: siggrade.1.in, out: siggrade.1.out, point: 50}
- {in: siggrade.2.in, out: siggrade.2.out, point: 50}
```

An example of the `entry` file:

```c
#include "header.h"
#include <stdio.h>
#include <stdbool.h>

static int n;

int main()
{
    scanf("%d", &n);
    bool valid = is_valid(n); // Defined in header
    printf(valid ? "correct" : "wrong");
    return 0;
}
```

An example of the `header` file:

```c
#ifndef _GRADER_HEADER_INCLUDED
#define _GRADER_HEADER_INCLUDED
#include <stdbool.h>
bool is_valid(int);
#endif
```
