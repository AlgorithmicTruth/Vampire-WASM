Vampire -- WebAssembly Build
============================

This documents how to compile Vampire to WebAssembly and run the
browser-based theorem-proving webapp locally.


Prerequisites
-------------

* **Emscripten SDK (emsdk)** -- install once:

  ```sh
  git clone https://github.com/emscripten-core/emsdk.git ~/emsdk
  cd ~/emsdk
  ./emsdk install latest
  ./emsdk activate latest
  ```

* **CMake** (>= 3.14):

  ```sh
  brew install cmake    # macOS
  ```

* **Git** (submodules are fetched automatically by CMake)


Building
--------

```sh
source ~/emsdk/emsdk_env.sh

# Initialize submodules (CaDiCaL, VIRAS)
git submodule update --init

# Configure
mkdir build_wasm && cd build_wasm
emcmake cmake .. \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_SHARED_LIBS=OFF \
  -DCHECK_LEAKS=OFF \
  -DTIME_PROFILING=OFF \
  -DCMAKE_CXX_FLAGS="-fexceptions" \
  -DCMAKE_EXE_LINKER_FLAGS="-fexceptions -sNO_DISABLE_EXCEPTION_CATCHING -sALLOW_MEMORY_GROWTH=1 -sSTACK_SIZE=8388608"

# Build
emmake make -j4 vampire

# Relink with MODULARIZE for Web Worker usage
em++ -O2 -fexceptions \
  CMakeFiles/vampire.dir/vampire.cpp.o \
  $(find CMakeFiles/common.dir -name '*.o' | sort | tr '\n' ' ') \
  -fexceptions -sNO_DISABLE_EXCEPTION_CATCHING \
  -sALLOW_MEMORY_GROWTH=1 -sSTACK_SIZE=8388608 -sMODULARIZE=1 \
  -sEXPORT_NAME='VampireModule' -sINVOKE_RUN=0 -sEXIT_RUNTIME=0 \
  -sFORCE_FILESYSTEM=1 -sEXPORTED_RUNTIME_METHODS='["callMain","FS"]' \
  -sENVIRONMENT='worker,node' \
  -o vampire.js
```

This produces two files in `build_wasm/`:

| File           | Size  | Description                          |
|----------------|-------|--------------------------------------|
| `vampire.js`   | ~94K  | Emscripten glue (module loader)      |
| `vampire.wasm` | ~8.3M | Compiled Vampire binary              |

Copy these into the webapp directory:

```sh
cp vampire.js vampire.wasm ../webapps/vampire/
```

The WASM binary is larger than E Prover (~2.4M) because Vampire is
C++ and bundles CaDiCaL and MiniSat SAT solvers.

Z3 is not included in the WASM build (optional dependency, disabled
automatically when not found).


Serving the webapp
------------------

Any static HTTP server will work.  The simplest option:

```sh
cd webapps/vampire
python3 -m http.server 8787
```

Then open http://localhost:8787 in a browser.

The webapp requires a server because browsers block `fetch()` and
`Worker` from `file://` URLs.  Any of these alternatives also work:

```sh
# Node.js
npx serve webapps/vampire

# Ruby
cd webapps/vampire && ruby -run -ehttpd . -p8787

# PHP
cd webapps/vampire && php -S localhost:8787
```


Using the webapp
----------------

The interface mirrors the Prover9+Mace4 webapp at
https://prover9.org/webapps/combo/ and follows the same conventions:

1. Enter TPTP (or SMT-LIB 2) formulas in the **Input** pane.
2. Click **Run** (or press Ctrl+Enter).
3. The prover runs in a Web Worker -- output streams live to the
   **Output** pane.
4. Click **Stop** to terminate a running search.

The prover runs with `--mode vampire --proof tptp -t 300` by default,
producing TPTP-format proofs with a 5-minute timeout.

Example problems are available in the **Examples** dropdown and are
served from `webapps/vampire/examples/`.


Testing with Node.js
--------------------

The WASM module can also be used directly from Node.js (without a
browser):

