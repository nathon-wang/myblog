title:  "Tornado源码分析系列之二: 让异常无处可逃的stack_context"
date: 2015-06-24 10:37:07
tags:
    - Python
    - Tornado
---

上一篇文章讨论了一下gen和Future，这次讨论一下Tornado的另一个特色的机制stack_context。首先看下异步调用过程中出现的问题。

~~~python
    def dosomething():
        def do_cb():
            raise ValueException()

        try:
            do_async(callback=do_cb)
        except ValueException:
            deal_with_exception()
~~~

上面的方法里try...except能捕获到do_async抛出的ValueException异常但是不能捕获do_cb中抛出的异常。因为do_cb并没有立即执行，只是被
放到IOLoop中，在适当的实际调用。而在do_cb调用的时候早就不在try...except块中。那么怎么才能做到统一的处理的，可以将try...except
部分封装起来，像下面这样。

~~~python
    def wrap(func):
        def inner(*args, **kwargs)
            try:
                return func(*args, **kwargs)
            except ValueException:
                deal_with_exception()
        return inner

    def dosomething():
        def do_cb():
            raise ValueException()

        wrap(do_async(callback=wrap(do_cb)))
~~~

stack_context就是上面的这种解决方案，不过使用的是contextmanager这一python中特性。contextmanger字面意思就是上下文管理器。
Python中将资源的分配和回收放到上下文管理器中，当with contextmanger_instances as inst 时，相当与调用contextmanger
的\_\_enter\_\_ 方法并将\_\_enter\_\_方法的返回值赋予as后面的变量并在with语句结束时调用contextmanger的\_\_exit\_\_方法。
