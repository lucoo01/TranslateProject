
# 雾里看花Python之Asyncio


最近我开始发力钻研Python的新[asyncio][4]模块。原因是我需要做一些事情，使用事件IO会使这些事情工作得更好，炙手可热的asynio正好可以用来牛刀小试。从试用的经历来看，该模块比我预想的复杂许多，我现在有足够的信心说，我已经不知道如何恰当地使用asyncio了。


从Twisted框架借鉴一些经验来理解asynio并非难事，但是，asyncio包含众多的元素，我开始动摇，不知道如何将这些孤立的零碎拼图组合成一副完整的图画。我已没有足够的智力提出任何更好的建议，这在里，只想说出我的困惑，求大神指点。


#### 原语


<cite>asyncio</cite>通过coroutines辅助来实现异步IO。最初它是通过<cite>yield</cite>和<cite>yield from</cite>表达式实现的一个库，因为Python语言本身演进的缘故，现在它已经变成一个更复杂的怪兽。所以，为了在同一个频道讨论下去，你需要了解如下一些术语：

*  事件循环（event loops）
*   事件循环策略（event loop policies）
*   awaitables
*   协程函数（coroutine function）
*   老式协程函数（old style coroutine functions）
*   协程（coroutines）
*   协程封装（coroutine wrappers）
*   生成器（generators）
*   futures
*   并发的futures（oncurrent futures）
*   任务（tasks）
*   句柄（handles）
*   执行器（executors）
*   传输（transports）
*   协议（protocols）

此外，Python还新增了一些新的特殊方法：
*   aenter和aexit，用于异步块操作
*   aiter和anext，用于异步迭代器（异步循环和异步推导）。为了更多的乐趣，协议已经改变一次。 在3.5它返回一个awaitable（协程）；在Python 3.6它返回一个新的异步生成器。
*   await，用于原生的awaitables

你还需要了解相当多的内容，文档涵盖了那些部分。尽管如此，我做了一些额外说明以便对其有更好的理解：

### 事件循环


asyncio事件循环和你第一眼看上去的略有不同。表面看，每个线程都有一个事件循环，然而事实并非如此。我认为他们应该按照如下的方式工作：
*   如果是主线程，当调用asyncio.get_event_loop()时创建一个事件循环。
*   如果是其他线程，当调用asyncio.get_event_loop()时返回运行时错误。
*   当前线程可以使用asyncio.set_event_loop()，在任何时间节点绑定事件循环。该事件循环可由asyncio.new_evet_loop()函数创建。
*   事件循环可以在不绑定到当前线程的情况下使用。
*   asyncio.get_event_loop()返回绑定线程的事件循环，而非当前运行的事件循环。

这些行为的组合是超混淆的，主要有以下几个原因。 首先，你需要知道这些函数是全局设置的基础事件循环策略的委托。 默认是将事件循环绑定到线程。 或者，可以在理论上将事件循环绑定到一个greenlet或类似的，如果有人想要的话。 然而，重要的是要知道库代码不控制策略，因此不能推断asyncio将适用于线程。

其次，asyncio不需要通过策略将事件循环绑定到上下文。 事件循环可以单独工作。 但是这正是库代码的第一个问题，因为协同程序或类似的东西不知道哪个事件循环负责调度它。 这意味着，如果从协程中调用asyncio.get_event_loop()，你可能得不到运行你的事件循环。 这也是所有API均采用可选的显式事件循环参数的原因。 举例来说，要弄清楚当前哪个协程正在运行，不能使用如下调用：

```
def get_task():
    loop = asyncio.get_event_loop()
    try:
        return asyncio.Task.get_current(loop)
    except RuntimeError:
        return None

```

相反，必须显式地传递事件循环。 这进一步要求你在库代码中显式地遍历事件循环，否则可能发生很奇怪的事情。 我不知道这种设计的思想是什么，但如果不解决这个问题（例如get_event_loop()返回实际运行的事件循环），那么唯一有意义的其他方案是明确禁止显式事件循环传递，并要求它绑定到当前上下文（线程等）。

由于事件循环策略不提供当前上下文的标识符，因此库也不可能以任何方式“索引”到当前上下文。 也没有回调函数用来监视这样的上下文的拆除，这进一步限制了实际可以开展的操作。

### 等待与协同

