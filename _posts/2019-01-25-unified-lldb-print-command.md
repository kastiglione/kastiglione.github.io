---
layout: post
title:  "Unified lldb Print Command"
date:   2019-01-25 22:26:23 -0800
categories: lldb
---

LLDB has a pair of new aliases, `v` and `vo`, and this is of interest because of what the Xcode 10.2 beta release notes have to say:

> The LLDB debugger has a new command alias, `v`, for the “frame variable” command to print variables in the current stack frame. Because it bypasses the expression evaluator, `v` can be a lot faster and should be preferred over `p` or `po`.

Most lldb users are used to `po`, which means this addition, _and this advice_, is going to change people's workflow. Before printing, you now might ask and answer yourself "Do I need `vo` or `po`"? This is another case where lldb seems to be designed to expose, rather than abstract, the implementation details of a debugger. This isn't the right choice for the majority of debugger users.

Fortunately, a Python `if` statement can give us a single unified print command that makes the right `vo` or `po` choice. At its essence, the logic is:

```python
if not vo(expression):
    po(expression)
```

### Details, Code

In addition to providing a single unified print command, this blog post aims to teach a little lldb Python. Being able to make the debugger do whatever you need feels really good, and I'd love for everyone to feel that from time to time. You don't need to know Python, -- like JavaScript, anyone can write Python without knowing it. Internet searches and trial and error get you _really_ far.

Here's the top level code for the unified print command:

```python
import lldb

@lldb.command("po")
def vo_else_po(debugger, expression, context, result, _):
    frame = context.frame
    if not vo(expression, frame, result):
        po(expression, frame, result)
```

This will first try the expression as a variable, and otherwise fallback to full expression evaluation.

Anyone new to lldb's Python API will have questions, like "what exactly is `@lldb.command(...)`". See the epilog for explanations of these API quirks.

But first, this code calls two helper commands, `vo()` and `po()` which we need to implement.

### `vo` in Python

The `vo` command is an alias for `frame variable -O`, and the matching Python function is [`GetValueForVariablePath`](https://lldb.llvm.org/python_reference/lldb.SBFrame-class.html#GetValueForVariablePath).

It's fairly simple to implement `vo`, call `GetValueForVariablePath()`, handle any errors, and print the description. The code is:

```python
def vo(expression, frame, result):
    value = frame.GetValueForVariablePath(expression)
    if value.error.fail:
        return False
    print(value.description, file=result)
    return True
```

This function returns `True` if the expression is the name of a variable, or a variable expression like `foo.bar`. Otherwise it returns `False` to indicate that `vo` failed, and to proceed with `po()`.

### `po` in Python

The `po` command is an alias for `expression -O`, and the equivalent Python function is [`EvaluateExpression`](https://lldb.llvm.org/python_reference/lldb.SBFrame-class.html#EvaluateExpression).

Similar to `vo()`, the `po()` function can be written by calling the `EvaluateExpression()`, handling any errors, and printing the description. Code:

```python
def po(expression, frame, result):
    value = frame.EvaluateExpression(expression)
    if value.error.fail:
        result.SetError(value.error)
        return False
    print(value.description, file=result)
    return True
```

### Complete Command

Putting it together, here is the code to have a unified `po` command. This will override the builtin `po` alias, with a smarter version that uses `vo` if possible, otherwise does `po`.

```python
from __future__ import print_function
import lldb

@lldb.command("po")
def vo_else_po(debugger, expression, context, result, _):
    frame = context.frame
    if not vo(expression, frame, result):
        po(expression, frame, result)

def vo(expression, frame, result):
    value = frame.GetValueForVariablePath(expression)
    if value.error.fail:
        return False
    # Use rstrip because of extra unwanted newline.
    print(value.description.rstrip(), file=result)
    return True

def po(expression, frame, result):
    value = frame.EvaluateExpression(expression)
    if value.error.fail:
        result.SetError(value.error)
        return False
    print(value.description, file=result)
    return True
```

To use it, add the following to your `~/.lldbinit`:

```
command script import path/to/vo_else_po.py
```

### Epilog: Python LLDB Answers

1. `@lldb.command("po")` -- decorator that registers the Python function as an lldb command named `po` (this overrides the builtin `po` alias)
2. `expression` -- the command input entered by the user, in this case the string is either a variable name or code to evaluate
3. `context` ([`SBExecutionContext`](https://lldb.llvm.org/python_reference/lldb.SBExecutionContext-class.html)) -- convenience accessor for the current `frame`, `thread`, `process`, etc
4. `result` ([`SBCommandReturnObject`](https://lldb.llvm.org/python_reference/lldb.SBCommandReturnObject-class.html)) -- status of an lldb command, such as whether it errored, and also stores the command's output (see `file=result`)
5. `error` ([`SBError`](https://lldb.llvm.org/python_reference/lldb.SBError-class.html)) -- although named "error", this is much like a `Result` type, it can be either `success` or `fail` (`error` is always non nil)
6. `description` ([`SBValue.description`](https://lldb.llvm.org/python_reference/lldb.SBValue-class.html#description)) -- generates the object description of a value, this is what separates `po` from `p`
