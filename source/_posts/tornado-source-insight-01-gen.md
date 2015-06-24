title:  "Tornado源码源码分析系列之一: 化异步为'同步'的gen模块"
date: 2015-06-24 10:37:07
tags:
    - Python
    - Tornado
---

用Tornado也有一段时间，Tornado的文档还是比较匮乏的，但是幸好其代码短小精悍，很有可读性，遇到问题时总是习惯深入到其源码中，
这对于提升自己的Python水平和对于网络及HTTP的协议的理解也很有帮助。这是Tornado源码系列的第一篇文章，网上关于Tornado源码分析的文章也不少，
大多是从Event loop入手，分析Event loop的工作原理，以及在其上如何构建TCPServer和HTTPServer。所以我就不想拾前人的牙慧，
再去分析一遍，当然这些内容我后续会涉及到，但是做为开篇第一章，我想从更加独特的角度来分析Tornado，这里就说说Tornado的gen和concurrent两
个模块， 这个话题网上似乎还不多，呵呵。

设计从需求出发，要考证一段的代码为什么写成这样而不是那样， 我们首先要看代码解决了什么需求。 让我们看下代码中的例子：

~~~ python
class AsyncHandler(RequestHandler):
    @asynchronous
    def get(self):
        http_client = AsyncHTTPClient()
        http_client.fetch('http://example.com', callback=self.on_fetch)

    def on_fetch(self, response):
        do_something_with_response(response)
        self.render('template.html')
~~~

经过gen.coroutine修饰之后上面的这段代码可以改为

~~~python
class GenAsyncHandler(RequestHandler):
    @gen.coroutine
    def get(self):
        http_client = AsyncHTTPClient()
        response = yield http_client.fetch('http://example.com')
        do_something_with_response(response)
        self.render('template.html')
~~~

可以看到gen.coroutine将一个基于callback的典型的异步调用适配成基于yield的伪同步，说是伪同步是因为代码流程上类似同步，但是实际确实异步的。
这样做有几个好处
1.控制流跟同步类似，我们知道callback里去做控制流还是比较恶心的，就算nodejs里的async这样的模块，但是分支多起来也非常不好写。
2.可以共享变量，没有了callback，所有的本地变量在同一个作用域中
3.可以并行执行，yield可以抛出list或dict，并行执行其中的异步流程.

神奇的gen.coroutine装饰器是怎么做到这一切的？我们首先不是进入到gen里面分析coroutine和Runner这两核心的方法和类，而是首先分析一些这些方法（类）
中用到的一些技术， 首先我们要理解generator，以及generator是如何通过yield与外界进行通信的。

~~~python
def test():
    print ('step 1.......')
    res = yield 10
    print ('step 2.......', res) (3)

gen = test()
gen.send(None) #next(gen)  (1)
data = gen.send(20) (2)
print ('yield out .....', data)
~~~

步骤1启动了generator，步骤2向generator内部发送数据，并通过yield向generator外部抛出结果10, 最后的执行结果是

~~~python
step 1.......
step 2....... 20
yield out ..... 10
~~~

其次让我再说说Future，Future是对异步调用结果的封装。一个callback型的异步调用的执行结果不仅包括调用的返回，还包括返回之后需要的执行回调，
所以一个Future基本可以这么写

~~~python
class Future(object):
    def __init__(self):
        self._callback = []
        self._result = None
        self._done = False

    def set_callback(self, cb):
        self._callback.append(cb)

    def _run_callback(self):
        for cb in self._callback:
            cb()

    def set_result(self, result)
        self._done = True
        self._result = result
        self._run_callback()

    def is_ready(self):
        return self._done is True
~~~

当然这只是个简约版的，详细可以考虑concurrent.Future。最后再来说说另一个重要的函数Task,
这是将一个callback型的函数适配成一个返回Future的函数，而这个Future会在调用callback时解析

~~~python
def Task(func, *args, **kwargs):
    future = Future

    def set_result(result):
        if future.done():
            return
        future.set_result(result)

    func(*args, callback=_argument_adapter(set_result), **kwargs)
    return future
~~~

这里忽略了一些与本文无关的部分。可以看到Task里面构造了一个callback，_argument_adapter是将callback的参数进行适配，将不定参数适配成一个参数也就是result, 最后通过future.set_result(result)将result赋值给future.这样future就被解析出来.

那么问题来了，AsyncHTTPClient并没有经过Task的适配，而是直接返回一个Future.这个Future是在什么时候解析的呢。让我们深入httpclient.py,来看下AsyncHTTPClient是如何解析Future的，首先是fetch函数

~~~python
def fetch(self, request, callback=None, raise_error=True, **kwargs):
    .....
    future = TracebackFuture()
    if callback is not None:
        callback = stack_context.wrap(callback)

        def handle_future(future):
            exc = future.exception()
            if isinstance(exc, HTTPError) and exc.response is not None:
                response = exc.response
            elif exc is not None:
                response = HTTPResponse(
                    request, 599, error=exc,
                    request_time=time.time()-request.start_time
                )
            else:
                response = future.result()
            self.io_loop.add_callback(callback, response)
        future.add_done_callback(handle_future)

    def handle_response(response):
        if raise_error and response.error:
            future.set_exception(response.error)
        else:
            future.set_result(response)
    self.fetch_impl(request, handle_response)
    return future
~~~

首先是定义一个future，如果callback设置了，并不是直接将callback传给fetch_impl，而是再封装一下成handle_future，而实际的callback则换成handle_response，
fetch_impl最后会当有response调用handle_response回调（这个有兴趣可以看下，如果以后有写httpserver相关的分析可能会再分析）
handle_response会解析出future，对没有设置callback的调用，future解析结束。对于设置了callback的调用，future完成之后会调用handle_future,
而handle_future通过io_loop调用原始callback. 至此我们可以看到如何把一个callback型的异步调用转换成一个返回future的异步调用，而这个future会在callback调用时被解析得到实际的值.

好了，现在让我们来深入gen.coroutine这个装饰器以及其最终实现Runner类。对于返回future的异步调用,
yield会将这个future抛出，而coroutine装饰器会那到这个future并进行相关的操作coroutine装饰器回调用_make_coroutine_wrapper方法, 我们只
看其中判断func返回generator的部分

~~~python

try:
    yielded = next(result)
except (StopIteration, Return) as e:
    future.set_result(getattr(e, 'value', None))
except Exception:
    future.set_exc_info(sys.exc_info())
else:
    Runner(result, future, yielded)
~~~

result就是我们被装饰的函数返回的generator，首先next启动这个generator, 如果generator抛出StopIteration和Return两个异常，表示generator有返回，
coroutine表示结果的future可以被解析。如果没有异常则进入Runner这个类里，Runner的参数result就是还没有运行完毕generator, future是经过coroutine
封装的func返回的future, yielded是func返回的future。再深入到Runner中，主要两个函数handle_yield和run

~~~python
    if self.handle_yield(first_yielded):
        self.run()
~~~

~~~

