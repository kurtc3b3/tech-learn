
## What & When

**multiprocessing** spawns separate Python interpreter processes, each with its own memory space and GIL. Unlike threads, processes truly run in parallel on multiple CPU cores — making it the right tool for CPU-bound work.

Use multiprocessing when:

- Heavy computation — image processing, ML inference, data transforms
- Bypassing the **GIL** (Global Interpreter Lock) for true parallelism
- Isolating crashes — a failed worker doesn't kill the main process
- Running CPU-bound tasks alongside an async FastAPI server

Do NOT use multiprocessing for:

- I/O-bound work — use `asyncio` or `threading` instead
- Sharing large amounts of data — IPC has overhead
- Simple scripts — the spawn overhead is not worth it

---

## multiprocessing vs threading vs asyncio

||multiprocessing|threading|asyncio|
|---|---|---|---|
|Parallelism|✅ True (multi-core)|❌ GIL-limited|❌ Single thread|
|Best for|CPU-bound|Blocking I/O (legacy)|Async I/O|
|Memory|Separate per process|Shared|Shared|
|Communication|IPC (Queue, Pipe)|Shared objects|Queues, Events|
|Overhead|High (spawn)|Medium|Low|
|Crash isolation|✅ Yes|❌ No|❌ No|

---

## Basic Process

```python
from multiprocessing import Process

def worker(name: str):
    print(f"Worker {name} running in PID {os.getpid()}")

if __name__ == "__main__":
    p = Process(target=worker, args=("Alice",))
    p.start()
    p.join()        # wait for process to finish
    print(f"Exit code: {p.exitcode}")
```

> [!warning] Always use `if __name__ == "__main__":` On Windows and macOS (spawn start method), the module is re-imported in each child process. Without this guard, spawning recurses infinitely.

---

## Process Lifecycle

```python
from multiprocessing import Process
import time

def task():
    time.sleep(5)

p = Process(target=task)

p.start()               # spawn the process
p.is_alive()            # True while running
p.pid                   # OS process ID
p.name                  # process name (settable)
p.exitcode              # None while running, int after done
p.join(timeout=3)       # wait up to 3 seconds
p.terminate()           # SIGTERM — ask to stop
p.kill()                # SIGKILL — force stop
p.close()               # release resources (after join/terminate)
```

---

## Process Pool — `Pool`

A pool of worker processes that execute tasks from a queue. Simpler than managing individual processes.

```python
from multiprocessing import Pool

def square(n: int) -> int:
    return n * n

if __name__ == "__main__":
    with Pool(processes=4) as pool:

        # map — applies function to each item, blocks until all done
        results = pool.map(square, range(10))
        print(results)   # [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

        # map with chunk size — batches items for efficiency
        results = pool.map(square, range(1000), chunksize=100)

        # starmap — multiple arguments per call
        results = pool.starmap(pow, [(2, 10), (3, 5), (4, 3)])
        print(results)   # [1024, 243, 64]

        # imap — lazy iterator, results as they complete
        for result in pool.imap(square, range(10)):
            print(result)

        # imap_unordered — faster, results in completion order
        for result in pool.imap_unordered(square, range(10)):
            print(result)
```

---

## `apply_async` — Non-Blocking Pool Tasks

```python
from multiprocessing import Pool

def process_image(path: str) -> str:
    # heavy image transform
    return f"processed: {path}"

def on_success(result):
    print(f"Done: {result}")

def on_error(error):
    print(f"Failed: {error}")

if __name__ == "__main__":
    with Pool(4) as pool:
        result = pool.apply_async(
            process_image,
            args=("/images/photo.jpg",),
            callback=on_success,
            error_callback=on_error,
        )

        # Do other work while task runs...
        value = result.get(timeout=10)   # block and retrieve
```

---

## `ProcessPoolExecutor` — Modern API

`concurrent.futures.ProcessPoolExecutor` is the modern, higher-level interface. Integrates naturally with asyncio via `run_in_executor`.

```python
from concurrent.futures import ProcessPoolExecutor, as_completed

def cpu_task(n: int) -> int:
    return sum(i * i for i in range(n))

if __name__ == "__main__":
    with ProcessPoolExecutor(max_workers=4) as executor:

        # Submit individual tasks
        future = executor.submit(cpu_task, 1_000_000)
        result = future.result(timeout=10)

        # Map over iterable
        results = list(executor.map(cpu_task, [100_000] * 8))

        # Process as completed
        futures = {executor.submit(cpu_task, n): n for n in range(8)}
        for future in as_completed(futures):
            n = futures[future]
            try:
                result = future.result()
                print(f"cpu_task({n}) = {result}")
            except Exception as e:
                print(f"cpu_task({n}) failed: {e}")
```

