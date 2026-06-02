 
## What & When

**tqdm** (from Arabic _taqaddum_ — progress) is a fast, extensible progress bar library for Python. It wraps any iterable and displays a real-time progress bar in the terminal, Jupyter notebooks, or any stream — with zero changes to your loop logic.

Use tqdm when:

- Processing large files, datasets, or API pages
- Long-running loops where feedback matters
- Data pipelines, ETL scripts, ML training epochs
- CLI tools where UX matters
- Parallel processing with visible per-worker progress

```bash
pip install tqdm
```

---

## Basic Usage

```python
from tqdm import tqdm
import time

# Wrap any iterable
for item in tqdm(range(100)):
    time.sleep(0.01)

# Output:
# 100%|████████████████████| 100/100 [00:01<00:00, 98.76it/s]
```

---

## What the Bar Shows

```
 73%|███████████████       | 730/1000 [00:07<00:02, 97.3it/s]
 ↑        ↑                    ↑         ↑          ↑
pct    filled bar            n/total  elapsed/remaining  rate
```

---

## Core Parameters

```python
from tqdm import tqdm

tqdm(
    iterable,
    desc="Processing",      # label shown left of bar
    total=1000,             # total count (inferred if iterable has __len__)
    unit="file",            # unit label — default "it"
    unit_scale=True,        # scale unit (1000 → 1k)
    unit_divisor=1024,      # divisor for scaling (use 1024 for bytes)
    leave=True,             # keep bar after completion (False = erase)
    ncols=80,               # bar width in characters
    mininterval=0.1,        # min seconds between display updates
    maxinterval=10.0,       # max seconds between display updates
    miniters=1,             # min iterations between updates
    ascii=False,            # use ASCII chars instead of Unicode blocks
    disable=False,          # set True to silence (e.g. in non-TTY)
    dynamic_ncols=True,     # auto-resize bar to terminal width
    colour="green",         # bar colour (green, red, blue, yellow, etc.)
    smoothing=0.3,          # ETA smoothing factor (0=avg, 1=instant)
    initial=0,              # starting value for n
    position=0,             # line position for nested bars
)
```

---

## Description & Postfix

```python
from tqdm import tqdm

# Static description
for item in tqdm(items, desc="Downloading"):
    download(item)

# Dynamic postfix — update during loop
pbar = tqdm(items, desc="Training")
for epoch, item in enumerate(pbar):
    loss = train(item)
    pbar.set_description(f"Epoch {epoch}")
    pbar.set_postfix(loss=f"{loss:.4f}", lr=0.001)

# Output:
# Epoch 42: 100%|████| 1000/1000 [00:10<00:00, 98it/s, loss=0.0023, lr=0.001]
```

---

## Manual Control — `tqdm` as Context Manager

When total is unknown upfront or you need full control.

```python
from tqdm import tqdm

with tqdm(total=1000, desc="Fetching pages", unit="page") as pbar:
    page = 1
    while True:
        data = fetch_page(page)
        if not data:
            break
        process(data)
        pbar.update(len(data))     # advance by batch size
        pbar.set_postfix(page=page)
        page += 1
```

---

## `trange` — Drop-in for `range`

```python
from tqdm import trange
import time

for i in trange(100, desc="Counting"):
    time.sleep(0.01)

# Equivalent to tqdm(range(100), desc="Counting")
```

---

## Nested Progress Bars

```python
from tqdm import tqdm
import time

epochs = 3
batches = 10

for epoch in tqdm(range(epochs), desc="Epochs", position=0):
    for batch in tqdm(range(batches), desc="Batches", position=1, leave=False):
        time.sleep(0.05)

# leave=False on inner bar erases it when done — keeps output clean
```

---

## `tqdm.write` — Print Without Breaking the Bar

Standard `print()` disrupts the progress bar display. Use `tqdm.write` instead.

```python
from tqdm import tqdm

for i in tqdm(range(100)):
    if i % 10 == 0:
        tqdm.write(f"Milestone reached: {i}")   # prints above the bar cleanly
```

---

## Pandas Integration — `progress_apply`

```python
import pandas as pd
from tqdm import tqdm

tqdm.pandas(desc="Processing rows")    # register tqdm with pandas

df = pd.DataFrame({"value": range(10000)})

# progress_apply instead of apply
result = df["value"].progress_apply(lambda x: x ** 2)

# Also works on groupby
result = df.groupby("group")["value"].progress_apply(sum)
```

---

## File Download with Byte Progress

```python
import requests
from tqdm import tqdm

def download_file(url: str, dest: str):
    response = requests.get(url, stream=True)
    total = int(response.headers.get("content-length", 0))

    with open(dest, "wb") as f, tqdm(
        desc=dest,
        total=total,
        unit="B",
        unit_scale=True,
        unit_divisor=1024,
    ) as pbar:
        for chunk in response.iter_content(chunk_size=8192):
            f.write(chunk)
            pbar.update(len(chunk))

download_file("https://example.com/large-file.zip", "file.zip")
# file.zip:  73%|████████████     | 73.1M/100M [00:07<00:02, 10.4MB/s]
```

---

## Async Progress with `tqdm.asyncio`

```python
import asyncio
from tqdm.asyncio import tqdm_asyncio, tqdm

# Wrap asyncio.gather with progress
async def main():
    async def task(i):
        await asyncio.sleep(0.1)
        return i

    results = await tqdm_asyncio.gather(
        *[task(i) for i in range(50)],
        desc="Async tasks",
    )
    return results

asyncio.run(main())
```

```python
# Async for loop
from tqdm.asyncio import tqdm as atqdm

async def process():
    async for item in atqdm(async_generator(), total=100, desc="Processing"):
        await handle(item)
```

---

## Multiprocessing with `tqdm`

