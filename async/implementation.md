## 实现

既然我们已经理解了rust中基于futures和async/await如何多任务协助．是时候向我们的内核中加入相关的支持了．因为`Future`trait是`core`库的一部分，并且async/await是语言本身的特性．所以我们不需要在我们的`![no_std]`内核中做什么额外的工作．唯一的要求是必须使用`2020-03-25`之后的nightly版本的rust，因为在这之前async/await不完全是`![no_std]`的．

使用足够新的nightly版本，我们可以在`main.rs`使用async/await

```rust
// in src/main.rs

async fn async_number() -> u32 {
    42
}

async fn example_task() {
    let number = async_number().await;
    println!("async number: {}", number);
}
```

`async_number`函数是一个`async`函数，所以编译器将它转换成一个实现了`Future`的状态机．因为函数只返回42，所以结果`Future`在第一次`poll`调用时将会直接返回`Poll:Ready(42)`．与`async_number`类似，`example_task`也是一个`async`函数，它等待由`async_number`返回的数然后使用println宏打印这个数.

为了运行由`example_task`返回的future，需要在它上面调用`poll`知道它返回`Poll:Ready`表示已经完成．为了做到这一点，我们要创建一个简单的执行器．

## Task

在实现执行器前，我们创建一个带有`Task`类型的`task`模块．

```rust
// in src/lib.rs

pub mod task;
```



```rust
// in src/task/mod.rs

use core::{future::Future, pin::Pin};
use alloc::boxed::Box;

pub struct Task {
    future: Pin<Box<dyn Future<Output = ()>>>,
}
```

`Task`结构体是一个包装了固定位置，带有空类型()作为输出，在堆上申请，动态分发的future.让我们来详细的理解它:

- 我们要求与一个task相关的future返回()．这意味着tasks不返回任何结果，他们只为了副作用被执行．举个例子，我们定义的`example_task`不返回任何值，但作为副作用，它在屏幕上打印一些东西.
- `dyn`关键字表明我们在`Box`中存放了一个`trait`对象．这意味着，这个future上的方法是动态分发的，使它能够在`Task`中存放不同的类型．这很重要因为每一个`async_fn`有它自己的类型，我们想要创建很多不同的tasks．
- 在所学的pinning章节中，`Pin<Box>`类型通过将它放在堆上并禁止创建它的可变引用来确保值不会在内存中移动．这很重要因为由async/await创建的future可能是自引用的，像包含指向自己的指针在future移动后会变为无效的．

为了从futures中创建新的`Task`结构，我们新建了`new`函数

```rust
// in src/task/mod.rs

impl Task {
    pub fn new(future: impl Future<Output = ()> + 'static) -> Task {
        Task {
            future: Box::pin(future),
        }
    }
}
```

这个函数接受一个任意输出类型为()的future并使用`Box:Pin`将它固定到内存中.然后将future包装在boxed的task中并返回．需要`	'static`因为返回的`Task`可能有存活任意时间，所以future需要在这个时间内有效．

我们也需要加入poll方法来使执行器调用存放的future:

```rust
// in src/task/mod.rs

use core::task::{Context, Poll};

impl Task {
    fn poll(&mut self, context: &mut Context) -> Poll<()> {
        self.future.as_mut().poll(context)
    }
}
```

因为`Future`trait的`poll`方法希望在`Pin<&mut T>`type是调用，我们使用`Pin::as_mut`方法将`self.future`域转换`Pin<Box<T>`.然后我们可以在转换过的`self.future`域上调用并返回结果．因为`Task:poll`只能被我们创建的执行器立即执行，我们使这个函数对task模块保持private.



## Simple Executor

因为执行器可能会很复杂，我们故意创建了一个很基础的执行器在我们实现更多的特性前，我们先创建一个`task::simple_executor`子模块

```rust
// in src/task/mod.rs

pub mod simple_executor;
```

```rust
// in src/task/simple_executor.rs

use super::Task;
use alloc::collections::VecDeque;

pub struct SimpleExecutor {
    task_queue: VecDeque<Task>,
}

impl SimpleExecutor {
    pub fn new() -> SimpleExecutor {
        SimpleExecutor {
            task_queue: VecDeque::new(),
        }
    }

    pub fn spawn(&mut self, task: Task) {
        self.task_queue.push_back(task)
    }
}
```

结构体只包含了一个`vecDeque`类型的`task_field`域，支持在两端进行pop和push操作.使用这个类型的原因是我们通过`spawn`方法将新任务加入到尾断，将下个要执行的任务从头部弹出．这样的话，我们得到了一个简单的fifo队列．

## Dummy Waker

为了调用`poll`方法，我们要创建一个包装了`Waker`的`Context`类型．从简单的开始，我们先创建了一个不做任何事的假waker．因此我们创建了一个`RawWaker`实例，它定义了不同`Waker`方法的实现，然后使用`Waker::from_raw`函数将它转换为`Waker`

```rust
// in src/task/simple_executor.rs

use core::task::{Waker, RawWaker};

fn dummy_raw_waker() -> RawWaker {
    todo!();
}

fn dummy_waker() -> Waker {
    unsafe { Waker::from_raw(dummy_raw_waker()) }
}
```

`from_raw`函数是不安全的，因为可能会发生为定义行为，如果不坚持文档中`RawWaker`的要求.在我们探究`dummy_raw_waker`函数的实现前	我们先试着理解RawWaker如果工作.

