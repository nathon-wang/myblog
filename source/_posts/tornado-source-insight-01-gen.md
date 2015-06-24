title:  "Tornado源码源码分析系列之一: 化异步为'同步'的gen模块"
date: 2015-06-24 10:37:07
tags:
    - Python
    - Tornado
---

用Tornado也有一段时间，Tornado的文档还是比较匮乏的，但是幸好其代码短小精悍，很有可读性，遇到问题时总是习惯深入到其源码中。
这对于提升自己的Python水平和对于网络及HTTP的协议的理解也很有帮助。本文是Tornado源码系列的第一篇文章，网上关于Tornado源码分
析的文章也不少，大多是从Event loop入手，分析Event loop的工作原理，以及在其上如何构建TCPServer和HTTPServer。所以我就不想拾前
人的牙慧再去写一遍，当然这些内容我后续会涉及到，但是做为开篇第一章，我想从更加独特的角度来分析Tornado，这里就说说Tornado的gen
和concurrent两个模块， 这个话题网上似乎还不多，呵呵。

设计从需求出发，要考证一段的代码为什么写成这样而不是那样， 首先要看代码解决了什么需求。 看下代码中的例子先:

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

经过gen。coroutine修饰之后上面的这段代码可以改为

~~~python
class GenAsyncHandler(RequestHandler):
    @gen.coroutine
    def get(self):
        http_client = AsyncHTTPClient()
        response = yield http_client.fetch('http://example.com')
        do_something_with_response(response)
        self.render('template.html')
~~~

初识这段代码觉得好神奇，其实gen。coroutine只不过是将一个基于callback的典型的异步调用适配成基于yield的伪同步，说是伪同步是因为代码流程上类
似同步，但是实际确实异步的。这样做有几个好处:
1。控制流跟同步类似，我们知道callback里去做控制流还是比较恶心的，就算nodejs里的async这样的模块，但是分支多起来也非常不好写。(爽)
2。可以共享变量，没有了callback，所有的本地变量在同一个作用域中。 (爽爽)
3。可以并行执行，yield可以抛出list或dict，并行执行其中的异步流程。(爽爽爽。。。此处省略一万个爽)

神奇的gen。coroutine装饰器是怎么做到这一切的？让我首先买个关子，不是进入到gen里面分析coroutine和Runner这两核心的方法(类)，而是首先分析一些这
些方法(类)中用到的一些技术， 然后再回到coroutine装饰器和Runner类中。首先要理解的是generator是如何通过yield与外界进行通信的。

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

步骤1启动了generator，步骤2向generator内部发送数据，并通过yield向generator外部抛出结果10， 最后的执行结果是

~~~python
step 1.......
step 2....... 20
yield out ..... 10
~~~

然后让我再说说Future，Future是对异步调用结果的封装。一个callback型的异步调用的执行结果不仅包括调用的返回，还包括返回之后需要的执行回调，所以才需要将
异步调用的结果封装以下，作为一个结果的占位符。Future基本可以这么写

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

当然这只是个简约版的，详细可以考虑concurrent。Future。最后再来说说另一个重要的函数Task， 这是将一个callback型的函数适配成一个返回Future的函数，而这个Future
会在调用callback时解析

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

这里忽略了一些与本文无关的部分。可以看到Task里面构造了一个callback，_argument_adapter是将callback的参数进行适配，将不定参数适配成一个参数也就是result， 最后通过
future。set_result(result)将result赋值给future，这样future就被解析出来。 那么问题来了，AsyncHTTPClient并没有经过Task的适配，而是直接返回一个Future。这个Future是在
什么时候解析的呢。让我们深入httpclient。py，来看下AsyncHTTPClient是如何解析Future的，这是simplehttpclient。py中的fetch函数

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

fetch中定义一个代表fetch执行结果的future，如果调用时传入了callback，并不是直接将callback传给fetch_impl，而是再封装一下成handle_future，而实际的callback则换成handle_response，
fetch_impl最后会在当收到response的时候调用handle_response回调(这个有兴趣可以看下，如果以后有写httpserver相关的分析可能会再分析),
handle_response会解析出代表执行结果的future。对没有设置callback的调用，future解析结束整个流程也就结束了。而对于设置了callback的调用，future完成之后会调用handle_future，
而handle_future通过io_loop调用原始的我们传入的callback。 至此可以看到AsyncHTTPClient是如何把一个callback型的异步调用转换成一个返回future的异步调用，而这个future会在handle_response
调用时被解析得到返回的response。

