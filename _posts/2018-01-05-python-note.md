---
layout: post
title: Python学习笔记
date: 2018-01-05
updated: 2018-01-05
author: Nicholas Huang
categories: Python
tags:
    - Python
--- 
# Python学习笔记
## 装饰器
### 利用装饰器获取函数运行时间
```
import time
def exeTime(func):
    def newFunc(*args, **args2):
        t0 = time.time()
        print "%s, {%s} start" % (time.strftime("%X", time.localtime()), func.__name__)
        back = func(*args, **args2);
        print "%s, {%s} end" % (time.strftime("%X", time.localtime()),func.__name__)
        print "%.3fs taken for {%s}" % (time.time() - t0, func.__name__)
    return back
return newFunc
    
@exeTime
def foo():
    for i in xrange(1000000):
        pass
```
还有一种就是超过一定时间打印bad，没超过打印good

```
def exeTime(t):
    def runFunc(func):
        def newFunc(*args, **args2):
            start = time.clock()
            back = func(*args, **args2)
            end = time.clock()
            if end - start > t:
                print 'bad'
            else:
                print 'good'
            return back
        return newFunc
    return runFunc
```

