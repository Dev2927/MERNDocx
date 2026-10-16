*Q1. What is Node.js and how does it work?*

Node.js is an open-source, cross-platform JavaScript Runtime Environment. Node.js is a software program that can execute JavaScript code. Put more properly, Node.js is a JavaScript runtime environment. It is an environment developed to make it possible to use JavaScript code for server-side scripting.

How Does Node.js Work

Node.js was written mostly with C/C++. As a program that is supposed to run web servers, Node.js needs to constantly interact with a device's operating system. Building Node.js with a low-level language like C made it easy for the software to access the operating system’s resources and use them to execute instructions. There are three main components we must understand to see how Node.js works. These components are:

1. V8 Engine: 

The V8 Engine is the JavaScript engine that interprets and runs JavaScript code in the Chrome browser. Some other browsers use a different engine, for example, Firefox uses SpiderMonkey, and Safari uses JavaScriptCore. Without the JavaScript engine, a computer can not understand JavaScript. The V8 engine contains a memory heap and call stack. They are the building blocks for the V8 engine. They help manage the execution of JavaScript code. The memory heap is the data store of the V8 engine. Whenever we create a variable that holds an object or function in JavaScript, the engine saves that value in the memory heap. To keep things simple, it is similar to a backpack that stores supplies for a hiker.
Whenever the engine is executing code and comes across any of those variables, it looks up the actual value from the memory heap – just like whenever a hiker is feeling cold and wants to start a fire, they can look into their backpack for a lighter.

The call stack is another building block in the V8 engine. It is a data structure that manages the order of functions to be executed. Whenever the program invokes a function, the function is placed on the call stack and can only leave the stack when the engine has handled that function. JavaScript is a single-threaded language, which means that it can only execute one instruction at a time. Since the call stack contains the order of instructions to be executed, it means that the JavaScript engine has just one order, one call stack.

![alt text](image.png)

2. Libuv:

Libuv is a C library used for performing Input/output (I/O) operations. I/O operations have to do with sending requests to the computer and receiving responses. These operations include reading and writing files, making network requests, and so on. This means that Libuv is cross-platform (can run on any operating system) and has a focus on Asynchronous I/O. The computer tends to take time to process I/O instructions, It can handle more than one I/O operation at once. This is what makes Node.js process I/O instructions efficiently despite being single-threaded. Whenever we pass a script to Node.js, the engine parses the code and starts processing it. The call stack holds the invoked functions and keeps track of the program. If the V8 engine comes across an I/O operation, it passes that operation over to Libuv. Libuv then executes the I/O operation. There are bindings that connect JavaScript functions to their actual implementation in Libuv. These bindings make it possible to use JavaScript code for I/O instructions. Node.js uses Libuv for the actual implementation but exposes Application Programming Interfaces (APIs). So, we can now use a Node.js API (which looks like a JavaScript function) to initiate an I/O operation. One interesting thing to note is that it is true that JavaScript is a single-threaded language, but Libuv—the low-level library Node.js uses— can make use of a thread pool (multiple threads) when executing instructions in the operating system. 

![alt text](image-1.png)

3. Event Loop:

The Event Loop in Node.js is a very important part of the process. From the name, we can see it is a loop. The loop starts running as Node.js begins executing a program. When we run our JavaScript program that contains some asynchronous code (like I/O instructions or timer-based actions), Node.js handles them using the Node.js APIs. Asynchronous functions usually have instructions to be executed after the function has finished processing. Those instructions are placed in a Callback Queue. The Callback Queue works with the First In First Out (FIFO) approach. That means the first instruction (callback) to enter the queue is the first to be invoked. As the event loop runs, it checks if the call stack is empty. If the call stack is not empty, it allows the ongoing process to continue. But if the call stack is empty, it sends the first instruction on the callback queue to the JavaScript engine. The engine then places that instruction (function) on the call stack and executes it. So, the event loop executes callbacks from asynchronous instructions using the JavaScript V8 engine in Node.js. And it is a loop, which means every time it runs, it checks the call stack to know if it will remove the foremost callback and send it to the JavaScript engine. Node.js is said to have an event-driven architecture. This means Node.js is built around listening to events and reacting to them promptly when they happen. These events can be timer events, network events, and so on. Node.js responds to those events by using an event loop to load event callbacks to the engine after something triggers an event. It is for this reason that Node.js is excellent for real-time data transfer in applications.

![alt text](image-2.png)

*Q2. Explain the Node.js event-driven architecture.*

NodeJS uses an event-driven architecture, which is a key part of how it handles many tasks at once without blocking. This approach relies on events, event emitters, and listeners to manage asynchronous operations efficiently. It is a design pattern where applications respond to events—changes in state, user input, or messages from other programs. Instead of a program's logic being tied to a specific sequence of steps, it reacts to events as they occur.This makes applications more flexible and responsive.

Event-Driven Architecture Working
NodeJS is inherently event-driven. The main focus is on a single-threaded event loop, which efficiently manages asynchronous operations. Here's how it works

- Events: Things that happen in your application, like a user clicking a button, a file being read, or a network request completing, are represented as "events."

- Event Loop: NodeJS has a central "event loop" that constantly monitors these events. It's like a traffic controller for your application.

- Callbacks: When an event occurs, NodeJS executes a corresponding "callback" function.This function contains the code that should run in response to that specific event.

- Non-Blocking: The event loop is non-blocking. This means that while waiting for an event (like a file to finish reading), NodeJS can handle other tasks.This is what makes NodeJS so efficient at handling many things at once.

- Asynchronous Operations: Many operations in NodeJS are asynchronous. For example, reading a file doesn't stop the rest of your program. Instead, NodeJS registers a callback function to be executed when the file is read.

*Q3. What is the difference between process.nextTick() and setImmediate()?*

process.nextTick() runs code immediately after the current operation, before I/O tasks. setImmediate() schedules code to run after the current event loop phase, following I/O tasks, impacting execution timing.

- process.nextTick() method

Whenever a new queue of operations is initialized we can think of it as a new tick. The process.nextTick() method adds the callback function to the start of the next event queue. It is to be noted that, at the start of the program process.nextTick() method is called for the first time before the event loop is processed.

Syntax: process.nextTick(callback);

- setImmediate() method

Whenever we call setImmediate() method, it's callback function is placed in the check phase of the next event queue. There is slight detail to be noted here that setImmediate() method is called in the poll phase and it's callback functions are invoked in the check phase.

Syntax: setImmediate(callback);

Example: This example illustrates that process.nextTick() callbacks execute before setImmediate() callbacks, even though both are scheduled in the same event loop iteration.

setImmediate(function A() {
	console.log("1st immediate");
});

setImmediate(function B() {
	console.log("2nd immediate");
});

process.nextTick(function C() {
	console.log("1st process");
});

process.nextTick(function D() {
	console.log("2nd process");
});

// First event queue ends here
console.log("program started");

Explanation:

For the above program, event queues are initialized in the following manner:

- In the first event queue only 'program started is printed'.

- Then second event queue is started and function C i.e. callback of process.nextTick() method is placed at the start of the event queue. C is executed and the queue ends.

- Then previous event queue ends and third event queue is initialized with callback D. Then callback function A of setImmdeiate() method is placed in the followed by B. Now, the third event queue looks like this - D A B - Now functions D, A, B are executed in the order they are present in the queue.

![alt text](image-3.png)

*Q3. How does the Node.js event loop differ from browser?*

Event Loop in Browser

![alt text](image-4.png)

1. Heap - It stores all the object reference and variables that we define in our function.

2. Call Stack - All the function that we use in our code is stacked here in LIFO manner such that the last function is at the top and first function is at the bottom.

3. Web API's - These API's are provided by browser which provides additional functionality over V8 engine. The functions which uses these API's are pushed to this container which on completion of the response of Web API is popped out of this container.

4. Queues - The queues are used to compute the asynchronous code response such that it doesn't block engine to execute further.

- Macro Task Queue - This queue executes async functions like DOM events, Ajax calls and setTimeout and has lower priority than Job queue.

- Micro Task Queue - This queue executes async functions which uses promises and has higher precedence over Message Queue.

The event loop checks the Call Stack, if the stack is empty it pushes the functions in the Queues to Call Stack and runs it. Functions already present are given higher priority and runs first in comparison to functions in message queue.

Event Loop in Node Js

![alt text](image-5.png)

1. Event Queue - On completion of the Thread Pool a callback function is issued and sent to the event queue. When call stack is empty the event goes through the event queue and sends callback to the call stack.

2. Thread Pool - The thread pool is composed of 4 threads which delegates operations that are too heavy for the event loop. I/O operations, Opening and closing connections, setTimeouts are the example of such operations.

3. Event loop in Node Js has different phases which has FIFO queue of callbacks to execute. When event loop enters a given phase it operates callbacks in that phase queue until the queue has been exhausted and maximum number of callbacks has executed and then moves to next phase.

The Event Loop is an endless loop which waits for the tasks, executes them and then sleeps until it receives more tasks. The event loop executes tasks from queue only when stack is empty. It processes the oldest task first and allows us to use callbacks and promises.

Difference between both the event loops

1. First difference is node uses a thread pool to manage disk I/O. It executes the I/O and other timer API's asynchronously.

2. Browser does not have setImmediate() function. This function execute once the I/O operation is done, if particular code is inside this it will be executed first. Whereas in setTimeout() callback function is executed after given minimum threshold value in milliseconds.

3. Node Js event loop has multiple phases and each phase handle specific type of tasks whereas browser has micro task and macro task queue within which all the tasks are processed in order they were placed into the queue.

*Q5. What are streams in Node.js?*

Streams are collections of data — just like arrays or strings. The difference is that streams might not be available all at once, and they don’t have to fit in memory. This makes streams really powerful when working with large amounts of data, or data that’s coming from an external source one chunk at a time.

However, streams are not only about working with big data. They also give us the power of composability in our code. Just like we can compose powerful linux commands by piping other smaller Linux commands, we can do exactly the same in Node with streams.

*Q6. Explain readable and writable streams.*

Readable

- To add data to a readable stream, you use the .push() function. When the stream is finished, you push(null).

- When ended, readable streams fire the 'end' event.

- You can read data from a readable stream by listening for the 'readable' event and then executing 'read()' until it returns null.

- Readable streams have a buffer, meaning that when you 'push()' to a buffer, if the buffer is full, then push() will return false. However, you can continue pushing to the buffer and filling it even though it's full. The 'highWaterMark' (or the buffer size) is really informational.

- Readable streams implement a _read() method to pull data from a non-stream source. You don't have to use this, however. You can just leave this method blank and use the push method described earlier. Whoever is using your stream could call read(), which would first read from the internal buffer, and then go call _read() when the buffer is empty.

Writable

- To add data to a writable stream, you use the .write() function. When the stream is finished, you use .end().

- When you call .end(), it does NOT end the stream immediately. It will use process.nextTick() to end the stream on the next tick! This has caused many race condition heartaches for me.

- Writable streams have a buffer. If the buffer is full (highWaterMark), then it will return false when you call .write(). However, you can keep writing to it and ignore this event if you want. Otherwise, I think there's something like a 'drain' event that notifies you that you can continue writing.

- Writable streams implement a _write() method to send the data to some back-end non-stream sink. If this method returns false, then the Writable stream will start buffering data and not call _write() again until 'drain'.

*Q7. What is middleware in Express?*

Middleware in Express.js are functions that run during the request–response lifecycle to process requests, modify responses, and control application flow.

- Executes custom logic for each request.
- Can read or modify req and res.
- May send a response and end the cycle.
- Uses next() to pass control to the next middleware.

![alt text](image-6.png)

Syntax

app.use((req, res, next) => {
    console.log('Middleware executed');
    next();
});

- (req, res, next) => {}: Middleware function to process the request and response before the final handler.
- next(): Passes control to the next middleware if the request–response cycle isn’t ended.

Working of Middleware in Express.js

In Express.js, middleware functions are executed sequentially in the order they are added to the application.

1. Request arrives at the server.
2. Middleware functions are applied to the request, one by one.
3. Each middleware either sends a response or passes control using next().
4. If no middleware ends the cycle, the route handler is reached, and a final response is sent.

Types of Middleware

1. Application-level Middleware

Application-level middleware runs across the entire Express application, handling common logic for all incoming requests.

- Registered using app.use() or app.METHOD().
- Executes for all routes and HTTP methods.
- Commonly used for logging, authentication, body parsing, and headers.

app.use(express.json()); // Parses JSON data for every incoming request
app.use((req, res, next) => {
  console.log('Request received:', req.method, req.url);
  next();
});

2. Router-level Middleware

Router-level middleware applies to a specific router instance, allowing middleware logic to be scoped to a defined group of routes.

- Registered using router.use() or router.METHOD().
- Executes only for routes within that router.
- Ideal for modular route grouping (e.g., auth or user routes).
- Improves code organization and maintainability by isolating middleware to related routes.

const router = express.Router();

// Apply middleware to only this router's routes
router.use((req, res, next) => {
  console.log('Router-specific middleware');
  next();
});

router.get('/dashboard', (req, res) => {
  res.send('Dashboard Page');
});

app.use('/user', router); // The middleware applies only to routes under "/user"

3. Built-in Middleware

Express offers built-in middleware functions to handle common server tasks efficiently.

- express.static() serves static files like images, CSS, and JS.
- express.json() parses incoming JSON request bodies.

app.use(express.static('public')); // Serves static files from the "public" folder
app.use(express.json()); // Parses JSON payloads in incoming requests

Middleware Chaining

Express allows middleware functions to be chained and executed sequentially, processing a request step by step before sending a response.

- Middleware runs in order as a chain.
- Each middleware can process or modify the request.
- The final handler sends the response to the client.
- next() passes control to the next middleware in the chain.

![alt text](image-7.png)

*Q7. Blocking the event loop. Which operations freeze all requests (sync fs, JSON.parse on huge payloads, heavy loops, catastrophic regex, bcrypt sync)? How do you detect event loop lag (perf_hooks.monitorEventLoopDelay, clinic.js), and how do you fix it with worker threads, child processes, chunking or moving work to a queue?*

Node.js uses an event-driven architecture in which JavaScript callbacks normally execute on a single main thread per process. This allows one process to handle many concurrent connections efficiently, but it also creates a critical constraint: **if one callback performs expensive synchronous work, other requests handled by that event loop must wait**.

My approach is to keep request handlers short, measure event loop delay, identify the blocking operation, and move CPU-intensive or long-running work to an appropriate execution mechanism.

## 1. Which operations block the event loop?

### A. Synchronous filesystem operations

```javascript
import fs from 'node:fs';

app.get('/report', (req, res) => {
  const data = fs.readFileSync('./large-report.json', 'utf8');
  res.json(JSON.parse(data));
});
```

`readFileSync()` blocks the JavaScript thread until the file read completes. Every request relying on that event loop waits during the operation.

**Fix:** use asynchronous filesystem APIs.

```javascript
import fs from 'node:fs/promises';

app.get('/report', async (req, res, next) => {
  try {
    const data = await fs.readFile('./large-report.json', 'utf8');
    res.json(JSON.parse(data));
  } catch (error) {
    next(error);
  }
});
```

The asynchronous read allows the event loop to handle other work while waiting for the file. However, `JSON.parse()` is still synchronous and can block if the document is huge.

### B. Large `JSON.parse()` and `JSON.stringify()` operations

```javascript
const parsed = JSON.parse(hugePayload);
```

Parsing a large JSON document consumes CPU and may allocate substantial memory. The event loop cannot process another callback until parsing finishes.

The same concern applies to serializing enormous objects with `JSON.stringify()`.

**Fixes:**
- Enforce request-body size limits.
- Paginate API responses instead of serializing huge collections.
- Avoid repeatedly parsing or serializing the same data.
- Use streaming parsers when the data format and processing requirements support streaming.
- Move genuinely expensive parsing or transformation to a worker thread or background job.

Changing the code to `await Promise.resolve()` before calling `JSON.parse()` does not make parsing asynchronous. The parsing operation still runs synchronously on the main thread.

### C. CPU-intensive loops and calculations

```javascript
app.get('/calculate', (req, res) => {
  let total = 0;

  for (let i = 0; i < 1_000_000_000; i++) {
    total += i;
  }

  res.json({ total });
});
```

The loop monopolizes the JavaScript thread until it finishes.

The issue is not simply the number of lines of code. It is the amount of computation performed before the event loop gets another opportunity to run.

**Fixes:** optimize the algorithm, bound the input, divide work into chunks, or move CPU-intensive calculations to worker threads.

### D. Catastrophic regular expressions (ReDoS)

Some regular expressions have pathological backtracking behavior on crafted inputs. Their execution time can grow exponentially or otherwise become disproportionately large.

For example, a vulnerable pattern with nested repetition may perform badly on a long string that almost matches.

**Fixes:**
- Avoid ambiguous nested quantifiers and other patterns with dangerous backtracking behavior.
- Validate and limit input lengths.
- Use simpler parsing logic or a regex engine with appropriate complexity guarantees.
- Consider a worker or isolated process for untrusted, potentially expensive matching, with a timeout and resource limits.

A timeout implemented using `setTimeout()` on the same blocked event loop cannot interrupt a running synchronous regex. The callback cannot run until the regex returns.

### E. Synchronous password hashing and cryptography

```javascript
import bcrypt from 'bcrypt';

const hash = bcrypt.hashSync(password, 12);
```

Synchronous bcrypt keeps the main JavaScript thread occupied while hashing. Doing this during login or registration can delay unrelated requests.

Use the asynchronous API instead:

```javascript
const hash = await bcrypt.hash(password, 12);
const valid = await bcrypt.compare(password, storedHash);
```

This avoids blocking the JavaScript event loop while the hashing operation runs. However, expensive cryptographic work still consumes system resources and may compete for a native worker pool, depending on the library implementation. I limit concurrency and monitor latency under realistic authentication load.

## 2. What about `async` and `await`?

`async` and `await` do not automatically make CPU-intensive operations non-blocking.

```javascript
app.get('/bad', async (req, res) => {
  const data = JSON.parse(hugePayload);
  res.json(data);
});
```

The `JSON.parse()` call still blocks.

Compare that with:

```javascript
const data = await fs.readFile('./data.json', 'utf8');
```

Here, the asynchronous filesystem operation allows other event-loop work to proceed while the file is being read. Once the read completes, the continuation runs on the JavaScript thread, and parsing remains synchronous if performed there.

A useful distinction is:

- **Asynchronous I/O:** waits without monopolizing the JavaScript thread.
- **CPU-intensive JavaScript:** runs on the executing thread unless explicitly moved elsewhere.
- **Native asynchronous operations:** may use operating-system facilities or libuv's worker pool, depending on the API.

Even asynchronous operations can overload the worker pool or exhaust memory if too many expensive jobs are submitted concurrently.

## 3. Detect event loop lag with `monitorEventLoopDelay()`

Node.js provides `monitorEventLoopDelay()` in the `node:perf_hooks` module. It measures delays in event-loop scheduling, with histogram values expressed in nanoseconds. <Cite refs={["turn691641search1","turn691641search4"]}/>

Example:

```javascript
import { monitorEventLoopDelay } from 'node:perf_hooks';

const histogram = monitorEventLoopDelay({
  resolution: 20
});

histogram.enable();

setInterval(() => {
  console.log({
    meanMs: histogram.mean / 1e6,
    p50Ms: histogram.percentile(50) / 1e6,
    p99Ms: histogram.percentile(99) / 1e6,
    maxMs: histogram.max / 1e6
  });

  histogram.reset();
}, 10_000).unref();
```

The histogram uses nanoseconds, so dividing by \(10^6\) converts the values to milliseconds.

I pay particular attention to **p99 event loop delay**, because averages can hide brief but severe stalls. I establish service-specific thresholds based on the application's latency objectives rather than treating one threshold as universally correct.

For example, repeated event loop delays of hundreds of milliseconds would be concerning for a latency-sensitive API.

I also monitor:
- Request latency percentiles, especially p95 and p99.
- Request throughput and error rates.
- Process CPU utilization and memory usage.
- Garbage-collection activity.
- Event Loop Utilization (ELU).
- Worker-pool contention where relevant.

Event loop delay is not the same as request latency: a slow database query can make a request slow without necessarily blocking the event loop.

### Event Loop Utilization

ELU helps quantify how much time the event loop spends active rather than idle.

```javascript
import { performance } from 'node:perf_hooks';

let previous = performance.eventLoopUtilization();

setInterval(() => {
  const current = performance.eventLoopUtilization(previous);
  previous = performance.eventLoopUtilization();

  console.log({
    utilization: current.utilization,
    activeMs: current.active,
    idleMs: current.idle
  });
}, 10_000).unref();
```

The utilization value is between `0` and `1`. High utilization combined with high event loop delay suggests that the loop is struggling to keep up, but I correlate these metrics with CPU and application traces before deciding on a fix.

## 4. Use Clinic.js and profiling tools

When metrics reveal a problem but not its cause, I profile the application under a representative workload.

Clinic.js provides profiling tools, including Clinic Flame, for examining where CPU time is spent. I can also use Node.js CPU profiles and flame graphs to identify expensive functions.

My workflow is:

1. Reproduce the slowdown with a controlled workload.
2. Capture CPU profiles while the problem occurs.
3. Identify hot functions such as JSON parsing, regex matching, serialization, or repeated calculations.
4. Inspect allocation and garbage-collection behavior if memory pressure is involved.
5. Fix the algorithm or execution model.
6. Repeat the same workload and compare latency, throughput, and event loop delay.

A CPU profile helps identify *where* time is spent. Event loop delay shows *when the event loop is not getting timely opportunities to run*. Together, they provide a much stronger diagnosis than looking at CPU utilization alone.

## 5. Choose the right fix

<box border radius="lg" padding={3} gap={3}>
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}>
      <icon name="file-check" size="xl" />
    </box>
    <box flex="1" gap={1}>
      **Asynchronous I/O**

      <text color="secondary" size="sm">Best for filesystem, network, and database operations that have asynchronous APIs.</text>

      Use `fs/promises`, asynchronous database clients, and non-blocking network operations.
    </box>
  </row>
  <divider color="subtle" />
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}>
      <icon name="cpu" size="xl" />
    </box>
    <box flex="1" gap={1}>
      **Worker threads**

      <text color="secondary" size="sm">Best for CPU-intensive JavaScript or suitable CPU-heavy native work.</text>

      Use a bounded worker pool for parsing, transformations, image processing, or calculations that justify the overhead.
    </box>
  </row>
  <divider color="subtle" />
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}>
      <icon name="terminal" size="xl" />
    </box>
    <box flex="1" gap={1}>
      **Child processes**

      <text color="secondary" size="sm">Best for process isolation or external executables.</text>

      Run converters, command-line tools, or untrusted computation in a separately managed process with resource limits.
    </box>
  </row>
  <divider color="subtle" />
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}>
      <icon name="layers" size="xl" />
    </box>
    <box flex="1" gap={1}>
      **Chunking and yielding**

      <text color="secondary" size="sm">Best when a task can be divided into small, independent steps.</text>

      Process bounded batches and yield to the event loop between batches.
    </box>
  </row>
  <divider color="subtle" />
  <row align="start" gap={3}>
    <box background="surface-secondary" radius="md" padding={2}>
      <icon name="list-ordered" size="xl" />
    </box>
    <box flex="1" gap={1}>
      **Background queue**

      <text color="secondary" size="sm">Best for long-running jobs that do not need to finish during the HTTP request.</text>

      Use a queue such as BullMQ with Redis, or a durable message broker, to process work asynchronously.
    </box>
  </row>
</box>

## 6. Worker threads: move CPU-intensive work off the main thread

Worker threads run JavaScript in separate threads and can execute CPU-intensive work without monopolizing the main event loop.

A simplified worker example:

**`worker.js`**

```javascript
import { parentPort, workerData } from 'node:worker_threads';

function calculate(limit) {
  let total = 0;

  for (let i = 0; i < limit; i++) {
    total += i;
  }

  return total;
}

parentPort.postMessage(calculate(workerData.limit));
```

**Main thread**

```javascript
import { Worker } from 'node:worker_threads';

function runCalculation(limit) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(
      new URL('./worker.js', import.meta.url),
      { workerData: { limit } }
    );

    worker.once('message', resolve);
    worker.once('error', reject);

    worker.once('exit', (code) => {
      if (code !== 0) {
        reject(new Error(`Worker exited with code ${code}`));
      }
    });
  });
}
```

A request handler can await `runCalculation()`. While the worker calculates, the main event loop can continue handling other requests.

This example creates a worker per job for clarity, but in production I would usually use a **bounded worker pool** to avoid the overhead of creating many threads. I would also implement timeouts, worker replacement after failures, concurrency limits, and graceful shutdown.

Worker threads are not automatically faster for small tasks. Transferring large objects between threads can introduce copying, serialization, and memory overhead; transferable buffers or shared memory may help in suitable cases.

## 7. Chunking: split work into bounded batches

If work can be processed incrementally, chunking lets the event loop handle other callbacks between batches.

```javascript
import { setImmediate as yieldToEventLoop } from 'node:timers/promises';

async function processItems(items, batchSize = 500) {
  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize);

    for (const item of batch) {
      processItem(item);
    }

    await yieldToEventLoop();
  }
}
```

Here, `processItem()` represents synchronous work that is individually cheap. The batch size should be chosen based on measured execution time, not just the number of items.

The `setImmediate` yield allows other event-loop callbacks to run between batches. It does not make any individual `processItem()` call asynchronous, and it does not help if a single item requires an enormous synchronous calculation.

For large workloads, chunking can also require progress tracking, cancellation, and careful handling of partial failures.

## 8. Move long-running work to a queue

Suppose a SaaS application needs to generate a large monthly report, import thousands of records, or process uploaded documents. I would not keep the HTTP request open while doing all that work.

Instead:

1. Validate the request and persist a job record.
2. Enqueue the job with a durable identifier.
3. Return `202 Accepted` with the job ID.
4. Let a background worker process the job.
5. Expose status and result endpoints.
6. Implement retries, idempotency, failure handling, and concurrency limits.

Example response:

```json
{
  "jobId": "job_12345",
  "status": "queued"
}
```

This improves request responsiveness and lets background capacity scale separately from API instances. It does not eliminate CPU work; it moves that work into a controlled execution environment.

For reliability, I avoid relying on an in-memory queue if jobs must survive process restarts. A durable queue and appropriate persistence strategy are preferable.

## 9. A practical production decision process

When an API's p99 latency suddenly increases, I would:

1. Check event loop delay and ELU alongside request latency and CPU metrics.
2. Inspect a CPU profile to identify expensive synchronous functions.
3. Determine whether the problem is CPU computation, synchronous I/O, memory pressure, or worker-pool saturation.
4. Bound inputs and fix inefficient algorithms before adding infrastructure.
5. Choose asynchronous I/O, worker threads, chunking, child processes, or a background queue based on the workload.
6. Apply backpressure and concurrency limits so that moving work off the event loop does not overload another resource.
7. Load-test the fix and verify that event loop delay and request latency improve under realistic concurrency.

**Interview summary:** Node.js handles many requests efficiently when each event-loop callback does a small amount of work. Synchronous I/O, CPU-heavy JavaScript, large JSON operations, pathological regexes, and synchronous cryptography can block all requests sharing that event loop. I detect the impact with event-loop delay metrics and profiling, then remove the bottleneck or isolate the workload in a worker, process, or background queue.

*Q8. Streams and backpressure. Explain the four stream types, pipe vs pipeline, and how backpressure works (write() returning false, the drain event, highWaterMark). How do you stream a large file or CSV upload to S3 without loading it into memory, and why is stream/promises pipeline the safer choice for error handling?*

## 1. What are streams in Node.js?

Streams allow us to process data **incrementally, in chunks, rather than loading the entire dataset into memory**. They are useful for large files, HTTP requests and responses, CSV processing, compression, and uploading data to cloud storage such as Amazon S3.

Node.js has four stream types:

| Stream type | Purpose | Example |
|---|---|---|
| **Readable** | Produces data that can be consumed | `fs.createReadStream()`, HTTP request |
| **Writable** | Accepts data to be written | `fs.createWriteStream()`, HTTP response |
| **Duplex** | Supports both reading and writing independently | TCP socket |
| **Transform** | A Duplex stream that transforms data as it passes through | `zlib.createGzip()`, encryption or CSV transformation |

Example:

```javascript
import fs from 'node:fs';
import zlib from 'node:zlib';
import { pipeline } from 'node:stream/promises';

await pipeline(
  fs.createReadStream('large.log'),
  zlib.createGzip(),
  fs.createWriteStream('large.log.gz')
);
```

Here, the file is read in chunks, compressed incrementally, and written to another file. The entire file does not need to reside in memory.

## 2. What is backpressure?

**Backpressure is the mechanism that prevents a fast data producer from overwhelming a slower consumer.**

For example, suppose a file is read at 100 MB/s, but the network can upload data at only 10 MB/s. Without backpressure, the application could accumulate data in memory until memory usage becomes excessive.

Node.js streams manage this through internal buffers and signals.

### How `write()`, `drain`, and `highWaterMark` work

Consider a Writable stream:

```javascript
import fs from 'node:fs';

const writable = fs.createWriteStream('output.txt');

writable.on('drain', () => {
  console.log('Writable buffer can accept more data');
});

writable.write('Some data');
```

The key behavior is:

- `write(chunk)` returns `true` when the stream can continue accepting data without requiring the producer to pause.
- It returns `false` when the internal buffer has reached its backpressure threshold. The producer should stop writing temporarily.
- The `drain` event indicates that the buffer has emptied sufficiently for writing to resume.
- `highWaterMark` is the buffer threshold, measured in bytes for ordinary byte streams or objects for object-mode streams.

For example:

```javascript
import fs from 'node:fs';

const writable = fs.createWriteStream('output.txt', {
  highWaterMark: 64 * 1024
});
```

This configures a 64 KiB threshold. It is **not a hard memory limit**: a chunk can exceed the threshold, and other stream buffers and application allocations also consume memory.

### Correct manual backpressure handling

```javascript
import fs from 'node:fs';

const writable = fs.createWriteStream('output.txt');

async function writeChunk(chunk) {
  if (!writable.write(chunk)) {
    await new Promise((resolve, reject) => {
      writable.once('drain', resolve);
      writable.once('error', reject);
    });
  }
}
```

This illustrates the concept, but production code must also handle stream closure, premature destruction, and errors without leaving listeners or promises hanging. Prefer `pipeline()` or async iteration whenever possible.

## 3. What is the difference between `pipe()` and `pipeline()`?

### `pipe()`

`pipe()` connects a readable stream to a writable stream and automatically manages normal backpressure.

```javascript
readable.pipe(writable);
```

It is convenient for simple stream connections, but **it does not provide the same comprehensive error propagation and cleanup guarantees as `pipeline()`**. An error in one stream does not necessarily clean up every connected stream automatically.

### `pipeline()`

`pipeline()` connects multiple streams, handles backpressure, propagates errors, and destroys participating streams when a pipeline fails.

```javascript
import fs from 'node:fs';
import zlib from 'node:zlib';
import { pipeline } from 'node:stream/promises';

try {
  await pipeline(
    fs.createReadStream('large.log'),
    zlib.createGzip(),
    fs.createWriteStream('large.log.gz')
  );

  console.log('Compression completed');
} catch (error) {
  console.error('Pipeline failed:', error);
}
```

The `stream/promises` version returns a Promise that resolves when the pipeline completes and rejects when an error occurs.

**Interview distinction:** `pipe()` handles backpressure for connected streams, while `pipeline()` additionally gives us centralized completion and error handling, making it the safer default for production pipelines.

## 4. How would you upload a large file to Amazon S3 without loading it into memory?

For a large local file, create a readable stream and pass it to the AWS SDK's multipart upload utility.

Using AWS SDK for JavaScript v3:

```javascript
import fs from 'node:fs';
import { S3Client } from '@aws-sdk/client-s3';
import { Upload } from '@aws-sdk/lib-storage';

const s3 = new S3Client({
  region: process.env.AWS_REGION
});

async function uploadLargeFile(filePath, bucket, key) {
  const fileStream = fs.createReadStream(filePath);

  const upload = new Upload({
    client: s3,
    params: {
      Bucket: bucket,
      Key: key,
      Body: fileStream,
      ContentType: 'application/octet-stream'
    },
    partSize: 8 * 1024 * 1024,
    queueSize: 2,
    leavePartsOnError: false
  });

  await upload.done();

  return { bucket, key };
}
```

The `Upload` utility from `@aws-sdk/lib-storage` supports multipart uploads, including large objects. The stream is consumed incrementally rather than first being converted into a complete in-memory buffer.

Important production considerations:

- **Memory:** `queueSize` controls concurrent multipart uploads, and `partSize` controls individual part sizes. Their combination affects memory usage.
- **Retries:** Configure appropriate retry behavior and handle failed uploads.
- **Cleanup:** `leavePartsOnError: false` requests cleanup of incomplete multipart uploads on failure. Also configure an S3 lifecycle rule to abort stale incomplete multipart uploads.
- **Security:** Use IAM roles or temporary credentials rather than hardcoded AWS keys.
- **Validation:** Enforce file-size limits, allowed file types, and authorization before accepting uploads.
- **Completion:** Do not report upload success until `upload.done()` resolves.

For a file already available on disk, `createReadStream()` is generally straightforward. For a large incoming HTTP upload, the design requires additional care.

## 5. How do you stream an incoming CSV upload to S3?

Suppose an Express endpoint accepts a CSV upload. A robust approach is to stream the incoming request through a CSV parser or validation Transform and into an S3 multipart upload stream.

Conceptually:

```text
HTTP request
    ↓
CSV parser / validation Transform
    ↓
S3 upload stream
    ↓
Amazon S3
```

This avoids buffering the entire upload in application memory.

However, the AWS SDK's `Upload` utility accepts a readable stream as its body; it is not itself a generic Writable stream that can simply be placed at the end of `pipeline()`. A practical approach is to connect the parser to a `PassThrough` stream and give that stream to `Upload`.

```javascript
import { PassThrough } from 'node:stream';
import { pipeline } from 'node:stream/promises';
import { Upload } from '@aws-sdk/lib-storage';

async function streamCsvToS3({
  request,
  s3,
  bucket,
  key,
  csvTransform
}) {
  const body = new PassThrough();

  const upload = new Upload({
    client: s3,
    params: {
      Bucket: bucket,
      Key: key,
      Body: body,
      ContentType: 'text/csv'
    },
    partSize: 8 * 1024 * 1024,
    queueSize: 2,
    leavePartsOnError: false
  });

  const pipelinePromise = pipeline(
    request,
    csvTransform,
    body
  );

  try {
    const [, result] = await Promise.all([
      pipelinePromise,
      upload.done()
    ]);

    return result;
  } catch (error) {
    body.destroy(error);

    // Wait for both operations to settle before returning.
    await Promise.allSettled([
      pipelinePromise,
      upload.done()
    ]);

    throw error;
  }
}
```

This example assumes `request` is the raw readable upload stream and `csvTransform` is a correctly configured Transform that parses or validates CSV data. The HTTP framework's upload handling, request limits, cancellation behavior, and multipart parser must be integrated appropriately.

The key design point is that **the pipeline and S3 upload are separate asynchronous operations**. Both must be monitored, and a failure in either must trigger cleanup of the other. A `PassThrough` is a bridge between the Writable side of the pipeline and the Readable side consumed by the S3 uploader.

