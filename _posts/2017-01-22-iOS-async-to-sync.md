---
layout:     post
title:      iOS 中的线程同步
date:       2017-01-22
summary:    写应用的时候，有时候会遇到等待多个异步操作完成之后再执行下一步的场景，本文讨论了几种实现方法。
categories: iOS,GCD
---

在写 [OneShare](http://slowwalker.me/lab/oneshare) 的时候，发送推特和微博都是使用的第三方库，它们都是使用的异步发送。这时遇到了需要获得这两个异步操作的返回结果，再进行下一步操作的场景。通过搜索，发现 GCD 中有三种方法可以解决此类线程同步的问题，分别是利用 Group，Barrier 和 Semaphores 来实现。

假设现在有三个任务 A、B、C，其中 C 必须在 A、B 之后执行，A、B 无要求。像下面这样直接用 GCD 异步执行的话，可以看出无法保证 C 在 A、B 执行完成之后再执行。

```objective-c
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
```

这段代码运行时，C 任务会在 A 和 B 还未完成就执行，输出结果如下：

```
07:46:25.048 WaitAsyncDemo[85891:5296876] A START
07:46:25.048 WaitAsyncDemo[85891:5296859] B START
07:46:25.048 WaitAsyncDemo[85891:5296861] C -> AFTER A&B
07:46:26.116 WaitAsyncDemo[85891:5296859] B FINISH
07:46:28.113 WaitAsyncDemo[85891:5296876] A FINISH
```

## Dispatch Groups

苹果对 dispatch groups 的描述就是： a way to monitor a set of block objects for completion，还给出了一个使用例子 [Waiting on Groups of Queued Tasks](https://developer.apple.com/library/content/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationQueues/OperationQueues.html#//apple_ref/doc/uid/TP40008091-CH102-SW25)，可以看出这个 API 的应用方法就是对应此文的场景。不过除了官方文档中 `dispatch_group_async` 这种调用方式，还有 `dispatch_group_enter` 和 `dispatch_group_leave` 配对的方式，enter 表示添加任务，leave 表示执行完成。

```objective-c
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

// dispatch_group_enter/dispatch_group_leave 配对方式
/**  
    *   dispatch_group_enter(group);
    *   dispatch_async(queue, ^{
    *        NSLog(@"A START");
    *        [NSThread sleepForTimeInterval:3];
    *        NSLog(@"A FINISH");
    *        dispatch_group_leave(group);
    *    });
    *    
    *    dispatch_group_enter(group);
    *    dispatch_async(queue, ^{
    *        NSLog(@"B Start");
    *        [NSThread sleepForTimeInterval:1];
    *        NSLog(@"B Finish");
    *        dispatch_group_leave(group);
    *    });
    *／

//  通知 main——queue 可以执行 C 任务了
dispatch_group_notify(group, dispatch_get_main_queue(), ^{
    NSLog(@"C - AFTER A&B(Into main queue)");
});
```

## Dispatch Semaphores

> Dispatch semaphores call down to the kernel only when the calling thread needs to be blocked because the semaphore is unavailable.

这是官方的定义，没有特别明白，直接看代码和运行结果比较明白。

```objective-c
dispatch_semaphore_t semaphore = dispatch_semaphore_create(1);

dispatch_async(queue, ^{
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    NSLog(@"A START");
    [NSThread sleepForTimeInterval:3];
    NSLog(@"A FINISH");
    dispatch_semaphore_signal(semaphore);
});

dispatch_async(queue, ^{
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    NSLog(@"B START");
    [NSThread sleepForTimeInterval:1];
    NSLog(@"B FINISH");
    dispatch_semaphore_signal(semaphore);
});

dispatch_async(queue, ^{
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);
    NSLog(@"C -> AFTER A&B");
    dispatch_semaphore_signal(semaphore);
});
```

运行结果是 A、B、C 依次执行，比可以看出 `dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER);` 是表示异步队列中此后的任务，需要等此任务通过 `dispatch_semaphore_signal(semaphore)` 告知队列此任务已完成之后，才能开始其他任务。

## Dispatch Barrier

`dispatch_barrier_async` 就像一道分水岭，分水岭内部的代码需要等之前的代码执行完之后才开始执行，之后的代码要等它执行完之后才开始执行。过了分水岭之后的异步代码又开始依据系统本身地设定来执行，而不需要等其他任务完成之后才执行。这段话比较绕，直接看代码和输出就能明白了。

```objective-c
dispatch_async(queue, ^{
    NSLog(@"A START");
    [NSThread sleepForTimeInterval:3];
    NSLog(@"A FINISH");
});
dispatch_barrier_async(queue, ^{
    NSLog(@"B START");
    [NSThread sleepForTimeInterval:2];
    NSLog(@"B FINISH");
});
dispatch_async(queue, ^{
    [NSThread sleepForTimeInterval:5];
    NSLog(@"C -> AFTER A&B");
});
dispatch_async(queue, ^{
    [NSThread sleepForTimeInterval:1];
    NSLog(@"D -> AFTER A&B");
});
```

其输出结果为:

```shell
10:48:25.960 WaitAsyncDemo[88816:5379642] A START
10:48:29.034 WaitAsyncDemo[88816:5379642] A FINISH
10:48:29.035 WaitAsyncDemo[88816:5379642] B START
10:48:31.102 WaitAsyncDemo[88816:5379642] B FINISH
10:48:32.177 WaitAsyncDemo[88816:5379644] D -> AFTER A&B
10:48:36.174 WaitAsyncDemo[88816:5379642] C -> AFTER A&B
```

---
完整代码地址：[Github](https://github.com/fyl00/iOSDemo/tree/master/WaitAsyncDemo)