好了，现在让我们来深入gen.coroutine这个装饰器以及其最终实现Runner类。对于返回future的异步调用，
yield会将这个future抛出，而coroutine装饰器会拿到这个future并进行相关的操作coroutine装饰器会调用_make_coroutine_wrapper方法， 我们只
看其中判断func(也就是被装饰的那个函数)返回generator的部分

~~~python

try:
    yielded = next(result)
except (StopIteration， Return) as e:
    future.set_result(getattr(e, 'value', None))
except Exception:
    future.set_exc_info(sys.exc_info())
else:
    Runner(result, future, yielded)
~~~

result就是被装饰的函数返回的generator，next启动这个generator， 如果generator抛出StopIteration和Return两个异常，表示generator已经解析出结果，将这个结果设置给最后coroutine返回的
future。如果有其他异常表示generator执行过程中发生了异常，将异常设置到future中。排除这两种情况，表示generator还没有执行完毕，调用Runner执行generator。Runner的参数result就是还没
有运行完毕generator， future是代表coroutine执行结果的那个future， 而yielded是func返回的future(或者YieldPoint，咱们只考虑future的情况)。再深入到Runner中，主要有两个函数handle_yield
和run，handle_yield主要是确定generator返回的yielded是否是一个执行完成的yielded(对于yielded是future的情况来说就是future.is_ready() == True)，如果没有执行完成则需要设置future完成时
执行run方法，也就是future.add_done_callback(future, lambda f:self.run())并返回False也就是不执行马上run， 否则返回True并立即执行run方法，因为这时候已经有异步调用的结果了。
run方法拿到yielded的执行结果，并传入到generator中。这样generator内部就能通过yield拿到异步调用的执行结果了。

~~~python
 def handle_yield(self, yielded):
    #处理YieldPoint忽略掉，但是原理跟Future是一样的
    try:
        self.future = convert_yielded(yielded)
    except BadYieldError:
        self.future = TracebackFuture()
        self.future.set_exc_info(sys.exc_info())

    if not self.future.done() or self.future is moment:
        self.io_loop.add_future(
            self.future, lambda f: self.run())
        return False
    return True

def run(self):
        if self.running or self.finished:
            return
        try:
            self.running = True
            while True:
                future = self.future
                if not future.done(): #执行run时generator返回的那个future必须已经有结果，否则就没必要传回到generator中了
                    return
                self.future = None
                try:
                    orig_stack_contexts = stack_context._state.contexts
                    exc_info = None

                    try:
                        value = future.result()
                    except Exception:
                        self.had_exception = True
                        exc_info = sys.exc_info()

                    if exc_info is not None:
                        yielded = self.gen.throw(*exc_info)
                        exc_info = None
                    else:
                        yielded = self.gen.send(value)

                    if stack_context._state.contexts is not orig_stack_contexts:
                        self.gen.throw(
                            stack_context.StackContextInconsistentError(
                                'stack_context inconsistency (probably caused '
                                'by yield within a "with StackContext" block)'))
                except (StopIteration, Return) as e:
                    #generator执行完毕并成功的处理
                except Exception:
                    #generator执行过程中异常的处理
                if not self.handle_yield(yielded):
                    #这里generator还没有执行完毕，yielded是generator迭代过一次之后返回的新yielded。如果yieled还没有被解析出结果就通过handle_yield给yieled设置完成时的重启run的回调,
                    #否则yielded已经有结果，就再次运行run，所以run中才会有一个循环
                    return
        finally:
            self.running = False
~~~

分析完毕，没看懂的同学可以在读两遍代码，主要还是要抓住coroutine装饰器只不过是将callback型调用转换成generator型伪同步调用的一个适配器这个关键点，阅读起代码来就明白多了。期待下篇吧，准备
写stack_context异步调用中的异常捕获问题

