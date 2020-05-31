---
title: libuv-event-loop
date: 2020-05-31 19:16:03
tags: libuv,event-loop,nodejs
---
## 前言

这可能是一篇很长的文章，甚至是乏味的。这是由process.nextTick主题引起的。 Javascript开发人员可能每天都要面对setTimeout，setImmediate，process.nextTick，但是它们之间有什么区别？一些老年人可以告诉：

- setImmediate缺乏浏览器方面的实现，只有高版本的Internet Explorer才有意义。
- 首先触发process.nextTick，有时最后触发setImmediate。
- node环境已实现所有API。

从一开始，我就试图从Node官方文档以及其他文章中寻找答案。但不幸的是，我无法了解真相。其他说明可以在reddit黑板1、2中找到，Node的作者写了一篇文章试图解释逻辑。

我无法告诉有多少人知道真相并想探索它，如果您知道更多信息，请与我联系，我正在等待更多有用的信息。

本文不仅涉及事件循环，还涉及非阻塞I/O。我要说的第一件事是关于I/O模型。

### I/O模型

操作系统的I/O模型可以分为五类：阻塞I/O，非阻塞I/O，I/O复用，信号驱动I/O和异步I/O。我将以网络I/O为例，尽管仍然有文件I/O和其他类型，例如Node中的DNS相对和用户代码。以网络方案为例只是为了方便。

#### 阻止I/O

阻塞I/O是最常见的模型，但是有很大的局限性，显然，它只能处理一个流（流可以是文件，套接字或管道）。其流程图如下所示：

#### 非阻塞I/O

非阻塞I/O，又称繁忙循环，它可以处理多个流。应用程序进程反复调用系统以获取数据状态，一旦任何流的数据准备就绪，应用程序进程将阻止数据复制，然后处理可用数据。但这有浪费CPU时间的巨大缺点。其流程图如下所示：

#### I/O多路复用

选择和轮询基于此类型，请参阅有关选择和轮询的更多信息。 I/O复用保留了非阻塞I/O的优势，它还可以处理多个流，但它也是阻塞类型之一。呼叫选择（或轮询）将阻止应用程序进程，直到任何流准备就绪为止。更糟糕的是，它引入了另一个系统调用（recvfrom）。

注意：另一个密切相关的I/O模型是将多线程与阻塞I/O一起使用。该模型与上述模型非常相似，除了程序使用多个线程（每个文件描述符一个），而不是使用select阻止多个文件描述符，然后每个线程可以自由调用阻止系统调用，例如recvfrom。

其流程图如下所示：

#### 信号驱动的I/O

在此模型中，应用程序进程系统调用sigaction并安装信号处理程序，内核将立即返回，并且应用程序进程可以执行其他工作而不会被阻塞。当准备好读取数据时，将为我们的过程生成SIGIO信号。我们可以通过调用recvfrom从信号处理程序中读取数据，然后通知主循环该数据已准备就绪，或者可以通知主循环并让其读取数据。其流程图如下所示：

#### 异步I/O

异步I/O由POSIX规范定义，这是理想的模型。通常，像aio_ *函数之类的系统调用通过告诉内核开始操作并在整个操作（包括从内核到缓冲区的数据复制）完成时通知我们来起作用。该模型与上一节中的信号驱动I/O模型之间的主要区别在于，使用信号驱动I/O，内核会告诉我们何时可以启动I/O操作，但是使用异步I/O，内核会告诉我们I/O操作何时完成。其流程如下所示：

在介绍了这些类型的I/O模型之后，我们可以使用当前的热点技术来识别它们，例如linux中的epoll和OSX中的kqueue。它们更像是I/O多路复用模型，只有Windows上的IOCP实现了第五种模型。

然后看一下libuv，它是用C ++编写的I/O库。

### libuv

libuv是Node的依赖项之一，另一个是众所周知的v8。 libuv负责在不同操作系统上的不同I/O模型实现，并将它们抽象为一个用于第三方应用程序的API。 libuv有一个叫libev的兄弟，它的出现早于libuv，但libev不支持Windows平台。当Node变得越来越流行时，这变得越来越重要，因此他们必须考虑Windows的兼容性。最后，Node开发团队放弃了libev，取而代之的是libuv。

