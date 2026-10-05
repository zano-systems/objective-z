<!-- SPDX-License-Identifier: Apache-2.0 -->

# Objective-Z user guide

This guide covers setup, Zephyr integration, resource configuration, and everyday
commands. Start with the [README](../README.md#quick-start) for the first sample.
For source-level ownership examples, see the [practical ARC guide](ARC_GUIDE.md).
The [dialect ledger](OBJECTIVE_C_DIALECT.md) and [ARC conformance ledger](ARC.md)
remain the contracts for supported semantics; this guide does not replace them.

## Prerequisites

- A Zephyr development environment, including Python 3, west, CMake, Ninja, and a
  host C compiler. Follow [Zephyr Getting Started](https://docs.zephyrproject.org/latest/develop/getting_started/index.html)
  for host dependencies and an active Python virtual environment with west installed.
- **Zephyr v4.4.2** and **Zephyr SDK 1.0.1** are the tested versions. The repository's
  [west manifest](../west.yml) pins Zephyr and imports its required CMSIS modules.
- **The SDK's LLVM component**, a separate opt-in download. Its **Clang 19** is
  tested for the JSON AST used for semantic and ownership information:

  ```sh
  west sdk install --llvm --version 1.0.1 -b ~/.local
  ```

  Run this from an initialized west workspace. Alternatively, use the SDK's
  `setup.sh -l` to install LLVM. GNU target toolchains alone do not supply Clang.
- **Rust and Cargo** to build the host-side `oz2c` transpiler. Install a Rust
  toolchain through [rustup](https://rustup.rs/) if neither is available.

After fetching the workspace, install Zephyr's Python requirements in the active
environment and run `west zephyr-export`; the [quick start](../README.md#quick-start)
shows those steps. Keep Rust and Cargo on `PATH` when configuring a build.

`objz_find_clang()` prefers the SDK's LLVM and warns on an untested compiler.
Pass `-DOBJZ_REQUIRE_TESTED_CLANG=ON` to CMake after west's `--` to make that
warning an error, as CI does:

```sh
west build -p always -b mps2/an385 samples/hello_world -- -DOBJZ_REQUIRE_TESTED_CLANG=ON
```

Apple Clang and Homebrew LLVM can work with a warning. RISC-V requires LLVM Clang,
not Apple Clang, which lacks a RISC-V backend. For host test harnesses, set
`OZ_CLANG` to the SDK's `llvm/bin/clang` when needed. The compiler search order
also checks `ZEPHYR_SDK_INSTALL_DIR`, SDKs under `~/.local`, Homebrew, and `PATH`.

## Using in your project

### 1. Add Objective-Z to your west manifest

Merge the Objective-Z project entry into your application's existing manifest.
This minimal example uses the tested Zephyr revision:

```yaml
manifest:
  remotes:
    - name: zephyrproject-rtos
      url-base: https://github.com/zephyrproject-rtos

  projects:
    - name: zephyr
      remote: zephyrproject-rtos
      revision: v4.4.2
      import:
        name-allowlist:
          - cmsis
          - cmsis_6

    - name: objective-z
      url: https://github.com/rodrigopex/objective-z
      revision: main
      path: objective-z

  self:
    path: my_app
```

Run `west update` from the workspace to fetch the module. Pin Objective-Z to a
reviewed commit for reproducible application builds rather than following `main`.

### 2. Directory layout

```text
my_app/
├── west.yml
├── CMakeLists.txt
├── prj.conf
└── src/
    └── main.m
```

### 3. CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.20.0)

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(my_app)

# Transpile .m sources to C (ARC always enabled)
objz_transpile_sources(app src/main.m)
```

West discovers the module through its manifest. If Objective-Z is instead a
separate checkout outside that manifest, add its path to `ZEPHYR_EXTRA_MODULES`
before `find_package(Zephyr ...)`, as the repository's samples do.

### 4. prj.conf

```ini
CONFIG_OBJZ=y
```

Import `<Foundation/Foundation.h>` to use the Foundation classes. Slab storage is
generated for classes with slab-allocation sites or explicit nonzero pool overrides,
not simply for every declared class. See the [resource model](#resource-model)
when budgeting capacity; unused or heap-only classes need not reserve a slab.
Collections additionally require `CONFIG_SYS_MEM_BLOCKS=y`.

### 5. Write your .m file

```objc
#import <Foundation/Foundation.h>

@interface Sensor: OZObject {
	int _value;
}
- (void)setValue:(int)v;
- (int)value;
@end

@implementation Sensor
- (void)setValue:(int)v
{
	_value = v;
}

- (int)value
{
	return _value;
}

- (void)dealloc
{
	OZLog("Sensor dealloc (value=%d)", _value);
}
@end

int main(void)
{
	Sensor *s = [[Sensor alloc] init];
	[s setValue:42];
	OZLog("value=%d", [s value]);
	/* ARC releases s here, invoking the cleanup hook. */
	return 0;
}
```

### 6. Build

From the application directory:

```sh
west build -p always -b mps2/an385 .
west build -t run
```

### CMake API

```text
objz_transpile_sources(<target> <source1.m> [source2.m ...]
    [ROOT_CLASS <name>]
    [POOL_SIZES <Class1=N,Class2=M,...>]
    [INCLUDE_DIRS <dir1> [dir2 ...]]
)
```

| Parameter | Default | Description |
|---|---|---|
| `ROOT_CLASS` | `OZObject` | Root class name for hierarchy resolution |
| `POOL_SIZES` | inferred | Override slab pool sizes per class |
| `INCLUDE_DIRS` | none | Additional source/header search directories |

## Resource model

### Default pool sizing

On Zephyr, ordinary object allocation uses fixed-capacity, per-class `k_mem_slab`
pools. Their storage is reserved at build time and does not grow when a pool fills.
This bounds the storage reserved for those objects; it does **not** prove that every
allocation succeeds, that the whole application is heap-free, or that its execution
meets a timing deadline.

`oz2c` derives default capacities from source allocation sites, including `alloc`,
inherited `new`, and allocations introduced by boxed numbers and collection literals.
String literals are static, immortal objects rather than slab allocations.
For allocations returned from a body, it applies call-site multiplicity where it
can resolve the callers (class-method sends and plain C calls). Instance-method
callers are not resolved by the sizing pass. An allocation site is not multiplied
by its runtime execution count.

**These defaults are not a proof of the maximum number of simultaneously live
objects.** Repeated calls, retained results, concurrent invocations, and callers
outside the analyzed source can require more slots. Supported loop-local lifetimes
can reuse slots; checks reject some accumulating or overlapping allocation shapes,
but they are not a general peak-liveness analysis. See the
[sizing implementation](../tools/oz2c/src/pools.rs) and
[loop-allocation tests](../tools/oz2c/tests/loop_allocation_bounds.rs) for the exact rules.

### Explicit capacity

Set an explicit capacity when the default does not cover the application's maximum
live instances, including temporary overlap during replacement and concurrent use.
For example, a source directive reserves eight Sensor slots:

```objc
/* oz-pool: Sensor=8 */
```

The same budget can be supplied through `objz_transpile_sources(... POOL_SIZES "Sensor=8")`
or the `oz2c` CLI's `--pool-sizes Sensor=8`. Explicit values replace the inferred
capacity, even when smaller. CMake/CLI values take precedence over source directives
for the classes they name. Eight is an example, not a recommended capacity.

Collection literals also reserve element buffers from a shared item pool, separate
from the collection objects' slabs. Its size is in object-pointer slots, not bytes;
the default counts literal element slots, not runtime executions.
Use `/* oz-item-pool: 32 */` or `--item-pool-size 32` to override it; the CLI value wins.
Zephyr collection use requires `CONFIG_SYS_MEM_BLOCKS=y`. The contiguous element
buffers need enough free space for each request, not just enough total slots across
fragmented free regions.

### Exhaustion and failure handling

Slab allocation uses `K_NO_WAIT`: with no free slot, `+alloc` returns `nil` rather
than waiting, growing the pool, or falling back to a heap. Handle allocation failure
before initialization or use; do not assume every factory propagates it safely.
The generated collection-literal builders return `nil` and free the newly allocated
collection object if its item-buffer allocation fails.

For diagnosis, compile the generated C with `-DOZ_TRAP_POOL_EXHAUSTION` to enable
an assertion naming the exhausted class pool; on Zephyr, also enable `CONFIG_ASSERT=y`.
This is opt-in, not the normal failure contract, and is not an item-pool exhaustion
instrument. The [pool behavior tests](../tools/oz2c/tests/behavior_pools.rs)
exercise capacity, failure, and slot reuse.

### Optional heap allocation

`CONFIG_OBJZ_HEAP` defaults to `n`. Enabling it makes `+dynamicAlloc` (system heap)
and `+dynamicAllocWithHeap:` (an explicit heap, or the system heap when passed `nil`)
available; the corresponding CLI flag is `--heap-support`. These are explicit
allocation paths, not fallback behavior for a full slab, and need their own capacity
and failure handling. Disabling them does not forbid ordinary C calls such as
`malloc` or control allocations made by Zephyr APIs.

## Configuration

`CONFIG_OBJZ` enables the transpiler pipeline and auto-selects `STATIC_INIT_GNU`.
Selected options are listed below; defaults are what a plain `CONFIG_OBJZ=y` gives
you. Pool capacities are configured separately in the [resource model](#resource-model).
The complete declarations and dependencies are in [Kconfig](../Kconfig).

| Option | Default | Effect |
|---|---|---|
| `CONFIG_OBJZ_HEAP` | `n` | Explicit heap allocation and the heap-aware free path |
| `CONFIG_OBJZ_NIL_SAFE_SENDS` | `y` | Nil-receiver guards for direct and protocol instance sends |
| `CONFIG_OBJZ_INTROSPECTION` | `y` | `-isKindOfClass:` and `-conformsToProtocol:` |
| `CONFIG_OBJZ_REFLECTION` | `y` | `@selector`, `SEL`, `-respondsToSelector:`, `-performSelector:` |
| `CONFIG_OBJZ_DEFAULT_DESCRIPTION` | `y` | Inherited `<ClassName: 0xADDRESS>` descriptions for `%@` |
| `CONFIG_OBJZ_DEBUG_LINES` | `y` under `CONFIG_DEBUG` | `#line` directives back to the `.m` |
| `CONFIG_OBJZ_LOG_BUFFER_SIZE` | `128`, range `32` to `1024` | Bytes reserved on the calling thread's stack for one formatted log line |

### Logging

`OZLog`'s formatting buffer is an automatic array, so each calling thread needs
enough stack headroom. Check that headroom with `kernel thread stacks` and adjust
the thread's stack before raising the buffer size. The 1024-byte ceiling equals
the default `CONFIG_MAIN_STACK_SIZE` on `mps2/an385`; it is not a safe stack budget
for the whole call.

An undersized buffer truncates the line without overflowing or reporting an error.
A `%@` at the boundary can be truncated mid-description. Collection descriptions
include their elements, so their length depends on the data. At a buffer size of
32, `transpiled_literals` historically printed `a = 10, b = 25002031, a + b = 2`
instead of the full sum ending in `25002041` (#420).

### Introspection and reflection

Tables are generated for constructs the program uses. Leaving these options on
does not emit unused tables. Turning either off refuses the corresponding constructs
with located diagnostics naming the option. Basic class identity (`[Foo class]`,
`[obj class]`, `-isMemberOfClass:`) remains available.

### Debugging Objective-C source

`CONFIG_OBJZ_DEBUG_LINES` attributes authored method bodies, plain C function
bodies, and hoisted blocks to their `.m` files. Synthesized support keeps generated-C
locations. Debugger breakpoints, stepping, backtraces, `addr2line`, and compiler
diagnostics can therefore name the authored source (#305).

The directives change source attribution, not instructions; `__FILE__` strings can
change, including assertion messages. They also make generated C larger and harder
to read because they carry absolute paths. A recorded `hello_category` measurement
grew `main.c` from 2658 to 5766 bytes; directives occupied 38% of px-keyboard's
generated tree, dependent on path depth (#395).

The option depends on `CONFIG_DEBUG` and defaults to `y` within it. Set it to `n`
in a debug build to inspect generated C without markers. Enabling `CONFIG_DEBUG`
also changes optimization and assertions: that is not the same image as a release
build. A shipped image must already contain attribution for it to be useful later.

## Architectures

- ARM Cortex-M
- ARM Cortex-A
- RISC-V 32/64-bit (requires LLVM Clang, not Apple Clang)
- x86 32/64-bit

Support depends on the target triples in
[`_objz_get_clang_target_triple()`](../cmake/ObjcClang.cmake), not merely on the
portability of generated C. Clang must validate the target's headers and inline
assembly when producing the AST. [Kconfig](../Kconfig) and the triple table must
agree; [the architecture test](../tools/oz2c/tests/arch_support_is_declared_once.rs)
checks that relationship (#612). The PR workflow builds and runs `hello_category`
on `mps2/an385`, `qemu_riscv32`, and `qemu_x86`.

## Build commands

Run these from the Objective-Z checkout:

```sh
west build -b mps2/an385 samples/hello_world
west build -p always -b mps2/an385 samples/hello_world
west build -t run
west flash
```

The first build is incremental; `-p always` rebuilds from a pristine configuration.
The run and flash commands use the current build directory. Choose a hardware board
before flashing; `mps2/an385` is the QEMU target. Use `-d <directory>` to keep separate
builds, and pass the same directory to subsequent west commands.

For another sample or architecture:

```sh
west build -p always -b nucleo_f429zi samples/arc_demo
west build -p always -b qemu_riscv32 samples/hello_world
```

The primary Rust gate is:

```sh
cargo test --manifest-path tools/oz2c/Cargo.toml
```

### Twister and host tests

Build the transpiler once before parallel sample configurations. Twister build/run
harnesses select eligible configurations for each board:

```sh
cargo build --manifest-path tools/oz2c/Cargo.toml
west twister -T samples/ -p mps2/an385 -c -O build-twister-arm
west twister -T samples/ -p qemu_riscv32 -c -O build-twister-riscv
west twister -T samples/ -p qemu_cortex_a53/qemu_cortex_a53/smp -c -O build-twister-smp
```

Sample configurations differ by board; do not infer their counts from the number
of sample directories. Twister's `--dry-run` reports the selected configurations.
`-c` replaces prior output in the chosen directory; use a distinct `-O` path to
retain a run or avoid another session's output. These output directories are
siblings of `build`, so a pristine rebuild of the default sample does not remove them.

The ztest suites over committed generated C run on `native_sim` on Linux, or an
eligible emulated board such as `mps2/an385`:

```sh
west twister -T tests/zephyr/ -p mps2/an385 -c -O build-twister-zephyr
```

For real hardware, use device testing with the repository's hardware map:

```sh
west twister -T samples/ -p nrf52833dk/nrf52833 --device-testing \
    --hardware-map hardware-map.yaml -c -O build-twister-hardware
```

That command flashes and runs eligible samples on an attached nRF52833DK. The
[benchmark guide](BENCHMARKS.md#reproducing-measurements) covers benchmark device tests.

Host corpora use Python harnesses after building `oz2c`:

```sh
python -m pytest tests/behavior/ -v
python -m pytest tests/adapted/ -v
python -m pytest tests/pal/ -v
python tests/smoke/run.py
```

See [test infrastructure](../tests/README.md) for Python dependencies, compiler
selection, sanitizer options, and test expectations.

### Generated output and cleanup

`west build -d build -t clean` removes compiled build products without deleting
the build configuration. `west build -p always` performs a pristine rebuild.
`cargo clean --manifest-path tools/oz2c/Cargo.toml` removes the host transpiler's
generated artifacts. Neither command removes tracked sources.

Other sample build directories and Twister outputs can retain substantial generated
data. Inspect their paths before removing anything; do not sweep another session's
output. See [working practices](WORKING.md) before running concurrent builds.

## Samples

Each directory under [samples](../samples) demonstrates part of the supported subset:

| Sample | Description |
|---|---|
| `hello_world` | Class and instance methods |
| `hello_category` | Category extensions and source attribution |
| `arc_demo` | ARC lifecycle, scoped cleanup, singletons, threads |
| `mem_demo` | Scope-based memory management |
| `pool_demo` | Slab pools, scoped reclaim, synchronization |
| `transpiled_blocks` | Non-capturing blocks, `__block`, enumeration |
| `transpiled_literals` | Boxed and collection literals |
| `transpiled_generics` | Typed collections |
| `transpiled_led` | LED control |
| `gpio_demo` | GPIO and devicetree |
| `zbus_objc` | zbus publish/subscribe |
| `zbus_service` | Request-response service pattern |
| `class_forward` | Forward class declarations |
| `class_side` | Class methods and inheritance |
| `heap_alloc` | Explicit heap allocation |
| `reflection_demo` | Selectors and reflection |
| `smp_shared` | Shared state on multiple cores |

```sh
west build -p always -b mps2/an385 samples/arc_demo
west build -t run
```

The introductory source is [hello_world/src/main.m](../samples/hello_world/src/main.m).
Its concrete instance and class sends lower to direct C calls; object creation uses
the per-class slab followed by initialization. Clang analyzes the source during the
build; the target C compiler compiles the generated code.

## Foundation and generated code

| Class or API | Purpose |
|---|---|
| `OZObject` | Root class: allocation, initialization, cleanup hook, identity and equality |
| `OZString` | Immutable strings: `cStr`, `length`, equality |
| `OZMutableString` | Mutable strings: append operations |
| `OZArray` | Immutable arrays: count, indexed access, enumeration |
| `OZDictionary` | Immutable dictionaries: count, keyed access, enumeration |
| `OZNumber` | Q31+shift fixed-point values and arithmetic |
| `OZHeap` | Explicit heap allocator |
| `OZSpinLock` | Explicit lock object; `@synchronized` lowering uses a stack-local lock |
| `OZDefer` | Scope-based cleanup |
| `OZLog` | printf-style logging with `%@` object formatting |

`oz2c` emits one `.h`/`.c` pair per origin file, not necessarily one per class,
plus shared `oz2c_dispatch.h` and `oz2c_dispatch.c`. Output is placed under
`oz2c_generated/` in the build directory. It contains struct definitions, method
functions, allocation support, dispatch macros, and required `const` tables.

The [platform abstraction layer](../include/platform) provides `static inline`
wrappers for Zephyr primitives and a host backend for tests. The Zephyr backend
uses slabs, atomics, spinlocks, and logging; the host backend uses malloc-backed
slabs and C11 atomics. Inlining can remove wrapper-call overhead, but the underlying
allocation, atomic, and lock operations still execute.

## Language boundaries

The [README](../README.md#limitations) summarizes the important boundaries. Consult
the [dialect ledger](OBJECTIVE_C_DIALECT.md) for the exact verdict and evidence for
each construct, rather than treating this guide as a second feature matrix.

Additional boundaries to consider when structuring an application:

- `OZNumber` converts to int8/16/32 and float, not int64/double.
- `Class<Protocol>` receivers are refused. Use a concrete class or an
  `id<Protocol>` instance.
- Unrelated classes must agree on a selector's return type; `instancetype` is
  exempt because dispatch collapses it to `void *` (#290).
- `id` is a reserved type name, not an available variable name (#317).
- `OZFN`/`OZM` block contents bypass Clang's type checks. Declare the return type
  explicitly; a signature mismatch reaches the C compiler, not a located
  Objective-C diagnostic (#603).

The [ARC ledger](ARC.md) also records unexamined rules. Full peak-liveness pool
analysis is not a current capability. Known gaps and design directions are not
delivery commitments; use current ledger verdicts when choosing firmware constructs.
