# 🎯 Kotlin Coroutines Android Development

---

# 🎯 Coroutine
- In Kotlin, a coroutine is a concurrency design pattern that simplifies asynchronous code, making it more readable and easier to manage. Think of them as lightweight threads that can suspend and resume execution without blocking the underlying thread. This allows you to perform multiple tasks concurrently without the overhead of traditional threading. 
- Here's a breakdown of key aspects:
  1. Asynchronous Programming Made Simple: 
    - Coroutines allow you to write asynchronous code (like fetching data from a network or accessing a database) in a sequential, synchronous style, which makes it much easier to reason about.
    - They help to manage long-running tasks that would otherwise block the main thread and make your app unresponsive. 
  2. Suspending and Resuming: 
    - A crucial concept is the suspend function, marked with the suspend keyword.
    - When a coroutine encounters a suspend function, its execution can be suspended (paused) without blocking the thread it's running on.
    - The coroutine can then be resumed later when the suspending function has completed its work. 
  3. Lightweight and Efficient: 
    - Coroutines are significantly less resource-intensive than traditional threads.
    - We can run many coroutines on a single thread, enabling efficient concurrency without the high cost of creating and managing a large number of threads. 
  4. Structured Concurrency: 
    - Coroutines in Kotlin follow structured concurrency.
    - This means coroutines are launched within a specific CoroutineScope, which defines their lifecycle.
    - Structured concurrency helps to manage coroutines effectively, ensuring that they are canceled when they are no longer needed, preventing resource leaks and improving error handling. 
  5. Builders: 
    - Kotlin provides functions like launch and async to launch new coroutines.
    - launch is used for "fire and forget" tasks, where you don't need a result back.
    - async is used when you expect a result from the coroutine, returning a Deferred object that allows you to get the result using await().

---

### ❓ What is a CoroutineScope?
- A CoroutineScope defines where and how long your coroutines will run.
- Think of it as a container for coroutines — it keeps track of them and allows cancellation all at once.
- Each scope comes with:
	1. A Job (manages the lifecycle of coroutine work)
	2. A CoroutineContext (defines how coroutines behave; it's a container for Job, Dispatcher, etc.)
	3. A Dispatcher (decides which thread the coroutine runs on)

### ❓ Why Scopes Are Required in Android?
1. Structured concurrency
2. Lifecycle management
3. Exception handling
4. Cancellation
5. Memory leak prevention

### 🟠 Anatomy of a CoroutineScope:

```
interface CoroutineScope {
	val coroutineContext: CoroutineContext
}
```

### 1: 🟠 Ordinary Scope

```
val scope = CoroutineScope(Job())
val job = scope.launch {
	// Coroutine work
}
```


### 2: 🟠 Supervisor Scope

```
val supervisorScope = CoroutineScope(SupervisorJob())
val supervisorJob = supervisorScope.launch {
	// Independent child coroutines
}
```

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

### 🟠 Coroutine Scope Reuse:

- You can reuse a CoroutineScope to launch multiple coroutines — as long as it’s active (not cancelled).
- This is safe, efficient, and encouraged:

```
val scope = CoroutineScope(Job() + Dispatchers.Default)

scope.launch { println("🔹 Task-1") }
scope.launch { println("🔸 Task-2") } // ✅ Reuse is fine

```

### ❓ What Happens After scope.cancel()?

```
val scope = CoroutineScope(Job())
scope.cancel()

scope.launch {
    println("❌ This will never run")
}
```

### 🟠 Internally
- The coroutine is immediately cancelled
- The scope becomes inactive
- Any new coroutine won’t start and doesn’t start executing any logic 
- A CancellationException is thrown internally
- The app won’t crash

### 🟠 Correct Reuse Pattern

- To reuse after cancellation, create a new scope:

```
var scope = CoroutineScope(Job() + Dispatchers.Default)

scope.launch { println("Task-A") }

scope.cancel() // Scope is now dead

scope = CoroutineScope(Job() + Dispatchers.Default)
scope.launch { println("Task-B") } // ✅ Works

```

## 🎯 CoroutineScope .launch & .async

### ❓ What is CoroutineScope.launch {}?
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

### ❓ What is CoroutineScope.async {}?
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
	println("Calculating...")
    77
}

val result = deferred.await()
println("Result: $result")
```

### 🟠 Coroutine Builders – launch {} and async {}
- CoroutineScope.launch {} and CoroutineScope.async {} are coroutine builders used to spawn child coroutines that inherit the job and context from the surrounding scope or coroutine.

---

## 🎯 Jobs

### ❓ What is an Ordinary Job?
- An Ordinary Job in Kotlin Coroutines is created using Job() without any additional supervision logic. It represents a coroutine's lifecycle and propagates cancellation down and up the coroutine hierarchy.
- If any child coroutine fails, it cancels the parent job, which then cancels all other sibling coroutines. This is the default behavior of structured concurrency.
- If any child coroutine fails, the entire job hierarchy gets cancelled.
- This is useful when you want fail-fast behavior — like in structured concurrency.

```
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

**Output:**
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

### ❓ What Happens Internally?
- All tasks are launched in the same CoroutineScope, which uses an ordinary Job() as its context.
- Task-2 throws a RuntimeException after 1 second.
- Because the scope uses an ordinary Job, the failure:
  - Cancels job1.
  - Propagates upward and cancels the entire CoroutineScope.
  - Consequently, job2 is also cancelled—even though its child tasks (Task-3, Task-4) are unrelated and running fine.

### 💡 Visual Hierarchy

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

### ❓ What is a SupervisorJob?

- A SupervisorJob is a special kind of Job in Kotlin Coroutines where the failure of a child does not cancel its siblings or the parent scope.
- It provides exception isolation, which means that one coroutine's failure does not bring down the whole scope.
- If any child coroutine fails, only that child is cancelled — the rest continue unaffected.
- This is ideal when you want resilience and isolation, not fail-fast behavior.

