---
layout: post
title:  "Short Guide to Writing C Extensions for Python"
date:   2019-01-06 18:06:00
tags: python c
---

Sometimes portions of a Python codebase could benefit from being written in C. 
For example, the code may need to interact with other C libraries, or we may simply
need the performance boost. Although the [Python documentation](https://docs.python.org/3/extending/extending.html)
for writing extensions is very thorough, there are some common operations which are
not as obvious as they may seem. In this blog series we will look at the boilerplate code
we need to write extensions for Python 3, as well as how to actually interact with various
Python objects. 

First, we will examine how to create a simple module which contains one function
which prints its argument. Before we start, it is important to know that there are some significant differences
between Python 2 and Python 3 in writing extensions. We will only look at Python 3, but
if you need to be compatible with both, I found [this blog post](http://python3porting.com/cextensions.html) helpful

To start, first we need to set up our development environment. We will need at minimum
two files, a module source in C, and a setup script in Python. Let's name these files
`module_source.c` and `setup.py`. 

First we will examine a minimal setup script. 
This file will control how our extension is compiled.

```python
#!/usr/bin/env python3 
from distutils.core import Extension, setup

module = Extension("test_module", sources=[ "module_source.c" ])
setup(name="test_module", version="1.0", ext_modules=[ module ])
```

As we can see we first need to create an `Extension` object. This describes the compilation
process, and there are [many arguments](https://docs.python.org/3/distutils/apiref.html#distutils.core.Extension)
we can pass to it. We can [add metadata](https://docs.python.org/3/distutils/setupscript.html#meta-data) to our extension by passing it to `setup` such as author, descriptions, etc.

Now we can create a minimal `module_source.c`. Obviously we will need to include `Python.h`. 

```c
#include <Python.h>
```
The setup script will make sure this file is available in the include path at compile time.
such as adding `-I/usr/include/python3.4m` for Linux. This also will include `stdio.h`, `stdlib.h`
and as some other standard library headers.

Since our extension will only be compatible with Python 3 (see the above link if you need Python 2) it
may be useful to safeguard against potential accidental compilation with Python 2.

```c
#if PY_MAJOR_VERSION < 3
#error "Requires Python 3"
#include "stopcompilation"
#endif
```
Including a nonexistant file is a guaranteed way to stop compilation.

We are now ready to write our first function. 
```c
static PyObject* hello(PyObject* self, PyObject* args) {
  (void)self;
  (void)args;

  printf("Hello world\n");

  Py_RETURN_NONE;
}
```

The `self` argument is our module. We could use this to [access information](https://docs.python.org/3/c-api/module.html)
about it, but for our purposes we will not need to use it, so we ignore it. Similarly, right now we are not using
our arguments so we ignore those as well.

As our function does not return anything, it may seem like we can just return `NULL`, but in this case
Python uses a return value of `NULL` to indicate that an error occured. So instead we return the `None` object.
Since this is a common pattern, the macro `Py_RETURN_NONE` will increase the reference count of the `None` object
and return it. 

Now we need to declare the methods our module exports. Our module will contain one function
called `hello`

```c
static PyMethodDef methods[] = {
  { "hello", &hello, METH_VARARGS, "Hello world function" },
  { NULL, NULL, 0, NULL }
};
```

In order, we provide a name we export, a function pointer to the function, the calling convention
(we use `METH_VARARGS` because we will need to in the next blog post, although there are many useful other [calling conventions](https://docs.python.org/3/c-api/structures.html#c.PyMethodDef)), and a function description. We add an entry for each
method we wish to export, followed by a `NULL`-terminator

Finally, we need to add a module definition and a module initialization function.

```c
static struct PyModuleDef module_def = {
  PyModuleDef_HEAD_INIT, // always required
  "test_module",         // module name
  "Testing module",      // description
  -1,                    // module size (-1 indicates we don't use this feature)
  methods,               // method table
};

PyMODINIT_FUNC PyInit_test_module() {
  printf("Initializing my module\n");

  return PyModule_Create(&module_def);
}
```

Our initialization function must be named `PyInit_<module name>`, but
there are no requirements for the module definition struct.
That is enough code to compile and use our module.
Using the setup script:

```shell
% ./setup.py build
```

This will create a `build/` directory, inside of which we will find a `lib` directory such as `lib.macosx-10.12-x86_64-3.7/`.
Inside that directory will be our compiled module. In this case it is `test_module.cpython-37m-darwin.so`. 

In order to use our module, we have some options: 
  - We can install to the system directory (e.g. `/usr/local/lib/python3.7/site-packages`) with `./setup.py install`.
  This is the best option if you are distributing the module to others
  - We can navigate to the directory containing the `.so` library before running the interpreter
  - We can copy the `.so` library to a directory listed in the environment variable `$PYTHONPATH`
  - We could [recompile the Python interpreter](https://docs.python.org/3/extending/extending.html#compilation-and-linkage) with our module 

For more information, I recommend [this page](https://leemendelowitz.github.io/blog/how-does-python-find-packages.html)

Now we are ready to use our module. 

```python
>>> import test_module
Initialization
>>> test_module
<module 'test_module' from '/Users/ishan/src/test/pyext/build/lib.macosx-10.12-x86_64-3.7/test_module.cpython-37m-darwin.so'>
>>> test_module.hello
<built-in function hello>
>>> test_module.hello()
Hello world
```

The descriptions we added earlier can be viewed with `help(test_module)` or `help(test_module.hello)`.

Here is the complete code for the extension:

```c
#include <Python.h>

#if PY_MAJOR_VERSION < 3
#error "Requires Python 3"
#include "stopcompilation"
#endif

static PyObject* hello(PyObject* self, PyObject* args) {
  (void)self;
  (void)args;

  printf("Hello world\n");

  Py_RETURN_NONE;
}

static PyMethodDef methods[] = {
  { "hello", &hello, METH_VARARGS, "Hello world function" },
  { NULL, NULL, 0, NULL }
};

static struct PyModuleDef module_def = {
  PyModuleDef_HEAD_INIT, // always required
  "test_module",         // module name
  "Testing module",      // description
  -1,                    // module size (-1 indicates we don't use this feature)
  methods,               // method table
};

PyMODINIT_FUNC PyInit_test_module() {
  printf("Initialization\n");
  return PyModule_Create(&module_def);
}
```

That completes the boilerplate we need to create a Python extension. In the next post we will cover
Python values and unpacking arguments.