For a production endpoint, also consider:

- Enforcing upload-size and request-time limits.
- Rejecting malformed CSV and invalid rows.
- Preventing unbounded row-level buffering in the parser.
- Aborting the S3 upload if the client disconnects.
- Ensuring incomplete multipart uploads are cleaned up.
- Avoiding duplicate uploads when clients retry requests.

If the goal is simply to store the original CSV unchanged, skip CSV parsing and stream the raw request directly to the S3 upload body. If validation is required, use a streaming Transform instead of collecting all rows into an array.

## 6. Why is `stream/promises` `pipeline()` safer?

It helps avoid common stream-handling failures:

1. **Centralized error propagation:** Errors from a source, Transform, or destination reject the pipeline Promise.
2. **Cleanup:** Pipeline failure destroys participating streams, reducing the risk of streams remaining open and resources leaking.
3. **Backpressure:** Data flow is coordinated between stages instead of requiring manual buffer management.
4. **Completion guarantees:** The Promise resolves only when the pipeline completes, so the caller can safely sequence dependent work.
5. **Simpler control flow:** `try/catch/finally` works naturally with asynchronous operations.

One important nuance: `pipeline()` cannot automatically clean up every external operation associated with a stream. If a destination represents an S3 upload managed by another API, you must explicitly coordinate cancellation, multipart cleanup, and error propagation for that upload.

## 7. What if you need to process the file row by row?

Use async iteration with a streaming parser rather than reading the entire CSV into memory:

```javascript
for await (const row of csvParser) {
  await processRow(row);
}
```

Awaiting `processRow()` applies application-level pacing, but it does not automatically guarantee that every upstream source is backpressured correctly. The parser and its input stream must support normal stream flow control.

Avoid patterns such as:

```javascript
const chunks = [];

for await (const chunk of readable) {
  chunks.push(chunk);
}

const completeFile = Buffer.concat(chunks);
```

That approach ultimately buffers the complete file in memory and defeats the purpose of streaming.

## Interview summary

Node.js streams process data incrementally. The four types are Readable, Writable, Duplex, and Transform. Backpressure prevents a producer from overwhelming a consumer: when `write()` returns `false`, a manual producer must pause until `drain`, while `highWaterMark` controls the buffer threshold. `pipe()` connects streams and handles ordinary backpressure; `pipeline()` is safer because it also centralizes error propagation, cleanup, and completion. For large S3 uploads, stream data into the AWS SDK's multipart uploader, avoid full-file buffering, and explicitly coordinate failures between the stream pipeline and the upload operation.

*Q9. Scaling a Node process: cluster, worker threads and child processes. What are the differences, and when do you use each (CPU-bound vs I/O-bound, isolation, shared memory with SharedArrayBuffer)? How does the cluster module distribute connections, why prefer PM2 or container orchestration with one process per container, and how do you share state across instances (Redis, not in-process memory)?*

## 1. Why do we need to scale a Node.js process?

Node.js runs JavaScript on a main thread using an event loop. It handles asynchronous I/O efficiently, but CPU-intensive JavaScript can block the event loop and delay incoming requests.

For example, a Node.js API can handle many concurrent database queries without creating a thread for each query. However, a large synchronous calculation, image transformation, or expensive data-processing task can block the main thread and increase API latency.

There are several ways to scale or isolate work:

- **Cluster:** Run multiple Node.js processes that can accept connections to the same server port.
- **Worker threads:** Run JavaScript in additional threads within one process, particularly for CPU-intensive work.
- **Child processes:** Launch separate operating-system processes for isolation or to run external programs.
- **PM2 or container orchestration:** Manage multiple application instances, restarts, deployment, and scaling.

## 2. Cluster vs. worker threads vs. child processes

| Feature | Cluster | Worker threads | Child processes |
|---|---|---|---|
| Execution model | Multiple processes running the same Node.js server | Multiple threads in one process | Separate operating-system processes |
| Best use case | Scale an HTTP server across CPU cores | CPU-intensive JavaScript calculations | External commands, separate services, or strong process isolation |
| Memory | Separate process memory | Separate JavaScript heaps by default | Separate process memory |
| Communication | IPC between processes | Message passing; optional shared memory | IPC, pipes, or other process communication |
| Crash isolation | Worker process can crash independently | A worker thread can fail without necessarily terminating the whole process, but shares process-level resources | Separate process can crash independently |
| Startup/resource cost | Relatively high | Usually lower than a process | Relatively high |
| Typical example | Multiple API server workers | Image processing, parsing, calculations | FFmpeg, Python scripts, shell commands |

### A. Cluster: Scale an HTTP server across CPU cores

The `cluster` module allows a primary process to create multiple worker processes. Each worker runs its own JavaScript event loop and can handle HTTP connections.

```javascript
import cluster from 'node:cluster';
import os from 'node:os';
import http from 'node:http';

if (cluster.isPrimary) {
  const cpuCount = os.availableParallelism();

  for (let i = 0; i < cpuCount; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code) => {
    console.log(`Worker ${worker.process.pid} exited: ${code}`);

    // Production code should use a deliberate restart policy
    // with backoff and shutdown handling.
  });
} else {
  const server = http.createServer((req, res) => {
    res.end(`Handled by process ${process.pid}`);
  });

  server.listen(3000);
}
```

The primary process manages workers, while each worker runs its own server and event loop.

**When to use it:** When an application needs to serve HTTP traffic using multiple Node.js processes on one machine.

Important considerations:

- Each process has its own heap, database connection pools, caches, and in-memory session data.
- Total database connections can increase significantly. For example, four workers with a pool of 20 connections each can create up to 80 connections.
- A worker restart policy should include backoff to avoid rapid restart loops.
- Graceful shutdown should stop accepting new work and allow in-flight requests to finish.
- More workers do not always mean better performance; measure CPU, memory, event-loop delay, and downstream capacity.

### B. Worker threads: Run CPU-intensive JavaScript in parallel

Worker threads are useful when computation is expensive enough to block the main event loop. Each worker has its own JavaScript execution context and event loop, while sharing the same operating-system process.

Example:

**`main.js`**

```javascript
import { Worker } from 'node:worker_threads';

function runCalculation(input) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(
      new URL('./calculation-worker.js', import.meta.url),
      { workerData: input }
    );

    worker.once('message', resolve);
    worker.once('error', reject);

    worker.once('exit', (code) => {
      if (code !== 0) {
        reject(new Error(`Worker exited with code ${code}`));
      }
    });
  });
}

const result = await runCalculation(100_000_000);
console.log(result);
```

**`calculation-worker.js`**

```javascript
import { parentPort, workerData } from 'node:worker_threads';

let total = 0;

for (let i = 0; i < workerData; i++) {
  total += i;
}

parentPort.postMessage(total);
```

The main thread remains available while the worker performs the calculation.

For production systems, use a **bounded worker pool** rather than creating a new thread for every request. Libraries such as Piscina can manage worker reuse, task queues, and concurrency limits.

Worker threads are most useful for:

- CPU-heavy calculations.
- Parsing or transforming large datasets.
- Computationally expensive image processing implemented in JavaScript.
- Other tasks where parallel computation outweighs messaging and scheduling overhead.

They generally do not improve ordinary asynchronous database or network I/O, because Node.js already handles such I/O efficiently.

### C. Child processes: Isolate work or execute external programs

Child processes are separate operating-system processes with their own memory and runtime. They are appropriate when running a different executable or when process-level isolation is desirable.

```javascript
import { spawn } from 'node:child_process';

const child = spawn('ffmpeg', [
  '-i', 'input.mp4',
  '-vf', 'scale=1280:-1',
  'output.mp4'
], {
  stdio: 'inherit'
});

child.on('error', (error) => {
  console.error('Failed to start FFmpeg:', error);
});

child.on('close', (code) => {
  console.log(`FFmpeg exited with code ${code}`);
});
```

Use `spawn()` for streaming output from long-running commands. Avoid using `exec()` for unbounded output because it buffers output in memory and has a configurable maximum buffer size. Never construct shell commands directly from untrusted input.

Child processes are useful for:

- Running tools such as FFmpeg.
- Executing separate programs written in Python, Go, or another language.
- Running isolated background tasks.
- Separating workloads that require different dependencies or failure boundaries.

A child process can still overload the machine if too many are created, so concurrency and resource limits remain necessary.

## 3. CPU-bound vs. I/O-bound: Which approach should you choose?

The key distinction is whether the application spends its time waiting for external operations or performing computation.

| Workload | Preferred approach | Reason |
|---|---|---|
| Database queries and HTTP requests | Async I/O on the main event loop | Waiting does not require a dedicated JavaScript worker |
| CPU-heavy JavaScript calculations | Worker threads or a worker pool | Computation can run concurrently on additional CPU cores |
| HTTP server handling more traffic | Cluster, multiple managed processes, or multiple containers | Multiple event loops can use multiple cores |
| Video transcoding using FFmpeg | Child process | FFmpeg is an external executable |
| Scheduled report generation | Background worker or job queue | Keeps expensive work away from request handling |
| Multiple independent API replicas | PM2 or container orchestration | Provides process management, deployment, and scaling |

For example, if an API spends most of its time waiting for MySQL, adding worker threads may not help. Better improvements might include connection pooling, query optimization, caching, and efficient concurrency limits.

If a synchronous JavaScript calculation consumes 100% of one core and blocks requests, move that calculation to a worker pool or a separate background service.

## 4. How does `cluster` distribute connections?

The primary process creates workers that can listen on the same port. The operating system and Node.js cluster implementation coordinate how incoming connections reach those workers.

There are two important models:

1. **Primary-process distribution:** The primary process accepts connections and distributes them to workers. On supported platforms, Node.js commonly uses a round-robin strategy for this model.
2. **Operating-system distribution:** Workers accept connections from a shared listening socket, and the operating system determines which worker receives each connection.

The exact scheduling behavior depends on the platform and cluster configuration.

A crucial distinction is that **cluster distributes connections, not necessarily individual HTTP requests**. An HTTP keep-alive connection can carry multiple requests to the same worker. Uneven connection lifetimes and workload sizes can therefore create imbalanced request distribution even when connections are distributed reasonably evenly.

Cluster also does not automatically share application state between workers. Sticky sessions may be required by some applications, but a better approach for scalable HTTP APIs is often to keep the application stateless and store shared session data externally.

## 5. What is `SharedArrayBuffer`, and how does it work?

Worker threads can communicate using messages or shared memory.

With ordinary structured-clone messaging, data is copied or transferred according to the messaging API. With `SharedArrayBuffer`, multiple threads can access the same underlying memory.

```javascript
import { Worker } from 'node:worker_threads';

const sharedBuffer = new SharedArrayBuffer(
  Int32Array.BYTES_PER_ELEMENT
);

const sharedCounter = new Int32Array(sharedBuffer);

const worker = new Worker(
  new URL('./shared-worker.js', import.meta.url),
  { workerData: sharedBuffer }
);

worker.on('message', (message) => {
  console.log(message);
});
```

**`shared-worker.js`**

```javascript
import {
  workerData,
  parentPort
} from 'node:worker_threads';

const counter = new Int32Array(workerData);

Atomics.add(counter, 0, 1);

parentPort.postMessage(Atomics.load(counter, 0));
```

Both threads can access the same memory region. `Atomics` provides synchronization primitives for supported shared typed-array operations, helping prevent data races for those operations.

Use shared memory when profiling shows that copying data or message-passing overhead is a bottleneck. It is more complex than message passing because concurrent access requires careful synchronization.

`SharedArrayBuffer` is not a replacement for Redis or a database: it shares memory between threads in one process, not between independent Node.js processes or separate containers.

## 6. Why use PM2 or container orchestration instead of managing `cluster` manually?

The `cluster` module provides primitives for creating workers, but production process management involves much more than forking processes.

### PM2

PM2 is a process manager that can run Node.js applications in cluster mode and provide:

- Automatic restart after crashes.
- Process monitoring and logs.
- Multiple application instances.
- Reload workflows intended to reduce downtime.
- Startup integration and environment configuration.

It can be a practical choice for deploying a Node.js application directly on a VPS.

### Containers and orchestration

With Docker and an orchestrator such as Kubernetes, a common architecture is **one main application process per container**, with multiple replicas managed by the orchestrator.

```text
                 Load balancer
                       |
          +------------+------------+
          |            |            |
       API replica  API replica  API replica
       Container 1  Container 2  Container 3
          |            |            |
          +------------+------------+
                       |
               Shared dependencies
             MySQL / Redis / S3
```

Benefits include:

- Horizontal scaling by changing replica count.
- Health checks and replacement of unhealthy instances.
- Resource limits and scheduling.
- Rolling deployments and rollback strategies.
- Clear separation between application lifecycle management and application code.

“One process per container” is a useful operational default, not an absolute rule. If a container needs to use multiple CPU cores efficiently, scale through replicas or deliberately use multiple processes within that container. The right choice depends on resource utilization, deployment architecture, and operational complexity.

PM2 and an orchestrator solve overlapping but different operational problems. PM2 is useful for managing processes on a server; Kubernetes or another orchestrator manages containers and their lifecycle across a cluster of machines.

## 7. How do you share state across multiple Node.js instances?

**Do not rely on in-process memory for state that must be consistent across multiple instances.**

Suppose an API stores login sessions in a JavaScript `Map`:

```javascript
const sessions = new Map();
```

This works only inside that process. If a load balancer sends a subsequent request to another instance, that instance cannot see the original map.

The same problem applies to in-memory caches, rate-limit counters, job state, and WebSocket connection metadata.

Use a shared external system based on the type of state:

| State | Typical solution |
|---|---|
| Login sessions | Redis or a shared database |
| Rate-limit counters | Redis with atomic operations |
| Distributed locks | Redis or a database with appropriate coordination guarantees |
| Durable business data | MySQL or PostgreSQL |
| Cache shared by instances | Redis |
| Background jobs | Redis-backed queue, RabbitMQ, Kafka, or another suitable broker |
| Uploaded files | S3 or another object store |
| Real-time event distribution | Redis Pub/Sub, a message broker, or another shared event system |

### Example: Redis-backed shared state

```javascript
import { createClient } from 'redis';

const redis = createClient({
  url: process.env.REDIS_URL
});

redis.on('error', (error) => {
  console.error('Redis error:', error);
});

await redis.connect();

await redis.set('session:123', JSON.stringify({
  userId: 42,
  role: 'admin'
}), {
  EX: 3600
});

const session = await redis.get('session:123');
```

All application instances connected to the same Redis deployment can access this key, subject to the configured Redis topology and availability.

For session storage, use a proper session middleware and a production-ready Redis store rather than manually implementing session management with ad hoc keys.

For rate limiting or counters, use atomic Redis operations or Lua scripts where needed. A separate `GET` followed by `SET` is not an atomic increment and can lose updates under concurrency.

For durable business data, keep the authoritative record in the database rather than relying on Redis alone.

## 8. Common scaling mistakes

- **Using worker threads for every asynchronous task:** They add overhead when ordinary asynchronous I/O is sufficient.
- **Forking unlimited processes:** Each process consumes memory and may create its own database connection pool.
- **Assuming cluster shares memory:** Each worker has its own process-level application state.
- **Treating `SharedArrayBuffer` as distributed memory:** It is shared between threads, not across separate processes or machines.
- **Using sticky sessions to compensate for all state problems:** Externalizing session state usually makes horizontal scaling easier.
- **Running CPU-heavy code on the request event loop:** This increases latency for unrelated requests.
- **Scaling application replicas without checking downstream limits:** More instances can overwhelm MySQL, Redis, third-party APIs, or connection quotas.
- **Ignoring graceful shutdown:** Requests and background jobs can be interrupted during deployments or restarts.

## Interview summary

Cluster runs multiple Node.js processes to serve HTTP traffic across CPU cores. Worker threads are best for CPU-intensive JavaScript and can share memory through `SharedArrayBuffer` with proper synchronization. Child processes provide process isolation and are suitable for external executables or separate workloads. PM2 manages processes on a server, while container orchestration commonly scales applications through replicas. Finally, state that must be consistent across instances should live in shared systems such as Redis or a database—not in process-local variables.

*Q10. Error handling and process safety. What is the difference between operational and programmer errors? What happens on an uncaughtException or unhandledRejection (and Node 15+ behaviour), and why should you log and exit rather than continue? How do you implement graceful shutdown on SIGTERM (stop accepting connections, drain in-flight requests, close DB pools)?*

## 1. What is the difference between operational errors and programmer errors?

In Node.js, errors can be classified by their cause and whether the application can reasonably recover from them.

### Operational errors

Operational errors are expected failures that can occur during normal application operation. They do not necessarily indicate a bug in the code.

Examples:
- A database connection times out.
- A remote API returns HTTP 503.
- A user requests a resource that does not exist.
- A file cannot be found.
- A network connection is reset.
- A request fails validation.

**How to handle them:** Handle them at the appropriate boundary, return a meaningful response, retry when appropriate, or use a fallback.

For example:

```javascript
try {
  const user = await userService.findById(userId);

  if (!user) {
    return res.status(404).json({
      error: 'USER_NOT_FOUND'
    });
  }

  return res.json(user);
} catch (error) {
  next(error);
}
```

The missing user is an expected business outcome, while an unexpected database failure is passed to centralized error handling.

### Programmer errors

Programmer errors result from defects or incorrect assumptions in the code.

Examples:
- Accessing a property on `undefined`.
- Calling a function with an invalid internal state.
- Violating an invariant.
- Using a variable incorrectly.
- Making an incorrect assumption about an object's structure.

**How to handle them:** Fix the underlying bug, add tests, and prevent the faulty code from running again. If the error escapes into an uncaught exception, treat the process as potentially unsafe.

For example:

```javascript
function calculateTotal(order) {
  return order.items.reduce(
    (sum, item) => sum + item.price,
    0
  );
}
```

If `order` is unexpectedly `undefined`, the code may throw a `TypeError`. The correct fix is to validate inputs at the appropriate boundary and maintain internal invariants—not to catch every `TypeError` and silently continue.

The distinction is useful, but not absolute. A database error can reveal a programming defect, and an invalid user input should usually be rejected before it reaches business logic.

## 2. What happens when an `uncaughtException` occurs?

An `uncaughtException` occurs when a synchronous exception escapes the JavaScript execution context without being handled.

```javascript
throw new Error('Unexpected failure');
```

If nothing catches this error, Node.js normally prints the error and stack trace to standard error and exits with a nonzero exit code.

You can register a handler:

```javascript
process.on('uncaughtException', (error) => {
  console.error('Uncaught exception:', error);
});
```

However, this changes the default behavior. Registering the handler can prevent Node.js from terminating automatically.

That creates a serious risk: the application may be left in an inconsistent state. An exception could have interrupted a multi-step operation, corrupted application invariants, or left locks and resources in an unexpected state.

Therefore, **an `uncaughtException` handler should be used for last-resort diagnostics and controlled termination—not to resume normal service**.

A minimal emergency handler might look like this:

```javascript
process.on('uncaughtException', (error) => {
  console.error('Fatal uncaught exception:', error);

  // Stop accepting work and terminate.
  // A process manager should restart the process.
  process.exit(1);
});
```

In a production application, synchronous logging to standard error is a useful last-resort measure, but logging infrastructure itself can fail. Avoid complex asynchronous cleanup in this handler; it is not a reliable recovery point.

If the application requires orderly draining, implement normal shutdown handling separately and use it for signals and controlled termination paths.

## 3. What happens on an `unhandledRejection`?

An `unhandledRejection` occurs when a Promise is rejected and no rejection handler is attached within the relevant event-loop turn.

```javascript
async function loadData() {
  throw new Error('Database unavailable');
}

loadData(); // Rejected Promise is not handled.
```

Node.js behavior depends on the runtime version and configuration.

**Since Node.js 15, the default `--unhandled-rejections` mode is `throw`.** Under this mode, an unhandled rejection is raised as an uncaught exception if no `unhandledRejection` hook handles it. This normally causes the process to terminate.

A process-level listener can alter that behavior, so do not add one merely to log errors and continue.

Handle expected Promise failures explicitly:

```javascript
try {
  const data = await loadData();
  console.log(data);
} catch (error) {
  console.error('Failed to load data:', error);
}
```

Or attach a rejection handler when the operation is intentionally launched without awaiting it:

```javascript
loadData().catch((error) => {
  logger.error({ error }, 'Background operation failed');
});
```

For background tasks, logging alone may not be enough. The application may also need retry logic, a failed-job queue, or an alert.

An important distinction: `process.on('unhandledRejection')` is useful for diagnostics and monitoring, but it should not become a blanket substitute for handling Promise failures where they originate.

## 4. Why log and exit instead of continuing?

After a fatal, uncaught error, the process may no longer be in a trustworthy state. Continuing can lead to:

- Incorrect business operations or partially committed workflows.
- Inconsistent in-memory state.
- Requests hanging because resources were not released.
- Further errors that obscure the original cause.
- Corrupted caches or invalid application invariants.

A safer recovery strategy is:

1. Capture the error and stack trace.
2. Emit diagnostic information through the logging system or standard error.
3. Stop accepting new requests where practical.
4. Terminate the process.
5. Let a process manager or orchestrator restart it.
6. Use health checks and monitoring to detect repeated crashes.

A process manager should not restart a process indefinitely without safeguards. Repeated crashes may indicate a persistent defect, a bad deployment, or a failing dependency.

**Do not confuse process recovery with request recovery.** Operational errors can often be handled at request level. An uncaught exception is a process-level safety problem.

## 5. How do you implement graceful shutdown on `SIGTERM`?

A graceful shutdown allows the application to stop receiving new work while giving existing requests and operations time to finish.

`SIGTERM` is commonly sent by container orchestrators and process managers during deployment, scaling, or termination.

A typical shutdown sequence is:

1. Mark the instance as not ready so the load balancer stops routing new traffic to it.
2. Stop accepting new connections.
3. Allow in-flight requests to finish within a defined timeout.
4. Close database pools, Redis connections, message consumers, and other resources.
5. Force termination if the deadline expires.

### Example: Express server with a database pool

```javascript
import http from 'node:http';
import express from 'express';
import mysql from 'mysql2/promise';

const app = express();

const pool = mysql.createPool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  connectionLimit: 10
});

app.get('/health/live', (req, res) => {
  res.status(200).json({ status: 'ok' });
});

app.get('/health/ready', (req, res) => {
  if (shuttingDown) {
    return res.status(503).json({
      status: 'shutting_down'
    });
  }

  return res.status(200).json({
    status: 'ready'
  });
});

const server = http.createServer(app);
let shuttingDown = false;
let shutdownPromise;

server.listen(process.env.PORT || 3000);

function closeServer() {
  return new Promise((resolve, reject) => {
    server.close((error) => {
      if (error) reject(error);
      else resolve();
    });
  });
}

function shutdown(signal) {
  if (shutdownPromise) {
    return shutdownPromise;
  }

  shuttingDown = true;
  console.log(`Received ${signal}; starting graceful shutdown`);

  shutdownPromise = (async () => {
    const forceExitTimer = setTimeout(() => {
      console.error('Graceful shutdown timed out');
      process.exit(1);
    }, 25_000);

    // Do not keep the event loop alive solely for this timer.
    forceExitTimer.unref();

    try {
      // Stop accepting new connections and wait for existing
      // HTTP connections to close.
      await closeServer();

      // Close the pool after requests have drained.
      await pool.end();

      console.log('Graceful shutdown completed');
      clearTimeout(forceExitTimer);
      process.exitCode = 0;
    } catch (error) {
      console.error('Graceful shutdown failed:', error);
      clearTimeout(forceExitTimer);
      process.exitCode = 1;
    }
  })();

  return shutdownPromise;
}

process.once('SIGTERM', () => {
  void shutdown('SIGTERM');
});

process.once('SIGINT', () => {
  void shutdown('SIGINT');
});
```

This illustrates the shutdown lifecycle, but a production implementation needs to account for several important details:

- `shuttingDown` must be declared before the readiness handler can execute.
- The server must stop accepting new connections before the database pool is closed.
- Long-running requests need a defined deadline and cancellation strategy.
- Active WebSocket connections, background jobs, and message consumers need their own shutdown logic.
- If the process receives another termination signal, the application may choose to escalate to forced termination.
- Avoid calling `process.exit()` immediately after logging normal shutdown completion, because doing so can truncate pending output or cleanup. Prefer setting `process.exitCode` and allowing the event loop to finish naturally.

The timer above is an emergency deadline. In a production implementation, make sure that the deadline covers the entire shutdown procedure, including any resource cleanup that could otherwise hang.

## 6. Does `server.close()` drain all in-flight requests?

`server.close()` stops the server from accepting new connections and waits for existing connections to close. Exact handling of idle keep-alive connections depends on the Node.js version and connection state.

For current Node.js versions, `server.close()` also closes idle connections. However, it does not guarantee that every application-level operation will finish successfully before an external termination deadline.

For long-running HTTP requests, you should:

- Track active requests or operations if explicit drain visibility is needed.
- Stop accepting new work.
- Allow active requests to complete within a bounded grace period.
- Cancel or terminate work that exceeds the deadline.
- Ensure the database pool remains available until dependent requests finish.

For HTTP servers, `server.closeAllConnections()` can be used as a forceful fallback, but it should not be called before the graceful-drain period if the goal is to preserve active requests. WebSocket and upgraded connections may require separate tracking and closure.

Also, a readiness check alone does not instantly remove an instance from every load balancer. The deployment platform's routing and termination behavior must be configured to allow sufficient propagation time.

## 7. What is the difference between liveness and readiness?

These health checks serve different purposes.

- **Liveness:** Is the process functioning, or should it be restarted?
- **Readiness:** Should this instance receive new traffic?

During graceful shutdown, the instance should generally become unready immediately while remaining alive long enough to drain requests.

Do not make liveness depend blindly on every downstream dependency. If MySQL is temporarily unavailable, restarting every API process may amplify the incident. Readiness can incorporate dependency status when appropriate, while liveness should focus on whether restarting the process is likely to help.

## 8. What about database pools, Redis, and background workers?

Shutdown must follow dependency order.

For example:

1. Stop accepting new HTTP requests.
2. Stop polling for new background jobs.
3. Wait for active requests and jobs to complete, subject to a deadline.
4. Close database and Redis clients.
5. Close telemetry exporters and other resources.

If the application uses a queue, acknowledge a job only after its work has completed successfully. If shutdown interrupts a job, allow the queue's retry or visibility-timeout mechanism to recover it safely.

For operations that may be retried, design idempotency into the workflow. A process can fail after performing an external side effect but before recording success, so a retry must not blindly duplicate a payment or other irreversible action.

## 9. Common interview mistakes

- Catching all errors and returning HTTP 200.
- Logging `uncaughtException` and continuing to serve traffic.
- Assuming all unhandled Promise rejections are harmless.
- Closing the database pool before active requests have drained.
- Calling `process.exit(0)` immediately after receiving `SIGTERM`.
- Using an unbounded shutdown wait that can exceed the container's termination grace period.
- Assuming readiness checks instantly stop all incoming requests.
- Retrying every operational error without limits, backoff, or idempotency.

## Interview summary

Operational errors are expected failures that should be handled at the appropriate application boundary. Programmer errors indicate defects and may violate internal invariants. Uncaught exceptions and unhandled rejections can leave a process in an unsafe state; log them for diagnosis and terminate rather than blindly continuing. For graceful shutdown, mark the instance unready, stop accepting new connections, drain in-flight work within a deadline, close database and other resource pools, and let the process manager or orchestrator handle restart and replacement.

*Q11. Memory management and leak hunting. How does V8’s generational garbage collector work, and what do --max-old-space-size, heap snapshots and --inspect with Chrome DevTools reveal? What are common leak sources (unbounded caches, listeners never removed, closures in long-lived scopes, global arrays), and how do you spot growth in production?*

## 1. How does memory management work in Node.js?

Node.js uses the V8 JavaScript engine, which automatically manages JavaScript memory through garbage collection (GC).

When an application creates objects, arrays, functions, and closures, V8 allocates memory for them. Garbage collection reclaims memory occupied by objects that are no longer reachable from the application's live references.

For example:

```javascript
function processRequest() {
  let data = { name: 'Dev' };
  data = null;
}

processRequest();
```

Once the function finishes and no references to the object remain, the object becomes eligible for garbage collection. However, **eligible for collection does not mean immediately collected**; V8 decides when to run the collector.

A memory leak occurs when an application retains references to objects that it no longer needs. The garbage collector cannot reclaim those objects because they are still reachable.

## 2. How does V8's generational garbage collector work?

V8 uses a generational approach based on the observation that many objects have short lifetimes.

At a high level, the heap is divided into areas with different collection strategies.

### A. Young generation

New objects are generally allocated in the young generation.

The young generation is collected relatively frequently because many temporary objects become unreachable quickly, such as request payloads, intermediate arrays, and short-lived response objects.

V8 uses the **Scavenger** collector for young-generation garbage collection. In its traditional copying approach, live objects are copied or moved between young-generation spaces, while unreachable objects are discarded.

Objects that survive enough collections may be promoted to the old generation.

### B. Old generation

Objects that survive longer may be promoted to the old generation. Examples include long-lived application configuration, caches, and persistent in-memory data structures.

Old-generation collection is generally more expensive because it may involve examining and reclaiming a larger set of objects.

V8 uses collectors including:

- **Major mark-compact collection:** Identifies live objects, reclaims unreachable objects, and can compact memory to reduce fragmentation.
- **Concurrent and incremental techniques:** Some marking and other GC work can occur concurrently with application execution or in smaller steps to reduce long pauses.
- **Other specialized collectors:** V8's implementation and collection strategies evolve between versions.

The exact algorithms are implementation-dependent, but the central idea is that V8 tries to collect short-lived objects cheaply while managing long-lived objects more selectively.

### C. What is the impact on application performance?

Garbage collection consumes CPU time. Collection activity can also cause pauses or increased latency, although modern V8 uses techniques to reduce their impact.

For example, an API that creates millions of temporary objects may experience frequent young-generation collections. If objects are unintentionally retained, old-generation usage may continue increasing, leading to expensive collections, higher latency, and eventually an out-of-memory failure.

**Important:** Garbage collection does not guarantee that memory usage always decreases immediately after an object becomes unreachable. V8 may defer collection, and the operating system's reported resident memory (RSS) can remain high even after the JavaScript heap has reclaimed objects.

## 3. What does `--max-old-space-size` do?

The `--max-old-space-size` option configures the approximate maximum size of V8's old-generation heap, measured in MiB.

Example:

```bash
node --max-old-space-size=4096 server.js
```

This sets the old-generation heap limit to approximately 4 GiB.

It is useful when a legitimate workload needs more heap memory than the default limit permits.

However, it does not:

- Set the total memory limit for the Node.js process.
- Set a hard limit on RSS or native allocations.
- Fix a memory leak.
- Guarantee that the process can allocate the entire configured amount.

A Node.js process can use additional memory for young-generation objects, native allocations, Buffers, code, stacks, and other runtime structures. Container memory limits may also be lower than the configured heap limit.

For example, setting the old-generation limit to 4 GiB inside a container limited to 2 GiB may cause the container to be killed by the operating system before V8 reaches its configured heap limit.

**Production recommendation:** Set heap limits based on measured memory usage and the container's actual memory budget. Leave sufficient headroom for native memory, buffers, runtime overhead, and temporary allocation peaks.

## 4. How do you identify a memory leak using production metrics?

The first step is to establish whether memory usage is actually growing over time.

Node.js exposes memory metrics through `process.memoryUsage()`:

```javascript
import process from 'node:process';

setInterval(() => {
  const memory = process.memoryUsage();

  console.log({
    rssMB: Math.round(memory.rss / 1024 / 1024),
    heapUsedMB: Math.round(memory.heapUsed / 1024 / 1024),
    heapTotalMB: Math.round(memory.heapTotal / 1024 / 1024),
    externalMB: Math.round(memory.external / 1024 / 1024),
    arrayBuffersMB: Math.round(memory.arrayBuffers / 1024 / 1024)
  });
}, 30_000);
```

In production, send these metrics to a monitoring system rather than relying only on console logs.

### What do the metrics mean?

| Metric | Meaning | What growth may indicate |
|---|---|---|
| `rss` | Total resident memory for the process | Overall process memory growth |
| `heapUsed` | JavaScript heap currently used | Retained JS objects or allocation pressure |
| `heapTotal` | Memory currently allocated to the V8 heap | Heap expansion; not necessarily a leak |
| `external` | Memory associated with native objects bound to JavaScript | Native resources or externally allocated memory |
| `arrayBuffers` | Memory used by `ArrayBuffer` and related buffers, including Node.js Buffers | Large or retained buffers |

`arrayBuffers` is included in `external`, so do not add those two values together when estimating memory consumption.

### How do you interpret the trends?

**Case 1: `heapUsed` repeatedly rises and its post-GC baseline keeps increasing.**

This suggests that more JavaScript objects are being retained over time. An unbounded cache, retained request objects, or leaked listeners may be responsible.

**Case 2: `heapUsed` remains relatively stable, but RSS increases.**

Investigate native allocations, Buffer usage, native dependencies, memory fragmentation, and allocator behavior. A JavaScript heap snapshot alone may not explain the problem.

**Case 3: `external` and `arrayBuffers` rise sharply.**

Investigate large file uploads, retained Buffers, streaming pipelines that accumulate chunks, and native libraries that allocate external memory.

**Case 4: Memory rises during traffic spikes and later falls.**

This can be normal allocation pressure. Compare memory with traffic volume, garbage-collection activity, request latency, and the post-GC baseline before concluding there is a leak.

The most useful signal is often the **trend in memory retained after comparable workload periods**, not a single high memory reading.

## 5. How do heap snapshots help find memory leaks?

A heap snapshot captures the JavaScript heap at a particular point in time. Chrome DevTools can inspect objects, references, and their relationships.

A practical investigation usually follows these steps:

1. Capture a baseline snapshot after startup and warm-up.
2. Exercise the application with a representative workload.
3. Allow temporary work to finish and garbage collection to occur naturally.
4. Capture another snapshot.
5. Compare the snapshots.
6. Inspect objects whose retained size or count has increased.
7. Follow their retaining paths to determine which references keep them alive.

### What should you look for?

- Arrays or Maps whose sizes grow with each request.
- Large numbers of objects associated with completed requests.
- Event listeners retaining old request or connection state.
- Closures retaining large objects.
- Buffers retained by caches or queues.
- Unexpectedly large numbers of instances of the same class.

Two important concepts are:

- **Shallow size:** Memory directly occupied by an object.
- **Retained size:** Memory that could potentially be reclaimed if the object and the objects reachable only through it became unreachable.

Retained size is especially useful for finding the root of a leak. A small object may retain a large object graph through its references.

A heap snapshot is not simply a list of large objects. The retaining path often tells you why an object is still alive.

### Example: An unbounded cache

```javascript
const cache = new Map();

app.get('/users/:id', async (req, res) => {
  const user = await loadUser(req.params.id);

  cache.set(req.params.id, user);

  res.json(user);
});
```

If requests use an ever-growing set of user IDs, the Map can retain every loaded user indefinitely.

