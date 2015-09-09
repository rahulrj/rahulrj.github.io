---
layout: post
title: "Diving in a pool of threads"
description: "Thread pool and its manager"
category: java
tags: [Threads, Thread Pool, Java]
subheading: Thread pool and its manager
permalink: java/pool-of-threads
---
This post is about Thread Pools, their use cases and how to handle them.
<br><br>

## Thread Pools
So as its clear from its name,its a pool of threads.But why do we need a pool of threads for executing a task or a set of tasks? Can't we just start a new ```Thread``` for whatever task i feel? And if there are multiple tasks, then we can use multiple threads right?
<br><br>
Yes we can obviously do that, but to complicate(to simplify) lives(things), we have thread pools.
There is a performance overhead associated with starting a new thread, and each thread is also allocated some memory for its stack etc. So when we have a large number of tasks to execute and we want to manage the above stated issues , then Thread Pool is the best choice. It will allow to run multiple tasks in parallel and we can manage also to wait the execution of task until a thread from our pooled collection becomes available.
<br><br>
So for a formal definition,a thread pool is a group of pre-instantiated threads that are ready for work. They are more suitable for executing large number of small tasks. This prevents the need of creating a thread large number of times. To create and manage these pools, we have its manager, ```ThreadPoolExecutor```.

## ThreadPoolExecutor
A Thread Pool is created and managed by functions provided by ```ThreadPoolExecutor```. Before coming to ```ThreadPoolExecutor```, there are two more components on which it is based upon.

### Executor
Its nothing but just an interface which provides a method ```execute(Runnable r)``` in which you can write your own
implementations to execute a task.
{% highlight java %}
public interface Executor {
    void execute(Runnable command);
}
{% endhighlight %}
It just provides a way of abstracting the task running mechanism. Users just need to call ```Executor.execute(r)```   and they are done. The `execute()` will handle all the implementation details.

### ExecutorService
```ExecutorService``` is just an ```Executor``` with some extra functionalities.
{% highlight java %}
public interface ExecutorService extends Executor {

  // methods to control the executor
}
{% endhighlight %}
These extra functionalities provides methods like to shut down an ```Executor```( so that it will reject the new incoming tasks),to return a ```Future``` so that we can keep progress on the submitted tasks etc.
<br><br>
Now lets make a Thread Pool and see what can we do with them and also how those things are done internally.
While there are factory methods provided by `Executors` class to create thread pool, we will take the hard path of
calling its constructor. I would omit the constructor syntax here but but lets see what it does. The constructor initializes some pretty obvious things.So what do you think would be required in this process?

-> **A container for holding the tasks** -  Suppose you have a pool of 5 threads and all of them are busy and you give          them a 6th task to execute.So for holding those tasks, a queue is used.In this case a `LinkedBlockingQueue` is used.The head of the queue is that element that has been in the queue the longest time. The tail of the queue is that element that has been on the queue the shortest time.
<br><br>
-> **Now where will the threads come from** - Here threads will come from a factory.Literally its called a `ThreadFactory`. Its an interface which is used to create new threads **on-demand**
{% highlight java %}
public interface ThreadFactory {

    Thread newThread(Runnable r);
}
{% endhighlight %}
So here the implementations can define things like priority,status etc for the newly created thread. Java uses ```DefaultThreadFactory``` here which creates non-daemon threads with normal priority.
<br><br>
-> **What is the size of this pool** - Its defined by two parameters that has to be supplied in the constructor.
```corePoolSize``` is  the no of threads that will be there in the pool even even they are lying idle(this is also configurable. No need to leave the threads idle). `maxPoolSizeMax` is the max  number of threads to allow in the pool.
When a new task is submitted in method ```execute(Runnable)```, and fewer than `corePoolSize` threads are running, a new thread is  created to handle the request, even if other worker threads are idle.  If there are more than `corePoolSize` but less than `maximumPoolSize` threads running, a new thread will be created only if the queue is full.

