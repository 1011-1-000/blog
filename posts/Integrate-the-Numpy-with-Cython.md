---
layout: post
title: "Integrate the Numpy with Cython"
date: 2016-11-29 19:20:15 +0800
comments: true
categories: python
tags: [python,cython]
---
A demo for calculate convovle of an image to demonstrate How to use Numpy in cython, and give some tips for use it in Cython to speed up your code.

<!--more-->

#### Initial code for calculate the convovle

```python
from __future__ import division

import numpy as np

def naive_convolve(f,g):
	if g.shape[0] % 2 != 1 or g.shape[1] % 2 != 1:
		raise ValueError('Paramters error')

	vmax = f.shape[0]
	wmax = f.shape[1]
	smax = g.shape[0]
	tmax = g.shape[1]

	smid = smax // 2
	tmid = tmax // 2
	xmax = vmax + 2 * smid
	ymax = wmax + 2 * tmid

	h = np.zeros([xmax, ymax], dtype = f.dtype)

	for x in range(xmax):
		for y in range(ymax):
			s_from = max(smid - x, -smid)
			s_to   = min((xmax - x) - smid, smid + 1)
			t_from = max(tmid - y, -tmid)
			t_to   = min((ymax - y) - tmid, tmid + 1)
			value = 0
			for s in range(s_from,s_to):
				for t in range(t_from,t_to):
					v = x - smid + s
					w = y - tmid + t
					value += g[smid - s, tmid - t] * f[v, w]
			h[x, y] = value

	return h
```



The performance of this scripts is: 1.1 sec

Let us cythonize this script and run it again(we didn't make any changes on this script): 682 ms, we have gained 2 times faster than before, although we do nothing on it.

#### Adding Types

Now, we havn't typed the types of all variables in script, and we will add the types to this script to see how can we get.There are two ways to add the types to the script,first we would add a pxd file has the same name with python file, and when we cythonize the script,it will use pxd file as a declared file for each functions and variables and cythonize it.Here is pxd file for this script:

```pxd
import cython
cimport numpy as np

@cython.locals(vmax = cython.int, wmax = cython.int, smax = cython.int,
				tmax = cython.int, smid = cython.int, tmid = cython.int,
				xmax = cython.int, ymax = cython.int, x = cython.int, y = cython.int,
				s = cython.int, t = cython.int, v = cython.int, w = cython.int,
				s_from = cython.int, s_to = cython.int, t_from = cython.int, t_to = cython.int)
cpdef naive_convolve(np.ndarray f, np.ndarray g)
```

run it again,481 ms cost, so we get the a little faster than nothing to do on the script.Another way to add these types to the variables is typed directly in script,we can change script like this:

```python
from __future__ import division
import numpy as np
cimport numpy as np

DTYPE = np.int

ctypedef np.int_t DTYPE_t

cpdef np.ndarray naive_convolve(np.ndarray f, np.ndarray g):
    if g.shape[0] % 2 != 1 or g.shape[1] % 2 != 1:
        raise ValueError("Only odd dimensions on filter supported")
    assert f.dtype == DTYPE and g.dtype == DTYPE
    
    cdef int vmax = f.shape[0]
    cdef int wmax = f.shape[1]
    cdef int smax = g.shape[0]
    cdef int tmax = g.shape[1]
    cdef int smid = smax // 2
    cdef int tmid = tmax // 2
    cdef int xmax = vmax + 2*smid
    cdef int ymax = wmax + 2*tmid
    cdef np.ndarray h = np.zeros([xmax, ymax], dtype=DTYPE)
    cdef int x, y, s, t, v, w
    
    cdef int s_from, s_to, t_from, t_to
    
    cdef DTYPE_t value
    for x in range(xmax):
        for y in range(ymax):
            s_from = max(smid - x, -smid)
            s_to = min((xmax - x) - smid, smid + 1)
            t_from = max(tmid - y, -tmid)
            t_to = min((ymax - y) - tmid, tmid + 1)
            value = 0
            for s in range(s_from, s_to):
                for t in range(t_from, t_to):
                    v = x - smid + s
                    w = y - tmid + t
                    value += g[smid - s, tmid - t] * f[v, w]
            h[x, y] = value
    return h
```

performance of this script is 752ms,I don't know why, I think it should be faster than previous one, I mean write all declaration in a separate file should be slower.but I did not have answer on that.

#### Tell the dimention of a array to complier

A little change on the scripts is typed f,g in another way, we define the **np.ndarray f ** to **np.ndarray[DTYPE_T, ndim = 2]**,also g,then we try it again:

we get a beautiful number here if we write this in pxd file, 67.8 ms, also try in script, the performance is 12.9, faster than write in pxd file, it's normal now.

so we have improved the performance from 1 sec to 10ms.Great! ^_^.

#### tips:

- Q: How to run the code with numpy ndarray type?

- A: create a setup file like this, include the numpy lib

  ```python
  from distutils.core import setup, Extension
  from Cython.Build import cythonize
  import numpy

  setup(
      ext_modules=cythonize("convolve_cy.pyx"),
      include_dirs=[numpy.get_include()]
  ) 
  ```

- Q: How to run numpy code quickly?

- A: tell the dimention of array to avoid python [] operation