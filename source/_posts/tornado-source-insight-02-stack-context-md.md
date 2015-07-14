title:  "Tornado源码分析系列之二: 让异常无处可逃的stack_context"
date: 2015-06-24 10:37:07
tags:
    - Python
    - Tornado
---

上一篇文章讨论了一下gen和Future，这次讨论一下Tornado的另一个特色的机制stack\_context。首先看下异步调用过程中出现的问题。

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
放到IOLoop中，在适当的时机调用。而在do_cb调用的时候早就不在try...except块中。那么怎么才能做到统一的处理的，可以将try...except部分封装起来，
像下面这样。

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

stack\_context就是使用了上面的这种解决方案，当然作为一个框架代码stack\_context.py逼格更高一点，使用contextmanager这一python中特性来实现上述功能。 contextmanger字面意思就是上下文管理器。 Python中将资源的分配和回收放到上下文管理器中，当with contextmanger\_instances as inst 时，相当于
调用contextmanger的\_\_enter\_\_方法并将\_\_enter\_\_方法的返回值赋予as后面的变量inst，并在退出with语句块时调用contextmanger的\_\_exit\_\_方法。
定义了\_\_enter\_\_和\_\_exit\_\_这两个方法的类就是一个contextmanger，也可以说这两个方法就是python里contextmanger的协议。不多说，直接上代码

~~~python
class StackContext(object):
    #注意，这个里context_factory就是我们实际想进入的那个context的工厂方法，StackContext只是这个context的管理器。
    #也就是说拿管理器来替代实际想进入的context。对于异常来说这里的context_factory就是exception_handler。
    def __init__(self, context_factory):
        self.context_factory = context_factory
        self.contexts = []
        self.active = True

    #这个方法在__enter__中返回， 如果你不想这个StackContext往里(下)传播，就应该with StackContext(some_method) as deactivate, 然后在with块
    #内部调用deactivate方法。
    def _deactivate(self):
        self.active = False

    #调用传进来的那个context的__enter__方法，也就是进入了context中了，注意传进来的那个context才是主角，
    #StackContext只不过是个工具，维护传进来的那个context的工具
    def enter(self):
        context = self.context_factory()
        self.contexts.append(context)
        context.__enter__()

    #调用传进来的那个context的__exit__方法，也就是退出了context
    def exit(self, type, value, traceback):
        context = self.contexts.pop()
        context.__exit__(type, value, traceback)

    #这个是StackContext这个context的__enter__方法，这个里面主要干了三件事
    #1.保存还没有进入StackContext之前的context状态（因为进入当前的StackContext之前可能已经进入了好几个其他的StackContext)
    #2.更新当前的context状态，将当前的StackContext入栈
    #3.调用传进来的那个context的enter方法,进入该context
    def __enter__(self):
        self.old_contexts = _state.contexts
        #当前的StackContext进栈
        self.new_contexts = (self.old_contexts[0] + (self,), self)
        _state.contexts = self.new_contexts

        try:
            self.enter()
        except:
            _state.contexts = self.old_contexts
            raise

        return self._deactivate

    #这个是StackContext这个context的__enter__方法，这个里面主要干了两件事
    #1.调用传进来的那个context的exit方法,退出该context
    #2.恢复到调用该StackContext之前的context状态
    def __exit__(self, type, value, traceback):
        try:
            self.exit(type, value, traceback)
        finally:
            final_contexts = _state.contexts
            _state.contexts = self.old_contexts

            self.new_contexts = None
~~~

如果耐心点看上面的代码的话，就很容易知道，Tornado把context的管理抽象成一个状态机。每进入或退出一个StackContext状态机就发生一次变迁，
这也是为什么那个Thread.local的变量名字叫_state，_state[0]是历史路径上的所有的StackContext，_state[1]是当前的那个StackContext。如果你把
\_state[0]看成一个栈，那么_state[1]就是栈的top指针了。这是一个小栈，同时还有一个大栈，这个大栈是不同时期的_state组成的。_state这个变量
就是top指针，每个时期的_state[1]里面有一个叫old_contexts的指针指向上一个时期的栈，这个大栈同时也是一个后插法构建的链表。说白了，_state的
栈与普通栈的不同就是普通栈进栈出栈只是在一个栈里面。而_state栈进栈会复制一个栈，出栈是回退到进栈之前的那个栈。这么多个栈肯定有点晕了。
画个图理解一下。
![stack_context变迁图](/img/stack_context.png)
通过上面的StackContext使用语句with StackContext(some_factory)只是完成了我第二段代码里wrap(do_async)的操作，do_cb还没有wrap呢。
这个时候stack_context.wrap这个函数就出场了，这也是stack_context里面，唯一有点难度的代码，当然如果看懂了上面这个图，这段代码其实也很简单。
不多说，上代码。