以我的愚见，Python最大的设计错误是过度重载迭代器。 它们现在不仅用于迭代，而且用于各种类型的协程。 Python中迭代器最大的设计错误之一是如果 StopIteration没有被捕获形成的气泡。 这可能导致非常令人沮丧的问题，其中某处的异常可能导致其他地方的生成器或协同程序中止。 这是一个长期运行的问题，基于Python的模板引擎如Jinja，必须奋力解决。 模板引擎在内部渲染为生成器，并且当由于某种原因的模板引起StopIteration时，该渲染就会在那里结束。

Python正在慢慢学习过度重载这个系统的教训。 首先在3.x 版本加入asyncio模块，并没有语言支持。 所以自始至终它不过仅仅是装饰器和生成器。 为了实现yield from等，StopIteration再次重载。 这导致了令人困惑的行为，像这样：

```
>>> def foo(n):
...  if n in (0, 1):
...   return [1]
...  for item in range(n):
...   yield item * 2
...
>>> list(foo(0))
[]
>>> list(foo(1))
[]
>>> list(foo(2))
[0, 2]

```

没有错误，没有警告。 只是不是你期望的行为。 这是因为从一个作为生成器的函数中返回的值实际上引发了一个带有单个参数的StopIteration，它不是由迭代器协议捕获，而只是在协程代码中处理。

在3.5和3.6有很多改变，因为现在除了生成器对象我们还有协程对象。 它不是通过包装一个生成器来生成协程，而是用一个单独的对象直接创建协程。通过用给函数加async前缀来实现。 例如async def x（）会产生这样的协程。 现在在3.6，将有单独的异步生成器，它通过触发AsyncStopIteration保持其独立性。 此外，对于Python 3.5和更高版本，导入新的future对象（generator_stop），如果代码在迭代步骤中触发StopIteration，它将引发RuntimeError。

为什么我提到这一切？ 因为老的实现方式并未真的消失。 生成器仍然具有send和throw方法以及协程仍然在很大程度上表现为生成器。你需要知道这些东西，它们将在未来伴随你相当长的时间。

为了统一很多这样的重复，现在我们在Python中有更多的概念了：

*   awaitable：具有await方法的对象。 应用于由本地协同程序和旧式协同程序以及一些其他协同程序实现。
*   coroutinefunction：返回本地协同程序的函数。 不要与返回协程的函数混淆。
* a coroutine: 原生的协同程序。 注意，目前为止，本文档不认为老asyncio协程是协同程序。 至少我不认为inspect.iscoroutine是协程。 尽管它被未来式/等待分支接纳。

特别令人困惑的是asyncio.iscoroutinefunction和inspect.iscoroutinefunction正在做不同的事情，这与inspect.iscoroutine和inspect.iscoroutinefunction相同。 到得注意的是，尽管在类型检查中不知道有关asycnio遗留协同功能的任何信息，但是当您检查等待状态时它显然知道它们，即使它与await不一致。

### 协程包装器

每当你运行async def ，Python就会调用一个线程局部coroutine包装器。 它由sys.set_coroutine_wrapper设置，并且它是可以包装这些东西的一个函数。 看起来有点像如下代码：

```
>>> import sys
>>> sys.set_coroutine_wrapper(lambda x: 42)
>>> async def foo():
...  pass
...
>>> foo()
__main__:1: RuntimeWarning: coroutine 'foo' was never awaited
42

```

在这种情况下，我从来没有实际调用原始的函数，只是给你一个提示，说明这个函数可以做什么。 目前我只能说它总是线程局部有效，所以，如果替换事件循环策略，你需要搞清楚如何让coroutine封装在相同的上下文同步更新。创建新线程不会从父线程继承那些标识。

这不要与asyncio协程包装代码混淆。

### Awaitables和Futures

有些东西是awaitables。 据我所见，以下概念被认为是awaitable:

*   原生的协程
*   配置了CO_ITERABLE_COROUTINE标识的生成器（文中有涉及）
* 具有await方法的对象


除了生成器由于遗留的原因不是使用await方法，其他的对象都使用。 CO_ITERABLE_COROUTINE标志来自哪里？ 它来自一个协程包装器（现在与sys.set_coroutine_wrapper有些混淆），即@ asyncio.coroutine。 通过一些间接方法，它使用types.coroutine（现在与types.CoroutineType或asyncio.coroutine有些混淆）包装生成器，并通过另外一个标志CO_ITERABLE_COROUTINE重新创建内部代码对象。

所以现在我们知道这些东西是什么，什么是future？ 首先，我们需要澄清一件事情：在Python 3中，实际上有两种（完全不兼容）future类型：asyncio.futures.Future和concurrent.futures。 其中一个再现在另一个之前，但他们都仍然在asyncio使用。 例如，asyncio.run_coroutine_threadsafe（）将调度一个协程到在另一个线程中运行的事件循环，但它返回一个concurrent.futures。Future对象而不是asyncio.futures。Future对象。 这是有道理的，因为只有concurrent.futures.Future对象是线程安全的。