A heap snapshot may reveal a large Map and many user objects retained through it. The fix is to implement an explicit cache policy, such as a maximum size, expiration, or a dedicated caching system with bounded storage.

## 6. How do `--inspect` and Chrome DevTools help?

The `--inspect` flag enables the Node.js Inspector protocol, which allows compatible debugging tools to connect to the process.

Start Node.js with:

```bash
node --inspect server.js
```

By default, the inspector listens on port `9229`. You can open Chrome and navigate to `chrome://inspect` to discover and debug the Node.js process.

For a local development session where you want the process to wait for a debugger:

```bash
node --inspect-brk server.js
```

This pauses execution at startup until a debugger attaches.

In Chrome DevTools, you can:

- Capture and compare heap snapshots.
- Record allocation timelines.
- Inspect retaining paths.
- View object counts and retained sizes.
- Profile CPU usage.
- Set breakpoints and inspect variables.
- Identify code paths that repeatedly allocate expensive objects.

### How do you use this in production?

Do not expose the inspector publicly. A debugger attached to a production process can inspect sensitive data and execute code in the process.

For production incidents:

1. Prefer monitoring and heap profiling designed for controlled operational use.
2. Reproduce the workload in a staging environment where possible.
3. If a live snapshot is necessary, use a secured access path, restrict access, and assess the impact first.
4. Consider memory and latency overhead before capturing a snapshot.
5. Ensure snapshots containing credentials, personal data, or other secrets are stored and handled securely.

Heap snapshots can temporarily consume significant additional memory and may pause the application. On a memory-constrained container, snapshot generation can cause an outage. Use them deliberately rather than capturing them automatically every few seconds.

## 7. What are common memory leak sources in Node.js?

### A. Unbounded caches

```javascript
const cache = new Map();

function cacheResult(key, result) {
  cache.set(key, result);
}
```

If keys are never removed, the cache grows continuously.

**Fix:** Use a bounded cache with a maximum entry count or memory budget, expiration, and an appropriate eviction policy such as LRU. TTL alone may not be sufficient if expired entries are not removed promptly.

### B. Event listeners that are never removed

```javascript
function handleRequest(req, res) {
  process.on('dataUpdated', () => {
    console.log(req.url);
  });
}
```

Every request adds another global listener, and each listener closes over `req`. Even after the request finishes, the listener can retain the request and related objects.

**Fix:** Register listeners at the correct lifecycle boundary and remove them when the work finishes. Use `once()` for one-time events where appropriate, or an `AbortController` to coordinate listener cleanup.

```javascript
const controller = new AbortController();

process.on('dataUpdated', handleUpdate, {
  signal: controller.signal
});

// When the operation finishes:
controller.abort();
```

The `signal` option is supported by Node.js event listener APIs such as `EventTarget`; do not assume every emitter implementation accepts it. For Node.js `EventEmitter` APIs that do not support the option, use `removeListener()` or `off()`.

### C. Closures in long-lived scopes

```javascript
function createHandler() {
  const largeData = loadLargeData();

  return function handler(req, res) {
    res.json({ status: 'ok' });
  };
}
```

The returned function closes over its lexical environment. If it retains a reference to `largeData`, the data may remain reachable as long as the handler remains reachable, even if the handler never uses that value in its body.

**Fix:** Avoid unnecessarily capturing large objects in long-lived callbacks. Keep scopes small and release references when their lifetime ends.

### D. Global arrays that grow indefinitely

```javascript
const requestHistory = [];

app.use((req, res, next) => {
  requestHistory.push({
    url: req.url,
    timestamp: Date.now()
  });

  next();
});
```

This records every request forever.

**Fix:** Use bounded in-memory buffers, aggregate counters, sampled telemetry, or external logging and monitoring systems.

### E. Unconsumed streams and queues

If producers continue adding data while consumers are slow or stopped, queues and buffers can grow without bound.

**Fix:** Apply backpressure, cap queue sizes, limit concurrency, and define what happens when capacity is exhausted.

### F. Timers and subscriptions

Repeatedly creating intervals, polling loops, or subscriptions without clearing them can retain callbacks and the objects they capture.

**Fix:** Track timer and subscription lifetimes and clear them during cleanup.

### G. Native resources and Buffers

Large Buffers, native modules, database clients, and other external allocations can cause RSS to grow even when JavaScript heap usage is stable.

**Fix:** Inspect `external`, `arrayBuffers`, and RSS alongside heap snapshots. Verify that streams, database connections, native resources, and subscriptions are properly closed.

## 8. How do you spot memory growth in production?

A practical workflow is:

**Step 1 — Establish a baseline.**

Measure RSS, heap used, external memory, request rate, latency, and garbage-collection activity after startup and warm-up.

**Step 2 — Observe under representative load.**

Compare memory usage across similar request volumes and workload periods. A continuously rising post-GC baseline is more suspicious than temporary spikes.

**Step 3 — Check the failure pattern.**

- Heap growth suggests retained JavaScript objects or excessive allocation.
- External memory growth suggests Buffers or native allocations.
- RSS growth without corresponding heap growth suggests looking beyond ordinary JavaScript objects.

**Step 4 — Capture diagnostic evidence.**

Use a heap snapshot or allocation profile in a controlled environment. Identify growing object types and follow retaining paths to the code holding references.

**Step 5 — Fix the ownership or lifetime problem.**

Bound caches, remove listeners, dispose of subscriptions, drain queues, and release unnecessary references.

**Step 6 — Verify the fix.**

Repeat the same workload and compare memory baselines. Confirm that memory stabilizes and that latency and garbage-collection pressure improve.

For operational visibility, use a metrics platform to track memory trends, GC activity, event-loop delay, and out-of-memory events. Alerts should account for container limits and sustained growth rather than triggering on every temporary spike.

## 9. Interview summary

V8 uses generational garbage collection: short-lived objects are collected frequently in the young generation, while longer-lived objects may be promoted to the old generation. `--max-old-space-size` configures the old-generation heap limit, not the entire process memory limit. Heap snapshots and Chrome DevTools reveal retained objects and their retaining paths. Common leaks include unbounded caches, listeners, closures, global collections, timers, and unconsumed queues. In production, monitor RSS, `heapUsed`, `external`, and `arrayBuffers`; compare memory after comparable workloads; investigate growing post-GC baselines; and validate fixes under repeatable load.

*Q12. Module systems and project structure. Compare CommonJS (require, caching, circular dependencies) and ES modules in Node ("type": "module", top-level await, import(), __dirname workarounds). How do you structure a maintainable Express/Fastify app (routes, controllers, services, repositories, config, dependency injection)?*

## 1. CommonJS vs. ES Modules

Node.js supports two primary JavaScript module systems: CommonJS (CJS) and ECMAScript Modules (ESM).

**CommonJS** is the traditional Node.js module system, using `require()` and `module.exports`. **ES modules** are the standardized JavaScript module system, using `import` and `export`.

### Key differences

| Feature | CommonJS | ES modules |
|---|---|---|
| Import | `require('./service')` | `import { getUser } from './service.js'` |
| Export | `module.exports = ...` | `export` / `export default` |
| Loading model | `require()` loads modules synchronously | Static imports are resolved and linked before module evaluation |
| Caching | `require.cache` caches loaded modules | ESM modules are cached by resolved module identity |
| Circular dependencies | May expose partially initialized exports | Live bindings can expose initialization-order problems |
| Top-level `await` | Not supported directly | Supported |
| Dynamic loading | `require()` | `import()` returns a Promise |
| File extensions | Often omitted for relative imports | Relative file extensions are normally explicit |
| Module scope | Wrapped in a function | Own module scope |
| `__dirname` | Available in ordinary CommonJS modules | Not available directly |

### A. CommonJS example

**`userService.cjs`**

```javascript
function getUser(id) {
  return { id, name: 'Dev' };
}

module.exports = { getUser };
```

**`app.cjs`**

```javascript
const { getUser } = require('./userService.cjs');

console.log(getUser(1));
```

CommonJS modules execute within a function wrapper that provides variables such as `require`, `module`, `exports`, `__filename`, and `__dirname`.

### B. ES module example

Set the following in `package.json`:

```json
{
  "type": "module"
}
```

**`userService.js`**

```javascript
export function getUser(id) {
  return { id, name: 'Dev' };
}
```

**`app.js`**

```javascript
import { getUser } from './userService.js';

console.log(getUser(1));
```

In a package configured with `"type": "module"`, ordinary `.js` files are interpreted as ES modules. The `.mjs` extension also explicitly identifies an ES module, while `.cjs` identifies CommonJS.

For new Node.js projects, ESM is a good default when your dependencies and tooling support it. CommonJS remains valid, particularly in older applications and packages.

## 2. How does module caching work?

Both module systems cache evaluated modules so repeated imports generally reuse the same module instance instead of executing the module again.

In CommonJS:

```javascript
const first = require('./config.cjs');
const second = require('./config.cjs');

console.log(first === second); // Usually true
```

This means that mutable module-level state is shared by consumers of that module within the same process.

For example:

```javascript
// counter.cjs
let count = 0;

module.exports = {
  increment() {
    return ++count;
  }
};
```

Every consumer of this module in the same process normally shares the same counter.

However, module-level state is **not shared across separate Node.js processes or containers**.

Caching can also create testing problems if a module captures environment variables or initializes connections at import time. Tests may then observe stale configuration or share mutable state between test cases.

A better pattern is to expose a factory function and inject configuration or dependencies explicitly.

## 3. What happens with circular dependencies?

A circular dependency occurs when modules depend on one another.

For example:

```text
userService → emailService → userService
```

Circular dependencies are legal in both module systems, but they can cause initialization-order problems.

### CommonJS behavior

CommonJS can expose partially initialized exports while a module is still executing.

**`a.cjs`**

```javascript
exports.name = 'A';

const b = require('./b.cjs');

exports.getB = () => b.name;
```

**`b.cjs`**

```javascript
const a = require('./a.cjs');

exports.name = 'B';
console.log(a.name);
```

In this example, `a.cjs` assigns `name` before requiring `b.cjs`, so `b.cjs` can see it. If the required export had not yet been assigned, the other module could see an incomplete object.

### ES module behavior

ES modules use live bindings, but cyclic modules still have initialization-order constraints.

If a module accesses a binding before that binding has been initialized, it can throw a `ReferenceError` because the binding is in its temporal dead zone.

**Best practice:** Avoid circular dependencies where possible. Extract shared functionality into a third module, invert dependencies through interfaces or callbacks, or restructure the modules around clear ownership boundaries. Do not rely on partially initialized exports.

## 4. Top-level `await` and dynamic `import()`

### Top-level `await`

ES modules allow `await` outside an `async` function at the module's top level.

```javascript
// config.js
const response = await fetch(
  'https://config.example.com/settings'
);

export const settings = await response.json();
```

This is useful for asynchronous initialization, but it delays evaluation of modules that depend on this module.

In a server application, avoid scattering network calls or database connection attempts throughout imported modules. Prefer an explicit application startup sequence so failures are handled predictably.

### Dynamic `import()`

`import()` loads a module dynamically and returns a Promise.

```javascript
async function loadReportGenerator() {
  const { generateReport } = await import('./reports.js');

  return generateReport();
}
```

Use dynamic imports for optional functionality, lazy loading, or conditionally loaded modules.

Unlike static imports, the import decision can depend on runtime conditions:

```javascript
async function loadProvider(providerName) {
  if (providerName === 'stripe') {
    return import('./providers/stripe.js');
  }

  if (providerName === 'razorpay') {
    return import('./providers/razorpay.js');
  }

  throw new Error('Unsupported provider');
}
```

Validate provider names against an explicit allowlist rather than constructing arbitrary module paths from untrusted input.

## 5. How do you replace `__dirname` in ES modules?

ES modules do not provide `__dirname` and `__filename` directly. Use `import.meta.url` with Node.js utilities.

```javascript
import path from 'node:path';
import { fileURLToPath } from 'node:url';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

console.log(__dirname);
```

For paths relative to the current module, `new URL()` is often simpler:

```javascript
import { readFile } from 'node:fs/promises';

const fileUrl = new URL('./templates/email.html', import.meta.url);

const template = await readFile(fileUrl, 'utf8');
```

This avoids manually constructing a filesystem path and is convenient for module-relative resources.

---

## 6. How should you structure a maintainable Express or Fastify application?

A maintainable backend separates responsibilities without introducing unnecessary abstraction.

A practical structure is:

```text
src/
├── app.js
├── server.js
├── config/
│   ├── env.js
│   └── database.js
├── modules/
│   └── users/
│       ├── user.routes.js
│       ├── user.controller.js
│       ├── user.service.js
│       ├── user.repository.js
│       ├── user.schema.js
│       └── user.test.js
├── middleware/
│   ├── authenticate.js
│   └── error-handler.js
├── infrastructure/
│   ├── logger.js
│   ├── redis.js
│   └── email-client.js
└── shared/
    ├── errors.js
    └── constants.js
```

This is a feature-oriented structure: code for the user domain stays together, while cross-cutting infrastructure and shared utilities remain separate.

For a small API, fewer files may be sufficient. Introduce additional layers when they create a useful boundary, not merely to follow a template.

### A. Routes

Routes define the HTTP endpoints, middleware, and validation.

They should not contain business logic or direct database queries.

```javascript
// modules/users/user.routes.js
import { Router } from 'express';
import { createUser, getUser } from './user.controller.js';

const router = Router();

router.get('/:id', getUser);
router.post('/', createUser);

export default router;
```

### B. Controllers

Controllers translate HTTP requests into application operations and translate results into HTTP responses.

```javascript
// modules/users/user.controller.js
import * as userService from './user.service.js';

export async function getUser(req, res, next) {
  try {
    const user = await userService.getUserById(req.params.id);

    if (!user) {
      return res.status(404).json({
        error: 'USER_NOT_FOUND'
      });
    }

    return res.json({ data: user });
  } catch (error) {
    next(error);
  }
}

export async function createUser(req, res, next) {
  try {
    const user = await userService.createUser(req.body);

    return res.status(201).json({ data: user });
  } catch (error) {
    next(error);
  }
}
```

The controller should remain thin. It should not implement complex pricing rules, business workflows, or database transaction logic.

This example illustrates the Express pattern; production code should validate route parameters and request bodies before calling the service.

### C. Services

Services contain business rules and orchestrate operations.

```javascript
// modules/users/user.service.js
import * as userRepository from './user.repository.js';

export async function getUserById(id) {
  return userRepository.findById(id);
}

export async function createUser(input) {
  const existingUser = await userRepository.findByEmail(input.email);

  if (existingUser) {
    const error = new Error('Email already exists');
    error.code = 'EMAIL_ALREADY_EXISTS';
    throw error;
  }

  return userRepository.create({
    name: input.name,
    email: input.email
  });
}
```

In a production application, define domain-specific errors rather than attaching arbitrary properties to generic errors. Map those errors to HTTP status codes in centralized error handling.

Services are also a natural place to coordinate repositories, transactions, and external integrations.

### D. Repositories

Repositories isolate data access from business logic.

```javascript
// modules/users/user.repository.js
import { pool } from '../../config/database.js';

export async function findById(id) {
  const [rows] = await pool.execute(
    'SELECT id, name, email FROM users WHERE id = ?',
    [id]
  );

  return rows[0] ?? null;
}

export async function findByEmail(email) {
  const [rows] = await pool.execute(
    'SELECT id FROM users WHERE email = ?',
    [email]
  );

  return rows[0] ?? null;
}

export async function create(user) {
  const [result] = await pool.execute(
    'INSERT INTO users (name, email) VALUES (?, ?)',
    [user.name, user.email]
  );

  return findById(result.insertId);
}
```

This example uses the MySQL2 Promise API and parameterized SQL. It assumes the relevant schema and pool are configured.

Repositories should encapsulate persistence details, not become a second service layer containing business rules.

Also, checking for an existing email before inserting is not sufficient to guarantee uniqueness under concurrent requests. Add a unique database constraint and handle duplicate-key errors.

### E. Configuration

Validate environment variables at startup and centralize configuration.

```javascript
// config/env.js
function required(name) {
  const value = process.env[name];

  if (!value) {
    throw new Error(`Missing required environment variable: ${name}`);
  }

  return value;
}

export const config = {
  port: Number(process.env.PORT ?? 3000),
  databaseUrl: required('DATABASE_URL'),
  nodeEnv: process.env.NODE_ENV ?? 'development'
};
```

A production configuration module should also validate numeric ranges, URLs, booleans, and other expected formats. Avoid reading and interpreting environment variables independently throughout the application.

Do not commit credentials or secrets to source control.

---

## 7. Where does dependency injection fit?

Dependency injection (DI) means a component receives its dependencies rather than constructing them internally.

Without DI, a service might directly import a global database pool and email client. This makes testing and replacing those dependencies harder.

With DI, create dependencies at the application composition root and pass them to the services that need them.

```javascript
// modules/users/user.service.js
export function createUserService({ userRepository }) {
  return {
    async getUserById(id) {
      return userRepository.findById(id);
    },

    async createUser(input) {
      const existing = await userRepository.findByEmail(input.email);

      if (existing) {
        throw new Error('Email already exists');
      }

      return userRepository.create(input);
    }
  };
}
```

Wire the dependencies in the application startup code:

```javascript
import { createUserService } from './modules/users/user.service.js';
import * as userRepository from './modules/users/user.repository.js';

const userService = createUserService({
  userRepository
});
```

The repository could also be injected with a database connection or transaction manager.

Benefits include:

- Unit tests can supply fake or in-memory dependencies.
- Business logic is less tightly coupled to infrastructure.
- External services can be replaced without rewriting core logic.
- Application initialization and resource ownership become explicit.

For a modest Express application, factory functions and manual DI are often sufficient. A full DI container is useful only when the application complexity justifies it.

## 8. How would you adapt this structure to Fastify?

Fastify encourages plugin-based composition. Routes, decorators, authentication, and shared infrastructure can be registered as plugins with explicit encapsulation boundaries.

```javascript
// modules/users/user.routes.js
export default async function userRoutes(fastify) {
  fastify.get('/:id', async (request, reply) => {
    const user = await fastify.userService.getUserById(
      request.params.id
    );

    if (!user) {
      return reply.code(404).send({
        error: 'USER_NOT_FOUND'
      });
    }

    return { data: user };
  });
}
```

Register the service as a Fastify decorator or supply it through a plugin, and use Fastify's schema validation and serialization facilities where appropriate.

Fastify can handle many errors through its request lifecycle and `setErrorHandler()`. Express commonly uses middleware and `next(error)` for centralized error handling. The architectural boundaries—routes, controllers or handlers, services, repositories, and infrastructure—can remain similar in either framework.

For Fastify, avoid assuming a decorator registered inside an encapsulated plugin is automatically visible to every sibling plugin. Register dependencies at the appropriate scope or use an appropriate plugin arrangement.

## 9. Practical design rules

1. **Keep HTTP concerns in routes and controllers.** Authentication, request validation, status codes, and response serialization belong at the transport boundary.
2. **Keep business rules in services or domain modules.** Avoid coupling core logic to Express request and response objects.
3. **Keep persistence in repositories.** Use parameterized queries and enforce integrity constraints in the database.
4. **Manage transactions deliberately.** Multi-step business operations should use an explicit transaction boundary, with the same transaction connection passed through the participating repositories.
5. **Validate at boundaries.** Validate incoming requests and configuration at startup; do not rely only on TypeScript types or client-side validation.
6. **Centralize error handling.** Use domain-specific errors and map them consistently to HTTP responses.
7. **Keep startup separate from app construction.** An `app.js` factory can create and configure the application, while `server.js` owns listening, dependency initialization, and graceful shutdown.
8. **Avoid premature abstraction.** Do not create interfaces, factories, and repositories for every trivial function unless they improve testability, reuse, or separation of concerns.
9. **Test the right boundaries.** Unit-test business logic, integration-test repositories against a real database when practical, and test important HTTP workflows end to end.

## Interview summary

CommonJS uses `require()` and `module.exports`, while ES modules use static `import` and `export`, support top-level `await`, and provide dynamic loading through `import()`. Both cache modules, but circular dependencies can expose incomplete exports or trigger initialization errors. In an Express or Fastify application, keep HTTP routing and response handling separate from business services and persistence. Centralize configuration, use dependency injection to make components testable, establish clear transaction boundaries, and avoid circular dependencies and unnecessary abstraction.

*Q13. Express/Fastify internals and middleware. How does the middleware chain work (next, error-handling middleware with four arguments, async errors in Express 4 vs 5)? How do you design validation (Zod/Joi), centralized error responses, request IDs, rate limiting, helmet, CORS and body size limits? What does Fastify’s schema-based serialization and hook lifecycle offer over Express?*

## 1. How does the Express middleware chain work?

Express processes an incoming request through an ordered sequence of middleware functions. Each middleware can inspect or modify the request and response, end the response, or pass control to the next middleware.

A middleware function typically has this signature:

```javascript
function middleware(req, res, next) {
  // Perform some operation.
  next();
}
```

There are three common ways to proceed:

- `next()` — pass control to the next middleware.
- `next(error)` — transfer control to error-handling middleware.
- `res.send()`, `res.json()`, or another response-ending method — finish the response without continuing the normal chain.

Example:

```javascript
app.use((req, res, next) => {
  console.log('Request received');
  next();
});

app.get('/users', (req, res) => {
  res.json({ users: [] });
});
```

Middleware execution follows registration order. If a middleware neither calls `next()` nor ends the response, the request can hang.

Middleware should also avoid calling `next()` more than once, because doing so can cause duplicate processing or errors.

### Example: Authentication middleware

```javascript
function authenticate(req, res, next) {
  const token = req.headers.authorization;

  if (!token) {
    return res.status(401).json({
      error: {
        code: 'UNAUTHORIZED',
        message: 'Authentication required'
      }
    });
  }

  try {
    req.user = verifyToken(token);
    return next();
  } catch {
    return res.status(401).json({
      error: {
        code: 'INVALID_TOKEN',
        message: 'Invalid or expired token'
      }
    });
  }
}
```

In production, token verification should distinguish expected authentication failures from unexpected internal errors. Invalid credentials should produce a 401 response, but a database outage or unexpected programming failure should not automatically be misclassified as an invalid token.

---

## 2. What is error-handling middleware?

Express identifies error-handling middleware by its four-argument signature:

```javascript
function errorHandler(err, req, res, next) {
  // Handle the error.
}
```

When middleware calls `next(error)`, Express skips the remaining normal middleware and routes and searches for an appropriate error handler.

Example:

```javascript
app.get('/users/:id', async (req, res, next) => {
  try {
    const user = await userService.getById(req.params.id);

    if (!user) {
      return res.status(404).json({
        error: { code: 'USER_NOT_FOUND' }
      });
    }

    return res.json({ data: user });
  } catch (error) {
    return next(error);
  }
});

app.use((err, req, res, next) => {
  if (res.headersSent) {
    return next(err);
  }

  console.error(err);

  return res.status(500).json({
    error: {
      code: 'INTERNAL_SERVER_ERROR',
      message: 'An unexpected error occurred'
    }
  });
});
```

Important rules:

- Register the centralized error handler after routes and ordinary middleware.
- Do not expose stack traces, SQL details, credentials, or internal exception messages in production responses.
- If headers have already been sent, delegate to Express's default error handler with `next(err)` rather than attempting to send a second response.
- Log unexpected failures with enough context to diagnose them, while redacting sensitive data.

A four-argument handler is recognized by its function signature. Omitting the fourth parameter can cause Express to treat it as ordinary middleware instead.

## 3. How do async errors differ between Express 4 and Express 5?

This is a frequently asked interview question.

### Express 4

Express 4 does not automatically forward rejected Promises from an `async` route handler to error-handling middleware.

```javascript
app.get('/users', async (req, res) => {
  const users = await userService.getUsers();
  res.json(users);
});
```

If `getUsers()` rejects, the rejection may remain unhandled unless the route catches it or forwards the error.

A common Express 4 pattern is:

```javascript
app.get('/users', async (req, res, next) => {
  try {
    const users = await userService.getUsers();
    return res.json(users);
  } catch (error) {
    return next(error);
  }
});
```

Alternatively, use a well-tested async-handler wrapper.

### Express 5

Express 5 automatically forwards a rejected Promise returned by a route handler or middleware to `next(error)`.

```javascript
app.get('/users', async (req, res) => {
  const users = await userService.getUsers();
  return res.json(users);
});
```

If the awaited operation rejects, Express forwards the error to the error-handling chain.

However, asynchronous work started but not returned or awaited by the handler is not automatically covered:

```javascript
app.get('/users', (req, res) => {
  someAsyncTask(); // Not awaited or returned.
  res.json({ ok: true });
});
```

Errors from detached background work must be handled separately.

**Interview takeaway:** Express 5 improves Promise-based error propagation, but it does not eliminate the need to handle background-task failures or design a centralized error policy.

---

## 4. How do you design request validation with Zod or Joi?

Validation should happen at the system boundary, before untrusted input reaches business logic.

Validate:

- Path parameters.
- Query parameters.
- Request bodies.
- Headers when relevant.
- Environment variables at application startup.

A schema library such as Zod or Joi lets the application define expected shapes, types, constraints, and transformations.

### Example using Zod

```javascript
import { z } from 'zod';

const createUserSchema = z.object({
  name: z.string().trim().min(1).max(100),
  email: z.string().trim().email().max(254),
  age: z.number().int().min(18).optional()
}).strict();

function validateBody(schema) {
  return (req, res, next) => {
    const result = schema.safeParse(req.body);

    if (!result.success) {
      return res.status(400).json({
        error: {
          code: 'VALIDATION_ERROR',
          message: 'Request validation failed',
          details: result.error.issues.map((issue) => ({
            path: issue.path.join('.'),
            message: issue.message
          }))
        }
      });
    }

    req.body = result.data;
    return next();
  };
}

app.post(
  '/users',
  validateBody(createUserSchema),
  createUser
);
```

The `.strict()` setting rejects unexpected object properties. That can help prevent accidental mass assignment, but it is a design choice: some APIs intentionally allow or strip unknown fields instead.

A few additional considerations:

- Express's default JSON parser produces numeric JSON values as numbers, but form submissions and query parameters are commonly strings.
- If coercion is needed, use an explicit schema transformation and understand its edge cases.
- Avoid returning all raw validation input in error messages, since it may contain sensitive data.
- Validation does not replace authorization. A valid `userId` still must be checked against the caller's permissions.
- Database constraints must enforce invariants that can be violated by concurrent requests, such as uniqueness.

### Where should validation live?

Request shape validation belongs near the HTTP boundary. Business invariants—such as whether an account can transition to a particular state—belong in the service or domain layer.

For Joi, the same design applies using `Joi.object()`, `schema.validate()`, and consistent conversion of validation failures into a 400 response.

---

## 5. How do you design centralized error responses?

A good API uses a consistent error structure rather than returning a different JSON shape from every endpoint.

For example:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "requestId": "req-8f31"
  }
}
```

The exact response should match the API's contract. Field-level details can be included for validation errors.

A useful error-handling strategy distinguishes expected application errors from unexpected failures:

| Error | Typical HTTP status |
|---|---:|
| Invalid request data | 400 |
| Missing or invalid authentication | 401 |
| Authenticated but not authorized | 403 |
| Resource not found | 404 |
| Conflict or duplicate resource | 409 |
| Rate limit exceeded | 429 |
| Unexpected internal failure | 500 |
| Temporary service unavailability | 503 |

These are common mappings, not universal rules. For example, 422 is also widely used for semantically invalid request content.

Define application error classes:

```javascript
export class AppError extends Error {
  constructor(statusCode, code, message) {
    super(message);
    this.statusCode = statusCode;
    this.code = code;
  }
}
```

Then centralize the response policy:

```javascript
app.use((err, req, res, next) => {
  if (res.headersSent) {
    return next(err);
  }

  const isOperational = err instanceof AppError;
  const statusCode = isOperational ? err.statusCode : 500;
  const code = isOperational
    ? err.code
    : 'INTERNAL_SERVER_ERROR';

  if (!isOperational) {
    req.log?.error({ err }, 'Unhandled request error');
  }

  return res.status(statusCode).json({
    error: {
      code,
      message: isOperational
        ? err.message
        : 'An unexpected error occurred',
      requestId: req.id
    }
  });
});
```

This example assumes that `req.log` and `req.id` are installed by earlier middleware. A production handler should also normalize known parser, validation, authentication, and database errors into safe public error codes.

Do not classify an error as safe to expose merely because it is an instance of `Error`. Public messages should be deliberately defined.

---

## 6. How do you implement request IDs and structured logging?

Request IDs help correlate a request across logs, database operations, downstream services, and distributed traces.

A simple Express implementation:

```javascript
import { randomUUID } from 'node:crypto';

app.use((req, res, next) => {
  req.id = randomUUID();
  res.setHeader('X-Request-ID', req.id);
  next();
});
```

In production, a trusted reverse proxy or gateway may supply a request ID. Only trust a client-supplied ID when the architecture and validation rules make that safe; otherwise, generate your own.

Use structured logging with fields such as:

- Request ID.
- HTTP method and route.
- Status code.
- Request duration.
- Error code.
- Relevant service or trace identifiers.

Avoid logging authorization headers, passwords, full payment details, or sensitive request bodies.

For distributed tracing, use trace context and a tracing implementation rather than treating a request ID as a complete substitute for trace IDs and span IDs.

---

## 7. How do you implement rate limiting?

Rate limiting restricts how many requests a client can make within a defined interval. It helps reduce abuse, brute-force attempts, and accidental overload.

Common algorithms include:

- **Fixed window:** Simple counters per time window, but requests near window boundaries can create bursts.
- **Sliding window:** More accurate rate calculations over a moving interval.
- **Token bucket:** Allows controlled bursts while enforcing an average rate.
- **Leaky bucket:** Smooths the rate at which requests are processed.

For a single Express process, an in-memory limiter can be sufficient for development or a simple single-instance service.

For a horizontally scaled production API, use a shared store such as Redis or an appropriately designed gateway-level limiter. An in-memory counter is independent in each process and can be bypassed simply by reaching another instance.

A typical middleware configuration with `express-rate-limit` looks like this:

```javascript
import rateLimit from 'express-rate-limit';

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  limit: 10,
  standardHeaders: 'draft-8',
  legacyHeaders: false,
  message: {
    error: {
      code: 'RATE_LIMIT_EXCEEDED',
      message: 'Too many attempts. Try again later.'
    }
  }
});

app.use('/auth/login', loginLimiter);
```

The exact option names depend on the installed library version. For distributed deployments, configure a compatible shared store.

Apply different limits to different endpoints. Login, password-reset, public search, and expensive report-generation routes generally need different policies.

Also configure Express's `trust proxy` carefully when operating behind a reverse proxy; otherwise, the client IP used by the limiter may be wrong or spoofable.

---

## 8. What do Helmet and CORS do?

### Helmet

Helmet sets HTTP response headers that help protect browsers against certain classes of attacks and reduce accidental exposure.

```javascript
import helmet from 'helmet';

app.use(helmet());
```

Helmet can configure headers related to content security policy, MIME-type sniffing, framing, and other browser security behaviors.

It does not replace authentication, authorization, input validation, SQL parameterization, or secure session management.

Content Security Policy often requires application-specific configuration, particularly when the application uses external scripts, inline scripts, or embedded resources.

### CORS

Cross-Origin Resource Sharing controls whether browsers allow frontend JavaScript from one origin to read responses from another origin.

```javascript
import cors from 'cors';

app.use(cors({
  origin: [
    'https://app.example.com',
    'https://admin.example.com'
  ],
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true
}));
```

With `credentials: true`, use explicit allowed origins rather than `*`.

Important distinction: **CORS is a browser-enforced cross-origin policy, not an authentication or authorization mechanism.** Non-browser clients can still call the API, so the server must independently enforce access control.

For cookie-based authentication, configure cookie attributes, CSRF protection where appropriate, and credentialed CORS consistently. A permissive CORS policy can expose browser-accessible API responses to unintended origins.

---

## 9. Why are body-size limits important?

An API should not accept arbitrarily large request bodies. Oversized payloads can consume memory, CPU, and bandwidth, causing denial of service or resource exhaustion.

Configure explicit limits:

```javascript
app.use(express.json({
  limit: '1mb'
}));

app.use(express.urlencoded({
  extended: true,
  limit: '100kb'
}));
```

Choose limits based on the endpoint's real requirements. A small JSON API might need only a few hundred kilobytes, while an upload endpoint should use a streaming upload design rather than increasing the JSON body limit to hundreds of megabytes.

For file uploads:

- Enforce file-size limits at the reverse proxy and application layers.
- Stream large files rather than buffering them fully in memory.
- Validate file content, not only filename extensions or client-provided MIME types.
- Apply upload timeouts and concurrency limits.
- Consider direct-to-object-storage uploads for large files.

Also remember that `express.json()` parses JSON bodies into memory. A small configured limit is a useful safety boundary, not a complete resource-management strategy.

---

## 10. What does Fastify offer over Express?

Both Express and Fastify can build production APIs. Fastify emphasizes schema-driven request validation, response serialization, plugin encapsulation, and a structured lifecycle.

### A. Schema-based validation and serialization

Fastify supports JSON Schema for validating requests and serializing responses.

```javascript
fastify.post('/users', {
  schema: {
    body: {
      type: 'object',
      required: ['name', 'email'],
      additionalProperties: false,
      properties: {
        name: { type: 'string', minLength: 1, maxLength: 100 },
        email: { type: 'string', format: 'email' }
      }
    },
    response: {
      201: {
        type: 'object',
        properties: {
          id: { type: 'integer' },
          name: { type: 'string' },
          email: { type: 'string' }
        },
        required: ['id', 'name', 'email']
      }
    }
  }
}, async (request, reply) => {
  const user = await userService.createUser(request.body);
  return reply.code(201).send(user);
});
```

Fastify can compile schemas for efficient validation and use its response serializer to produce output matching the declared response schema.

Response schemas can also help prevent accidental disclosure of internal fields by limiting what is serialized. They should not replace deliberate data-access controls or authorization.

Only use trusted schemas and validators; do not allow arbitrary untrusted schemas to be compiled dynamically.

### B. Hook lifecycle

Fastify exposes lifecycle hooks that let developers run logic at specific stages of request processing.

Common hooks include:

- `onRequest` — early request processing, before body parsing.
- `preParsing` — inspect or transform the incoming payload stream.
- `preValidation` — run logic before schema validation.
- `preHandler` — run after validation, before the route handler.
- `preSerialization` — modify a payload before serialization.
- `onSend` — inspect or modify the outgoing payload.
- `onResponse` — run after the response has been sent.
- `onError` — observe request errors.

These hooks make it easier to implement authentication, instrumentation, request enrichment, and response processing at the appropriate lifecycle stage.

The lifecycle also includes plugin registration and encapsulation, which lets routes and hooks be scoped to specific parts of the application.

### C. Plugin encapsulation

Fastify plugins provide explicit composition boundaries. Decorators and hooks registered within a plugin are generally scoped according to the plugin tree, unless registration choices alter that behavior.

This can help organize authentication policies, shared services, and route-specific functionality without applying every hook globally.

### D. Performance and trade-offs

Fastify is designed to minimize framework overhead and can perform well for high-throughput APIs. However, real application performance also depends on database queries, network latency, serialization costs, validation complexity, logging, and business logic.

Do not claim that Fastify will automatically make every API faster. Benchmark representative workloads and compare operational needs, ecosystem support, team familiarity, and maintainability.

---

## 11. Express vs. Fastify: Interview comparison

| Concern | Express | Fastify |
|---|---|---|
| Core request pipeline | Ordered middleware and routes | Route lifecycle with hooks |
| Async route errors | Express 4 requires explicit forwarding; Express 5 forwards rejected handler Promises | Async handlers and hooks integrate with Fastify's error lifecycle |
| Validation | Usually added through middleware or libraries | Built-in JSON Schema integration |
| Response serialization | Usually handled manually or through additional libraries | Schema-driven serialization support |
| Extension model | Middleware and routers | Plugins, decorators, hooks, encapsulation |
| Ecosystem | Large, mature ecosystem | Strong plugin ecosystem with a more structured framework model |
| Best choice | Simple APIs, existing Express applications, broad middleware compatibility | Schema-driven APIs, structured lifecycle, plugin encapsulation |

## Interview summary

Express middleware executes in registration order, and `next(error)` transfers control to four-argument error-handling middleware. Express 4 requires explicit handling of rejected route Promises, whereas Express 5 forwards them automatically. A robust API validates input at the boundary, returns consistent sanitized errors, correlates logs with request IDs, and applies rate limits, Helmet, CORS, and body-size limits according to its threat model. Fastify adds schema-driven validation and response serialization, lifecycle hooks, and plugin encapsulation, which can improve consistency and reduce framework overhead when used appropriately.

*Q14. Authentication, security and dependency risk. How do you implement JWT access plus refresh tokens or sessions securely in Node? What do you do about prototype pollution, ReDoS, SQL/NoSQL injection, SSRF and path traversal on file endpoints? How do you manage npm supply-chain risk (lockfiles, npm audit, pinning, --ignore-scripts)?*

## 1. How do you implement authentication securely in Node.js?

Authentication verifies **who the user is**; authorization determines **what the user is allowed to do**. I implement authentication using either server-side sessions or JWT access tokens with refresh tokens, depending on the application's architecture.

### A. JWT access tokens + refresh tokens

A JWT-based system typically uses two tokens:

- **Access token:** Short-lived, usually around 5–15 minutes, used to authorize API requests.
- **Refresh token:** Longer-lived, used to obtain a new access token without requiring the user to log in again.

**Authentication flow:**

1. The user submits credentials over HTTPS.
2. The server verifies the password using Argon2id or bcrypt.
3. The server issues a short-lived, signed access token and a refresh token.
4. The client sends the access token in the `Authorization: Bearer <token>` header.
5. When the access token expires, the client sends the refresh token to a dedicated refresh endpoint.
6. The server validates the refresh token, rotates it, and issues a new token pair.
7. On logout, the server revokes the refresh token or its token family.

Example using the `jose` library:

```javascript
import { SignJWT, jwtVerify } from "jose";
import { randomUUID } from "node:crypto";