~~~python
def wrap(fn):
    #防止wrap被重复调用
    if fn is None or hasattr(fn, '_wrapped'):
        return fn

    #这个变量很关键，是do_cb被wrap时的contexts。为啥要用list来装_state.contexts而不直接赋值呢？
    #其实很简单，因为python2.x中只能访问闭包中的变量不能修改。直到python3.x才有nonlocal关键字。
    #把_state.contexts放到列表里，列表本身不会被修改，但是列表中的内容却可以修改，tornado源码很多地方也用了此技术。
    cap_contexts = [_state.contexts]

    #如果当前没有contexts，也是就是_state == ((), None)。这个时候没有必要进行wrapped这个耗的操作，直接走捷径。
    if not cap_contexts[0][0] and not cap_contexts[0][1]:
        def null_wrapper(*args, **kwargs):
            try:
                current_state = _state.contexts
                _state.contexts = cap_contexts[0]
                return fn(*args, **kwargs)
            finally:
                _state.contexts = current_state
        null_wrapper._wrapped = True
        return null_wrapper

    #_state有内容则返回wrapped
    def wrapped(*args, **kwargs):
        ret = None
        try:
            #这个是函数do_cb执行时的contexts，也就是当前的contexts与上面的cap_contexts不是一回事，上面是生成闭包时，下面是执行闭包时
            #注意这个两个时间的不同。这两个时间之间可能已经进入和退出了很多的StackContext。这也是为什么要设计成进入
            #StackContext就要保存前一个_state栈并复制一份修改复制的而不是直接修改前一个，因为这样可以很方便回退到前面的_state。
            current_state = _state.contexts

            #删除那些deactivate被调用过的StackContext。前文也说过可以用deactivate取消StackContext的传播。
            #_remove_deactivated在下面单讲。
            cap_contexts[0] = contexts = _remove_deactivated(cap_contexts[0])

            #恢复状态到生成闭包时的，这样do_cb就和do_async处于同样的执行环境了。
            _state.contexts = contexts

            exc = (None, None, None)
            top = None

            #从这开始，开始执行_state[0]中的每一个context，执行其__enter__方法进入context，执行其__exit__方法退出context，处理执行过程中可能发生异常。
            last_ctx = 0
            stack = contexts[0]

            #按顺序进入context
            for n in stack:
                try:
                    n.enter()
                    last_ctx += 1
                except:
                    #context.__enter__也可能会发生异常，要处理这种情况。top指针指向发生了异常的前一个StackContext。
                    #因为发生enter异常的那个StackContext我们没有进入，所以也没有必要在异常发生的时候去调用exit退出。
                    #对比_handle_exception和处理fn调用时发生异常的top = contexts[1]来理解一下这段代码
                    exc = sys.exc_info()
                    top = n.old_contexts[1]

            #如果没有发生异常就执行fn，如果执行fn的过程中发生了异常，则top就是指向当前那个StackContext
            if top is None:
                try:
                    ret = fn(*args, **kwargs)
                except:
                    exc = sys.exc_info()
                    #因为当前的StackContext已经调用enter，所以top指向了当前的StackContext，在_handle_exception中调用exit
                    top = contexts[1]

            #如果发生了异常就处理异常
            if top is not None:
                exc = _handle_exception(top, exc)
            else:
                #反向调用context.__exit__
                while last_ctx > 0:
                    last_ctx -= 1
                    c = stack[last_ctx]

                    try:
                        c.exit(*exc)
                    except:
                        exc = sys.exc_info()
                        top = c.old_contexts[1]
                        break
                else:
                    top = None

                if top is not None:
                    exc = _handle_exception(top, exc)

            if exc != (None, None, None):
                raise_exc_info(exc)
        finally:
            这里恢复为执行do_cb之前的_state
            _state.contexts = current_state
        return ret

    wrapped._wrapped = True
    return wrapped

def _remove_deactivated(contexts):
    #首先移除掉contexts[0]中没有激活的context
    stack_contexts = tuple([h for h in contexts[0] if h.active])

    #上面只是移掉了当前_state里面的那些未激活的StackContext，但是没有移掉由_state[1].old_contexts链接的
    #那个大栈中的未激活的StackContext，下面的操作就是清理那个大栈（单链表）
    #首先是找到一个处于激活状态的header
    head = contexts[1]
    while head is not None and not head.active:
        head = head.old_contexts[1]

    #下面是单指针链表删除指定条件的节点。删除指定条件的节点，首先要找到其前一个节点，然后将这个节点的指针指向
    #要删除节点的下一个节点。对单链表不熟的同学可以google一下。
    ctx = head
    while ctx is not None:
        parent = ctx.old_contexts[1]

        while parent is not None:
            if parent.active:
                break
            ctx.old_contexts = parent.old_contexts
            parent = parent.old_contexts[1]

        ctx = parent

    return (stack_contexts, head)

def _handle_exception(tail, exc):
    #这里从发生异常的StackContext开始，顺着old_contexts指针向前传播异常并调用用exit退出进入过的context。
    #注意contextmanager可以通过context.__exit__返回True来阻止异常进一步的传播。这也是为什么tail.exit(*exc) == True
    #时要将exc设置为(None, None, None)
    while tail is not None:
        try:
            if tail.exit(*exc):
                exc = (None, None, None)
        except:
            exc = sys.exc_info()

        tail = tail.old_contexts[1]

    return exc
~~~

打个比喻好来说下stack_context的机制，就好像你出远门了，家里面每天吃什么你都不知道，于是你爸妈就把家里每天吃的都录下来，等你回家了按照录像里的重做一遍。
好了基本的都说的差不多了。ExceptionContext就是StackContext的简化版本，有兴趣的童鞋可以去看看，其实Tornado中用ExceptionContext用的更多一点，因为
异常处理的地方很多，使用StackContext管理资源的就少一点了。下一篇准备来聊聊IOLoop。

