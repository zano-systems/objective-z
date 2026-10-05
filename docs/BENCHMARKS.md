<!-- SPDX-License-Identifier: Apache-2.0 -->

# Benchmarks and historical measurements

This page preserves the measurements previously embedded in the README, separates
them from current feature claims, and describes how to collect new results.

## Provenance and limits

The tables below were carried in
[README.md at `2991867`](https://github.com/rodrigopex/objective-z/blob/29918677f5ae9a8b352c39a5d542bb46404aca98/README.md).
That is the **documentation revision**, not a verified revision of the binary
measured. The original record does not identify every measurement's transpiler
revision, Zephyr/SDK version, or raw output. Those missing facts cannot be recovered
by moving the tables.

The transpiler comparisons report **nRF52833 DK**, ARM Cortex-M4F at **64 MHz**,
using the DWT cycle counter with overhead calibration. Speed and footprint results
use `-O2`; the memory footprint comparison uses `-Os`. The retired-runtime record
is separate and does not establish the same clock or configuration.

**These are historical results, not current `oz2c` performance guarantees.** In
particular, the 266-cycle `@synchronized` result describes an older OZSpinLock
allocation path; current lowering uses a stack-local lock. The object-size records
also disagree on some Foundation sizes. Preserve the distinction between those
records instead of choosing one as today's ABI.

Comparisons describe these benchmark programs, not every application or a language
in general. Allocator choice, enabled features, object representation, and workload
all affect the result. Slab block costs exclude unused reserved capacity and shared
allocator structures. Firmware “total” is an ELF-section sum, not a single storage
region: Flash is text + data; RAM is data + bss.

## Reproducing measurements

Use the [user-guide prerequisites](USER_GUIDE.md#prerequisites). These commands
flash an attached **nRF52833DK**; they are not QEMU timing tests. Run from the
Objective-Z checkout:

```sh
cargo build --manifest-path tools/oz2c/Cargo.toml
west build -p always -b nrf52833dk/nrf52833 -d build-bench-objc benchmarks/objc
west flash -d build-bench-objc
west build -p always -b nrf52833dk/nrf52833 -d build-bench-cpp benchmarks/cpp
west flash -d build-bench-cpp
west build -p always -b nrf52833dk/nrf52833 -d build-bench-mem-c benchmarks/memory/c
west flash -d build-bench-mem-c
west build -p always -b nrf52833dk/nrf52833 -d build-bench-mem-cpp benchmarks/memory/cpp
west flash -d build-bench-mem-cpp
west build -p always -b nrf52833dk/nrf52833 -d build-bench-mem-objc benchmarks/memory/objc
west flash -d build-bench-mem-objc
```

For memory attribution, use west's build targets on each retained build:

```sh
west build -d build-bench-objc -t rom_report
west build -d build-bench-objc -t ram_report
```

The corresponding ELF is under `<build-directory>/zephyr/zephyr.elf`. The
repository also retains an [ELF section collector](../benchmarks/footprint.sh)
used for the historical footprint comparison.

Twister device testing runs the benchmark harnesses with
[hardware-map.yaml](../hardware-map.yaml); it needs a matching attached device:

```sh
west twister -T benchmarks/ --device-testing --hardware-map hardware-map.yaml \
    -c -O build-twister-bench
```

Build sources and configuration live under [benchmarks](../benchmarks).

For a new report, record:

- Objective-Z, benchmark-source, and Zephyr revisions;
- SDK and C compiler versions, board and clock configuration;
- optimization, debug, and Objective-Z feature settings;
- exact commands, raw serial output, and the ELF section report;
- what each measured operation includes, such as allocation, initialization,
  cleanup, or dispatch.

Do not compare a fresh run with these tables as though their configurations were
fully pinned. No benchmark hardware was run as part of this documentation move.

## Historical transpiler comparisons

### Allocation

| Operation | OZ (cycles) | C++ (cycles) |
|---|---:|---:|
| slab alloc + init + release (Base) | 215 | --- |
| slab alloc + init + release (Child) | 217 | --- |
| slab alloc + init + release (GChild) | 218 | --- |
| Value type on stack | --- | 12 |
| new/delete (heap) | --- | 865 |
| unique_ptr create/destroy | --- | 517 |
| placement new + slab + dtor + free | --- | 105 |

The 105-cycle C++ and 215-cycle OZ paths both use slabs. The 865-cycle C++
heap path uses a different allocator; its ratio to OZ is not an isolated
language-overhead comparison.

### Dispatch and callable objects

| Operation | OZ (cycles) | C++ (cycles) |
|---|---:|---:|
| C function pointer (baseline) | 12 | 8 |
| Static / direct call | 12 | 12 |
| Class / static method | 12 | 12 |
| Compile-time protocol dispatch | 12 | --- |
| Vtable / virtual dispatch (depth=0) | 21 | 20 |
| Vtable / virtual dispatch (depth=1) | 29 | 14 |
| Vtable / virtual dispatch (depth=2) | 20 | 14 |
| Block / lambda (non-capturing) | 12 | 12 |
| std::function (int capture) | --- | 16 |
| std::function copy + destroy | --- | 42 |

Known receivers permit direct calls. Protocol runtime selection uses generated
`const` tables; its cost cannot be inferred from the direct-call measurement.

### Object lifecycle

| Operation | OZ (cycles) | C++ (cycles) |
|---|---:|---:|
| alloc + init + release | 218 | --- |
| alloc + init + retain + 2x release | 253 | --- |
| new + delete | --- | 853 |
| placement new + slab | --- | 105 |
| make_unique create/destroy | --- | 503 |

The original allocation and lifecycle sections recorded slightly different heap
and smart-pointer timings. They are retained as separate measurements.

### Reference counting

| Operation | OZ (cycles) | C++ (cycles) |
|---|---:|---:|
| retain / atomic inc | 22 | 7 |
| retain + release pair | 44 | 17 |
| shared_ptr copy | --- | 5 |
| shared_ptr copy + reset | --- | 12 |

The original explanation attributed the difference from raw atomic operations to
null checks and call overhead. That explanation has not been remeasured for current
generated code.

### Properties and synchronization

| Operation | OZ (cycles) | C++ (cycles) |
|---|---:|---:|
| property get (nonatomic) | 12 | 12 |
| property set (nonatomic) | 1 | 2 |
| property get (atomic, k_spinlock) | 10 | 12 |
| property set (atomic, k_spinlock) | 11 | 12 |
| @synchronized / syncNop (k_spinlock) | 266 | 15 |

The last row measured the old RAII object allocation/free path, not today's
stack-local synchronization lowering.

### Foundation and collections

| Operation | OZ (cycles) | C++ (cycles) |
|---|---:|---:|
| Raw int32_t[] sum (10 elems, baseline) | 99 | 81 |
| OZArray objectAtIndex: / access | 12 | 13 |
| String loop + length (10 items) | 483 | 263 |
| String iterator (virtual, length) | 341 | 211 |
| OZDictionary objectForKey: (lookup) | 154 | --- |

### C++ introspection

| Operation | C++ (cycles) |
|---|---:|
| dynamic_cast (hit) | 12 |
| dynamic_cast (miss) | 12 |
| typeid() + name() | 7 |

These are C++ timings only. The old OZ introspection path used legacy C helpers;
the measurements were not retaken against current `-isKindOfClass:` and reflection.
See [the current design](STATUS.md#introspection-and-reflection-226).

### Object sizes: detailed record

| Object | OZ (B) | C++ (B) |
|---|---:|---:|
| Base (metadata + refcount) | 8 | 8 |
| Child (+ 1 int ivar) | 12 | 12 |
| GrandChild (+ 1 int ivar) | 16 | 16 |
| OZString / SimpleString | 20 | 12 |
| OZNumber / --- | 16 | --- |
| OZArray / --- | 20 | --- |
| OZDictionary / --- | 24 | --- |
| shared_ptr / --- | --- | 8 |
| unique_ptr / --- | --- | 4 |
| std::function<int()> / --- | --- | 16 |
| k_spinlock | --- | 1 |

### Object costs: former summary record

This separate summary appeared alongside the detailed record. Its OZNumber size
differs; it is not reconciled into a current claim.

| Metric | C++ | OZ | Notes |
|---|---:|---:|---|
| Base object sizeof | 8 | 8 | Metadata + refcount |
| Slab alloc overhead | n/a | 0 | Block = sizeof |
| Heap alloc overhead | 4 | n/a | C++ sys_heap header |
| shared_ptr control block | 12 | 0 | OZ inline refcount |
| OZNumber / SimpleString | 16 | 12 | OZ Q31+shift vs vptr+data+len |

### Firmware footprint

| Benchmark | Metric | C++ | OZ | Diff |
|---|---|---:|---:|---:|
| Speed (`-O2`) | text | 50,588 | 34,272 | -32% |
| Speed (`-O2`) | data | 312 | 768 | +146% |
| Speed (`-O2`) | bss | 9,861 | 8,605 | -13% |
| Speed (`-O2`) | total | 60,761 | 43,645 | -28% |
| Speed (`-O2`) | Flash | 50,900 | 35,040 | -31% |
| Speed (`-O2`) | RAM | 10,173 | 9,373 | -8% |
| Memory (`-Os`) | text | 22,840 | 21,344 | -7% |
| Memory (`-Os`) | data | 180 | 180 | 0% |
| Memory (`-Os`) | bss | 15,558 | 7,344 | -53% |
| Memory (`-Os`) | total | 38,578 | 28,868 | -25% |
| Memory (`-Os`) | Flash | 23,020 | 21,524 | -6% |
| Memory (`-Os`) | RAM | 15,738 | 7,524 | -52% |

Those percentages belong to these programs and configurations. They do not predict
another application's image size, and they are not current measurements.

## Historical memory comparison

The C/C++ variants used a dedicated 8 KB `sys_heap`; the OZ variant used per-class
`k_mem_slab` pools. This compares allocator configurations as well as object models.

### Object sizes

| Metric | C | C++ | OZ |
|---|---:|---:|---:|
| Base object (sizeof) | 8 B | 8 B | 8 B |
| Child (+ 1 int) | 12 B | 12 B | 12 B |
| GrandChild (+ 2 ints) | 16 B | 16 B | 16 B |
| Dispatch mechanism | 4 B | 4 B | 4 B (enum) |
| Refcount field | 4 B | 4 B | 4 B |

### Single allocation

| Object type | C | C++ | OZ |
|---|---:|---:|---:|
| Base | 16 B | 16 B | 8 B |
| Child | 16 B | 16 B | 12 B |
| GrandChild | 24 B | 24 B | 16 B |

### Bulk allocation (20 objects)

| Object type | C | C++ | OZ |
|---|---:|---:|---:|
| 20x Child | 320 B | 320 B | 240 B |
| 20x GrandChild | 480 B | 480 B | 320 B |
| Per GrandChild avg | 24 B | 24 B | 16 B |

The original “33% less per object” conclusion compares the GrandChild blocks
(16 vs 24 B), not total reserved memory or unused pool capacity.

### Smart pointers and reference counting

| Metric | C++ | OZ |
|---|---:|---:|
| sizeof(unique_ptr) | 4 B | - |
| sizeof(shared_ptr) | 8 B | - |
| make_unique heap cost | 16 B | - |
| make_shared heap cost | 24 B (+ ctrl) | - |
| shared_ptr(new T) heap cost | 40 B (2 allocs) | - |
| Manual atomic<int> refcount | 4 B (inline) | 4 B (inline) |

The original explanation described `make_shared`'s control block as approximately
16 B, unlike the former summary's 12 B. Neither estimate is a current allocator
or standard-library guarantee.

## Retired runtime reference

The following data belongs to the retired Objective-C runtime (`objc_msgSend`,
heap allocation, and runtime ARC), **not `oz2c`**. Its benchmark compilation path
no longer builds. The original record's nanosecond conversions are retained;
do not apply the transpiler comparison's 64 MHz clock to them.

### Message dispatch

With flat dispatch (`CONFIG_OBJZ_FLAT_DISPATCH=y`, a retired option):

| Operation | Cycles | ns |
|---|---:|---:|
| C function call (baseline, cached IMP) | 13 | 520 |
| objc_msgSend (instance method) | 205 | 8,200 |
| objc_msgSend (class method) | 212 | 8,480 |
| objc_msgSend (inherited depth=1) | 205 | 8,200 |
| objc_msgSend (inherited depth=2) | 205 | 8,200 |

Without flat dispatch:

| Operation | Cycles | ns |
|---|---:|---:|
| C function call (baseline, cached IMP) | 13 | 520 |
| objc_msgSend (instance method) | 560 | 22,400 |
| objc_msgSend (class method) | 743 | 29,720 |
| objc_msgSend (inherited depth=1) | 887 | 35,480 |
| objc_msgSend (inherited depth=2) | 1,328 | 53,120 |

### Object lifecycle

| Operation | Cycles | ns |
|---|---:|---:|
| alloc/init/release (heap) | 4,474 | 178,960 |
| alloc/init/release (static pool) | 2,151 | 86,040 |

### Reference counting

| Operation | Cycles | ns |
|---|---:|---:|
| retain (via dispatch) | 240 | 9,600 |
| retain + release pair | 320 | 12,800 |
| objc_retain (ARC, direct C call) | 58 | 2,320 |
| objc_release (ARC) | 135 | 5,400 |
| objc_storeStrong (ARC) | 221 | 8,840 |

### Introspection

| Operation | Cycles | ns |
|---|---:|---:|
| class_respondsToSelector (YES) | 148 | 5,920 |
| class_respondsToSelector (NO) | 461 | 18,440 |
| object_getClass | 20 | 800 |

### Blocks

| Operation | Cycles | ns |
|---|---:|---:|
| C function pointer call (baseline) | 10 | 400 |
| Global block invocation | 20 | 800 |
| Heap block invocation (int capture) | 20 | 800 |
| _Block_copy + _Block_release (int capture) | 3,060 | 122,400 |
| _Block_copy (retain heap block) | 154 | 6,160 |

### Block memory

| Metric | Size |
|---|---:|
| C function pointer | 4 B |
| Block pointer (reference) | 4 B |
| struct Block_layout | 20 B |
| Block + int capture (descriptor size) | 24 B |
| Block + ObjC object capture (descriptor size) | 24 B |
| Block + __block int (descriptor size) | 24 B |
| Heap cost: _Block_copy (int capture) | 32 B |
| Heap cost: _Block_copy (obj capture) | 32 B |
| Heap cost: _Block_copy (__block int) | 56 B |

### Logging

| Operation | Cycles | ns |
|---|---:|---:|
| printk (simple string) | 2,301 | 92,040 |
| LOG_INF (simple string) | 2,903 | 116,120 |
| OZLog (simple string) | 3,280 | 131,200 |
| printk (integer format) | 2,196 | 87,840 |
| LOG_INF (integer format) | 2,797 | 111,880 |
| OZLog (integer format) | 3,883 | 155,320 |
| printk (string format) | 2,039 | 81,560 |
| LOG_INF (string format) | 2,640 | 105,600 |
| OZLog (string format) | 3,892 | 155,680 |
| OZLog (%@ object format) | 8,480 | 339,200 |

### Firmware footprint

| Configuration | FLASH | RAM | FLASH delta | RAM delta |
|---|---:|---:|---:|---:|
| Bare Zephyr (no ObjC) | 12,104 B | 6,120 B | - | - |
| All features enabled | 39,568 B | 26,020 B | +27,464 B | +19,900 B |

Flat dispatch cost:

| Metric | Flat dispatch | No flat dispatch | Delta |
|---|---:|---:|---:|
| FLASH | 39,568 B | 38,384 B | +1,184 B |
| RAM (BSS + data) | 26,020 B | 22,180 B | +3,840 B |

Blocks runtime cost:

| Metric | Blocks on | Blocks off | Delta |
|---|---:|---:|---:|
| FLASH | 39,568 B | 36,576 B | +2,992 B |
| RAM (BSS + data) | 26,020 B | 25,996 B | +24 B |
