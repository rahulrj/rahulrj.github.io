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
It just provides a way of abstracting the task running mechanism. Users just need to call ```Executor.execute()```   and they are done.

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
