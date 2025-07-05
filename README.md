# Kotlin-Coroutines

● [Kotlin Coroutines and Flow for Android Development](https://www.udemy.com/course/coroutines-on-android/?couponCode=NVDINCTA35CTR)

---

![Kotlin Coroutines and Flow for Android Development](https://github.com/user-attachments/assets/11008b26-e3a2-473a-8b5e-eaee062f16b2)

---


## What is a coroutine in Kotlin?

- In Kotlin, a coroutine is a concurrency design pattern that simplifies asynchronous code, making it more readable and easier to manage. Think of them as lightweight threads that can suspend and resume execution without blocking the underlying thread. This allows you to perform multiple tasks concurrently without the overhead of traditional threading. 

- Here's a breakdown of key aspects:

  1. Asynchronous Programming Made Simple: 
    Coroutines allow you to write asynchronous code (like fetching data from a network or accessing a database) in a sequential, synchronous style, which makes it much easier to reason about.
    They help to manage long-running tasks that would otherwise block the main thread and make your app unresponsive. 

  2. Suspending and Resuming: 
    A crucial concept is the suspend function, marked with the suspend keyword.
    When a coroutine encounters a suspend function, its execution can be suspended (paused) without blocking the thread it's running on.
    The coroutine can then be resumed later when the suspending function has completed its work. 

  3. Lightweight and Efficient: 
    Coroutines are significantly less resource-intensive than traditional threads.
    We can run many coroutines on a single thread, enabling efficient concurrency without the high cost of creating and managing a large number of threads. 

  4. Structured Concurrency: 
    Coroutines in Kotlin follow structured concurrency.
    This means coroutines are launched within a specific CoroutineScope, which defines their lifecycle.
    Structured concurrency helps to manage coroutines effectively, ensuring that they are canceled when they are no longer needed, preventing resource leaks and improving error handling. 

  5. Builders: 
    Kotlin provides functions like launch and async to launch new coroutines.
    launch is used for "fire and forget" tasks, where you don't need a result back.
    async is used when you expect a result from the coroutine, returning a Deferred object that allows you to get the result using await().


🔍 1. What is a CoroutineScope?
A CoroutineScope defines where and how long your coroutines will run.

💡 Think of it as a container for coroutines — it keeps track of them and allows cancellation all at once.

Each scope comes with:

1. A Job (manages the lifecycle of coroutine work)
2. A CoroutineContext (defines how coroutines behave; it's a container for Job, Dispatcher, etc.)
3. A Dispatcher (decides which thread the coroutine runs on)

🚨 2. Why Scopes Are Required in Android?
1. Structured concurrency
2. Lifecycle management
3. Exception handling
4. Cancellation
5. Memory leak prevention

🧬 3. Anatomy of a CoroutineScope:

```
interface CoroutineScope {
	val coroutineContext: CoroutineContext
}
```

1: Ordinary Scope

```
val scope = CoroutineScope(Job())
val job = scope.launch {
// Coroutine work
}
```


2: Supervisor Scope

```
val supervisorScope = CoroutineScope(SupervisorJob())
val supervisorJob = supervisorScope.launch {
// Independent child coroutines
}
```

Example:

```
suspend fun myCoroutine() {
        val scope = CoroutineScope(Job() + Dispatchers.Default + CoroutineExceptionHandler{ _, throwable -> println("CoroutineExceptionHandler: $throwable") } )

        val job1 = scope.launch {
            launch {
                println("Task-1 stared.")
                delay(1000)
                println("Task-1 finished.")
            }
            launch {
                println("Task-2 stared.")
                delay(2000)
                println("Task-2 finished.")
            }
        }

        val job2 = scope.launch {
            launch {
                println("Task-3 stared.")
                delay(800)
                println("Task-3 finished.")
            }
            launch {
                println("Task-4 stared.")
                delay(1000)
                println("Task-4 finished.")
            }
        }
        
        job1.invokeOnCompletion {
            println("Job-1 Completed.")
        }

        job2.invokeOnCompletion {
            println("Job-2 Completed.")
        }

        joinAll(job1, job2)
        scope.cancel()
    }
```

## 🚀 Coroutine Scope Reuse

- You can reuse a CoroutineScope to launch multiple coroutines — as long as it’s active (not cancelled).
- This is safe, efficient, and encouraged:

```
val scope = CoroutineScope(Job() + Dispatchers.Default)

scope.launch { println("🔹 Task-1") }
scope.launch { println("🔸 Task-2") } // ✅ Reuse is fine

```

### ❌ What Happens After scope.cancel()?

```
val scope = CoroutineScope(Job())
scope.cancel()

scope.launch {
    println("❌ This will never run")
}
```

### ⚠️ Internally

- The coroutine is immediately cancelled
- The scope becomes inactive
- Any new coroutine won’t start and doesn’t start executing any logic 
- A CancellationException is thrown internally
- The app won’t crash

### ✅ Correct Reuse Pattern

- To reuse after cancellation, create a new scope:

```
var scope = CoroutineScope(Job() + Dispatchers.Default)

scope.launch { println("Task-A") }

scope.cancel() // Scope is now dead

scope = CoroutineScope(Job() + Dispatchers.Default)
scope.launch { println("Task-B") } // ✅ Works

```

## 🚀 CoroutineScope .launch & .async

### ✅ What is CoroutineScope.launch {}?
- launch {} is an extension function on CoroutineScope
- It is a coroutine builder — it starts a new coroutine without expecting any result.
- It is the entry point of a coroutine inside a scope.
- It returns a Job, which represents the coroutine’s lifecycle (you can cancel it, observe completion, etc.)
- launch is used when:
	- You don’t need a result
	- You're triggering jobs like database writes, logging, etc.
	- You care about lifecycle control via Job

```
val job = CoroutineScope(Dispatchers.Default).launch {
    println("Running something...")
}
```

### ✅ What is CoroutineScope.async {}?
- async {} is an extension function on CoroutineScope
- It is a coroutine builder — used to start a new coroutine that returns a result
- It is also an entry point into a coroutine
- It returns a Deferred<T>, a lightweight future-like object. You get the result by calling await() on it (which is a suspend function)
- If you use async {} and don’t call await(), the coroutine still runs, and:
	- Any exception will be lost. Exceptions will be silently missed.
	- Your result will be unused
	- May cause bugs or memory leaks
- So always remember to await() call

```
val deferred: Deferred<Int> = CoroutineScope(Dispatchers.Default).async {
	println("🧮 Calculating...")
    77
}

val result = deferred.await()
println("✅ Result: $result")
```

### 🧱 Coroutine Builders – launch {} and async {}
✅ CoroutineScope.launch {} and CoroutineScope.async {} are coroutine builders
used to spawn child coroutines that inherit the job and context from the surrounding scope or coroutine.

---

## 📘 Kotlin Coroutines – Ordinary Job Behavior with Exception Flow

### 🔰 What is an Ordinary Job?

An **Ordinary Job** in Kotlin Coroutines is created using `Job()` without any additional supervision logic. It represents a coroutine's lifecycle and propagates cancellation **down** and **up** the coroutine hierarchy.

> 🔥 **Key Rule:**  
> If any child coroutine fails, it **cancels the parent job**, which then cancels all other sibling coroutines. This is the default behavior of structured concurrency.

---

### 🔗 Core Principle: Ordinary Job is Strict

> If any child coroutine fails, the **entire job hierarchy gets cancelled**.

This is useful when you want **fail-fast behavior** — like in structured concurrency.

---

### 🧪 Example: Understanding Ordinary Job Cancellation

```kotlin
suspend fun myCoroutine() {
    val scope = CoroutineScope(
        Job() + Dispatchers.Default +
        CoroutineExceptionHandler { _, throwable ->
            println("CoroutineExceptionHandler: $throwable")
        }
    )

    val job1 = scope.launch {
        launch {
            println("Task-1 started.")
            delay(2000)
            println("Task-1 finished.")
        }

        launch {
            println("Task-2 started.")
            delay(1000)
            throw RuntimeException("Task-2 failed!")
            println("Task-2 finished.") // Never executes
        }
    }

    val job2 = scope.launch {
        launch {
            println("Task-3 started.")
            delay(800)
            println("Task-3 finished.")
        }

        launch {
            println("Task-4 started.")
            delay(1000)
            println("Task-4 finished.")
        }
    }

    job1.invokeOnCompletion {
        println("Job-1 Completed." + (it?.let { " Exception: $it" } ?: ""))
    }

    job2.invokeOnCompletion {
        println("Job-2 Completed." + (it?.let { " Exception: $it" } ?: ""))
    }

    joinAll(job1, job2)
    scope.cancel()
}

```

### 🧵 Output Snapshot (Expected)

```
Task-1 stared.
Task-2 stared.
Task-3 stared.
Task-4 stared.
Task-3 finished.
Job-2 Completed. Exception: kotlinx.coroutines.JobCancellationException: Parent job is Cancelling; job=JobImpl{Cancelling}@1a6dee65
CoroutineExceptionHandler: java.lang.RuntimeException
Job-1 Completed. Exception: java.lang.RuntimeException
```

### 📌 What Happens Internally?

All tasks are launched in the same `CoroutineScope`, which uses an **ordinary `Job()`** as its context.

- `Task-2` throws a `RuntimeException` after 1 second.
- Because the scope uses an **ordinary Job**, the failure:
  - Cancels `job1`.
  - Propagates upward and cancels the entire `CoroutineScope`.
  - Consequently, `job2` is also cancelled—even though its child tasks (`Task-3`, `Task-4`) are unrelated and running fine.


### ❗ Important Rule of Ordinary Job

    In a coroutine hierarchy built on a regular Job(),
    if any child fails, the entire scope and all siblings are cancelled.

This behavior ensures structured concurrency, but can be too aggressive in scenarios where tasks are independent.

### ✅ Visual Hierarchy

```
CoroutineScope (Job)
├── job1
│   ├── Task-1
│   └── Task-2 ❌ → Exception
│
└── job2            ← Gets cancelled ❌
    ├── Task-3
    └── Task-4
```

---

## 📘 Kotlin Coroutines – SupervisorJob Behavior with Exception Isolation

### 🔰 What is a SupervisorJob?

A **SupervisorJob** is a special kind of `Job` in Kotlin Coroutines where the failure of a child **does not cancel its siblings** or the parent scope.

> 💡 It provides **exception isolation**, which means that one coroutine's failure **does not bring down the whole scope**.

### 🔗 Core Principle: SupervisorJob is Tolerant

> If any child coroutine fails, **only that child is cancelled** — the rest continue unaffected.

This is ideal when you want **resilience and isolation**, not fail-fast behavior.

---

### 🧪 Example: Understanding SupervisorJob Isolation

```kotlin
suspend fun myCoroutine() {
    val supervisorScope = CoroutineScope(
        SupervisorJob() + Dispatchers.Default +
                CoroutineExceptionHandler { _, throwable ->
                    println("CoroutineExceptionHandler: $throwable")
                }
    )

    val job1 = supervisorScope.launch {
        launch {
            println("Task-1 started.")
            delay(2000)
            println("Task-1 finished.")
        }

        launch {
            println("Task-2 started.")
            delay(1000)
            throw RuntimeException("Task-2 failed!")
            println("Task-2 finished.") // Never executes
        }
    }

    val job2 = supervisorScope.launch {
        launch {
            println("Task-3 started.")
            delay(800)
            println("Task-3 finished.")
        }

        launch {
            println("Task-4 started.")
            delay(1000)
            println("Task-4 finished.")
        }
    }

    job1.invokeOnCompletion {
        println("Job-1 Completed." + (it?.let { " Exception: $it" } ?: ""))
    }

    job2.invokeOnCompletion {
        println("Job-2 Completed." + (it?.let { " Exception: $it" } ?: ""))
    }

    joinAll(job1, job2)
    supervisorScope.cancel()
}
```

### 🔍 What Happens Internally?

All tasks are launched in the same `CoroutineScope`, which uses a **SupervisorJob()**.

- `Task-2` throws a `RuntimeException` after 1 second.
- Since the scope uses a **SupervisorJob**:
  - Only `Task-2` fails and is cancelled.
  - `job1` finishes because `Task-1` is unaffected.
  - `job2` and its children (`Task-3`, `Task-4`) continue to run normally.




### ❗ Important Rule of SupervisorJob

    In a coroutine hierarchy built on a SupervisorJob(),
    If any child fails, only that child is cancelled, others continue normally.

### ✅ Visual Hierarchy

```
CoroutineScope (SupervisorJob)
├── job1
│   ├── Task-1 ✅
│   └── Task-2 ❌ → Exception
│
└── job2            ← Runs normally ✅
    ├── Task-3 ✅
    └── Task-4 ✅
```

## 📘 `Dispatchers.Main` in Android

### 🚀 What is `Dispatchers.Main`?

`Dispatchers.Main` is a coroutine dispatcher designed to run coroutines on the **main (UI) thread** of Android.

It is used for:
- UI updates
- ViewModel coroutine launching (`viewModelScope`)
- Observing LiveData / Flows
- Safe interaction with Android Views

### ⚠️ Main Dispatcher – Concurrent but Not Parallel

- Coroutines on Dispatchers.Main can start at the same time (concurrently), but they execute one at a time (not in parallel).

- That’s because:
  1. The Main thread is single-threaded
  2. Only one coroutine can run at any given moment
  3. Others are suspended and resumed in order
 
### 🧪 Example: Synchronous (Sequential) behaviour

 ```
  class MyViewModel : ViewModel() {

    init {
        viewModelScope.launch {
            repeat(1000) {
                println("Task-1: Count-$it")
            }
        }

        viewModelScope.launch {
            repeat(1000) {
                println("Task-2: Count-$it")
            }
        }
    }
}
```

### 🧵 Output Snapshot (Expected)

```
Task-1: Count-0
...
Task-1: Count-999
Task-2: Count-0
...
Task-2: Count-999
```

### 🧠 Behavior

- viewModelScope uses Dispatchers.Main by default.
- Android’s Main thread is single-threaded, meaning it cannot run multiple tasks in parallel.
- Both launch {} blocks are submitted concurrently, but they are scheduled on the same thread.
- The first coroutine starts and runs a tight repeat(1000) loop.
- Since there is no suspension point like delay(), yield(), or withContext() in the first coroutine, it never gives up control of the thread.
- As a result, the second coroutine must wait for the first to complete.
- This leads to sequential execution, even though launch syntactically looks concurrent.


### ❗ Core Principle

- On Dispatchers.Main, only one coroutine executes at a time because the main thread is single-threaded.
- Without any suspension points (delay, yield, withContext), a coroutine blocks the main thread completely until it finishes.
- Therefore, coroutines run sequentially, even if launched concurrently.
- 🔄 What If It Hits a Suspension Point?
    - When a coroutine reaches a suspension point like delay(), yield(), or withContext(...):
    - It suspends its execution, releases the main thread, and goes into a waiting state.
    - This gives other pending coroutines a chance to run, enabling concurrent (interleaved) behavior.
    - Execution resumes when the suspension completes (e.g., after delay time or background task).
    - At this point, coroutine behavior switches from synchronous (sequential) to concurrent (interleaved) — but still not parallel.

---

### 🧪 Example: Concurrent (Interleaved) behaviour

 ```
class MyViewModel : ViewModel() {

    init {
        viewModelScope.launch {
            repeat(1000) {
                println("Task-1: Count-$it")
                delay(100)
            }
        }

        viewModelScope.launch {
            repeat(1000) {
                println("Task-2: Count-$it")
                delay(100)
            }
        }
    }
}
```

### 🧵 Output Snapshot (Expected)

```
Task-1: Count-0
Task-2: Count-0
Task-1: Count-1
Task-2: Count-1
Task-1: Count-2
Task-2: Count-2
...
```

### 🧠 Behavior: Concurrent (Interleaved)

- viewModelScope.launch {} creates two coroutines on the Main dispatcher.
- Each coroutine executes repeat(1000) with a delay(100) after every print.
- delay() is a suspension point that:
- Releases the main thread when hit.
- Suspends the current coroutine without blocking the UI.
- Allows the other coroutine to resume and run.

### 🔁 Execution Cycle

```
Main Thread
↓
Task-1: Count-0
delay(100) → suspends → allows Task-2
Task-2: Count-0
delay(100) → suspends → back to Task-1
...
```

### ❗ Important Reminder
 - Coroutines are not parallel on Dispatchers.Main — they are just interleaved using suspension.
 - No two coroutines run at the same moment, but they take turns efficiently by suspending and resuming.
   
---

## 📘 Dispatchers.Default in Kotlin Coroutines

### 🚀 What is Dispatchers.Default?

- Dispatchers.Default is a coroutine dispatcher optimized for CPU-intensive tasks.
- It runs coroutines on a shared pool of background threads, allowing true parallelism, and the number of threads is based on available CPU cores: (at least 2), and typically equal to the number of cores (2, 4, 8, etc.)
- Coroutines launched with Dispatchers.Default can run simultaneously in parallel on multiple threads.
- Suspension points (yield, delay, withContext, etc.) are not required for parallelism.
- It also runs coroutines concurrently, especially when many coroutines share the same thread — similar to Dispatchers.Default is when the number of coroutines exceeds the available CPU threads.

- It is used for:
	- Sorting large lists
	- Performing heavy calculations
	- Parsing big JSON/XML files
	- Executing pure computational logic without blocking the main thread


### 🧪 Example: Parallel (Asynchronous) behaviour

 ```
class MyViewModel : ViewModel() {

    init {
        viewModelScope.launch(Dispatchers.Default) {
            repeat(1000) {
                println("Task-1: Count-$it")
            }
        }

        viewModelScope.launch(Dispatchers.Default) {
            repeat(1000) {
                println("Task-2: Count-$it")
            }
        }
    }
}
 ```

### 🧵 Output Snapshot (Expected)

```
Task-1: Count-0
Task-2: Count-0
Task-1: Count-1
Task-2: Count-1
Task-1: Count-2
Task-2: Count-2
...
```

---

## 📘 Dispatchers.IO in Kotlin Coroutines

### 🚀 What is Dispatchers.IO?

	- Dispatchers.IO is a coroutine dispatcher optimized for I/O-bound operations.
	- It uses a shared thread pool with a larger number of threads than Dispatchers.Default.
 	- Dispatchers.IO can run a large number of coroutines concurrently, even if some block.
	- It starts with a few threads but can expand up to 64 threads to handle blocking tasks efficiently.
	- If multiple coroutines share the same thread (e.g., >64 coroutines), they still run concurrently using suspension (delay, withContext, yield, etc.).

	- It is used for:
		- Reading/writing files
		- Database access
		- Network calls (Retrofit, HTTP, etc.)
		- Any long-running blocking I/O task


### 🧠 Behavior

	✅ Designed for blocking I/O tasks — it's okay if a coroutine blocks a thread for a while.
    ✅ Can grow its thread pool dynamically, up to 64 threads (default upper limit).
    ✅ Coroutines run concurrently, even when sharing threads — by suspending at I/O points.


### 🧪 Example: Parallel (Asynchronous) behaviour

 ```
class MyViewModel : ViewModel() {

    init {
        viewModelScope.launch(Dispatchers.IO) {
            repeat(1000) {
                println("Task-1: Count-$it")
            }
        }

        viewModelScope.launch(Dispatchers.IO) {
            repeat(1000) {
                println("Task-2: Count-$it")
            }
        }
    }
}
 ```

### 🧵 Output Snapshot (Expected)

```
Task-1: Count-0
Task-2: Count-0
Task-1: Count-1
Task-2: Count-1
Task-1: Count-2
Task-2: Count-2
...
```

---

## 🔁 Coroutine Thread Switching

	- A coroutine can start on one thread and finish on another.
	- Coroutines are not tightly bound to a single thread — they’re bound to a dispatcher, which may assign different threads across suspension/resume points.
 	- It applies to:
  		- Dispatchers.Default
		- Dispatchers.IO
		- Dispatchers.Unconfined (if it suspends)
		- Any custom coroutine dispatcher
	- Exceptions:
		- On Dispatchers.Main (Android UI), it will always resume on the same Main thread.
		- If you use a newSingleThreadContext, coroutine will resume on the same thread to guarantee sequential behavior.


### 🧪 Example: Thread Switch After Suspension

```
fun main() = runBlocking {
    launch(Dispatchers.Default) {
        println("Started on: ${Thread.currentThread().name}")
        delay(500)
        println("Resumed on: ${Thread.currentThread().name}")
    }
}
```

### 🧵 Output Snapshot (Expected)

```
Started on: DefaultDispatcher-worker-1
Resumed on: DefaultDispatcher-worker-3
```

---

## ❌ Coroutine Cancellation

### Job Cancellation at Suspension Points:

🚦 Cancellation in Kotlin Coroutines
	- Cancellation in coroutines is cooperative —
	- The coroutine must reach a suspension point to respond to cancellation.
 	- A suspension point is a place where the coroutine pauses and checks for cancellation. 
  	Most common:
		- delay()
		- yield()
		- withContext()

```
  suspend fun main() {
    val scope = CoroutineScope(Job() + Dispatchers.Default)

    val job = scope.launch {
        println("Task started on ${Thread.currentThread().name}")
        delay(5000) // ← Suspension point (cancellation possible here)
        println("Task completed")
    }

    delay(1000)
    job.cancel()
}
```


### 🧵 Output Snapshot (Expected)

```
Task started on DefaultDispatcher-worker-1
```

✅ Then, cancellation happens — the coroutine never finishes.

---

### ⚠️ What Happens Inside Suspension Point on Cancellation
- When a coroutine is cancelled during a suspension point like delay(), yield(), or withContext, that suspending function throws a CancellationException.

```
suspend fun main() {
    val job = CoroutineScope(Dispatchers.Default).launch {
        try {
            println("🔁 Delaying...")
            delay(5000)
            println("✅ Completed") // ❌ This won't execute
        } catch (exception: CancellationException) {
            println("❗Caught CancellationException: $exception")
            performCleanup()
            throw exception // 🔁 Always rethrow to respect coroutine cancellation
        }
    }
    delay(1000)
    job.cancel()
}
```

```
suspend fun main() {
    val job = CoroutineScope(Dispatchers.Default).launch {
        try {
            println("🚀 Started long task")
            delay(5000)
            println("✅ Work done") // Won’t reach here if cancelled
        } catch (e: CancellationException) {
            println("❗Caught CancellationException")
            throw e // ✅ Always rethrow!
        } finally {
            println("🧼 Cleaning up after cancellation or completion")
        }
    }
    delay(1000)
    job.cancel()
}
```

### ⚠️ Why Rethrow?
- If you suppress CancellationException and don’t rethrow it, the parent coroutine may never know its child was cancelled. 
- That breaks the coroutine hierarchy and can cause silent bugs or leaked coroutines.

✅ Pattern to Follow

```
try {
    // Suspends
} catch (exception: CancellationException) {
    // Perform cleanup
    throw exception // ✅ Must rethrow
}
```

```
launch {
    try {
        // Suspends
    } catch (exception: CancellationException) {
        // Optional catch
        throw exception
    } finally {
        // Always runs
    }
}
```

### ⚠️ Non-Reachable Code After Suspension in catch

```
suspend fun main() {
    val job = CoroutineScope(Dispatchers.Default).launch {
        try {
            println("🚀 Started long task")
            delay(5000)
            println("✅ Work done") // ❌ Won’t run if cancelled
        } catch (e: CancellationException) {
            delay(100) // ← Already cancelled, so this throws again!

            // ❌ Non-reachable code — never runs
            performCleanUp()

            println("❗Caught CancellationException")
            throw e // ✅ Unnecessary here, exception already re-thrown
        }
    }

    delay(1000)
    job.cancel()
}
```

#### 🔥 What Happens Internally?

- Coroutine starts and hits delay(5000)
- After 1 sec, you call job.cancel() → This cancels the coroutine and throws CancellationException
- The catch block catches it. 
- Then inside the catch, you call: delay(100)
- But the coroutine is already cancelled. So:
- delay(100) throws another CancellationException
- Everything after ```delay(100)``` is skipped
- Code like performCleanUp() and ```throw exception``` never run
- It’s like throwing another exception while handling an exception, so the remaining code is lost.

#### ✅ How to Fix It?

- Use a non-suspending cleanup approach in catch, or move suspending cleanup into a finally block (which runs safely even during cancellation):

```
suspend fun main() {
    val job = CoroutineScope(Dispatchers.Default).launch {
        try {
            println("🚀 Started long task")
            delay(5000)
            println("✅ Work done")
        } catch (exception: CancellationException) {
            // 🔥 Don't call suspending functions here if already cancelled
            performImmediateCleanup() // ✅ Non-suspending
            throw exception
        } finally {
            // ✅ Safe place for suspending cleanup
            delay(100)
            println("🧼 Final cleanup after cancellation")
        }
    }

    delay(1000)
    job.cancel()
}
```

#### 🧠 Rule of Thumb
✅ Use catch for quick non-suspending recovery
✅ Use finally for cleanup, including suspending calls
❌ Don’t call suspending functions in catch after cancellation unless you're handling nested suspensions very carefully.


### Job Cancellation in Synchronous / Non-Suspending Code

- Kotlin coroutine cancellation is cooperative.
- If your coroutine never suspends, it will not respond to cancellation.
- So if you write code like this:
  ```
    val job = CoroutineScope(Dispatchers.Default).launch {
        println("Starting heavy loop")
        for(i in 0..1_00_000) {
            println("Count: $i")
            // ❌ No delay, yield, or withContext → won't cancel!
        }
        println("Finished")
    }
    delay(100)
    job.cancel() // ← Won’t stop the loop!
  ```
  Even after calling job.cancel(), the loop keeps running because there's no suspension point, and cancellation is not checked manually.


### 🔥 Why It Happens?

- Coroutine cancellation relies on checking cancellation status
- This check only happens at:
	1. Suspension points (delay, yield, withContext, etc.)
  				OR
	2. If you explicitly check using isActive, ensureActive()

### ✅ Solution: Cooperative Cancellation
- To make non-suspending code cancellable, you must manually check:

✅ 1. Using isActive

```
val job = CoroutineScope(Dispatchers.Default).launch {
        println("Starting heavy loop")

        for(i in 0..1_00_000) {
            if (!isActive) break // ✅ Check if coroutine is still active

            // Do work
            println("Count: $i")
        }

        println("After Loop")
    }

    delay(100)
    job.cancel()
}
```

### 🧵 Output Snapshot (Expected):

```
Starting heavy loop
Count-0
...
Count-8127
After Loop
```

- isActive is safe when you want to exit gracefully.

### Non-reachable code

```
val job = CoroutineScope(Dispatchers.Default).launch {
    println("Starting heavy loop")

    for (i in 0..1_00_000) {
        if (isActive) {
            println("Count: $i") // ✅ Still active, continue work
        } else {
            delay(100) // ❌ Throws CancellationException immediately
            println("Perform clean-up") // ❌ Never reached
            throw CancellationException()
        }
    }

    println("After Loop") // ❌ Not reached
}

delay(100)
job.cancel()
```

### 🧵 Output Snapshot (Expected):
```
Starting heavy loop
Count-0
...
Count-8127
```

### ⚠️ What’s the Problem?
- When job.cancel() is called, isActive becomes false, so execution enters the else block.
- delay(100) is a suspension point.
- The coroutine is already cancelled, delay() checks that and throws CancellationException immediately.
- So "Perform clean-up" is never executed, and the coroutine exits before proper cleanup. And so "After Loop" also is not executed.


## 🛡️ withContext(NonCancellable)
- withContext(NonCancellable) is used to temporarily disable cancellation inside a coroutine block.
- Even if the coroutine is cancelled, any suspending operations within this block will still execute safely.

```
withContext(NonCancellable) { // ✅ Handles cancellation — exception won't propagate to the Job
    delay(1000)               // ✅ Any suspension point here won't cancel the coroutine
    println("✅ Always runs after cancellation too")
}
```

### ✅ Fixed with withContext(NonCancellable):
```
val job = CoroutineScope(Dispatchers.Default).launch {
    println("Starting heavy loop")

    for (i in 0..1_00_000) {
        if (isActive) {
            println("Count: $i")
        } else {
            withContext(NonCancellable) {
                delay(100) // ✅ Runs safely
                println("Perform clean-up") // ✅ Executed
            }

            throw CancellationException() // ✅ Rethrow to exit cleanly
        }
    }

    println("After Loop") // ❌ Not reached
}

delay(100)
job.cancel()

```

- When job.cancel() is called, isActive becomes false, so execution enters the else block.
- delay(100) is a suspension point.
- The coroutine is already cancelled, delay() checks that and throws CancellationException immediately.
- This time, the code inside the else block is wrapped in withContext(NonCancellable). This special context suppresses cancellation inside the block.
- Now, delay(100) executes safely even though the coroutine was already cancelled. And below line of code executes reliably.
- It is the recommended pattern for cleanup involving suspension in a cancelled coroutine.

  
✅ 2. Using ensureActive()

```
val job = CoroutineScope(Dispatchers.Default).launch {
        println("Starting heavy loop")

        for(i in 0..1_00_000) {
            ensureActive() // ✅ Will throw CancellationException if cancelled

            // Do work
            println("Count: $i")
        }

        println("After Loop")
    }

    delay(100)
    job.cancel()
}
```

### 🧵 Output Snapshot (Expected):

```
Starting heavy loop
Count-0
...
Count-8127
```

- ensureActive() is more aggressive: it throws CancellationException immediately.

### 🧠 Behavior
- When ensureActive() is used inside a coroutine, it immediately checks whether the coroutine has been cancelled. If it has, ensureActive() throws a CancellationException at that exact line. As a result, any code written after the ensureActive() call becomes unreachable and does not execute.
- So "After Loop" is not printed here.


### ✅ with try-catch + ensureActive()

```
val job = CoroutineScope(Dispatchers.Default).launch {
    println("Starting heavy loop")

    for (i in 0..1_00_000) {
        try {
            ensureActive() // Throws CancellationException if cancelled
        } catch (exception: CancellationException) {
            println("Perform clean-up") // ✅ Runs once, inside loop
            throw exception // ✅ Re-throws to exit the coroutine
        }

        println("Count: $i") // Runs only until cancelled
    }

    println("After Loop") // ❌ Skipped, coroutine exits early
}

delay(100)
job.cancel()
```

### 🧵 Output Snapshot (Expected)

```
Starting heavy loop  
Count: 0  
Count: 1  
...  
Count: 8127  
Perform clean-up
```

### 🔍 What’s Happening Internally?
- When job.cancel() is called, it immediately sends a cancellation signal to the coroutine.
- As the loop continues, it eventually hits ensureActive(), which checks for cancellation and throws a CancellationException.
- The try-catch block catches this exception and runs the catch block.
- Inside the catch, the exception is rethrown, and due to rethrowing the CancellationException from the catch block, the coroutine gets immediately cancelled.

### ✅ Best Practice Reminder
- Always rethrow CancellationException after cleanup to maintain structured concurrency and prevent orphaned coroutines.


### ⚠️ Suspension after Cancellation (inside catch block)

```
val job = CoroutineScope(Dispatchers.Default).launch {
    println("Starting heavy loop")

    for (i in 0..1_00_000) {
        try {
            ensureActive() // Throws CancellationException if cancelled
        } catch (exception: CancellationException) {
            println("Perform clean-up") // ✅ Runs once after cancellation

            delay(10) // ❗Suspending call after catching CancellationException

            throw exception // ✅ Re-throw to respect cancellation
        }

        println("Count: $i") // ❌ Not executed after cancellation
    }

    println("After Loop") // ❌ Not reached
}

delay(100)
job.cancel()
```

### 🔍 What’s Happening Internally?
- When job.cancel() is called, it immediately sends a cancellation signal to the coroutine.
- As the loop continues, it eventually hits ensureActive(), which checks the coroutine's cancellation status and throws a CancellationException.
- The try-catch block catches this exception and executes the catch block.
- Inside the catch, the suspension point delay(10) runs without throwing another CancellationException.
- That’s because the first suspension point that detects cancellation (in this case, ensureActive()) throws the exception.
- If that exception is caught and not rethrown immediately, later suspension points do not rethrow unless cancellation is re-triggered.
- Finally, the exception is manually rethrown inside the catch.
- Due to this rethrow of CancellationException, the coroutine exits immediately, and no further code (like "Count: $i" or "After Loop") executes.

### 🔥 Key Insight:
- Once a coroutine detects cancellation and throws a CancellationException,
- If that exception is handled, the coroutine resumes "normally", and later suspension points won’t throw again unless you retrigger cancellation or manually rethrow the original exception.


### 🔥 Goldan rule:

- When job.cancel() is called, it signals the coroutine to cancel by propagating a CancellationException.
- However, this exception is not thrown immediately — instead, it is thrown only when the coroutine reaches a suspension point such as delay(), withContext, yield(), or explicitly via ensureActive().
- Alternatively, cancellation can be detected non-suspensively using isActive, though it won’t throw — it only returns false.
- Importantly, CancellationException is not caught by declarative tools like CoroutineExceptionHandler because it's considered a controlled cancellation, not an unhandled failure.
- To handle it, you must use imperative try-catch blocks.
- But catching the exception does not automatically stop the coroutine — if you catch and suppress it, the coroutine continues executing.
- Therefore, to ensure proper cancellation after cleanup or custom logic, it is crucial to rethrow the CancellationException manually.