在我们进一步了解libuv之前，我应该给自己一个提示，我已经阅读了许多介绍JavaScript中的事件循环的文章和指南，它们似乎没有多大区别。我认为这些作者的意思是尽可能容易地解释这一概念。并且大多数不显示Webkit或Node的源代码。许多读者忘记了事件循环如何安排异步任务，并在再次遇到异步事件时遇到相同的问题。

首先，我想区分事件循环和事件循环迭代的概念。事件循环是一个与单个线程绑定的任务队列，它们是一对一的关系。事件循环迭代是运行时检查任务（代码段）在事件循环中排队并执行时的过程。这两个概念在`libuv`中映射了`tp`的两个重要功能/对象。一个是`uv_loop_t`，代表一个事件循环对象和`API：uv_run`，可以将其视为事件循环迭代的入口点。 `libuv`中所有以`uv_`开头的函数均以`uv_`开头，这确实使读取源代码变得容易。

#### uv_run

`libuv`中最重要的`API`是`uv_run`，每次调用此函数都可以进行事件循环迭代。 `uv_run`代码如下所示：

```c++
/**
 * 事件轮询核心
 */
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  int timeout;
  int r;
  int ran_pending;

  r = uv__loop_alive(loop);
  // 如果当前有存活handle或req
  if (!r)
    uv__update_time(loop);
  // 当前有存活handle或req 且 轮询未停止
  while (r != 0 && loop->stop_flag == 0) {
    uv__update_time(loop);
    // 执行timer队列任务，包含setTimeout setInterval
    uv__run_timers(loop);
    // 执行处于pending状态的异步I/O回调（上一个循环中被延迟的I/O回调）
    ran_pending = uv__run_pending(loop);
    // 系统空闲期 准备阶段
    uv__run_idle(loop);
    uv__run_prepare(loop);

    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);
    // 轮询阶段 timeout 最大为1789569ms ≈ 30mins，最多执行48个回调
    uv__io_poll(loop, timeout);
    // 检测 setImmediate，也就是下一loop开始前，本次loop末尾
    uv__run_check(loop);
    // close回调，如socket.on("close")
    uv__run_closing_handles(loop);

    if (mode == UV_RUN_ONCE) {
      /* UV_RUN_ONCE implies forward progress: at least one callback must have
       * been invoked when it returns. uv__io_poll() can return without doing
       * I/O (meaning: no callbacks) when its timeout expires - which means we
       * have pending timers that satisfy the forward progress constraint.
       *
       * UV_RUN_NOWAIT makes no guarantees about progress so it's omitted from
       * the check.
       */
      uv__update_time(loop);
      uv__run_timers(loop);
    }

    r = uv__loop_alive(loop);
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
  }

  /* The if statement lets gcc compile it to a conditional store. Avoids
   * dirtying a cache line.
   */
  if (loop->stop_flag != 0)
    loop->stop_flag = 0;

  return r;
}
void uv__run_timers(uv_loop_t* loop) {
  struct heap_node* heap_node;
  uv_timer_t* handle;

  for (;;) {
    heap_node = heap_min(timer_heap(loop));
    if (heap_node == NULL)
      break;

    handle = container_of(heap_node, uv_timer_t, heap_node);
    if (handle->timeout > loop->time)
      break;

    uv_timer_stop(handle);
    uv_timer_again(handle);
    handle->timer_cb(handle);
  }
}
static int uv__run_pending(uv_loop_t* loop) {
  QUEUE* q;
  QUEUE pq;
  uv__io_t* w;

  if (QUEUE_EMPTY(&loop->pending_queue))
    return 0;

  QUEUE_MOVE(&loop->pending_queue, &pq);

  while (!QUEUE_EMPTY(&pq)) {
    q = QUEUE_HEAD(&pq);
    QUEUE_REMOVE(q);
    QUEUE_INIT(q);
    w = QUEUE_DATA(q, uv__io_t, pending_queue);
    w->cb(loop, w, POLLOUT);
  }

  return 1;
}
```

每次运行时执行事件循环迭代时，它将执行如下图所示的有序代码，我们可以知道在每次事件循环迭代期间将调用哪种回调。事件循环迭代

