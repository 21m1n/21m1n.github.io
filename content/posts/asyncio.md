+++
title = 'Asyncio in Python'
date = 2025-04-06T07:19:42+08:00
draft = false
tags = ['python', 'concurrency']
+++

With the advent of large language models (LLMs), asynchronous programming has become a critical skill in AI application development. When working with these computationally intensive models, developers face two key challenges: LLMs are typically served via network endpoints, introducing significant latency that can severely impact application responsiveness. This is where asynchronous programming shine. 

Unlike traditional synchronous programming, where operations execute sequentially and block until completion, asynchronous programming allows applications to continue processing CPU-bound operations while waiting for I/O-bound responses in the background. This approach not only maximuizes computational resource utilization but also dramatically improves user experiences by keep applications responsive, even when interacting with resource-intensive services like LLMs.

For developers working with LLM-powered applications, mastering asynchronous programming patterns has become essential for building scalable, responsive systems. In this article, we will explore Python's asyncIO framework: how it evolved, its core components, practical implementations, and why it's particularly valuable for AI application development.

## Understanding Concurrency Paradigms 

Before diving into asyncIO, let's clarify the fundamental concepts that underpin different approaches to concurrent programming. These distinctions are crucial for choosing the right approach for your specific use case.

**Parallelism vs. Concurrency**

- **Parallelism** refers to executing multiple computations simultaneously,typically across multiple CPU cores. It's like _having multiple chefs working independently in different kitchen stations_. 
- **Concurrency** refers to managing multiple tasks with overlapping time periods. It doesn't necessarily mean tasks execute at the exact same time. This is like _a single chef rapidly switching between multiple dishes on the stove, ensuring none burns while others are cooking_.

**Multiprocessing vs. Multithreading vs. Asynchronous programming** 

- **Multiprocessing** involves spawning multiple python processors, each process operates independently with its own python interpreter and memory space. This approach achieves true parallelism but requires more system resources and introduces communication overhead between processes.
- **Multithreading** allows multiple threads within a single process to run concurrently. These threads share the same memory space, enabling efficient communication but may lead to race conditions or deadlocks if not handled properly. In Python, the Global Interpreter Lock (GIL) limits the benefits of multithreading for CPU-bound tasks.
- **Asynchronous programming** uses a single-threaded, cooperative multitasking approach where tasks voluntarily yield control when waiting for I/O operations. This approach excels at handling I/O-bound operations like network requests or database queries without the overhead of multiple threads or processes. 

## Synchronous vs. Asynchronous 

To understand why asynchronous programming is so valuable for LLM applications, let's compare how synchronous and asynchronous approaches handle multiple operations. The difference becomes particularly significant when dealing with I/O-bound tasks like API calls to LLM services:
  
```python
# simple benchmark comparing sync vs async approaches
import asyncio
import time

# synchronous approach
def sync_fetch(url):
    # simulate network delay
    time.sleep(1)  
    return f"Data from {url}"

def sync_main():
    start = time.time()
    results = []
    # process URLs sequentially
    for i in range(5):
        results.append(sync_fetch(f"url_{i}"))
    end = time.time()
    print(f"Sync approach took {end-start:.2f} seconds")
    return results

# asynchronous approach
async def async_fetch(url):
    # Simulate network delay without blocking
    await asyncio.sleep(1)  
    return f"Data from {url}"

async def async_main():
    start = time.time()
    # initiate all requests concurrently
    tasks = [async_fetch(f"url_{i}") for i in range(5)]
    # Wait for all to complete
    results = await asyncio.gather(*tasks)
    end = time.time()
    print(f"Async approach took {end-start:.2f} seconds")
    return results

# Run both approaches
sync_results = sync_main()  # Takes ~5 seconds
async_results = asyncio.run(async_main())  # Takes ~1 second
```

This example demonstrates the dramatic differences between two approaches. The synchronous version processes requests sequentially, waiting for each to complete before starting the next. With five one-second operations, it takes roughly give seconds to complete. The asynchronous approach, however, initiates all requests concurrently and awaits their completion in parallel completing in just over one second. 

This performance gap widens further with more operations or longer wait times, highlighting why asynchronous programming is essential when building applications that make multiple calls to LLM services, databases, or other I/O-bound resources.