### RawWaker

`RawWaker`类型需要程序员显示定义一个虚函数表，虚函数表明确了函数RawWaker被克隆，唤醒或丢弃时要调用的函数．虚函数表的布局由`RawWakerTable`类型定义．每个函数需要接受一个`*const()`参数，那是一个基础的类型擦除的指向某些类型的指针，比如申请在堆上的类型．使用`*const()`类型指针而不是合适的引用的理由是RawWaker类型应该是非通用的但仍然支持任意类型．传递给函数的指针值是提供给 RawWaker::new 的`Data`指针．

典型的，`RawWaker`为了一个包装在`Box`或`arc`类型在堆上的结构，对这样的类型，像`Box::into_raw`方法可以将Box<T>转换为`*const T`指针．这个指针可以被转换为一个匿名的`*const()`指针并传递给RawWaker::new．因为每个虚函数接受同样的*const()作为参数，函数可以寒气的将指针转换回`Box<T>`或`&T`来操作他．如你所想，这个过程非常危险，很容易导致有关错误的未定义行为．因此．手动创建一个RawWaker是不推荐的除非必要．

### A dummy RawWaker

目前没有其他方法创建一个假的`Waker`由于手动创建RawWaker是不推荐的.幸运的是，我们什么都不想做使得实现一个`dummy_raw_waker`函数是相对安全的：

```rust
// in src/task/simple_executor.rs

use core::task::RawWakerVTable;

fn dummy_raw_waker() -> RawWaker {
    fn no_op(_: *const ()) {}
    fn clone(_: *const ()) -> RawWaker {
        dummy_raw_waker()
    }

    let vtable = &RawWakerVTable::new(clone, no_op, no_op, no_op);
    RawWaker::new(0 as *const (), vtable)
}

```

首先，定义两个内部函数`no_op`和`clone`．`no_op`函数使用`*const()`作为参数并没有做任何事．`clone`也接受一个`*const()`通过调用`dummy_raw_waker`并返回一个新的RawWaker.我们用这两个函数来创建一个最小的`RawWakerTable`.clone函数用来克隆操作，no_op用于其他操作．既然RawWaker什么也没做，我们从clone中返回一个新的RawWaker而不是复制他就是无关紧要的．

创建`vtable`后，我们用`RawWaker::new`函数创建`RawWaker`.传递`*const()`并不重要，因为没有虚函数使用它．因此我们简单的传递一个空指针．

### A `run` Method

现在我们有方法创建一个`Waker`实例，我们可以使用它在执行器实现`run`方法.最简单的run方法是重复poll所有任务队列直到所有的全部完成．这非常没有效率，因为没有利用`waker`类型的通知．但这是使代码运行起来的简单方法．

```rust
// in src/task/simple_executor.rs

use core::task::{Context, Poll};

impl SimpleExecutor {
    pub fn run(&mut self) {
        while let Some(mut task) = self.task_queue.pop_front() {
            let waker = dummy_waker();
            let mut context = Context::from_waker(&waker);
            match task.poll(&mut context) {
                Poll::Ready(()) => {} // task done
                Poll::Pending => self.task_queue.push_back(task),
            }
        }
    }
}

```

函数使用`while let`循环处理`task_queue`中的所有任务．对每个任务首先创建一个包装由`dummy_waker`返回的`Waker`实例的`Context`类型．然后用context唤醒`Task::poll`如果poll返回`Poll::Ready`，任务已完成继续下一个任务．如果仍然返回`Poll::Pending`我们把它加入队尾以便在随后的循环便利中再次Poll.

### Trying it

有了我们的`SimpleExecutor`类型，我们可以在`main.rs`试着运行由`example_task`返回的函数

```rust
// in src/main.rs

use blog_os::task::{Task, simple_executor::SimpleExecutor};

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    // […] initialization routines, including `init_heap`

    let mut executor = SimpleExecutor::new();
    executor.spawn(Task::new(example_task()));
    executor.run();

    // […] test_main, "it did not crash" message, hlt_loop
}


// Below is the example_task function again so that you don't have to scroll up

async fn async_number() -> u32 {
    42
}

async fn example_task() {
    let number = async_number().await;
    println!("async number: {}", number);
}
```

当运行这段代码时，我们希望看到“async number: 42”打印在屏幕上

![](./qemu-simple-executor.png)

我们总结一下这个例子中的步骤:

- 首先，创建一个带有空的`task_queue`的`SimpleExecutor`新实例.
- 接着，我们调用异步的`example_task`函数,返回一个future.　将这个future包装在`Task`类型中，Task将future移动到堆上并固定它，然后通过spwan方法加入执行器的`task_queue`．
- 然后调用run方法开始执行队列中的单个任务，包括
  - 弹出任务队列头部的任务
  - 创建未任务一个`RawWaker`，将它转换为一个`Waker`实例，然后用它创建Context实例
  - 使用我们光创建的Context,在task的future上调用`poll`
  - 因为`example_task`没有等待任何东西，在第一次poll调用后直接运行，就是在`async number: 42`这一行打印时．
  - 因为example_task直接返回Poll:Ready,，它没有被加入会队列末端．
- run方法在任务队列空后返回.`kernel_main`函数返回，＂it didn’t crash”信息被打印

