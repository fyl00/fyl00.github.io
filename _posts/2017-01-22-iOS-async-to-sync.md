---
layout:     post
title:      iOS 多线程之等待完成
date:       2017-01-22
summary:    写应用的时候，有时候会遇到等待多个异步操作完成之后再执行下一步的场景，本文讨论了几种实现方法。
categories: iOS,GCD
---

在写 [OneShare](http://slowwalker.me/lab/oneshare) 的时候，发送推特和微博都是使用的第三方库，它们都是使用的异步发送。这时遇到了需要获得这两个异步操作的返回结果，再进行下一步操作的场景。来用个具体的代码例子来说明下，假设现在有三个任务 A、B、C，其中 C 必须在 A、B 之后执行，A、B 无要求。

像下面这样直接用 GCD 异步执行的话，可以看出无法保证 C 在 A、B 执行完成之后再执行，C 任务会在 A 和 B 还未完成就执行，

```objective-c
dispatch_queue_t queue = dispatch_queue_create("me.slowwalker.defaultQueue", DISPATCH_QUEUE_CONCURRENT);

dispatch_async(queue, ^{
    NSLog(@"A START");
    [NSThread sleepForTimeInterval:3];
    NSLog(@"A FINISH");
});
dispatch_async(queue, ^{
    NSLog(@"B START");
    [NSThread sleepForTimeInterval:1];
    NSLog(@"B FINISH");
});
dispatch_async(queue, ^{
    NSLog(@"C -> AFTER A&B");
});

// 输出结果：
// 07:46:25.048 WaitAsyncDemo[85891:5296876] A START
// 07:46:25.048 WaitAsyncDemo[85891:5296859] B START
// 07:46:25.048 WaitAsyncDemo[85891:5296861] C -> AFTER A&B
// 07:46:26.116 WaitAsyncDemo[85891:5296859] B FINISH
// 07:46:28.113 WaitAsyncDemo[85891:5296876] A FINISH

```

查看官方的 [Concurrency Programming Guide](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008091-CH1-SW1) 文档会发现 GCD 提供了[三种队列相关操作](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW30），分别是 Dispatch groups，Dispatch semaphores 和 Dispatch sources。仔细研究下文档，会发现前两者都能用于解决前文提到的问题。

## Dispatch Groups

苹果对 dispatch groups 的描述就是： a way to monitor a set of block objects for completion，还给出了一个使用例子 [Waiting on Groups of Queued Tasks](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW25)，可以看出这个 API 的应用方法就是对应此文的场景。具体代码如下：

```objective-c
dispatch_queue_t queue = dispatch_queue_create("me.slowwalker.defaultQueue", DISPATCH_QUEUE_CONCURRENT);

dispatch_group_t group = dispatch_group_create();

// dispatch_group_async 方式
dispatch_group_async(group, queue, ^{
    NSLog(@"A START");
    [NSThread sleepForTimeInterval:3];
    NSLog(@"A FINISH");
});

dispatch_group_async(group, queue, ^{
    NSLog(@"B Start");
    [NSThread sleepForTimeInterval:1];
    NSLog(@"B Finish");
});

//  通知 main——queue 可以执行 C 任务了
dispatch_group_notify(group, dispatch_get_main_queue(), ^{
    NSLog(@"C - AFTER A&B(Into main queue)");
});
```

除了官方文档中 `dispatch_group_async` 这种调用方式，还有另外一种方式，用 `dispatch_group_enter` 表示添加任务，用 `dispatch_group_leave` 表示执行完成。这里要注意的是，一定要成双成对。

```objective-c
// dispatch_group_enter/dispatch_group_leave 配对方式
dispatch_group_enter(group);
dispatch_async(queue, ^{
    NSLog(@"A START");
    [NSThread sleepForTimeInterval:3];
    NSLog(@"A FINISH");
    dispatch_group_leave(group);
});

dispatch_group_enter(group);
dispatch_async(queue, ^{
    NSLog(@"B Start");
    [NSThread sleepForTimeInterval:1];
    NSLog(@"B Finish");
    dispatch_group_leave(group);
});
```

## Dispatch Semaphores

> Dispatch semaphores call down to the kernel only when the calling thread needs to be blocked because the semaphore is unavailable.

这是官方的定义，大意就是可以通过 semaphore 来调度线程。官方同时还给了个[使用介绍](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW24)：

> 1. When you create the semaphore (using the dispatch_semaphore_create function), you can specify a positive integer indicating the number of resources available.
> 2. In each task, call dispatch_semaphore_wait to wait on the semaphore.
> 3. When the wait call returns, acquire the resource and do your work.
> 4. When you are done with the resource, release it and signal the semaphore by calling the dispatch_semaphore_signalfunction.

大致就是在创建完 Semaphore 后，调用 dispatch_semaphore_wait 询问系统是否能执行内部代码，它会接收一个 Semaphore 信号和时间值，若信号的信号量为 0，则会阻塞当前线程，直到信号量大于 0 或者经过输入的时间值；若信号量大于 0，则会使信号量减1并返回，程序继续住下执行。运行完调用 dispatch_semaphore_signal 告诉 Semaphore 释放资源，信号量加 1。**如果用来解决此文中的存在于同一个队列中的问题，显得有点大材小用，更适合解决两个不同的队列之间调度问题**。

```objective-c
dispatch_queue_t queueA = dispatch_queue_create("me.slowwalker.semaphoreQueueA", DISPATCH_QUEUE_CONCURRENT);
dispatch_queue_t queueB = dispatch_queue_create("me.slowwalker.semaphoreQueueB", DISPATCH_QUEUE_CONCURRENT);
dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);

dispatch_async(queueA, ^{
    NSLog(@"QueueA START");
    [NSThread sleepForTimeInterval:3];
    NSLog(@"QueueA FINISH");
    dispatch_semaphore_signal(semaphore);
});
NSLog(@"semaphore");
dispatch_async(queueB, ^{
    NSLog(@"QueueB START");
    [NSThread sleepForTimeInterval:1];
    NSLog(@"QueueB FINISH");
    dispatch_semaphore_signal(semaphore);
});

dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
NSLog(@"C -> AFTER A&B");

// 运行结果：
// 2017-06-18 23:51:29.698 WaitAsyncDemo[4610:389664] QueueA START
// 2017-06-18 23:51:29.698 WaitAsyncDemo[4610:389946] QueueB START
// 2017-06-18 23:51:30.702 WaitAsyncDemo[4610:389946] QueueB FINISH
// 2017-06-18 23:51:32.702 WaitAsyncDemo[4610:389664] QueueA FINISH
// 2017-06-18 23:51:32.703 WaitAsyncDemo[4610:389626] C -> AFTER A&B
```

除了这个场景，还有一个典型的场景，就是多线程下，同时对一个文件或者可变数组的操作，也就是线程安全的问题。等仔细研究后，再来总结下。

---
本文代码地址：[Github](https://github.com/fyl00/iOSDemo/tree/master/WaitAsyncDemo)