## Evolution of AsyncIO in Python

Before the introduction of AsyncIO in Python 3.4, asynchronous programming was often simulated using generators combined with callbacks or frameworks (like `tornado` or `Twisted`). Generators allowed developers to write asynchronous code in a way that resembled synchronous code, using `yield` keyword to pause and resume execution:

```python
# example of using generators simulating asynchronous programming
import time 

def async_task():
    yield "start task" 
    time.sleep(1) # simulate blocking I/O
    yield "task completed"

def run(generator):
    # a simple scheduler to run the generator
    # it iterates over the generator, simulating a 
    # scheduler that drives the asynchronous execution
    for step in generator:
        print(step)

task = async_task()
run(task)
```

The code above mimics asynchronous programming by breaking tasks into smaller steps and executing them **sequentially**. However, it doesn't provide the true concurrency. To do that, developers would need to use generators in combination with callbacks, leveraging exceptions like `StopIteration` to signal completion:

```python
import time

def async_sleeping(duration):
    yield f"sleeping for {duration} seconds..."
    time.sleep(duration)
    yield f"slept for {duration} seconds!"
    
def run(generator):
    try:
        while True:
            step = next(generator)
            print(step)
    except StopIteration:
        pass

task = async_sleeping(2)
run(task)
```

When a generator exhausted its `yield` statements, it raised `StopIteration`, signaling completion to the event loop. This allowed the loop to clean up resources or schedule new tasks. But still, generators are not true asynchronous programming, because generators can't handle true non-blocking I/O operations efficiently, and developers had to implement event loops and schedulers manually, leading to error-prone code.

Python 3.4 introduced the `asyncio` module, making a significant milestone in Python's asynchronous capabilities. This dedicated framework provided standardized tool for asynchronous programming, including event loops, coroutines, futures and tasks. However, the early syntax still relied on generators and decorators, and working with the event loop required considerable boilerplate code:

```python
# example from the book: 
import asyncio
import time

async def main():
    print(f"{time.ctime()} hello!")
    await asyncio.sleep(1.0)
    print(f"{time.ctime()} goodbye!")

# a loop instance is required to run coroutines 
loop = asyncio.get_event_loop()

# OPTIONAL:
# use create_task() to schedule the coroutine to be run on the loop 
# the returned object can be used to monitor the status of the task 
task = loop.create_task(main())

# this call blocks the current thread and keep the loop running 
# only until the given coroutine completes 
# other tasks scheduled on the loop will also run while the loop is running.
results = loop.run_until_complete(task)

# gather the still-pending tasks, cancel them
pending = asyncio.all_tasks(loop=loop)
for task in pending:
    task.cancel()
group = asyncio.gather(*pending, return_exceptions=True)
loop.run_until_complete(group)

# to be called on a stopped loop
# it will clear all the queues and shut down the executor
loop.close()
```

As shown above, developers had to manually create, manage and close the event loop using `get_event_loop`, `run_until_complete` and `loop.close()`, leading to verbose code even for simple asynchronous operations. This complexity was significantly reduced with Python 3.7's introduction of `asyncio.run()`, which handles all the event loop management automatically:

```python
import asyncio
import time

async def main():
    print(f"{time.ctime()} hello!")
    await asyncio.sleep(1.0)
    print(f"{time.ctime()} goodbye!")

# Python 3.7+
results = asyncio.run(main())
```

### Running blocking functions in an asynchronous context

One common challenge when working with asyncIO integrating existing synchronous, blocking code into an asynchronous application. This is particularly relevant when working with LLMs, as many existing Python libraries may not have sync-native APIs.

When faced with a blocking function that would halt your event loop, you need to offload it to a separate thread or process. Prior to Pytnon 3.9, this was typically done using `asyncio.run_in_executor()`:

```python 
import time 
import asyncio 

async def main():
    print(f"{time.ctime()} hello!")
    await asyncio.sleep(1.0)
    print(f"{time.ctime()} goodbye!")

def blocking():
    time.sleep(0.5)
    print(f"{time.ctime()} Hello from a thread!")

loop = asyncio.get_event_loop()
task = loop.create_task(main())

# run the blocking function in the default executor
# note that the first argument must be `None` to use the default executor.
loop.run_in_executor(None, blocking)
loop.run_until_complete(task)

pending = asyncio.all_tasks(loop=loop)
for task in pending:
    task.cancel()
group = asyncio.gather(*pending, return_exceptions=True)
loop.run_until_complete(group)
loop.close()
```