正如`libuv`描述下划线原理一样，将在`uv__run_timers(loop)`步骤中调用计时器相关的回调，但是没有提及`setImmediate`和`process.nextTick`。显然，`libuv`是`Node`的下层，因此`Node`本身的逻辑将被考虑在内（更加灵活）。深入研究`Node`项目的源代码之后，我们可以看到`setTimeout/setInterval，setImmediate`和`process.nextTick`发生了什么。

Node
`Node`是`JavaScript`的流行和著名平台，本文不涉及任何其他技术书籍中有关Node的任何主要技术。如果您愿意，我愿意向您推荐Action。和精通Node.js的Node.js。

本文基于v4.4.0（LTS）显示的所有Node源代码。

### 设定

设置`Node`进程时，它还会设置事件循环。请参阅`src/node.cc`中的输入方法`StartNodeInstance`，如果事件循环仍然存在，则do-while循环将继续。

```c++
{
  SealHandleScope seal(isolate_);
  bool more;
  env->performance_state()->Mark(
      node::performance::NODE_PERFORMANCE_MILESTONE_LOOP_START);
  do {
    uv_run(env->event_loop(), UV_RUN_DEFAULT);

    per_process::v8_platform.DrainVMTasks(isolate_);

    more = uv_loop_alive(env->event_loop());
    if (more && !env->is_stopping()) continue;

    env->RunBeforeExitCallbacks();

    if (!uv_loop_alive(env->event_loop())) {
      EmitBeforeExit(env.get());
    }

    // Emit `beforeExit` if the loop became alive either after emitting
    // event, or after running some callbacks.
    more = uv_loop_alive(env->event_loop());
  } while (more == true && !env->is_stopping());
  env->performance_state()->Mark(
      node::performance::NODE_PERFORMANCE_MILESTONE_LOOP_EXIT);
}
```

### 异步用户代码

```c++
// timeout_vs_immediate.js
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => {
    console.log('timeout');
  }, 0);
  setImmediate(() => {
    console.log('immediate');
  });
});
```

这里的结果一定是 immediate timeout
从poll定义来看，这里接收到I/O 回调，所以此时轮询队列中不为空，所以遵循「轮询结束，往下执行检测阶段」

#### setTimeout

计时器对于Node.js至关重要。在内部，任何TCP I/O连接都会创建一个计时器，以便我们可以超时连接。此外，许多用户用户库和应用程序还使用计时器。因此，在任何给定时间可能安排了大量超时。因此，计时器的执行性能和效率非常重要。

在Node中，setTimeout和setInterval的定义位于lib/timer.js。为了提高性能，将超时值及其超时值存储在Key-Value结构中（JavaScript中的对象），key是超时值（以毫秒为单位），value是一个链表，其中包含所有共享相同超时值的超时对象。它使用src/timer_wrap.cc中定义的C++句柄作为链接列表中的每个项目。当您编写代码setTimeout（callback，timeout）时，Node初始化一个包含超时对象的链表（如果不存在）。超时对象的_timer字段指向一个TimerWrap实例（C ++对象），然后调用_timer.start方法将超时任务委托给Node。来自`lib/timer.js`的代码段在`JavaScript`中称为`setTimeout`时，将新建一个`TimersList()`对象作为链接列表节点，并将其存储在键值结构中。

```c++
function TimersList(expiry, msecs) {
  this._idleNext = this; // Create the list with the linkedlist properties to
  this._idlePrev = this; // Prevent any unnecessary hidden class changes.
  this.expiry = expiry;
  this.id = timerListId++;
  this.msecs = msecs;
  this.priorityQueuePosition = null;
}
```

//代码被忽略

list = new TimersList（毫秒，未引用）;

//代码被忽略

list._timer.start（msecs，0）;

list [msecs] = 列表；
list._timer [kOnTimeout] = listOnTimeout;

注意TimersList的构造函数，它初始化了一个TimerWrap对象，该对象构建了它的handle_字段，即uv_timer_t对象。 src/timer_wrap.cc中的代码段显示：