const secret = new TextEncoder().encode(
  process.env.JWT_ACCESS_SECRET
);

export async function createAccessToken(user) {
  return new SignJWT({
    role: user.role,
  })
    .setProtectedHeader({ alg: "HS256" })
    .setSubject(String(user.id))
    .setIssuer("my-api")
    .setAudience("my-client")
    .setIssuedAt()
    .setJti(randomUUID())
    .setExpirationTime("10m")
    .sign(secret);
}

export async function verifyAccessToken(token) {
  const { payload } = await jwtVerify(token, secret, {
    algorithms: ["HS256"],
    issuer: "my-api",
    audience: "my-client",
  });

  return payload;
}
```

The important security controls are:

- Explicitly allow the expected signing algorithm; never trust an algorithm selected by an untrusted token.
- Validate the signature, expiration, issuer, audience, and relevant claims.
- Keep signing secrets in a secret manager or protected environment configuration, never in source control.
- Use HTTPS everywhere.
- Never put passwords, API keys, or sensitive personal data in JWT payloads. Signed JWTs are not necessarily encrypted.
- Perform authorization checks on every protected operation, not just at login.
- Keep access tokens short-lived and refresh tokens revocable.

**Refresh-token security:** Generate refresh tokens using a cryptographically secure random generator. Store a hash of each token server-side, associate it with a user, device or session, and expiry, and rotate it atomically after every successful use. If an already-used token is presented again, treat it as possible token theft and revoke the associated token family.

A JWT access token generally remains valid until expiry unless the server checks a revocation mechanism. Short expiry reduces this exposure; it does not provide immediate revocation by itself.

### B. Server-side sessions

For traditional web applications, server-side sessions can be simpler and easier to revoke.

The flow is:

1. Authenticate the user.
2. Create a session record in Redis or a database.
3. Send the browser an opaque session ID in a cookie.
4. On each request, look up the session and verify that it is valid.
5. On logout or session expiry, invalidate the server-side session.

Example with Express and `express-session`:

```javascript
import session from "express-session";
import RedisStore from "connect-redis";
import { createClient } from "redis";

const redis = createClient({
  url: process.env.REDIS_URL,
});

await redis.connect();

app.use(
  session({
    store: new RedisStore({ client: redis }),
    secret: process.env.SESSION_SECRET,
    resave: false,
    saveUninitialized: false,
    name: "__Host-session",
    cookie: {
      httpOnly: true,
      secure: true,
      sameSite: "lax",
      path: "/",
      maxAge: 30 * 60 * 1000,
    },
  })
);
```

This is a configuration example; production deployment also needs appropriate proxy configuration, session lifecycle handling, and a strong secret. Do not use Express's default in-memory session store in production.

After successful login, regenerate the session ID to prevent session fixation:

```javascript
req.session.regenerate((err) => {
  if (err) return next(err);

  req.session.userId = user.id;
  res.json({ message: "Login successful" });
});
```

**JWT vs. sessions:**

| Consideration | JWT access + refresh | Server-side sessions |
|---|---|---|
| Request validation | Verify signed token | Look up session |
| Immediate revocation | Requires server-side checks or short expiry | Invalidate session centrally |
| Shared infrastructure | Signing keys and possibly token storage | Shared session store |
| Browser storage | Prefer secure cookie or carefully managed memory | Secure, HTTP-only cookie |
| Best fit | Distributed APIs and service integrations | Traditional web apps and centralized login |

For browser-based applications, I avoid storing long-lived tokens in `localStorage` because JavaScript running through an XSS vulnerability can read them. HTTP-only cookies reduce token theft through JavaScript, but cookie-based authentication also requires CSRF protection appropriate to the application's design. `SameSite` is useful, but should not be treated as a universal substitute for CSRF defenses.

---

## 2. How do you prevent prototype pollution?

Prototype pollution occurs when an attacker manipulates special object properties such as `__proto__`, `constructor`, or `prototype`, potentially modifying inherited properties and changing application behavior.

A common source is unsafe merging of untrusted JSON into configuration objects.

**Vulnerable pattern:**

```javascript
const config = Object.assign(defaultConfig, req.body);
```

The risk depends on the merge operation, object shape, and downstream code. Recursive merge utilities and dynamic property assignment are especially common sources of prototype pollution.

**Prevention:**

1. Validate request bodies against an explicit schema.
2. Allow only known fields rather than copying arbitrary input into internal objects.
3. Avoid unsafe recursive merge utilities.
4. Reject dangerous property names when handling arbitrary nested objects.
5. Use `Map` or `Object.create(null)` for suitable dictionary-like data structures.
6. Keep dependencies patched and test untrusted nested input.

Example using Zod:

```javascript
import { z } from "zod";

const updateUserSchema = z.object({
  displayName: z.string().trim().min(1).max(80),
  email: z.string().email(),
}).strict();

const input = updateUserSchema.parse(req.body);

await userService.updateProfile(req.user.id, input);
```

The strict schema rejects unexpected properties rather than blindly merging them. For nested data, validate each nested structure as well.

Using `Object.create(null)` can remove the normal prototype from a dictionary, but it is not a replacement for input validation or safe object-handling logic.

---

## 3. How do you prevent ReDoS?

Regular Expression Denial of Service (ReDoS) occurs when a regular expression takes excessive time to process a carefully crafted input. Certain patterns have catastrophic backtracking and can block Node.js's event loop.

For example, a pattern such as:

```javascript
/^(a+)+$/
```

can take disproportionately long to reject some long strings consisting of repeated `a` characters followed by a different character.

**Prevention:**

- Avoid nested, ambiguous quantifiers and overlapping alternatives in regex patterns.
- Set strict input length limits before running expensive validation.
- Prefer simple, well-understood patterns for email, identifiers, and user input.
- Use vetted validation libraries and review their regex behavior.
- Consider a linear-time regex engine or a safe alternative for complex matching.
- Run potentially expensive work in a worker or separate process if isolation is needed.

A timeout around a synchronous regex does **not** solve the problem: while the regex blocks the event loop, the same event loop cannot execute the timeout callback.

For example:

```javascript
function validateIdentifier(value) {
  if (typeof value !== "string" || value.length > 100) {
    return false;
  }

  return /^[a-zA-Z0-9_-]+$/.test(value);
}
```

This example uses a bounded input and a straightforward character-class pattern. The general principle is to bound attacker-controlled work, not merely to catch errors after it becomes expensive.

---

## 4. How do you prevent SQL and NoSQL injection?

### SQL injection

SQL injection happens when untrusted input is concatenated into SQL text and changes the meaning of the query.

**Unsafe:**

```javascript
const sql =
  `SELECT * FROM users WHERE email = '${req.body.email}'`;
```

**Safe:**

```javascript
const [rows] = await pool.execute(
  "SELECT id, email FROM users WHERE email = ?",
  [req.body.email]
);
```

Parameterized queries separate query structure from data. I use them for values and avoid building SQL fragments from arbitrary user input.

Some SQL elements, such as table names and sort directions, cannot generally be supplied as ordinary value parameters. For those, use a strict allowlist:

```javascript
const allowedSortColumns = {
  name: "name",
  createdAt: "created_at",
};

const sortColumn =
  allowedSortColumns[req.query.sort] ?? "created_at";

const rows = await query(
  `SELECT id, name FROM users ORDER BY ${sortColumn}`
);
```

Only trusted, predefined column names are interpolated into the query.

### NoSQL injection

NoSQL injection can occur when user input is treated as a query operator rather than ordinary data. For example, an attacker might submit an object containing `$ne` where the application expects a username string.

**Unsafe:**

```javascript
const user = await User.findOne({
  username: req.body.username,
  password: req.body.password,
});
```

If those fields are accepted as arbitrary objects, query semantics may be manipulated. This example also incorrectly treats password comparison as a database lookup.

**Safer approach:**

```javascript
import { z } from "zod";

const loginSchema = z.object({
  username: z.string().min(1).max(100),
  password: z.string().min(1).max(200),
}).strict();

const { username, password } = loginSchema.parse(req.body);

const user = await User.findOne({ username });

if (!user || !(await verifyPassword(password, user.passwordHash))) {
  throw new UnauthorizedError("Invalid credentials");
}
```

Here, the input schema requires strings, the database lookup uses the validated username, and the password is checked against a stored password hash.

For MongoDB, I also avoid passing arbitrary client-supplied objects directly into filters or update operators, and I restrict which fields clients can update.

**Key distinction:** Parameterization protects SQL values; schema validation and controlled query construction prevent untrusted input from becoming database operators or query structure.

---

## 5. How do you prevent SSRF?

Server-Side Request Forgery (SSRF) occurs when an attacker tricks the backend into making requests to unintended destinations, such as cloud metadata endpoints, internal services, or localhost.

For example, an application that fetches an image from a user-provided URL may accidentally expose internal network resources.

**Prevention strategy:**

1. Prefer accepting a known resource ID rather than an arbitrary URL.
2. Allowlist approved domains or destinations whenever possible.
3. Permit only required protocols, usually HTTPS.
4. Reject loopback, private, link-local, and other prohibited IP ranges for both IPv4 and IPv6.
5. Resolve DNS and validate the resulting addresses before connecting.
6. Revalidate every redirect destination or disable redirects.
7. Apply outbound firewall rules so the application cannot reach sensitive internal networks.
8. Set connection and total request timeouts, response-size limits, and rate limits.

A URL's hostname must not be trusted simply because it looks legitimate. DNS rebinding and redirects can bypass naive hostname checks. A robust implementation validates the actual connection destination and prevents the HTTP client from independently resolving to a prohibited address after validation.

**Important:** Do not rely solely on a regex that rejects strings containing `localhost` or `127.0.0.1`. IP representations, IPv6, DNS resolution, redirects, and alternate hostnames make such checks insufficient.

For a URL-fetching feature, I would use a well-maintained HTTP client, explicit destination validation, restricted egress networking, and strict response limits. If arbitrary URLs are not a real requirement, I would remove that capability entirely.

---

## 6. How do you prevent path traversal in file endpoints?

Path traversal occurs when a user manipulates a file path, for example using `../../`, to access files outside the intended directory.

**Unsafe:**

```javascript
const filePath = path.join(
  uploadDirectory,
  req.params.filename
);

res.sendFile(filePath);
```

This is unsafe if the endpoint does not enforce containment and other file-access restrictions.

A safer approach is to resolve the candidate path and verify that it stays within the intended root:

```javascript
import path from "node:path";

function resolveUploadPath(rootDirectory, filename) {
  // Prefer rejecting separators and accepting only a defined
  // filename format, rather than accepting arbitrary paths.
  if (
    typeof filename !== "string" ||
    filename.length === 0 ||
    filename.includes("/") ||
    filename.includes("\\") ||
    filename === "." ||
    filename === ".."
  ) {
    throw new Error("Invalid filename");
  }

  const root = path.resolve(rootDirectory);
  const candidate = path.resolve(root, filename);
  const relative = path.relative(root, candidate);

  if (
    relative === "" ||
    relative.startsWith(`..${path.sep}`) ||
    relative === ".." ||
    path.isAbsolute(relative)
  ) {
    throw new Error("Invalid file path");
  }

  return candidate;
}
```

This illustrates lexical path containment; it is not a complete defense for every filesystem scenario.

For production file endpoints, I also:

- Prefer opaque file IDs mapped to server-controlled paths.
- Store uploads outside the web root.
- Generate server-side filenames instead of trusting client filenames.
- Enforce file size, extension, MIME type, and actual file-content checks.
- Prevent executable uploads and apply malware scanning when appropriate.
- Consider symbolic-link attacks and filesystem race conditions.
- Enforce authorization on every download or deletion.

A file belonging to another user must remain inaccessible even if the path itself is valid. Path validation and authorization solve different problems.

---

## 7. How do you manage npm supply-chain risk?

The npm supply chain includes direct dependencies, transitive dependencies, package maintainers, installation scripts, registries, and build tools. A compromised or vulnerable dependency can introduce malicious code or expose sensitive data.

### A. Commit and enforce the lockfile

I commit `package-lock.json` and use:

```bash
npm ci
```

`npm ci` installs the dependency tree defined by the lockfile and fails when `package.json` and the lockfile are inconsistent. It is preferable for reproducible CI builds.

I also:

- Review lockfile changes in pull requests.
- Use consistent Node.js and npm versions in CI.
- Avoid routinely deleting and regenerating the lockfile without a reason.
- Pin direct dependency versions where reproducibility or change control requires it.
- Update dependencies through reviewed, automated pull requests.

A lockfile improves reproducibility; it does **not** guarantee that packages are safe.

### B. Audit dependencies

```bash
npm audit
npm audit --omit=dev
npm outdated
```

- `npm audit` checks the dependency tree against known vulnerability advisories.
- `npm audit --omit=dev` focuses the audit request on the production dependency tree.
- `npm outdated` identifies packages with newer published versions.

I assess severity, exploitability, whether the vulnerable code path is used, and whether a safe fix is available. I update affected packages, run tests, and deploy through the normal release process.

I avoid blindly running:

```bash
npm audit fix --force
```

because it can introduce breaking major-version changes. An audit also cannot detect every malicious package or unknown vulnerability.

### C. Pinning and package selection

I prefer maintained packages with a clear ownership history, sensible release practices, and a reasonable dependency footprint. I review new dependencies before adoption and avoid unnecessary packages.

For higher-risk environments, I also consider:

- Automated vulnerability and license scanning.
- Dependency review in pull requests.
- SBOM generation and dependency provenance where supported.
- Private registry controls and trusted publishing mechanisms.
- Monitoring for compromised maintainers, typosquatting, and suspicious releases.

Version pinning reduces unexpected updates, but a pinned vulnerable or compromised version remains vulnerable. Pinning must be paired with a patching process.

### D. Control installation scripts

npm packages can execute lifecycle scripts such as `preinstall`, `install`, and `postinstall`. Malicious scripts may run during dependency installation in CI or developer environments.

For a controlled CI installation, I can use:

```bash
npm ci --ignore-scripts
```

This suppresses package lifecycle scripts during installation. It reduces one avenue of supply-chain execution, but it is **not a complete security boundary**. Some packages need installation scripts to compile native modules or download required binaries, so I verify compatibility and explicitly allow only necessary trusted build steps where appropriate.

I never assume that `--ignore-scripts` makes an untrusted package safe: its code can still execute later when the application or build tooling imports or runs it.

### E. Secure CI/CD credentials

- Use short-lived, least-privilege CI credentials.
- Avoid exposing publishing tokens to untrusted pull-request builds.
- Separate dependency installation, build, test, and release permissions where practical.
- Run builds with restricted network access and filesystem permissions.
- Keep secrets out of logs and avoid running arbitrary third-party code in privileged environments.

The objective is to limit the impact if a dependency or build step is compromised.

---

## 8. What is your overall security approach?

I use defense in depth:

1. **Identity:** Strong password hashing, secure token/session lifecycle, and MFA where appropriate.
2. **Authorization:** Deny by default and verify ownership and permissions on every protected resource.
3. **Input handling:** Validate schemas, restrict accepted fields, parameterize database queries, and bound input sizes.
4. **Resource protection:** Rate limits, timeouts, response-size limits, and protection against event-loop blocking.
5. **Network and filesystem isolation:** Restrict outbound access and confine file operations to authorized locations.
6. **Dependency security:** Lockfiles, audits, reviewed updates, controlled install scripts, and least-privilege CI.
7. **Monitoring:** Structured security logs, alerts for suspicious authentication activity, dependency vulnerability tracking, and incident-response procedures.

I would also test these controls with negative cases: expired or reused refresh tokens, privilege escalation attempts, malicious nested objects, pathological regex input, injection payloads, internal-address SSRF attempts, traversal filenames, and intentionally vulnerable dependencies.

**Interview summary:** I secure Node.js applications by enforcing explicit authentication and authorization, validating untrusted input, using parameterized queries, restricting filesystem and network access, and controlling the software supply chain. No single technique is sufficient: JWTs need revocation and lifecycle controls, validation does not replace authorization, and lockfiles or `npm audit` do not guarantee dependency integrity.

*Q15. Production readiness and observability. How do you configure logging (pino, structured JSON, correlation IDs), metrics (Prometheus, event loop lag, heap), tracing (OpenTelemetry), health/readiness endpoints, and connection pooling to MySQL or Postgres? What belongs in a Dockerfile for Node (non-root user, multi-stage build, NODE_ENV=production, signal handling with tini)?*

Production readiness means an application can be operated reliably under real traffic, diagnosed when something fails, and shut down or deployed without unnecessarily interrupting users. I focus on five areas: **structured logging, metrics, distributed tracing, health checks, and resource management**, followed by a secure and efficient Docker deployment.

## 1. Logging with Pino and structured JSON

I use a structured logger such as Pino instead of scattered `console.log()` statements. Structured JSON logs are machine-readable and can be searched, filtered, and correlated in systems such as Elasticsearch, Loki, or a cloud logging platform.

A useful log record includes:

- Timestamp and severity level.
- Service name, environment, and application version.
- Request or correlation ID.
- HTTP method, route, status code, and duration.
- Error type, message, and stack trace when appropriate.

Example:

```javascript
import pino from "pino";

export const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  redact: {
    paths: [
      "req.headers.authorization",
      "req.headers.cookie",
      "password",
      "accessToken",
      "refreshToken",
    ],
    censor: "[REDACTED]",
  },
  base: {
    service: "orders-api",
    environment: process.env.NODE_ENV ?? "development",
  },
});
```

Usage:

```javascript
logger.info(
  { orderId: order.id, userId: user.id },
  "Order created"
);

logger.error(
  { err, requestId },
  "Order processing failed"
);
```

I use consistent field names and log levels: `debug`, `info`, `warn`, and `error`. I avoid logging passwords, tokens, database credentials, payment details, or unnecessary personal data.

### Correlation IDs

A correlation ID lets me follow a request through application logs and downstream services. I generate an ID at the API boundary or accept an incoming ID only after validating it according to a defined format and trust policy.

Example Express middleware:

```javascript
import { randomUUID } from "node:crypto";

app.use((req, res, next) => {
  const requestId = randomUUID();

  req.requestId = requestId;
  res.setHeader("X-Request-ID", requestId);

  req.log = logger.child({ requestId });
  next();
});
```

A production implementation should also ensure that downstream HTTP clients propagate the appropriate trace context and request metadata. If a trusted gateway supplies a correlation ID, the application can preserve it after validation rather than accepting arbitrary unbounded values from every caller.

**Important distinction:** A correlation ID is useful for log searches, but it is not automatically a distributed trace ID. OpenTelemetry provides standardized trace and span identifiers for tracking work across services.

---

## 2. Metrics with Prometheus

Logs explain individual events; metrics reveal trends and system-wide behavior.

I monitor four categories.

| Category | Examples |
|---|---|
| Traffic | Requests per second, requests by route |
| Errors | HTTP 5xx rate, failed jobs, database errors |
| Latency | p50, p95, p99 request duration |
| Saturation | CPU, memory, event-loop lag, DB pool usage |

I expose application metrics through a Prometheus-compatible endpoint, commonly `/metrics`, using a library such as `prom-client`.

Example:

```javascript
import client from "prom-client";

client.collectDefaultMetrics({
  register: client.register,
});

const httpDuration = new client.Histogram({
  name: "http_request_duration_seconds",
  help: "HTTP request duration in seconds",
  labelNames: ["method", "route", "status_code"],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2, 5],
});

app.use((req, res, next) => {
  const start = process.hrtime.bigint();

  res.once("finish", () => {
    const duration =
      Number(process.hrtime.bigint() - start) / 1e9;

    httpDuration.observe(
      {
        method: req.method,
        route: req.route?.path ?? "unmatched",
        status_code: String(res.statusCode),
      },
      duration
    );
  });

  next();
});

app.get("/metrics", async (req, res) => {
  res.set("Content-Type", client.register.contentType);
  res.end(await client.register.metrics());
});
```

This is a starting point. In a real application, I register instrumentation once, avoid recording raw URLs containing IDs or sensitive data, and ensure the route label has bounded cardinality. I also handle unmatched routes and metrics collection carefully.

### Why cardinality matters

A metric labeled by `userId`, full URL, or request ID can create millions of distinct time series. This increases Prometheus memory usage and makes queries expensive.

I use low-cardinality labels such as HTTP method, normalized route, and status class. High-cardinality identifiers belong in logs or traces instead.

### Event-loop lag

Node.js runs JavaScript callbacks on an event loop. Long synchronous operations can prevent it from processing incoming requests even when CPU or memory dashboards look acceptable.

I monitor event-loop delay with Node's `perf_hooks`:

```javascript
import { monitorEventLoopDelay } from "node:perf_hooks";

const histogram = monitorEventLoopDelay({
  resolution: 20,
});

histogram.enable();

setInterval(() => {
  const p99Ms = histogram.percentile(99) / 1e6;
  const meanMs = histogram.mean / 1e6;

  logger.info({
    eventLoopDelayP99Ms: p99Ms,
    eventLoopDelayMeanMs: meanMs,
  });

  histogram.reset();
}, 10_000).unref();
```

This example logs the values; production instrumentation should export them as gauges or histograms and handle the no-sample case. I correlate event-loop lag with CPU usage, garbage collection, latency, and throughput to identify whether synchronous computation or other blocking work is causing delays.

### Heap and memory monitoring

I track `process.memoryUsage()`:

```javascript
const memory = process.memoryUsage();

logger.info({
  rssBytes: memory.rss,
  heapUsedBytes: memory.heapUsed,
  heapTotalBytes: memory.heapTotal,
  externalBytes: memory.external,
  arrayBuffersBytes: memory.arrayBuffers,
}, "Process memory");
```

- `rss`: total resident memory used by the process, including more than the JavaScript heap.
- `heapUsed`: memory occupied by JavaScript objects in use.
- `heapTotal`: memory currently allocated for the V8 heap.
- `external`: native memory associated with JavaScript objects.
- `arrayBuffers`: memory used by `ArrayBuffer` and related buffers.

A steadily rising heap after garbage collection may indicate a memory leak. Rising RSS with a stable heap may point to native allocations, buffers, fragmentation, or other non-heap usage.

I also monitor container memory limits and restart counts. I do not automatically treat every increase in memory as a leak.

---

## 3. Distributed tracing with OpenTelemetry

Metrics show that a problem exists; logs provide event details; distributed traces help identify where latency or failure occurs across a request's path.

For example, an order request might call:

1. Node.js API.
2. Authentication service.
3. PostgreSQL.
4. Payment provider.
5. Message broker.

A distributed trace represents this work as a trace containing spans. Each span records an operation's timing, status, and relevant attributes.

I use OpenTelemetry for standardized instrumentation and export traces to a backend such as Jaeger, Grafana Tempo, or a compatible observability platform.

A simplified setup using the Node SDK:

```javascript
// instrumentation.mjs
import { NodeSDK } from "@opentelemetry/sdk-node";
import { getNodeAutoInstrumentations } from
  "@opentelemetry/auto-instrumentations-node";
import { OTLPTraceExporter } from
  "@opentelemetry/exporter-trace-otlp-http";

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_TRACES_ENDPOINT,
  }),
  instrumentations: [
    getNodeAutoInstrumentations(),
  ],
});

sdk.start();
```

Load the instrumentation before importing the application modules it needs to instrument:

```bash
node --import ./instrumentation.mjs server.mjs
```

The exact initialization and shutdown APIs depend on the installed OpenTelemetry SDK version and deployment setup.

I instrument important operations such as database queries, outbound HTTP requests, and queue processing. I propagate W3C Trace Context across service boundaries and add useful attributes without recording credentials or sensitive request bodies.

**Sampling:** Tracing every request may be expensive at high traffic volumes. I use an appropriate sampling strategy, potentially retaining errors and slow traces at higher rates, depending on the tracing backend and operational requirements.

I also shut down the tracing SDK during graceful application shutdown so buffered telemetry has an opportunity to be exported.

---

## 4. Health checks and readiness endpoints

I distinguish between **liveness** and **readiness** because they answer different questions.

### Liveness: Is the process alive?

A liveness endpoint indicates whether the process is functioning sufficiently to continue running. It should generally be lightweight and should not fail simply because an external dependency is temporarily unavailable.

```javascript
app.get("/health/live", (req, res) => {
  res.status(200).json({ status: "alive" });
});
```

If liveness fails, an orchestrator may restart the container. Incorrect liveness checks can create restart loops during database outages.

### Readiness: Can the instance serve traffic?

Readiness checks whether the application has completed startup and can handle requests, including whether critical dependencies are usable.

```javascript
app.get("/health/ready", async (req, res) => {
  try {
    await pool.query("SELECT 1");

    res.status(200).json({
      status: "ready",
    });
  } catch {
    res.status(503).json({
      status: "not_ready",
    });
  }
});
```

This is a PostgreSQL-style example; the query API differs between database drivers. The route assumes `pool` is already initialized.

For production, I also check startup completion and any other genuinely critical dependencies. Checks should have short timeouts, avoid expensive queries, and not expose credentials or internal infrastructure details.

A Redis or message-broker outage should make readiness fail only if that dependency is essential to serving the instance safely. Otherwise, the application may be able to continue operating in a degraded mode.

I keep the following distinction clear:

- **Liveness:** Should this process be restarted?
- **Readiness:** Should this instance receive traffic?
- **Startup probe:** Has the application completed initialization?

---

## 5. Database connection pooling: MySQL and PostgreSQL

Creating a database connection for every HTTP request adds connection overhead and can overwhelm the database under load. I use a connection pool to reuse established connections and limit concurrency.

### MySQL example

Using `mysql2/promise`:

```javascript
import mysql from "mysql2/promise";

export const pool = mysql.createPool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  waitForConnections: true,
  connectionLimit: 10,
  queueLimit: 100,
});
```

Usage:

```javascript
const [rows] = await pool.execute(
  "SELECT id, email FROM users WHERE id = ?",
  [userId]
);
```

### PostgreSQL example

Using `pg`:

```javascript
import pg from "pg";

const pool = new pg.Pool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  max: 10,
  connectionTimeoutMillis: 3000,
  idleTimeoutMillis: 30000,
});
```

Usage:

```javascript
const result = await pool.query(
  "SELECT id, email FROM users WHERE id = $1",
  [userId]
);
```

The placeholders differ: MySQL commonly uses `?`, while PostgreSQL uses `$1`, `$2`, and so on.

### How do you choose pool size?

Pool size is not simply the number of CPU cores. I consider:

- Expected concurrent database operations.
- Query duration and transaction duration.
- Database connection limits.
- Number of application instances and workers.
- Other consumers, such as background jobs and administration tools.

For example, if eight application instances each permit 20 connections, they can collectively consume up to 160 database connections, before counting other clients.

I size pools against the database's total connection budget, load-test under realistic concurrency, and monitor active, idle, waiting, and timed-out connections.

### Transactions and connection release

For a multi-statement transaction, I acquire one connection and release it in a `finally` block:

```javascript
const connection = await pool.getConnection();

try {
  await connection.beginTransaction();

  await connection.execute(
    "UPDATE accounts SET balance = balance - ? WHERE id = ?",
    [amount, sourceId]
  );

  await connection.execute(
    "UPDATE accounts SET balance = balance + ? WHERE id = ?",
    [amount, destinationId]
  );

  await connection.commit();
} catch (error) {
  await connection.rollback();
  throw error;
} finally {
  connection.release();
}
```

This is illustrative; a real transfer must validate balances, amounts, ownership, and concurrency conditions, and must handle rollback failures appropriately. A checked-out connection must always be released, including when an operation fails.

For PostgreSQL, I use `pool.connect()` and `client.release()` for explicit transactions. Calling `pool.query()` independently for each transaction statement is incorrect because those statements may run on different connections.

I also set query and transaction timeouts where appropriate, avoid holding connections while waiting for external APIs, and close pools gracefully during shutdown.

---

## 6. What belongs in a production Node.js Dockerfile?

A production Docker image should be reproducible, minimal, secure, and easy to operate. I typically use a multi-stage build when the project has a build step, run the process as a non-root user, install only production dependencies in the runtime stage, and ensure the process receives termination signals correctly.

Example for a JavaScript Node API that does not require a compilation step:

```dockerfile
# syntax=docker/dockerfile:1

FROM node:22-bookworm-slim AS dependencies
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

FROM dependencies AS production-dependencies
RUN npm prune --omit=dev

FROM node:22-bookworm-slim AS runtime
ENV NODE_ENV=production

WORKDIR /app

COPY --from=production-dependencies --chown=node:node \
  /app/node_modules ./node_modules

COPY --chown=node:node package.json ./
COPY --chown=node:node src ./src

USER node

EXPOSE 3000

CMD ["node", "src/server.js"]
```

This example uses separate dependency stages, but for TypeScript or another compiled application I would add a build stage and copy only the compiled output and production dependencies into the runtime stage. I would also verify that runtime assets and package files required by the application are included.

For a stronger production setup, I pin the base image to a reviewed version or digest, update it regularly, scan it for vulnerabilities, and use a `.dockerignore` file to exclude `.git`, local `node_modules`, `.env` files, tests or other unnecessary files as appropriate.

### A. Why multi-stage builds?

A multi-stage build separates build-time tools and development dependencies from the final runtime image.

Benefits include:

- Smaller images and reduced attack surface.
- No need to ship compilers or development tooling when they are not required at runtime.
- More predictable deployments.
- Better separation between building and running the application.

I use a `.dockerignore` file to avoid copying local dependencies, credentials, and unnecessary files into the build context.

### B. Why run as a non-root user?

If an attacker compromises the application process, a non-root user limits the privileges available inside the container.

The official Node image includes a `node` user. I use `USER node` and ensure the application only has write access to the directories it needs. I do not grant unnecessary Linux capabilities or mount sensitive host paths into the container.

A non-root user is one layer of defense, not a replacement for container isolation or host security.

### C. Why `NODE_ENV=production`?

```dockerfile
ENV NODE_ENV=production
```

This communicates that the application is running in production mode. Libraries and frameworks may use it to disable development behavior, enable production optimizations, or alter logging.

It does not itself secure the application or guarantee that development dependencies are excluded. I explicitly install the appropriate dependency set and verify the final image.

### D. Why use `tini`?

A container's main process should handle `SIGTERM` and `SIGINT` correctly. When Node.js runs as PID 1, signal handling and orphaned-child reaping can require extra care.

A small init process such as `tini` forwards signals and reaps orphaned child processes. This is useful when the application spawns subprocesses or when predictable process supervision is required.

For example, a runtime stage can install `tini` and use:

```dockerfile
ENTRYPOINT ["tini", "--"]
CMD ["node", "src/server.js"]
```

The package installation and verification depend on the chosen base image. Some container platforms can provide an init process themselves, so I avoid running duplicate init layers unnecessarily.

Use Docker's **exec form** for `CMD` and `ENTRYPOINT`; it avoids an unnecessary shell wrapper and helps signals reach the intended process.

### E. Graceful shutdown

When the container receives `SIGTERM`, the application should stop accepting new traffic, allow in-flight requests to finish within a deadline, and close database pools and telemetry exporters.

Example Express shutdown outline:

```javascript
let shuttingDown = false;

process.on("SIGTERM", () => {
  if (shuttingDown) return;
  shuttingDown = true;

  server.close(async (error) => {
    try {
      await pool.end();
      await telemetrySdk.shutdown();
    } catch (shutdownError) {
      logger.error(
        { err: shutdownError },
        "Shutdown failed"
      );
      process.exitCode = 1;
    }

    if (error) {
      logger.error({ err: error }, "HTTP server close failed");
      process.exitCode = 1;
    }
  });
});
```

Here, `server` is the HTTP server returned by `app.listen()`, and `pool` and `telemetrySdk` represent the configured resources. Exact shutdown APIs vary by library.

This outline needs a bounded shutdown deadline and coordination with background jobs, queue consumers, and readiness checks before it is production-complete. The container's termination grace period must be long enough for normal cleanup, but not unlimited.

---

## 7. How do you know the system is production-ready?

Before release, I verify the following:

- Logs are structured, searchable, and free of secrets.
- Request IDs and distributed traces connect relevant operations.
- Metrics include request rate, error rate, latency percentiles, event-loop delay, memory, and database pool pressure.
- Liveness, readiness, and startup checks have clear, different semantics.
- Database pools have explicit limits, timeouts, and graceful shutdown behavior.
- Docker runs as a non-root user and contains only necessary runtime artifacts.
- The service responds correctly to `SIGTERM` and does not lose in-flight work unnecessarily.
- Dashboards and alerts are based on user-impacting symptoms, not just raw CPU or memory thresholds.
- Load tests and failure tests cover database unavailability, slow downstream dependencies, traffic spikes, and container restarts.

**Interview summary:** I make Node.js production-ready by combining structured logs, low-cardinality Prometheus metrics, OpenTelemetry tracing, meaningful health checks, bounded database connection pools, and secure container practices. The key is not merely collecting telemetry; it is using it to detect degradation, identify the failing component, and recover without causing cascading failures.

*Q15. Buffers, encodings and binary data. What is a Buffer (a Uint8Array subclass allocated outside the V8 heap), and how do Buffer.alloc, allocUnsafe and the buffer pool differ? How do you handle UTF-8 multi-byte characters split across stream chunks (StringDecoder), base64/hex conversions, and binary protocols or file headers with readUInt32BE-style methods?*

## 1. What is a Buffer in Node.js?

A `Buffer` is a Node.js API for working with raw binary data. It is a subclass of `Uint8Array` and represents a sequence of bytes, where each byte has a value from `0` to `255`.

JavaScript strings represent text, while buffers represent the underlying bytes used by files, network packets, images, encryption, and binary protocols.

For example, the string `"Hi"` encoded as UTF-8 consists of two bytes:

```javascript
const buffer = Buffer.from("Hi", "utf8");