`asyncio.run_in_executor()` doesn't block the main thread, it only schedules the executor task to run. The executor task begins executing only after `run_until_complete()` is called, allowing the event loop to manage the execution alongside other async tasks.

**Note:** Python 3.9 introduced the cleaner and more intuitive `asyncio.to_thread()` function, allowing the preceding example to be rewritten more elegantly:

```python
import time 
import asyncio 

async def main():
    print(f"{time.ctime()} hello!")
    await asyncio.sleep(1.0)
    print(f"{time.ctime()} goodbye!")

def blocking():
    time.sleep(0.5)
    print(f"{time.ctime()} Hello from a thread!")

async def run():
    task = asyncio.create_task(main())
    await asyncio.to_thread(blocking)
    await task

asyncio.run(run())
```


### Progression of Python's Async Capabilities
Python's asynchronous programming features have evolved significantly over the past decade, with each version bringing important improvements to make async code more powerful, readable, and maintainable:

- Python 3.4 (2014): Introduction of AsyncIO module with `@asyncio.coroutine` decorator and `yield` from syntax
- Python 3.5 (2015): Native `async/await` syntax added, making async code more readable
- Python 3.6 (2016): Asynchronous generators and comprehensions
- Python 3.7 (2018): `asyncio.run()`, better debugging, performance improvements
- Python 3.8 (2019): `asyncio.create_task()` becomes the standard API, `asyncio.get_running_loop()` introduced
- Python 3.9 (2020): Improved error messages, `asyncio.to_thread()` added
- Python 3.10 (2021): Structural pattern matching for async results, `asyncio.TaskGroup`
- Python 3.11 (2022): Task exception groups, performance optimizations
- Python 3.12 (2023): Enhanced cancellation features, improved reliability
- Python 3.13 (2024): Improved task scheduling efficiency and memory usage optimization


## Core AsyncIO components 

At its core, asyncio is built on three fundamental concepts: **coroutines**, **the event loop**, and **tasks**. Think of them as the trinity of async programming in Python.

### Coroutines

Coroutines are the fundamental building blocks of AsyncIO, defined using `async def` syntax introduced in Python 3.5. They represent functions that can be paused and resumed, allowing other coroutines to run while waiting for I/O operations. This cooperative multitasking approach is what enables efficient concurrency without the overhead of multiple threads.

To understand how coroutines work, let's first look at a simple example:

```python
# this is a coroutine 
async def fetch_data():
    await asyncio.sleep(1)
    return "Data fetched"

# type(fetch_data) -> function 
# f = fetch_data(), type(f) -> coroutine
```

Under the hood, a coroutine is triggered by sending it a `None`; and when it returns, a `StopIteration` exception is raised. While these can be marked as the beginning and the end of a coroutine, `throw` can be used to inject exceptions into a coroutine:

```python 
coro = f()
coro.send(None)
coro.throw(Exception, "blah")
```

Or cancel a task:

```python
import asyncio 
async def f():
    try:
        while True: await asyncio.sleep(0)
    except asyncio.CancelledError:
        print("I was cancelled")
    else:
        return 111

coro = f()
coro.send(None)
coro.send(None)
coro.throw(asyncio.CancelledError)
# i was cancelled!
```

Coroutines bear a structural resemblance to generators, which is why generators were initially used to simulate coroutines before the introduction of native `async/await` syntax. Developers would use generators with special decorators (like `@asyncio.coroutine`) to approximate coroutine behavior.

What fundamentally distinguishes a coroutine from a regular function or generator is its execution model. When called, a **coroutine doesn't execute immediately** but instead returns a coroutine object - essentially a promise or placeholder for a future result. This deferred execution is what enables the event loop to orchestrate multiple coroutines efficiently.

### Event Loop

The event loop coordinates the execution of coroutines. It manages the switching between different coroutines, catches exceptions, listens to sockets and file descriptors for events, and ensures that the application remains responsive while handling multiple operations concurrently.