The number of non-core idle threads in the pool can also be reduced through `keepAliveTimeUnit` (supplied in the constructor ) and the same for core idle threads can be done by `allowCoreThreadTimeout(boolean b)`.
<br><br>
-> **Something for handling the rejected tasks** - When tasks are supplied even after the ```ThreadPoolExecutor``` is shut down or when the queue is full, they are rejected. For handling the rejected tasks, we have ```RejectedExecutionHandler```. Its not an Android ```Handler``` but just a literal handler for handling the tasks.Its just an interface with a method
{% highlight java %}
void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
{% endhighlight %}
The default `RejectedExecutionHandler` used here just throws an exception when any task is rejected. There are other REHs available for use like `DiscardPolicy` and `DiscardOldestPolicy`. Of course we can also create our own and supply it in ```setRejectedExecutionHandler (RejectedExecutionHandler)```.

## How the Execution happens
Internally, there is no magic happening but the same basic operations that you can think of like polling the queue, creating a thread when core threads are insufficient and interrupting the thread when `shutdown()` is called.
I will touch on these internal things a bit.

### Bit packing

A `ThreadPoolExecutor` has 5 states internally called `runState` i.e RUNNING, SHUTDOWN, STOP, TIDYING and TERMINATED and i will describe how it reaches these states. Also it has a property `workerCount`(effective no. of threads). These two variables are packed into a single Integer. This is one of the interesting things here. Packing multiple variables into a single one saves us a lot of memory and gives us an easy way of setting and getting these states. For example, 8 booleans can be packed into a single `unsigned char` of 8 bytes by setting 1 bit corresponding to each boolean. In `ThreadPoolExecutor`, 3 higher order bits are kept for states while the remaining 29 bits are for representing the number of worker thread count. The thread control state variable `ctl` is
{% highlight java %}
ctl=runState | workerCount ;
where runState= -1,0,1,2,3 >> 29 for respective states.
{% endhighlight %}
Now since `workerCount` is stored in the lower 29 bits, if you want to retrieve the value of `workerCount` at any time, just do a bitwise AND of `ctl` with 00011111...1 (32 bits).

### Workers
Now one could have simply created threads when needed and store them somewhere in a HashMap if they are lying idle or executing. And if we want to shutdown the executor, then interrupt all the threads in the HashMap. But this approach has some limitations. What if we want to avoid firing `Thread.interrupt()` for threads which have not even been started? What if we want to wake up the idle threads using our interrupts rather than stopping the execution of running threads?
<br><br>
For solving these problems, we need to maintain a **state** of each `Thread` and we also need an efficient locking mechanism while executing these threads. So we have a `Worker` class which extends from `AbstractQueuedSynchronizer`. Now `AbstractQueuedSynchronizer` serves here as a base for implementing a non-reentrant locking system and also it helps in maintaining the state of each thread.

 Each `Worker` object maintains its own `Thread`, which is created by `ThreadFactory`. So basically if a new `Thread` is needed anywhere means we have to create a new `Worker` object. As stated above, it is needed to maintain the interrupt states of its `Thread`. A thread here has three states

* -1 -> the thread has not been started.
*  0 -> the thread is lying idle
*  1 -> the thread is executing some task

These state are pretty useful for usecases in which an action is needed to be performed only on certain threads. We will see how they help in both execution and shutdown of the executor.

### Executing a Runnable
Now with all the above concepts in hand, its not hard to think what going on when we fire `execute(Runnable r)`. Its very clear from the Java's source code also but i will write it in short.

1. Check if no. of `workerCount`<`corePoolSize`.If yes, then add a `worker`, start its thread with the runnable given and put  the worker in a `HashSet`.
1. If its unable to add a `worker`, then try putting it the `workQueue`.
1. If putting it in the queue also fails, then again try adding a `worker`( with pool limit as `maxPoolSizeMax`).If that also fails, then reject the runnable.

### shutDown() and shutDownNow()
In `shutDown()` only the idle workers are interrupted( means ongoing tasks can be finished) while in `shutDownNow()`, all the workers are interrupted and tasks are removed from the queue instantaneously. How the idle workers are identified? Using the state as '0'.