```c++
static void Initialize(Local <Object> object,
                       Local <Value> unused,
                       Local <Context> context){

  //代码被忽略

  env-> SetProtoMethod(constructor, "start", Start);
  env-> SetProtoMethod(constructor, "stop", Stop);
}

static void New(const FunctionCallbackInfo <Value>＆args){
  //代码被忽略
  
  new TimerWrap(env, args.This());
}

TimerWrap(Environment * env，Local <Object> object)
    ：HandleWrap(env,
                 object,
                 reinterpret_cast <uv_handle_t *>(＆handle_),
                 AsyncWrap::PROVIDER_TIMERWRAP){
  int r = uv_timer_init(env-> event_loop(), ＆handle_);
  CHECK_EQ(r, 0);
}
```

然后，TimerWrap将超时任务指定为其handle_。工作流程非常简单，Node不必担心超时回调的确切时间，这完全取决于libuv的时间表。我们可以在TimerWrap的Start方法（从客户端JavaScript代码中调用）中看到它，它调用uv_timer_start来安排超时回调。

```c++
static void Start(const FunctionCallbackInfo <Value>＆args){
  TimerWrap * wrap = Unwrap <TimerWrap>(args.Holder();

  CHECK(HandleWrap::IsAlive(wrap);

  int64_t timeout= args [0]-> IntegerValue();
  int64_t repeat = args [1]-> IntegerValue();
  int err = uv_timer_start(＆wrap-> handle_, OnTimeout, timeout, repeat);
  args.GetReturnValue().Set(err);
}
```

返回上图libuv部分，您可以在每个事件循环迭代中找到，event_loop可以安排超时任务本身，并通过更新event_loop的时间，可以准确地知道何时执行TimerWrap的OnTimeout回调。可以调用绑定的javascript函数。

```c++
static void OnTimeout(uv_timer_t* handle) {
  TimerWrap* wrap = static_cast<TimerWrap*>(handle->data);
  Environment* env = wrap->env();
  HandleScope handle_scope(env->isolate());
  Context::Scope context_scope(env->context());
  wrap->MakeCallback(kOnTimeout, 0, nullptr);
}
```

深入libuv源代码，每次uv_run调用在deps/uv/src/unix/timer.c中定义的uv__run_timers(loop)时，它都会比较uv_timer_t的超时时间和event_loop的时间，如果它们相互碰撞，则执行回调。

```c++
void uv__run_timers(uv_loop_t* loop) {
  struct heap_node* heap_node;
  uv_timer_t* handle;

  for (;;) {
    heap_node = heap_min((struct heap*) &loop->timer_heap);
    if (heap_node == NULL)
      break;

    handle = container_of(heap_node, uv_timer_t, heap_node);
    if (handle->timeout > loop->time)
      break;

    uv_timer_stop(handle);
    uv_timer_again(handle);
    handle->timer_cb(handle);
  }
}
```

注意上面的uv_timer_again（handle）代码，它与setTimeout和setInterval不同，但是两个API在下划线调用了相同的函数。

#### setImmediate

当您将JavaScript代码编写为setImmediate（callback）时，Node用setTimeout完成了大部分相同的工作。 setImmediate的定义也在lib / timer.js中，当调用此方法时，它将向处理对象添加私有属性_immediateCallback，以便C ++绑定可以标识已注册的回调代理，最后返回一个直接对象（您可以将其视为另一个对象）。超时对象，但没有超时属性）。在内部，timer.js维护一个名为InstantQueue的数组，其中包含当前事件循环迭代中安装的所有回调。

```c++
exports.setImmediate = function(callback, arg1, arg2, arg3) {
  // code ignored
  
  var immediate = new Immediate();

  L.init(immediate);

  // code ignored

  if (!process._needImmediateCallback) {
    process._needImmediateCallback = true;
    process._immediateCallback = processImmediate;
  }

  if (process.domain)
    immediate.domain = process.domain;

  L.append(immediateQueue, immediate);

  return immediate;
};
```

在Node设置事件循环之前，它将构建一个由所有当前过程代码共享的Environment对象（Environment类定义位于src / env.h中）。请参阅src / node.cc中的方法CreateEnvironment。 CreateEnvironment设置其事件循环使用的uv_check_t。片段显示了这一点。