For most applications since Python 3.7, `asyncio.run(coro)` is the recommended approach, as it manages the event loop for you. This high-level function creates an event loop, runs the coroutine until it completes, and then closes the loop, handling many of the details that previously required manual implementation. 

The event loop in AsyncIO handles all of the switching between coroutines, as well as catching the exceptions, listening to sockets and file descriptors for events.

While modern Python asyncIO code often uses high-level APIs that manage the event loop automatically, sometimes you need direct access to the loop itself - for example, when scheduling custom callbacks, working with low-level transports, or integrating with non-asyncio code. Python provides several ways to access the current event loop, each appropriate for different scenarios: 

- `asyncio.get_running_loop()` (_recommended_): callable from inside the context of a coroutine 
- `asyncio.get_event_loop()`: only works in the same thread; can implicitly create new event loop if none exists 


### Tasks and Futures

Tasks and Futures are closely related concepts that represent different aspects of asynchronous execution:

A **Task** is a wrapper around a coroutine that schedules it for execution on the event loop. You create a Task by passing a coroutine to functions like `loop.create_task(coro)` or the newer `asyncio.create_task(coro)`. Once created, a Task actively executes its coroutine when the event loop runs, without requiring additional action from the developer.

A **Future**, on the other hand, is a lower-level construct that doesn't run any code itself. Futures serve as placeholders for results that will be available in the future. They provide a way to track the status of an operation (pending, completed, or failed) and access its result when available.

```python
import asyncio 

async def main(f: asyncio.Future):
    await asyncio.sleep(1)
    f.set_result("I have finished.")

loop = asyncio.get_event_loop()
fut = asyncio.Future()
print(fut.done()) # False 

loop.create_task(main(fut))
loop.run_until_complete(fut) # "I have finished."

print(fut.done()) # True 
print(fut.result()) # "I have finished."
```

In the example above, we created the Future instance `fut` using `asyncio.Future()`. By default, this Future instance is **tied to the loop** but is **not (will not be) attached to any coroutine**, as opposed to Task instances. Then, we schedule the `main()` coroutine, execute the future, the result of the future instance is set when the execution is completed.

From an implementation perspective, a Task inherits from Future - the Future class is a **superclass** that provides primitive building blocks for Tasks. While both represent asynchronous operations, a Future is more generic, representing a **future completion state** of any activity managed by the event loop. A Task is more specific, representing a running coroutine that's been scheduled on the event loop.

This relationship explains why Tasks have all the features of Futures (like the ability to check completion status or retrieve results) plus additional capabilities specific to managing coroutines.

#### `ensure_future()` vs. `create_task()`

Some developers recommend using `asyncio.create_task()` to run coroutines while others recommend `asyncio.ensure_future()`. In summary, `asyncio.ensure_future()` returns a Task instance when passing in a coroutine; it returns the input unchanged when passing in a Future instance. 

```python
import asyncio 

async def f():
    pass

coro = f()
loop = asyncio.get_event_loop()

task = loop.create_task(coro)
assert isinstance(task, asyncio.Task) # True

new_task = asyncio.ensure_future(coro)
assert isinstance(new_task, asyncio.Task) # True

new_new = asyncio.ensure_future(task)
assert new_new is task # True
```

## Beyond the Basics: Advanced AsyncIO Patterns

As you build more complex asynchronous applications, particularly those interacting with external services like LLM APIs, you'll need more advanced patterns to manage resources and control flow efficiently. 

## Async Context Management: `async with` 

Context managers are crucial for properly managing resources in Python, ensuring proper setup and cleanup even when exceptions occur. When working with asynchronous resources - like database connections or API sessions - standard context managers won't suffice because they can block the event loop.

Async context managers solve this problem by providing asynchronous versions of the familiar context management protocol. Instead of `__enter__` and `__exit__`, they use `__aenter__` and `__aexit__` methods that can be awaited. This allows resources acquisition and release to happen asynchronously without blocking the event loop.

```python
class AsyncDatabase:
    async def __aenter__(self):
        await self.connect() # connect to database object asynchronously 
        return self # returns the database object to be used in the with block

    async def __aexit__(self, exc_type, exc, tb): 
        await self.disconnect() # disconnect when exiting the with block

async def process_data():
    async with AsyncDatabase() as db: # creates database and connects
        await db.query("SELECT * FROM data")
```