```
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

### ❓ What Happens Internally?
- All tasks are launched in the same CoroutineScope, which uses a SupervisorJob().
- Task- throws a RuntimeException after 1 second.
- Since the scope uses a SupervisorJob:
	- Only Task-2 fails and is cancelled.
  	- job1 finishes because Task-1 is unaffected.
  	- job2 and its children (Task-3, Task-4) continue to run normally.

### 💡 Visual Hierarchy

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

---

## 🎯 Dispatchers.Main

### ❓ What is Dispatchers.Main?
- Dispatchers.Main is a coroutine dispatcher designed to run coroutines on the main (UI) thread of Android.
- It is used for:
	- UI updates
	- ViewModel coroutine launching (viewModelScope)
	- Observing LiveData / Flows
	- Safe interaction with Android Views

### ⚠️ Main Dispatcher – Concurrent but Not Parallel
- Coroutines on Dispatchers.Main can start at the same time (concurrently), but they execute one at a time (not in parallel).
- That’s because:
  1. The Main thread is single-threaded
  2. Only one coroutine can run at any given moment
  3. Others are suspended and resumed in order
 
### 🟠 Synchronous (Sequential) behaviour

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

**Output:**

```
Task-1: Count-0
...
Task-1: Count-999
Task-2: Count-0
...
Task-2: Count-999
```

### 🟠 Behavior
- viewModelScope uses Dispatchers.Main by default.
- Android’s Main thread is single-threaded, meaning it cannot run multiple tasks in parallel.
- Both launch {} blocks are submitted concurrently, but they are scheduled on the same thread.
- The first coroutine starts and runs a tight repeat(1000) loop.
- Since there is no suspension point like delay(), yield(), or withContext() in the first coroutine, it never gives up control of the thread.
- As a result, the second coroutine must wait for the first to complete.
- This leads to sequential execution, even though launch syntactically looks concurrent.

### 🟠 Core Principle
- On Dispatchers.Main, only one coroutine executes at a time because the main thread is single-threaded.
- Without any suspension points (delay, yield, withContext), a coroutine blocks the main thread completely until it finishes.
- Therefore, coroutines run sequentially, even if launched concurrently.
- ❓ What If It Hits a Suspension Point?
    - When a coroutine reaches a suspension point like delay(), yield(), or withContext(...):
    - It suspends its execution, releases the main thread, and goes into a waiting state.
    - This gives other pending coroutines a chance to run, enabling concurrent (interleaved) behavior.
    - Execution resumes when the suspension completes (e.g., after delay time or background task).
    - At this point, coroutine behavior switches from synchronous (sequential) to concurrent (interleaved) — but still not parallel.

### 🟠 Concurrent (Interleaved) behaviour

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

**Output:**

```
Task-1: Count-0
Task-2: Count-0
Task-1: Count-1
Task-2: Count-1
Task-1: Count-2
Task-2: Count-2
...
```

### 🟠 Behavior: Concurrent (Interleaved)
- viewModelScope.launch {} creates two coroutines on the Main dispatcher.
- Each coroutine executes repeat(1000) with a delay(100) after every print.
- delay() is a suspension point that:
- Releases the main thread when hit.
- Suspends the current coroutine without blocking the UI.
- Allows the other coroutine to resume and run.

### 🟠 Execution Cycle

```
Main Thread
↓
Task-1: Count-0
delay(100) → suspends → allows Task-2
Task-2: Count-0
delay(100) → suspends → back to Task-1
...
```

### 🟠 Important Reminder
 - Coroutines are not parallel on Dispatchers.Main — they are just interleaved using suspension.
 - No two coroutines run at the same moment, but they take turns efficiently by suspending and resuming.

## 🎯 Dispatchers.Default

### ❓ What is Dispatchers.Default?

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


### 🟠 Parallel (Asynchronous) behaviour:

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

**Output:**

```
Task-1: Count-0
Task-2: Count-0
Task-1: Count-1
Task-2: Count-1
Task-1: Count-2
Task-2: Count-2
...
```

## 🎯 Dispatchers.IO

### ❓ What is Dispatchers.IO?
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
- Designed for blocking I/O tasks — it's okay if a coroutine blocks a thread for a while.
- Can grow its thread pool dynamically, up to 64 threads (default upper limit).
- Coroutines run concurrently, even when sharing threads — by suspending at I/O points.


### 🟠 Parallel (Asynchronous) behaviour:

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

**Output:**

```
Task-1: Count-0
Task-2: Count-0
Task-1: Count-1
Task-2: Count-1
Task-1: Count-2
Task-2: Count-2
...
```

## 🟠 Coroutine Thread Switching
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


### 🟠 Thread Switch After Suspension:

```
fun main() = runBlocking {
    launch(Dispatchers.Default) {
        println("Started on: ${Thread.currentThread().name}")
        delay(500)
        println("Resumed on: ${Thread.currentThread().name}")
    }
}
```

**Output:**

```
Started on: DefaultDispatcher-worker-1
Resumed on: DefaultDispatcher-worker-3
```

## 🎯 Coroutine Cancellation

### 🟠 Job Cancellation at Suspension Points:
- Cancellation in coroutines is cooperative.
- The coroutine must reach a suspension point to respond to cancellation.
- A suspension point is a place where the coroutine pauses and checks for cancellation. 
- Most common:
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

**Output:**

```
Task started on DefaultDispatcher-worker-1
```

### ❓ What Happens Inside Suspension Point on Cancellation
- When a coroutine is cancelled during a suspension point like delay(), yield(), or withContext, that suspending function implicitly throws a CancellationException.

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

### ❓ Why Rethrow?
- If you suppress CancellationException and don’t explicitly rethrow it, the parent coroutine may never know its child was cancelled. 
- That breaks the coroutine hierarchy and can cause silent bugs or leaked coroutines.

### 🟠 Pattern to Follow:

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

### 🟠 Non-Reachable Code After Suspension in catch:

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

### ❓ What Happens Internally?
- Coroutine starts and hits delay(5000)
- After 1 sec, you call job.cancel() → This cancels the coroutine and implicitly throws CancellationException
- The catch block catches it explicitly. 
- Then inside the catch, delay(100) is called.
- But the coroutine is already cancelled.
- So: delay(100) implicitly throws another CancellationException
- Everything after ```delay(100)``` is skipped
- Code like performCleanUp() and ```throw exception``` never run
- It’s like throwing another exception while handling an exception, so the remaining code is lost.

### ❓ How to Fix It?
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

### 🟠 Rule of Thumb:
- ✅ Use catch for quick non-suspending recovery
- ✅ Use finally for cleanup, including suspending calls
- ❌ Don’t call suspending functions in catch after cancellation unless you're handling nested suspensions very carefully.

### 🟠 Job Cancellation in Synchronous / Non-Suspending Code
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

### ❓ Why It Happens?
- Coroutine cancellation relies on checking the cancellation status
- This check only happens at:
	1. Implicitly: By Suspension points (delay, yield, withContext, etc.)
  				OR
	2. Explicitly: By using isActive, ensureActive()

### 🟠 Solution: Cooperative Cancellation
- To make non-suspending code cancellable, you must manually check:

### 🟠 1. Using isActive

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

**Output:**

```
Starting heavy loop
Count-0
...
Count-8127
After Loop
```

- isActive is safe when you want to exit gracefully.

### 🟠 Non-reachable code

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

**Output:**

```
Starting heavy loop
Count-0
...
Count-8127
```

### ❓ What’s the Problem?
- When job.cancel() is called, isActive becomes false, so execution enters the else block.
- delay(100) is a suspension point.
- The coroutine is already cancelled, delay() checks that and implicitly throws CancellationException immediately.
- So "Perform clean-up" is never executed, and the coroutine exits before proper cleanup. And so "After Loop" also is not executed.


### 🟠 withContext(NonCancellable)
- withContext(NonCancellable) is used to temporarily disable cancellation inside a coroutine block.
- Even if the coroutine is cancelled, any suspending operations within this block will still execute safely.

```
withContext(NonCancellable) { // ✅ Handles cancellation — exception won't propagate to the Job
    delay(1000)               // ✅ Any suspension point here won't cancel the coroutine
    println("✅ Always runs after cancellation too")
}
```

### 🟠 Fixed with withContext(NonCancellable):

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
- The coroutine is already cancelled, delay() checks that and implicitly throws CancellationException immediately.
- This time, the code inside the else block is wrapped in withContext(NonCancellable). This special context suppresses cancellation inside the block.
- Now, delay(100) executes safely even though the coroutine was already cancelled. And below line of code executes reliably.
- It is the recommended pattern for cleanup involving suspension in a cancelled coroutine.

  
### 🟠 2. Using ensureActive()

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

**Output:**

```
Starting heavy loop
Count-0
...
Count-8127
```

- ensureActive() is more aggressive: it throws CancellationException immediately.
- When ensureActive() is used inside a coroutine, it immediately checks whether the coroutine has been cancelled. If it has, ensureActive() implicitly throws a CancellationException at that exact line. As a result, any code written after the ensureActive() call becomes unreachable and does not execute.
- So "After Loop" is not printed here.


### 🟠 with try-catch + ensureActive()

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

**Output:**

```
Starting heavy loop  
Count: 0  
Count: 1  
...  
Count: 8127  
Perform clean-up
```

### ❓ What’s Happening Internally?
- When job.cancel() is called, it immediately sends a cancellation signal to the coroutine.
- As the loop continues, it eventually hits ensureActive(), which checks for cancellation and implicitly throws a CancellationException.
- The try-catch block catches this exception and runs the catch block.
- Inside the catch, the exception is explicitly rethrown, and due to rethrowing the CancellationException from the catch block, the coroutine gets immediately cancelled.

### 🟠 Best Practice Reminder
- Always explicitly rethrow CancellationException after cleanup to maintain structured concurrency and prevent orphaned coroutines.

### 🟠 Suspension after Cancellation (inside catch block)

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

### ❓ What’s Happening Internally?
- When job.cancel() is called, it immediately sends a cancellation signal to the coroutine.
- As the loop continues, it eventually hits ensureActive(), which checks the coroutine's cancellation status and implicitly throws a CancellationException.
- The try-catch block catches this exception explicitly and executes the catch block.
- Inside the catch, the suspension point delay(10) runs without throwing another CancellationException.
- That’s because the first suspension point that detects cancellation (in this case, ensureActive()) throws the exception.
- If that exception is caught and not rethrown immediately, later suspension points do not rethrow unless cancellation is re-triggered.
- Finally, the exception is explicitly rethrown inside the catch.
- Due to this explicit rethrow of CancellationException, the coroutine exits immediately, and no further code (like "Count: $i" or "After Loop") executes.

### 🟠 Key Insight:
- Once a coroutine detects cancellation and throws a CancellationException,
- If that exception is handled, the coroutine resumes "normally", and later suspension points won’t throw again unless you retrigger cancellation or manually rethrow the original exception.

### 🟠 Rule:
- When job.cancel() is called, it signals the coroutine to cancel by propagating a CancellationException.
- However, this exception is not thrown immediately — instead, it is thrown only when the coroutine reaches a suspension point such as delay(), withContext, yield(), or explicitly via ensureActive().
- Alternatively, cancellation can be detected non-suspensively using isActive, though it won’t throw — it only returns false.
- Importantly, CancellationException is not caught by declarative tools like CoroutineExceptionHandler because it's considered a controlled cancellation, not an unhandled failure.
- To handle it, you must use imperative try-catch blocks.
- But catching the exception does not automatically stop the coroutine — if you catch and suppress it, the coroutine continues executing.
- Therefore, to ensure proper cancellation after cleanup or custom logic, it is crucial to rethrow the CancellationException manually.

---

## 🎯 Exception Handling in CoroutineScope
- Both launch {} and CoroutineScope.launch {} start new coroutines. However, where an exception is caught depends entirely on the presence and placement of a CoroutineExceptionHandler in the coroutine context.

### 🟠 Exception Propagation in launch {}
- The exception thrown in a coroutine that is not caught imperatively in a try-catch does propagate the exception upward to the scope. If the coroutine's parent scope or the coroutine itself has a CoroutineExceptionHandler, it will catch the exception declaratively only in CoroutineExceptionHandler.
- Uncatched exceptions always propagate upward till the parent scope, if the coroutine itself does not have a CoroutineExceptionHandler. And this uncaught exceptions is only caught by declaratively only in CoroutineExceptionHandler.

### 🟠 Declarative Exception Handling with CoroutineExceptionHandler

### 🟠 1. Coroutine-level Exception Handler
- If a CoroutineExceptionHandler is passed directly to the coroutine via launch(handler), then exceptions are caught locally in that coroutine:

```
val scope = CoroutineScope(Job())

    val job = scope.launch(CoroutineExceptionHandler{ _, throwable -> println("CoroutineExceptionHandler: $throwable") }) {
        println("Task-1 stared.")
        delay(1000)
        throw RuntimeException()
        println("Task-1 finished.")
    }

    job.join()
    scope.cancel()
