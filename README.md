# Go Concurrency for Beginners
### *From Zero to Confident — One Chapter at a Time*

> **Who this book is for:** You know a little Go (variables, functions, loops), but the moment someone says "goroutine" or "channel" your brain goes blank. This book is written *for you*.

---

## Table of Contents

1. [Chapter 1 — What Even Is Concurrency?](#chapter-1)
2. [Chapter 2 — Your First Goroutine](#chapter-2)
3. [Chapter 3 — The Problem: Goroutines Don't Wait](#chapter-3)
4. [Chapter 4 — WaitGroup: Teaching Go to Wait](#chapter-4)
5. [Chapter 5 — Channels: Goroutines Talking to Each Other](#chapter-5)
6. [Chapter 6 — Buffered Channels](#chapter-6)
7. [Chapter 7 — The select Statement](#chapter-7)
8. [Chapter 8 — Mutex: Protecting Shared Data](#chapter-8)
9. [Chapter 9 — Real Patterns You'll Actually Use](#chapter-9)
10. [Chapter 10 — Common Mistakes & How to Avoid Them](#chapter-10)
11. [Chapter 11 — Putting It All Together: A Mini Project](#chapter-11)
12. [Quick Reference Cheat Sheet](#cheatsheet)

---

<a name="chapter-1"></a>
## Chapter 1 — What Even Is Concurrency?

### The coffee shop analogy

Imagine you own a coffee shop. You have **one barista** (that's your program). A customer walks in and orders a latte. The barista:

1. Grinds the beans (2 min)
2. Steams the milk (1 min)
3. Assembles the drink (30 sec)

While grinding beans, **is the barista just standing there staring at the machine?** No. A smart barista starts steaming the milk while the beans grind. That's **concurrency** — doing multiple things *in overlapping time periods*.

### Concurrency vs Parallelism (the most confused pair in programming)

| | Concurrency | Parallelism |
|---|---|---|
| **Definition** | Dealing with multiple things at once | Actually doing multiple things at the exact same moment |
| **Requires** | Smart scheduling | Multiple CPU cores |
| **Analogy** | One barista juggling tasks | Two baristas each making a drink simultaneously |

> **Key insight:** Concurrency is about *structure*. Parallelism is about *execution*. Go gives you concurrency, and the Go runtime may run things in parallel if you have multiple cores — but you don't need to think about that yet.

### Why do we need concurrency in programs?

Real programs spend a lot of time **waiting**:
- Waiting for a file to load from disk
- Waiting for an HTTP response from an API
- Waiting for a database query

Without concurrency, your program sits there doing **nothing** during all that waiting time. With concurrency, you can do other useful work while waiting.

```
Without concurrency:
[Task A: 3s]---------->[Task B: 2s]---------->[Task C: 1s] = 6 seconds total

With concurrency:
[Task A: 3s]
   [Task B: 2s]
      [Task C: 1s]                                         = ~3 seconds total
```

### Go's approach to concurrency

Most languages use **threads** directly. Threads are powerful but heavy (each one uses ~1MB of memory) and tricky to manage.

Go invented **goroutines**. A goroutine starts with only ~2KB of memory and is managed by the Go runtime, not the OS. You can run **millions** of goroutines. They are the heart of Go concurrency.

Go's concurrency philosophy is summed up in this famous quote:

> *"Do not communicate by sharing memory; instead, share memory by communicating."*

In practice: instead of multiple goroutines reaching into a shared variable (dangerous), they send data to each other through **channels** (safe). You'll master both in this book.

---

<a name="chapter-2"></a>
## Chapter 2 — Your First Goroutine

### What is a goroutine?

A goroutine is a **function that runs concurrently** with other functions. You launch one with the `go` keyword. That's it.

```go
go someFunction()
```

One word. That's the entire syntax.

### Your first goroutine

```go
package main

import (
    "fmt"
    "time"
)

func sayHello() {
    fmt.Println("Hello from a goroutine!")
}

func main() {
    go sayHello() // Launch the goroutine

    // Give it a moment to run (we'll fix this ugly hack in Chapter 3)
    time.Sleep(100 * time.Millisecond)

    fmt.Println("Hello from main!")
}
```

**Output:**
```
Hello from a goroutine!
Hello from main!
```

### What just happened?

```
main() starts
   │
   ├──> go sayHello()  ← spawned, runs on its own
   │         │
   │    (prints "Hello from a goroutine!")
   │
   └──> time.Sleep (main pauses 100ms)
   │
   └──> prints "Hello from main!"
   │
main() ends → ALL goroutines are killed
```

> ⚠️ **Critical rule:** When `main()` ends, **every goroutine is immediately killed**, even if they haven't finished. This is why we need `time.Sleep` above — a dirty hack we'll replace soon.

### Running multiple goroutines

```go
package main

import (
    "fmt"
    "time"
)

func worker(id int) {
    fmt.Printf("Worker %d starting\n", id)
    time.Sleep(time.Second) // simulate work
    fmt.Printf("Worker %d done\n", id)
}

func main() {
    fmt.Println("Starting workers...")

    go worker(1)
    go worker(2)
    go worker(3)

    time.Sleep(2 * time.Second)
    fmt.Println("All workers done")
}
```

**Output (order may vary each run!):**
```
Starting workers...
Worker 3 starting
Worker 1 starting
Worker 2 starting
Worker 1 done
Worker 3 done
Worker 2 done
All workers done
```

Notice the order is **not guaranteed**. Goroutines run concurrently, and the Go scheduler decides when each one gets CPU time. This is expected and normal.

### Goroutines with anonymous functions

You don't need a named function. You can launch a goroutine inline:

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    go func() {
        fmt.Println("I'm an anonymous goroutine!")
    }() // ← the () at the end calls the function immediately

    time.Sleep(100 * time.Millisecond)
}
```

This is very common in Go code. Get comfortable with the `go func() { ... }()` pattern.

### ✅ When to use goroutines

Use a goroutine when you want to:
- Do something in the background while your main code continues
- Run multiple independent tasks at the same time
- Handle each incoming request in a web server concurrently

### ❌ When NOT to use goroutines

Don't use a goroutine when:
- The task is so fast it's not worth the overhead (e.g., adding two numbers)
- You need the result before you can continue (use a channel or WaitGroup instead)
- You're just learning and aren't sure — start simple, add concurrency only when you need it

---

<a name="chapter-3"></a>
## Chapter 3 — The Problem: Goroutines Don't Wait

### Why `time.Sleep` is a terrible solution

In Chapter 2 we used `time.Sleep` to wait for goroutines. This is terrible because:

1. How do you know how long to sleep? 
2. What if a goroutine takes longer than expected?
3. Sleeping wastes time — your program could finish sooner but it's stuck waiting

```go
// BAD: What if worker takes 2 seconds instead of 1?
go worker()
time.Sleep(1 * time.Second) // worker might not be done!
```

### The race condition problem

Here's something scarier. What happens when two goroutines touch the same variable?

```go
package main

import (
    "fmt"
)

func main() {
    counter := 0

    go func() { counter++ }()
    go func() { counter++ }()

    // (assuming they both finish before main exits)
    fmt.Println(counter) // What will this print?
}
```

You'd expect `2`, right? But you might get `1`, or even `0`. This is called a **race condition** — when multiple goroutines read and write the same data without coordination.

Here's why it happens at the CPU level:

```
Goroutine 1: reads counter (gets 0)
Goroutine 2: reads counter (gets 0)   ← reads BEFORE goroutine 1 writes!
Goroutine 1: writes counter = 0 + 1 = 1
Goroutine 2: writes counter = 0 + 1 = 1  ← overwrites goroutine 1's work!
Result: counter = 1, not 2
```

Go has a built-in race detector for exactly this. Run your programs with:
```bash
go run -race main.go
```

It will scream at you if you have a race condition. **Use this. Always.**

### The three solutions

Go gives us three main tools to solve these problems:

| Problem | Solution |
|---|---|
| "Wait for goroutines to finish" | `sync.WaitGroup` |
| "Protect shared data" | `sync.Mutex` |
| "Communicate between goroutines" | `channels` |

We'll cover all three. Let's start with `WaitGroup`.

---

<a name="chapter-4"></a>
## Chapter 4 — WaitGroup: Teaching Go to Wait

### What is a WaitGroup?

A `WaitGroup` is like a **counter** that tracks how many goroutines are still running. Your main function tells it "I'm waiting for N goroutines" and then blocks until the counter reaches zero.

It lives in the `sync` package: `sync.WaitGroup`

### The three methods you need to know

```go
var wg sync.WaitGroup

wg.Add(1)   // "Hey WaitGroup, I'm about to launch 1 goroutine"
wg.Done()   // "Hey WaitGroup, I (the goroutine) am finished"
wg.Wait()   // "Main is blocking here until count reaches 0"
```

### Your first WaitGroup

```go
package main

import (
    "fmt"
    "sync"
)

func worker(id int, wg *sync.WaitGroup) {
    defer wg.Done() // ← Always use defer! It runs when the function returns
    fmt.Printf("Worker %d starting\n", id)
    // do some work...
    fmt.Printf("Worker %d done\n", id)
}

func main() {
    var wg sync.WaitGroup

    for i := 1; i <= 3; i++ {
        wg.Add(1)       // Add 1 for each goroutine we're about to launch
        go worker(i, &wg) // Pass a POINTER to wg (important!)
    }

    wg.Wait() // Block here until all workers call Done()
    fmt.Println("All workers finished!")
}
```

**Output:**
```
Worker 1 starting
Worker 3 starting
Worker 2 starting
Worker 2 done
Worker 1 done
Worker 3 done
All workers finished!
```

No `time.Sleep`. No guessing. It waits *exactly* until all goroutines are done.

### Why `defer wg.Done()`?

`defer` means "run this when the function returns, no matter what". Even if the function panics or returns early, `wg.Done()` will still be called. If you forget to call `Done()`, your program will hang forever at `wg.Wait()`. Always use `defer`.

```go
// ✅ Good
func worker(wg *sync.WaitGroup) {
    defer wg.Done() // Runs no matter what
    // ... work
}

// ❌ Risky — what if there's an early return or panic?
func worker(wg *sync.WaitGroup) {
    // ... work
    wg.Done() // Might not be reached
}
```

### Why pass `&wg` (a pointer)?

Because `WaitGroup` maintains internal state (the counter). If you pass it by value, you copy the whole thing and the goroutine's `Done()` call affects the *copy*, not the original. Always pass `*sync.WaitGroup`.

```go
// ✅ Correct
go worker(i, &wg)

// ❌ Wrong — goroutine gets a copy, Done() doesn't affect main's counter
go worker(i, wg)
```

### A complete real-world example

Imagine you need to fetch data from 5 URLs concurrently:

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func fetchURL(url string, wg *sync.WaitGroup) {
    defer wg.Done()
    
    fmt.Printf("Fetching: %s\n", url)
    time.Sleep(500 * time.Millisecond) // Simulate network call
    fmt.Printf("Done fetching: %s\n", url)
}

func main() {
    urls := []string{
        "https://example.com/api/users",
        "https://example.com/api/posts",
        "https://example.com/api/comments",
        "https://example.com/api/photos",
        "https://example.com/api/todos",
    }

    var wg sync.WaitGroup

    start := time.Now()

    for _, url := range urls {
        wg.Add(1)
        go fetchURL(url, &wg)
    }

    wg.Wait()

    elapsed := time.Since(start)
    fmt.Printf("\nAll fetched in %v\n", elapsed)
    // With concurrency: ~500ms
    // Without concurrency: ~2500ms
}
```

### ✅ When to use WaitGroup

- When you launch N goroutines and need to wait for **all of them** to finish
- When goroutines don't need to return values back to you (they just do work)
- When you want to fan out work and then collect when everything is done

### ❌ When NOT to use WaitGroup

- When goroutines need to send results back — use **channels** instead
- When you need only *some* goroutines to finish, not all

---

<a name="chapter-5"></a>
## Chapter 5 — Channels: Goroutines Talking to Each Other

### The big idea

So far our goroutines do work but don't give anything back. What if you want a goroutine to **return a result**? You can't just `return` from a goroutine — nobody is listening.

Channels solve this. A channel is a **pipe** that connects goroutines. One goroutine puts data in one end, another picks it up from the other end.

```
Goroutine A ──[data]──► channel ──[data]──► Goroutine B
```

### Creating a channel

```go
// Create a channel that carries integers
ch := make(chan int)

// Create a channel that carries strings
ch := make(chan string)

// Create a channel that carries any type
ch := make(chan MyStruct)
```

### Sending and receiving

```go
ch <- 42        // SEND 42 into the channel
value := <-ch   // RECEIVE from the channel
```

The arrow `<-` always points in the direction data flows.

### Your first channel

```go
package main

import "fmt"

func sum(a, b int, ch chan int) {
    result := a + b
    ch <- result // Send result into the channel
}

func main() {
    ch := make(chan int)

    go sum(3, 7, ch) // Launch goroutine

    result := <-ch // Receive result — blocks until goroutine sends
    fmt.Println("3 + 7 =", result)
}
```

**Output:**
```
3 + 7 = 10
```

Notice: **no WaitGroup, no Sleep needed!** The `<-ch` on the receiving end automatically blocks until the goroutine sends something. It's built-in synchronization.

### Channels block — this is a feature, not a bug

```
SEND blocks if nobody is ready to receive.
RECEIVE blocks if nobody has sent anything yet.
```

This is what makes channels safe. You can't miss a message or read garbage data.

```go
ch := make(chan int)

// This would block forever (deadlock) if run in main:
// value := <-ch  // Nobody will ever send!

// This would also block forever:
// ch <- 42  // Nobody will ever receive!
```

Go will detect certain deadlocks and panic with: `fatal error: all goroutines are asleep - deadlock!`

### Returning results from multiple goroutines

```go
package main

import "fmt"

func double(n int, ch chan int) {
    ch <- n * 2
}

func main() {
    ch := make(chan int)

    go double(5, ch)
    go double(10, ch)
    go double(15, ch)

    // Receive exactly 3 results
    a := <-ch
    b := <-ch
    c := <-ch

    fmt.Println(a, b, c) // 10, 20, 30 (order may vary)
}
```

### Directional channels (read-only / write-only)

You can restrict a channel to only sending or only receiving. This is great for making your code clearer and catching bugs at compile time:

```go
func producer(ch chan<- int) { // chan<- means "can only SEND"
    ch <- 42
}

func consumer(ch <-chan int) { // <-chan means "can only RECEIVE"
    value := <-ch
    fmt.Println(value)
}

func main() {
    ch := make(chan int) // bidirectional channel
    go producer(ch)
    consumer(ch)
}
```

If `producer` tries to receive, or `consumer` tries to send, it's a **compile error** — caught before your program even runs.

### Closing a channel

You can close a channel to signal "no more data will come":

```go
close(ch) // No more values will be sent on ch
```

Receivers can check if a channel is closed:

```go
value, ok := <-ch
if !ok {
    fmt.Println("Channel is closed!")
}
```

Or use a `for range` loop — it automatically stops when the channel closes:

```go
package main

import "fmt"

func sendNumbers(ch chan int) {
    for i := 1; i <= 5; i++ {
        ch <- i
    }
    close(ch) // Signal that we're done
}

func main() {
    ch := make(chan int)
    go sendNumbers(ch)

    for num := range ch { // Loops until channel is closed
        fmt.Println(num)
    }
}
```

**Output:**
```
1
2
3
4
5
```

> ⚠️ **Only the sender should close a channel.** Closing a channel that's already closed panics. Sending to a closed channel panics. Receiving from a closed channel is fine (you get zero values).

### ✅ When to use channels

- When goroutines need to send results back to the caller
- When goroutines need to communicate/signal each other
- When you want to stream data (one goroutine produces, another consumes)
- When you want built-in synchronization without explicit locks

### ❌ When NOT to use channels

- When simple shared state with a Mutex is cleaner (a counter, a map)
- When you just need to wait (use WaitGroup)
- Don't over-channel everything — sometimes simple is better

---

<a name="chapter-6"></a>
## Chapter 6 — Buffered Channels

### The problem with unbuffered channels

In Chapter 5, every send blocks until someone receives, and every receive blocks until someone sends. They must meet in the middle, like a perfect handshake.

But what if your producer is fast and you want it to queue up several values before a consumer picks them up?

### Buffered channels

A buffered channel has a **built-in queue**. The sender can put multiple values in without blocking — until the buffer is full.

```go
ch := make(chan int, 3) // Buffer of size 3
```

```
ch <- 1  // Doesn't block (buffer has room)
ch <- 2  // Doesn't block (buffer has room)
ch <- 3  // Doesn't block (buffer has room)
ch <- 4  // BLOCKS — buffer is full, waiting for receiver
```

### Unbuffered vs Buffered

```go
// Unbuffered: sender blocks until receiver is ready
ch := make(chan int)

// Buffered: sender only blocks when buffer is full
ch := make(chan int, 5)
```

### A clear example

```go
package main

import "fmt"

func main() {
    ch := make(chan string, 3) // Buffer of 3

    // All three sends happen immediately — no goroutine needed!
    ch <- "first"
    ch <- "second"
    ch <- "third"
    // ch <- "fourth" // This would BLOCK — buffer is full

    fmt.Println(<-ch) // "first"
    fmt.Println(<-ch) // "second"
    fmt.Println(<-ch) // "third"
}
```

Output:
```
first
second
third
```

### Real-world use: a job queue

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func worker(id int, jobs <-chan int, wg *sync.WaitGroup) {
    defer wg.Done()
    for job := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, job)
        time.Sleep(200 * time.Millisecond) // simulate work
    }
}

func main() {
    const numJobs = 10
    const numWorkers = 3

    jobs := make(chan int, numJobs) // Buffer holds all jobs
    var wg sync.WaitGroup

    // Start workers
    for w := 1; w <= numWorkers; w++ {
        wg.Add(1)
        go worker(w, jobs, &wg)
    }

    // Send all jobs into the buffered channel
    for j := 1; j <= numJobs; j++ {
        jobs <- j
    }
    close(jobs) // No more jobs — workers will stop when empty

    wg.Wait()
    fmt.Println("All jobs complete!")
}
```

This is the **worker pool pattern** — one of the most useful patterns in Go. We'll revisit it in Chapter 9.

### Checking buffer capacity

```go
ch := make(chan int, 5)
ch <- 1
ch <- 2

fmt.Println(len(ch)) // 2 — how many items are currently buffered
fmt.Println(cap(ch)) // 5 — total buffer capacity
```

### ✅ When to use buffered channels

- When the producer might be temporarily faster than the consumer
- When you want to limit how many goroutines can work at once (use buffer size = limit)
- For job queues where you want to pre-load all the work

### ❌ When NOT to use buffered channels

- Don't use a huge buffer to hide the fact that your producer and consumer are wildly different speeds — that's a design problem
- Don't use buffers to "avoid blocking" if blocking is actually the right behavior

---

<a name="chapter-7"></a>
## Chapter 7 — The `select` Statement

### The problem

You have multiple channels. You want to receive from **whichever one is ready first**. How?

You can't just do:
```go
v1 := <-ch1 // What if ch1 never sends? You're stuck forever
v2 := <-ch2
```

### Enter `select`

`select` is like a `switch`, but for channels. It waits for **any** of its cases to be ready, then runs that one.

```go
select {
case v := <-ch1:
    fmt.Println("Received from ch1:", v)
case v := <-ch2:
    fmt.Println("Received from ch2:", v)
}
```

Whichever channel has data first "wins". If both are ready, Go picks one randomly.

### A simple example

```go
package main

import (
    "fmt"
    "time"
)

func fastWorker(ch chan string) {
    time.Sleep(100 * time.Millisecond)
    ch <- "fast result"
}

func slowWorker(ch chan string) {
    time.Sleep(500 * time.Millisecond)
    ch <- "slow result"
}

func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    go fastWorker(ch1)
    go slowWorker(ch2)

    select {
    case result := <-ch1:
        fmt.Println("Winner:", result)
    case result := <-ch2:
        fmt.Println("Winner:", result)
    }
}
```

**Output:**
```
Winner: fast result
```

### The `default` case

If no channels are ready and you don't want to block, use `default`:

```go
select {
case v := <-ch:
    fmt.Println("Got:", v)
default:
    fmt.Println("Nothing ready, moving on...")
}
```

This is a **non-blocking receive** — it either gets data or immediately falls through to `default`.

### Timeouts with select

This is one of the most useful patterns in all of Go:

```go
package main

import (
    "fmt"
    "time"
)

func slowOperation(ch chan string) {
    time.Sleep(2 * time.Second)
    ch <- "result"
}

func main() {
    ch := make(chan string)
    go slowOperation(ch)

    select {
    case result := <-ch:
        fmt.Println("Got result:", result)
    case <-time.After(1 * time.Second): // Timeout after 1 second
        fmt.Println("Timed out! Operation took too long.")
    }
}
```

**Output:**
```
Timed out! Operation took too long.
```

`time.After(d)` returns a channel that receives a value after duration `d`. Pair it with `select` for dead-simple timeouts.

### Quit/Done channels (cancellation)

Another super common pattern — telling goroutines to stop:

```go
package main

import (
    "fmt"
    "time"
)

func worker(done chan bool) {
    fmt.Println("Worker: working...")
    for {
        select {
        case <-done:
            fmt.Println("Worker: received stop signal, exiting")
            return
        default:
            // Keep working
            time.Sleep(100 * time.Millisecond)
            fmt.Println("Worker: tick")
        }
    }
}

func main() {
    done := make(chan bool)
    go worker(done)

    time.Sleep(350 * time.Millisecond)
    done <- true // Signal the worker to stop

    time.Sleep(100 * time.Millisecond)
    fmt.Println("Main: done")
}
```

**Output:**
```
Worker: working...
Worker: tick
Worker: tick
Worker: tick
Worker: received stop signal, exiting
Main: done
```

### select in a loop

Often you use `select` inside an infinite loop to continuously process from multiple channels:

```go
for {
    select {
    case v := <-dataCh:
        process(v)
    case <-done:
        return
    }
}
```

### ✅ When to use select

- Waiting for the first of multiple channels to be ready
- Implementing timeouts
- Cancellation / done signals
- Non-blocking channel operations (with `default`)

---

<a name="chapter-8"></a>
## Chapter 8 — Mutex: Protecting Shared Data

### When channels aren't the right tool

Channels are great for communicating. But sometimes you just have **shared state** — a counter, a map, a cache — that multiple goroutines need to read and write.

For this, Go provides `sync.Mutex` (short for *mutual exclusion*).

### What a Mutex does

A Mutex is a lock. Only one goroutine can hold the lock at a time. Others must wait.

```go
var mu sync.Mutex

mu.Lock()   // "I'm taking the lock. Nobody else can lock until I unlock."
// ... access shared data safely ...
mu.Unlock() // "I'm done. Someone else can have the lock now."
```

### The broken counter (without Mutex)

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    counter := 0
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++ // RACE CONDITION!
        }()
    }

    wg.Wait()
    fmt.Println("Counter:", counter) // Probably NOT 1000
}
```

Run this with `go run -race main.go` and Go will warn you loudly about the race condition.

### The fixed counter (with Mutex)

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    counter := 0
    var mu sync.Mutex
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            mu.Lock()   // Lock before touching counter
            counter++
            mu.Unlock() // Unlock when done
        }()
    }

    wg.Wait()
    fmt.Println("Counter:", counter) // Always 1000 ✅
}
```

### Wrapping state in a struct (the clean way)

In real code, you bundle the Mutex with the data it protects:

```go
package main

import (
    "fmt"
    "sync"
)

// SafeCounter is a thread-safe counter
type SafeCounter struct {
    mu    sync.Mutex
    value int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}

func (c *SafeCounter) Get() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}

func main() {
    counter := &SafeCounter{}
    var wg sync.WaitGroup

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter.Increment()
        }()
    }

    wg.Wait()
    fmt.Println("Counter:", counter.Get()) // Always 1000 ✅
}
```

This is the pattern you'll see in production Go code. The Mutex "guards" the fields inside the struct.

### RWMutex: Multiple readers, one writer

Sometimes you have lots of reads and few writes. Using a regular Mutex means readers block each other unnecessarily. `sync.RWMutex` solves this:

- Multiple goroutines can **read** at the same time
- Only one goroutine can **write** at a time (and blocks all readers while writing)

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type SafeMap struct {
    mu   sync.RWMutex
    data map[string]string
}

func (m *SafeMap) Set(key, value string) {
    m.mu.Lock()         // Exclusive write lock
    defer m.mu.Unlock()
    m.data[key] = value
}

func (m *SafeMap) Get(key string) string {
    m.mu.RLock()         // Shared read lock — multiple readers OK
    defer m.mu.RUnlock()
    return m.data[key]
}

func main() {
    sm := &SafeMap{data: make(map[string]string)}
    
    // Writer
    go func() {
        for i := 0; i < 5; i++ {
            sm.Set("key", fmt.Sprintf("value-%d", i))
            time.Sleep(100 * time.Millisecond)
        }
    }()

    // Multiple readers
    var wg sync.WaitGroup
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for j := 0; j < 5; j++ {
                v := sm.Get("key")
                fmt.Printf("Reader %d got: %s\n", id, v)
                time.Sleep(80 * time.Millisecond)
            }
        }(i)
    }

    wg.Wait()
}
```

### sync.Once — run something exactly once

For one-time initialization (like loading a config or creating a singleton):

```go
package main

import (
    "fmt"
    "sync"
)

var (
    instance string
    once     sync.Once
)

func getInstance() string {
    once.Do(func() {
        fmt.Println("Initializing...") // Only runs once, ever
        instance = "I am the one and only instance"
    })
    return instance
}

func main() {
    var wg sync.WaitGroup
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            fmt.Println(getInstance())
        }()
    }
    wg.Wait()
}
```

Output:
```
Initializing...    ← only once
I am the one and only instance
I am the one and only instance
I am the one and only instance
I am the one and only instance
I am the one and only instance
```

### ✅ When to use Mutex

- Shared counters
- Shared maps or slices
- Any shared state that multiple goroutines read/write
- When channels would be more complex than helpful

### ❌ When NOT to use Mutex

- Don't reach for a Mutex before considering channels — channels are often cleaner
- Never hold a lock while doing something slow (I/O, network calls)
- Never lock a Mutex in one goroutine and unlock it in another

---

<a name="chapter-9"></a>
## Chapter 9 — Real Patterns You'll Actually Use

Now that you know the primitives, let's look at the **patterns** that appear again and again in Go codebases.

---

### Pattern 1: Pipeline

**What:** Process data in stages. Each stage is a goroutine. Data flows through channels like an assembly line.

**When to use:** Data transformation that can be broken into independent steps (e.g., read → process → write).

```go
package main

import "fmt"

// Stage 1: Generate numbers
func generate(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        for _, n := range nums {
            out <- n
        }
        close(out)
    }()
    return out
}

// Stage 2: Square each number
func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * n
        }
        close(out)
    }()
    return out
}

// Stage 3: Double each number
func double(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for n := range in {
            out <- n * 2
        }
        close(out)
    }()
    return out
}

func main() {
    // Chain the stages: generate → square → double
    numbers := generate(1, 2, 3, 4, 5)
    squared := square(numbers)
    doubled := double(squared)

    // Read from the final stage
    for result := range doubled {
        fmt.Println(result) // (1²×2), (2²×2), ... = 2, 8, 18, 32, 50
    }
}
```

**Output:**
```
2
8
18
32
50
```

Each function receives a channel, starts a goroutine to process it, and returns a new channel. You can chain them infinitely.

---

### Pattern 2: Fan-Out

**What:** Distribute work from one goroutine across many workers.

**When to use:** You have a stream of jobs that are slow and can be done independently (e.g., HTTP requests, image processing).

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

func fanOut(jobs <-chan int, numWorkers int) []<-chan int {
    channels := make([]<-chan int, numWorkers)
    
    for i := 0; i < numWorkers; i++ {
        workerID := i + 1
        out := make(chan int)
        channels[i] = out
        
        go func() {
            defer close(out)
            for job := range jobs {
                fmt.Printf("Worker %d processing job %d\n", workerID, job)
                time.Sleep(200 * time.Millisecond)
                out <- job * 10 // result
            }
        }()
    }
    
    return channels
}

func main() {
    jobs := make(chan int, 9)
    for i := 1; i <= 9; i++ {
        jobs <- i
    }
    close(jobs)

    results := fanOut(jobs, 3) // 3 workers

    var wg sync.WaitGroup
    for _, ch := range results {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                fmt.Printf("Result: %d\n", v)
            }
        }(ch)
    }
    wg.Wait()
}
```

---

### Pattern 3: Fan-In (Merge)

**What:** Combine multiple channels into one. The opposite of fan-out.

**When to use:** You have multiple producers (e.g., multiple workers) and want to collect all their results in one place.

```go
package main

import (
    "fmt"
    "sync"
)

// merge combines multiple channels into one
func merge(channels ...<-chan int) <-chan int {
    var wg sync.WaitGroup
    merged := make(chan int)

    // Start a goroutine for each input channel
    output := func(c <-chan int) {
        defer wg.Done()
        for v := range c {
            merged <- v
        }
    }

    wg.Add(len(channels))
    for _, c := range channels {
        go output(c)
    }

    // Close merged when all inputs are done
    go func() {
        wg.Wait()
        close(merged)
    }()

    return merged
}

func makeChannel(vals ...int) <-chan int {
    ch := make(chan int)
    go func() {
        for _, v := range vals {
            ch <- v
        }
        close(ch)
    }()
    return ch
}

func main() {
    ch1 := makeChannel(1, 2, 3)
    ch2 := makeChannel(4, 5, 6)
    ch3 := makeChannel(7, 8, 9)

    for v := range merge(ch1, ch2, ch3) {
        fmt.Println(v) // All values from all channels, in any order
    }
}
```

---

### Pattern 4: Worker Pool (The Most Common Pattern)

**What:** A fixed number of workers processing a stream of jobs.

**When to use:** You have many jobs but want to limit concurrency (e.g., don't open 10,000 connections at once).

```go
package main

import (
    "fmt"
    "sync"
    "time"
)

type Job struct {
    ID   int
    Data string
}

type Result struct {
    JobID  int
    Output string
}

func worker(id int, jobs <-chan Job, results chan<- Result, wg *sync.WaitGroup) {
    defer wg.Done()
    for job := range jobs {
        // Simulate work
        time.Sleep(100 * time.Millisecond)
        result := Result{
            JobID:  job.ID,
            Output: fmt.Sprintf("Processed '%s' by worker %d", job.Data, id),
        }
        results <- result
    }
}

func main() {
    const numWorkers = 3
    const numJobs = 10

    jobs := make(chan Job, numJobs)
    results := make(chan Result, numJobs)

    var wg sync.WaitGroup

    // Start workers
    for w := 1; w <= numWorkers; w++ {
        wg.Add(1)
        go worker(w, jobs, results, &wg)
    }

    // Send jobs
    for j := 1; j <= numJobs; j++ {
        jobs <- Job{ID: j, Data: fmt.Sprintf("task-%d", j)}
    }
    close(jobs) // No more jobs

    // Wait for workers, then close results
    go func() {
        wg.Wait()
        close(results)
    }()

    // Collect results
    for result := range results {
        fmt.Println(result.Output)
    }
}
```

This is the pattern to reach for when you think "I need to process these N things concurrently but not all at once."

---

### Pattern 5: Context for Cancellation (The Professional Way)

The `context` package is how real Go programs handle cancellation, timeouts, and deadlines. It's the grown-up version of our "done channel" from Chapter 7.

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func doWork(ctx context.Context, id int) {
    for {
        select {
        case <-ctx.Done():
            fmt.Printf("Worker %d: cancelled (%v)\n", id, ctx.Err())
            return
        default:
            fmt.Printf("Worker %d: working...\n", id)
            time.Sleep(200 * time.Millisecond)
        }
    }
}

func main() {
    // Cancel after 1 second
    ctx, cancel := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel() // Always call cancel to free resources!

    for i := 1; i <= 3; i++ {
        go doWork(ctx, i)
    }

    time.Sleep(1500 * time.Millisecond)
    fmt.Println("Main done")
}
```

**Output:**
```
Worker 1: working...
Worker 2: working...
Worker 3: working...
... (repeats for ~1 second)
Worker 1: cancelled (context deadline exceeded)
Worker 2: cancelled (context deadline exceeded)
Worker 3: cancelled (context deadline exceeded)
Main done
```

The three ways to create a context:

```go
ctx, cancel := context.WithCancel(context.Background())   // cancel manually
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second) // auto-cancel after 5s
ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(5*time.Second)) // cancel at specific time
```

Always `defer cancel()` to avoid resource leaks.

---

<a name="chapter-10"></a>
## Chapter 10 — Common Mistakes & How to Avoid Them

### Mistake 1: Goroutine Leak

A goroutine that never finishes is called a **leak**. It keeps consuming memory forever.

```go
// ❌ LEAK: nobody ever reads from ch, so this goroutine blocks forever
func leaky() {
    ch := make(chan int)
    go func() {
        ch <- 42 // blocks forever — nobody reads it
    }()
    // Function returns, but the goroutine is stuck
}
```

**Fix:** Always ensure goroutines have a way to exit. Use done channels or context:

```go
// ✅ Fixed: goroutine can exit via context
func notLeaky(ctx context.Context) {
    ch := make(chan int, 1) // buffered so send doesn't block
    go func() {
        select {
        case ch <- 42:
        case <-ctx.Done():
            return // Exit cleanly
        }
    }()
}
```

---

### Mistake 2: Closing a channel twice (panic!)

```go
ch := make(chan int)
close(ch)
close(ch) // PANIC: close of closed channel
```

**Fix:** Only close a channel once, and only from the sender. Use `sync.Once` if multiple goroutines might try to close it.

---

### Mistake 3: Sending to a closed channel (panic!)

```go
ch := make(chan int)
close(ch)
ch <- 1 // PANIC: send on closed channel
```

**Fix:** Only close after you're sure no more sends will happen. Usually close right after the last send or use a done channel to coordinate.

---

### Mistake 4: Loop variable capture bug

This is one of the most common bugs in Go and catches everyone:

```go
// ❌ BUG: all goroutines print the same value (the final i)
for i := 0; i < 5; i++ {
    go func() {
        fmt.Println(i) // captures i by reference!
    }()
}
// Likely prints: 5 5 5 5 5
```

Why? All goroutines share the same `i` variable. By the time they run, the loop has finished and `i` is 5.

```go
// ✅ Fix 1: pass i as a parameter
for i := 0; i < 5; i++ {
    go func(n int) {
        fmt.Println(n) // n is a copy
    }(i)
}

// ✅ Fix 2: create a local copy (Go 1.22+ fixes this automatically in for loops)
for i := 0; i < 5; i++ {
    i := i // shadow the variable with a new local copy
    go func() {
        fmt.Println(i)
    }()
}
```

> **Note:** Go 1.22 (released Feb 2024) fixed this behavior — loop variables are now per-iteration. But knowing why it happens is still important.

---

### Mistake 5: Forgetting `wg.Add` before the goroutine

```go
// ❌ RACE: wg.Wait() might return before the goroutine even starts
var wg sync.WaitGroup
go func() {
    wg.Add(1)     // Too late! Add must be BEFORE the goroutine
    defer wg.Done()
    // ...
}()
wg.Wait()
```

```go
// ✅ Always Add BEFORE launching the goroutine
var wg sync.WaitGroup
wg.Add(1)          // Add first
go func() {
    defer wg.Done()
    // ...
}()
wg.Wait()
```

---

### Mistake 6: Holding a lock while doing slow I/O

```go
// ❌ BAD: lock held during network call — blocks all other goroutines
mu.Lock()
result := makeHTTPRequest() // This could take seconds!
data[key] = result
mu.Unlock()
```

```go
// ✅ Good: do slow work without the lock, only lock for the final write
result := makeHTTPRequest() // No lock here
mu.Lock()
data[key] = result          // Lock only for the fast write
mu.Unlock()
```

---

### Mistake 7: Deadlock

A deadlock happens when goroutines are all waiting for each other and nobody can proceed.

```go
// DEADLOCK: ch1 waits for ch2, ch2 waits for ch1
ch1 := make(chan int)
ch2 := make(chan int)

go func() {
    v := <-ch1  // waits for ch1
    ch2 <- v
}()

v := <-ch2  // waits for ch2 — but ch2 waits for ch1, and ch1 has no sender!
ch1 <- 42
```

Go's runtime detects simple deadlocks and panics. The fix is to carefully think about the data flow and ensure there's always a goroutine that can make progress.

---

<a name="chapter-11"></a>
## Chapter 11 — Putting It All Together: A Mini Project

Let's build a **concurrent web scraper** that fetches multiple URLs simultaneously, processes results, and handles errors — all using what we've learned.

```go
package main

import (
    "context"
    "fmt"
    "math/rand"
    "sync"
    "time"
)

// Result holds the outcome of fetching a URL
type Result struct {
    URL      string
    Content  string
    Error    error
    Duration time.Duration
}

// fetch simulates fetching a URL (in real code, use net/http)
func fetch(ctx context.Context, url string) Result {
    start := time.Now()

    // Simulate variable network latency
    delay := time.Duration(rand.Intn(500)+100) * time.Millisecond

    select {
    case <-time.After(delay):
        // Success
        return Result{
            URL:      url,
            Content:  fmt.Sprintf("Content of %s (%dms)", url, delay.Milliseconds()),
            Duration: time.Since(start),
        }
    case <-ctx.Done():
        // Cancelled
        return Result{
            URL:   url,
            Error: fmt.Errorf("cancelled: %w", ctx.Err()),
        }
    }
}

// scrape fetches all URLs concurrently with a worker pool
func scrape(ctx context.Context, urls []string, numWorkers int) []Result {
    jobs := make(chan string, len(urls))
    results := make(chan Result, len(urls))

    var wg sync.WaitGroup

    // Start worker pool
    for w := 0; w < numWorkers; w++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for url := range jobs {
                result := fetch(ctx, url)
                results <- result
            }
        }()
    }

    // Send all URLs as jobs
    for _, url := range urls {
        jobs <- url
    }
    close(jobs)

    // Close results when all workers finish
    go func() {
        wg.Wait()
        close(results)
    }()

    // Collect all results
    var allResults []Result
    for result := range results {
        allResults = append(allResults, result)
    }
    return allResults
}

func main() {
    urls := []string{
        "https://example.com/page1",
        "https://example.com/page2",
        "https://example.com/page3",
        "https://example.com/page4",
        "https://example.com/page5",
        "https://example.com/page6",
        "https://example.com/page7",
        "https://example.com/page8",
    }

    // Allow max 3 seconds total
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    fmt.Printf("Scraping %d URLs with 3 workers...\n\n", len(urls))
    start := time.Now()

    results := scrape(ctx, urls, 3)

    elapsed := time.Since(start)

    // Report results
    var successes, failures int
    for _, r := range results {
        if r.Error != nil {
            fmt.Printf("❌ %s — Error: %v\n", r.URL, r.Error)
            failures++
        } else {
            fmt.Printf("✅ %s — %v — %s\n", r.URL, r.Duration, r.Content)
            successes++
        }
    }

    fmt.Printf("\n---\n")
    fmt.Printf("Total time:  %v\n", elapsed)
    fmt.Printf("Successes:   %d\n", successes)
    fmt.Printf("Failures:    %d\n", failures)
    fmt.Printf("\nWithout concurrency this would have taken: ~%v\n",
        time.Duration(len(urls))*300*time.Millisecond)
}
```

**This mini-project uses:**
- ✅ Goroutines (workers)
- ✅ Channels (jobs queue, results collection)
- ✅ Buffered channels (job buffer)
- ✅ WaitGroup (waiting for all workers)
- ✅ Context (timeout/cancellation)
- ✅ Worker pool pattern
- ✅ Fan-in (collecting all results)

---

<a name="cheatsheet"></a>
## Quick Reference Cheat Sheet

### Goroutines

```go
go myFunc()                    // Launch a goroutine
go func() { ... }()            // Anonymous goroutine
```

### WaitGroup

```go
var wg sync.WaitGroup
wg.Add(1)                      // Before launching goroutine
go func() {
    defer wg.Done()            // Call when goroutine finishes
    // work...
}()
wg.Wait()                      // Block until count = 0
```

### Channels

```go
ch := make(chan int)            // Unbuffered
ch := make(chan int, 5)        // Buffered (capacity 5)
ch <- value                    // Send (blocks if full/no receiver)
value := <-ch                  // Receive (blocks if empty/no sender)
value, ok := <-ch              // Receive with closed check
close(ch)                      // Close (sender only!)
for v := range ch { }         // Loop until closed
```

### Directional Channels

```go
func sender(ch chan<- int)     // Send-only
func receiver(ch <-chan int)   // Receive-only
```

### Select

```go
select {
case v := <-ch1:               // From ch1
case ch2 <- val:               // Send to ch2
case v := <-time.After(1*time.Second): // Timeout
default:                       // Non-blocking fallback
}
```

### Mutex

```go
var mu sync.Mutex
mu.Lock()
defer mu.Unlock()              // Always defer!
// access shared data
```

### RWMutex

```go
var mu sync.RWMutex
mu.RLock(); defer mu.RUnlock() // For reads
mu.Lock();  defer mu.Unlock()  // For writes
```

### Context

```go
ctx, cancel := context.WithCancel(context.Background())
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
<-ctx.Done()                   // Wait for cancellation
ctx.Err()                      // Why it was cancelled
```

---

## Closing Words

Concurrency in Go is one of the most elegant designs in any programming language — but it takes time to click. Here's the order to think in when writing concurrent code:

1. **Start simple.** Does this really need to be concurrent?
2. **Use goroutines** for independent tasks.
3. **Use channels** to communicate between goroutines.
4. **Use WaitGroup** to wait for goroutines to finish.
5. **Use Mutex** to protect shared state that isn't flowing through channels.
6. **Use context** for cancellation and timeouts in real applications.
7. **Always run** `go run -race` to catch race conditions.

