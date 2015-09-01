---
layout: post
title: "Digging into Android Asynctask"
description: "It was the first time i looked into the source code of an Android component deeply.I wrote it
long back before this blog was made.Found it very interesting and now digging into the source code of every
other thing has become a habit."
category: android
tags: [Threads, AsyncTask, Android]
subheading: Implementation of our mighty AsyncTask
permalink: android/Android-asynctask
---

This post deals with the internal implementation of AsyncTask.. I was just curious to find out what happens when we run an ```AsyncTask```. Is is just a background thread for ```doInBackground()```? I have found out now, so i decided to publish it so that it can help others.

## My Implementation
SO the best i could think of AsyncTask is that it does the following when i call `doInBackground(params).`
{% highlight java %}
Thread thread = new Thread() {
    @Override
    public void run() {

    }
};
thread.start();
{% endhighlight %}

I had no idea how the different ```Threads``` are handled in this case or how do i return the result from ```get()``` method? So i dived-deep and here are the results.

## AsyncTask Components
The important components that constitute an AsyncTask which will help to understand its functioning better is

### Callable  

This component is almost like ```Runnable``` in the sense that both are designed for classes whose instances are to be executed on other thread.  But the difference between them is that it can also return a result and it can throw checked exceptions while a ```Runnable``` can't do these. While implementing it yourself, you would have to override its ```call()``` method to execute the task and make it return a value.  

In AsyncTask, we wrap our main task in a ```Callable``` and pass that to a ```FutureTask```(explained below). The ```Callable``` that is used here is ```WorkerRunnable<Params,Result>mWorker```. ```Result``` is passed as parameter to the ```Callable``` because it returns a value of type ```Result``` from its ```call()``` method. We will discuss later what happens in this callable.
{% highlight java %}
 class WorkerRunnable<Params, Result> implements Callable<Result> {
        Params[] mParams;
{% endhighlight %}

### FutureTask
* It is a component that helps in writing asynchonous code.
* A given task can be wrapped in a FutureTask and the it can be run asynchronously on an ```Executor``` using a background thread or a pool of threads and a FutureTask promises to return some result in the future.
* It provides methods to start and cancel a computation, query to check if the computation is complete and also return the value of computation. Thats why we are able to get the result from ```AsyncTask``` using ```get()``` method and also cancel the task using ```cancel()``` method.

In AsyncTask, a ```FutureTask``` is created, which upon running will execute the above ```Callable.```This ```FutureTask``` is executed on a different thread.And we will specify what should be done when ```FutureTask``` finishes its operation in ```done()``` method.
{% highlight java %}
  mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {

            }
  };
{% endhighlight %}

### Executor
This component executes the submitted tasks.An ```Executor``` can be used for running the ```Runnable``` objects without having to explicitly write ```new Thread(new(RunnableTask())).start()``` for each task and also mostly reusing the already created threads. We can use the syntax ```executor.execute(new RunnableTask1())``` for running the tasks.

Its an interface with only one method ```execute(Runnable)```. The task submitted can execute in the same thread, in a different thread or a pool of threads, depending on the implementation of the ```execute``` method.

The ```Executor``` used here is ```SerialExecutor```. Its nothing but an implementation of ```Executor```, that executes tasks one at a time in serial order. It maintains an ```ArrayDeque``` in which tasks are enqueued and are extracted for execution.

### Thread Pools