```

**Output:**

```
Task-1 started.
CoroutineExceptionHandler: java.lang.RuntimeException
```

- The exception is caught by the handler attached directly to the coroutine.

### 🟠 2. Scope-level Exception Handler
- If no handler is passed to the coroutine, but the CoroutineScope itself was created with a CoroutineExceptionHandler, then exceptions are caught at the scope level:

```
val scope = CoroutineScope(Job() + CoroutineExceptionHandler{ _, throwable -> println("CoroutineExceptionHandler: $throwable") })

val job = scope.launch {
    println("Task-1 stared.")
    delay(1000)
    throw RuntimeException()
    println("Task-1 finished.")
}

job.join()
scope.cancel()
```

**Output:**
```
Task-1 started.
CoroutineExceptionHandler: java.lang.RuntimeException
```

- The exception bubbles up to the parent scope and is caught by the scope's handler.

### 🟠 3. No Exception Handler — App Crashes
- If neither the coroutine nor the parent scope has a CoroutineExceptionHandler, then the exception goes uncaught and will crash the app or terminate the thread:

```
val scope = CoroutineScope(Job())

val job = scope.launch {
    println("Task-1 started.")
    delay(1000)
    throw RuntimeException()
    println("Task-1 finished.")
}

job.join()
scope.cancel()
```

**Output:**

```
Task-1 started.
Exception in thread "DefaultDispatcher-worker-1" java.lang.RuntimeException
```

- No handler exists to catch the exception, so the app/thread crashes.


### 🟠 Imperative Exception Handling Inside Coroutine Blocks
- In Kotlin Coroutines, exceptions thrown inside a coroutine or child coroutine can be caught imperatively using a try-catch block placed within the coroutine body.
- If an exception is caught imperatively, it will not propagate upward to the parent scope, and CoroutineExceptionHandler will not be triggered — unless the exception is re-thrown inside the catch block.

### 🟠 Handling Exception in a Single Coroutine:

```
val scope = CoroutineScope(Job())

val job = scope.launch {
    try {
        println("Task-1 started.")
        delay(1000)
        throw RuntimeException()
        println("Task-1 finished.")
    } catch (e: Exception) {
        println("Caught Exception: $e")
    }
}

job.join()
scope.cancel()
```

**Output:**

```
Task-1 started.
Caught Exception: java.lang.RuntimeException
```

- The exception is thrown inside the coroutine block and is caught by the internal try-catch.
- Hence, it does not propagate upward, and no crash or CoroutineExceptionHandler is invoked.

### 🟠 Handling Exception in a Nested Child Coroutine:

```
val scope = CoroutineScope(Job())

val job = scope.launch {
    launch {
        try {
            println("Task-1 started.")
            delay(1000)
            throw RuntimeException()
            println("Task-1 finished.")
        } catch (e: Exception) {
            println("Caught Exception: $e")
        }
    }
}

job.join()
scope.cancel()
```

Output:

```
Task-1 started.
Caught Exception: java.lang.RuntimeException
```

- The inner coroutine throws an exception, but it's wrapped with an internal try-catch.
- So, even though it’s a child coroutine, the exception is contained locally and does not propagate to the outer job or scope.
- CoroutineExceptionHandler is not triggered because the exception was already handled.
- If a coroutine or any of its children handle exceptions imperatively using try-catch, those exceptions won’t propagate upward and will not be caught declaratively by a CoroutineExceptionHandler — unless the catch block rethrows the exception.

### 🟠 try-catch Around launch {} Won’t Catch Exceptions
- Wrapping launch {} or CoroutineScope.launch {} inside a try-catch block does not catch coroutine exceptions.
- Exceptions from coroutines are caught declaratively using CoroutineExceptionHandler, either:
	- At the coroutine level (launch(handler) {})
  						OR
	- At the scope level (CoroutineScope(handler))
- If required to handle an exception imperatively, it needs to place the try-catch inside the coroutine block.
  
### 🟠 Scope has a CoroutineExceptionHandler:

```
val scope = CoroutineScope(
    Job() + CoroutineExceptionHandler { _, throwable ->
        println("CoroutineExceptionHandler: $throwable")
    }
)

try {
    val job = scope.launch {
        println("Task-1 started.")
        delay(1000)
        throw RuntimeException()
        println("Task-1 finished.")
    }

    job.join()
    scope.cancel()
} catch (e: Exception) {
    println("Caught Exception: $e") // ❌ Won’t execute
}
```

**Output:**

```
Task-1 started.
CoroutineExceptionHandler: java.lang.RuntimeException
```

### 🟠 Handler at the Coroutine Level:

```
val scope = CoroutineScope(Job())

val job = scope.launch(
    CoroutineExceptionHandler { _, throwable ->
        println("CoroutineExceptionHandler: $throwable")
    }
) {
    try {
        launch {
            println("Task-1 started.")
            delay(1000)
            throw RuntimeException()
            println("Task-1 finished.")
        }
    } catch (e: Exception) {
        println("Caught Exception: $e") // ❌ Won’t execute
    }
}

job.join()
scope.cancel()
```

**Output:**

```
Task-1 started.
CoroutineExceptionHandler: java.lang.RuntimeException
```

- A CoroutineExceptionHandler is invoked only for uncaught exceptions in root coroutines that are launched directly by a scope without a parent coroutine and it ignored in child coroutines because the exception propagates up to the parent context.

## 🎯 Coroutine Job Lifecycle — Completion
- A coroutine always completes implicitly — either normally, with an exception, or via cancellation.
- No need to cancel explicitly.
- Once completed (in any form), the Job is no longer active.
- Use invokeOnCompletion { throwable -> ... } to monitor any of the outcomes in a unified way.
- A coroutine always completes implicitly — either:
	- ✅ Normally (successful execution)
	- ❗ With an exception
	- 🚫 Via cancellation
- ❌ No need to cancel explicitly, unless cancellation is part of the logic. Coroutines do not require explicit cancellation if you don’t need to interrupt them.
- ✅ Once completed (Normally, With an exception or Via cancellation), the Job is no longer active.
- 📌 Use invokeOnCompletion { throwable -> ... } to observe the final result — success, failure, or cancellation — in a unified way.

### 🟠 1. Completed normally:
   
```
val job = CoroutineScope(Dispatchers.Default).launch {
    println("Task started")
    delay(500)
    println("Task finished")
}

job.invokeOnCompletion { throwable ->
    println("Job Completed." + (throwable?.let { " Exception: $it" } ?: " Successfully ✅"))
}
```

**Output:**

```
Task started
Task finished
Job Completed. Successfully ✅
```

### 🟠 2. Completed with exception:

```
val job = CoroutineScope(Dispatchers.Default + CoroutineExceptionHandler { _, throwable -> println("Caught by CoroutineExceptionHandler: $throwable") }).launch {
    println("Task started")
    delay(500)
    throw RuntimeException()
}

job.invokeOnCompletion { throwable ->
    println("Job Completed." + (throwable?.let { " Exception: $it" } ?: " Successfully ✅"))
}
```

**Output:**

```
Task started
Job Completed. Exception: java.lang.RuntimeException
```

### 3. 🟠 Cancel explicitly:

```
val job = CoroutineScope(Dispatchers.Default).launch {
    println("Task running...")
    delay(2000)
}

delay(500)
job.cancel() // 🚫 Explicit cancellation

job.invokeOnCompletion { throwable ->
    println("Job Completed." + (throwable?.let { " Exception: $it" } ?: " Successfully ✅"))
}
```

**Output:**

```
Task running...
Job Completed. Exception: kotlinx.coroutines.JobCancellationException'
```

## 🎯 Coroutine Completion: Imperative vs Declarative Exception Handling

### 🟠 1. Imperative Handling (try-catch inside coroutine)
- If an exception is caught inside the coroutine using try-catch, it is handled locally.
- The exception does not propagate upward, so CoroutineExceptionHandler is not triggered.
- The coroutine completes successfully from the job’s perspective.
- The Job's invokeOnCompletion { throwable -> ... } gets called with throwable == null.

```
val job = CoroutineScope(Dispatchers.Default).launch {
    try {
        println("Task started")
        delay(1000)
        throw RuntimeException("Something went wrong")
    } catch (e: Exception) {
        println("Caught imperatively: $e")
    }
}

job.invokeOnCompletion {
    println("Job completed with throwable: $it") // 👉 it == null
}
```

**Output:**

```
Task started
Caught imperatively: java.lang.RuntimeException: Something went wrong
Job completed with throwable: null
```

### 🟠 2. Declarative Handling (Using CoroutineExceptionHandler)
- If the exception is not caught inside the coroutine, it propagates upward.
- If a CoroutineExceptionHandler is present (in coroutine or its parent scope), it will catch the exception declaratively.
- The coroutine's Job completes with the throwable.
- The invokeOnCompletion { throwable -> ... } gets called with throwable != null.

```
val job = CoroutineScope(Dispatchers.Default + CoroutineExceptionHandler { _, throwable -> println("Caught declaratively: $throwable") }).launch {
    println("Task started")
    delay(1000)
    throw RuntimeException("Something went wrong")
}