console.log(buffer);          // <Buffer 48 69>
console.log(buffer.length);   // 2
console.log(buffer[0]);       // 72
console.log(buffer.toString("utf8")); // Hi
```

The byte `0x48` represents `H`, and `0x69` represents `i`.

### Is Buffer memory outside the V8 heap?

Generally, yes. Buffer byte storage is allocated in memory managed outside the ordinary V8 JavaScript object heap, using Node.js's backing-store and allocator mechanisms. However, the `Buffer` object itself is a JavaScript object managed by V8.

This distinction matters because a process can consume substantial memory through buffers even when `heapUsed` is relatively low. I monitor `rss`, `external`, and `arrayBuffers` in addition to `heapUsed`.

---

## 2. `Buffer.alloc()` vs `Buffer.allocUnsafe()`

These methods differ primarily in how the allocated memory is initialized.

### `Buffer.alloc(size)`

Allocates a buffer and initializes every byte to zero.

```javascript
const buffer = Buffer.alloc(4);

console.log(buffer);
// <Buffer 00 00 00 00>
```

This is the safer default when the buffer will be filled with data incrementally or when any unwritten bytes could be exposed.

### `Buffer.allocUnsafe(size)`

Allocates a buffer without initializing every byte first.

```javascript
const buffer = Buffer.allocUnsafe(4);

buffer.writeUInt32BE(100, 0);

console.log(buffer.readUInt32BE(0)); // 100
```

Before every byte is overwritten, its contents are not safe to rely on. They may contain residual data from previously used memory.

**Never expose or transmit an `allocUnsafe()` buffer before completely initializing every byte that could be read.**

For example, this is dangerous:

```javascript
const buffer = Buffer.allocUnsafe(100);

buffer.write("Hello");
res.end(buffer); // Dangerous: most bytes may be uninitialized.
```

The safe version is:

```javascript
res.end(Buffer.from("Hello"));
```

Or initialize the entire buffer and then fill it.

### Comparison

| Feature | `Buffer.alloc()` | `Buffer.allocUnsafe()` |
|---|---|---|
| Initialization | All bytes set to zero | Bytes are not guaranteed to be initialized |
| Performance | May incur zero-filling overhead | Can be faster for carefully controlled use |
| Security | Safer default | Requires complete initialization |
| Typical use | General-purpose buffer creation | Performance-sensitive code that overwrites every byte |

I use `allocUnsafe()` only when profiling justifies it and I can prove every byte is initialized before use.

---

## 3. What is the Buffer pool?

Node.js uses an internal memory pool for certain small buffer allocations to reduce allocation overhead. In current Node.js implementations, `Buffer.poolSize` defaults to 8 KiB, and eligible small allocations can use slices of this shared backing allocation.

Common APIs that may use the pool include:

- `Buffer.allocUnsafe()`
- `Buffer.from(string)`
- `Buffer.from(array)`
- `Buffer.from(arrayBuffer, ...)` does not use the pool in the same way; it shares the supplied backing memory.

The exact pooling behavior depends on the API and allocation size. For example, `Buffer.alloc()` does not use the small-buffer pool in the same way as `Buffer.allocUnsafe()`.

### Why does pooling matter?

It can improve performance by avoiding a separate allocation for every small buffer. But it also affects memory retention: a small buffer slice can keep a larger backing allocation alive until that slice is no longer referenced.

When investigating memory usage, I distinguish between:

- The size of the individual buffer.
- The size of its backing `ArrayBuffer`.
- Whether multiple buffers share that backing allocation.

If a small buffer must outlive a large source buffer, copying it can sometimes reduce retained memory.

```javascript
const smallCopy = Buffer.from(source.subarray(0, 20));
```

This creates a separate copy of the selected bytes.

By contrast:

```javascript
const smallView = source.subarray(0, 20);
```

creates a view over the same underlying memory rather than copying the bytes.

---

## 4. How do you handle UTF-8 characters split across stream chunks?

A stream does not guarantee that each chunk ends at a character boundary.

UTF-8 characters can require one to four bytes. For example, a character such as `€` is encoded as three bytes. If a stream splits those bytes between chunks, independently converting each chunk to a string can corrupt the character.

**Problematic approach:**

```javascript
stream.on("data", (chunk) => {
  console.log(chunk.toString("utf8"));
});
```

If a multi-byte character is split between two chunks, each independent conversion may replace the incomplete sequence with the Unicode replacement character `�`.

### Solution: `StringDecoder`

Node.js provides `StringDecoder` in the `node:string_decoder` module. It retains incomplete trailing sequences and completes them when the next chunk arrives.

```javascript
import { StringDecoder } from "node:string_decoder";

const decoder = new StringDecoder("utf8");

stream.on("data", (chunk) => {
  const text = decoder.write(chunk);
  processText(text);
});

stream.on("end", () => {
  const remaining = decoder.end();

  if (remaining) {
    processText(remaining);
  }
});
```

The important methods are:

- `decoder.write(chunk)`: Decodes the bytes and retains incomplete multi-byte sequences.
- `decoder.end()`: Flushes the decoder's remaining state when the stream ends.

The decoder must be flushed at the end of the stream. Otherwise, any incomplete final sequence may not be handled as intended.

### Alternative: `Readable.setEncoding()`

When I want a readable stream to emit strings instead of buffers, I can use:

```javascript
stream.setEncoding("utf8");

stream.on("data", (text) => {
  processText(text);
});
```

Node.js handles UTF-8 character boundaries for the stream.

**Rule:** Use `StringDecoder` when managing chunk decoding manually, or `setEncoding()` when the entire readable stream should emit decoded text. Keep binary data as buffers rather than decoding it prematurely.

---

## 5. Base64 and hexadecimal conversions

Base64 and hexadecimal are textual representations of binary bytes. They are not encryption; they simply encode binary data into printable characters.

### UTF-8 text to Base64

```javascript
const text = "Hello";
const base64 = Buffer.from(text, "utf8").toString("base64");

console.log(base64); // SGVsbG8=
```

Decode it:

```javascript
const decoded = Buffer.from(base64, "base64").toString("utf8");

console.log(decoded); // Hello
```

### Binary data to hexadecimal

```javascript
const buffer = Buffer.from([0x48, 0x69]);

console.log(buffer.toString("hex")); // 4869
```

Convert hexadecimal back to bytes:

```javascript
const bytes = Buffer.from("4869", "hex");

console.log(bytes); // <Buffer 48 69>
```

### Important distinction

```javascript
Buffer.from("4869", "utf8"); // Bytes for the four text characters
Buffer.from("4869", "hex");  // Bytes 0x48 and 0x69
```

The first creates four bytes representing the characters `"4869"`. The second interprets the string as hexadecimal and produces two bytes.

### Practical considerations

- Base64 increases the encoded size by approximately one-third.
- Hexadecimal requires two characters per byte.
- Encoding and decoding large payloads can consume CPU and memory.
- Node's Base64 decoder accepts some non-canonical input; validate the format explicitly if strict canonical encoding is required by the protocol.
- Never assume Base64 provides confidentiality or integrity.

For large files or HTTP bodies, I prefer streaming instead of reading the entire file into a buffer and encoding it in one operation.

---

## 6. Reading and writing binary protocols

Binary protocols define fields at specific byte offsets. Those fields may represent integers, flags, lengths, timestamps, or payloads.

Node.js provides methods such as:

- `readUInt16BE()` and `readUInt32BE()`
- `readUInt16LE()` and `readUInt32LE()`
- `readInt32BE()` for signed integers
- `readBigUInt64BE()` for 64-bit unsigned integers
- `writeUInt32BE()` for writing an unsigned 32-bit integer

`BE` means big-endian: the most significant byte comes first. `LE` means little-endian: the least significant byte comes first.

### Example: Reading a binary header

Suppose a protocol defines a six-byte header:

| Offset | Length | Field |
|---|---:|---|
| 0 | 4 bytes | Payload length, unsigned big-endian |
| 4 | 2 bytes | Message type, unsigned big-endian |

We can parse it like this:

```javascript
function parseHeader(buffer) {
  if (buffer.length < 6) {
    throw new Error("Incomplete header");
  }

  const payloadLength = buffer.readUInt32BE(0);
  const messageType = buffer.readUInt16BE(4);

  return {
    payloadLength,
    messageType,
  };
}
```

For example:

```javascript
const header = Buffer.from([
  0x00, 0x00, 0x00, 0x05, // Payload length: 5
  0x00, 0x02,             // Message type: 2
]);

console.log(parseHeader(header));
// { payloadLength: 5, messageType: 2 }
```

### Writing the same header

```javascript
function createHeader(payloadLength, messageType) {
  if (
    !Number.isInteger(payloadLength) ||
    payloadLength < 0 ||
    payloadLength > 0xffffffff
  ) {
    throw new RangeError("Invalid payload length");
  }

  if (
    !Number.isInteger(messageType) ||
    messageType < 0 ||
    messageType > 0xffff
  ) {
    throw new RangeError("Invalid message type");
  }

  const header = Buffer.alloc(6);

  header.writeUInt32BE(payloadLength, 0);
  header.writeUInt16BE(messageType, 4);

  return header;
}
```

For binary protocol code, I validate both field ranges and buffer boundaries. An out-of-range read or write can throw a `RangeError`.

---

## 7. Handling binary data arriving in network chunks

One of the most important interview distinctions is that **a TCP stream does not preserve application message boundaries**.

If a protocol has a four-byte length prefix, a single network read may contain:

- Only part of the header.
- The complete header but only part of the payload.
- One complete message.
- Multiple complete messages.

Therefore, I cannot assume one `data` event corresponds to one complete protocol message.

A robust parser accumulates bytes until it has enough data to read the header and then the complete payload. It must also enforce a maximum permitted payload length.

Illustrative parsing logic:

```javascript
const HEADER_SIZE = 4;
const MAX_PAYLOAD_SIZE = 1024 * 1024;

let pending = Buffer.alloc(0);

function onChunk(chunk) {
  pending = Buffer.concat([pending, chunk]);

  while (pending.length >= HEADER_SIZE) {
    const payloadLength = pending.readUInt32BE(0);

    if (payloadLength > MAX_PAYLOAD_SIZE) {
      throw new Error("Payload exceeds permitted size");
    }

    const frameLength = HEADER_SIZE + payloadLength;

    if (pending.length < frameLength) {
      break; // Wait for more bytes.
    }

    const payload = pending.subarray(
      HEADER_SIZE,
      frameLength
    );

    processMessage(payload);

    pending = pending.subarray(frameLength);
  }
}
```

This illustrates the framing logic, but repeated `Buffer.concat()` calls can become expensive for large or highly fragmented inputs because they copy the accumulated data. A production parser should use a bounded buffer strategy, a queue of chunks, or another incremental parser to avoid unnecessary copying.

The parser must also handle connection closure with an incomplete frame, enforce deadlines, and prevent unbounded buffering when the sender advertises an excessive payload length.

---

## 8. Reading file headers and magic numbers

File formats often start with a fixed signature called a *magic number*. This helps identify a file format, but it is not sufficient to prove that the file is valid or safe.

For example, PNG files start with these eight bytes:

```javascript
const PNG_SIGNATURE = Buffer.from([
  0x89, 0x50, 0x4e, 0x47,
  0x0d, 0x0a, 0x1a, 0x0a,
]);

function hasPngSignature(buffer) {
  return (
    buffer.length >= PNG_SIGNATURE.length &&
    buffer.subarray(0, PNG_SIGNATURE.length)
      .equals(PNG_SIGNATURE)
  );
}
```

I can use a signature check as an initial format check, but I also validate the file structure with an appropriate parser and enforce size limits. An attacker can construct a file with a valid signature but malicious or malformed content.

When reading a file header, I verify the minimum buffer length before reading offsets. For untrusted formats, I also validate lengths and offsets before allocating memory or seeking to a position specified by the file.

---

## 9. Common Buffer mistakes to avoid

1. **Treating chunks as complete messages:** Network and file streams can split data at arbitrary byte boundaries.
2. **Decoding each UTF-8 chunk independently:** Multi-byte characters can be corrupted; use `StringDecoder` or `setEncoding()`.
3. **Exposing uninitialized memory:** Fully initialize every byte in an `allocUnsafe()` buffer before reading or transmitting it.
4. **Ignoring endianness:** `readUInt32BE()` and `readUInt32LE()` interpret the same bytes differently.
5. **Trusting untrusted length fields:** Enforce payload limits before buffering or allocating memory.
6. **Confusing a view with a copy:** `subarray()` shares memory; `Buffer.from(buffer)` creates a copy.
7. **Using Base64 as encryption:** It is only an encoding.
8. **Ignoring external memory:** Monitor `external`, `arrayBuffers`, and RSS, not just V8 heap usage.

**Interview summary:** A Node.js `Buffer` represents raw bytes and is backed by memory outside the ordinary V8 heap. `Buffer.alloc()` zero-initializes memory, while `allocUnsafe()` trades initialization work for performance and requires careful handling. I use `StringDecoder` for chunk-safe text decoding, explicit Base64 or hexadecimal encodings when required, and endian-aware read/write methods for binary protocols. For untrusted streams and file formats, I validate boundaries, enforce size limits, and never assume that one chunk equals one complete message.

*Q16. EventEmitter internals and pitfalls. How do on, once, off and emit work, and what happens when a listener throws or an 'error' event has no handler? What does the MaxListenersExceededWarning signal, how do listener leaks happen, and how do you build a typed, async-safe event bus (events.once, AbortSignal support)?*

## 1. What is EventEmitter?

`EventEmitter` is a core Node.js class from the `node:events` module that implements the observer pattern. Objects emit named events, and registered listeners execute when those events are emitted.

It is used in streams, HTTP servers, database clients, and application-level event-driven systems.

```javascript
import { EventEmitter } from "node:events";

const emitter = new EventEmitter();

emitter.on("userCreated", (user) => {
  console.log("User created:", user.id);
});

emitter.emit("userCreated", { id: 101 });
```

The important point is that **standard EventEmitter listeners execute synchronously, in registration order**. EventEmitter does not automatically make listeners asynchronous or wait for promises returned by them.

---

## 2. How do `on()`, `once()`, `off()`, and `emit()` work?

### `on(eventName, listener)`

Registers a listener that remains registered until it is removed.

```javascript
function handleOrder(order) {
  console.log("Order:", order.id);
}

emitter.on("orderCreated", handleOrder);
```

Calling `on()` multiple times with the same function registers multiple listener entries. EventEmitter does not automatically deduplicate listeners.

### `once(eventName, listener)`

Registers a listener that is automatically removed after its first invocation.

```javascript
emitter.once("connected", () => {
  console.log("Connected for the first time");
});
```

This is useful for one-time initialization or waiting for a single event. The listener is removed before it is invoked, so a recursive emission does not cause that same `once` listener to execute again.

### `off(eventName, listener)`

Removes a specific listener registered for an event. It is an alias for `removeListener()`.

```javascript
function handleMessage(message) {
  console.log(message);
}

emitter.on("message", handleMessage);
emitter.off("message", handleMessage);
```

The function reference must match the registered listener. An inline anonymous function cannot easily be removed unless its reference was saved.

For a `once()` listener, Node internally wraps the original listener. Removing it by the original listener reference is supported.

### `emit(eventName, ...args)`

Synchronously invokes the listeners registered for that event, passing the supplied arguments to each listener.

```javascript
emitter.on("paymentReceived", (paymentId, amount) => {
  console.log(paymentId, amount);
});

const hadListeners = emitter.emit(
  "paymentReceived",
  "pay_123",
  500
);

console.log(hadListeners); // true
```

`emit()` returns `true` if the event had listeners and `false` otherwise. The return value does not indicate whether the listeners succeeded.

### Summary

| Method | Purpose | Behavior |
|---|---|---|
| `on()` | Persistent listener | Runs on every emission |
| `once()` | One-time listener | Automatically removed after invocation |
| `off()` | Remove listener | Removes the matching listener |
| `emit()` | Dispatch event | Invokes listeners synchronously |

---

## 3. What happens if a listener throws an error?

Because EventEmitter invokes ordinary listeners synchronously, a synchronous exception propagates back to the caller of `emit()`.

```javascript
emitter.on("task", () => {
  throw new Error("Task failed");
});

emitter.on("task", () => {
  console.log("This may not execute");
});

try {
  emitter.emit("task");
} catch (error) {
  console.error("Caught:", error.message);
}
```

The first listener throws, so `emit()` stops dispatching and the second listener is not called.

The exception is not automatically converted into an `'error'` event. It propagates through the normal JavaScript call stack unless caught.

This has an important operational consequence: a listener exception can unexpectedly interrupt the code that emitted the event.

### What about asynchronous listeners?

Consider:

```javascript
emitter.on("task", async () => {
  throw new Error("Async failure");
});

emitter.emit("task");
```

An `async` function returns a Promise. Throwing inside it rejects that Promise rather than throwing synchronously through the caller. Standard `emit()` ignores listener return values, so it does not await that Promise or automatically catch its rejection.

This can produce an unhandled Promise rejection.

For asynchronous event processing, I explicitly handle failures:

```javascript
emitter.on("task", async () => {
  try {
    await processTask();
  } catch (error) {
    logger.error({ err: error }, "Task processing failed");
  }
});
```

For a shared event bus, I may instead use a dedicated asynchronous dispatch mechanism that awaits listeners and defines a clear failure policy.

Node's `captureRejections` option can route rejected listener Promises to an appropriate error handler, but it does not make `emit()` await the listeners or guarantee sequential processing.

---

## 4. What happens when an `'error'` event has no listener?

The `'error'` event has special behavior.

```javascript
emitter.emit("error", new Error("Database unavailable"));
```

If the emitter has no registered `'error'` listener, Node.js throws the supplied error. If it remains uncaught, it can terminate the process.

This differs from ordinary events, for which emitting without listeners simply returns `false`.

A basic error handler:

```javascript
emitter.on("error", (error) => {
  logger.error({ err: error }, "Emitter error");
});
```

However, an `'error'` listener does not automatically catch every exception thrown by other listeners. It handles explicitly emitted `'error'` events and certain errors routed through Node's rejection-capture mechanism.

**Production rule:** Define how errors are handled at the component boundary. Do not emit `'error'` without knowing whether a handler exists, and do not assume that adding an error listener makes every listener failure safe.

---

## 5. What does `MaxListenersExceededWarning` mean?

By default, EventEmitter warns when more than 10 listeners are registered for a single event on one emitter.

```javascript
for (let i = 0; i < 11; i++) {
  emitter.on("update", () => {});
}
```

This may produce a `MaxListenersExceededWarning`.

The limit is a **diagnostic threshold, not a hard maximum**. Node.js does not automatically stop registering listeners after the threshold is reached.

The warning often indicates that listeners are accumulating unexpectedly, although more than 10 listeners may be legitimate in some designs.

### How do listener leaks happen?

Common causes include:

1. Registering listeners repeatedly inside a request handler without removing them.
2. Creating new anonymous callbacks on every operation.
3. Subscribing to global emitters or long-lived objects without an unsubscribe mechanism.
4. Forgetting to remove listeners when a component, connection, or job finishes.
5. Registering listeners on retries without cleaning up the previous attempt.
6. Using `once()` for an event that never occurs, leaving the listener registered indefinitely.

Example of a leak:

```javascript
app.get("/status", (req, res) => {
  emitter.on("statusChanged", (status) => {
    console.log(status);
  });

  res.json({ ok: true });
});
```

Every request adds another listener. The closures can retain references to request-related objects and other state.

A better design registers a long-lived listener once, or uses a subscription whose lifetime is explicitly tied to the operation.

### How do you investigate the warning?

```javascript
process.on("warning", (warning) => {
  if (warning.name === "MaxListenersExceededWarning") {
    logger.warn({
      message: warning.message,
      stack: warning.stack,
      emitterType: warning.emitter?.constructor?.name,
      eventType: warning.type,
      listenerCount: warning.count,
    }, "Possible listener leak");
  }
});
```

The warning includes useful diagnostic properties such as the emitter, event type, and listener count.

I inspect listener registration and cleanup paths, use `emitter.listenerCount(eventName)` where appropriate, and profile retained memory if necessary.

Increasing the limit with `setMaxListeners()` can be valid when the number of listeners is intentional, but **raising the limit should not be the default response to a warning**. It can conceal a genuine leak.

---

## 6. How do you wait for events safely with `events.once()`?

Node provides a Promise-based `once()` helper in `node:events`. It is different from the instance method `emitter.once()`.

```javascript
import { once } from "node:events";

const [message] = await once(emitter, "message");

console.log(message);
```

The Promise resolves with an array containing the emitted arguments.

For example:

```javascript
emitter.emit("message", "hello", 42);

// The resolved value is ["hello", 42].
```

The helper also rejects if the emitter emits `'error'` before the awaited event occurs, except when `'error'` itself is the event being awaited.

### Important race condition

Register event listeners before starting an operation that might emit the event immediately.

```javascript
const connected = once(socket, "connect");

socket.connect();

await connected;
```

If the operation can emit an event before the listener is registered, the event may be missed. Creating the Promise first avoids that particular race.

For multiple events, Node's Promise-based `once()` has an important nuance: events emitted synchronously in the same `process.nextTick()` batch can be missed if the second `once()` Promise is not created until after the first is awaited. Create all relevant waiting Promises before awaiting them.

---

## 7. How do you support `AbortSignal`?

An `AbortSignal` allows a listener or asynchronous operation to be cancelled when its owner no longer needs it.

### EventEmitter with `signal`

Node's `emitter.on()` supports an options object with a `signal` property:

```javascript
const controller = new AbortController();

emitter.on(
  "data",
  handleData,
  { signal: controller.signal }
);

// Later, when the subscription is no longer needed:
controller.abort();
```

Aborting the controller automatically removes the listener.

This is particularly useful when a subscription belongs to a request, component, or job with a defined lifetime.

### Promise-based `once()` with cancellation

```javascript
import { once } from "node:events";

const controller = new AbortController();

try {
  const [message] = await once(
    emitter,
    "message",
    { signal: controller.signal }
  );

  processMessage(message);
} catch (error) {
  if (error.name === "AbortError") {
    logger.info("Waiting for message was cancelled");
  } else {
    throw error;
  }
}
```

If the controller is aborted before the awaited event arrives, the Promise rejects with an abort error.

The same cancellation signal can be passed to other APIs that support it, enabling coordinated cancellation of listeners, timers, and network operations.

---

## 8. How do you build a typed, async-safe event bus?

For a small Node.js application, I can extend `EventEmitter`. In TypeScript, I define an event map to constrain event names and payload types.

### A. Define typed events

```typescript
import { EventEmitter } from "node:events";

type AppEvents = {
  userCreated: [{ userId: string; email: string }];
  orderPlaced: [{ orderId: string; total: number }];
  paymentFailed: [{ orderId: string; reason: string }];
};

class TypedEventBus extends EventEmitter {
  emit<K extends keyof AppEvents>(
    event: K,
    ...args: AppEvents[K]
  ): boolean {
    return super.emit(event, ...args);
  }

  on<K extends keyof AppEvents>(
    event: K,
    listener: (...args: AppEvents[K]) => void
  ): this {
    return super.on(event, listener);
  }

  off<K extends keyof AppEvents>(
    event: K,
    listener: (...args: AppEvents[K]) => void
  ): this {
    return super.off(event, listener);
  }
}

const bus = new TypedEventBus();

bus.on("userCreated", ({ userId, email }) => {
  console.log(userId, email);
});

bus.emit("userCreated", {
  userId: "u123",
  email: "dev@example.com",
});

// TypeScript catches incorrect event names and payload shapes.
```

This provides compile-time checks for the events covered by these methods. A complete abstraction should also type `once()`, and potentially other listener APIs consistently.

TypeScript types disappear at runtime, so external messages must still be validated.

### B. Async-safe event dispatch

A typed `EventEmitter` does not make asynchronous handlers safe by itself. If listeners must finish before dispatch completes, I define an explicit asynchronous API.

For example, a simpler asynchronous bus can use typed listener registries and await listeners:

```typescript
type AsyncHandler<T> = (payload: T) => void | Promise<void>;

class AsyncEventBus<Events extends object> {
  private listeners = new Map<
    keyof Events,
    Set<AsyncHandler<unknown>>
  >();

  on<K extends keyof Events>(
    event: K,
    handler: AsyncHandler<Events[K]>
  ): () => void {
    let handlers = this.listeners.get(event);

    if (!handlers) {
      handlers = new Set();
      this.listeners.set(event, handlers);
    }

    handlers.add(handler as AsyncHandler<unknown>);

    return () => {
      handlers?.delete(handler as AsyncHandler<unknown>);
    };
  }

  async emit<K extends keyof Events>(
    event: K,
    payload: Events[K]
  ): Promise<void> {
    const handlers = [...(this.listeners.get(event) ?? [])];

    await Promise.all(
      handlers.map((handler) =>
        Promise.resolve().then(() =>
          handler(payload)
        )
      )
    );
  }
}
```

Usage:

```typescript
type Events = {
  orderPlaced: { orderId: string; total: number };
};

const bus = new AsyncEventBus<Events>();

bus.on("orderPlaced", async (order) => {
  await sendConfirmationEmail(order.orderId);
});

await bus.emit("orderPlaced", {
  orderId: "ord_123",
  total: 499,
});
```

This implementation demonstrates typed asynchronous dispatch, but it has deliberate limitations:

- `Promise.all()` runs listeners concurrently and rejects if a listener rejects; other listeners may continue running.
- It does not support `once()`, `AbortSignal`, listener limits, or a defined error-aggregation policy.
- It is in-memory and provides no durable delivery, retry guarantees, or cross-process messaging.
- It assumes a simple payload argument rather than arbitrary event argument lists.

For production, I define the required delivery semantics before expanding the abstraction: sequential versus concurrent handlers, whether failures are aggregated or isolated, cancellation, unsubscribe behavior, and whether events need retries or durable delivery.

### C. When should you use a message broker instead?

An in-process event bus is appropriate for decoupling components within one process. It is not a distributed messaging system.

If an order must reliably trigger invoicing, inventory updates, and notifications across multiple services, I consider a broker such as RabbitMQ, Kafka, or a managed queue. For critical business events, I may use a transactional outbox so the database update and the intent to publish the event are recorded atomically.

A local `emit()` call does not guarantee that listeners finish successfully, that events survive a process crash, or that another application instance receives the event.

---

## 9. Common EventEmitter pitfalls

1. **Assuming `emit()` is asynchronous:** Standard listeners execute synchronously.
2. **Ignoring rejected listener Promises:** `emit()` does not await or catch them automatically.
3. **Emitting an unhandled `'error'` event:** This can throw and terminate the process if uncaught.
4. **Ignoring listener cleanup:** Accumulating subscriptions can retain memory and duplicate work.
5. **Increasing listener limits without investigation:** This can hide leaks.
6. **Assuming `once()` prevents every race:** Register listeners before initiating operations that may emit.
7. **Assuming a local event bus is durable:** Process restarts can lose in-memory work.
8. **Assuming TypeScript guarantees runtime validity:** External event payloads still need validation.

**Interview summary:** EventEmitter provides synchronous, ordered event dispatch with persistent and one-time listeners. Its biggest pitfalls are synchronous exception propagation, unhandled asynchronous listener rejections, special `'error'` behavior, and listener leaks. I use `events.once()` and `AbortSignal` for lifecycle-aware subscriptions, typed event maps for compile-time safety, and an explicit async event-bus abstraction when handlers must be awaited. For durable cross-service workflows, I use a message broker rather than relying on in-process events.

*Q17. HTTP server and client behaviour. How do Node’s http server, keep-alive, server.keepAliveTimeout/headersTimeout and agent socket pooling (http.Agent, maxSockets) interact with load balancers like ALB or Nginx, and why does a mismatch cause random 502s or ECONNRESET? How do timeouts, retries and AbortController work with fetch/undici?*

## 1. How does Node.js HTTP handling work?

Node.js provides the `node:http` module for building HTTP servers and clients. Modern Node.js also provides a global `fetch()` API backed by Undici for outbound HTTP requests.

An HTTP server accepts connections, parses requests, and emits request/response objects. A single TCP connection can carry multiple HTTP/1.1 requests sequentially when keep-alive is enabled, avoiding the overhead of establishing a new TCP connection for every request.

The important distinction is that **HTTP keep-alive, connection pooling, request timeouts, and load-balancer timeouts control different things**.

- Keep-alive allows a connection to be reused.
- Connection pooling manages how many connections are created and reused.
- Timeouts determine how long a particular operation or connection can remain idle or incomplete.
- Load balancers introduce their own connection lifecycle and timeout policies.

When these settings conflict, a connection may be closed by one component while another still considers it reusable. That is a common cause of intermittent `502` responses and `ECONNRESET` errors.

---

## 2. How do keep-alive and server timeouts work?

Consider a typical request path:

<box border radius="lg" padding={3} gap={2}>
  <box background="surface-secondary" radius="md" padding={3} align="center">
    **Client / Browser**
  </box>
  <row justify="center">
    <icon name="arrow-down-up" size="lg" />
  </row>
  <box background="rgba(59,130,246,0.10)" radius="md" padding={3} align="center">
    **ALB or Nginx**
    <text color="secondary" size="sm">Frontend and backend connection pools</text>
  </box>
  <row justify="center">
    <icon name="arrow-down-up" size="lg" />
  </row>
  <box background="rgba(20,184,166,0.10)" radius="md" padding={3} align="center">
    **Node.js HTTP server**
    <text color="secondary" size="sm">Request handling, keep-alive and timeouts</text>
  </box>
  <row justify="center">
    <icon name="arrow-down-up" size="lg" />
  </row>
  <box background="surface-secondary" radius="md" padding={3} align="center">
    **Database / Downstream services**
  </box>
</box>

Each hop may have its own TCP connection and timeout configuration. A timeout on one hop does not automatically configure the others.

### `server.keepAliveTimeout`

This controls how long Node.js waits for additional requests on an HTTP connection after it has finished writing a response, before closing that connection.

```javascript
import http from "node:http";

const server = http.createServer((req, res) => {
  res.writeHead(200, {
    "Content-Type": "application/json",
  });

  res.end(JSON.stringify({ ok: true }));
});

server.keepAliveTimeout = 5_000;
```

The default is typically 5 seconds in modern Node.js releases, but I verify the behavior for the Node version actually deployed.

If Nginx or another proxy tries to reuse a backend connection after Node.js has closed it, the proxy may encounter a reset or closed connection and return a `502`.

### `server.headersTimeout`

This limits how long Node.js allows a client to transmit the complete HTTP request headers.

```javascript
server.headersTimeout = 60_000;
```

The default in modern Node.js versions is generally 60 seconds. The exact interaction with `requestTimeout` and other settings depends on the Node version.

This timeout helps protect the server against clients that send headers very slowly. It is not a general application execution timeout.

### `server.requestTimeout`

This limits how long Node.js allows the client to send the entire incoming request. It is particularly relevant for large or slow uploads.

```javascript
server.requestTimeout = 120_000;
```

It does not mean that the application handler must finish processing the request within that duration.

### Other important timeout settings

- `server.timeout`: Socket inactivity timeout; its default is `0` in current Node.js releases, meaning no socket timeout is configured through this property.
- `server.headersTimeout`: Incoming header completion deadline.
- `server.requestTimeout`: Incoming request receipt deadline.
- `server.keepAliveTimeout`: Idle period after a response while waiting for another request.
- `server.keepAliveTimeoutBuffer`: An additional internal buffer used in supported recent Node.js versions to reduce premature keep-alive socket closure; verify availability in the deployed version.

These properties are not interchangeable. A slow upload, a slow application handler, and an idle persistent connection are different conditions and need different controls.

---

## 3. How does `http.Agent` and socket pooling work?

For outbound HTTP/1.1 requests made using Node's `http` and `https` modules, an `Agent` manages socket reuse and pooling.

```javascript
import http from "node:http";

