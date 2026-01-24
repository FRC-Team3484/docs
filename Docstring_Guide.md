# Docstring Guide

The goal of docstrings are to make sure the python linter (utility to help with debugging and autocompletion) can tell you what a function does when you hover over it in the editor.

## Our Standard

> [!NOTE]
> Google's docstring standard can be found [here](google.github.io/styleguide/pyguide.html#38-comments-and-docstrings)

Consider the following function:
```python
def foo(a: int, b: int, c: int) -> bool:
  return (a+b) == int(c)
```

### Example

while a google-standard function docstring would look like the following:
```python
  '''
  Args:
    a: an integer that is added to the b integer.
    b: another integer. a is added to it.
    c: another integer that is compared to the sum of the two integers

  Returns:
    bool: True if a+b = c, and False otherwise
  '''
```

ours would look like:
```python
  '''
  Parameters:
    - a (`int`): an integer that is added to `b`.
    - b (`int`): an integer that is added to `a`.
    - c (`int`): the integer that the sum of `a+b` is compared to.

  Returns:
    bool: True if `c` is equal to `a+b`, `False` otherwise.
    
  '''
```

### Differences

the main difference is the fact that our "Parameters" section is slightly better styled than the google "Args" one.

other than that, the only other *real* difference is the fact that we use inline codeblock backticks (\`) for every name and type.

