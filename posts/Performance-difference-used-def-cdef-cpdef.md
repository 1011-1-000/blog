---
layout: post
title: "Performance difference used def cdef cpdef"
date: 2016-11-23 11:40:15 +0800
comments: true
categories: python
tags: [python,cython]
---

#### general difference between these types

There are three types when you define a function in cython, the general difference between these types are:

- def function is called from Python code with Python objects as arguments, returns a python object.
- cdef is used for Cython function that are intended to be pure 'C' function. **All types must be declared and it is not visible to python code when import the module include this function**
- cpdef functions combine both def and cdef features.

Here will give you a intuitive feeling to the performance if you write in different types.

<!--more-->

#### source files

a python file

```py
# Fibo.py
# a common python file
def fib(n):
	if n < 2:
		return n
	return fib(n - 2) + fib(n - 1)
```

a optimize file which contains different type of function

```py
# cyFibo.pyx
# normal function
def fib(n):
	if n < 2:
		return n
	return fib(n - 2) + fib(n - 1)

# tell the compiler the type of n
def fib_int(int n):
	if n < 2:
		return n
	return fib_int(n - 2) + fib(n - 1)

# this function will call a nearly c function wirte below
def fib_cdef(int n):
	return fib_in_c(n)

# try to generate a pure c code and it is not visible to the python so it will be called by 
# fib_cdef function
cdef int fib_in_c(int n):
	if n < 2:
		return n
	return fib_in_c(n -2) + fib_in_c(n - 1)

# function can be called  by python and cython
cpdef fib_cpdef(int n):
	if n < 2:
		return n
	return fib_cpdef(n - 2) + fib_cpdef(n - 1)
```

setup file for cyFibo.pyx

```py
# setup.py
from distutils.core import setup
from Cython.Build import cythonize

setup(
	ext_modules = cythonize(
		'cyFibo.pyx'
		)
)
```



#### run the code

run the command below to generate the module can be called:

> python setup.py build_ext --inplace

run these command under python3.5 to check the performance of these code

> python -m timeit -s "import Fibo" "Fibo.fib(30)"
> python -m timeit -s "import cyFibo" "cyFibo.fib(30)"
> python -m timeit -s "import cyFibo" "cyFibo.fib_int(30)"
> python -m timeit -s "import cyFibo" "cyFibo.fib_cdef(30)"
> python -m timeit -s "import cyFibo" "cyFibo.fib_cpdef(30)"

#### Result

Machine configuration:

> CPU: 2GHz
> Memory: 24G
> Virtual Processors: 4

the best performance of each function is shown in this table:

| Fibo.fib | cyFibo.fib | cyFibo.fib_int | cyFibo.fib_cdef | cyFibo.fib_cpdef |
| -------- | ---------- | -------------- | --------------- | ---------------- |
| 684ms    | 236ms      | 238ms          | 6.05ms          | 44ms             |

It's easy to have a conclusion that if we tell compiler the type of the variable and function, and call it from python, it will be fast.

There are some tips for improve the performance:

-  do loops in cython and more loops more relative faster
-  tell the compiler the type of the variables
-  try to write the key algorithm in 'pure' c code and then give a wrapper make it can be called from python