const agent = new http.Agent({
  keepAlive: true,
  maxSockets: 50,
  maxFreeSockets: 10,
  timeout: 30_000,
});
```

The key options are:

| Option | Purpose |
|---|---|
| `keepAlive` | Reuse eligible sockets instead of closing them after each request |
| `maxSockets` | Maximum active sockets per origin |
| `maxFreeSockets` | Maximum idle sockets retained per origin |
| `timeout` | Socket timeout configuration; not a universal request deadline |

The exact lifecycle behavior depends on the Node version and agent implementation. In particular, do not treat `timeout` as a replacement for explicit request or application deadlines.

### What happens when `maxSockets` is reached?

Suppose an application sends 500 concurrent requests to the same upstream service but allows only 50 active sockets for that origin.

The remaining requests generally wait in the agent's pending queue until a socket becomes available.

This can cause:

- Increased request latency.
- Growing queues and memory use.
- Request deadlines expiring before a socket is acquired.
- A backlog that worsens during downstream outages.

Increasing `maxSockets` is not always the right fix. It may overload the upstream service. I tune pool limits using expected concurrency, upstream capacity, latency, and measured queueing.

Also, `http.Agent` settings do not configure the connection pool used by global `fetch()`, which is backed by Undici.

---

## 4. How do Node.js, ALB and Nginx timeout mismatches cause random 502s?

A `502 Bad Gateway` means a gateway or proxy could not obtain a valid response from its upstream. It does not identify the root cause by itself.

Common causes include:

1. The proxy reuses a backend connection that Node.js has already closed.
2. Node.js closes an idle socket while the proxy still has it in its keep-alive pool.
3. The upstream process crashes or is restarted while a request is in flight.
4. A connection is reset during request or response transmission.
5. The application or upstream closes the connection before completing a valid HTTP response.
6. The proxy cannot connect to the upstream or receives an invalid response.

### Example: keep-alive mismatch

Suppose:

- Node.js closes an idle backend connection after 5 seconds.
- Nginx keeps the corresponding upstream connection in its pool for longer.
- Nginx selects that stale connection for a new request.

Nginx may send the request on a socket that has already been closed by Node.js. Depending on timing, this may produce a connection reset or a `502`.

The problem appears intermittent because it depends on timing: some requests reuse a healthy socket, while others encounter a connection that has just become invalid.

### How do I configure the timeouts?

For a conventional HTTP/1.1 proxy-to-Node setup, a useful starting principle is:

**The proxy's backend idle keep-alive timeout should be shorter than the Node.js server's keep-alive timeout**, leaving enough margin for scheduling and network delays.

For example:

| Setting | Illustrative value |
|---|---:|
| Nginx upstream keep-alive idle timeout | 4 seconds |
| Node.js `server.keepAliveTimeout` | 5 seconds |
| Node.js `server.keepAliveTimeoutBuffer` | 1 second, if supported |

These are example values, not universal production defaults. Nginx's upstream connection reuse and idle timeout directives depend on its version and configuration. AWS ALB has its own idle timeout behavior, which must be checked separately.

I also ensure that a proxy's response timeout and Node.js's application deadline are compatible. A proxy timing out before the application finishes can terminate the client-facing request even if the Node process continues working.

For long-running requests, I explicitly decide whether to increase the timeout, return an asynchronous job ID, or stream the response. Simply increasing every timeout can tie up connections and resources.

### ALB-specific consideration

An AWS Application Load Balancer has an idle timeout for connections that remain inactive. If the application, proxy, and ALB use incompatible idle policies, requests may encounter stale connections or unexpected closure.

I verify the actual ALB idle timeout, backend protocol, target health, and relevant access logs instead of assuming that Node.js's keep-alive settings alone explain the problem.

### Nginx-specific consideration

For Nginx, I inspect:

- Upstream keep-alive pool configuration.
- `proxy_http_version` and connection headers.
- `keepalive_timeout` for client-facing connections.
- Upstream connection reuse settings supported by the deployed Nginx version.
- `proxy_connect_timeout`, `proxy_send_timeout`, and `proxy_read_timeout`.
- Error logs for upstream resets, premature closures, and timeout messages.

`proxy_read_timeout` generally measures the interval between successive reads from the upstream, not necessarily a total wall-clock deadline for the complete response.

---

## 5. How do `fetch()` and Undici handle timeouts?

Node's global `fetch()` is implemented using Undici. It supports standard Fetch API behavior, including `AbortSignal`, but it does not expose every `http.Agent` configuration option.

A common mistake is assuming `fetch()` has a universal built-in timeout for the entire request. I set an explicit deadline when the application requires one.

### Use `AbortController`

```javascript
async function fetchWithTimeout(url, timeoutMs = 5_000) {
  const controller = new AbortController();

  const timer = setTimeout(() => {
    controller.abort(
      new Error("Upstream request timed out")
    );
  }, timeoutMs);

  try {
    const response = await fetch(url, {
      signal: controller.signal,
    });

    if (!response.ok) {
      throw new Error(
        `Upstream returned HTTP ${response.status}`
      );
    }

    return await response.json();
  } finally {
    clearTimeout(timer);
  }
}
```

This applies a deadline signal to the fetch operation. For a response with a body, I also ensure body consumption completes within the intended deadline, because receiving the response headers does not mean the body has been fully downloaded.

Modern Node.js supports `AbortSignal.timeout()` as a concise alternative:

```javascript
const response = await fetch(url, {
  signal: AbortSignal.timeout(5_000),
});
```

If I need to cancel because either a parent operation is cancelled or a timeout expires, I combine signals using a supported API such as `AbortSignal.any()`.

### What does aborting actually do?

Aborting signals that the operation should stop. The underlying client generally cancels the request or response-body consumption and releases associated resources as the implementation permits.

It does not guarantee that a remote server has stopped processing the request. If the server already received a payment or created an order, aborting the client connection cannot undo that side effect.

Therefore, I use idempotency keys and server-side deduplication for operations that must not execute twice.

### Undici-specific timeouts

Undici provides more granular timeout controls through its dispatcher and lower-level APIs, including connection, header, and body-related timeout settings depending on the API and version.

I distinguish these from an application deadline:

- **Connection timeout:** Time allowed to establish a connection.
- **Headers timeout:** Time allowed to receive response headers.
- **Body timeout:** Maximum permitted inactivity while receiving the body, depending on the API.
- **Application deadline:** Maximum total time the application is willing to spend on the operation.

A per-phase timeout does not necessarily limit the total duration. I use an overall abort deadline and configure phase-specific limits where necessary.

---

## 6. How does retry behavior work?

Retries can improve resilience to transient failures, but indiscriminate retries can amplify an outage.

I classify errors before retrying.

| Failure | Typical policy |
|---|---|
| Temporary connection reset | Retry when safe |
| Temporary DNS/network failure | Retry when safe |
| HTTP `429` | Respect `Retry-After` where applicable |
| HTTP `502`, `503`, `504` | Often retry within a bounded policy |
| HTTP `400`, `401`, `403` | Usually do not retry unchanged |
| Validation error | Do not retry |
| Request timeout | Retry only if the operation is safe or idempotent |

### Exponential backoff with jitter

A typical policy uses exponential backoff:

\[
d_n = \min(d_{\max}, d_0 \cdot 2^n)
\]

where \(d_0\) is the initial delay and \(d_{\max}\) is the maximum delay. I add random jitter to prevent many clients from retrying simultaneously.

Example:

```javascript
function sleep(ms, signal) {
  return new Promise((resolve, reject) => {
    if (signal?.aborted) {
      reject(signal.reason);
      return;
    }

    const timer = setTimeout(() => {
      cleanup();
      resolve();
    }, ms);

    function onAbort() {
      clearTimeout(timer);
      cleanup();
      reject(signal.reason);
    }

    function cleanup() {
      signal?.removeEventListener("abort", onAbort);
    }

    signal?.addEventListener("abort", onAbort, {
      once: true,
    });
  });
}
```

This helper supports cancellation during the delay. A production retry implementation also needs an overall deadline, a retry budget, and a policy for deciding which errors and HTTP statuses are retryable.

### Important retry rules

1. Set a maximum number of attempts.
2. Use backoff with jitter.
3. Respect the caller's deadline and cancellation signal.
4. Retry only transient failures.
5. Do not blindly retry non-idempotent operations such as payments or order creation.
6. Reuse an idempotency key when the API supports it.
7. Avoid retries at multiple layers without coordination, because retries can multiply dramatically.

For example, three retries in the client and three retries in the service layer can produce up to 16 attempts when both layers retry each attempt four times.

A retry should be an explicit resilience policy, not a generic catch-and-repeat loop.

---

## 7. How do you investigate intermittent `502` and `ECONNRESET` errors?

I correlate proxy logs, Node.js logs, request IDs, and metrics around the exact failure time.

My investigation order is:

1. **Identify the failing hop.** Determine whether the reset occurred between client and proxy, proxy and Node.js, or Node.js and an upstream service.
2. **Check proxy error logs.** Look for messages indicating an upstream reset, premature closure, connection timeout, or invalid response.
3. **Compare timeout configuration.** Inspect Node.js keep-alive settings, proxy idle settings, ALB idle timeout, and request deadlines.
4. **Inspect connection reuse.** Check whether failures disproportionately occur on reused idle sockets.
5. **Check resource saturation.** Review event-loop lag, CPU, memory, active sockets, agent queues, and database pool pressure.
6. **Check application shutdowns and crashes.** A restart or deployment can terminate connections in flight.
7. **Review retries.** Confirm that retries are bounded and not amplifying the incident.

I do not assume every `ECONNRESET` is a keep-alive mismatch. A client abort, peer restart, network intermediary, or upstream failure can produce the same error.

For outbound HTTP calls, I record the destination service, duration, attempt number, error code, and trace ID without logging credentials or sensitive payloads.

---

## 8. Production checklist

- Configure server header, request, and keep-alive timeouts according to the workload.
- Configure proxy idle and response timeouts with the correct semantics for each hop.
- Size `http.Agent` or Undici pools based on upstream capacity and concurrency.
- Use explicit request deadlines and cancellation signals.
- Apply bounded retries with exponential backoff and jitter.
- Protect non-idempotent operations with idempotency keys or equivalent deduplication.
- Track HTTP status rates, latency percentiles, connection resets, timeout counts, and pool saturation.
- Verify behavior under load, idle connection reuse, slow responses, deployments, and upstream failures.

**Interview summary:** Node.js HTTP reliability depends on coordinating connection reuse, timeout semantics, and resource limits across every hop. A keep-alive mismatch can cause stale socket reuse and intermittent `502` or `ECONNRESET` errors. For outbound requests, I use Undici-backed `fetch()` with explicit `AbortSignal` deadlines, appropriate connection and body limits, and bounded retries that respect idempotency. I diagnose failures by identifying the failing connection hop rather than assuming every reset has the same cause.

*Q18. Async patterns and concurrency control. How do you limit concurrency (p-limit, a queue, async iterators with batching) when calling external APIs or processing thousands of records? Compare Promise.all, allSettled, for await...of and streams for batch workloads, and explain how AsyncLocalStorage carries request context (request IDs, user, tenant) across async calls.*

## 1. Why do we need concurrency control?

When processing thousands of records or calling external APIs, starting every operation simultaneously can overwhelm:

- External APIs and trigger rate limits (`429 Too Many Requests`).
- Database connection pools.
- Memory, because thousands of pending promises retain data.
- CPU and the event loop, especially if callbacks perform expensive synchronous work.

**Concurrency** is the number of operations running at the same time. **Throughput** is the number of operations completed per unit of time. **Rate limiting** controls how frequently operations start.

For example, processing 10,000 records with unlimited concurrency might create 10,000 simultaneous API requests. A concurrency limit of 10 allows at most 10 active requests, while a rate limiter can additionally restrict requests to, say, 50 per second.

## 2. `Promise.all` vs `Promise.allSettled` vs `for await...of` vs streams

| Approach | Execution behavior | Error behavior | Best use |
|---|---|---|---|
| `Promise.all()` | Runs supplied promises concurrently | Rejects on the first rejection; other operations continue | Small, bounded sets of independent tasks |
| `Promise.allSettled()` | Runs supplied promises concurrently | Waits for all and returns every outcome | Independent tasks where partial failures are acceptable |
| `for await...of` | Consumes an async iterable sequentially by default | Throws when iteration rejects | Sequential processing or consuming async results |
| Streams | Processes data incrementally, supporting backpressure | Errors propagate through the stream pipeline | Large files, network data and memory-efficient pipelines |

### `Promise.all()`

```js
const results = await Promise.all(
  ids.map(id => fetchRecord(id))
);
```

All operations are started when `map()` invokes `fetchRecord`. `Promise.all()` does not limit concurrency.

If one promise rejects, the aggregate promise rejects immediately. The remaining requests are not automatically cancelled.

Use it when the task count is small or already bounded.

### `Promise.allSettled()`

```js
const outcomes = await Promise.allSettled(
  ids.map(id => fetchRecord(id))
);

const successful = outcomes
  .filter(result => result.status === "fulfilled")
  .map(result => result.value);

const failed = outcomes
  .filter(result => result.status === "rejected");
```

This is useful for bulk imports, notifications and independent record updates where one failure should not discard the results of other operations.

However, **`allSettled()` does not limit concurrency**. Passing 10,000 tasks still starts all 10,000 if the mapping function starts each operation immediately.

### `for await...of`

```js
for await (const record of records) {
  await processRecord(record);
}
```

This consumes an async iterable one item at a time, assuming `processRecord()` is awaited inside the loop.

Advantages:
- Simple sequential processing.
- Natural integration with async generators and streams.
- Avoids creating thousands of simultaneous promises.

Disadvantage: processing is sequential, so it may underutilize available I/O capacity.

Important distinction: `for await...of` does not inherently guarantee that the source prefetches only one item. The iterable itself determines its buffering behavior.

### Streams

```js
import { createReadStream } from "node:fs";
import { createInterface } from "node:readline";

const input = createReadStream("records.csv");

const lines = createInterface({
  input,
  crlfDelay: Infinity
});

for await (const line of lines) {
  await processLine(line);
}
```

This reads and processes the file incrementally instead of loading the entire file into memory.

Node.js streams support **backpressure**: when a downstream consumer cannot keep up, a properly implemented stream pipeline slows the upstream producer.

This example processes lines sequentially. If processing each line involves slow network I/O, a bounded concurrent worker queue may improve throughput.

---

## 3. Limit concurrency with a worker pool

For thousands of records, a fixed number of workers is a practical solution. It avoids creating one promise per record all at once.

```js
async function mapWithConcurrency(items, limit, mapper) {
  if (!Number.isInteger(limit) || limit < 1) {
    throw new RangeError("limit must be a positive integer");
  }

  const results = new Array(items.length);
  let nextIndex = 0;

  async function worker() {
    while (true) {
      const index = nextIndex++;

      if (index >= items.length) return;

      results[index] = await mapper(items[index], index);
    }
  }

  const workerCount = Math.min(limit, items.length);

  await Promise.all(
    Array.from({ length: workerCount }, () => worker())
  );

  return results;
}

// At most 10 calls are active at once.
const results = await mapWithConcurrency(
  records,
  10,
  record => fetchRecord(record.id)
);
```

### How it works

1. Create at most `limit` workers.
2. Each worker claims the next index.
3. It awaits the current task before claiming another.
4. `Promise.all()` waits for the workers to finish.
5. Results are stored by index, preserving input order.

This simple implementation stops the overall operation if a mapper rejects, but other workers may already be processing tasks and can continue. Production implementations should explicitly define cancellation, error collection and shutdown behavior.

It also assumes `items` is already in memory. For an unbounded or very large input source, use a bounded queue or async iterable instead.

## 4. Use `p-limit` for a simpler implementation

The `p-limit` package provides a concurrency limiter without requiring a custom worker pool.

```js
import pLimit from "p-limit";

const limit = pLimit(10);

const results = await Promise.all(
  records.map(record =>
    limit(() => fetchRecord(record.id))
  )
);
```

Here, no more than 10 limited functions execute simultaneously.

**Important:** `records.map()` still creates a promise for every record. For a modest list, this is fine. For millions of records, use a streaming input source and a bounded queue to control both active work and queued memory.

If you want to continue after individual failures:

```js
const outcomes = await Promise.allSettled(
  records.map(record =>
    limit(() => fetchRecord(record.id))
  )
);
```

This combines bounded concurrency with per-record outcome tracking.

Use a concurrency limiter when the primary requirement is a maximum number of simultaneous operations. Use a queue when you also need queue capacity, prioritization, retries, cancellation, rate limiting or graceful shutdown.

---

## 5. Async iterators with batching

Batching groups records into chunks, while concurrency determines how many operations execute simultaneously. These are separate controls.

```js
async function* batches(iterable, batchSize) {
  let batch = [];

  for await (const item of iterable) {
    batch.push(item);

    if (batch.length === batchSize) {
      yield batch;
      batch = [];
    }
  }

  if (batch.length > 0) {
    yield batch;
  }
}

for await (const batch of batches(records, 100)) {
  await saveBatch(batch);
}
```

This processes records in batches of 100, sequentially. It is useful for database bulk inserts and APIs that accept batch payloads.

To process a limited number of batches concurrently:

```js
import pLimit from "p-limit";

const limit = pLimit(3);

const pending = [];

for await (const batch of batches(records, 100)) {
  pending.push(
    limit(() => saveBatch(batch))
  );

  // Bound the number of queued batches as well.
  if (pending.length >= 3) {
    await Promise.all(pending.splice(0));
  }
}

await Promise.all(pending);
```

This allows up to three batches to be processed concurrently, with at most three batch promises held in `pending` at a time. It waits for the current group before submitting more work.

If one batch fails, `Promise.all()` rejects and the remaining submitted operations are not automatically cancelled. Choose `allSettled()` or explicit error handling if partial completion is required.

For high-throughput production pipelines, a dedicated bounded queue or worker pool can keep workers busy as soon as capacity becomes available instead of waiting for an entire group to finish.

---

## 6. Concurrency limiting vs rate limiting

These solve different problems.

- **Concurrency limit:** At most 10 operations are active at a time.
- **Rate limit:** At most 50 operations start per second.
- **Queue capacity:** At most 500 tasks are waiting to run.

A concurrency limit alone does not guarantee a request-per-second limit. If each request takes 10 ms, 10 concurrent workers could theoretically start hundreds of requests per second.

For external APIs, use both concurrency and rate limiting when required by the provider's policy.

A token bucket or sliding-window limiter can enforce request rates. On `429`, respect `Retry-After` when provided, and use bounded exponential backoff with jitter. Retry only failures that are safe to retry, and use idempotency keys for operations such as payments where duplicate execution would be harmful.

Also account for **per-tenant and global limits**: ten workers per incoming request can still produce excessive concurrency if 100 requests each create their own worker pool. Shared limits may be necessary across requests or application instances.

---

## 7. `AsyncLocalStorage` for request context

`AsyncLocalStorage` from `node:async_hooks` maintains context across asynchronous operations without passing the same metadata through every function argument.

Typical context includes:
- Request ID or trace ID.
- Authenticated user ID.
- Tenant or organization ID.
- Correlation metadata for logs and tracing.

### Example with Express

```js
import express from "express";
import { AsyncLocalStorage } from "node:async_hooks";
import { randomUUID } from "node:crypto";

const app = express();
const requestContext = new AsyncLocalStorage();

app.use((req, res, next) => {
  const context = {
    requestId: randomUUID(),
    userId: null,
    tenantId: null
  };

  requestContext.run(context, () => next());
});

function getContext() {
  return requestContext.getStore();
}

app.get("/records", async (req, res, next) => {
  try {
    const context = getContext();

    // In real applications, populate identity and tenant
    // from verified authentication and authorization data.
    context.userId = req.user?.id ?? null;
    context.tenantId = req.user?.tenantId ?? null;

    await loadRecords();

    res.json({ requestId: context.requestId });
  } catch (error) {
    next(error);
  }
});

async function loadRecords() {
  await new Promise(resolve => setTimeout(resolve, 10));

  console.log("Loading records", getContext());
}
```

The example assumes authentication middleware has already populated `req.user`. Never trust a tenant ID supplied by the client without authorization checks.

### Why it works

`requestContext.run(store, callback)` establishes a context for the callback and its supported asynchronous execution chain. Node propagates that context through standard asynchronous mechanisms such as promises and timers.

A database service, logger or API client can retrieve the current request ID without requiring every caller to pass it explicitly.

Use `run()` to establish request-scoped context. Avoid `enterWith()` for ordinary request middleware because it can unintentionally affect subsequent synchronous event handlers.

### Integrating context with logging

```js
function logInfo(message, extra = {}) {
  const context = requestContext.getStore();

  console.log(JSON.stringify({
    level: "info",
    message,
    requestId: context?.requestId,
    userId: context?.userId,
    tenantId: context?.tenantId,
    ...extra
  }));
}
```

In production, use a structured logger such as Pino and redact credentials, tokens and other sensitive fields. Ensure untrusted `extra` fields cannot overwrite trusted context fields; the example above is illustrative, and production code should control field precedence.

### Common pitfalls

1. **Context loss:** Some callback-based or custom asynchronous integrations may not preserve context correctly. Use `AsyncResource` or supported context-binding techniques when needed.
2. **Shared mutable state:** Every request should get its own store object. Do not reuse one global object across requests.
3. **Background jobs:** A task that runs after the request ends may need a new context or an explicitly propagated trace ID.
4. **Security:** Context is metadata, not an authorization mechanism. Always enforce tenant access at the service or data-access boundary.
5. **Memory leaks:** Do not retain request objects or large payloads in long-lived context stores.
6. **Distributed tracing:** `AsyncLocalStorage` works within one Node.js execution context. Propagate trace context explicitly through HTTP headers or messaging metadata when crossing process boundaries.

---

## 8. Practical production design

For a workload that imports 100,000 records and calls an external API, I would:

1. Read records incrementally using an async iterable or stream.
2. Validate each record before scheduling work.
3. Use a bounded queue or worker pool to cap active requests and waiting tasks.
4. Apply provider-specific rate limits in addition to concurrency limits.
5. Retry transient failures with bounded exponential backoff and jitter.
6. Collect per-record success and failure results; send permanent failures to a dead-letter queue if appropriate.
7. Use `AsyncLocalStorage` for request-scoped correlation, and explicitly propagate trace metadata to background jobs.
8. Measure queue depth, active workers, latency, retry count, failure rate and throughput.
9. Make writes idempotent and support graceful shutdown so in-flight work can finish or be safely resumed.

**Interview summary:** `Promise.all()` and `Promise.allSettled()` aggregate concurrent promises but do not limit concurrency. `for await...of` is useful for sequential async iteration, while streams provide incremental processing and backpressure. A worker pool, `p-limit` or a bounded queue controls concurrency; rate limiting controls how quickly operations start. `AsyncLocalStorage` propagates request-scoped metadata across asynchronous calls, improving logging and tracing without turning context into shared global state.

*Q19. Database access in Node. How do connection pools (mysql2, pg) work, and what size should they be relative to instances and DB max_connections? How do you handle transactions safely in async code (one connection per transaction, always release in finally), prepared statements, pool exhaustion symptoms, and query timeouts?*

## 1. How connection pools work

A database connection is a TCP connection with protocol state, authentication, session settings and server-side resources. Creating one for every query is expensive, so Node.js applications typically reuse connections through a **connection pool**.

Libraries such as `mysql2` and `pg` provide connection pooling.

The lifecycle is:

1. Application requests a connection from the pool.
2. If an idle connection is available, the pool assigns it.
3. If the pool has capacity, it creates a new connection.
4. If all connections are busy and the pool is at its limit, the request waits in a queue or fails, depending on the library and configuration.
5. After the query finishes, the connection is returned to the pool for reuse.

<text color="secondary" size="sm">A pool manages connections, not query execution itself. The database server still parses, executes and returns results.</text>

### MySQL with `mysql2`

```js
import mysql from "mysql2/promise";

const pool = mysql.createPool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,

  waitForConnections: true,
  connectionLimit: 10,
  queueLimit: 100,
  enableKeepAlive: true
});

const [rows] = await pool.execute(
  "SELECT id, name FROM users WHERE id = ?",
  [userId]
);
```

- `connectionLimit`: maximum connections in this pool.
- `waitForConnections`: whether requests wait when the pool is busy.
- `queueLimit`: maximum number of queued connection requests; `0` means unlimited.
- `execute()`: uses prepared-statement execution with parameter binding.

Create the pool once per application process, rather than creating a new pool for every request.

### PostgreSQL with `pg`

```js
import pg from "pg";

const { Pool } = pg;

const pool = new Pool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,

  max: 10,
  connectionTimeoutMillis: 2000,
  idleTimeoutMillis: 30000,
  maxLifetimeSeconds: 1800
});

const result = await pool.query(
  "SELECT id, name FROM users WHERE id = $1",
  [userId]
);
```

- `max`: maximum connections in the pool.
- `connectionTimeoutMillis`: maximum time to establish a new connection; it is not a general query execution timeout.
- `idleTimeoutMillis`: how long an idle connection can remain before being closed.
- `maxLifetimeSeconds`: optional maximum connection lifetime.

Check the exact options against the installed library version, particularly for lifetime and timeout behavior.

---

## 2. How large should the pool be?

The pool size must be planned across **all application instances**, not just one process.

The fundamental constraint is:

\[
\text{Total potential connections}
=
\text{Instances}\times\text{Pool max}
\]

Suppose PostgreSQL has `max_connections = 200`.

| Consumer | Count | Maximum connections each | Total |
|---|---:|---:|---:|
| API instances | 6 | 20 | 120 |
| Worker instances | 2 | 10 | 20 |
| Admin, migrations and monitoring | — | Reserved budget of 20 | 20 |
| Unallocated headroom | — | — | 40 |
| **Total budget** | | | **200** |

This is an arithmetic budget, not a recommended default configuration. In production, retain headroom for failover, traffic spikes and operational access.

A useful planning formula is:

\[
P \leq \frac{C-R}{N}
\]

Where:

- \(P\) = pool maximum per instance.
- \(C\) = database connection limit.
- \(R\) = reserved capacity for other consumers and headroom.
- \(N\) = maximum number of application and worker instances.

For example, with 200 connections, 40 reserved connections and 10 maximum instances:

\[
P \leq \frac{200-40}{10}=16
\]

That gives a theoretical ceiling of 16 connections per instance, not proof that 16 is optimal.

### Why bigger pools are not always faster

Increasing pool size can make performance worse because it can cause:

- More database contention.
- More concurrent disk I/O and CPU work.
- More lock contention.
- More memory usage and session overhead.
- Longer queues inside the database itself.

The right pool size depends on query latency, workload, database capacity, connection hold time and measured throughput.

Use load testing and production metrics to tune it. If connections are busy because queries are slow, optimize the queries before blindly increasing the pool.

**Important:** Autoscaling changes the connection budget. If your API scales from 5 to 30 instances, the total potential connection count can increase sixfold even though each instance's pool configuration remains unchanged.

---

## 3. Transactions: one connection for the entire transaction

A transaction must execute on the same database connection from `BEGIN` through `COMMIT` or `ROLLBACK`.

This is critical because transaction state is connection-specific. Using a pool-level query method for different transaction statements may execute them on different connections.

### Safe transaction with `mysql2`

```js
async function transferFunds(fromId, toId, amount) {
  const connection = await pool.getConnection();

  try {
    await connection.beginTransaction();

    const [debit] = await connection.execute(
      `UPDATE accounts
       SET balance = balance - ?
       WHERE id = ? AND balance >= ?`,
      [amount, fromId, amount]
    );

    if (debit.affectedRows !== 1) {
      throw new Error("Insufficient funds or account not found");
    }

    const [credit] = await connection.execute(
      `UPDATE accounts
       SET balance = balance + ?
       WHERE id = ?`,
      [amount, toId]
    );

    if (credit.affectedRows !== 1) {
      throw new Error("Destination account not found");
    }

    await connection.commit();
  } catch (error) {
    try {
      await connection.rollback();
    } catch (rollbackError) {
      // Log both errors; the connection may be unusable.
      console.error("Transaction rollback failed", rollbackError);
    }

    throw error;
  } finally {
    connection.release();
  }
}
```

### Why this pattern matters

- Acquire one connection for the transaction.
- Run every transactional statement on that connection.
- Commit only after all required operations succeed.
- Roll back on failure.
- Release the connection in `finally`, including when errors occur.

The example assumes `amount` has already been validated as a positive, correctly represented monetary amount. Real financial code should use an appropriate exact decimal or integer-minor-unit representation, enforce business invariants and consider concurrency controls such as row locks.

A further production concern: if `COMMIT` fails because the network connection drops, the client may not know whether the database committed the transaction. Do not blindly repeat a non-idempotent transaction. Use idempotency keys or a reconciliation strategy.

### Safe transaction with PostgreSQL `pg`

```js
async function createOrder(customerId, total) {
  const client = await pool.connect();

  try {
    await client.query("BEGIN");

    const orderResult = await client.query(
      `INSERT INTO orders (customer_id, total)
       VALUES ($1, $2)
       RETURNING id`,
      [customerId, total]
    );

    const orderId = orderResult.rows[0].id;

    await client.query(
      `INSERT INTO order_events (order_id, event_type)
       VALUES ($1, $2)`,
      [orderId, "created"]
    );

    await client.query("COMMIT");

    return orderId;
  } catch (error) {
    try {
      await client.query("ROLLBACK");
    } catch (rollbackError) {
      console.error("Transaction rollback failed", rollbackError);
    }

    throw error;
  } finally {
    client.release();
  }
}
```

With `pg`, use `pool.connect()` and execute all transaction queries through the returned client. Do not mix `pool.query()` into a transaction because it can use a different connection.

For both drivers, if the connection has become unusable, do not return it to the pool as if it were healthy. Follow the driver's connection-destruction or release-with-error API and verify the precise behavior for the installed version.

### Keep transactions short

Avoid holding a database connection while performing unrelated network calls:

```js
await client.query("BEGIN");

const response = await fetch("https://external-api.example/data");

await client.query("UPDATE orders SET status = $1 WHERE id = $2", [
  response.ok ? "processed" : "failed",
  orderId
]);

await client.query("COMMIT");
```

This pattern holds locks and a pool connection during the external request. If the API is slow, pool exhaustion can follow.

Prefer a design that performs external work outside the transaction where business consistency allows it. For reliable cross-system workflows, consider an outbox pattern, idempotent consumers or a saga instead of holding a database transaction open across a network request.

---

## 4. Prepared statements and parameterized queries

Prepared statements separate SQL structure from input values. They help prevent SQL injection and can reduce repeated parsing or planning overhead, depending on the driver and database.

### MySQL: `mysql2`

```js
const [rows] = await pool.execute(
  "SELECT id, email FROM users WHERE email = ?",
  [email]
);
```

`execute()` uses prepared-statement execution with parameters. `mysql2` can cache prepared statements per connection, so statement reuse and cache capacity matter when workloads contain many distinct SQL strings.

### PostgreSQL: `pg`

```js
const result = await pool.query({
  name: "user-by-email",
  text: "SELECT id, email FROM users WHERE email = $1",
  values: [email]
});
```

The `name` identifies a named prepared statement. PostgreSQL prepared statements are connection-specific, so using a pool means the statement may need to be prepared on each connection that executes it.

Avoid generating unbounded numbers of unique statement names.

### Parameters cannot replace identifiers

This does not safely parameterize a column name:

```js
// Incorrect approach for a dynamic column name:
await pool.query(
  "SELECT * FROM users ORDER BY $1",
  [columnName]
);
```

SQL parameters represent values, not identifiers or SQL syntax. For dynamic table names, column names or sort directions, use a strict allowlist and construct only the trusted SQL fragment.

```js
const allowedColumns = new Set(["name", "created_at"]);
const column = allowedColumns.has(requestedColumn)
  ? requestedColumn
  : "created_at";

const result = await pool.query(
  `SELECT id, name FROM users ORDER BY ${column} DESC LIMIT $1`,
  [100]
);
```

The interpolated identifier is safe here only because it is selected from the fixed allowlist.

---

## 5. Pool exhaustion: symptoms and diagnosis

Pool exhaustion occurs when all available connections are occupied and incoming operations cannot obtain one quickly enough.

### Common symptoms

- Requests become slow before SQL execution even begins.
- Connection acquisition timeouts or errors.
- MySQL errors such as `Too many connections`.
- PostgreSQL errors such as `remaining connection slots are reserved`.
- Increasing API latency and request timeouts.
- Database connections remaining active or idle in transactions.
- A queue of pending pool requests growing continuously.

Distinguish two situations:

1. **Pool saturation:** The application's pool is at capacity, but the database may still have spare capacity.
2. **Database connection exhaustion:** The database has reached its own connection limit, potentially due to many application instances, other services or leaked connections.

### Typical causes

- Pool maximum is too small for the workload.
- Queries are slow or blocked by locks.
- Transactions remain open too long.
- Connections are not released after manually acquiring them.
- A new pool is created for every request.
- Application replicas scale up without recalculating the global connection budget.
- Large amounts of work are queued without backpressure.

### What I monitor

| Metric | What it tells me |
|---|---|
| Total, idle and waiting pool connections | Whether the pool is saturated |
| Connection acquisition latency | Time spent waiting before a query can run |
| Query duration by operation | Slow SQL or downstream database latency |
| Database CPU, I/O and lock waits | Whether the database itself is overloaded |
| Active and idle-in-transaction sessions | Potential long transactions or application issues |
| Connection count by application and instance | Whether scaling or leaks are driving connection usage |

For PostgreSQL, inspect `pg_stat_activity` and `pg_stat_database`. For MySQL, inspect `SHOW PROCESSLIST`, `SHOW STATUS` and Performance Schema where available.

Fix the root cause. Increasing the pool is appropriate only if the database has capacity to support the additional concurrency.

---

## 6. Query timeouts and connection timeouts

These are different timeout categories:

- **Connection timeout:** How long the application waits to establish a connection.
- **Pool acquisition timeout:** How long it waits to obtain an available connection.
- **Query execution timeout:** How long a statement may run before cancellation or termination.
- **Application deadline:** The maximum time allowed for the complete request, including pool waiting, queries and other operations.

A timeout option on one layer does not automatically enforce all the others.

### PostgreSQL

Use a server-side statement timeout to limit query execution:

```js
const client = await pool.connect();

try {
  await client.query("BEGIN");

  await client.query(
    "SET LOCAL statement_timeout = '3s'"
  );

  const result = await client.query(
    "SELECT * FROM orders WHERE customer_id = $1",
    [customerId]
  );

  await client.query("COMMIT");
  return result.rows;
} catch (error) {
  try {
    await client.query("ROLLBACK");
  } catch {}

  throw error;
} finally {
  client.release();
}
```

`SET LOCAL` applies within the transaction and resets when that transaction ends. This example requires a transaction; configure an appropriate session-level or role/database-level timeout for queries executed outside transactions.

PostgreSQL's `statement_timeout` limits statement duration on the server. It is not a substitute for a bounded pool-acquisition timeout or a full request deadline.

### MySQL

With `mysql2`, a `connectTimeout` setting controls connection establishment, not general query execution. Do not assume it will terminate a long-running SQL statement.

For query deadlines, use the appropriate server-side timeout or cancellation mechanism for the specific MySQL operation and driver version. MySQL supports settings such as `max_execution_time` for applicable `SELECT` statements, but this is not a universal timeout for every statement type.

Also distinguish a client-side timeout from server-side cancellation: stopping the caller from waiting does not necessarily stop the database from executing the statement. If a query times out, determine whether the statement was cancelled and whether the connection remains safe to reuse.

### End-to-end deadline

Suppose an API has a 2-second response budget. Spending 1.8 seconds waiting for a pool connection and then allowing a 3-second query violates that budget.

Allocate the budget across stages:

- Pool acquisition: 200 ms.
- Query execution: 1 second.
- Serialization and other work: remaining budget.
- Reserve time for error handling and response delivery.

These are illustrative allocations, not universal defaults. Derive real values from latency targets and observed query distributions.

---

## 7. Production checklist

- Create one pool per application process, not per request.
- Calculate the maximum total connections across all replicas and workers.
- Leave database connection headroom for migrations, administration and other services.
- Use one dedicated connection per transaction and always release it in `finally`.
- Roll back on errors and handle rollback failures.
- Parameterize values; allowlist dynamic identifiers.
- Bound pool waiting and query execution time.
- Track pool saturation, acquisition latency, slow queries and database lock waits.
- Keep transactions short and avoid network calls inside them.
- Use graceful shutdown to stop accepting new work and close the pool cleanly.
- Never blindly retry a transaction when its commit outcome is uncertain.

**Interview summary:** A pool reuses database connections and caps per-process concurrency. Pool sizing must account for every application replica and reserve capacity for other database clients. Transactions require one connection from `BEGIN` to `COMMIT` or `ROLLBACK`, with release in `finally`. Parameterized queries prevent injection through values, while timeouts and pool metrics help distinguish connection starvation from slow SQL and database overload.

*Q20. Testing Node applications. How do you structure unit and integration tests with Jest/Vitest or the built-in node:test runner? How do you test HTTP endpoints with Supertest, mock external services (nock/MSW), use a real database through Testcontainers, test streams and timers (fake timers), and avoid shared-state flakiness between tests?*

## 1. Unit tests vs integration tests vs end-to-end tests

I use a test pyramid: many fast unit tests, a strong set of integration tests, and a smaller number of end-to-end tests covering critical user journeys.

| Test type | What it verifies | Example |
|---|---|---|
| Unit | A function or module in isolation | Price calculation, validation, permission checks |
| Integration | Multiple components working together | Service + repository + PostgreSQL |
| HTTP/API | Request routing, middleware and response behavior | `POST /orders` returns `201` |
| End-to-end (E2E) | A complete user-facing workflow | Login, create order, verify persisted order |
| Contract | Compatibility between service boundaries | API response matches an agreed schema |

**The key principle:** mock dependencies you do not want to exercise, but use real implementations for the behavior the test is supposed to verify.

For example, mock an external payment provider in a service unit test, but use a real PostgreSQL instance when testing transaction behavior.

## 2. Choosing a test runner

- **Jest:** mature ecosystem, mocking utilities, snapshots and broad community adoption.
- **Vitest:** fast, especially in Vite-based projects, with a Jest-like API.
- **`node:test`:** built into modern Node.js, with tests, hooks, mocking and subtests without installing a separate runner.

Use one primary runner per project unless there is a clear reason to maintain more than one.

### Example using the built-in `node:test`

```js
// price.js
export function calculateTotal(items) {
  return items.reduce((total, item) => {
    if (item.quantity < 0 || item.price < 0) {
      throw new Error("Invalid item");
    }

    return total + item.quantity * item.price;
  }, 0);
}
```

```js
// price.test.js
import test from "node:test";
import assert from "node:assert/strict";
import { calculateTotal } from "./price.js";

