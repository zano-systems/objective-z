<!-- SPDX-License-Identifier: Apache-2.0 -->

# Practical ARC guide

This is a guide to writing source for Objective-Z's constrained ARC lowering.
The [ARC conformance ledger](ARC.md) is the rule-by-rule contract; the
[dialect ledger](OBJECTIVE_C_DIALECT.md) records author-visible constructs.
For the analysis architecture and its limits, see the
[hybrid model](STATUS.md#the-hybrid-model-what-this-backends-arc-is).

ARC is always enabled. Clang checks Objective-C semantics with `-fobjc-arc`;
`oz2c` implements the retain/release lowering into C. Manual sends of `retain`,
`release`, `autorelease`, `dealloc`, and `retainCount` are refused, as are declarations
or definitions of those methods except a `-dealloc` override (#428, #436).
Use the plain C function `oz_retain_count` for diagnostic refcount reads.

## Owned locals and cleanup

An allocation bound to a supported local lifetime is reclaimed when that scope
ends. Objects stored in owning slots or returned to a caller can outlive the scope;
cleanup is not simply a release of every object mentioned in the body.

```objc
#import <Foundation/Foundation.h>

@interface Sensor: OZObject
@property (nonatomic, strong) id delegate;
- (void)measure;
@end

@implementation Sensor
@synthesize delegate = _delegate;

- (void)measure
{
	OZLog("Measuring...");
}

- (void)dealloc
{
	OZLog("Sensor deallocated");
	/* Do not send [super dealloc]: the generated chain runs it. */
}
@end

void demo(void)
{
	Sensor *s = [[Sensor alloc] init];
	[s measure];
	/* ARC releases this owned local at scope exit. */
}
```

The generated cleanup covers supported early exits, including return, break, and
continue. See [the ownership matrix](../tools/oz2c/tests/ownership_matrix.rs) for
tested destinations of owned references, and
[the selector ownership matrix](../tools/oz2c/tests/selector_ownership_matrix.rs)
for the selectors and constructs involved. These tests are evidence, not a proof
for arbitrary source.

## Strong properties, ivars, and deallocation

An owning property or ivar keeps its object alive. Replacing a strong slot releases
the previous value and establishes ownership of the new one. When the owner dies,
generated cleanup releases its owned ivars.

**The `-dealloc` hook runs before owned ivars are released.** It can still read those
ivars. The generated code invokes the deallocation chain from the derived class
through its superclasses, releases owned ivars, and frees storage. Do not send
`[super dealloc]` or manually release the ivars. The previous README's claim that
a hidden `.cxx_destruct` ran before the hook did not describe the generated C;
see [the actual deallocation lowering](../tools/oz2c/src/companion.rs).

```objc
@interface Driver: OZObject
@property (nonatomic, strong) Sensor *sensor;
@end

@implementation Driver
@synthesize sensor = _sensor;

- (void)dealloc
{
	OZLog("Driver deallocated");
	/* _sensor is still owned here; generated cleanup releases it afterward. */
}
@end

void use_driver(void)
{
	Driver *d = [[Driver alloc] init];
	Sensor *sensor = [[Sensor alloc] init];
	d.sensor = sensor;
	/* Releasing d also releases its owned sensor. */
}
```

Use a named owning local in this example. The inline form
`d.sensor = [[Sensor alloc] init]` currently retains through the synthesized setter
without balancing the allocation's original reference, so the sensor leaks.
This is a lowering defect, not an alternative ownership contract. The named local
receives scope cleanup while the property keeps its own reference.
The defect is tracked as [#616](https://github.com/zano-systems/objective-z/issues/616).

## Allocation and initialization are separate

`+alloc` takes a slab slot and zero-initializes it. `-init` is an ordinary method
and may be called more than once without taking another slot. An initializer must
therefore be idempotent: release or free existing state before replacing it. A
strong-ivar store handles the previous object; raw allocation or repeated Zephyr
callback registration does not.

`[u init]` does not create a second owned reference when `u` already owns the
allocation. An ordinary cast around that send does not change the ownership
decision. Method-family and initializer rules are recorded in [ARC.md](ARC.md);
do not infer ownership from a selector's prefix alone.

Allocation can fail. See [pool capacity and exhaustion](USER_GUIDE.md#resource-model)
before relying on a factory or initializer to propagate `nil`.

## Use scopes, not autorelease pools

`@autoreleasepool` is refused (#430). Objective-Z has no pending-autorelease
mechanism. A plain braced scope is the lifetime boundary for its owned locals.
A loop body already provides a per-iteration scope:

```objc
void process(void)
{
	for (int i = 0; i < 1000; i++) {
		Sensor *tmp = [[Sensor alloc] init];
		[tmp measure];
		/* tmp is reclaimed at the end of this iteration. */
	}
}
```

That example needs one Sensor at a time. It does not justify a one-slot pool for
loops that retain their results elsewhere, call overlapping factories, or run
concurrently.

## Retain cycles and non-owning references

Two objects owning each other can remain alive after their external owners are
gone. `__weak` and weak properties are refused (#448); there is no zeroing weak
reference facility.

A back-reference can be non-owning:

```objc
@class Parent;

@interface Child: OZObject
@property (nonatomic, unsafe_unretained) Parent *parent;
@end
```

**Non-owning pointers are not cleared when the object dies.** Ensure the referenced
object outlives every use, or clear the back-reference before releasing it.
`__unsafe_unretained` changes ownership; it does not make a dangling pointer safe.

Alternatively, break an owning cycle while you still have access to its objects:

```objc
/* Node has a strong next property. */
void no_leak(void)
{
	Node *a = [[Node alloc] init];
	Node *b = [[Node alloc] init];
	a.next = b;
	b.next = a;

	b.next = nil;
	/* Scope cleanup can now reclaim the objects. */
}
```

## C APIs, callbacks, and `__bridge`

`__bridge` casts between object pointers and C pointers without ownership transfer.
The generated cast does not extend the object's lifetime. A C API retaining a user
data pointer is not the same as retaining an Objective-Z object.

```objc
/* MyTarget declares -onTimeout. An owning reference elsewhere must keep
 * it alive until the timer is stopped and callbacks can no longer use it. */
static void on_expiry(struct k_timer *t)
{
	MyTarget *tgt = (__bridge MyTarget *)k_timer_user_data_get(t);
	[tgt onTimeout];
}

K_TIMER_DEFINE(my_timer, on_expiry, NULL);

/* Within MyTarget's implementation: */
- (void)arm
{
	k_timer_user_data_set(&my_timer, (__bridge void *)self);
	k_timer_start(&my_timer, K_MSEC(100), K_NO_WAIT);
}
```

- `(__bridge void *)obj` hands C a borrowed pointer.
- `(__bridge Type *)ptr` recovers a borrowed object reference.
- Maintain an independent owning reference for the duration of C's use, including
  outstanding callbacks. Stopping new work alone may not settle work already running.
- `__bridge_retained`, `__bridge_transfer`, and ownership-changing attributes such
  as `ns_consumed` / `ns_returns_retained` are refused (#458, #460).
- Ordinary casts do not erase ownership. `(void)[t copy]` still needs cleanup;
  casting an allocation to its concrete type does not remove its scope-exit release.

Owned temporary arguments to an Objective-C send are released after the send;
a strong setter can establish the destination's ownership before that release.
An initializer send on an already-owned receiver does not add another reference.

An owning result passed directly to a **plain C function** is not automatically
released by the current lowering. For example, `OZLog("%@", [Foo new])` leaks.
Bind the result to an owned local:

```objc
Foo *value = [Foo new];
OZLog("%@", value);
/* Scope-based cleanup releases value. */
```

That also does not authorize C to keep the pointer after the local's lifetime.

## Returns, aliases, and protocol dispatch

`oz2c` follows supported aliases and escapes so returning an alias does not release
its owner prematurely. Where return provenance cannot be established, its hybrid
model can retain the returned reference. This is not a general solution to an
unknown ownership contract at a call site.

Protocol dispatch must have a consistent ownership contract across implementations;
runtime class selection cannot repair a disagreement. Use the
[hybrid-model explanation](STATUS.md#the-hybrid-model-what-this-backends-arc-is)
and [conformance ledger](ARC.md) for the precise rules rather than assuming all
Clang ARC behavior is reproduced.

## Checklist

| Do | Do not |
|---|---|
| Use `objz_transpile_sources()` for the configured build path | Bypass semantic checks and assume equivalent coverage |
| Use strong slots for owning relationships | Send manual memory-management messages |
| Let supported scope cleanup reclaim locals | Write `@autoreleasepool` |
| Treat `-dealloc` as a resource cleanup hook | Send `[super dealloc]` or release owned ivars yourself |
| Keep an owner alive across C callback use | Treat `__bridge` as a retain |
| Manage non-owning back-reference lifetimes | Expect `__unsafe_unretained` to be zeroed |
| Budget capacities and handle allocation failure | Treat ARC as a peak-memory proof |

Known limitations and unexamined rules belong in [ARC.md](ARC.md), with per-site
evidence in the ownership tests. A successful transpile or green suite does not
establish memory safety for every accepted program.