job.invokeOnCompletion {
    println("Job completed with throwable: $it") // 👉 it != null
}
```

**Output:**

```
Task started
Caught declaratively: java.lang.RuntimeException: Something went wrong
Job completed with throwable: java.lang.RuntimeException: Something went wrong
```

## 🎯 Why CoroutineExceptionHandler Is Not Invoked on Cancellation
1. CoroutineExceptionHandler is only triggered for uncaught exceptions in root coroutines.
2. It is not triggered for CancellationException, because cancellation is not considered an error.

```
  val scope = CoroutineScope(Job() + CoroutineExceptionHandler{ _, throwable -> println("CoroutineExceptionHandler: $throwable") })

    val job = scope.launch {
        repeat(1_00_000) {
            println("Count: $it")
            delay(10)
        }
    }

    job.invokeOnCompletion {
        println("Job Completed: $it")
    }

    delay(100)
    job.run {
        cancel()
        join()
    }
```

**Output:**

```
Count: 0
Count: 1
Count: 2
Count: 3
Count: 4
Count: 5
Count: 6
Job Completed: kotlinx.coroutines.JobCancellationException: StandaloneCoroutine was cancelled; job=StandaloneCoroutine{Cancelled}@7d9de84e
```

## 🎯 Coroutine Scope Cancellation
1. When scope.cancel() is called, all jobs launched within the scope receive a cancellation signal.
2. When a coroutine reaches a suspension point (delay, yield, withContext, ensureActive, etc.), it throws a CancellationException if the job was canceled.
3. If this exception is not caught, it propagates upward, and the job is canceled. If caught job and not rethrown explicitly, the coroutine may continue running until another suspension point or natural completion.
4. If a coroutine uses withContext(NonCancellable), then that block ignores cancellation — it continues to run even after cancellation is requested, and must finish on its own.
5. The coroutine scope remains alive as long as any job within it is running. If jobs hold strong references (like to an Activity, Fragment, or ViewModel), it may prevent GC, causing memory leaks.
6. Once a job completes (normally or via cancellation), invokeOnCompletion is called. If the job was canceled, the exception is passed; otherwise, it's null.

```
val scope = CoroutineScope(Job() + CoroutineExceptionHandler{ _, throwable -> println("CoroutineExceptionHandler: $throwable") })

    val job1 = scope.launch {
        println("Task-1 stared.")
        delay(150)
        println("Task-1 finished.") // ❌ Not reached
    }

    val job2 = scope.launch {
        println("Task-2 stared.")
        delay(200)
        println("Task-2 finished.") // ❌ Not reached
    }

    job1.invokeOnCompletion {
        println("Job-1 Completed: $it")
    }

    job2.invokeOnCompletion {
        println("Job-2 Completed: $it")
    }

    delay(100)
    scope.cancel()  // 🔥 Cancels both jobs
    joinAll(job1, job2)
```

**Output:**

```
Task-1 stared.
Task-2 stared.
Job-1 Completed: kotlinx.coroutines.JobCancellationException: Job was cancelled; job=JobImpl{Cancelling}@39fe891
Job-2 Completed: kotlinx.coroutines.JobCancellationException: Job was cancelled; job=JobImpl{Cancelling}@39fe891
```

### 🟠 Coroutine Scope Non-Cancellable Behaviors
- These are scenarios where coroutines do not immediately stop despite scope.cancel() or job.cancel() being called. They ignore or delay cancellation, potentially causing memory leaks or undesired continuation.

### 🟠 1. Synchronous Execution Without Suspension Points:

```
    val scope = CoroutineScope(Job() + CoroutineExceptionHandler{ _, throwable -> println("CoroutineExceptionHandler: $throwable") })

    val job = scope.launch {
        repeat(1_00_000) {
            println("Count: $it")
        }
    }

    job.invokeOnCompletion {
        println("Job Completed: $it")
    }

    delay(10)
    scope.cancel()
    job.join()
```

**Output:**

```
Count: 0
...
Count: 99999
Job Completed: kotlinx.coroutines.JobCancellationException: Job was cancelled; job=JobImpl{Cancelling}@5dd6783d
```

### 🟠 2. Try-Catch Suppressing CancellationException:

```
    val scope = CoroutineScope(Job() + CoroutineExceptionHandler{ _, throwable -> println("CoroutineExceptionHandler: $throwable") })

    val job = scope.launch {
        repeat(1_00_000) {
            try {
                println("Count: $it")
                delay(1)
            } catch (exception: CancellationException) {
                println("Perform clean-up")
            }
        }
    }

    job.invokeOnCompletion {
        println("Job Completed: $it")
    }

    delay(10)
    scope.cancel()
    job.join()
```

**Output:**

```
Count: 0
Count: 1
Count: 2
Count: 3
Perform clean-up
Count: 4
Perform clean-up
...
Count: 99999
Perform clean-up
Job Completed: kotlinx.coroutines.JobCancellationException: Job was cancelled; job=JobImpl{Cancelling}@5dd6783d
```

### 🟠 3. Code Executed Under withContext(NonCancellable):

```
    val scope = CoroutineScope(Job() + CoroutineExceptionHandler{ _, throwable -> println("CoroutineExceptionHandler: $throwable") })

    val job = scope.launch {
        withContext(NonCancellable) {
            repeat(1_00_000) {
                println("Count: $it")
                delay(10)
            }
        }
    }

    job.invokeOnCompletion {
        println("Job Completed: $it")
    }

    delay(100)
    scope.cancel()
    job.join()
