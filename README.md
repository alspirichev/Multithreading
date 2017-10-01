# Multithreading

Apple’s mobile and desktop operating systems provide the same APIs for concurrent programming. We are going to take a look at **pthread** and **NSThread**, **Grand Central Dispatch**, **NSOperationQueue**, and **NSRunLoop**.

Technically, *run loops* are the odd ones out in this list, because they don’t enable true **parallelism**. But they are related closely enough to the topic that it’s worth having a closer look.

- [Terminology](#terminology)
- [Threads](#threads)

Queue-based concurrency APIs:
- [Grand Central Dispatch](#grand-central-dispatch)
- [Operation Queues](#operation-queues)
- [GCD vs. Operation Queues](#gcd-vs-operation-queues)

- [Run Loop](#run-loop)
- [Challenges of Concurrent Programming](#challenges-of-concurrent-programming)

# Terminology

- Task

  Task a simple, single piece of work that needs to be done.

- Thread

  Thread a mechanism provided by the operating system that allows multiple sets of instructions to operate at the same time within a single application.

- Process

  Process an executable chunk of code, which can be made up of multiple threads.

- Serial vs. Concurrent

  These terms describe **when** tasks are executed with respect to each other. Tasks executed **serially** are always executed one at a time. Tasks executed **concurrently** might be executed at the same time.

- Synchronous vs. Asynchronous

  Within **GCD**, these terms describe **when a function completes** with respect to another task that the function asks **GCD** to perform. A **synchronous** function returns **only after the completion** of a task that it orders.

  An **asynchronous** function, on the other hand, returns **immediately**, ordering the task to be done but does not wait for it. Thus, an asynchronous function does not block the current thread of execution from proceeding on to the next function.

# Threads

 **Threads** are subunits of processes, which can be scheduled independently by the operating system scheduler. Virtually all concurrency APIs are built on top of threads under the hood – that’s true for both **Grand Central Dispatch** and **operation queues**.

The important thing to keep in mind is that you have no control over where and when your code gets scheduled, and when and for how long its execution will be paused in order for other tasks to take their turn. This kind of thread scheduling is a very powerful technique. However, it also comes with great complexity.

Leaving this complexity aside for a moment, you can either use the **POSIX** thread API, or the Objective-C *wrapper* around this API, **NSThread**, to create your own threads.

**NSThread** is a simple Objective-C wrapper around pthreads. This makes the code look more familiar in a Cocoa environment. For example, you can define a thread as a subclass of **NSThread**, which encapsulates the code you want to run in the background.

```objective-c
NSThread* myThread = [[NSThread alloc] initWithTarget:self
                                        selector:@selector(myThreadMainMethod:)
                                        object:nil];
[myThread start];
```
# Grand Central Dispatch

**GCD** or **Grand Central Dispatch** is a modern feature of Objective-C that provides a rich set of methods and API's to use in order to support common multi-threading tasks. **GCD** provides a way to queue tasks for dispatch on either the main thread, a **concurrent** queue (tasks are run in parallel) or a **serial** queue (tasks are run in FIFO order).

GCD exposes five different queues:

* *main* queue (running on the main thread)

* three **background** queues with different *priorities*

* one **background** queue with an even lower priority (which is I/O throttled)

Furthermore, you can create **custom** queues, which can either be serial or concurrent queues.

![image](threads.png)

Making use of *several* queues with different priorities sounds pretty straightforward at first. However, we strongly recommend that you use the **default** priority queue in almost all cases. Scheduling tasks on queues with different priorities can quickly result in **unexpected behavior** if these tasks access shared resources.

```objective-c
dispatch_queue_t myQueue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_async(myQueue, ^{
   /* Do something. */
});
```
# Operation Queues

**Operation queues** are a Cocoa abstraction of the queue model exposed by **GCD**. While **GCD** offers more *low-level* control, **operation queues** implement several convenient features on *top of it*, which often makes it the best and safest choice for application developers.

The NSOperationQueue class has **two** different types of queues:

* **main** queue

* custom queues

The **main queue** runs on the **main thread**, and **custom queues** are processed in the **background**. In any case, the tasks which are processed by these queues are represented as subclasses of **NSOperation**.

```objective-c
NSOperationQueue *myQueue = [[NSOperationQueue alloc] init];
YourOperation *operation = [[YourOperation alloc] init];
[queue  addOperation:operation];
[myQueue addOperationWithBlock:^{
   /* Do something. */
}];
```

By the very nature of abstractions, **operation queues** come with a small **performance** hit compared to using the **GCD** API. However, in almost all cases, this impact is *negligible* and operation queues are the tool of choice.

# GCD vs. Operation Queues

### Advantages and disadvantages:

* Operation Queues:

	* Dependencies

	This is a powerful concept that enables developers to execute tasks in a specific order. An operation is ready when every dependency has finished executing.

	* Observable

	The **NSOperation** and **NSOperationQueue** classes have a number of properties that can be observed, using **KVO**. This is another important benefit if you want to *monitor the state* of an operation or operation queue.

	* Pause, Cancel, Resume

	Operations can be **paused, resumed,** and **cancelled**. Once you dispatch a task using **Grand Central Dispatch**, you no longer have control or insight into the execution of that task. The **NSOperation** API is more flexible in that respect, giving the developer control over the operation’s life cycle.

	* Control

	The **NSOperationQueue** also adds a number of benefits to the mix. For example, you can specify the *maximum number of queued operations* that can run simultaneously. This makes it easy to control how many operations run at the same time or to create a serial operation queue.

* Grand Central Dispatch:

	* GCD can potentially optimize your code with higher performance primitives for common patterns such as singletons

	* GCD is ideal if you just need to dispatch a block of code to a *serial or concurrent* queue. If you don’t want to go through the hassle of creating an **NSOperation** subclass for a trivial task, then **Grand Central Dispatch** is a great alternative.

	* Another benefit of **Grand Central Dispatch** is that you can keep related code together.

Dispatch **queues, groups, semaphores, sources**, and **barriers** comprise an essential set of *concurrency primitives*, on top of which all of the system frameworks are built.

For one-off computation, or simply speeding up an existing method, it will often be more convenient to use a lightweight **GCD** dispatch than employ **NSOperation**.

> <h2> Always use the highest-level abstraction available to you, and drop down to lower-level abstractions when measurement shows that they are needed.

# Run Loop

**Run loops** are part of the fundamental infrastructure associated with threads. A **run loop** is an event processing loop that you use to schedule work and coordinate the receipt of incoming events. The **purpose of a run loop** is to keep your thread busy when there is work to do and put your thread to sleep when there is none.

*Run loop management is not entirely automatic*. You **must** still design your thread’s code to **start** the run loop at appropriate times and respond to incoming events. Both Cocoa and Core Foundation provide run loop objects to help you configure and manage your thread’s run loop. Your application **does not need to create** these objects explicitly; **each thread, including the application’s main thread, has an associated run loop object**. Only **secondary threads** need to run their run loop explicitly, however. The app frameworks **automatically** set up and run the run loop on the **main thread** as part of the application startup process.

It is a loop your thread enters and uses to run **event** handlers in response to incoming events. Your code *provides* the control statements used to implement the actual loop portion of the run loop—in other words, your code provides the **while** or **for** loop that drives the **run loop**. Within your loop, you use a run loop object to "**run**” the event-processing code that receives events and calls the installed handlers.

A run loop receives events from two different types of sources.

* *Input sources* deliver **asynchronous** events, usually messages from **another thread** or from a **different application**.

* **Timer sources** deliver **synchronous** events, occurring at a **scheduled time** or **repeating interval**.

Both types of source use an application-specific handler routine to process the event when it arrives.

![image](runLoop.png)

In addition to handling sources of input, **run loops** also generate *notifications* about the run loop’s behavior. **Registered** run-loop observers can receive these notifications and use them to do additional processing on the thread. You use **Core Foundation** to install run-loop observers on your threads.

# When Would You Use a Run Loop?

The only time you need to run a **run loop** explicitly is when you create **secondary threads** for your application. The run loop for your application’s **main thread** is a crucial piece of infrastructure.

For **secondary threads**, you need to decide whether a **run loop** is necessary, and if it is, *configure and start* it yourself. You **do not need** to start a thread’s run loop in all cases. For example, if you use a thread to perform some **long-running** and **predetermined** task, you can probably **avoid** starting the run loop. Run loops are intended for situations where you want **more interactivity with the thread**.

For example, you need to start a **run loop** if you plan to do any of the following:

* Use **ports** or custom *input sources* to communicate with **other** threads.

* Use **timers** on the thread.

* Use any of the *performSelector…* methods in a Cocoa application.

* Keep the thread around to perform **periodic** tasks

# Challenges of Concurrent Programming

### Race Condition

This is a situation where the behavior of a software system depends on a specific sequence or timing of events that execute in an uncontrolled manner, such as the exact order of execution of the program’s concurrent tasks. **Race conditions** can produce unpredictable behavior that aren’t immediately evident through code inspection.

For example, thread **A** and thread **B** both **read** the value of the counter from memory; let’s say it is *17*. Then thread **A** increments the counter by one and writes the resulting *18* back to memory. At the same time, thread **B** also increments the counter by one and writes a *18* back to memory, just **after** thread **A**. At this point the data has become **corrupted**, because the counter holds an *18* after it was **incremented twice** from a *17*.

![image](race_condition.png)

### Mutual Exclusion

**Mutual exclusive** access means that only **one** thread at a time gets access to a certain *resource*. In order to ensure this, each thread that wants to access a resource first needs to acquire a **mutex lock** on it. Once it has finished its operation, it releases the lock, so that other threads get a chance to access it.

![image](Mutual Exclusion)

Of course the implementation of a **mutex lock** in itself needs to be **race-condition free**. 

### Dead Locks

**Mutex locks** solve the problem of **race conditions**, but unfortunately they also introduce a new problem at the same time: **dead locks**. A **dead lock** occurs when multiple threads are waiting on each other to finish and get stuck.

![image](Dead Locks)

### Starvation

Locking shared *resources* can result in the **readers-writers problem**. In many cases, it would be wasteful to **restrict** reading access to a resource to one access at a time. Therefore, taking a **reading lock** is allowed as long as there is **no writing lock** on the resource. In this situation, a thread that is waiting to acquire a write lock can be **starved** by more read locks occurring in the meantime.

In order to solve this issue, more clever solutions than a simple read/write lock are necessary, e.g. giving **writers preference** or using the **read-copy-update algorithm**.

### Priority Inversion

**Priority inversion** describes a condition where a *lower priority* task **blocks** a *higher priority* task from executing, effectively **inverting** task priorities.

![image](Priority Inversion)

The problem can occur when you have a *high-priority* and a *low-priority* task share a **common resource**. When the *low-priority* task takes a lock to the common resource, it is supposed to finish off quickly in order to release its lock and to let the *high-priority* task execute without significant delays. Since the *high-priority* task is blocked from running as long as the *low-priority* task has the lock, there is a window of opportunity for *medium-priority* tasks to run and to preempt the *low-priority* task, because the *medium-priority* tasks have now the highest priority of all currently runnable tasks. At this moment, the *medium-priority* tasks hinder the *low-priority* task from releasing its lock, therefore effectively gaining priority over the still waiting, *high-priority* tasks.

In general, **don’t use different priorities**. Often you will end up with **high-priority** code waiting on **low-priority** code to finish. When you’re using **GCD**, always use the **default priority** queue (directly, or as a target queue). If you’re using different priorities, more likely than not, it’s actually going to make things worse.

### Thread Safe

**Thread safe** code can be safely called from multiple threads or concurrent tasks without causing any problems (data corruption, crashing, etc). Code that is not thread safe must only be run in one context at a time. An example of thread safe code is **NSDictionary**. You can use it from multiple threads at the same time **without issue**. On the other hand, **NSMutableDictionary** is **not thread safe** and should only be accessed from one thread at a time.

# Sources

- [objc.io](https://www.objc.io/issues/2-concurrency/concurrency-apis-and-pitfalls)
- [raywenderlich.com](https://www.raywenderlich.com/60749/grand-central-dispatch-in-depth-part-1)
- [codementor.io](https://www.codementor.io/mattgoldspink/ios-interview-tips-questions-answers-objective-c-du1088nfb)
- [raywenderlich.com](https://www.raywenderlich.com/76341/use-nsoperation-nsoperationqueue-swift)
- [cocoacasts.com](https://cocoacasts.com/choosing-between-nsoperation-and-grand-central-dispatch)
- [developer.apple.com](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html)
- [NSHipster.com](http://nshipster.com/nsoperation/)