所以现在我们知道有两个不兼容的future，我们应该澄清哪个future在asyncio中。 老实说，我不完全确定差异在哪里，但我打算暂时称之为“最终”。 它是一个最终将持有一个值的对象，你可以在最终结果仍然在计算时做一些处理。 future对象的一些变种称为deferred，还有一些叫做promise。 我实在难以理解它们真正的区别。

你能用一个future对象做什么？ 你可以关联一个准备就绪时将被调用的回调函数，或者你可以关联一个失败时将被触发的回调函数。 此外，你可以等待它（它实现await，因此可以等待），也可以取消它。

那么你怎样才能得到这样的future对象？ 通过在await对象上调用asyncio.ensure_future。 它会把一个旧版的生成器转变为future对象。 然而，如果你阅读文档，你会读到asyncio.ensure_future实际上返回一个任务。 那么问题来了，什么是任务？

### 任务

任务是一个包装协同程序的future对象。 它的工作方式和future类似，但它也有一些额外的方法来提取当前栈中包含的协同程序。 我们已经看到了前面提到的任务，因为它是通过Task.get_current确定事件循环当前正在做什么的主要方式。

在如何取消工作上任务和future也有区别，但这超出了本文的范围。 取消是它们自己最大的问题。 如果你在一个协程上，并且知道自己正在运行，你可以通过前面提到的Task.get_current获取自己的任务，但这需要你知道自己被派遣在哪个事件循环，该事件循环可能是也可能不是已绑定的线程。

协程不可能知道它与哪个循环一起使用。任务也没有提供该信息公共API。 然而，如果你确实可以获得一个任务，你可以访问task._loop，通过它反指到事件循环。

### 句柄

除了上面提到的所有一切还有句柄。 句柄是等待执行的透明对象，不能等待，但可以被取消。 特别是如果你使用call_soon或者call_soon_threadsafe（还有其他一些）调度一个调用的执行，你可以通过句柄尽力尝试取消执行，但不能等待实际调用生效。

### 执行器

因为你可以有多个事件循环，但这并不意味着每个线程理所当然地应用多个事件循环，最常见的情形还是一个线程一个事件循环。 那么你如何通知另一个事件循环做一些工作？ 你不能到另一个线程的事件循环中执行回调函数并获取结果。 这种情况下，你需要使用执行器。

执行器来自concurrent.futures，它允许你将工作安排到本身未发生事件的线程中。 例如，如果在事件循环中使用run_in_executor来安编排将在另一个线程中调用的函数。 其返回结果是asyncio协程，而不是像run_coroutine_threadsafe这样的并发协同程序。 我还没有足够的心理能力来弄清楚为什么设计这样的API，应该如何使用，以及什么时候使用。 文档建议执行器可以用于构建多进程。

### 传输和协议

我总是认为传输与协议也凌乱不堪，实际这部分内容基本上是对Twisted的逐字拷贝。详情毋庸赘述也，请直接阅读相关文档。

### 如何使用asyncio

现在我们已经大致了解asyncio，我发现了一些模式，人们似乎在使用asyncio代码时使用：