---

## Integrating with asyncio — `run_in_executor`

Run CPU-bound work in a process pool without blocking the FastAPI event loop.

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

def heavy_computation(data: list) -> list:
    # runs in a separate process — GIL not an issue
    return [x ** 2 for x in data]

# Shared executor — reuse across requests
executor = ProcessPoolExecutor(max_workers=4)

async def process_data(data: list) -> list:
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(executor, heavy_computation, data)
    return result

# In FastAPI
from fastapi import FastAPI
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    executor.shutdown(wait=True)        # clean up on shutdown

app = FastAPI(lifespan=lifespan)

@app.post("/process")
async def process_endpoint(data: list[int]):
    result = await process_data(data)
    return {"result": result}
```

> [!warning] Functions passed to `run_in_executor` must be picklable Lambda functions, closures, and methods on non-picklable objects cannot be sent to child processes. Use module-level functions instead.

---

## Communication — `Queue`

Safe data exchange between processes.

```python
from multiprocessing import Process, Queue

def producer(q: Queue):
    for i in range(5):
        q.put(f"item-{i}")
    q.put(None)                 # sentinel

def consumer(q: Queue):
    while True:
        item = q.get()
        if item is None:
            break
        print(f"Consumed: {item}")

if __name__ == "__main__":
    q = Queue(maxsize=10)
    p1 = Process(target=producer, args=(q,))
    p2 = Process(target=consumer, args=(q,))

    p1.start(); p2.start()
    p1.join();  p2.join()
```

---

## Communication — `Pipe`

Lower-level, faster than Queue for two-process communication.

```python
from multiprocessing import Process, Pipe

def child(conn):
    conn.send("Hello from child")
    msg = conn.recv()
    print(f"Child got: {msg}")
    conn.close()

if __name__ == "__main__":
    parent_conn, child_conn = Pipe()
    p = Process(target=child, args=(child_conn,))
    p.start()

    msg = parent_conn.recv()
    print(f"Parent got: {msg}")
    parent_conn.send("Hello from parent")

    p.join()
```

---

## Shared Memory

For large data that would be expensive to pickle and send through a queue.

```python
from multiprocessing import Process, Value, Array
import ctypes

def increment(counter, lock):
    for _ in range(1000):
        with lock:
            counter.value += 1

if __name__ == "__main__":
    from multiprocessing import Lock

    counter = Value(ctypes.c_int, 0)    # shared integer
    lock    = Lock()

    processes = [Process(target=increment, args=(counter, lock)) for _ in range(4)]
    for p in processes: p.start()
    for p in processes: p.join()

    print(counter.value)                # 4000
```

```python
# Shared array
arr = Array(ctypes.c_double, [1.0, 2.0, 3.0, 4.0])

# Large data with shared_memory (Python 3.8+)
from multiprocessing.shared_memory import SharedMemory
import numpy as np

shm = SharedMemory(create=True, size=1024 * 1024)   # 1MB
array = np.ndarray((256, 256), dtype=np.float32, buffer=shm.buf)
# share shm.name with child processes to attach
shm.close()
shm.unlink()        # free the shared memory
```

---

## `Manager` — Shared High-Level Objects

Share Python dicts, lists, and other objects across processes.

```python
from multiprocessing import Process, Manager

def worker(shared_dict, key, value):
    shared_dict[key] = value

if __name__ == "__main__":
    with Manager() as manager:
        shared = manager.dict()         # proxy dict shared across processes
        shared_list = manager.list()    # proxy list

        processes = [
            Process(target=worker, args=(shared, f"key-{i}", i))
            for i in range(5)
        ]
        for p in processes: p.start()
        for p in processes: p.join()

        print(dict(shared))             # {"key-0": 0, ..., "key-4": 4}
```

> [!tip] Manager vs Value/Array `Manager` objects are accessed via a proxy (slower, network-like IPC). `Value`/`Array` use actual shared memory (faster, but limited to simple types).

---

## `Lock`, `Event`, `Semaphore`

```python
from multiprocessing import Lock, Event, Semaphore

# Lock — mutual exclusion
lock = Lock()
with lock:
    # only one process at a time
    shared_resource.write(data)

# Event — signal between processes
event = Event()
event.set()             # signal
event.clear()           # reset
event.wait(timeout=5)   # block until set
event.is_set()          # check state