The workflow follows a similar pattern to synchronous context managers but with asynchronous operations:

1. when we call `async with AsyncDatabase() as db`, it creates an `AsyncDatabase` instance 
2. the event loop awaits the `__aenter__` method, which connects to the database asynchronously 
3. once connected, the database object is assigned to the variable `db`
4. within the context block, we can use await and the database's methods (`await db.query()`)
5. when the `async with` block ends (or an exception occurs), the event loop awaits the `__aexit__` method, ensuring the database disconnects cleanly

This pattern is particularly valuable when working with LLM services, as it ensures connections are properly established and released, even if errors occur during processing.

### Simplifying Context Management with Decorators 

While class-based async context managers offer complete control, they require defining multiple methods and can be verbose. Python provides decorator-based alternatives that are often more concise for simpler cases.

First, let's recall how standard synchronous `@contextmanager` decorator works for regular context managers:

```python
from contextlib import contextmanager 

# note: download_webpage and update_stats are hypothetical functions
# that would typically perform network or file operations

# use @contextmanager to transform a generator
# function into a context manager
@contextmanager  
def web_page(url):
    try:
        data = download_webpage(url) # CPU bound 
        yield data 
    except Exception as e:
        print(f"Error: {str(e)}")
    finally:
        update_stats(url) # CPU bound 

with web_page("google.com") as data:
    process(data) # assume non CPU bound 
```

This code above works well for synchronous operations, but would block the event loop if used in an async context. For asynchronous operations, Python provides the `@asynccontextmanager` decorator, which is analogous to the synchronous version but designed for async functions:

```python
from contextlib import asynccontextmanager

# when using the @asynccontextmanager, the 
# decorated function needs to be async  
@asynccontextmanager  
async def web_page(url):
    try:
        # download_webpage() needs to be modified
        # to be async as well
        data = await download_webpage(url) 
        yield data 
    except Exception as e:
        print(f"Error: {str(e)}")
    finally:
        await update_stats(url)

async with web_page("google.com") as data:
    process(data) # assume non CPU bound 
```

### Bridging Synchronous and Asynchronous Worlds

A common challenge when working with LLMs or other external services is integrating existing synchronous libraries into an async application. If you're importing functions like `download_webpage()` from a third party library that doesn't support async operations, you need to adapt them to work within your async context.

There are two primary approaches to wrap synchronous functions for use in async code:

- using `run_in_executor()`:

```python 
from contextlib import asynccontextmanager
import asyncio 

@asynccontextmanager
async def web_page(url):
    try: 
        loop = asyncio.get_event_loop()
        data = await loop.run_in_executor(None, download_webpage, url)
        yield data 
    except Exception as e:
        print(f"Error: {str(e)}")
    finally:
        await loop.run_in_executor(None, update_stats, url)

async with web_page("google.com") as data:
    process(data)
```

- using `to_thread()`:

```python
from contextlib import asynccontextmanager 
import asyncio 

@asynccontextmanager
async def web_page(url):
    try:
        data = await asyncio.to_thread(download_webpage, url)
        yield data 
    except Exception as e:
        print(f"Error: {str(e)}")
    finally:
        await asyncio.to_thread(update_stats, url)

async with web_page("google.com") as data:
    process(data)
```


## Working with Streams: Async Iteration and Comprehension 

When working with LLMs, you might need to process streaming responses or handle data that arrives incrementally. AsyncIO provides specialized syntax for working with asynchronous iterables - data sources that yield values asynchronously over time.

```python
async def process_stream():
    async for item in async_stream():
        result = await process_item(item)
        yield result

# Async comprehension
results = [item async for item in process_stream()]
```

Under the hood, async iteration operations similarly to the synchronous versions but with asynchronous methods: when you use `async for`, Python calls the object's `__aiter__` method to get an async iterator. Then for each iteration, it awaits the iterator's `__anext__` method, which returns a value for each iteration and raises `StopAsyncIteration` when finished.

This pattern is particularly useful when working with streaming LLM responses, where tokens or chunks arrive over time and need to be processed as they become available.

## Ensuring Reliability: Shutdown gracefully

For LLM applications running as services, proper shutdown handling is critical. When a service terminates abruptly, it might leave resources in an inconsistent sate or drop user requests mid-processing. Implementing graceful shutdown mechanisms ensures your application can terminate cleanly, completing critical operations and releasing resources properly.