```c++
Environment* CreateEnvironment(Isolate* isolate,
                               uv_loop_t* loop,
                               Local<Context> context,
                               int argc,
                               const char* const* argv,
                               int exec_argc,
                               const char* const* exec_argv) {

  // code ignored
  
  uv_check_init(env->event_loop(), env->immediate_check_handle());
  uv_unref(
      reinterpret_cast<uv_handle_t*>(env->immediate_check_handle()));

  // code ignored
  
  SetupProcessObject(env, argc, argv, exec_argc, exec_argv);
  LoadAsyncWrapperInfo(env);
  
  return env;
}  
```

方法src/node.cc中定义的SetupProcessObject，这可以使全局过程对象具有必要的绑定。从下面的代码中，我们可以发现它通过对NeedImmediateCallbackSetter方法调用need_imm_cb_string（）来分配一个访问器。如果我们查看src / env.h，我们知道need_imm_cb_string（）返回字符串：“_needImmediateCallback”。这意味着每次javascript代码在流程对象上设置“_needImmediateCallback”属性时，NeedImmediateCallbackSetter都会被调用。

```c++
void SetupProcessObject(Environment* env,
                        int argc,
                        const char* const* argv,
                        int exec_argc,
                        const char* const* exec_argv) {
  
  // code ignored
  
  maybe = process->SetAccessor(env->context(),
                               env->need_imm_cb_string(),
                               NeedImmediateCallbackGetter,
                               NeedImmediateCallbackSetter,
                               env->as_external());
  // code ignored

}  
```

在方法NeedImmediateCallbackSetter中，如果process._needImmediateCallback设置为true（由libuv的事件循环（env-> event_loop（））管理，并在CreateEnvironment方法中初始化），则它将启动uv_check_t句柄。

```c++
static void NeedImmediateCallbackSetter(
    Local<Name> property,
    Local<Value> value,
    const PropertyCallbackInfo<void>& info) {
  Environment* env = Environment::GetCurrent(info);

  uv_check_t* immediate_check_handle = env->immediate_check_handle();
  bool active = uv_is_active(
      reinterpret_cast<const uv_handle_t*>(immediate_check_handle));

  if (active == value->BooleanValue())
    return;

  uv_idle_t* immediate_idle_handle = env->immediate_idle_handle();

  if (active) {
    uv_check_stop(immediate_check_handle);
    uv_idle_stop(immediate_idle_handle);
  } else {
    uv_check_start(immediate_check_handle, CheckImmediate);
    // Idle handle is needed only to stop the event loop from blocking in poll.
    uv_idle_start(immediate_idle_handle, IdleImmediateDummy);
  }
}
```

最后，我们进入CheckImmediate，请注意，在timer.js中看到的Instant_callback_string方法将返回字符串：“ _ immediateCallback”。

```c++
static void CheckImmediate(uv_check_t* handle) {
  Environment* env = Environment::from_immediate_check_handle(handle);
  HandleScope scope(env->isolate());
  Context::Scope context_scope(env->context());
  MakeCallback(env, env->process_object(), env->immediate_callback_string());
}
```

因此我们知道，在每次事件循环迭代中，setImmediate回调都将在uv__run_check（loop）中执行；步骤紧跟着uv__io_poll（loop，timeout）;。如果有些困惑，可以按事件循环迭代执行顺序返回到该图。

#### process.nextTick

process.nextTick可能具有隐藏其实现^ _ ^的能力。我已经为节点项目发布了它，以获取更多信息。最后我自己关闭了它，因为我认为作者对我的问题有些困惑。调试到源代码也很有意义，我在源代码中添加日志，然后重新编译整个项目以查看发生了什么。

入口位于src / node.js中，有一个processNextTick方法可构建process.nextTick API。 process._tickCallback是必须由C ++代码正确执行的回调函数（或者，如果您使用require（'domain'），它将被process._tickDomainCallback覆盖）。每次您从JavaScript代码调用process.nextTick（callback）时，它都会维护nextTickQueue和tickInfo对象，以记录必要的刻度信息。