```python
from multiprocessing import Pool
from tqdm import tqdm

def process(x):
    return x ** 2

if __name__ == "__main__":
    items = range(1000)

    with Pool(4) as pool:
        results = list(tqdm(
            pool.imap(process, items),
            total=len(items),
            desc="Parallel processing",
        ))
```

```python
# With ProcessPoolExecutor
from concurrent.futures import ProcessPoolExecutor
from tqdm import tqdm

with ProcessPoolExecutor(max_workers=4) as executor:
    results = list(tqdm(
        executor.map(process, items),
        total=len(items),
        desc="Futures",
    ))
```

---

## Jupyter Notebook — `tqdm.notebook`

```python
# Auto-detects environment
from tqdm.auto import tqdm         # recommended — works in both terminal and notebook

for i in tqdm(range(100)):
    ...

# Explicit notebook version (HTML progress bar)
from tqdm.notebook import tqdm

for i in tqdm(range(100), desc="Training"):
    ...
```

> [!tip] Use `tqdm.auto` for portable code `tqdm.auto` automatically uses the notebook version in Jupyter and the terminal version in scripts — write once, run anywhere.

---

## `disable` — Silence in Production

```python
import os
from tqdm import tqdm

# Disable when not in a TTY (CI, Docker logs, etc.)
IS_TTY = os.isatty(1)

for item in tqdm(items, disable=not IS_TTY):
    process(item)

# Or based on log level / env var
VERBOSE = os.getenv("VERBOSE", "false").lower() == "true"
for item in tqdm(items, disable=not VERBOSE):
    process(item)
```

---

## Custom Bar Format

```python
from tqdm import tqdm

# Full custom format string
bar_format = "{desc}: {percentage:3.0f}%|{bar}| {n_fmt}/{total_fmt} [{elapsed}<{remaining}, {rate_fmt}]"

for item in tqdm(items, bar_format=bar_format, desc="Custom"):
    process(item)

# Minimal — just a spinner with count
for item in tqdm(items, bar_format="{desc}: {n_fmt} [{elapsed}]", desc="Processing"):
    process(item)
```

### Bar Format Variables

|Variable|Description|
|---|---|
|`{desc}`|Description string|
|`{percentage}`|Completion percentage (float)|
|`{bar}`|The progress bar itself|
|`{n}`|Current count (int)|
|`{n_fmt}`|Current count (formatted)|
|`{total}`|Total count (int)|
|`{total_fmt}`|Total count (formatted)|
|`{elapsed}`|Elapsed time (formatted)|
|`{remaining}`|Estimated remaining time|
|`{rate}`|Iterations per second (float)|
|`{rate_fmt}`|Rate (formatted with unit)|
|`{unit}`|Unit label|
|`{postfix}`|Postfix dict|

---

## Writing to File / Logger

```python
import logging
from tqdm import tqdm

logger = logging.getLogger(__name__)

# Redirect tqdm output to a file
with open("progress.log", "w") as f:
    for item in tqdm(items, file=f, desc="Processing"):
        process(item)

# Redirect to logger
import io

class TqdmToLogger(io.StringIO):
    def __init__(self, logger, level=logging.INFO):
        super().__init__()
        self.logger = logger
        self.level  = level

    def write(self, buf):
        buf = buf.strip()
        if buf:
            self.logger.log(self.level, buf)

    def flush(self): pass

with tqdm(items, file=TqdmToLogger(logger), desc="Processing", mininterval=5) as pbar:
    for item in pbar:
        process(item)
```

---

## FastAPI / Long-Running Tasks

Track background task progress — store in memory or Redis, poll via API.

```python
from fastapi import FastAPI, BackgroundTasks
from tqdm import tqdm
import asyncio

app = FastAPI()
job_progress: dict[str, dict] = {}

def run_job(job_id: str, items: list):
    job_progress[job_id] = {"total": len(items), "done": 0, "status": "running"}
    with tqdm(total=len(items), desc=f"Job {job_id}") as pbar:
        for item in items:
            process(item)
            job_progress[job_id]["done"] += 1
            pbar.update(1)
    job_progress[job_id]["status"] = "done"

@app.post("/jobs")
async def start_job(background_tasks: BackgroundTasks):
    import uuid
    job_id = str(uuid.uuid4())
    items  = list(range(100))
    background_tasks.add_task(run_job, job_id, items)
    return {"job_id": job_id}

@app.get("/jobs/{job_id}")
async def job_status(job_id: str):
    progress = job_progress.get(job_id)
    if not progress:
        return {"error": "Job not found"}
    pct = progress["done"] / progress["total"] * 100
    return {**progress, "percent": round(pct, 1)}
```

---

## Quick Reference

|Task|Code|
|---|---|
|Basic loop|`for x in tqdm(iterable)`|
|Range|`trange(n)`|
|Label|`tqdm(iterable, desc="Label")`|
|Unit|`tqdm(iterable, unit="file")`|
|Byte progress|`unit="B", unit_scale=True, unit_divisor=1024`|
|Manual update|`pbar.update(n)`|
|Set postfix|`pbar.set_postfix(loss=0.01)`|
|Set description|`pbar.set_description("Epoch 5")`|
|Print safely|`tqdm.write("message")`|
|Nested bars|`position=0/1, leave=False` on inner|
|Pandas|`tqdm.pandas(); df.col.progress_apply(fn)`|
|Async gather|`tqdm_asyncio.gather(*coros)`|
|Multiprocessing|`tqdm(pool.imap(fn, items), total=n)`|
|Notebook|`from tqdm.auto import tqdm`|
|Silence|`tqdm(iterable, disable=True)`|
|Custom format|`bar_format="{desc}: {percentage:.0f}%|

---

## Tags

#python #tqdm #progress-bar #cli #data-pipeline #backend #scripting