*   将事件循环传递给所有协程。 这似乎是社区中一部分人的做法。 把事件循环信息提供给协程为协程获取自己运行的任务提供了可能性。
*   或者你要求事件循环绑定到线程，这也能达到同样的目的。 理想情况下两者都支持。 可悲的是，社区已经分化。
*   如果想使用上下文数据（如线程本地数据），你可谓是运气不佳。 最流行的变通方法显然是atlassian的aiolocals，它基本上需要你手动传递上下文信息到协程，因为解释器不提供支持。 这意味着如果你有一个实用程序库生成协程，你将失去上下文。
* 忽略Python中的旧协程。 只使用3.5版本中async def和co关键字。 你总可能要用到它们，因为在老版本中，没有异步上下文管理器，这是非常必要的资源管理。
*  学习重新启动事件循环进行善后清理。 这部分功能和我预想的不同，我花了比较长的时间来厘清它的实现。清理操作的最好方式是不断重启事件循环直到没有等待事件。 遗憾的是没有什么通用的模式来处理清理操作，你只能用一些丑陋的临时方案糊口度日。 例如aiohttp的web支持也做这个模式，所以如果你想要结合两个清理逻辑，你可能需要重新实现它提供的实用程序助手，因为该助手功能实现后，它彻底破坏了事件循环的设计。 当然，它不是我见过的第一个干这种坏事的库:(
*  使用子进程是不明显的。 你需要一个事件循环在主线程中运行，我想它是在监听信号事件，然后分派到其他事件循环。 这需要通过asyncio.get_child_watcher（）、attach_loop（...）通知循环。
* 编写同时支持异步和同步的代码在某种程度上注定要失败。 尝试在同一个对象上支持with和async with是危险的事情。
*  如果你想给coroutine一个更好的名字，弄清楚为什么它没有被等待，设置name没有帮助。 你需要设置qualname。
* 有时内部类型交换使你麻痹。 特别是asyncio.wait（）函数将确保所有的事情都是future，这意味着如果你传递协程，你将很难发现你的协程是否已经完成或者正在等待，因为输入对象不再匹配输出对象 。 在这种情况下，唯一真正理智的做法是确保前期一切都是future。

### 上下文数据

除了疯狂的复杂性和缺乏理解如何更好地编写API，我最大的问题是完全缺乏对上下文本地数据的考虑。这是Node社区现在学习的东西。continuation-local-storage存在，但实现太晚。连续本地存储和类似概念被定期用于在并发环境中实施安全策略，并且该信息的损坏可能导致严重的安全问题。

事实上，Python甚至没有任何存储，这令人失望至极。我正在研究这个内容，因为我正在调查如何最好地支持[Sentry's breadcrumbs] [3] 的asyncio，然而我并没有看到一个合理的方式做到这一点。在asyncio中没有上下文的概念，没有办法从通用代码中找出您正在使用的事件循环，并且如果没有monkeypatching（运行环境下的补丁），也无法获取这些信息。

Node当前正在经历如何[找到这个问题的长期解决方案] [2]的过程。这个问题不容忽视，因为它在所有生态系统中反复出现过，如JavaScript，Python和.NET环境。问题[被命名为异步上下文传播] [1]，其解决方案有许多名称。在Go中，需要使用上下文包，并明确地传递给所有goroutine（不是一个完美的解决方案，但至少有一个）。 .NET具有本地调用上下文形式的最佳解决方案。它可以是线程上下文，Web请求上下文或类似的东西，除非被抑制，否则它会自动传播。微软的解决方案是我们的黄金标准。我现在相信，微软在15年前已经解决了该问题。

我不知道生态系统是否还年轻，还可以添加逻辑调用上下文，可能现在仍然为时未晚。

### 个人感想

输出asynio的man帮助是复杂的，并且它变得越来越复杂。 我没有心理能力随便使用asyncio。 它需要不断地更新所有Python语言更新的知识，这很大程度上使语言本身变得复杂。 令人鼓舞的是，一个生态系统正围绕着它不断发展，只是不知道需要几年的时间，才能带给开发者愉快和稳定的开发体验。

3.5版本引入的东西（新的协程对象）非常棒。 特别是这些变化包括引入了一个合理的基础，这些都是我在早期的版本中一直期盼的。在我心中， 通过生成器实现协程是一个错误。 关于什么是asyncio，我难以置喙。 这是一个非常复杂的事情，内部令人眼花缭乱。 我很难理解它工作的所有细节。 你什么时候可以传递一个生成器，什么时候它必须是一个真正的协程，future是什么，任务是什么，事件循环如何工作，这甚至还没有触碰到真正的IO部分。

最糟糕的是，asyncio甚至不是特别快。 David Beazley演示的他设计的asyncio的替代品是原生版本速度的两倍。 asyncio巨复杂，很难理解，也无法兑现自己在主要特性上的承诺，对于它，我只想说我想静静。我知道，至少我对asyncio理解的不够透彻，没有足够的信心对人们如何构建代码给出建议。

--------------------------------------------------------------------------------

via: http://lucumr.pocoo.org/2016/10/30/i-dont-understand-asyncio/

作者：[Armin Ronacher][a]

译者：[firstadream](https://github.com/译者ID)

校对：[校对者ID](https://github.com/校对者ID)

本文由 [LCTT](https://github.com/LCTT/TranslateProject) 原创编译，[Linux中国](https://linux.cn/) 荣誉推出

[a]:http://lucumr.pocoo.org/about/
[1]:https://docs.google.com/document/d/1tlQ0R6wQFGqCS5KeIw0ddoLbaSYx6aU7vyXOkv-wvlM/edit
[2]:https://github.com/nodejs/node-eps/pull/18
[3]:https://docs.sentry.io/learn/breadcrumbs/
[4]:https://docs.python.org/3/library/asyncio.html