---
layout: post
title:  "Python multiprocessing cheatsheet"
date:  2023-11-02 12:00:00 -500
tags: [python, programming]
categories: [python]
---
### Multiprocessing using pool map
The `Pool.map` function allows you to select a function and
an iterable containing the list of parameters that are to be fed to the function 
for each process :
```python
from multiprocessing import Pool
def square(number):
    return number**2

pool = Pool(processes=4)
result = pool.map(function, [1,2,3,4,5,6])
print(result)
# The result will be [1, 4, 9, 16, 25, 36]
```