process._setupNextTick是src / node.js中的另一个重要方法，它映射到src / node.cc中名为SetupNextTick的C ++绑定函数。在此方法中，它采用第一个参数并将其tick_callback_function设置为存储在Environment对象上的Persistent。 tick_callback_function是精确执行绑定回调的函数，如javascript代码中所示。您可以从node.js中看到该代码段。注意，_combinedTickCallback调用绑定的回调。
```js
//使用tickInfo这个东西，以便src / node.cc中的C ++代码
//可以轻松访问我们的nextTick状态，并避免不必要的操作
//调用JS land。
const tickInfo = process._setupNextTick（_tickCallback，_runMicrotasks）;

function _tickCallback（）{
  var callback，args，tock;
  do{
    while（tickInfo [kIndex] <tickInfo [kLength]）{
      tock = nextTickQueue [tickInfo [kIndex] ++];
      回调= tock.callback;
      args = tock.args;
      //使用单独的回调执行函数可以直接
      //使用少量参数进行回调调用以避免
      //与使用`fn.apply（）相关的性能影响
      _combinedTickCallback（args，callback）;
      如果（1e4 <tickInfo [kIndex]）
        tickDone（）;
    }
    tickDone（）;
    _runMicrotasks（）;
    generatePendingUnhandledRejections（）;
  } while（tickInfo [kLength]！== 0）;
}
```

来自node.cc的SetupNextTick方法中的片段显示，当当前滴答声传输到下一个短语时，env对象将获得从参数传递的tick_callback_function作为回调。

```c++
env->SetMethod(process, "_setupNextTick", SetupNextTick);

void SetupNextTick(const FunctionCallbackInfo<Value>& args) {
  Environment* env = Environment::GetCurrent(args);

  CHECK(args[0]->IsFunction());
  CHECK(args[1]->IsObject());

  env->set_tick_callback_function(args[0].As<Function>());
  env->SetMethod(args[1].As<Object>(), "runMicrotasks", RunMicrotasks);

  // Do a little housekeeping.
  env->process_object()->Delete(
      env->context(),
      FIXED_ONE_BYTE_STRING(args.GetIsolate(), "_setupNextTick")).FromJust();

  // code ignored

  args.GetReturnValue().Set(Uint32Array::New(array_buffer, 0, fields_count));
}
```

因此，我们只需要搜索何时调用tick_callback_function（）即可。一段时间后，我发现可以从两个地方调用env（）-> tick_callback_function（）。首先是src / env.cc中定义的Environment :: KickNextTick。它由src / node.cc中的node :: MakeCallback调用，仅内部由API调用。其次是src / async_wrap.cc中定义的AsyncWrap :: MakeCallback。与作者提到的相同，只有AsyncWrap :: MakeCallback可以被公共调用。

我添加了一些日志以查看内部发生了什么。仅知道在每个事件循环结束阶段运行的回调对我来说还不够。最后，我发现每个AsyncWrap都是异步操作的包装，TimerWrap继承自AsyncWrap。超时处理程序执行回调OnTimeout时，实际上会执行AsyncWrap :: MakeCallback。您可以在setTimeout部分中看到之前显示的相同代码：

```c++
static void OnTimeout(uv_timer_t* handle) {
    TimerWrap* wrap = static_cast<TimerWrap*>(handle->data);
    Environment* env = wrap->env();
    HandleScope handle_scope(env->isolate());
    Context::Scope context_scope(env->context());
    wrap->MakeCallback(kOnTimeout, 0, nullptr);
  }
```

现在似乎是漆黑的日子，uv_run中的每个阶段都可以是事件循环迭代的最后阶段，例如超时，它检查并执行env（）-> tick_callback_function（）。另一个API setImmediate以内部调用的节点node :: MakeCallback结尾，该节点执行相同的工作。 StartNodeInstance的最后一部分EmitBeforeExit（env）和EmitExit（env）也将调用node :: MakeCallback，以确保tick_callback_function可以在进程退出之前被调用。

文件I/O

网络I/O

### 结论

当调用setTimeout和setImmediate时，它会将回调函数调度为要在下一个事件循环迭代中执行的任务。但是nextTick不会。它会在当前事件循环迭代结束之前被调用。我们还可以预见，如果我们递归调用nextTick，则超时任务将没有机会在此过程中执行。