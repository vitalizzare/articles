# A subtle detail of decorators in Python

Python's decorator syntax has a small but important detail: in the `@decorator` form, you can apply a decorator to a function **with the same name**.

```python
def func(f):
    def wrapper(*args, **kwargs):
        print("Wrapper...")
        return f(*args, **kwargs)
    return wrapper

# This works
@func
def func():
    print("Runner...")

# But this does not
def func():
    print("Runner...")
func = func(func)
```

According to the Python [glossary][1]:

> **decorator**\
> A function returning another function, usually applied as a function transformation using the `@wrapper` syntax. Common examples for decorators are `classmethod()` and `staticmethod()`.\
> The decorator syntax is merely syntactic sugar, the following two function definitions are semantically equivalent:
> ```python
> def f(arg):
>     ...
> f = staticmethod(f)
>
> @staticmethod
> def f(arg):
>     ...
> ```

However, the "syntactic sugar" explanation is slightly simplified in practice. 

A function name in Python has two distinct responsibilities:

1. The **object-level name** of the function (for example, `function.__name__`)
2. The **binding name** of the function inside a namespace (for example, a key inside a module's `__dict__`)

Internally, a function is first stored as a [code object][2], where its name is saved in `co_name`. Then, during execution, Python creates a function object from the code object and binds it to a name in the namespace. 

Example disassembly:

```python
>>> dis('def my_func(): pass')
  0           RESUME                   0

  1           LOAD_CONST               0 (<code object my_func at 0x7c452e7472f0, file "<dis>", line 1>)
              MAKE_FUNCTION
              STORE_NAME               0 (my_func)
              LOAD_CONST               1 (None)
              RETURN_VALUE

Disassembly of <code object my_func at 0x7c452e7472f0, file "<dis>", line 1>:
  1           RESUME                   0
              LOAD_CONST               0 (None)
              RETURN_VALUE
```

With `@decorator` syntax, the decorator is applied **after the function object is created**, but **before it is bound to a name in the namespace**. 

Example:

```python
>>> dis('''
@my_decorator
def my_func():
    pass
''')

  0           RESUME                   0

  2           LOAD_NAME                0 (my_decorator)

  3           LOAD_CONST               0 (<code object my_func at 0x7c452e746db0, file "<dis>", line 2>)
              MAKE_FUNCTION

  2           CALL                     0

  3           STORE_NAME               1 (my_func)
              LOAD_CONST               1 (None)
              RETURN_VALUE

Disassembly of <code object my_func at 0x7c452e746db0, file "<dis>", line 2>:
  2           RESUME                   0

  4           LOAD_CONST               0 (None)
              RETURN_VALUE
```

Here we can see the following execution order:

1. Load decorator function `LOAD_NAME my_decorator`
2. Load code object of the new function `LOAD_CONST <code object my_func>`
3. Create function object `MAKE_FUNCTION`
4. Call decorator with that function `CALL`
5. Store the result under the function name `STORE_NAME my_func`

Because decoration happens before the function is bound to its final name, Python allows using the same identifier for both the decorator and the decorated function. In other words, you can redefine an object through itself.

A common real-world example is class properties:

```python
class A:

    @property
    def x(self):
        ...

    @x.setter
    def x(self, value):
        ...
```

In this example, `x.setter` is applied before the setter method is stored under the name `x` in class `A`. At the same time, it is important for us that the name of the new setter coincides with the name of the property, because it is this name that will be used to save the updated property in the `A` class space.

## Key takeaway

The `@decorator` statement is conceptually similar to `func = decorator(func)`, but operationally different because of **when** the binding happens. That timing difference is what allows decorators and decorated functions to share the same name without conflicts. 


  [1]: https://docs.python.org/3/glossary.html#term-decorator
  [2]: https://docs.python.org/3/reference/datamodel.html#code-objects