```

**Output:**

```
Count: 0
...
Count: 99999
Job Completed: kotlinx.coroutines.JobCancellationException: Job was cancelled; job=JobImpl{Cancelling}@5dd6783d
```

## 🎯 Why There's No Need to Call Job.cancel() Explicitly (Unless You Intend to Cancel It)
- Coroutines Complete Automatically. Every coroutine completes implicitly when:
	- It reaches the end of execution (normal completion)
	- It throws an exception (exceptional completion)
	- It is cancelled (cancellation)
- So, once the work is done, there is nothing to cancel. Once a coroutine finishes execution, whether by:
	- Normal completion (success),
	- Cancellation (job.cancel() / scope.cancel()), or
	- Failure (exception thrown),
- The associated Job transitions to the completed state.
- If no strong references to the job remain (e.g., from ViewModel, Fragment, Activity, etc.), then the job object becomes eligible for Garbage Collection.
- Also, CoroutineScope.cancel() sends a cancellation signal to all associated Jobs, which leads to their implicit cancellation — that’s why the Job gets cancelled automatically.

## 🎯 Why Explicitly Cancelling a CoroutineScope?
- All coroutines are launched within a CoroutineScope, and each coroutine is backed by a Job. Once all associated coroutines complete — either normally, with an uncaught exception, or due to cancellation — their corresponding Job objects become eligible for GC.
- However, the CoroutineScope itself remains active and not eligible for GC even after all child jobs complete, unless it is explicitly cancelled. This becomes especially important when a scope is created using the CoroutineScope interface and tied to a context like a ViewModel, Activity, or custom component.
- If the underlying context is destroyed (e.g., the Activity/Fragment is closed or the ViewModel is cleared), but the scope is not cancelled, then:
	- The scope and its resources remain alive in memory,
	- And if any jobs or references linger, it can cause memory leaks and resource waste.
- Therefore, when creating a scope manually (e.g., CoroutineScope(Job()) or CoroutineScope(SupervisorJob())), it is highly recommended to call scope.cancel() explicitly — even if all jobs are already completed — to allow proper cleanup and garbage collection of the scope itself.
- The Kotlin documentation explicitly recommends:
```
“If you create your own CoroutineScope using the CoroutineScope() function, it is your responsibility to cancel it when it's no longer needed.”
```

## 🎯 Why You Should Avoid Creating Custom CoroutineScopes in Android?

### 🟡 Official Android Guidance:
```
“Avoid creating your own CoroutineScope in components like Activities, Fragments, or ViewModels. Instead, use lifecycle-aware scopes like viewModelScope, lifecycleScope, or repeatOnLifecycle.”
 ```
- Why:
	1. ✅ Lifecycle Awareness Built-In:
		- Scopes like viewModelScope and lifecycleScope are automatically cancelled:
			1. viewModelScope → when ViewModel.onCleared() is called
			2. lifecycleScope → when the lifecycle reaches DESTROYED
		- This means you don’t have to manually call scope.cancel(), and there's no risk of leaking coroutines.

	2. ⚠️ Custom Scopes Are Manual & Risky
		- If you use CoroutineScope(Job()) or similar in an Activity/Fragment:
		- You must manually call scope.cancel() in onDestroy()
		- If you forget? 🔥 Memory leaks, zombie coroutines, or wasted CPU running in the background
		- Android warns that many developers forget this manual step — causing real-world app issues

	3. 📦 Lifecycle-Aware Scopes Are Well-Tested
		- Provided scopes (viewModelScope, lifecycleScope) are:
			- ✅ Tested
			- ✅ Integrated with Jetpack Lifecycle components
			- ✅ Less boilerplate
			- ✅ Error-proof
- Custom CoroutineScopes should be avoided in Android unless you are managing lifecycle yourself.
- Prefer viewModelScope, lifecycleScope, or rememberCoroutineScope() — they handle cleanup automatically and protect your app from memory leaks and wasted background work.

### 🟠 Nested Scopes Are Isolated
- In Kotlin Coroutines, even when one CoroutineScope is created inside another, they remain completely independent — they do not form a parent-child relationship unless explicitly linked through the same job.

```
CoroutineScope(Job()).launch {                // Scope-A

    CoroutineScope(Job()).launch { }          // Scope-B

    CoroutineScope(Job()).launch { }          // Scope-C
}
```

- At first glance, it may seem that Scope-A is the parent and Scope-B and Scope-C are its children or siblings — but that’s an illusion.
- In reality: Each CoroutineScope(Job()) creates a new, standalone job hierarchy.
- They do not share a parent job, and hence are not part of the same coroutine family tree.

### 🟠 Behavior Summary of Nested CoroutineScopes with individual Job

1. ⚠️ Cancellation:
- Each CoroutineScope(Job()) creates a separate coroutine hierarchy.
- Cancelling one scope (e.g., Scope-A) sends a cancellation signal only to its own coroutines, not to other nested scopes like Scope-B or Scope-C.
- Likewise, if Scope-B or Scope-C is cancelled, it does not affect Scope-A or its siblings.
- Cancellation is strictly scoped. No scope can cancel another unless they share the same job or parent context.

```
suspend fun main () {
    val scope1 = CoroutineScope(Job())
    val scope2 = CoroutineScope(Job())

    val job = scope1.launch {

        delay(100)
        scope2.launch {
            launch {
                println("Scope-2 coroutine is started")
                repeat(1000) {
                    println("Scope-2: Coroutine -> Count: $it")
                    delay(10)
                }
                println("Scope-2 coroutine is finished")
            }.invokeOnCompletion {
                println("Scope-2 coroutine is completed: $it")
            }
        }.invokeOnCompletion {
            println("Scope-2 is completed: $it")
        }

        launch {
            println("Scope-1 coroutine is started")
            repeat(1000) {
                println("Scope-1: Coroutine -> Count: $it")
                delay(10)
            }
            println("Scope-1 coroutine is finished")
        }.invokeOnCompletion {
            println("Scope-1 coroutine is completed: $it")
        }

    }

    job.invokeOnCompletion {
        println("Scope-1 is completed: $it")
    }

    delay(1000)
    scope1.cancel()
    println("Scope-1 cancelled")

    delay(2000)
    scope2.cancel()
    println("Scope-2 cancelled")
}
```

**Output:**

```
Scope-2 coroutine is started
Scope-1 coroutine is started
Scope-1: Coroutine -> Count: 0
Scope-2: Coroutine -> Count: 0
Scope-1: Coroutine -> Count: 1
Scope-2: Coroutine -> Count: 1
...
Scope-2: Coroutine -> Count: 52
Scope-1: Coroutine -> Count: 52
Scope-1 cancelled
Scope-1 coroutine is completed: kotlinx.coroutines.JobCancellationException: Job was cancelled; job=JobImpl{Cancelling}@1d744bbd
Scope-1 is completed: kotlinx.coroutines.JobCancellationException: Job was cancelled; job=JobImpl{Cancelling}@1d744bbd
Scope-2: Coroutine -> Count: 53
...
Scope-2: Coroutine -> Count: 172
Scope-2 cancelled
Scope-2 coroutine is completed: kotlinx.coroutines.JobCancellationException: Job was cancelled; job=JobImpl{Cancelling}@3a70f639
Scope-2 is completed: kotlinx.coroutines.JobCancellationException: Job was cancelled; job=JobImpl{Cancelling}@3a70f639
```
  
2. 🚫 Exception:
- If an uncaught exception is thrown in one scope (e.g., Scope-B), it stays within that scope.
- The exception does not propagate to outer scopes or sibling scopes.
- The only way to catch it is:
	- Imperatively, using try-catch inside the coroutine.
   							OR
	- Declaratively, using a CoroutineExceptionHandler in the same scope.
- Exceptions are not shared across different scopes. This avoids unwanted failures spilling into unrelated coroutine hierarchies.

```
suspend fun main () {
    val scope1 = CoroutineScope(Job())
    val scope2 = CoroutineScope(Job() + CoroutineExceptionHandler{_, throwable -> println("Scope-2:- CoroutineExceptionHandler: $throwable") })

    scope1.launch {

        launch {
            println("Scope-1 coroutine is started")
            delay(2000)
            println("Scope-1 coroutine is finished")
        }.invokeOnCompletion {
            println("Scope-1 coroutine is completed: $it")
        }

        scope2.launch {
            launch {
                println("Scope-2 coroutine is started")
                delay(1000)
                throw RuntimeException()
                println("Scope-2 coroutine is finished")
            }.invokeOnCompletion {
                println("Scope-2 coroutine is completed: $it")
            }
        }.invokeOnCompletion {
            println("Scope-2 is completed: $it")
        }
    }.run {
        invokeOnCompletion {
            println("Scope-1 is completed: $it")
        }
        join()
    }
    scope1.cancel()
    scope2.cancel()
}
```

**Output:**

```
Scope-1 coroutine is started
Scope-2 coroutine is started
Scope-2 coroutine is completed: java.lang.RuntimeException
Scope-2:- CoroutineExceptionHandler: java.lang.RuntimeException
Scope-2 is completed: java.lang.RuntimeException
Scope-1 coroutine is finished
Scope-1 coroutine is completed: null
Scope-1 is completed: null
```

3. ✅ Completion
- Each scope manages its own lifecycle independently.
- When Scope-A completes (normally or due to cancellation), it does not wait for Scope-B or Scope-C to complete, and vice versa.

```
suspend fun main () {
    val scope1 = CoroutineScope(Job())
    val scope2 = CoroutineScope(Job())

    scope1.launch {
        
        scope2.launch {
            delay(2000)
        }.invokeOnCompletion {
            println("Scope-2 is completed: $it")
        }

        delay(1000)
    }.invokeOnCompletion {
        println("Scope-1 is completed: $it")
    }
    
    delay(3100)
    scope1.cancel()
    scope2.cancel()
}
```

**Output:**

```
Scope-1 is completed: null
Scope-2 is completed: null
```

### 🟠 Summary:
- Even if coroutine scopes are written one inside another, they are logically isolated.
- They do not form parent-child or sibling relationships unless they share the same context or job.

---

### 🟠 Behavior Summary of Nested CoroutineScopes with same Job

When do:

```
val job = Job()
val scope1 = CoroutineScope(job)
val scope2 = CoroutineScope(job)