# Semaphore — limit concurrency
sem = Semaphore(3)      # max 3 processes at once
with sem:
    do_limited_work()
```

---

## Start Methods

```python
import multiprocessing as mp

# fork   — fast, copies parent memory (default on Linux)
#          can cause issues with threads or open file handles
# spawn  — clean slate, safe (default on Windows & macOS)
#          slower — re-imports the module
# forkserver — hybrid, safer than fork with threads

mp.set_start_method("spawn")    # call once at program start

# Or per-context
ctx = mp.get_context("spawn")
p = ctx.Process(target=worker)
```

---

## Worker Class Pattern

Cleaner than bare functions for stateful workers.

```python
from multiprocessing import Process, Queue
import time

class WorkerProcess(Process):
    def __init__(self, task_queue: Queue, result_queue: Queue):
        super().__init__()
        self.task_queue   = task_queue
        self.result_queue = result_queue
        self.daemon = True          # die when parent dies

    def run(self):
        while True:
            task = self.task_queue.get()
            if task is None:        # sentinel
                break
            result = self.process(task)
            self.result_queue.put(result)

    def process(self, task):
        time.sleep(0.1)             # simulate work
        return task * 2

if __name__ == "__main__":
    tasks   = Queue()
    results = Queue()

    workers = [WorkerProcess(tasks, results) for _ in range(4)]
    for w in workers: w.start()

    for i in range(20):
        tasks.put(i)

    for _ in workers:
        tasks.put(None)             # one sentinel per worker

    for w in workers: w.join()

    while not results.empty():
        print(results.get())
```

---

## Multiprocessing with FastAPI — Real Pattern

Offload CPU-heavy work without blocking the event loop.

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor
from contextlib import asynccontextmanager
from fastapi import FastAPI
from pydantic import BaseModel

# Module-level — picklable
def analyse_data(payload: dict) -> dict:
    import pandas as pd                 # import inside worker
    df = pd.DataFrame(payload["rows"])
    return {"mean": df["value"].mean(), "std": df["value"].std()}

executor = ProcessPoolExecutor(max_workers=4)

@asynccontextmanager
async def lifespan(app: FastAPI):
    yield
    executor.shutdown(wait=True)

app = FastAPI(lifespan=lifespan)

class DataPayload(BaseModel):
    rows: list[dict]

@app.post("/analyse")
async def analyse(payload: DataPayload):
    loop = asyncio.get_event_loop()
    result = await loop.run_in_executor(
        executor,
        analyse_data,
        payload.model_dump(),
    )
    return result
```

---

## Common Pitfalls

> [!warning] Pickling Everything sent between processes must be picklable. Lambdas, open file handles, database connections, and lock objects cannot be pickled. Use module-level functions and re-initialise resources inside the worker.

> [!warning] Import side effects With the `spawn` start method, child processes re-import the module. Heavy imports (torch, tensorflow, cv2) run again in each worker. Use an initialiser function to load them once per worker:

```python
def init_worker():
    global model
    import torch
    model = torch.load("model.pt")      # loaded once per worker process

with ProcessPoolExecutor(max_workers=4, initializer=init_worker) as executor:
    results = executor.map(predict, inputs)
```

> [!warning] Forking with threads `fork` + threads = danger. If the parent has threads when it forks, child processes inherit a corrupted threading state. Use `spawn` or `forkserver` when mixing multiprocessing with asyncio or threading.

---

## Quick Reference

|Task|Code|
|---|---|
|Single process|`Process(target=fn, args=(...))`|
|Process pool|`Pool(n).map(fn, items)`|
|Modern pool|`ProcessPoolExecutor(max_workers=n)`|
|Async integration|`loop.run_in_executor(executor, fn, args)`|
|Send/receive data|`Queue` or `Pipe`|
|Shared scalar|`Value(ctypes.c_int, 0)`|
|Shared array|`Array(ctypes.c_double, [...])`|
|Shared dict/list|`Manager().dict()` / `Manager().list()`|
|Mutual exclusion|`Lock()`|
|Signal processes|`Event()`|
|Limit workers|`Semaphore(n)`|
|Pre-load in worker|`ProcessPoolExecutor(initializer=fn)`|
|Set start method|`mp.set_start_method("spawn")`|

---

## Related Notes

- [[Python - asyncio]]
- [[FastAPI - Overview & Key Advantages]]
- [[FastAPI - Lifespan]]

## Tags

#python #multiprocessing #concurrency #parallelism #cpu #backend