```js
async function run() {
  const VampireModule = require('./webapps/vampire/vampire.js');
  const Module = await VampireModule({
    noInitialRun: true,
    print: function(text) { console.log(text); },
    printErr: function(text) { console.error(text); }
  });

  Module.FS.writeFile('/input.p', [
    'fof(a, axiom, ![X]: mult(X, e) = X).',
    'fof(b, axiom, ![X]: mult(X, inv(X)) = e).',
    'fof(c, axiom, ![X,Y,Z]: mult(mult(X,Y),Z) = mult(X,mult(Y,Z))).',
    'fof(goal, conjecture, ![X]: mult(e, X) = X).'
  ].join('\n'));

  try {
    var code = Module.callMain([
      '--mode', 'vampire', '--input_syntax', 'tptp',
      '--proof', 'tptp', '-t', '30', '/input.p'
    ]);
    console.log('Exit code:', code);
  } catch(err) {
    if (typeof err === 'object' && 'status' in err)
      console.log('Exit code:', err.status);
  }
}
run();
```

Run with Node.js >= 18:

```sh
node test_vampire.js
```

Note: Vampire's module factory is `async` (unlike E Prover's), so
you must `await` it or use `.then()`.


Worker API
----------

The webapp uses a Web Worker (`vampire-worker.js`) that speaks the
same protocol as the Prover9 and E Prover webapp workers:

**Main thread -> Worker:**

```js
worker.postMessage({
  type: "run",
  input: "fof(...)",        // problem text
  filename: "/input.p",     // virtual filesystem path
  args: ["--mode", "vampire", "--input_syntax", "tptp", "/input.p"]
});
```

**Worker -> Main thread:**

```js
{ type: "ready" }                          // WASM loaded
{ type: "stdout", line: "..." }            // stdout line
{ type: "stderr", line: "..." }            // stderr line
{ type: "done", exitCode: 0 }             // finished
{ type: "error", message: "..." }          // fatal error
```

To stop a running search, terminate and respawn the worker.


Source modifications
--------------------

All changes use `#ifdef __EMSCRIPTEN__` guards so the native build
is unaffected.  The main adaptations:

* **Timer thread removed** -- `Lib/Timer.cpp`: the background timer
  thread (`std::thread`) is replaced with a no-op stub.  Elapsed time
  is still tracked via `std::chrono::steady_clock`.  Time limits are
  not enforced by the prover itself; use the Web Worker termination
  from JavaScript instead.

* **Portfolio mode disabled** -- `CASC/PortfolioMode.cpp`:
  `PortfolioMode::perform()` returns false immediately.  Portfolio
  mode requires `fork()` which is unavailable in WASM.  Use
  `--mode vampire` for single-process proving.

* **Signal handlers disabled** -- `Lib/System.cpp`: signal
  registration is skipped (no `SIGTERM`, `SIGXCPU`, etc.).

* **Process management stubbed** -- `Lib/Sys/Multiprocessing.cpp`:
  `fork()`, `wait()`, `kill()` all stubbed.

* **Resource limits disabled** -- `Lib/Allocator.cpp`: `setrlimit()`
  and `getrusage()` are excluded (WASM is sandboxed anyway).

* **64-bit pointer assertions relaxed** --
  `Kernel/TermOrderingDiagram.hpp`: `static_assert(sizeof(void*) ==
  sizeof(uint64_t))` guarded with `#if UINTPTR_MAX == UINT64_MAX`
  for 32-bit WASM compatibility.

* **Exit handling** -- `Lib/System.hpp`: `std::_Exit()` replaced with
  `exit()` under Emscripten so that Emscripten's exit handling works
  correctly.

* **C++ exceptions enabled** -- Vampire uses C++ exceptions
  internally.  The build uses `-fexceptions` and
  `-sNO_DISABLE_EXCEPTION_CATCHING` (increases WASM size but is
  required for correctness).

The native build (`cmake .. && make`) is not affected by any of
these changes.


Exit codes
----------

| Code | Meaning                        |
|------|--------------------------------|
| 0    | Proof found (Theorem)          |
| 1    | No proof (search exhausted)    |


Differences from E Prover WASM build
-------------------------------------

| Aspect            | E Prover          | Vampire              |
|-------------------|-------------------|----------------------|
| Language          | C (gnu99)         | C++ (C++17)          |
| Build system      | Custom Makefile   | CMake + manual relink|
| WASM size         | ~2.4 MB           | ~8.3 MB              |
| Exceptions        | Not needed        | Required (`-fexceptions`) |
| Pointer size      | 32-bit OK         | Needed assert relaxation |
| Timer             | No timer thread   | Timer thread stubbed |
| SAT solvers       | PicoSAT           | MiniSat + CaDiCaL   |
| Auto mode         | `--auto`          | `--mode vampire`     |