test("calculates the total for multiple items", () => {
  const total = calculateTotal([
    { price: 100, quantity: 2 },
    { price: 50, quantity: 3 }
  ]);

  assert.equal(total, 350);
});

test("rejects a negative quantity", () => {
  assert.throws(
    () => calculateTotal([{ price: 100, quantity: -1 }]),
    /Invalid item/
  );
});
```

Run it with:

```bash
node --test
```

This test verifies business logic without starting a server, connecting to a database or making network calls.

One issue with the implementation above: it validates only negative values, not malformed numbers such as `NaN` or non-numeric strings. Production validation should define and enforce the expected input types explicitly.

## 3. Structure the test suite

A practical structure for a Node.js API:

```text
src/
  routes/
    orders.routes.js
  services/
    orders.service.js
  repositories/
    orders.repository.js
  app.js
  server.js

test/
  unit/
    orders.service.test.js
  integration/
    orders.repository.test.js
  api/
    orders.api.test.js
  fixtures/
  helpers/
```

I separate application construction from listening on a port:

```js
// app.js
import express from "express";

export function createApp({ orderService }) {
  const app = express();

  app.use(express.json());

  app.post("/orders", async (req, res, next) => {
    try {
      const order = await orderService.create(req.body);
      res.status(201).json(order);
    } catch (error) {
      next(error);
    }
  });

  return app;
}
```

```js
// server.js
import { createApp } from "./app.js";
import { orderService } from "./dependencies.js";

const app = createApp({ orderService });

app.listen(process.env.PORT ?? 3000);
```

This separation makes `app` testable without opening a network port. It also allows dependency injection: tests can supply a fake service, while production supplies the real service.

A production application should additionally have centralized error-handling middleware and validation. Those behaviors should be tested too.

---

## 4. HTTP endpoint tests with Supertest

Supertest sends HTTP-style requests to an Express application and checks responses without requiring a fixed TCP port.

Install it alongside your chosen test runner:

```bash
npm install --save-dev supertest
```

Example with Jest or Vitest:

```js
import request from "supertest";
import { describe, it, expect, vi } from "vitest";
import { createApp } from "../../src/app.js";

describe("POST /orders", () => {
  it("creates an order and returns 201", async () => {
    const orderService = {
      create: vi.fn().mockResolvedValue({
        id: "order-123",
        total: 250
      })
    };

    const app = createApp({ orderService });

    const response = await request(app)
      .post("/orders")
      .send({ productId: "product-1", quantity: 2 })
      .expect("Content-Type", /json/)
      .expect(201);

    expect(response.body.id).toBe("order-123");
    expect(response.body.total).toBe(250);
    expect(orderService.create).toHaveBeenCalledWith({
      productId: "product-1",
      quantity: 2
    });
  });
});
```

This test verifies routing, JSON parsing, status code, response body and service invocation. The service itself is mocked, so this is an API-layer test, not a full integration test.

I also test:

- Invalid request bodies return `400` or the documented validation status.
- Unauthenticated requests return `401`.
- Authenticated users without permission receive `403`.
- Missing resources return `404`.
- Unexpected failures produce a safe `5xx` response without leaking stack traces or secrets.
- Pagination, filtering and content types behave as documented.

**Important:** test authorization at the endpoint, not just in isolated permission functions. A service can have correct permission logic while a route accidentally fails to invoke it.

---

## 5. Mocking external HTTP services: Nock and MSW

External APIs introduce nondeterminism: network failures, rate limits, changing data, latency and credentials. Most tests should not depend on a live third-party service.

### Nock

Nock intercepts Node.js HTTP requests and provides controlled responses. It is useful when the application uses compatible Node HTTP clients.

```js
import nock from "nock";
import test from "node:test";
import assert from "node:assert/strict";

test("handles a successful payment provider response", async () => {
  const scope = nock("https://payments.example")
    .post("/charges", {
      amount: 5000,
      currency: "INR"
    })
    .reply(201, {
      id: "charge-123",
      status: "succeeded"
    });

  const result = await createCharge({
    amount: 5000,
    currency: "INR"
  });

  assert.equal(result.status, "succeeded");
  assert.equal(scope.isDone(), true);
});
```

The example assumes `createCharge()` sends the matching request through a Nock-compatible HTTP transport.

A useful setup is to disable unexpected network requests in tests where all HTTP traffic should be mocked, and verify that expected interceptors were consumed. Clean up interceptors after each test.

Nock's interception behavior depends on the HTTP client and transport. Do not assume it intercepts every client automatically, particularly clients based on Undici or the built-in `fetch()` implementation.

### MSW

Mock Service Worker (MSW) defines reusable request handlers. It is particularly useful when the same mock API behavior must be shared across browser and Node.js tests.

```js
import { http, HttpResponse } from "msw";
import { setupServer } from "msw/node";

const server = setupServer(
  http.get("https://catalog.example/products/1", () => {
    return HttpResponse.json({
      id: "1",
      name: "Keyboard"
    });
  })
);

beforeAll(() => {
  server.listen({ onUnhandledRequest: "error" });
});

afterEach(() => {
  server.resetHandlers();
});

afterAll(() => {
  server.close();
});
```

This example uses Jest/Vitest-style lifecycle hooks; import the hooks from the selected runner when they are not available globally.

Use Nock when you need straightforward HTTP interception for a compatible Node client. Use MSW when reusable, declarative handlers across multiple testing environments are valuable.

Test failure scenarios as well as success:

- Connection refused and network errors.
- Request timeouts.
- `429` responses and `Retry-After`.
- `5xx` responses.
- Invalid JSON or unexpected response schemas.
- Retries, backoff and idempotency behavior.

Avoid mocking the external client's own implementation in every test. Instead, test your integration code against realistic HTTP responses so that URL construction, headers, serialization and response handling are exercised.

---

## 6. Integration tests using a real database and Testcontainers

Mocks cannot reliably verify SQL syntax, database constraints, transaction isolation, locking, migrations or database-specific behavior.

Testcontainers starts disposable services, usually through Docker, allowing integration tests to run against a real database with controlled setup and teardown.

For PostgreSQL:

```bash
npm install --save-dev @testcontainers/postgresql pg
```

Example using the built-in Node test runner:

```js
import test from "node:test";
import assert from "node:assert/strict";
import { PostgreSqlContainer } from "@testcontainers/postgresql";
import pg from "pg";

const { Pool } = pg;

test("persists and reads an order", async t => {
  const container = await new PostgreSqlContainer(
    "postgres:16-alpine"
  ).start();

  const pool = new Pool({
    connectionString: container.getConnectionUri(),
    max: 3
  });

  t.after(async () => {
    await pool.end();
    await container.stop();
  });

  await pool.query(`
    CREATE TABLE orders (
      id SERIAL PRIMARY KEY,
      customer_id INTEGER NOT NULL,
      total NUMERIC(12, 2) NOT NULL
    )
  `);

  const inserted = await pool.query(
    `INSERT INTO orders (customer_id, total)
     VALUES ($1, $2)
     RETURNING id, customer_id, total`,
    [42, "199.99"]
  );

  assert.equal(inserted.rows[0].customer_id, 42);
  assert.equal(inserted.rows[0].total, "199.99");
});
```

This test starts a real PostgreSQL container, creates a table, inserts a row and verifies the result. PostgreSQL drivers commonly return `NUMERIC` values as strings to preserve decimal precision.

For a real project, I would run the actual migrations instead of manually creating the table in every test. This also verifies that the migration scripts work.

### What should database integration tests cover?

- CRUD operations and query correctness.
- Unique constraints, foreign keys and other schema constraints.
- Rollback when an operation fails.
- Transaction isolation and concurrent updates.
- Pagination and ordering.
- Database migrations from a clean state.
- Timeouts and connection cleanup where practical.

### Isolation strategies

Use a separate database or schema for each test worker or suite. Then either:

- Wrap each test in a transaction and roll it back, where the application code and test setup use the same connection and this strategy is compatible with the test.
- Truncate tables between tests, respecting foreign-key constraints.
- Recreate a disposable database/schema for stronger isolation.

A transaction opened on a test connection will not automatically wrap queries issued by the application through another pool connection. This is a common source of false assumptions in test isolation.

**Operational considerations:** Testcontainers requires a working container runtime, and the first test run may need to download the image. Pin image versions rather than using `latest`, and ensure containers are stopped even when tests fail.

---

## 7. Testing streams and backpressure

Streams should be tested for their emitted data, error behavior, completion and cleanup—not only whether the happy path produces the expected output.

Node's `node:stream/promises` provides `pipeline()`, which propagates errors and coordinates stream completion.

```js
import test from "node:test";
import assert from "node:assert/strict";
import { Readable, Writable } from "node:stream";
import { pipeline } from "node:stream/promises";

test("transforms streamed chunks", async () => {
  const received = [];

  const source = Readable.from(["hello", " ", "world"]);

  const destination = new Writable({
    write(chunk, encoding, callback) {
      received.push(chunk.toString().toUpperCase());
      callback();
    }
  });

  await pipeline(source, destination);

  assert.deepEqual(received, ["HELLO", " ", "WORLD"]);
});
```

In real applications, test additional conditions:

- Large input processed without buffering everything in memory.
- A source that emits an error.
- A destination that fails or closes early.
- Correct handling of partial UTF-8 characters across chunks using `StringDecoder` when decoding binary streams.
- Backpressure when a slow writable consumer cannot keep up.
- Cleanup of files, sockets and other resources after cancellation.

For a transformation stream, test both the output and failure propagation. For a production pipeline, also measure memory use and verify that the implementation respects backpressure.

---

## 8. Testing timers, retries and scheduled jobs

Tests that depend on real time are slower and flaky. Use fake timers for deterministic tests involving delays, retries, debouncing, expiration or scheduled callbacks.

Example using Vitest:

```js
import { afterEach, describe, expect, it, vi } from "vitest";

function retryLater(callback, delay) {
  return setTimeout(callback, delay);
}

describe("retryLater", () => {
  afterEach(() => {
    vi.useRealTimers();
  });

  it("runs after the configured delay", () => {
    vi.useFakeTimers();

    const callback = vi.fn();

    retryLater(callback, 1000);

    expect(callback).not.toHaveBeenCalled();

    vi.advanceTimersByTime(999);
    expect(callback).not.toHaveBeenCalled();

    vi.advanceTimersByTime(1);
    expect(callback).toHaveBeenCalledOnce();
  });
});
```

For asynchronous callbacks, prefer the runner's asynchronous timer-advancement APIs, such as Vitest's `vi.advanceTimersByTimeAsync()`, when scheduled callbacks await promises or schedule additional timers.

Also test:

- Retry limits and exponential backoff.
- Cancellation before a timer fires.
- The behavior after the final failed attempt.
- Timer cleanup after a test.
- Whether overlapping scheduled jobs are prevented when required.

Avoid blindly advancing all timers in code that schedules recurring intervals indefinitely. It can produce infinite loops or unrealistic behavior.

Fake timers do not automatically control every source of time or asynchronous activity. For example, database I/O and real network requests are not made deterministic merely by faking `setTimeout`.

---

## 9. Preventing shared-state flakiness

A flaky test passes or fails depending on execution order, timing or environmental state rather than the behavior under test.

The most common causes are shared mutable objects, concurrent tests touching the same database rows, leaked timers, open handles and incomplete asynchronous cleanup.

### Rules I follow

1. **Create fresh fixtures per test.** Do not mutate a shared user, order or request object across tests.
2. **Isolate databases.** Use unique IDs, separate schemas or disposable databases; avoid parallel tests writing to the same rows.
3. **Restore mocks and timers.** Clear or restore spies, mock implementations and fake timers after each test.
4. **Await asynchronous work.** Return or await every promise that the test depends on. Do not let background work leak into the next test.
5. **Clean up resources.** Close database pools, HTTP servers, streams and containers.
6. **Avoid arbitrary sleeps.** Wait for explicit conditions, events or completion promises rather than using `setTimeout(..., 1000)` to guess when something finishes.
7. **Make external dependencies deterministic.** Mock HTTP calls or use controlled local services.
8. **Test parallel execution.** Run tests with concurrency enabled where appropriate to expose accidental shared state.
9. **Avoid order-dependent assertions.** Do not assume database rows, filesystem contents or mock call histories begin empty unless the test establishes that state.
10. **Keep production modules testable.** Avoid creating servers, timers, database pools or network connections as uncontrolled import-time side effects.

### Example cleanup with Vitest

```js
import { afterEach, vi } from "vitest";

afterEach(() => {
  vi.useRealTimers();
  vi.restoreAllMocks();
});
```

This is useful cleanup, but it does not reset every kind of mock state in every situation. If using `vi.fn()` with custom implementations or module mocks, choose the appropriate clearing, resetting or unstubbing API deliberately. Module-level singletons may also need explicit reset or dependency injection.

For diagnosing flaky tests, rerun the failing test repeatedly, run the suite in isolation, then run it alongside other tests. A test that passes alone but fails in the suite often indicates leaked state or a resource conflict.

---

## 10. A practical CI strategy

I would structure CI into stages:

1. **Unit tests:** run quickly on every pull request.
2. **API and integration tests:** run against disposable dependencies, including Testcontainers where appropriate.
3. **Static checks:** linting, type checking and coverage thresholds.
4. **E2E tests:** cover the highest-value business flows.
5. **Cleanup and diagnostics:** capture test reports and logs, and ensure test services are stopped even after failures.

Coverage is useful for finding untested code, but a high percentage does not guarantee good tests. Focus on business invariants, failure handling, authorization boundaries and concurrency-sensitive behavior.

**Interview summary:** I use unit tests for isolated logic, Supertest for HTTP behavior, Nock or MSW for controlled external HTTP calls, and Testcontainers for realistic database integration tests. Streams are tested for output, errors and backpressure; fake timers make time-dependent behavior deterministic. To avoid flaky tests, isolate state, await asynchronous work, clean up every resource and verify that tests can run independently and concurrently.

*Q21. TypeScript on the backend. How do you set up tsconfig for Node (module resolution, strict, path aliases and their runtime pitfalls), validate runtime data when types are erased (Zod with inferred types), type Express request extensions safely, and build and run in production (compiled JS vs tsx/ts-node)?*

## 1. Configure `tsconfig.json` correctly for Node.js

The first decision is whether the application uses **native ES modules (ESM)** or **CommonJS**. The TypeScript configuration, package settings, import syntax and runtime behavior must agree.

For a modern Node.js API using ESM, I would start with this configuration:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",

    "rootDir": "./src",
    "outDir": "./dist",

    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,

    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true,

    "sourceMap": true,
    "declaration": false
  },
  "include": ["src/**/*.ts"],
  "exclude": ["node_modules", "dist", "test"]
}
```

In `package.json`:

```json
{
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "build": "tsc -p tsconfig.json",
    "start": "node dist/server.js",
    "typecheck": "tsc --noEmit"
  }
}
```

Install the relevant development dependencies:

```bash
npm install --save-dev typescript tsx @types/node
```

Install Express and its types if you are building an Express API:

```bash
npm install express
npm install --save-dev @types/express
```

This is a starting point, not a universal configuration. The appropriate settings depend on the Node.js version, package dependencies, module format and whether tests or scripts use a different compilation setup.

### What the important compiler options do

| Option | Purpose |
|---|---|
| `target` | Controls the JavaScript language level emitted by TypeScript |
| `module: NodeNext` | Applies Node-aware ESM/CommonJS module rules |
| `moduleResolution: NodeNext` | Resolves imports according to Node's module rules |
| `strict` | Enables the main strict type-checking options |
| `noUncheckedIndexedAccess` | Makes potentially missing indexed values explicit in types |
| `exactOptionalPropertyTypes` | Distinguishes omitted optional properties from explicitly assigned `undefined` |
| `rootDir` | Defines the source directory structure expected in emitted output |
| `outDir` | Writes compiled JavaScript to `dist` |
| `sourceMap` | Generates maps for debugging compiled code |
| `esModuleInterop` | Improves interoperability with CommonJS packages |

`strict: true` is particularly important. It enables checks such as `strictNullChecks` and `noImplicitAny`, helping catch errors before deployment.

For example:

```ts
function getUserName(user: { name: string } | null): string {
  if (!user) {
    return "Unknown";
  }

  return user.name;
}
```

Without the null check, strict null checking would flag the unsafe access.

**Important:** `tsc` does not automatically make the application type-safe at runtime. It checks the code's static types and emits JavaScript; runtime validation is a separate concern.

---

## 2. ESM vs CommonJS and module resolution

Node.js supports both module systems.

**ESM:**

```ts
import express from "express";
import { createUser } from "./services/user.service.js";
```

**CommonJS:**

```ts
const express = require("express");
const { createUser } = require("./services/user.service");
```

For new TypeScript services, I generally prefer ESM when the dependency ecosystem and deployment environment support it consistently.

With `module` and `moduleResolution` set to `NodeNext`, TypeScript uses Node's package rules to determine how files and imports are interpreted.

For example, when compiling ESM TypeScript to JavaScript, write relative imports using the emitted `.js` extension:

```ts
// src/server.ts
import { createApp } from "./app.js";
```

Even though `app.ts` is the source file, Node will execute the emitted `app.js`. TypeScript understands this mapping under Node-aware resolution.

The `"type": "module"` setting in `package.json` tells Node to interpret ordinary `.js` files in that package scope as ESM.

### Common mistake

A project compiles successfully, but production fails with:

```text
Error [ERR_MODULE_NOT_FOUND]
```

Possible causes include:

- Missing `.js` extensions in relative ESM imports.
- Incorrect `package.json` module settings.
- Output files not included in the deployment.
- Incorrect import paths or case sensitivity differences between development and production.
- Dependencies whose module format differs from the application's assumptions.

The fix is to test the **compiled output with the actual Node.js runtime**, not just rely on a successful type check.

If maintaining a CommonJS application, configure the module system consistently rather than mixing ESM and CommonJS settings arbitrarily.

---

## 3. Path aliases and their runtime pitfalls

Path aliases make large projects easier to navigate.

For example:

```ts
import { UserService } from "@/services/user.service.js";
import { logger } from "@/infrastructure/logger.js";
```

You can define an alias in TypeScript:

```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

This tells TypeScript how to resolve the alias during compilation and type checking.

**The critical limitation:** `paths` does not generally rewrite import specifiers in emitted JavaScript.

The compiled file may still contain:

```js
import { UserService } from "@/services/user.service.js";
```

Plain Node.js does not automatically interpret `@/` as `src/`. The application may type-check and compile successfully, then fail at startup.

### Practical options

1. Use relative imports, particularly in smaller projects.
2. Use a bundler that resolves aliases and emits runtime-compatible output.
3. Use a runtime resolver or loader that explicitly supports your alias configuration.
4. Use Node.js package imports with the `#` prefix when appropriate, configured through `package.json`'s `"imports"` field.

For example, package imports can map an internal alias to compiled output:

```json
{
  "type": "module",
  "imports": {
    "#services/*": "./dist/services/*.js",
    "#infrastructure/*": "./dist/infrastructure/*.js"
  }
}
```

Source code can then use:

```ts
import { createUser } from "#services/user.service";
```

This approach needs a matching source-time mapping so TypeScript resolves the same import to the appropriate source file. The mapping must also account for how tests execute and how the project is built. Validate the exact configuration against your TypeScript and Node.js versions.

For backend APIs, I would avoid introducing an alias system unless it genuinely improves maintainability. A broken runtime resolver adds complexity without delivering business value.

---

## 4. Runtime validation with Zod

TypeScript types are erased during compilation. They do not validate incoming JSON, environment variables, database records or external API responses.

For example:

```ts
interface CreateUserInput {
  email: string;
  age: number;
}
```

This interface does not stop a client from sending:

```json
{
  "email": 123,
  "age": "twenty"
}
```

The server must validate untrusted data at runtime.

Zod provides runtime schemas and can infer TypeScript types from those schemas.

Install it:

```bash
npm install zod
```

### Define one schema and infer its type

```ts
import { z } from "zod";

export const CreateUserSchema = z.object({
  email: z.string().email(),
  age: z.number().int().min(18).max(120),
  name: z.string().trim().min(1).max(100)
});

export type CreateUserInput =
  z.infer<typeof CreateUserSchema>;
```

Now the schema is the source of truth for both runtime validation and the inferred TypeScript type.

### Validate incoming data

```ts
const result = CreateUserSchema.safeParse(req.body);

if (!result.success) {
  return res.status(400).json({
    error: "Validation failed",
    issues: result.error.issues.map(issue => ({
      path: issue.path,
      message: issue.message
    }))
  });
}

const input: CreateUserInput = result.data;
```

`safeParse()` returns a success/failure result instead of throwing for ordinary validation failures.

Use `parse()` when exceptions are the intended control flow; use `safeParse()` when handling invalid user input explicitly is more convenient.

### Validate environment variables too

```ts
import { z } from "zod";

const EnvSchema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]),
  PORT: z.coerce.number().int().min(1).max(65535),
  DATABASE_URL: z.string().min(1)
});

const env = EnvSchema.parse(process.env);

export default env;
```

This fails fast when required configuration is missing or malformed rather than allowing the service to fail later during a database connection.

### Type input and output can differ

Zod can coerce, transform or preprocess data. In these cases, distinguish the raw input type from the validated output type.

```ts
const AgeSchema = z.coerce.number().int().min(18);

type AgeInput = z.input<typeof AgeSchema>;
type AgeOutput = z.output<typeof AgeSchema>;

const age = AgeSchema.parse("24"); // number: 24
```

Here, the output is a number even though the input may be a string.

### Use schemas at trust boundaries

Validate:

- HTTP request bodies, query parameters and route parameters.
- Environment variables.
- External API responses.
- Message queue payloads.
- Imported files and webhooks.

You do not necessarily need to revalidate every internal function call when data has already been validated and remains under application control. Keep schemas close to the boundary where untrusted data enters the system.

Also remember that schema validation does not replace authorization. A valid `tenantId` is not proof that the authenticated user may access that tenant.

---

## 5. Extending Express request types safely

A common requirement is attaching authenticated user information or tenant context to `req`.

A quick but unsafe pattern is:

```ts
const userId = (req as any).userId;
```

This removes useful compiler checks and allows invalid assumptions to spread.

### Option A: Extend Express's request interface

Create a declaration file such as `src/types/express.d.ts`:

```ts
import "express";

declare global {
  namespace Express {
    interface Request {
      user?: {
        id: string;
        role: "admin" | "manager" | "viewer";
        tenantId: string;
      };
    }
  }
}

export {};
```

Now middleware and route handlers can access the property with proper types:

```ts
import type { Request, Response, NextFunction } from "express";

function requireUser(
  req: Request,
  res: Response,
  next: NextFunction
) {
  if (!req.user) {
    return res.status(401).json({
      error: "Unauthorized"
    });
  }

  next();
}
```

The property is optional because it does not exist until authentication middleware populates it.

Ensure the declaration file is included by `tsconfig.json`.

**Important:** interface augmentation only changes compile-time types. It does not populate `req.user`, validate it or prove authentication succeeded. The authentication middleware must establish the value from a verified credential.

### Option B: Use a typed request handler

For route-specific input types, Express's generic request types can describe route parameters, response bodies, request bodies and query parameters.

```ts
import type { Request, Response } from "express";

type CreateOrderBody = {
  productId: string;
  quantity: number;
};

type CreateOrderResponse = {
  id: string;
  status: "created";
};

function createOrder(
  req: Request<
    Record<string, never>,
    CreateOrderResponse,
    CreateOrderBody
  >,
  res: Response<CreateOrderResponse>
) {
  const { productId, quantity } = req.body;

  // Runtime validation is still required.
  return res.status(201).json({
    id: "order-123",
    status: "created"
  });
}
```

These generics describe what the handler expects and returns. They do not validate the incoming HTTP payload.

For most Express applications, I use declaration merging for cross-cutting request properties such as the authenticated user, and route-specific types plus Zod for endpoint input and output.

---

## 6. Compiled JavaScript vs `tsx` vs `ts-node`

These tools serve different purposes.

| Approach | How it works | Best use |
|---|---|---|
| `tsc` + Node.js | Compiles TypeScript to JavaScript, then runs the output | Production deployments |
| `tsx` | Executes TypeScript using a fast runtime transpilation workflow | Development and scripts |
| `ts-node` | Executes TypeScript through a Node.js integration | Development or specialized workflows |

### Development with `tsx`

```bash
npx tsx watch src/server.ts
```

This provides a convenient edit-and-restart workflow without a separate manual compilation step.

### Production with compiled JavaScript

```bash
npm run build
npm start
```

The deployment contains the compiled `dist/` directory, production dependencies and required runtime assets.

Benefits include:

- No TypeScript transpilation required at application startup.
- Clear separation between source and executable output.
- Easier artifact validation during CI.
- Less runtime tooling in the production image.

For production, I generally prefer **compile once in CI, then deploy the tested artifact**. That makes the build reproducible and avoids depending on development tools during startup.

### Does `tsc` guarantee correctness?

No. TypeScript compilation checks types, but it does not automatically run tests, validate runtime input or ensure all runtime imports resolve correctly.

A production pipeline should run:

```bash
npm ci
npm run typecheck
npm test
npm run build
node dist/server.js
```

In practice, build and test commands should be configured to match the repository's scripts. CI should also verify the compiled application starts and its critical health checks succeed.

### Production deployment considerations

- Use a supported Node.js LTS release.
- Pin dependency versions with a lockfile.
- Exclude unnecessary development dependencies from the final runtime image.
- Include required assets and migration files.
- Set environment variables and secrets outside the source code.
- Handle termination signals and close HTTP servers and database pools gracefully.
- Run the compiled artifact in a clean environment, not only from the developer's machine.

---

## 7. Recommended backend workflow

For a new Node.js + TypeScript API, my default would be:

1. ESM with `NodeNext`, provided dependencies support it.
2. `strict: true` and a small number of additional strictness options.
3. Explicit dependency injection and a clear separation between application construction and server startup.
4. Zod for runtime validation at external boundaries.
5. Express request augmentation for authenticated context, without treating type declarations as proof of authentication.
6. Unit tests for business logic and integration tests against real database containers.
7. Compiled JavaScript for production, with `tsx` for local development.
8. CI checks for type errors, tests, build output and runtime startup.

**Interview summary:** TypeScript provides static guarantees, not runtime validation. Node-aware module configuration keeps ESM/CommonJS behavior consistent, while path aliases require an actual runtime resolution strategy. Zod validates untrusted data and can infer types from schemas. Express request augmentation improves type safety but does not create or authenticate properties. For production, compile with `tsc`, test the emitted JavaScript and run it directly under Node.js.

*Q22. Background jobs and scheduling in Node. How do you run reliable jobs with BullMQ (Redis): retries with backoff, delayed and repeatable jobs, concurrency, stalled-job detection, idempotent processors and a dead-letter strategy? Why is setInterval or in-process node-cron unsafe when running multiple instances, and what replaces it?*

## 1. Why use a background job queue?

A background job moves slow or deferrable work out of the HTTP request lifecycle.

Examples include sending emails, processing uploaded files, generating reports, synchronizing external APIs, sending WhatsApp notifications and processing restaurant orders.

Instead of making the HTTP request wait for everything:

1. The API validates the request.
2. It records the required business state and enqueues a job.
3. It returns a response, often `202 Accepted` when processing is asynchronous.
4. A separate worker processes the job.
5. Failed jobs are retried or sent to a dead-letter workflow.

**BullMQ** is a Redis-backed job queue for Node.js. It supports distributed workers, retries, delayed jobs, concurrency controls, job scheduling and stalled-job recovery.

The important distinction is that **a queue makes work durable and coordinated, but does not automatically make job execution exactly once**. A job may execute more than once, so processors must be designed for duplicate delivery.

## 2. Basic BullMQ setup

Install BullMQ and the Redis client dependency:

```bash
npm install bullmq
```

Assume Redis is available through `REDIS_URL`.

### Define a queue and enqueue jobs

```js
// queue.js
import { Queue } from "bullmq";

export const emailQueue = new Queue("email", {
  connection: {
    url: process.env.REDIS_URL
  }
});

export async function enqueueWelcomeEmail(userId) {
  return emailQueue.add(
    "welcome-email",
    { userId },
    {
      attempts: 5,
      backoff: {
        type: "exponential",
        delay: 1000
      },
      removeOnComplete: 1000,
      removeOnFail: false
    }
  );
}
```

This example illustrates the queue options; verify the connection configuration against the installed BullMQ and Redis client versions. In deployments where the client expects host, port and authentication fields instead of a URL, parse the URL into the supported connection options.

`attempts: 5` permits up to five total processing attempts. Exponential backoff increases the delay between retries. Retaining failed jobs makes diagnosis and replay easier, but failed-job retention should be bounded operationally.

### Process jobs in a separate worker

```js
// worker.js
import { Worker } from "bullmq";

const worker = new Worker(
  "email",
  async job => {
    switch (job.name) {
      case "welcome-email":
        await sendWelcomeEmail(job.data.userId);
        break;

      default:
        throw new Error(`Unknown job: ${job.name}`);
    }
  },
  {
    connection: {
      url: process.env.REDIS_URL
    },
    concurrency: 10
  }
);

worker.on("failed", (job, error) => {
  console.error("Job failed", {
    jobId: job?.id,
    name: job?.name,
    error: error.message
  });
});

worker.on("error", error => {
  console.error("BullMQ worker error", error);
});
```

The worker runs independently from the API process. It can be deployed as a separate service and scaled independently according to queue depth and processing latency.

`concurrency: 10` allows up to ten jobs to be processed concurrently by this worker instance, subject to the processor's behavior and resource constraints. It is not a global limit across all worker replicas.

---

## 3. Retries and backoff

Transient failures should usually be retried. Permanent failures should not be retried indefinitely.

| Failure | Typical handling |
|---|---|
| Temporary network error | Retry with backoff |
| HTTP `429` | Respect `Retry-After` and provider limits |
| HTTP `503` | Retry with backoff and jitter where appropriate |
| Invalid payload | Fail permanently and investigate |
| Missing required business data | Usually fail permanently or wait for corrective action |
| Duplicate request | Detect through idempotency and return the existing outcome |

BullMQ supports fixed and exponential backoff.

```js
await emailQueue.add(
  "send-notification",
  { notificationId: "n-123" },
  {
    attempts: 5,
    backoff: {
      type: "exponential",
      delay: 2000
    }
  }
);
```

An exponential strategy increases the delay with each attempt. The exact timing depends on the configured delay and attempt number.

For production jobs, I also consider:

- Jitter to avoid many workers retrying simultaneously.
- Provider-specific rate limits.
- A maximum end-to-end job age or deadline.
- Cancellation or abandonment when the business request is no longer relevant.
- Whether the operation is safe to repeat.

BullMQ supports custom backoff strategies when provider-specific rules are required. For example, a processor can classify an error and use a custom delay rather than treating every failure identically.

**Do not retry every error.** Retrying an invalid payload five times only adds load and delays diagnosis.

---

## 4. Delayed and recurring jobs

### Delayed jobs

A delayed job becomes eligible for processing after the specified delay:

```js
await emailQueue.add(
  "payment-reminder",
  { invoiceId: "inv-123" },
  {
    delay: 60 * 60 * 1000
  }
);
```

This schedules the job to become eligible in approximately one hour. It does not guarantee execution at the exact deadline; actual execution depends on worker availability, queue load and infrastructure health.

Common uses include reminders, retry windows, scheduled notifications and delayed follow-up tasks.

### Recurring jobs

For recurring jobs, use BullMQ's supported Job Schedulers API in current versions rather than relying on every application instance to run its own timer.

For example, on BullMQ versions supporting `upsertJobScheduler()`:

```js
await emailQueue.upsertJobScheduler(
  "daily-summary",
  {
    pattern: "0 9 * * *"
  },
  {
    name: "generate-daily-summary",
    data: { reportType: "daily" },
    opts: {
      attempts: 3,
      backoff: {
        type: "exponential",
        delay: 5000
      }
    }
  }
);
```

This illustrates a daily 9:00 schedule in the scheduler's configured time interpretation. If the business requirement is a specific local timezone, configure and verify timezone behavior using the installed BullMQ version and its supported scheduling options.

Use a stable scheduler ID so repeated deployment or initialization does not create a new independent schedule every time.

Older BullMQ versions may use repeatable-job APIs instead. Check the documentation for the installed major version rather than mixing APIs from different releases.

Recurring scheduling is not the same as an exact-once guarantee. If a daily report must be generated once per business date, the processor should enforce that invariant using a database uniqueness constraint or another durable idempotency mechanism.

---

## 5. Concurrency and queue capacity

BullMQ workers support configurable concurrency:

```js
const worker = new Worker("reports", processReport, {
  connection: redisConnection,
  concurrency: 5
});
```

If you deploy four worker processes with concurrency five, the theoretical maximum is approximately 20 concurrently processed jobs, assuming all workers are healthy and no other bottleneck intervenes.

Choose concurrency based on the workload:

- **I/O-heavy jobs:** Higher concurrency may improve throughput.
- **CPU-heavy jobs:** More promises do not make CPU-intensive JavaScript run in parallel on a single event loop. Use worker threads or separate processes for CPU-intensive computation.
- **Database-heavy jobs:** Limit concurrency to protect the database connection budget and avoid lock contention.
- **Rate-limited external APIs:** Coordinate rate limits across all worker replicas, not just within each process.

A concurrency setting alone does not limit how quickly requests are sent to an external provider. Use BullMQ's rate-limiting capabilities where appropriate, and account for provider quotas shared across workers or services.

Also control queue growth. If jobs arrive faster than workers complete them, Redis storage and job age can grow without bound. Apply admission controls, queue-depth alerts, retention policies and capacity planning.

---

## 6. Stalled-job detection and worker crashes

A worker can stop processing because of:

