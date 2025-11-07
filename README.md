# execr

See [source repo](https://github.com/Bing-su/execr). Readme generated using ChatGPT-5

---

## High‑Level Purpose

This module provides a very small execution harness for running arbitrary Python code inside a WebAssembly (WASM) build of CPython (python-3.12.0.wasm) via the Wasmtime runtime and WASI (WebAssembly System Interface). In other words, it implements:

- Sandboxed Python code execution
- Ephemeral filesystem + controlled stdio
- Optional resource accounting using Wasmtime "fuel"
- Basic runtime metrics (memory pages, data length)
- A simple function `execute(...)` returning a `Result` object (stdout, stderr, fuel usage, memory stats)

This is useful for:
- Secure (or more isolated) code evaluation
- Educational environments or code runners
- Embedding Python script execution in larger services where native interpreter isolation is desired
- Experimentation with WASM resource limiting via fuel

---

## Key Components

### 1. Data Classes

1. RuntimeConfig
   - use_fuel: whether to enable fuel consumption tracking in Wasmtime (a coarse resource limiter).
   - fuel: initial fuel amount given to the Store if use_fuel is True (acts like an instruction budget).
   - mount_fs: path to a directory exposed to the WASM instance as its root (preopened as "/").
   - stdio: directory for stdin, stdout, stderr files.

   Defaults use `mkdtemp()` so if you call `_execute` directly (not recommended publicly) those temp dirs are created and NOT auto-cleaned since there's no surrounding TemporaryDirectory. The public `execute()` manages cleanup properly.

2. Result
   - stdout / stderr: text captured from execution.
   - fuel_consumed: difference between initial fuel and remaining (only if use_fuel).
   - memory: number of memory pages (each page = 64 KiB).
   - data_len: size of accessible linear memory (bytes). (In Wasmtime, `memory.data_len(store)` gives the current length in bytes; `memory.size(store)` returns pages.)

### 2. Cached Binary Loader

```python
@cache
def binary() -> bytes:
    return files("execr").joinpath("files").joinpath("python-3.12.0.wasm").read_bytes()
```

- Uses `importlib.resources.files` to load an embedded WASM CPython binary.
- `@cache` means it is loaded and read once per process, improving performance for multiple executions.

### 3. Internal Execution Function `_execute(...)`

Responsibilities:
- Prepare Wasmtime `Config` (enabling fuel if requested).
- Build `Engine`, `Linker`, and load the CPython `Module`.
- Set up WASI config:
  - Mount `cfg.mount_fs` as "/".
  - Optionally mount an external package path from `EXECR_PYTHONPATH` into "/__pypackages__" and set PYTHONPATH environment.
- Generate a random filename for the user’s code and write it into the mounted filesystem.
- Prepare stdio redirection by writing stdin into a file and pointing WASI to the file paths.
- Construct argv: `["python", <random_script>.py]` plus optional args.
- Instantiate module and optionally set fuel.
- Retrieve the `_start` exported function (entrypoint of WASI Python process) and call it.
- Suppresses any exception during `_start` (meaning you rely solely on stderr output to discover runtime errors).
- Reads back stdout/stderr files and collects memory and fuel metrics.
- Returns a Result.

Important design points:
- Exceptions inside Wasmtime invocation are swallowed; this keeps the API simple but hides engine-level issues (e.g., trap vs. Python-level tracebacks). Tracebacks still appear in stderr if produced by Python itself.
- No exit code is returned (WASI processes can expose exit status); adding that would improve semantics.

### 4. Public Convenience Wrapper `execute(...)`

- Creates a temporary directory (`tmp`) and inside it two subdirectories: `mount_fs` and `stdio` (unless provided explicitly).
- Ensures those directories exist.
- Constructs a `RuntimeConfig` pointing to those paths.
- Delegates to `_execute`.
- On exit, the temp structure is cleaned up automatically.
- This is the function you should call; it produces no residue on disk (unless you pass your own mount paths).

---

## Fuel Usage Explained

Wasmtime’s fuel mechanism is an instruction metering feature to limit or account for execution. When `use_fuel=True`:

1. `store.set_fuel(cfg.fuel)` assigns the budget.
2. After execution, `store.get_fuel()` returns remaining fuel.
3. `fuel_consumed = initial - remaining` is returned.

If the fuel is exhausted before normal termination, Wasmtime would trap. Because exceptions are suppressed, you would likely see truncated output and maybe an empty stderr unless Python managed to flush before the trap. You may want to stop suppressing exceptions or at least log them.

---

## Memory Metrics

- `memory.size(store)` returns number of pages (1 page = 64 KiB).
  To get bytes: `memory.size(store) * 64 * 1024`.
- `memory.data_len(store)` returns current data length in bytes (which should match pages * 64 KiB unless grown dynamically).

These give you a coarse indicator of memory footprint. Not a full heap analysis.

---

## Python Package Support (`EXECR_PYTHONPATH`)

If the environment variable `EXECR_PYTHONPATH` is set:
- That directory is mounted inside the WASM FS at `/__pypackages__`.
- `PYTHONPATH` env var in WASI is set to point to that directory.

Use case: Provide additional libraries to the sandboxed interpreter.

Example:
```bash
export EXECR_PYTHONPATH=/path/to/embedded/site-packages
```

---

## Example Usage

### Basic Execution
```python
from execr.executor.core import execute

code = "print('Hello from WASM Python!')"
result = execute(code)

print("STDOUT:", result.stdout)
print("STDERR:", result.stderr)
print("Fuel consumed:", result.fuel_consumed)
print("Memory pages:", result.memory)
print("Approx bytes:", result.memory * 64 * 1024)
```

### With Stdin and Args
```python
code = """
import sys
data = sys.stdin.read()
print("Args:", sys.argv[1:])
print("Data len:", len(data))
"""

result = execute(
    code=code,
    stdin="sample input",
    args=["first", "second"]
)

print(result.stdout)
```

### With Fuel Accounting
```python
busy_code = "sum(range(10_000_000))"
result = execute(busy_code, use_fuel=True, fuel=50_000_000)
print("Fuel consumed:", result.fuel_consumed)
```

### Reusing a Custom Mount Filesystem
If you want to inspect or persist files created by the executed code:
```python
import tempfile, pathlib
mount_dir = tempfile.mkdtemp()
stdio_dir = tempfile.mkdtemp()

code = "open('output.txt', 'w').write('Generated file')"

res = execute(code, mount_fs=mount_dir, stdio=stdio_dir)

print("Result stdout:", res.stdout)
print("Produced files:", list(pathlib.Path(mount_dir).iterdir()))
```

---

## Potential Improvements / Suggestions

1. Error Transparency:
   - Remove `with suppress(Exception):` or at least capture the exception and include it in `stderr` or an additional field.

2. Exit Code:
   - WASI processes have exit codes; exposing them would help distinguish success vs failure.

3. Resource Controls:
   - Add optional memory limiting (Wasmtime supports memory limits via imports or configuration).
   - Timeouts (wrap call in a host-side timer and force abort if exceeded).

4. Security Considerations:
   - WASM/WASI improves isolation, but mounting user-provided directories could allow them to affect host filesystem if not carefully controlled.
   - Ensure directories are not symlinked into sensitive areas.
   - Validate `EXECR_PYTHONPATH` usage (read-only mount?).

5. Deterministic Environment:
   - Provide interface to set additional env vars beyond PYTHONPATH.
   - Make locale/timezone deterministic if needed for reproducibility.

6. Performance:
   - Reuse the Wasmtime `Engine` (currently each call creates a new `Linker(Engine(...))`). You could cache the engine and linker since the module also is static.

7. API Expansion:
   - Provide `compile(code)` to precompile Python before execution (maybe not trivial with CPython WASM).
   - Batch execution (run multiple scripts in same interpreter / store) to amortize startup cost.

8. Documentation:
   - Expand README with:
     - Purpose
     - Installation
     - Supported Python version (3.12.0)
     - Sandboxing disclaimers
     - Fuel and memory explanation
     - Examples (like those above)

9. Testing:
   - Current test only exercises `binary()`. Add tests for:
     - Basic “hello world”.
     - Fuel accounting.
     - Arg passing.
     - Stdin consumption.
     - Handling of syntax errors (stderr capture).

10. Clean Up Behavior:
    - When `_execute()` is used directly with default RuntimeConfig, temp dirs created by `mkdtemp()` may remain. Encourage using `execute()` or implement explicit cleanup in `RuntimeConfig` context manager.

---

## Edge Cases to Be Aware Of

- Very large stdout/stderr: reading whole files into memory; may need streaming or size limits.
- Fuel exhaustion causing silent trap (since exceptions suppressed).
- Non-UTF-8 output: files are read assuming UTF-8 encoding.
- Concurrency: `@cache` for `binary()` is thread-safe under CPython, but multiple simultaneous executions could create race conditions writing the code file if mount_fs reused externally. Use unique mount dirs per run or ensure locking.

---

## Conceptual Flow Diagram (Textual)

1. User calls `execute(code, ...)`.
2. Temp directory created -> subdirs: mount_fs, stdio.
3. RuntimeConfig built.
4. `_execute`:
   - Load WASM CPython (cached).
   - Initialize Wasmtime engine/linker.
   - Prepare WASI (mount dirs, env, stdio redirection).
   - Write code → random file in mount_fs.
   - Write stdin → stdio/stdin.txt.
   - Instantiate module.
   - Optional fuel set.
   - Call `_start` (runs `python <file>.py`).
   - Collect stdout/stderr text.
   - Compute fuel consumed, memory stats.
   - Return Result.
5. Temp directory cleaned automatically.

---

## Summary

This module is a lightweight abstraction around Wasmtime + WASI CPython execution, optimized for simplicity: write code to file, run it, collect outputs and resource usage. To use it effectively:

- Prefer `execute` unless you need finer control.
- Use `use_fuel` to measure or restrict instruction count.
- Provide `mount_fs` if you need persistent artifacts.
- Watch out for suppressed exceptions; rely on stderr for Python errors.
- Enhance documentation and testing if you plan to expand the project.

If you’d like, I can draft an expanded README section incorporating this explanation—just ask.

Let me know if you want deeper dives (e.g., how to embed additional packages, security hardening, or benchmarking).
