---
layout: post
title: "How to config cython environment"
date: 2016-11-22 11:20:15 +0800
comments: true
categories: python
tags: [python,cython]
---
#### Things we need :

- conda(was included in your anaconda)
- python3.5
- vcvarsall.bat (install vs2015 community)
<!--more-->

#### Use conda to manage your python virtual environment

1. install python 3.5, we already have a default version (2.7.9),use the command below to create a new version in your virtual environment .
   > conda create --name snakes python=3.5

2. list all the python version:
   > conda info --env

3. if you want to use python3.5 instead of default version,activate it first with this command:
   > activate snakes

4. install cython in your 3.5 virtual environment :
   > conda install cython

tips: you can replace snakes with whatever name that you like

#### Configure compile environment for C files

1. install vs2015 community on your VDI
2. add ${VC directory path of vs2015}\vcvarsall.bat to the system path

#### How to compile a pyx file with cython

> http://cython.readthedocs.io/en/latest/src/tutorial/cython_tutorial.html

#### Compare the performance of c files and py files

1. save this script to py_fib.py and cy_fib.pyx file.

   ```py
   # py_fib.py
   def fib(n):
       """Print the Fibonacci series up to n."""
       a, b = 0, 1
       while b < n:
           a, b = b, a + b
   ```

   ```py
   # cy_fib.pyx
   def fib(long n):
       """Print the Fibonacci series up to n."""
       cdef long a = 0
       cdef long b = 1
       while b < n:
           a, b = b, a + b
   ```

2. create setup file for  cy_fib.pyx

   ```pyt
   from distutils.core import setup
   from Cython.Build import cythonize

   setup(
   	ext_modules = cythonize(
   		'cy_fib.pyx'
   		)
   )
   ```

3. write a test file for test.

   ```py
   import timeit
   import py_fib
   import cy_fib

   # py_test
   if __name__ == '__main__':
   	toc = timeit.default_timer()
   	for x in range(1,10000):
   		py_fib.fib(10000)
   	tic = timeit.default_timer()
   	print (tic - toc)
   ```

   ```py
   import timeit
   import py_fib
   import cy_fib

   # cy_test
   if __name__ == '__main__':
   	toc = timeit.default_timer()
   	for x in range(1,10000):
   		cy_fib.fib(10000)
   	tic = timeit.default_timer()
   	print (tic - toc)
   ```

4. run **python setup.py build_ext --inplace** to create a pyd file that can be call by a python scripts

5. run py_test.py and cy_test.py, on my vdi. py_test,cy_test,this is result of two scripts:

| py_test.py            | cy_test.py              |
| --------------------- | ----------------------- |
| 0.03114385974972276 s | 0.0018707819090167182 s |
