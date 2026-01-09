```lang-none
jupyter-notebook : 6.1.4
ipykernel        : 5.3.4
```

# Overriding the `input` function in Jupyter Notebook

Consider a long-running Jupyter Notebook that pauses execution several times to ask for user input — perhaps to make decisions or perform physical connections. You might want to start the notebook, walk away, and be alerted only when input is required. One way to do this is to override `input` so that it emits an audible notification when a prompt is reached.

A first attempt might look like this:

```python
import builtins
import subprocess

_original_input = builtins.input

def input_with_bell(*args, **kwargs):
    subprocess.call(['espeak', 'Ding-dong!'])
    return _original_input(*args, **kwargs)

builtins.input = input_with_bell

data = input()
```

This works, but only within a single notebook cell. Once execution moves to another cell, the override no longer applies.

To make the behavior persistent across all cells, we need to hook into how Jupyter actually handles `input` when using the IPython kernel. Internally, user input is requested via the `IPythonKernel._input_request` method, which can be monkey-patched as follows:

```python
from ipykernel.ipkernel import IPythonKernel
import subprocess

_original_input = IPythonKernel._input_request

def input_with_bell(*args,  **kwargs):
    subprocess.call(['espeak', 'Ding-dong!'])
    return _original_input(*args, **kwargs)

IPythonKernel._input_request = input_with_bell

data = input()
```

## ⚠️ Warning: Monkey-Patching Internals

This approach relies on monkey-patching a private, internal method of IPython (`_input_request`). Such internals are not part of the public API and may change or break without notice between IPython or Jupyter releases. Use this technique only for personal workflows, experiments, or short-lived notebooks — not in production or shared environments — unless you are prepared to maintain it over time.

---

P.S. A more reliable approach may be to work with the kernel instance's `raw_input` method:

```python
instance = get_ipython()
_raw_input = instance.kernel.raw_input

def input_with_bell(*args,  **kwargs):
    subprocess.call(['espeak', 'Ding-dong!'])
    return _raw_input(*args, **kwargs)

instance.kernel.raw_input = input_with_bell
```

This works with `jupyter-notebook 7.4.7` and `ipykernel 6.30.1`, in addition to the previously mentioned versions.