scope1.launch { ... }
scope2.launch { ... }
```

- You are injecting the same parent job into multiple scopes. So all coroutines launched from both scopes become structured children of that single job.
- They are logically related, even though the scopes are technically separate instances.

1. ⚠️ Cancellation:
- Calling job.cancel() will cancel all child coroutines launched by both scopes.
- If a child coroutine throws a CancellationException, the parent job gets canceled (if not caught).
- If multiple scopes share the same Job, cancelling one scope cancels all associated scopes and their coroutines.

```
suspend fun main () {
    val job = Job()
    val scope1 = CoroutineScope(job)
    val scope2 = CoroutineScope(job)

    val job1 = scope1.launch {

        delay(100)
        scope2.launch {
            launch {
                println("Scope-2 coroutine is started")
                repeat(1000) {
                    println("Scope-2: Coroutine -> Count: $it")
                    delay(10)
                }
                println("Scope-2 coroutine is finished")
            }.invokeOnCompletion {
                println("Scope-2 coroutine is completed: $it")
            }
        }.invokeOnCompletion {
            println("Scope-2 is completed: $it")
        }

        launch {
            println("Scope-1 coroutine is started")
            repeat(1000) {
                println("Scope-1: Coroutine -> Count: $it")
                delay(10)
            }
            println("Scope-1 coroutine is finished")
        }.invokeOnCompletion {
            println("Scope-1 coroutine is completed: $it")
        }

    }

    job1.invokeOnCompletion {
        println("Scope-1 is completed: $it")
    }

    delay(1000)
    job.cancel()
    println("Scope-1 isActive: ${scope1.isActive}")
    println("Scope-2 isActive: ${scope2.isActive}")
}
```

**Output:**

```
Scope-2 coroutine is started
Scope-2: Coroutine -> Count: 0
Scope-1 coroutine is started
Scope-1: Coroutine -> Count: 0
Scope-2: Coroutine -> Count: 1
Scope-1: Coroutine -> Count: 1
...
Scope-2: Coroutine -> Count: 54
Scope-1: Coroutine -> Count: 54
Scope-1 isActive: false
Scope-2 isActive: false
```

```
suspend fun main () {
    val scope1 = CoroutineScope(Job())
    val scope2 = CoroutineScope(Job())

    val job = scope1.launch {

        delay(100)
        scope2.launch {
            launch {
                println("Scope-2 coroutine is started")
                repeat(1000) {
                    println("Scope-2: Coroutine -> Count: $it")
                    delay(10)
                }
                println("Scope-2 coroutine is finished")
            }.invokeOnCompletion {
                println("Scope-2 coroutine is completed: $it")
            }
        }.invokeOnCompletion {
            println("Scope-2 is completed: $it")
        }

        launch {
            println("Scope-1 coroutine is started")
            repeat(1000) {
                println("Scope-1: Coroutine -> Count: $it")
                delay(10)
            }
            println("Scope-1 coroutine is finished")
        }.invokeOnCompletion {
            println("Scope-1 coroutine is completed: $it")
        }

    }

    job.invokeOnCompletion {
        println("Scope-1 is completed: $it")
    }

    delay(1000)
    scope1.cancel()
    println("Scope-1 cancelled")

    delay(2000)
    scope2.cancel()
    println("Scope-2 cancelled")
}
```

**Output:**

```
Scope-2 coroutine is started
Scope-2: Coroutine -> Count: 0
Scope-1 coroutine is started
Scope-1: Coroutine -> Count: 0
Scope-2: Coroutine -> Count: 1
Scope-1: Coroutine -> Count: 1
...
Scope-1: Coroutine -> Count: 55
Scope-2: Coroutine -> Count: 56
Scope-1 cancelled
Scope-2 isActive: false
Scope-2 coroutine is completed: kotlinx.coroutines.JobCancellationException: Job was cancelled; job=JobImpl{Cancelling}@6eb715d3
Scope-2 is completed: kotlinx.coroutines.JobCancellationException: Job was cancelled; job=JobImpl{Cancelling}@6eb715d3
```
  
2. 🚫 Exception:
- An uncaught exception in a coroutine launched from any scope sharing the same Job() will propagate upward to that shared job.
- If the shared Job() is an ordinary Job and:
	- A CoroutineExceptionHandler is attached to the job (either via CoroutineScope(CoroutineExceptionHandler) or launch(CoroutineExceptionHandler)),
	- The exception will be caught declaratively, and
	- The job will be cancelled, causing all associated scopes and coroutines to be cancelled.
- If the job is a SupervisorJob, then:
	- The exception is still caught by the CoroutineExceptionHandler (if present),
	- But cancellation will not propagate to sibling coroutines — other scopes will continue running unaffected.
- If there is no CoroutineExceptionHandler, the uncaught exception will propagate and:
	- If on the main thread, then the exception crashes the app, because it bubbles up to the main thread’s uncaught exception handler, which usually terminates the process.
   	- If running on an IO or Default dispatcher, then there may be a chance the exception may be caught by the Global Coroutine Exception handler or the Default Exception Handler, and just the log will print on the console "Exception in thread "DefaultDispatcher-worker-1" java.lang.RuntimeException: RuntimeException"


```
suspend fun main () {
    val job = Job()
    val scope1 = CoroutineScope(job)
    val scope2 =
        CoroutineScope(job + CoroutineExceptionHandler { _, throwable -> println("Scope-2:- CoroutineExceptionHandler: $throwable") })

    scope1.launch {

        launch {
            println("Scope-1 coroutine is started")
            delay(2000)
            println("Scope-1 coroutine is finished")
        }.invokeOnCompletion {
            println("Scope-1 coroutine is completed: $it")
        }

        scope2.launch {
            launch {
                println("Scope-2 coroutine is started")
                delay(1000)
                throw RuntimeException()
                println("Scope-2 coroutine is finished")
            }.invokeOnCompletion {
                println("Scope-2 coroutine is completed: $it")
            }
        }.invokeOnCompletion {
            println("Scope-2 is completed: $it")
        }
    }.run {
        invokeOnCompletion {
            println("Scope-1 is completed: $it")
        }
        join()
    }
    scope1.cancel()
    scope2.cancel()
}
```

**Output:**

```
Scope-1 coroutine is started
Scope-2 coroutine is started
Scope-2 coroutine is completed: java.lang.RuntimeException
Scope-1 coroutine is completed: kotlinx.coroutines.JobCancellationException: Parent job is Cancelling; job=JobImpl{Cancelling}@2dec2da5
Scope-1 is completed: kotlinx.coroutines.JobCancellationException: Parent job is Cancelling; job=JobImpl{Cancelling}@2dec2da5
Scope-2:- CoroutineExceptionHandler: java.lang.RuntimeException
Scope-2 is completed: java.lang.RuntimeException
```

```
suspend fun main () {
    val supervisorJob = SupervisorJob()
    val scope1 = CoroutineScope(supervisorJob)
    val scope2 =
        CoroutineScope(supervisorJob + CoroutineExceptionHandler { _, throwable -> println("Scope-2:- CoroutineExceptionHandler: $throwable") })

    scope1.launch {

        launch {
            println("Scope-1 coroutine is started")
            delay(2000)
            println("Scope-1 coroutine is finished")
        }.invokeOnCompletion {
            println("Scope-1 coroutine is completed: $it")
        }

        scope2.launch {
            launch {
                println("Scope-2 coroutine is started")
                delay(1000)
                throw RuntimeException()
                println("Scope-2 coroutine is finished")
            }.invokeOnCompletion {
                println("Scope-2 coroutine is completed: $it")
            }
        }.invokeOnCompletion {
            println("Scope-2 is completed: $it")
        }
    }.run {
        invokeOnCompletion {
            println("Scope-1 is completed: $it")
        }
        join()
    }
    scope1.cancel()
    scope2.cancel()
}
```

**Output:**

```
Scope-1 coroutine is started
Scope-2 coroutine is started
Scope-2 coroutine is completed: java.lang.RuntimeException
Scope-2:- CoroutineExceptionHandler: java.lang.RuntimeException
Scope-2 is completed: java.lang.RuntimeException
Scope-1 coroutine is finished
Scope-1 coroutine is completed: null
Scope-1 is completed: null
```

3. ✅ Completion
- If multiple scopes share the same Job, like scope1 and scope2, then the parent Job completes only when all child coroutines launched from all scopes are completed — either normally or via cancellation.
- This is part of structured concurrency: the parent job won’t complete until all of its children (direct or indirect) are finished.
- invokeOnCompletion executes after all children complete, and provides insight into how the job finished.


---

### 🟠 Nested Coroutines with individual Jobs Are Isolated

- If a coroutine is launched with its own Job(), then:
	1. It breaks structured concurrency with the parent — it won’t be cancelled when the parent is cancelled.
	2. It becomes a separate coroutine with an independent lifecycle.
	3. It is not a true child of the parent — even if visually nested, it is not linked to the parent’s job.
	4. It still inherits other context elements (like Dispatcher, CoroutineName) from the parent if not explicitly overridden, but not the Job.
		- A coroutine cannot set a default dispatcher; it inherits unless provided explicitly.
	5. It forms a new coroutine hierarchy — separate from the parent — and is excluded from the parent’s cancellation, exception handling, and structured waiting.

1. Cancellation:
- When a coroutine is launched with its own Job(), it is not linked to the parent’s job hierarchy.
- If the parent scope or coroutine is cancelled, the coroutine with its own job will not be cancelled — it continues to run independently.
- Likewise, if this coroutine is cancelled, it does not affect the parent or sibling coroutines — they continue to run.
- To cancel such a coroutine, you must explicitly cancel its Job using the reference.
- This breaks structured concurrency, making lifecycle management manual and isolated.
   
```
suspend fun main () {
    val scope = CoroutineScope(Job())

    val job = scope.launch {

        launch(Job()) {    // Injecting a new Job — breaks parent relationship!
            println("Task-1 is started")
            repeat(1000) {
                println("Coroutine-A -> Count: $it")
                delay(10)
            }
            println("Task-1 is finished")
        }.invokeOnCompletion {
            println("Coroutine-1 is completed: $it")
        }

        launch {
            println("Task-2 is started")
            repeat(1000) {
                println("Coroutine-B -> Count: $it")
                delay(10)
            }
            println("Task-2 is finished")
        }.invokeOnCompletion {
            println("Coroutine-2 is completed: $it")
        }

    }

    job.invokeOnCompletion {
        println("Coroutine is completed: $it")
    }

    delay(500)
    scope.cancel()
    println("Scope cancelled")

    delay(20000)
}
```

**Output:**

```
Task-1 is started
Coroutine-A -> Count: 0
Task-2 is started
Coroutine-B -> Count: 0
Coroutine-A -> Count: 1
Coroutine-B -> Count: 1
...
Coroutine-A -> Count: 30
Coroutine-B -> Count: 30
Scope cancelled
Coroutine-A -> Count: 31
Coroutine-2 is completed: kotlinx.coroutines.JobCancellationException: Job was cancelled; job=JobImpl{Cancelling}@e8d68ea
Coroutine is completed: kotlinx.coroutines.JobCancellationException: Job was cancelled; job=JobImpl{Cancelling}@e8d68ea
Coroutine-A -> Count: 32
...
Coroutine-A -> Count: 999
Task-1 is finished
Coroutine-1 is completed: null
```

```
suspend fun main () {
    val scope = CoroutineScope(Job())

    val job = scope.launch {

        launch(Job()) {     // Injecting a new Job — breaks parent relationship!
            println("Task-1 is started")
            repeat(1000) {
                println("Coroutine-A -> Count: $it")
                delay(10)
            }
            println("Task-1 is finished")
        }.invokeOnCompletion {
            println("Coroutine-1 is completed: $it")
        }

        launch {
            println("Task-2 is started")
            repeat(1000) {
                println("Coroutine-B -> Count: $it")
                delay(10)
            }
            println("Task-2 is finished")
        }.invokeOnCompletion {
            println("Coroutine-2 is completed: $it")
        }

    }

    job.invokeOnCompletion {
        println("Coroutine is completed: $it")
    }

    delay(500)
    job.cancel()
    println("Job cancelled")

    delay(20000)
    scope.cancel()
}
```

**Output:**

```
Task-1 is started
Coroutine-A -> Count: 0
Task-2 is started
Coroutine-B -> Count: 0
Coroutine-A -> Count: 1
...
Coroutine-B -> Count: 29
Coroutine-A -> Count: 29
Job cancelled
Coroutine-2 is completed: kotlinx.coroutines.JobCancellationException: StandaloneCoroutine was cancelled; job=StandaloneCoroutine{Cancelling}@68698b33
Coroutine is completed: kotlinx.coroutines.JobCancellationException: StandaloneCoroutine was cancelled; job=StandaloneCoroutine{Cancelled}@68698b33
Coroutine-A -> Count: 30
...
Coroutine-A -> Count: 999
Task-1 is finished
Coroutine-1 is completed: null
```

2. Exception:
- When a coroutine has its own Job(), it must handle exceptions either:
- Imperatively using try-catch inside the coroutine, or
- Declaratively by adding its own CoroutineExceptionHandler in its coroutine context.
- The exception will not propagate to the parent scope or coroutine.
- The parent’s CoroutineExceptionHandler will not catch it.
- Other sibling coroutines remain unaffected and will continue to run independently.

```
suspend fun main () {
    val scope = CoroutineScope(Job() + CoroutineExceptionHandler{_, throwable -> println("scope:- CoroutineExceptionHandler: $throwable") })

    val job = scope.launch {

        launch(Job() + CoroutineExceptionHandler{_, throwable -> println("Coroutine-1:- CoroutineExceptionHandler: $throwable") }) {
            println("Task-1 is started")
            delay(2000)
            println("Task-1 is finished")
        }.invokeOnCompletion {
            println("Coroutine-1 is completed: $it")
        }

        launch {
            println("Task-2 is started")
            delay(1000)
            throw RuntimeException()
            println("Task-2 is finished")
        }.invokeOnCompletion {
            println("Coroutine-2 is completed: $it")
        }

    }

    job.invokeOnCompletion {
        println("Coroutine is completed: $it")
    }

    delay(3000)
    scope.cancel()
}
```

**Output:**

```
Task-1 is started
Task-2 is started
Coroutine-2 is completed: java.lang.RuntimeException
scope:- CoroutineExceptionHandler: java.lang.RuntimeException
Coroutine is completed: java.lang.RuntimeException
Task-1 is finished
Coroutine-1 is completed: null
```

```
suspend fun main () {
    val scope = CoroutineScope(Job() + CoroutineExceptionHandler{_, throwable -> println("scope:- CoroutineExceptionHandler: $throwable") })

    val job = scope.launch {

        launch(Job() + CoroutineExceptionHandler{_, throwable -> println("Coroutine-1:- CoroutineExceptionHandler: $throwable") }) {
            println("Task-1 is started")
            delay(1000)
            throw RuntimeException()
            println("Task-1 is finished")
        }.invokeOnCompletion {
            println("Coroutine-1 is completed: $it")
        }

        launch {
            println("Task-2 is started")
            delay(2000)
            println("Task-2 is finished")
        }.invokeOnCompletion {
            println("Coroutine-2 is completed: $it")
        }

    }

    job.invokeOnCompletion {
        println("Coroutine is completed: $it")
    }

    delay(3000)
    scope.cancel()
}
```

**Output:**

```
Task-1 is started
Task-2 is started
Coroutine-1:- CoroutineExceptionHandler: java.lang.RuntimeException
Coroutine-1 is completed: java.lang.RuntimeException
Task-2 is finished
Coroutine-2 is completed: null
Coroutine is completed: null
```

3. Completion:
- A coroutine with its own Job() is independent, so:
- It completes independently, either normally or with an exception or cancellation.
- Its completion is not awaited by the parent unless explicitly done using job.join().
- The parent or sibling coroutines do not wait for this coroutine to complete.
- Similarly, its completion (success or failure) does not affect the parent or other coroutines.

---

## 🎯suspend fun coroutineScope { }
- coroutineScope is a suspending function that creates a new coroutine scope inside the current suspending function or coroutine.
- It does not launch a coroutine by itself; instead, it provides a scope where you can launch child coroutines using launch, async, etc.
- It suspends the caller until all child coroutines inside the block complete, giving it a synchronous behavior from the caller’s perspective.
- If any child coroutine fails with an exception, all other children are automatically cancelled, and the first exception is re-thrown to the caller.
- The scope created by coroutineScope inherits the context (e.g., Job, Dispatcher, CoroutineExceptionHandler) from its parent.
- It ensures structured concurrency by guaranteeing that all launched coroutines complete or fail before the surrounding coroutine resumes.
- If the outer coroutine is cancelled, the coroutineScope and all its children are also cancelled automatically.
- coroutineScope { } does not return a Job or CoroutineScope. It returns the result of the lambda block, if any.
- So cannot explicitly cancel coroutineScope from outside using .cancel() like with launch or async.
- The scope created by coroutineScope inherits the parent coroutine's context, including its Job.
- If the parent scope or coroutine is cancelled, then coroutineScope is implicitly cancelled.
- All child coroutines launched inside coroutineScope are bound to the parent and follow structured concurrency rules.
- In coroutineScope { }, all coroutines launched inside it are regular child coroutines under the same Job. So, failure of any one child cancels all other children, and the first exception is rethrown.

```
val result = coroutineScope {
    launch { }
    "Result"
}
```

```
suspend fun myCoroutine() {
    val scope = CoroutineScope(Job())

    scope.launch {
        try {
            coroutineScope {
                launch {
                    println("Task-1 stared.")
                    delay(1000)
                    throw RuntimeException()
                }.invokeOnCompletion {
                    println("Coroutine-1 is completed. Error: $it")
                }
                launch {
                    println("Task-2 stared.")
                    delay(2000)
                    println("Task-2 finished.")
                }.invokeOnCompletion {
                    println("Coroutine-2 is completed. Error: $it")
                }
                "Tasks 1 & 2 are completed"
            }.run {
                println("Status CS: $this")
            }
        } catch (e: Exception) {
            println("Caught Exception: $e")
        }
    }.run {
        invokeOnCompletion {
            println("Job is completed. Error: $it")
        }
        join()
    }

    scope.cancel()
}
```

**Output:**

```
Task-1 stared.
Task-2 stared.
Coroutine-1 is completed. Error: java.lang.RuntimeException
Coroutine-2 is completed. Error: kotlinx.coroutines.JobCancellationException: Parent job is Cancelling; job=ScopeCoroutine{Cancelling}@3a133d77
Caught Exception: java.lang.RuntimeException
Job is completed. Error: null
```

### 🟠 Nesting in coroutineScope:

**1. Parent-Child**
- coroutineScope {} always takes context from the parent, which means it always inherits from the parent, that why they are children of the parent scope, so nested coroutine scopes are always parent-child of each other respectively.
- If a parent coroutineScope { } is cancelled due to its parent coroutine or scope being cancelled, then all nested coroutineScope { } blocks and their child coroutines are also implicitly cancelled.
- This behavior ensures proper structured concurrency and lifecycle tracking — the parent cannot finish until all nested scopes complete or fail.

 ```
