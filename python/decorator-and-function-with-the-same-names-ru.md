# Об особенности декораторов в Python

У синтаксиса декораторов Python есть незаметная, но важная особенность — в конструкции `@decorator` мы можем применить декоратор к функции с тем же именем: 

```python
def func(f):
    def wrapper(*args, **kwargs):
        print("Wrapper...")
        return f(*args, **kwargs)
    return wrapper

# Это декорирование возможно
@func
def func():
    print("Runner...")

# А такое декорирование невозможно
def func():
    print("Runner...")
func = func(func)
```

Что говорит о декораторах [глоссарий][1]:

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
> The same concept exists for classes, but is less commonly used there. See the documentation for function definitions and class definitions for more about decorators.

То есть, применение `@decorator` перед `def func(): ...` представлено как замена выражению `func = decorator(func)`. Но это не совсем так.

Начнём с того, что у имени функции есть две разные задачи:
1) именовать функцию на уровне объекта (например, через свойство функции `__name__`)
2) обозначать функцию как объект в соответствующем пространстве имён (например, как ключ в словаре `__dict__` модуля) 

Изначально функция хранится как [кодовый объект][2], в свойстве `co_name` которого сохраняется её имя (это первая задача). А при выполнении программы функция создаётся из кодового объекта и связывается с заданным именем (это вторая задача):

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

Если же мы применяем синтаксис декоратора, то в соответствующей последовательности команд байт-кода декоратор обрабатывает функцию после её создания, но до того, как она будет связана с именем в соответствующем пространстве имён:

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

То есть:
* сначала загружается функция декоратора `LOAD_NAME 0 (my_decorator)`; 
* затем — кодовый объект новой функции `LOAD_CONST 0 (<code object my_func ...>)`;
* кодовый объект превращается в функцию `MAKE_FUNCTION`;
* к полученной функции применяется загруженный ранее декоратор `CALL`;
* результат сохраняется под именем функции `STORE_NAME 1 (my_func)`.

Тем самым мы получаем возможность использовать одно и то же имя для декоратора и декорируемой функции в синтаксисе `@decorator`. А это позволяет переопределять объект через него же. Такой подход используется, например, при создании свойств класса:  

```python
class A:  

    @property
    def x(self):
        return ...

    @x.setter
    def x(self, value):
        ...
```

В этом примере `x.setter` применяется раньше, чем метод получит имя в словаре свойств `A`, а под именем метода сохраняется уже результат декорирования. То есть, несмотря на совпадение имён свойства и метода, конфликта не будет. При этом нам важно, чтобы имя нового сеттера совпадало с именем свойства, потому что именно оно будет использовано для сохранения обновлённого свойства в пространстве класса `А`.


  [1]: https://docs.python.org/3/glossary.html#term-decorator
  [2]: https://docs.python.org/3/reference/datamodel.html#code-objects