When we run Task instances using `asyncio.run()`, it automatically helps clean up pending tasks in simple scenarios. However, in real-world applications, you may encounter error messages like "Task was destroyed but it is pending" during shutdown. These typically occur when:

- `asyncio.run()` exits before all tasks complete 
- a parent task is cancelled without handling child tasks 
- the event loop is closed while tasks are still running 
- background tasks are created but not properly tracked

### Shutdown Strategies and Best Practices

Implementing proper shutdown handling especially important for LLM applications that may be processing user queries or performing resource-intensive operations. Here are key strategies for ensuring graceful shutdown:

- keep track of all background tasks 
- use proper signal handlers (for `SIGINT`/`SIGTERM`)：
  - `SIGINT` corresponds to `KeyboardInterrupt`
  - `SIGTERM` is more common in network services, which corresponds to `kill` in a Unix shell
- cancel all tasks during shutdown 
- wait for cancellations to complete (with timeout)
- use `try`/`finally` to ensure cleanup runs
- avoid creating any tasks inside a cancellation handler 
- consider using `gather(..., return_exceptions=True)`

The example below provides a minimal approach to handling a keyboard interrupt (Ctrl+C):

```python
# a simple example to handle KeyboardInterrupt / Ctrl-C
import asyncio 
from signal import SIGINT, SIGTERM

async def main():
    while True:
        print("<Your app is running>")
        await asyncio.sleep(1)

if __name__ == "__main__":
    loop = asyncio.get_event_loop()
    task = loop.create_task(main())
    try:
        loop.run_until_complete(task)
    except KeyboardInterrupt:
        print("Got signal: SIGINT, shutting down") # only Ctrl+C can stop the loop
    tasks = asyncio.all_tasks(loop=loop)
    for t in tasks:
        t.cancel()
    group = asyncio.gather(*tasks, return_exceptions=True)
    loop.run_until_complete(group)
    loop.close()
```

However, real-world applications typically need more robust signal handling. For instance, you might want to handle both SIGTERM and SIGINT signals, provide a graceful cleanup period, and ensure your application responds appropriately even when multiple shutdown signals are received in quick succession. The following example builds on the previous one by implementing these more advanced features:

```python
# added feature:
# - handling both SIGTERM and SIGINT
# - handle CancelledError, and the cleanup code 
# - handle multiple signals 

# when hitting Ctrl-C multiple times, the process shuts down when 
# the main() coroutine eventually completes.

import asyncio 
from signal import SIGINT, SIGTERM

async def main():
    try:
        while True:
            print("<Your app is running>")
            await asyncio.sleep(1)
    except asyncio.CancelledError: # a callback handler when a signal is received
        for i in range(3): # wait for 3 seconds while the run_until_complete() is running
            print("<Your app is shutting down>")
            await asyncio.sleep(1)

def handler(sig):
    """
    Handler to stop the loop, unblock the run_forever,
    and allow pending tasks collection and cancellation
    """
    loop.stop()
    print(f"Got signal: {sig!s}, shutting down")
    # ignore signals while shutting down
    loop.remove_signal_handler(SIGTERM)
    # disable KeyboardInterrupt / Ctrl-C, 
    # unable to use loop.remove_signal_handler(SIGINT)
    # otherwise KeyboardInterrupt / Ctrl-C will be restored 
    loop.add_signal_handler(SIGINT, lambda: None)

if __name__ == "__main__:
    loop = asyncio.get_event_loop()
    for sig in (SIGTERM, SIGINT):
        # setting a signal_handler on SIGINT means KeyboardInterrupt 
        # will no longer be raised on SIGINT
        loop.add_signal_handler(sig, handler, sig)
    task = loop.create_task(main())
    loop.run_forever() # only stops when either SIGINT or SIGTERM is sent to the process
    tasks = asyncio.all_tasks(loop=loop)
    for t in tasks:
        t.cancel()
    group = asyncio.gather(*tasks, return_exceptions=True)
    loop.run_until_complete(group)
    loop.close()
```
While the previous example works well for many applications, it still uses the older event loop management style. In modern Python (3.7+), we typically use `asyncio.run()` to manage the event loop. However, this creates a special challenge for signal handling, since asyncio.run() takes control of the event loop creation and management.