Thread Pools are used for executing the task with a ```ThreadPoolExecutor```. I have given a description and implementation of Thread Pools [here](https://github.com/rahulrj/Deep-Dive/wiki/Thread-Pools).

### ThreadPoolExecutor
It is an ExecutorService that executes each submitted task using one of possibly several pooled threads. A description is given [here](https://github.com/rahulrj/Deep-Dive/wiki/ThreadPoolExecutor).Its instance ```THREAD_POOL_EXECUTOR``` is used in ```AsyncTask```.We will see how this is used in AsyncTask.


## AsyncTask Working
Those were the key components that will help in getting how ```AsyncTask``` works. Now lets see what happens in each step  

### What happens when we initialize an AsyncTask?  

It initializes the following things  

- WorkerRunnable( mWorker)- It is actually a ```Callable``` inside which ```doInBackground()``` is executed and returns the Result to ```onPostExecute()```. The thread is given a ```BACKGROUND_PRIORITY``` here which is less than the priority of the UI thread. Threads with ```BACKGROUND_PRIORITY``` are assigned only a small fraction of the CPU if other foreground threads are busy, so that it wont hurt the foreground performance. BTW, ```mWorker``` is only initialized here and not yet executed.  

- FutureTask(mFuture)- ```mFuture``` is the ```FutureTask``` here which accepts the above ```Callable```. It is also just initialized here with the ```Callable``` and not executed here.

### When we execute() an AsyncTask?
The ```execute()``` method is declared in ```AsyncTask``` as
{% highlight java %}
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
        return executeOnExecutor(sDefaultExecutor, params);
    }
{% endhighlight %}
It returns itself and calls the ```Executor```. Here ```sDefaultExecutor``` is a ```SerialExecutor```. Now in ```executeOnExecutor()```,status of the task is checked and is set to ```RUNNING``` if the task is not already running and it throws exception if you try to run the same task again before it has finished. Also ```onPreExecute()``` is called from here, which calls ```onPreExecute()``` of our instance of ```AsyncTask.```

Now, the ```params``` that are passed with ```AsyncTask``` are set in the ```Callable(mWorker)```. And then the most important part happens i.e the ```Executor(sDefaultExecutor)``` executes the ```FutureTask(mFuture)```. Note that the ```SerialExecutor``` used here is static and hence the same one will be used across all ```AsyncTask``` instances.

```SerialExecutor``` maintains an ```ArrayDeque``` of ```Runnable(mTasks)``` . So when we say ```executor.execute()```,it inserts the ```FutureTask(mFuture)```at the end of the queue(mTasks). That task is extracted from the queue and it is executed by  ```ThreadPoolExecutor(THREAD_POOL_EXECUTOR)```.

The ```ThreadPoolExecutor``` is used to execute tasks in parallel using its pool of threads.Its ```CORE_POOL_SIZE``` is the number of available processor cores in the phone+1 and ```MAX_POOL_SIZE``` is twice the no. of available processor cores.Here is the structure of the TPE used in ```AsyncTask```.
{% highlight java %}
public static final Executor THREAD_POOL_EXECUTOR
            = new ThreadPoolExecutor(CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE,
                    TimeUnit.SECONDS, sPoolWorkQueue, sThreadFactory);
{% endhighlight %}
```sThreadFactory``` is literally a Thread Factory which is used to generate ```Thread``` for the Thread Pools. ```ThreadFactory``` is actually an interface which you can implement to give a custom implementation of the available function ```Thread newThread(Runnable r)```. Its used to remove the hardwiring of calls to ```new Thread(Runnable)``` and we can also keep track of a thread  statistics like its count using our own implementation. The following is the implementation of ```ThreadFactory``` in ```AsyncTask.```
{% highlight java %}
 private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };
{% endhighlight %}
```sPoolWorkQueue``` is a ```BlockingQueue```. A ```BlockingQueue``` supports operations that wait for the queue to become non-empty when retrieving an element, and wait for space to become available in the queue when storing an element.Storing and retrieving of elements is done according to the conditions mentioned in [TPE](https://github.com/rahulrj/Deep-Dive/wiki/ThreadPoolExecutor) page.

Now coming back to the process, the ```THREAD_POOL_EXECUTOR``` executes the ```FutureTask(mFuture)```, which in turn executes the task of ```Callable(mWorker)```. As mentioned earlier, ```doInBackground()``` is executed from this ```Callable``` and its result is wrapped into a class ```AsyncResult``` and is sent to the ```Handler(sHandler)``` with the key ```MESSAGE_POST_RESULT.``` The ```Handler``` decodes the result using the passed key and it sends the result to ```onPostExecute()``` from there.

For passing the Progress values to the UI thread,again it is sent by ```Handler(sHandler)``` to the UI thread using the key ```MESSAGE_POST_PROGRESS```. The Handler then calls ```onProgressUpdate(Progress... values)``` on the main thread itself.

```Handler``` is used here because we are passing the data from worker thread to the UI thread.

For ```AsyncTask``` we have also a function ```get()``` which returns the result calculated from ```doInBackground()```. This internally calls ```mFuture.get()``` which gives the result computed by the ```FutureTask```. It is a blocking call and it will wait if it is called and the result has not been calculated.

Finally, there is ```cancel(boolean mayInterruptIfRunning)``` function available in ```AsyncTask```. Calling this will call ```mFuture.cancel()``` and a flag is set.Quoting from the docs itself, if the task has not been started yet and a call to ```cancel()``` is made,the task will never start. If the task is already running, then the parameter ```mayInterruptIfRunning``` determines whether the thread executing this task should be interrupted or not. If the task is cancelled, then ```onPostExecute()``` will never be called. Details of how a running thread can stops its execution can be seen [here](https://10kloc.wordpress.com/2013/12/24/cancelling-tasks-in-executors/)