coroutineScope { // A (Parent of B)
    coroutineScope { // B (Child of A)
        // Inherits context from the parent coroutineScope
    }
}
 ```

**2. Peers:**
- Principle-separated coroutineScope within same CoroutineScope(Job).launch {} are peers. As per job sharing, they can look siblings, but not as per execution nature. The first scope will execute and after complete second will start and so due to this nature they are peers not siblings.

```
CoroutineScope(Job()).launch {	// Parent
	coroutineScope{ } // Child-A: Have parent context
	coroutineScope{ } // Child-B: Have parent context
	// Both are Peers of each other
}
```

```
CoroutineScope(Job()).launch {
    
    // First coroutineScope starts and completes fully before the second begins
    coroutineScope {
        repeat(10) {
            println("coroutinScope-1-> Count: $it")
            delay(10)
        }
    }

    // Second coroutineScope starts only after the first completes
    coroutineScope {
        repeat(10) {
            println("coroutinScope-2-> Count: $it")
            delay(10)
        }
    }

}.run {
    join()
    cancel()
}
```

**Output:**

```
coroutineScope-1-> Count: 0
...
coroutineScope-1-> Count: 9
coroutineScope-2-> Count: 0
...
coroutineScope-2-> Count: 9
```

### 🟠 Exception handling:
- coroutineScope is a suspend function that provides a new scope to launch child coroutines.
- This new scope inherits the context (like Dispatcher and Job) from its parent but creates its own new Job, forming the root of a structured concurrency block.
- All child coroutines launched inside this scope are tied to this new Job.
- If any child coroutine fails with an exception, it cancels the entire scope (i.e., the new Job), and all sibling coroutines are cancelled as well.
- The thrown exception is not caught by any CoroutineExceptionHandler attached to the child coroutine launch(CoroutineExceptionHandler) inside coroutineScope {}.
- Instead, the exception bubbles up and is thrown from the suspend function itself, so it must be:
	- Handled imperatively using try-catch around the coroutineScope {} call
									OR
	- Handled declaratively by a CoroutineExceptionHandler in the parent coroutine or scope
- This strict behavior ensures structured concurrency: a coroutineScope will not complete until all of its children complete successfully or one fails and the failure is handled.

1. 🟠 Imperatively using try-catch:

```
 CoroutineScope(Job()).launch {
        try {
            coroutineScope{
                delay(10)
				throw RuntimeException()
            }
        } catch (e: Exception) {
            println("Caught Exception: $e")
        }
    }.join()
```

```
 CoroutineScope(Job()).launch {
        try {
            coroutineScope{
                launch {
                    delay(10)
                    throw RuntimeException()
                }
            }
        } catch (e: Exception) {
            println("Caught Exception: $e")
        }
    }.join()
```
   
```
 CoroutineScope(Job()).launch {
        try {
            coroutineScope{
                launch(CoroutineExceptionHandler{ _, throwable -> println("CoroutineExceptionHandler: $throwable") }) {
                    delay(10)
                    throw RuntimeException()
                }
            }
        } catch (e: Exception) {
            println("Caught Exception: $e")
        }
    }.join()