- A process crash or container restart.
- A machine failure.
- A blocked Node.js event loop.
- A long pause that prevents the worker from maintaining its Redis lock.
- Redis connectivity problems.

BullMQ uses locks and stalled-job detection to recover jobs whose worker no longer appears to own them. A recovered job can be returned to waiting and processed by another worker, subject to the queue's stalled-job limits and version-specific behavior.

This creates a crucial consequence:

**A job can execute again even if its earlier execution performed some side effects.**

For example:

1. A worker sends an email successfully.
2. The worker crashes before BullMQ records successful completion.
3. The job is recovered and retried.
4. The recipient may receive the email twice.

Stalled-job recovery protects against lost work, but it cannot undo a completed external side effect.

### Operational practices

- Keep job processors bounded and avoid blocking the event loop.
- Use worker threads or separate processes for CPU-heavy tasks.
- Configure lock duration and stalled-job checks according to the workload and BullMQ version.
- Monitor stalled-job events and repeated recoveries.
- Make processors idempotent.
- Ensure workers shut down gracefully and finish or safely abandon in-flight work.

Increasing lock duration indiscriminately is not a complete fix for event-loop blocking. It can delay recovery from a genuinely dead worker.

---

## 7. Idempotent processors

An idempotent operation produces the same intended business outcome when repeated with the same logical input.

For background jobs, idempotency is essential because retries, worker crashes and stalled-job recovery can cause duplicate execution.

Suppose a queue job requests an invoice payment. A naive processor might charge the customer each time it runs. That is unsafe.

A stronger design uses a stable business operation ID:

```js
async function processPayment(job) {
  const { paymentRequestId } = job.data;

  const paymentRequest = await paymentRepository
    .getById(paymentRequestId);

  if (paymentRequest.status === "succeeded") {
    return paymentRequest;
  }

  return paymentProvider.charge({
    amount: paymentRequest.amount,
    currency: paymentRequest.currency,
    idempotencyKey: paymentRequestId
  });
}
```

This is illustrative: a production implementation also needs atomic state transitions, error handling and reconciliation for ambiguous provider outcomes. The provider must actually support the supplied idempotency key.

### Stronger database guarantees

For database-owned operations, use a unique constraint or transactional claim:

```sql
CREATE TABLE processed_jobs (
  job_key VARCHAR(200) PRIMARY KEY,
  processed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

Within a transaction, claim the key and apply the business update only if the claim succeeds. A duplicate key indicates that the logical operation has already been claimed or processed, depending on the transaction design.

The claim and business mutation must be coordinated atomically. Inserting a processed marker in one transaction and performing the actual operation outside it can still leave inconsistent outcomes after a crash.

For external side effects, combine durable state tracking with the provider's idempotency support or a reconciliation process. A database transaction alone cannot make an external API call atomic with the database commit.

---

## 8. Dead-letter strategy

A dead-letter queue (DLQ) is a durable place for jobs that cannot be processed successfully after the allowed retry policy.

A sensible workflow is:

1. Apply bounded retries for transient failures.
2. Keep failed-job metadata, including the job ID, payload reference, attempt count, timestamps and error classification.
3. Alert on repeated failures and dead-letter volume.
4. Inspect and correct the root cause.
5. Replay jobs deliberately after confirming that replay is safe.

BullMQ retains failed jobs when configured to do so, so a separate queue is not always necessary just to preserve failure history. A separate DLQ is useful when failed jobs need distinct retention, access controls, alerting or replay workflows.

Example of an explicit dead-letter transfer:

```js
import { Queue, Worker } from "bullmq";

const deadLetterQueue = new Queue("notifications-dlq", {
  connection: redisConnection
});

const worker = new Worker(
  "notifications",
  processNotification,
  {
    connection: redisConnection,
    concurrency: 10
  }
);

worker.on("failed", async (job, error) => {
  if (!job) return;

  const attemptsAllowed = job.opts.attempts ?? 1;
  const attemptsUsed = job.attemptsMade;

  if (attemptsUsed >= attemptsAllowed) {
    await deadLetterQueue.add(
      "failed-notification",
      {
        originalJobId: job.id,
        originalName: job.name,
        originalData: job.data,
        errorMessage: error.message,
        attemptsMade: attemptsUsed,
        failedAt: new Date().toISOString()
      },
      {
        jobId: `failed-${job.id}`
      }
    );
  }
});
```

This is a simplified pattern, not a complete atomic DLQ implementation. Confirm how the installed BullMQ version updates `attemptsMade` and emits failure events, including cases where a job is moved to failed or retried.

There is also a failure window: the process can crash after the original job fails but before the DLQ enqueue succeeds. For critical workflows, use a durable failure record or a carefully designed reconciliation process rather than relying solely on an event callback. If the original job may be replayed, ensure DLQ insertion and replay operations are themselves idempotent.

Do not blindly replay all dead-letter jobs. Some may represent permanent validation errors, expired requests or business operations whose external outcome is unknown.

---

## 9. Why `setInterval` and in-process `node-cron` are unsafe across instances

Consider this code:

```js
setInterval(async () => {
  await generateDailyReport();
}, 60_000);
```

It can be acceptable for a simple local development task, but it is not a reliable distributed scheduler.

### Problems

**Duplicate execution:** If five API replicas run this code, all five execute the task every minute.

**Lost executions:** If the process is down at the scheduled time, the task does not run. A normal interval does not persist missed work.

**Overlapping runs:** If a job takes longer than the interval, the next invocation may start before the previous one finishes.

**No durable queue:** There is no built-in persistent record of pending jobs, retry attempts or failures.

**No distributed coordination:** In-process timers do not coordinate across containers or machines.

**Deployment and clock behavior:** Restarts, pauses, clock changes and event-loop delays can affect timing.

`node-cron` improves cron expression handling, but an ordinary in-process scheduler still runs independently in every replica unless you add reliable distributed coordination.

### What replaces it?

Use a durable, distributed scheduling approach appropriate to the workload:

- **BullMQ Job Schedulers:** Redis-backed scheduling that feeds work into BullMQ workers.
- **Managed cloud schedulers:** For example, AWS EventBridge Scheduler or Google Cloud Scheduler, often triggering a queue or service.
- **OS cron or a platform scheduler:** Suitable for controlled single-host or infrastructure-managed jobs, with external coordination if multiple instances could run them.
- **A database-backed scheduler or leader-elected scheduler:** When requirements justify the additional implementation and operational complexity.

For an application already using Redis and BullMQ, I would normally use BullMQ for job execution and scheduling, then scale worker processes independently from API instances.

Regardless of scheduler, enforce business-level idempotency. A distributed scheduler reduces duplicate scheduling, but it does not eliminate retries, crashes or ambiguous execution outcomes.

---

## 10. Reliability checklist for production

- Use a durable queue instead of relying on in-process timers for critical jobs.
- Configure bounded retries, backoff and error classification.
- Keep recurring schedules stable across deployments.
- Enforce idempotency at the database or external-provider boundary.
- Monitor queue depth, oldest job age, processing latency, retries and stalled jobs.
- Retain failed-job metadata and define a DLQ inspection/replay process.
- Set queue retention limits to prevent unbounded Redis growth.
- Coordinate external API and database limits across all worker replicas.
- Handle shutdown signals and close workers and queue connections cleanly.
- Test process crashes, duplicate delivery, Redis interruptions and retry exhaustion.

**Interview summary:** BullMQ provides durable queueing, retries, delayed and recurring jobs, concurrency control and stalled-job recovery through Redis. Reliability still depends on idempotent processors, bounded retries, operational monitoring and a dead-letter workflow. In-process timers and `node-cron` do not coordinate multiple replicas or preserve missed work; use a distributed scheduler and durable queue, while designing every critical job to tolerate repeated execution.

*Q23. Real-time with WebSockets/Socket.IO. How do you authenticate a WebSocket connection, manage rooms, and scale across instances with a Redis adapter? How do you handle reconnection, missed messages, heartbeats, per-connection memory and backpressure when a slow client can’t keep up?*

In production, I treat real-time communication as a distributed messaging system, not just a persistent connection. The main concerns are **authentication, authorization, room management, horizontal scaling, reconnection, message delivery guarantees, and slow-client handling**.

## 1. WebSockets vs. Socket.IO

- **WebSocket** provides a persistent, bidirectional connection over a single TCP connection.
- **Socket.IO** adds higher-level features such as named events, rooms, automatic reconnection, acknowledgements, and adapters for cross-instance broadcasting.

Socket.IO is **not wire-compatible with a plain WebSocket client**. Both are useful, but I choose based on the application's requirements.

## 2. Authenticate and authorize connections

A connection should be authenticated during the Socket.IO handshake, before the server allows access to protected events or rooms.

For browser clients, I generally use secure session cookies where practical, or a short-lived access token supplied through Socket.IO's `auth` option. I avoid putting tokens in query parameters because URLs are more likely to be logged.

```javascript
// Client
import { io } from "socket.io-client";

const socket = io("https://api.example.com", {
  auth: {
    token: accessToken,
  },
  reconnection: true,
});
```

Server-side authentication:

```javascript
import { Server } from "socket.io";

const io = new Server(httpServer, {
  cors: {
    origin: ["https://app.example.com"],
    credentials: true,
  },
});

io.use(async (socket, next) => {
  try {
    const token = socket.handshake.auth?.token;

    if (!token) {
      return next(new Error("Unauthorized"));
    }

    // Verify signature, issuer, audience, expiry, and claims.
    const user = await verifyAccessToken(token);

    socket.data.userId = user.id;
    socket.data.roles = user.roles;

    next();
  } catch {
    next(new Error("Unauthorized"));
  }
});
```

Important details:

- **Verify, don't decode:** validate the token cryptographically and check its expiry, issuer, audience, and expected claims.
- **Authorize each sensitive event:** authentication proves identity, not permission to access every resource.
- **Validate payloads:** enforce schemas and size limits on incoming events.
- **Protect browser handshakes:** configure allowed origins and, when using cookies, appropriate CSRF protections. CORS alone is not authentication.
- **Handle revocation:** disconnect or revalidate sessions when accounts are disabled, tokens expire, or permissions change. A token verified at connection time does not automatically become invalid when its claims change.

For example, a user joining `order:123` must be checked against the database or trusted authorization state to confirm they can access order 123.

## 3. Manage rooms safely

Rooms are logical groups of sockets. Typical examples include:

- `user:42` — all active sessions for a particular user.
- `order:123` — clients authorized to receive updates for one order.
- `tenant:7` — authorized users belonging to one tenant.

```javascript
io.on("connection", (socket) => {
  const userId = socket.data.userId;

  socket.join(`user:${userId}`);

  socket.on("order:subscribe", async ({ orderId }, ack) => {
    try {
      if (!isValidOrderId(orderId)) {
        return ack?.({ ok: false, error: "Invalid order ID" });
      }

      const allowed = await canViewOrder(userId, orderId);

      if (!allowed) {
        return ack?.({ ok: false, error: "Forbidden" });
      }

      await socket.join(`order:${orderId}`);
      ack?.({ ok: true });
    } catch {
      ack?.({ ok: false, error: "Subscription failed" });
    }
  });
});
```

Broadcasting to a room:

```javascript
io.to("order:123").emit("order:updated", {
  orderId: "123",
  status: "preparing",
});
```

**Security rule:** never trust a client-supplied room name. Derive room names from validated identifiers and authorize access before joining. On logout or permission changes, remove affected sockets from protected rooms or disconnect them.

For multi-tenant systems, include tenant identity in authorization checks. A room naming convention by itself does not enforce tenant isolation.

## 4. Scale across instances with Redis

A single Socket.IO server can manage its own connections and rooms. When running multiple instances behind a load balancer, each instance ordinarily knows about its own local sockets.

For example:

- Instance A owns Alice's connection.
- Instance B owns Bob's connection.
- A message emitted only through A will not automatically reach Bob's connection on B.

A Redis adapter allows Socket.IO instances to distribute broadcasts and related adapter operations across servers.

```javascript
import { createClient } from "redis";
import { createServer } from "node:http";
import { Server } from "socket.io";
import { createAdapter } from "@socket.io/redis-adapter";

const httpServer = createServer();

const io = new Server(httpServer, {
  cors: {
    origin: ["https://app.example.com"],
    credentials: true,
  },
});

const pubClient = createClient({
  url: process.env.REDIS_URL,
});

const subClient = pubClient.duplicate();

await Promise.all([
  pubClient.connect(),
  subClient.connect(),
]);

io.adapter(createAdapter(pubClient, subClient));

httpServer.listen(3000);
```

Now a broadcast such as `io.to("order:123").emit(...)` can reach authorized room members connected to different instances.

### What the Redis adapter does—and does not do

- **Does:** distribute Socket.IO broadcasts between instances using Redis Pub/Sub.
- **Does not:** automatically persist messages for disconnected clients or guarantee durable, exactly-once delivery.
- **Does not:** eliminate the need for authorization on every server.
- **Does not:** make every Redis-backed operation durable; Pub/Sub messages are transient.

The standard Redis adapter also does not provide the connection-state recovery feature supported by some other Socket.IO adapters. Verify the adapter's recovery support before designing around it.

### Load balancing and sticky sessions

If HTTP long-polling is enabled, requests belonging to the same Socket.IO session generally need to reach the same server. Configure sticky sessions at the load balancer, or use WebSocket-only transport if its trade-offs are acceptable.

WebSocket-only transport avoids the polling-session affinity requirement, but clients lose the polling fallback. Network restrictions and proxy configuration still matter.

I also monitor Redis connectivity, publish/subscribe failures, adapter latency, and connection counts per instance. Redis failure can interrupt cross-instance delivery even if existing local connections remain alive.

## 5. Reconnection and missed messages

Socket.IO supports automatic reconnection, but **reconnection does not mean that every message sent during a disconnection will be replayed**.

A robust client should reconnect with backoff and jitter, refresh expired credentials, and resynchronize its state.

```javascript
const socket = io("https://api.example.com", {
  auth: (callback) => {
    callback({ token: getCurrentAccessToken() });
  },
  reconnection: true,
  reconnectionAttempts: Infinity,
  reconnectionDelay: 500,
  reconnectionDelayMax: 5000,
  randomizationFactor: 0.5,
});

socket.on("connect", () => {
  // Re-establish subscriptions and synchronize missed state.
  socket.emit("orders:sync", {
    lastSeenSequence: getLastSeenSequence(),
  });
});
```

The exact callback shape should match the Socket.IO client version in use.

### Reliable resynchronization strategy

For important business events, I use a durable source of truth, such as a database event table or message broker.

1. Assign each event a stable ID or monotonic sequence number within the relevant stream.
2. Persist the event before considering it committed.
3. Let the client track the last successfully processed sequence.
4. On reconnect, request events after that sequence.
5. Replay them in order, subject to retention and authorization.
6. Deduplicate by event ID and update the client's checkpoint only after processing.

Example synchronization query:

```sql
SELECT event_id, sequence_no, payload
FROM order_events
WHERE order_id = ?
  AND sequence_no > ?
ORDER BY sequence_no ASC
LIMIT 500;
```

If the client is too far behind or the event history has expired, return a fresh snapshot instead of replaying an unbounded backlog.

Socket.IO connection-state recovery can help restore session state and missed packets in supported configurations, but it has a recovery window and adapter-specific limitations. I still implement application-level resynchronization for critical data.

**Interview distinction:** acknowledgements and retries can help establish whether an operation was received or processed, but they do not independently guarantee exactly-once business effects. Use idempotency keys and durable state.

## 6. Heartbeats and dead connections

WebSocket connections can appear alive even after a network interruption, device sleep, or broken proxy path. The server needs a way to detect dead peers.

Socket.IO has a built-in heartbeat mechanism based on ping and pong packets, configured using options such as:

```javascript
const io = new Server(httpServer, {
  pingInterval: 25_000,
  pingTimeout: 20_000,
});
```

These values are examples, not universal defaults. Tune them against network conditions, mobile-device sleep behavior, proxy timeouts, and the latency budget.

For raw WebSockets, implement ping/pong handling if the library does not provide it. Also configure compatible proxy and load-balancer idle timeouts.

When a heartbeat fails, the server closes the connection and releases associated resources. The client can then reconnect and resynchronize.

## 7. Per-connection memory and connection lifecycle

A WebSocket connection is relatively lightweight, but thousands of connections can consume substantial memory through socket buffers, session data, event listeners, subscriptions, and application-level queues.

I follow these practices:

- Store only minimal identity and connection metadata in `socket.data`.
- Do not store entire database entities, large payloads, or unbounded histories on the socket.
- Avoid registering duplicate listeners during reconnection or repeated subscriptions.
- Remove timers, listeners, subscriptions, and other resources during disconnect.
- Enforce connection limits per user, IP, and tenant, with sensible exceptions for legitimate multi-device usage.
- Set maximum inbound payload sizes and validate message schemas.
- Measure heap usage, active connections, RSS, event-loop delay, and memory per connection.

Example lifecycle cleanup:

```javascript
io.on("connection", (socket) => {
  const cleanup = () => {
    clearInterval(socket.data.someTimer);
    // Release application-owned resources and subscriptions.
  };

  socket.on("disconnect", cleanup);
});
```

Do not assume all application resources are automatically cleaned up merely because the socket disconnects.

## 8. Backpressure and slow clients

Backpressure is one of the most important production concerns. A server can generate messages faster than a client can receive or process them. If the application buffers every message, memory usage can grow until the server becomes unstable.

Socket.IO does not automatically give application code a complete, durable backpressure policy. I design one based on message semantics.

### Choose a policy by message type

| Message type | Recommended strategy |
|---|---|
| Typing indicators, cursor positions | Drop stale updates; keep only the latest |
| Live metrics or dashboard values | Coalesce updates or sample |
| Presence | Keep the latest state per user |
| Chat messages | Persist, sequence, acknowledge, and replay |
| Payments or order transitions | Durable event log, idempotency, and resynchronization |

For replaceable data, use a bounded queue or a latest-value slot. If the client is slow, discard obsolete intermediate states rather than accumulating them.

For important events, don't silently discard data: persist it, cap the live queue, and let the client recover from the durable source.

### Example: bound application-level pending work

```javascript
const MAX_PENDING = 100;

function enqueueForClient(clientState, message) {
  if (clientState.pending.length >= MAX_PENDING) {
    // Mark the client as needing a snapshot or resync.
    clientState.needsResync = true;
    clientState.pending.length = 0;
    return false;
  }

  clientState.pending.push(message);
  return true;
}
```

This is only a queue-policy illustration; a production implementation also needs a drain loop, delivery acknowledgements where required, and limits on bytes—not just message count. For snapshot-style updates, coalescing by key is often better than clearing and replaying every update.

With raw WebSocket libraries, check the socket's buffered amount and stop sending or close the connection when configured limits are exceeded. With Socket.IO, account for its own buffering and transport behavior as well as any application queue.

I also enforce:

- A maximum queued byte count per client.
- A maximum message size.
- A bounded amount of in-flight work.
- A slow-client threshold and timeout.
- Metrics for queue size, dropped/coalesced updates, resyncs, and forced disconnects.

A key principle is that **TCP flow control alone is not sufficient**: it does not stop an application from building an unbounded queue above the transport layer.

## 9. Production architecture

A typical design looks like this:

```text
Browser / Mobile clients
          |
   Load balancer
   (sticky sessions if polling)
          |
   +------+------+
   |             |
Socket.IO A   Socket.IO B
   |             |
   +------+------+
          |
    Redis adapter
   (cross-instance broadcasts)

Business API / Workers
          |
   Database / durable event log
          |
  Replay and snapshot API
```

The Redis adapter distributes real-time notifications; the database or durable event log remains the source of truth for critical business state.

For important updates, I prefer a transactional outbox: commit the business change and an outbox event in the same database transaction, then publish the event asynchronously. This avoids the failure window where a database update succeeds but the process crashes before publishing its notification.

## 10. What I monitor in production

- Active connections and connection churn per instance.
- Authentication failures and authorization denials.
- Reconnect frequency and handshake latency.
- Heartbeat timeouts and disconnect reasons.
- Redis adapter errors and cross-instance broadcast latency.
- Event-loop lag, heap usage, RSS, and garbage-collection pauses.
- Per-client queued messages and bytes.
- Dropped/coalesced updates, replay volume, and resynchronization failures.
- End-to-end message latency for important events.

### Interview-ready summary

“I authenticate clients during the handshake, authorize every sensitive subscription or event, and use rooms for logical fan-out. For horizontal scaling, I configure the Socket.IO Redis adapter and sticky sessions when polling is enabled. I don't assume reconnecting restores every missed message: critical events are persisted, sequenced, replayed, and deduplicated. Socket.IO heartbeats detect dead connections, while connection limits and lifecycle cleanup control memory. For slow clients, I bound queues, coalesce replaceable updates, and disconnect or resynchronize clients that fall too far behind. Redis handles cross-instance fan-out; durable storage handles business correctness.”

*Q24. Native addons, performance profiling and runtime internals. How do you profile a slow endpoint (--prof, --cpu-prof, flame graphs with clinic/0x, perf_hooks)? What are the roles of V8, libuv and N-API/node-gyp, how does require resolution and caching work, and what do the permission model, --watch, the single executable and the built-in test runner change in modern Node versions?*

In production, I approach a slow Node.js endpoint by measuring where time is spent before changing the code. The bottleneck could be JavaScript execution, database queries, network latency, event-loop blocking, garbage collection, or native code.

## 1. Profile a slow endpoint

My workflow is:

1. Measure endpoint latency and throughput under representative load.
2. Separate application time from database and external-service time.
3. Check CPU utilization, event-loop delay, memory usage, and garbage collection.
4. Capture a CPU profile or flame graph.
5. Identify the dominant functions and validate the suspected bottleneck.
6. Optimize and benchmark again under the same workload.

A single slow request is not enough to prove a CPU bottleneck. For example, an endpoint waiting 800 ms for a database query may show very little CPU activity.

### A. `--cpu-prof`: capture a V8 CPU profile

Start your application with:

```bash
node --cpu-prof --cpu-prof-dir=./profiles app.js
```

Reproduce the slow endpoint, then stop the process cleanly so the profile can be finalized. Node writes a V8 CPU profile that can be inspected in Chrome DevTools under **Performance** by loading the profile.

Use this to identify functions consuming CPU time, including JavaScript execution and visible native frames.

### B. `--prof`: V8's lower-level profiler

```bash
node --prof app.js
```

Reproduce the issue and stop the process. V8 writes a log file, which can be processed using:

```bash
node --prof-process isolate-*.log
```

The output summarizes hot functions and sampled execution. It can expose JavaScript and native-code activity, although interpreting native frames may require more expertise.

**Difference:** `--cpu-prof` produces a profile convenient for DevTools; `--prof` produces a V8 profiler log that you process for textual summaries and deeper analysis.

Both are sampling profilers. They estimate where execution time is spent rather than tracing every operation, so very short functions may not appear reliably.

### C. Flame graphs with Clinic.js or 0x

A flame graph visualizes sampled stack traces. Wider frames represent more sampled execution, not necessarily a function's total wall-clock duration.

For example, a CPU flame graph might show:

```text
HTTP handler
  └── buildReport
       └── normalizeRows
            └── expensiveTransform
```

If `expensiveTransform` dominates the samples, inspect its algorithm, allocations, repeated work, and input size.

Common tools include:

- **0x:** generates flame graphs from Node.js profiling data.
- **Clinic.js:** provides profiling and diagnostic tools, including CPU-focused analysis and event-loop diagnostics.

Follow each tool's current CLI documentation for installation and invocation because command options can vary between versions.

A flame graph is particularly useful for identifying:
- Repeated JSON parsing or serialization.
- Expensive regular expressions.
- Synchronous file operations.
- Large loops and inefficient algorithms.
- Excessive object allocation and garbage collection.
- Native functions that consume significant CPU.

A CPU flame graph alone does not explain why an endpoint is waiting on a slow external service. Use distributed tracing and timing instrumentation for that.

## 2. Use `perf_hooks` to measure application behavior

Node's built-in `node:perf_hooks` module provides high-resolution timing and performance instrumentation.

### Measure a critical section

```javascript
import { performance } from "node:perf_hooks";

async function getReport() {
  const start = performance.now();

  try {
    return await buildReport();
  } finally {
    const elapsedMs = performance.now() - start;
    console.log({ operation: "buildReport", elapsedMs });
  }
}
```

In production, use a structured logger or metrics system rather than logging every request synchronously. Record histograms and percentiles, such as p50, p95, and p99 latency.

### Measure event-loop delay

```javascript
import { monitorEventLoopDelay } from "node:perf_hooks";

const histogram = monitorEventLoopDelay({ resolution: 20 });
histogram.enable();

setInterval(() => {
  console.log({
    p99EventLoopDelayMs: histogram.percentile(99) / 1e6,
    maxEventLoopDelayMs: histogram.max / 1e6,
  });

  histogram.reset();
}, 10_000).unref();
```

The histogram reports delay in nanoseconds, hence the division by \(10^6\) to convert to milliseconds. This is a process-level diagnostic, not a per-request measurement.

High event-loop delay can indicate CPU-heavy JavaScript, synchronous blocking, or other event-loop stalls. It is not by itself proof of the exact cause.

You can also use `performance.mark()` and `performance.measure()` to instrument specific phases of a request, or `PerformanceObserver` to collect supported performance entries.

### CPU time vs. wall-clock time

- **CPU time:** time spent executing on a CPU.
- **Wall-clock time:** elapsed time from start to finish, including waits.

A request with high wall-clock latency but low CPU time is often waiting on I/O. A request with high CPU utilization and high event-loop delay may be doing too much work on the event-loop thread.

## 3. V8, libuv, and the Node.js runtime

Node.js is not just V8. It combines a JavaScript engine, native runtime APIs, and asynchronous I/O infrastructure.

<text> </text>

| Component | Responsibility |
|---|---|
| V8 | Parses and compiles JavaScript, executes it, optimizes hot code, manages the JavaScript heap, and performs garbage collection |
| libuv | Implements the event loop, asynchronous I/O abstractions, timers, and a worker pool used by certain operations |
| Node.js core | Exposes APIs such as HTTP, streams, files, networking, modules, and process management |
| Native addons | Extend Node with compiled native code, commonly C or C++, for platform APIs or specialized workloads |

### How asynchronous I/O works

Consider:

```javascript
const contents = await fs.promises.readFile("report.json", "utf8");
```

The JavaScript thread initiates an asynchronous operation and can process other work while the operation is pending. Once it completes, Node schedules the continuation.

The exact implementation depends on the operation:

- Network I/O generally uses operating-system readiness or completion mechanisms integrated with libuv.
- Some filesystem operations use libuv's thread pool.
- Certain cryptographic and compression operations also use the thread pool.
- JavaScript callbacks and promise continuations still execute on the JavaScript thread unless work is explicitly moved elsewhere.

**Important distinction:** `async`/`await` does not automatically make CPU-intensive JavaScript run in parallel. A large synchronous loop still blocks that event loop. For CPU-heavy work, consider `worker_threads`, separate processes, or an external worker service.

The libuv worker pool is not the same as `worker_threads`. Increasing `UV_THREADPOOL_SIZE` may help workloads that actually use that pool, but it does not parallelize arbitrary JavaScript.

## 4. Native addons, N-API, and `node-gyp`

A native addon is compiled code that Node can load into a process. Addons are useful when an application needs an operating-system API unavailable in core Node, must integrate with a native library, or has a demonstrated performance bottleneck that justifies native complexity.

### N-API

N-API provides a stable ABI for native addons across supported Node.js versions. An addon using N-API can often be reused across Node versions without recompilation, provided the required N-API version and platform constraints are satisfied.

It does **not** guarantee portability across all operating systems, CPU architectures, or external native-library ABIs.

### `node-gyp`

`node-gyp` is a build tool that generates native build files and compiles addons. Depending on the platform and toolchain, building may require Python, a C/C++ compiler, and platform-specific development tools.

A package containing native addons may fail to install when:
- A compatible prebuilt binary is unavailable.
- The local compiler or build tools are missing.
- The addon does not support the current Node ABI or architecture.
- An external native library is missing or incompatible.

### When would I write a native addon?

I first profile the application. Native code introduces memory-safety risks, debugging complexity, build and deployment constraints, and potential crashes that can terminate the entire Node process.

Before writing one, I consider algorithm improvements, streaming, caching, native libraries already available through maintained packages, and `worker_threads`.

For a stable addon API, I generally prefer N-API over directly depending on V8 internals, unless there is a specific reason to couple the addon to a particular engine version.

## 5. How `require()` resolution and caching work

CommonJS `require()` does more than load a file. Node resolves the requested module, evaluates it if necessary, and caches its exports.

```javascript
const config = require("./config");
const configAgain = require("./config");

console.log(config === configAgain); // Usually true
```

For a resolved CommonJS module, Node normally evaluates the module once per resolved cache entry and returns the cached `module.exports` on subsequent `require()` calls.

### Resolution process

For a request such as:

```javascript
const helper = require("./utils/helper");
const express = require("express");
const fs = require("node:fs");
```

Node generally:

1. Recognizes built-in modules, relative paths, absolute paths, or package names.
2. Resolves the specifier using the relevant CommonJS resolution rules.
3. Finds a file or package entry point, considering supported extensions, package metadata, and directory/package rules.
4. Loads and evaluates the module if it is not already cached.
5. Returns its `module.exports`.

The `node:` prefix explicitly identifies a built-in module, such as `node:fs`.

For package imports, resolution can involve `node_modules` lookup and package metadata such as `main` and `exports`. The `exports` field can restrict which package subpaths consumers are allowed to import.

### Caching and its pitfalls

- The CommonJS cache is available through `require.cache`.
- Cache identity is based on the resolved module filename, so different resolution paths can sometimes produce separate instances.
- Editing a module on disk does not automatically re-evaluate an already cached module.
- Deleting a cache entry can force a subsequent load, but it is not a safe general-purpose hot-reloading strategy.
- Circular dependencies can expose partially initialized exports before a module has finished executing.

Example:

```javascript
console.log(Object.keys(require.cache));
```

**Interview distinction:** CommonJS `require()` is synchronous and uses the CommonJS module cache. ECMAScript modules use a separate module-loading and caching model, with static imports, asynchronous module loading where necessary, and live bindings. Their resolution rules are not identical.

## 6. The modern Node.js permission model

The Node.js permission model provides a way to restrict selected capabilities of a process. Depending on the Node version, options can restrict filesystem access, network access, child processes, workers, native addons, and other capabilities.

For example, in versions supporting these flags:

```bash
node --permission \
  --allow-fs-read=./config \
  app.js
```

This illustrates a restricted filesystem-read grant; check the documentation for your exact Node release to confirm supported permissions and syntax.

The goal is to apply least privilege: give an application only the capabilities it needs.

However, the permission model is **not a complete sandbox**. Its protections and exceptions depend on the Node version, granted permissions, and how code is executed. Review the documentation carefully before treating it as a security boundary.

I would still use:
- Least-privilege operating-system users and containers.
- Restricted network access and firewall rules.
- Secrets management.
- Dependency auditing and supply-chain controls.
- Filesystem and process isolation where appropriate.

The permission model complements these controls; it does not replace them.

## 7. `--watch`: restart during development

Node's watch mode automatically restarts a process when watched files change.

```bash
node --watch app.js
```

For a test-driven development loop:

```bash
node --watch --test
```

This is useful for local development and rapid feedback. It is not a production process supervisor.

In production, I use a process manager, container orchestrator, or service manager that can restart crashed processes, enforce resource limits, manage deployments, and provide health checks.

Watch behavior and stability status depend on the Node release, so verify the version in the project's runtime requirements.

## 8. Single Executable Applications (SEA)

Node.js supports creating a single executable application that bundles a Node runtime with an application entry point and, in supported configurations, additional assets.

The general workflow is:

1. Prepare the application entry point.
2. Create the SEA configuration.
3. Generate the SEA preparation blob using the tooling provided by the Node release.
4. Inject or package the blob into a Node executable using the supported procedure for that release.
5. Test the resulting executable on the target platform.

A SEA can simplify distribution when you want to deliver a command-line tool or standalone application without asking users to install Node separately.

However, it is not equivalent to compiling JavaScript into native machine code. The executable still embeds a Node runtime, and platform-specific packaging, external files, native dependencies, and runtime behavior require testing.

**Key limitation:** native addons and some dynamic loading patterns may not work in the same way as in a conventional Node installation. SEA tooling and supported capabilities have evolved across releases, so follow the documentation for the exact version you deploy.

## 9. The built-in test runner

Node includes a test runner through `node:test`, avoiding the need for a third-party test framework for many projects.

```javascript
import test from "node:test";
import assert from "node:assert/strict";

function add(a, b) {
  return a + b;
}

test("adds two numbers", () => {
  assert.equal(add(2, 3), 5);
});
```

Run tests with:

```bash
node --test
```

The built-in runner supports test suites, subtests, hooks, skipping and filtering tests, and integration with Node's assertion APIs. Exact capabilities depend on the installed Node version.

For asynchronous behavior:

```javascript
test("fetches a report", async () => {
  const report = await fetchReport();

  assert.equal(report.status, "ready");
});
```

A rejected promise or thrown error causes the test to fail unless the test explicitly expects that failure.

### When would I use it instead of Jest or Vitest?

The built-in runner is attractive when:
- I want fewer dependencies.
- Standard assertions and test lifecycle hooks are sufficient.
- The project already targets a modern Node version.

I might choose Jest or Vitest when the project benefits from their mocking systems, ecosystem integrations, snapshot workflows, or particular developer tooling. For HTTP integration tests, I may still add a client such as Supertest, and for database tests I may use a real database or Testcontainers.

The built-in test runner reduces dependencies; it does not eliminate the need for a test strategy.

## 10. Production troubleshooting example

Suppose an endpoint takes 2.5 seconds at p95 and CPU usage is high.

I would proceed as follows:

1. **Measure:** establish p50/p95/p99 latency, throughput, error rate, CPU, RSS, and event-loop delay.
2. **Decompose:** instrument database queries, external HTTP calls, serialization, and application logic separately.
3. **Profile:** capture a CPU profile under representative load and inspect the dominant stack frames.
4. **Diagnose:** distinguish expensive computation from excessive allocation, garbage collection, blocking calls, or waiting on I/O.
5. **Optimize:** improve the algorithm, reduce repeated work, stream large payloads, cache appropriate results, or move CPU-intensive work to workers.
6. **Verify:** rerun the same load test and check latency percentiles, throughput, resource use, and correctness.

If the endpoint is waiting on a database, a CPU flame graph may not reveal the primary problem. If the event loop is blocked, adding more asynchronous syntax will not solve it. If a native addon is responsible, inspect its frames and library behavior rather than assuming the JavaScript code is at fault.

### Interview-ready summary

“I start with latency metrics and event-loop diagnostics, then use `--cpu-prof`, `--prof`, or flame graphs to find CPU hotspots. V8 executes JavaScript and manages its heap, while libuv provides the event loop and asynchronous I/O infrastructure. Native addons extend Node through compiled code, and N-API offers a more stable ABI than direct V8 integration. CommonJS `require()` resolves modules and caches their exports, while ESM follows a distinct loading model. Modern Node features add development watch mode, a built-in test runner, a permission model, and single-executable packaging—but each has version-specific limitations and does not replace production observability, operating-system security, or proper deployment practices.”