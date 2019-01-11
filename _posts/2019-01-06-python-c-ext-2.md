---
layout: post
title:  "Dealing with Python values in C"
date:   2019-01-06 20:00:00 -0800
tags: python c
---

In the previous post we saw how to set up the boilerplate code necessary for a C
Python extension. We will use that code and see how to handle and throw errors,
deal with Python objects, and convert from C to Python types.

Let's create a new function called `my_function`. This function will take 
a string, an integer, and a floating-point number. First
we will look at how to extract these arguments from the `args` object.

```c
static PyObject* my_function(PyObject* Py_UNUSED(self), PyObject* args) {
  const char* a;
  int b;
  double c;

  if (!PyArg_ParseTuple(args, "sid", &a, &b, &c)) 
    return NULL;
```

Python passes all arguments to functions as a tuple, so we need to unpack them
with `PyArg_ParseTuple`. This function will take our `args` tuple and a _format string_.
Here we use `"sid"` to indicate we want a string, integer, and double. If Python doesn't 
receive exactly these three arguments with those types, the parse function will _set an error_
and return `false`. In this case we also return `NULL` to propagate this error. 

There are many useful format string values, accessible [here](https://docs.python.org/3/c-api/arg.html).

We can also set errors ourselves. Let's say we want the second parameter `b` to be non-negative.
```c
  if (b < 0) {
    PyErr_SetString(PyExc_ValueError, "Cannot be negative");
    return NULL;
  }
```
This code is analogous to `raise ValueError("Cannot be negative")`. Note that we return `NULL` to
indicate that we raised an error.

Let's quickly use these values:

```c
  for (int i = 0; i < b; i++)
    printf("%f %s\n", c, a);
```

Now let's say we want to return the quantity `b * c`. In Python, everything, including primitives, is a
`PyObject*`. We can create a floating-point number object with `PyFloat_FromDouble`

```c
  return PyFloat_FromDouble((double)b * c);
}
```

For the next function we implement, it will take in a list of list of doubles,
and then add up all the doubles.

```c
static PyObject* process_list(PyObject* Py_UNUSED(self), PyObject* args) {
  PyObject* list;

  if (!PyArg_ParseTuple(args, "O!", &PyList_Type, &list)) 
    return NULL;
```
Here we use the format `"O!"`. The `O` indicates we are need an object,
and the `!` indicates we will specify a type the object must have. We need
to do this because Python lists don't have a corresponding C primitive.
Then we pass in the address of the type object first. Now, Python will
also throw an error if the user provides an invalid object type.

We can retrieve the size of the list:
```c
  Py_ssize_t list_size = PyList_Size(list);
```
And we can get individual items with `PyList_GetItem`. Further information
can be found at the [List API](https://docs.python.org/3.7/c-api/list.html)

First let's check that all items of our list to be lists themselves, while adding them up.
We can do this with `PyList_Check`:
```c
  double sum = 0;

  for (Py_ssize_t i = 0; i < list_size; i++) {
    PyObject* sublist = PyList_GetItem(list, i);

    if (!PyList_Check(sublist)) {
      PyErr_SetString(PyExc_TypeError, "List must contain lists");
      return NULL;
    }
```
Now we can iterate over each sublist and add them up
```c
    Py_ssize_t sublist_size = PyList_Size(sublist);

    for (Py_ssize_t j = 0; j < sublist_size; j++) {
      sum += PyFloat_AsDouble(PyList_GetItem(sublist, j));

      if (PyErr_Occurred()) return NULL;
    }
  }

  return PyFloat_FromDouble(sum);
}
```

We convert `PyObject*`s to `double`s here with `PyFloat_AsDouble`. But this function has no way
to signal errors (for example, if the list item isn't the right type) because it returns the
primitive `double`.
So instead we call `PyErr_Occurred` to see if the conversion was successfull, and if not, we return `NULL`.

After adding these functions to the method table, let's try them out. If we pass in invalid arguments:
```python
>>> test_module.my_function("hello", "wrong argument", 2)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: an integer is required (got type str)
```
And then with valid arguments:
```python
>>> x = test_module.my_function("hello", 2, 3.1415)
3.141500 hello
3.141500 hello

>>> x
6.283
```

Here are some errors from type-checking with `process_list`
```python
>>> test_module.process_list("not a list")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: argument 1 must be list, not str

>>> test_module.process_list([ "still not right" ])
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: List must contain lists

>>> test_module.process_list([ [ 1.23, "almost there"], [2.2, 3.3, 4] ])
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: must be real number, not str
```

And when used correctly:
```python
>>> test_module.process_list([ [ 1.23 ], [2.2, 3.3, 4] ])
10.73
```

Here is the complete code:
```c
#include <Python.h>

#if PY_MAJOR_VERSION < 3
#error "Requires Python 3"
#include "stopcompilation"
#endif

static PyObject* hello(PyObject* Py_UNUSED(self), PyObject* Py_UNUSED(args)) {
  printf("Hello world\n");

  Py_RETURN_NONE;
}

static PyObject* my_function(PyObject* Py_UNUSED(self), PyObject* args) {
  const char* a;
  int b;
  double c;

  if (!PyArg_ParseTuple(args, "sid", &a, &b, &c))
    return NULL;

  if (b < 0) {
    PyErr_SetString(PyExc_ValueError, "Cannot be negative");
    return NULL;
  }

  for (int i = 0; i < b; i++)
    printf("%f %s\n", c, a);

  return PyFloat_FromDouble((double)b * c);
}

static PyObject* process_list(PyObject* Py_UNUSED(self), PyObject* args) {
  PyObject* list;

  if (!PyArg_ParseTuple(args, "O!", &PyList_Type, &list))
    return NULL;

  Py_ssize_t list_size = PyList_Size(list);

  double sum = 0;

  for (Py_ssize_t i = 0; i < list_size; i++) {
    PyObject* sublist = PyList_GetItem(list, i);

    if (!PyList_Check(sublist)) {
      PyErr_SetString(PyExc_TypeError, "List must contain lists");
      return NULL;
    }

    Py_ssize_t sublist_size = PyList_Size(sublist);

    for (Py_ssize_t j = 0; j < sublist_size; j++) {
      sum += PyFloat_AsDouble(PyList_GetItem(sublist, j));

      if (PyErr_Occurred()) return NULL;
    }
  }

  return PyFloat_FromDouble(sum);
}

static PyMethodDef methods[] = {
  { "hello", &hello, METH_VARARGS, "Hello world function" },
  { "my_function", &my_function, METH_VARARGS, "Takes string, int, float"},
  { "process_list", &process_list, METH_VARARGS, "Adds up list of lists of doubles" },
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

```typescript
function getText(x: string, y: number): string {
  const result = "";
  for (let i = 0; i < y; i++) 
    result += x;

  return result;
}
```