The following example shows how to implement robust signal handling in an application using `asyncio.run()`. Note how we need to adapt our approach to work within the constraints of the higher-level API:

```python
import asyncio
from signal import SIGINT, SIGTERM

async def main():
    loop = asyncio.get_event_loop()
    # because asyncio.run() takes control of the event loop
    # startup, the handlers must be added in the main() func
    for sig in (SIGINT, SIGTERM):
        loop.add_signal_handler(sig, handler, sig)

    try:
        while True:
            print("<Your app is running>")
            await asyncio.sleep(1)
    except asyncio.CancelledError:
        for i in range(3):
            print("Your app is shutting down.")
            await asyncio.sleep(1)

def handler(sig):
    loop = asyncio.get_running_loop()
    # cannot stop the loop using loop.stop()
    # otherwise will get warning about how the 
    # loop was stopped before the task created 
    # for main() was completed. 
    for task in asyncio.all_tasks(loop=loop):
        task.cancel()
    print(f"Got signal: {sig!s}, shutting down")
    loop.remove_signal_handler(SIGTERM)
    loop.add_signal_handler(SIGINT, lambda: None)

if __name__=="__main__":
    asyncio.run(main())
```

## Advanced AsyncIO Modules 

`asyncio.gather()` allows you to run multiple coroutines concurrently and wait for all of them to complete, returning their results in the same order as the input coroutines

```python
import asyncio

async def fetch_data(url):
    await asyncio.sleep(1)
    return f"Data from {url}"

async def main():
    results = await asyncio.gather(
        fetch_data("google.com"),
        fetch_data("bing.com"),
        return_exceptions=True
    )
    print(results)

asyncio.run(main())
```

`asyncio.as_complete()` returns an iterator that yields tasks as they complete, regardless of the order they were submitted. This is perfect for scenarios where you want to process results as soon as they're available.

```python
import asyncio 
import random 

async def process_item(item):
    await asyncio.sleep(random.uniform(0.5, 3))
    return f"processed {item}"

async def main():
    tasks = [process_item(i) for i in range(10)]

    for future in asyncio.as_completed(tasks):
        result = await future 
        print(f"got result: {result}")

asyncio.run(main())
```

`asyncio.Semaphore()` limit the number of coroutines that can access a resource concurrently. This is crucial when working with APIs that have rate limits or when you need to control resource usage.

```python 
import asyncio 
import aiohttp

async def fetch_url(url, session, semaphore):
    async with semaphore:
        print(f"fetching {url}")
        async with session.get(url) as response:
            return await response.text()

async def main():
    semaphore = asyncio.Semaphore(5)
    urls = [f"https://example.com/{i}" for i in range(20)]

    async with aiohttp.ClientSession() as session:
        tasks = [fetch_url(url, session, semaphore) for url in urls]
        results = await asyncio.gather(*tasks)

    print(f"completed {len(results)} requests")

asyncio.run(main())
```

`asyncio.TaskGroup()` provides a cleaner, more structured approach to concurrent task management with automatic cleanup and error propagation. 

```python 
import asyncio 

async def process_data(id):
    await asyncio.sleep(1)
    print(f"processed data {id}")
    return id*10

async def main():
    async with asyncio.TaskGroup() as tg:
        task1 = tg.create_task(process_data(1))
        task2 = tg.create_task(process_data(2))
        task3 = tg.create_task(process_data(3))
    
    print(f"results: {task1.result()}, {task2.result()}, {task3.result()}")

asyncio.run(main())
```

## Writing Tests for AsyncIO Code with pytest

Testing asynchronous code requires special handling. The `pytest-asyncio` plugin makes this straightforward by providing the `pytest.mark.asyncio` decorator. 

```python 
import pytest
import asyncio 

async def fetch_data(id):
    await asyncio.sleep(0.1)
    return f"Data for {id}"

# mark the test as asyncio 
@pytest.mark.asyncio
async def test_fetch_data():
    result = await fetch_data(1)
    assert result == "Data for 1"

# testing exceptions 
@pytest.mark.asyncio 
async def test_fetch_data_with_timeout():
    with pytest.raises(asyncio.TimeoutError):
        await asyncio.wait_for(asyncio.sleep(1), timeout=0.5)
```

