---
layout: post
title:  "Annotation gimmicks"
date:   2026-06-27 03:34:19
categories: jekyll update
katex: true
---

Anyone can tell you Python is a dynamically typed language. Until it isn't. Because everyone decided dynamic types suck and are bad, Python uses a variety of methods, such as type checkers (such as `mypy`, `ty`, and `pyright`), type hints, and runtime type checking (such as `isinstance`, `pydantic`, and `pydantic-core`) to enforce this.

These methods ultimately rely on type hints to be (enforced? checked?). Type hints essentially put types next to function signatures and variables in a way that can be seen by tools such as type checkers and running code.

For example the snippet:
```py
def my_func(x: int) -> str:
    return x + 3
```

creates a function `my_func` that presumably takes in an `int` and returns a `str`.

However, this doesn't stop you from passing in a different type (hence it being called a type hint or type annotation), with whatever consequences that may have.

```py
my_func(3.0)  # 6.0
```

However, tools like `pydantic` check these type annotations for functionality (and in `pydantic`'s case, do runtime type checking).

```py
>>> import pydantic
>>> @pydantic.validate_call
... def my_func(x: int) -> int:
...     return x + 3
...     
>>> my_func(3)
6
>>> my_func(3.0)
6
>>> my_func(3.1)
Traceback (most recent call last):
  File "<python-input-4>", line 1, in <module>
    my_func(3.1)
    ~~~~~~~^^^^^
  ...
pydantic_core._pydantic_core.ValidationError: 1 validation error for my_func
0
  Input should be a valid integer, got a number with a fractional part [type=int_from_float, input_value=3.1, input_type=float]
    For further information visit https://errors.pydantic.dev/2.13/v/int_from_float
>>> 
```

## Changing type annotations in runtime

Annotations can be accessed in runtime with `annotationlib.get_annotations`, returning a dictionary.

```py
>>> def func(x: int) -> int:
...     return 0
...     
>>> import annotationlib
>>> annotationlib.get_annotations(func)
{'x': <class 'int'>, 'return': <class 'int'>}
```

It maps parameter names (as the key), to the type (as a `type` object). `return` denotes the function's return type.

Checking the docs of `annotationlib.get_annotations`, it states under the functionality of the function for the `format` parameter:

> VALUE: object.__annotations__ is tried first; if that does not exist, the object.__annotate__ function is called if it exists.

By default, `format` is `Format.VALUE`, `Format` being an `enum.IntEnum`. This implies the existence of an `__annotations__` attribute.

```py
>>> func.__annotations__
{'x': <class 'int'>, 'return': <class 'int'>}
```

which, is similarly a `dict` mapping with the same structure as `annotationlib.get_annotations`.

What if we try to modify the dictionary?

```py
>>> func.__annotations__["x"] = str
>>> annotationlib.get_annotations(func)
{'x': <class 'str'>, 'return': <class 'int'>}
```

The "new" annotation seems to persist after being modified.

Of course, type checkers will still yell at you if you try to pass in a string into `func`, because it performs static analysis and doesn't see the `__annotations__` parameter was modified later on.

## Tricking `pydantic` into accepting the "wrong" type (or right, depending on who you ask)

From here, it's relatively obvious how to essentially "trick" pydantic (is it a trick? not really).

Decorators in Python are essentially just functions that modify other functions. As in the third code snippet:

```py
import pydantic


@pydantic.validate_call
def my_func(x: int) -> int:
    return x + 3
```

expands to

```py
import pydantic

def my_func(x: int) -> int:
    return x + 3


my_func = pydantic.validate_call(my_func)
```

so if you just modify the `__annotations__` dictionary prior to wrapping it:

```py
import pydantic

def my_func(x: int) -> int:
    return x + 3


my_func.__annotations__["x"] = float
my_func = pydantic.validate_call(my_func)

my_func(3.1)  # 6.1
```

## Conclusion

Felt like writing something because I haven't in a while :p.
