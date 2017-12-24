---
layout: post
title: "Comparison of W&R performance in python"
date: 2016-12-22 10:20:15 +0800
comments: true
categories: python
tags: [python,cython]
---
Background here, we want to process a big dataset with a high speed, besides calculation, IO is a bottleneck, How to optimize these IO operation to a tolerable time is what we want to get, and there are many numpy and pandas calculations, so the basic data structure is DataFrame in Pandas, This article is organized according to the Peter's material.

<!--more-->

#### Tool used here and Machine configuration

python : 3.5.2 |Continuum Analytics, Inc.| (default, Jul  5 2016, 11:41:13) [MSC v.1900 64 bit (AMD64)]
pandas : 0.19.1
numpy  : 1.11.2

RAM:24G
Processor: Intel(R) Xeon(R) CPU	X7550 @2.00GHz 2.00GHz

#### Initialization

##### function and utilities

```python
import pandas as pd
import numpy as np
import sys
import contextlib
import time

@contextlib.contextmanager
def timethis(name):
    startTime = time.time()
    yield
    elapsedTime = time.time() - startTime
    print('===> {:50s} : {: >5,d} ms'.format(name, int(elapsedTime * 1000)))

def b2mb(num_bytes, si=False):
    "returns num_bytes as megabytes"
    mega_byte = 1000**2 # S.I k,MM, etc.
    mebi_byte = 1024**2 # mega binary, traditional computer

    # default ratio is 1024^2 bytes to megabytes
    ratio_b_mb = mebi_byte if (not si) else mega_byte
    
    return num_bytes / ratio_b_mb
   
    
def sizeof(py_obj):
    "returns the size of a python object in MB"
    return b2mb(sys.getsizeof(py_obj))


def sizeof_fmt(py_obj, fmt='{:,.0f}'):
    "returns the size of a python object in MB in nice human readable string"
    return fmt.format(sizeof(py_obj))
```

##### create test data

```python
# make the test data, a np array of doubles
# records_n = 200000000
records_n = 75000000
test_data = np.array(np.arange(records_n),dtype='float64')
print('test_data is a np array of doubles')
print('  records_n  : {:15,d}'.format(records_n))
print('  size       : {:15,.0f} MB'.format(sizeof(test_data)))

# wrap in dataframe (note that this is a copy operation)
test_data_df = pd.DataFrame({'a' : test_data })

print('test_data_df is a pandas dataframe containing single column test_data')
print('  shape(m,n) : {:15,d}{:5,d} col'.format(*test_data_df.shape))
print('  size       : {:15,.0f} MB'.format(sizeof(test_data_df)))
```



#### Write

##### pandas.to_csv

```python
with timethis('time of pandas.to_csv'):
    test_data_df.to_csv('testdata_pandas.csv', float_format='%15.10e', index=False, header=False)
    
with timethis('time of pandas.read_csv'):
    pd.read_csv('testdata_pandas.csv')
```

For peter, it takes 7min 28s on SSD disk, for me, I don't know for now, it still running - -,OK, it's done!

|       | Peter | Leo    |
| ----- | ----- | ------ |
| write | 448 s | 677 s  |
| read  | N/A   | 34.8 s |


##### cython

**tips:** need run it in cython environment

```cython
cimport libc.stdio as stdio
cimport libc.stdlib as stdlib
cimport cython

@cython.boundscheck(False)
@cython.cdivision(False)
@cython.wraparound(False)
@cython.nonecheck(False)
cpdef cy_to_csv(double [:] a, filename):
    
    cdef stdio.FILE* fp
    cdef Py_ssize_t i 
    
    # open the file for writing, text mode, create if new, overwrite if existing
    fp = stdio.fopen(filename, "w")
    
    for i in range(a.shape[0]):
        # write in scientific notation e to handle any scaling         
        stdio.fprintf(fp, "%15.10e\n", a[i])
        
    
    stdio.fclose(fp)
    
    
@cython.boundscheck(False)
@cython.cdivision(False)
@cython.wraparound(False)
@cython.nonecheck(False)
cpdef cy_from_csv(double [:] a, filename):
    
    cdef stdio.FILE* fp
    cdef Py_ssize_t i 
    
    # open the file for writing, text mode, create if new, overwrite if existing
    fp = stdio.fopen(filename, "r")
    
    for i in range(a.shape[0]):
        # read floating point (including scientific notation)         
        stdio.fscanf(fp, "%lf\n", &a[i])
        
    stdio.fclose(fp)
```

|       | Peter  | Leo    |
| ----- | ------ | ------ |
| write | 55.3 s | 89.8 s |
| read  | N/A    | 57.04  |

No surprise here.

##### sqlite

There is no comparison on this, it takes 12 min on peter's machine, I did not do it on mine!

##### pandas dataframe, to_records and then numpy save

```python
# put the two step process into a single function
def npy_save_df(filename, df):
    "converts df to recarray and then saves to disk "
    
    # step 1: create a recarray from the df
    tmp_recarray = df.to_records()
    
    # step 2: save ndarray recarray to disk using save
    np.save(filename, tmp_recarray)

    
def npy_read_df(filename):
    "returns a pandas dataframe from saved recarray, see npy_save_df()"
   
	# step 1: read recarray from disk
    tmp_recarray = np.load(filename)
    
    # step 2: convert recarray to dataframe
    return np.from_records(tmp_recarray)
```

save the test_data directly with np.save to a .npy file:

|       | Peter  | Leo    |
| ----- | ------ | ------ |
| write | 1.64 s | 8.07 s |
| read  | N/A    | 2.61 s |

save the test_data_df with the function we wrote:

|       | Peter  | Leo    |
| ----- | ------ | ------ |
| write | 4.09 s | 26.06s |

##### HDF5

```python
def hdf5_save_df(filename, dfname, df):
    "stores df to hdf5 filename with name dfname (uses pd.HDFStore())"
    
    file_hdf5 = pd.HDFStore(filename)
    file_hdf5[dfname] = df
    file_hdf5.flush()
    file_hdf5.close()

    
def hdf5_read_df(filename, dfname, df):
    "returns df dfname from hdf5 filename (uses pd.HDFStore())"
    
    file_hdf5 = pd.HDFStore(filename)
    df = file_hdf5[dfname]
    file_hdf5.close()
    
    return df  
```

|       | Peter  | Leo     |
| ----- | ------ | ------- |
| write | 3.92 s | 22.58 s |
| read  | N/A    | 2.82 s  |

#### Conclusion

for write piece, write the raw data to .npy file is the fastest in these methods, and save dataframe to file, HDF5 is very efficient.

And another conclusion is my VDI is not so good as peter's! - -!