### Mocking Asynchronous Functions 

Testing async code often requires mocking external service. Here's how to do it:

```python 
import pytest
from unittest.mock import AsyncMock, patch 

async def get_user_data(client, user_id):
    return await client.fetch_user(user_id)

@pytest.mark.asyncio
async def test_get_user_data():
    # create a mock client with an async method
    mock_client = AsyncMock()
    mock_client.fetch_user.return_value = {"id": 1, "name": "Test User"}

    # test our function with the mock client 
    result = await get_user_data(mock_client, 1)

    # verify the result and that the mock was called correctly
    assert result == {"id": 1, "name": "Test User"}
    mock_client.fetch_user.assert_called_once_with(1)

@pytest.mark.asyncio
async def test_with_patch():
    with patch("module_name.api_client") as mock_client:
        mock_client.fetch_data = AsyncMock(return_value="mocked data")
        # ...
```

### Test Asynchronous Context Managers

```python
import pytest
from unittest.mock import AsyncMock 

@pytest.mark.asyncio 
async def test_async_context_manager():
    # create a mock for an async context manager 
    mock_db = AsyncMock()
    mock_db.__aenter__.return_value = mock_db 
    mock_db.query.return_value = ["result1", "result2"]

    # use the mock in an async with statement 
    async with mock_db as db:
        result = await db.query("SELECT * FROM TABLE")

    # verify results
    assert result == ["result1", "result2"]
    mock_db.query.assert_called_once_with("SELECT * FROM TABLE")
    mock_db.__aexit__.assert_called_once()
```

## Ecosystem Integrations 

## Working with HTTP Requests: `aiohttp`

`aiohttp` is an asynchronous HTTP client/server framework built on top of asyncio, perfect for making non-blocking HTTP requests. Its key features include:

- reuses connections from a ClientSession pool 
- supports streaming responses 
- handles cookies, headers and authentication 
- provides both client and server implementation 


```python
import asyncio 
import aiohttp 
import time 

async def fetch_url(session, url):
    async with session.get(url) as response:
        return await response.text()

async def fetch_all(urls):
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_url(session, url) for url in urls]
        return await asyncio.gather(*tasks)

async def main():

    urls = [
        "https://example.com",
        "https://python.org",
        "https://docs.python.org"
    ]

    start = time.perf_counter()
    results = await fetch_all(urls)
    end = time.perf_counter()

    print(f"Fetched {len(results)} sites in {end-start:.2f} seconds")
    print(f"First 100 chars of each: {[r[:100] for r in results]}")

asyncio.run(main())
```

## Asynchronous File Operations: `aiofiles`

`aiofiles` provides asynchronous file I/O operations, allowing file operations to run without blocking the event loop. It is particularly useful when writing/reading large files, processing multiple files concurrently, working with files while maintaining UI responsiveness, and integrating file operations with other async I/O operations.

```python
import asyncio
import aiofiles

async def read_large_file(filename):
    async with aiofiles.open(filename, 'r') as file:
        return await file.read()

async def write_data(filename, data):
    async with aiofiles.open(filename, 'w') as file:
        await file.write(data)

async def process_files(filenames):
    tasks = [read_large_file(filename) for filename in filenames]
    contents = await asyncio.gather(*tasks)
    
    # Process contents...
    results = [content.upper() for content in contents]****
    
    # Write results
    write_tasks = [
        write_data(f"processed_{filename}", result) 
        for filename, result in zip(filenames, results)
    ]
    await asyncio.gather(*write_tasks)

async def main():
    await process_files(['file1.txt', 'file2.txt', 'file3.txt'])

asyncio.run(main())
```

## Conclusion 

AsyncIO has transformed Python's approach to handling I/O-bound operations, offering a simple yet powerful approach to concurrency without the complexity of traditional multithreading. Its core advantages - improved responsiveness, maximized resource utilization, and elegant syntax - make it particularly well-suited for AI application development. The extensive AsyncIO ecosystem, with libraries for HTTP requests, file operations, and database access, extends these capabilities across all I/O domains. As AI services continue to expand in importance and complexity, AsyncIO's role will only grow, becoming a fundamental pattern for developers building responsive, scalable applications in an increasingly API-driven landscape.