```

**Output:**

```
Caught Exception: java.lang.RuntimeException
```


2. 🟠 Declaratively by a CoroutineExceptionHandler:
   
```
CoroutineScope(Job()+CoroutineExceptionHandler{ _, throwable-> println("CoroutineExceptionHandler: $throwable") }).launch {
        coroutineScope{
            launch {
                delay(10)
                throw RuntimeException()
            }
        }
    }.join()
```

**Output:**

```
CoroutineExceptionHandler: java.lang.RuntimeException
```
---

## 🎯suspend fun supervisorScope { }
- supervisorScope is a suspending function that creates a new coroutine scope with a SupervisorJob, allowing child coroutines to operate independently in terms of failure.
- Like coroutineScope, it does not launch a coroutine by itself; you launch child coroutines using launch, async, etc., within the block.
- It suspends the caller until all child coroutines inside it complete, providing synchronous behavior within the suspending function.
- If any child coroutine fails, the other children continue unaffected — they are not cancelled automatically, unlike in coroutineScope.
- Only if an exception escapes the scope (i.e., not caught in children), the first uncaught exception is re-thrown to the caller after all children complete.
- It inherits the context from the outer coroutine, replacing the existing Job with a SupervisorJob.
- It also supports structured concurrency, but with failure isolation between child coroutines.
- If the outer coroutine is cancelled, the supervisorScope and all its children are also cancelled.
- supervisorScope { } does not return a Job or CoroutineScope. It returns the result of the lambda block, if any.
- So cannot explicitly cancel supervisorScope from outside using .cancel() like with launch or async.
- The scope created by supervisorScope inherits the parent coroutine's context, but replaces the Job with a SupervisorJob.
- If the parent scope or coroutine is cancelled, then supervisorScope is implicitly cancelled.
- All child coroutines launched inside supervisorScope are bound to the parent and follow structured concurrency rules.
- In supervisorScope { }, all coroutines launched directly inside it are treated as Independent & Top-level child coroutines under a SupervisorJob. So, the failure of one does not cancel others.

```
val result = supervisorScope {
    launch { }	// Top-level coroutine
	"Result"
}
```

```
suspend fun myCoroutine() {
    val scope = CoroutineScope(Job()+CoroutineExceptionHandler{_, throwable -> println("CoroutineExceptionHandler: $throwable") })

    scope.launch {
        supervisorScope {
            launch {
                println("Task-1 stared.")
                delay(1000)
                throw RuntimeException()
            }.invokeOnCompletion {
                println("Coroutine-1 is completed. Error: $it")
            }
            launch {
                println("Task-2 stared.")
                delay(2000)
                println("Task-2 finished.")
            }.invokeOnCompletion {
                println("Coroutine-2 is completed. Error: $it")
            }
            "Tasks 1 & 2 are completed"
        }.run {
            println("Status CS: $this")
        }
    }.run {
        invokeOnCompletion {
            println("Job is completed. Error: $it")
        }
        join()
    }

    scope.cancel()
}
```

**Output:**

```
Task-1 stared.
Task-2 stared.
CoroutineExceptionHandler: java.lang.RuntimeException
Coroutine-1 is completed. Error: java.lang.RuntimeException
Task-2 finished.
Coroutine-2 is completed. Error: null
Status CS: Tasks 1 & 2 are completed
Job is completed. Error: null
```


**1. Parent-Child:** 
- supervisorScope { } always takes context from the parent, but replaces the parent Job with a SupervisorJob, while retaining other context elements like Dispatcher and ExceptionHandler.
- So nested supervisorScope blocks are parent-child with each other contextually, but exception handling is isolated — failure of a child doesn't cancel its siblings.
- If a parent supervisorScope { } is cancelled due to its parent coroutine or scope being cancelled, then all nested supervisorScope { } blocks and their child coroutines are also implicitly cancelled.
- This behavior still supports structured concurrency, while allowing failure independence among children.

```
supervisorScope { // A (Parent of B)
    supervisorScope { // B (Child of A)
        // Inherits context but uses a new SupervisorJob
    }
}
```

**2. Peers:**
- Principle-separated supervisorScope within same CoroutineScope(Job()).launch { } are peers. They share the same parent context (Dispatcher, ExceptionHandler), but each has its own independent SupervisorJob. As per job sharing, they may appear to be siblings, but due to sequential structure and isolated supervision, they are peers by context, not by execution or failure behavior. The first scope runs to completion, then the next starts.

```
CoroutineScope(Job()).launch {    // Parent
    supervisorScope {
        launch {  } // Top-level coroutine
        launch {  } // Top-level coroutine
    } // Child-A: Has an independent SupervisorJob
    supervisorScope {
        launch {  } // Top-level coroutine
        launch {  } // Top-level coroutine
    } // Child-B: Has an independent SupervisorJob
    // Both Child A & B are Peers of each other
}
```

```
CoroutineScope(Job()).launch {

    // First supervisorScope starts and completes fully before the second begins
    supervisorScope {
        launch {	// Top-level coroutine in this scope
            repeat(10) {
                println("supervisorScope-1-> Count: $it")
                delay(10)
            }
        }
    }

    // Second supervisorScope starts only after the first completes
    supervisorScope {
        launch {	// Top-level coroutine in this scope
            repeat(10) {
                println("supervisorScope-2-> Count: $it")
                delay(10)
            }
        }
    }

}.run {
    join()
    cancel()
}
```

**Output:**

```
supervisorScope-1-> Count: 0
...
supervisorScope-1-> Count: 9
supervisorScope-2-> Count: 0
...
supervisorScope-2-> Count: 9
```

### 🟠 Exception handling:
- suspend fun supervisorScope {} inherits the coroutine context from its parent but has its own SupervisorJob.
- All coroutines launched directly inside this scope are top-level coroutines and structured children of the SupervisorJob — they are independent in failure, so if one child fails, it does not cancel other children.
- If a child coroutine throws an exception, it can be caught:
	Imperatively, using try-catch inside that coroutine
							OR
	Declaratively, if it is a root coroutine with its own CoroutineExceptionHandler
- If the exception is not caught, it is not rethrown by the supervisorScope suspend function itself.
- Instead, it propagates upward through the coroutine context, and can be handled declaratively by a CoroutineExceptionHandler in the parent scope.
- If the parent scope does not have a CoroutineExceptionHandler, then the coroutine will fail and crash the program.
- However, if an exception is thrown directly inside the body of the suspend function (not from a child coroutine), then supervisorScope {} will rethrow that exception, and it can be handled:
	Imperatively, using try-catch around the supervisorScope call
							OR
	Declaratively, using a CoroutineExceptionHandler in the parent scope

1. 🟠 Imperatively using try-catch:
   
```
CoroutineScope(Job() + CoroutineExceptionHandler{ _, throwable -> println("CoroutineExceptionHandler: $throwable") }).launch {
    try {
        supervisorScope {
            println("Task stated")
            delay(10)
            throw RuntimeException()
        }
    } catch (e: Exception) {
        println("Caught Exception: $e")
    }
}.run{
    join()
    cancel()
}
```

**Output:**

```
Task stated
Caught Exception: java.lang.RuntimeException
```

2. 🟠 Declaratively by a CoroutineExceptionHandler:

```
CoroutineScope(Job() + CoroutineExceptionHandler{ _, throwable -> println("CoroutineExceptionHandler: $throwable") }).launch {
    try {
        supervisorScope {
            launch {
                println("Task stated")
                delay(10)
                throw RuntimeException()
            }
        }
    } catch (e: Exception) {
        println("Caught Exception: $e")
    }
}.run{
    join()
    cancel()
}
```

**Output:**

```
Task stated
CoroutineExceptionHandler: java.lang.RuntimeException
```

3. 🟠 Declaratively by a child Coroutines CoroutineExceptionHandler:

```
CoroutineScope(Job() + CoroutineExceptionHandler{ _, throwable -> println("CoroutineExceptionHandler: $throwable") }).launch {
    try {
        supervisorScope {
            launch(CoroutineExceptionHandler{ _, throwable -> println("Inner-CoroutineExceptionHandler: $throwable") }) {
                println("Task stated")
                delay(10)
                throw RuntimeException()
            }
        }
    } catch (e: Exception) {
        println("Caught Exception: $e")
    }
}.run{
	join()
	cancel()
}
```

**Output:**

```
Task stated
Inner-CoroutineExceptionHandler: java.lang.RuntimeException
```

4. 🟠 Declaratively by a CoroutineExceptionHandler even nested inside coroutineScope{ }:

```
CoroutineScope(Job() + CoroutineExceptionHandler{ _, throwable -> println("CoroutineExceptionHandler: $throwable") }).launch {
    try {
        coroutineScope{
            supervisorScope {
                launch {
                    println("Task stated")
                    delay(10)
                    throw RuntimeException()
                }
            }
        }
    } catch (e: Exception) {
        println("Caught Exception: $e")
    }
}.run{
    join()
    cancel()
}
```

**Output:**

```
Task stated
CoroutineExceptionHandler: java.lang.RuntimeException
```

---
---
● [Kotlin Coroutines and Flow for Android Development](https://www.udemy.com/course/coroutines-on-android/?couponCode=NVDINCTA35CTR)

---

![Kotlin Coroutines and Flow for Android Development](https://github.com/user-attachments/assets/11008b26-e3a2-473a-8b5e-eaee062f16b2)

---
