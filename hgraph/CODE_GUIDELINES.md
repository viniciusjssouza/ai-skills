# Hiero Consensus Node -- Code Review Guidelines

---

## 1. Determinism

Determinism is the single most critical property of this codebase. Every consensus node must process every transaction identically. A violation causes an ISS (Invalid State Signature), which takes nodes offline.

### Use Ordered, Deterministic Collections

All collections used in consensus-critical paths must have deterministic iteration order. Use `List` (sorted by a stable key such as `nodeId`), `TreeMap`, or `TreeSet`. Never use `HashMap`, `HashSet`, or `Stream.collect(Collectors.toSet())` where iteration order matters.

> "These should be deterministic. So use a `List`"
>
> "We should not use protocol buffer maps in state due to the specified non-deterministic characteristic of protocol buffer map iteration."

### Network Properties Must Be Identical Across Nodes

**Network properties** (stored in File 121) must be the same on every node. **Node properties** can differ per node. Environment variables and system properties must never be able to override network properties, or an ISS is almost inevitable.

> "Every node in the network must have the same properties or they will potentially ISS."

### Config Changes Must Be Deterministic

Dynamic config changes applied at different times on different nodes can cause divergence. Be very careful with config that affects consensus-critical behavior.

> "Changing this on the fly can lead to non-deterministic results."

---

## 2. Error Handling

### Never Throw or Catch `java.lang.Error`

Application code must never throw `Error`. Use specific exceptions (`IllegalArgumentException`, `UnsupportedOperationException`, etc.) or at worst `RuntimeException`.

> "We really should not ever throw `Error`. That's always been ill-advised in Java."

### Never Catch Broad `Exception`

Catch specific checked exceptions, and only if necessary catch `RuntimeException` as a catch-all for unchecked exceptions.

> "We should not ever catch `Exception`. There isn't any value in doing so."

### Fail Loudly for Impossible Conditions

If a condition "should never happen," throw an exception. Do not log a warning and continue. Silent failures mask bugs.

> "If not, an exception should be thrown here rather than just silently ignoring the issue."

### Question Every Defensive Check

Every null check, catch block, or conditional guard should have a justifiable reason. If a situation is truly impossible, either assert it or remove the check.

> "Is this really possible? If not, an exception should be thrown."

### Assertions vs. Runtime Checks

Use `assert` for internal invariants between trusted components (debug-time checks). Use runtime checks (throwing exceptions) for anything that could be violated due to external input, configuration, or integration boundaries.

> "This is not user / external input, this is to check code flow between our classes -- they help with debugging, but are not expected in production. In cases like this we use assertions, not runtime checks."

### Prefer HandleException Over Node-Crashing Preconditions

In the handle workflow, prefer `HandleException` over `Preconditions.checkArgument` or other mechanisms that could halt the node.

> "This is a strong precondition and might stop the node? We usually throw `HandleException`."

### Fee Calculators Must Never Throw

Fee calculation code must handle missing data gracefully. A thrown exception in a fee calculator can have cascading effects on transaction processing.

> "We should not throw any errors in fee calculations."

### Prefer Logging Over Throwing in Recoverable Scenarios

If a class can survive an incorrect call, log an error rather than throwing an exception that might turn a survivable failure into a fatal one.

> "Since this class can easily survive an incorrect call to `closeBlock()`, we should be certain the upstream failure that caused the incorrect call is already fatal before throwing an exception here and risking network stability for no real benefit."

---

## 3. Log Level Discipline

Log levels are tied to operational impact. `WARNING` and `ERROR` logs normally trigger pager alerts.

|  Level  |                                      When to Use                                       |
|---------|----------------------------------------------------------------------------------------|
| `ERROR` | Unrecoverable conditions. "Is this event worth waking you in the middle of the night?" |
| `WARN`  | Unexpected but survivable states. Operations may need to investigate.                  |
| `INFO`  | Key state transitions, sanity-check points for manual verification.                    |
| `DEBUG` | Detailed diagnostic information for development.                                       |
| `TRACE` | Very fine-grained diagnostic detail.                                                   |

> "Unless this is likely to cause the system to fail or exit, `Error` is generally not the best log level."
>
> "We reserve `ERROR` for unrecoverable conditions."

### Library Code Logs One Level Lower

Library/framework logs should be at least one level below what application code would use for a similar event (error → warning, warning → info, etc.).

> "In general, library logs are easier to manage if they're at least one level below what you would use for a similar event in application code."

### No String Concatenation in Debug Logs

Use parameterized logging to avoid the performance cost of string concatenation when the log level is disabled.

> "This has a performance cost for a debug log. Perhaps replace the catenated string with a constant."

### Log at Transition Points

Add `INFO` or `WARN` logs at state transitions so operators can verify correctness.

> "Maybe an `INFO` log here would be nice so we can do a quick manual sanity-check."

---

## 4. Performance

### Avoid Unnecessary Object Allocations

Do not create builders when constructors suffice. Do not create iterators when a direct access method exists. Do not use boxed types when primitives will do.

> "Can we avoid using builders where we could just use constructor. It is wasteful to create a builder object if it can be avoided."
>
> "There is a big question... should we be using Eclipse Collections primitive collections?"

### Avoid `Optional` in Hot Paths

`Optional` creates unnecessary allocations and prevents JIT optimizations. Use if-else or ternary in performance-critical code.

> "Using an `Optional` and the `.map().orElse()` methods here creates unnecessary memory allocations and potentially creates a performance bottleneck."

### Cache Expensive Computations That Rarely Change

Parse and cache data that changes infrequently (like fee schedules) rather than recomputing on every transaction.

> "It is expensive to do this every time, because the feeSchedule changes rarely... we should parse and read feeSchedule once when it changes."

### Avoid Redundant Iterations

Prefer single-pass algorithms over multiple iterations over the same collection. Combine related lookups.

> "I would prefer looking up block node existence and adding rewards here, instead of going through the loops of nodeActivities again."

### Don't Add Caches Where the Store Already Caches

Understand the caching layers beneath you before adding your own.

> "Why do we need a cache? The KV states already cache any account lookups."

### Prefer `getAndAdd` Over `addAndGet` When Return Value Is Unused

> "We aren't using the return value, so `getAndAdd` might be a slightly better choice (makes cleaner use of CPU cache)."

### Avoid Locks When Atomics Suffice

> "We should not use a lock here. If we need to prevent running multiple responses simultaneously we can just use an AtomicBoolean flag."

### Avoid Buffering Entire Responses in Memory

Use output streams directly instead of building responses in memory.

> "Why are we building the entire response in memory? The server response has an output stream that we should use directly."

---

## 5. Numeric Precision

### Never Convert Integers to `double`

Many integer values cannot be expressed accurately as `double` due to IEEE 754 limitations. Never convert `long` metrics or financial values to `double`.

> "We should _never_ transform an integer into a `double` in metrics."

### Use Overflow-Safe Arithmetic for Financial Calculations

Use `clampedAdd` and `clampedMultiply` for fee and amount calculations.

> "We should use `clampedAdd` here to be safe. Same for all long additions or multiplications."

### Understand Floating-Point Limitations

Even simple decimal values like `0.1`, `1.4`, or `1.6` cannot be expressed exactly in `double`. Use explicit fractions and fixed-point decimal where precision matters.

> "Keep in mind that while `1.5` can be expressed correctly in floating point, `1.4` and `1.6` cannot."

---

## 6. State Management

### State Storage Is Precious

Every byte stored in state is preserved forever in the block stream. Do not store data on a "we might need it later" basis.

> "State storage is precious and every byte stored in state also adds to the 'store this forever' cost of the block chain."

### Never Store Transaction Bodies in State

> "We really should not be storing transaction bodies in state; it sets a bad pattern."

### Never Use Protobuf Maps in State

Protobuf map iteration order is non-deterministic by specification. Use virtual maps instead.

> "We should not use protocol buffer maps in state due to the specified non-deterministic characteristic of protocol buffer map iteration."

### Be Careful with Noisy State Changes

State changes are written to the block stream forever. Summarize per-round when possible.

> "We should be careful of noisy state changes as they will all get written to block stream forever."

### Lifecycle Awareness: `migrate()` vs `restart()` vs `reconnect()`

Be precise about which lifecycle hook code runs in:
- `migrate()` -- runs only on upgrade boundaries (one-time state changes)
- `restart()` -- runs on every `InitTrigger.RESTART`, including after reconnect
- `reconnect()` -- runs specifically after reconnect

Migration-only logic must not go in `restart()`.

> "This should be done in `migrate()` method right? `restart()` runs on reconnect also."
>
> "The semantics of what we're doing are clearer by overriding `migrate()`: it runs only on upgrade boundaries, which emphasizes this is a one-time state update."

---

## 7. Annotations

### Use Only `edu.umd.cs.findbugs.annotations`

Always use `@NonNull` and `@Nullable` from `edu.umd.cs.findbugs.annotations`. Never use `javax.annotation`, `org.jetbrains.annotations`, or any other annotation library for nullability.

> "This annotation should not be used (use the `edu.umd.cs.findbugs.annotations` version instead). It might help to adjust your IDE settings to exclude `javax.annotation` from automatic imports entirely."

### Annotation Ordering

Follow the convention `@NonNull final` (annotation before `final`).

> "I think our coding standard is `@NonNull final`."

### Be Consistent with Nullability Contracts

If a parameter is `@NonNull` in one method of an API, it should be `@NonNull` everywhere it appears in that API.

> "In `SimpleFeeCalculator.calculateTxFee()` the `FeeContext` argument is annotated `@NonNull` but here `@Nullable`. This suggests some confusion."

---

## 8. Immutability and `final`

### Use `final` Everywhere Possible

Mark variables, fields, parameters, and classes as `final` whenever possible. It makes code "tighter" with fewer loose ends.

> "Personally, I like using final where possible. It just feels 'tighter' -- fewer loose ends."
>
> "Make the arg final, please (in other methods, too)."

### Prefer `final` Classes for Handlers and Leaf Types

Unless a class is specifically designed for extension, make it `final`.

> "Should this at least be final? If not, someone can override and return something random."

---

## 9. Naming

### Names Must Be Precise and Accurate

Names must accurately reflect what they represent. If a variable's role changes, rename it. Use plural for collections, singular for single items.

> "This name should be `associatedRegisteredNodes` instead of `associatedRegisteredNode`."
>
> "Keys are bucket IDs, not user keys."

### Include Units in Method/Variable Names

When a value has a unit, make it explicit in the name.

> "I would lean toward naming these e.g. `addTinybarServiceFee()` to make the units of the argument explicit."

### Use Standard Casing Conventions

- Constants: `FILE_ID_KEY` not `FILEID_KEY` (separate words)
- Loggers: `logger` in lowercase
- Repeated proto fields: use plural (`transactions` not `transaction`)
- Config class names should match config key structure

> "I guess normally we'd say `UPGRADE_FILE_ID_KEY`, so that `FILE_ID` rather than `FILEID`."

### `is*()` Methods Must Be Pure

Methods with the `is*()` prefix should be side-effect-free. If a method has side effects, use a verb-based name.

> "I think the `is*()` naming convention might make the side-effects a bit less obvious to the reader."

### Avoid Prefixes Like `Default` When Only One Implementation Exists

> "I think the name here should be just 'GossipService' instead of 'DefaultGossipService'."

---

## 10. Encapsulation and Module Boundaries

### Keep Implementation Classes Package-Private

Implementation details should not leak through public APIs. Minimize visibility to the narrowest scope possible.

> "I'm wondering whether this class has to be public or can be package private."
>
> "Could this be reduced to package scope?"

### Respect the API/Impl Split

Never expose `impl` modules to other `impl` modules at compile-time. Depend on the API module at compile-time and the impl module at runtime only.

> "This should not exist here. If somebody were to call this, it would be disastrous."

### Never Expose Writable Stores from Read-Only Contexts

`ReadableStoreFactory` should never provide access to writable stores. Fee calculators and query contexts must be read-only.

> "This doesn't seem right, we should not have access to `writableStore` from `ReadableStoreFactory`."

### Proper Encapsulation of Mutable State

Even when a class has mutable state, use `private` fields with a public method API that maintains internal invariants.

> "Just because a class has mutable state doesn't mean we should expose its internals as mutable `public` variables."

---

## 11. API Design

### No Surprising Side Effects

Method names must not hide side effects. A getter should not mutate state.

> "This name reads like a POJO getter that returns `true` if the item was added... But it _actually_ may have a side-effect to mark the item as added."

### Promote Methods to the Right Abstraction Level

Methods should live at the most general appropriate level. If a method is useful across implementations, put it on the interface.

> "Even better, the `getOffHeapConsumption()` method can be promoted to the `LongList` interface."

### Every Abstraction Must Earn Its Existence

Do not create wrappers, interfaces, or layers that don't add value. When only one implementation exists, consider merging it with the interface.

> "No we are getting rid of the in memory implementation, should we consider merging the interface and single implementation?"

### Minimize Interface Hierarchies

Having many interfaces is a cognitive load and a JVM performance cost (classes implementing more than two interfaces take a slow dispatch path).

> "If a class implements more than two interfaces all method invocations go down a slow path."

### Use `var` Judiciously

Use `var` consistently when it improves readability (long generic types), but prefer explicit types for short type names like `long` or `int`.

> "Although `var` for lengthy type names improves readability, `long` and `var` have nearly the same number of letters -- and obviously `long` is more informative."

### Prefer Dynamic Config Over Cached Config

Read config dynamically when needed rather than caching at construction time. The purpose of `ConfigProvider` is to give the current config at an arbitrary point in time.

> "Config like this is better to get dynamically when needed rather than stored. It is possible the config options will change."

---

## 12. Protobuf Style and Convention

### Field Naming: `lower_snake_case`

Protobuf compilers translate `lower_snake_case` correctly to idiomatic style for various languages.

> "We strictly use `lower_snake` style for field names because the compilers translate that correctly to idiomatic style for various languages."

### Enum Values: `ALL_CAPS_SNAKE`

> "The style we've used for enum values has generally been `ALL_CAPS_SNAKE`."

### 4-Space Indentation

Proto files use 4-space indentation, matching Java code.

> "Indentation is incorrect in these proto files. It should be 4 spaces per level, not 2."

### Newline at End of File

> "Missing newline at EOF."

### SPDX License Headers

Use the SPDX identifier, not the obsolete copyright block.

> "Please use the proper SPDX identifier instead of this obsolete comment block."

### Prefer `uint64` Over `int64` for Non-Negative Values

> "We should prefer unsigned for these unless negative values are expected."

### Full Specification Documentation Required

All messages and fields need full specification documentation, not just brief descriptions. Use RFC keywords (SHALL, MUST, MAY) correctly.

> "All of the messages and fields need full specification, rather than just brief description."
>
> "`CAN` isn't a keyword in the RFC, I recommend changing it."

### Use Proper Package Names for New Proto Files

New proto files should use proper packages (`com.hedera.hapi.node.<service>`), not legacy package names.

> "Please do not perpetuate this terrible pattern. For completely new services, to the extent possible, we want to define the protocol buffers in a proper package."

### Do Not Use `ProtoLong` in New Code

Use `uint64` (if no explicit presence is needed) or `google.protobuf.UInt64Value` (if explicit presence is required).

### Fixed Fields Before Repeated Fields

There is a subtle efficiency benefit to having fixed-size fields precede repeated fields in proto messages.

### Don't Reserve Fields in Non-Final Protos

If the proto schema is not yet final, just renumber fields instead of reserving removed ones.

> "We do not need to reserve 4 as this protobuf message is not final yet."

### Remove Empty Messages

> "It is very odd to have empty messages."

---

## 13. Use PBJ, Not Google Protobuf

All new code should use PBJ (Protobuf Builder for Java) codecs and objects, not google protobuf.

> "We should not require any google protobuf objects here anymore."

### Use `copyBuilder` for Protobuf Modifications

When modifying a protobuf object, use the `copyBuilder` pattern rather than constructing from scratch.

> "I would have used the `copyBuilder` here, unless there is a reason not to."

### Use PBJ Idioms

Use PBJ's built-in `*OrThrow()` methods instead of manual `requireNonNull` after `has*()` checks.

> "Instead of requireNonNull, you can do `endpoint.blockNodeOrThrow()` after checking `hasBlockNode`."

### Hash Format Is Public API

The data hashed and its format are API -- they must be explainable, documented, and specified in protobuf. No special binary encodings.

> "The whole use of HashBuilder is concerning. The data we hash and its format is 'API'!"

---

## 14. Concurrency and Thread Safety

### Be Explicit About Threading Contracts

Document which thread calls what. Mark fields that need cross-thread visibility correctly. Do not synchronize on non-final fields.

> "How is this field synchronized? It's updated on the scanner task thread only, but it can be read later on different threads."

### Use Proper Synchronization (or Document Why Not)

If external synchronization exists, a `ConcurrentHashMap` is unnecessary -- use `HashMap`. If no synchronization exists, use atomic types or synchronize properly.

> "If synchronized on 'this', this can be just a HashMap."

### Catch `Throwable` in Pool Threads

`ForkJoinPool.commonPool()` threads can die silently. Always catch and log `Throwable` in pool-submitted tasks.

> "I've had `ForkJoinPool.commonPool()` threads die silently on me one too many times not to be paranoid."

### Prefer `AtomicReference` Over `volatile`

> "The static analyzer was sure volatile was a bug. We know atomic reference for sure will work."

### `MessageDigest` Is Not Thread-Safe

Never share `MessageDigest` instances as fields. Create fresh instances where needed.

> "MessageDigest is mutable and not thread-safe."

---

## 15. Testing

### Never Weaken Tests to Make Them Pass -- Fix the Code

Tests represent the specification. If a test fails after a change, the change is wrong, not the test.

> "The test logic should not change. We need to align the behavior in the modular code to match the mono service behavior, and the tests are the guard to ensure we do it correctly."

### Write Precise Assertions

Test exact values, not just non-null or non-empty. Verify the translated object has the right values.

> "Can we have a more precise test that verifies the translated object actually has the right hash values, and not just are longer than 0?"

### Prefer AssertJ Over JUnit Assertions

> "We should favor AssertJ instead of JUnit Jupiter assertions."

### Test Edge Cases

Always test null handling, default values, empty arrays, boundary conditions, and round-trip serialization.

> "Are we testing all the edge cases like null, default etc? Also do we test that written objects are readable with PBJ generated code from schema?"

### Use Parameterized Tests for Multiple Scenarios

> "I suggest making this a parameterized test and adding a few other scenarios."

### Add Meaningful Display Names to Tests

> "Maybe the display name would be clearer with a description also of what should happen."

### In Unit Tests, Prefer `throws` Over Catch-and-Rethrow

Let test exceptions propagate naturally for cleaner CI stack traces.

> "In a unit test, instead of catch-and-rethrow, perhaps just add the exception to the `throws` clause."

---

## 16. Remove Dead Code and Artifacts

### No Dead Code, Debug Artifacts, or Commented-Out Code

Remove `System.out.println`, unused imports, unused parameters, unused methods, and commented-out code.

> "Let's make sure to remove these printlns."
>
> "Either uncomment, or remove, please."

### Remove Dead Abstractions

If a class, method, or interface is no longer used, delete it entirely.

> "I believe this class can be deleted completely now."

### Remove Dead Migration Code

If old migration paths are no longer supported, delete their tests too.

> "Hashes to hash chunks migration will not be supported after this fix. Why having a test for it?"

---

## 17. No TODOs Without Tickets

Never leave bare `TODO` comments. Either fix it now, delete the comment, or create a tracked ticket with a target release and link it as `FUTURE WORK`.

> "Let's not have any TODOs in the code. It should be either removed or changed to FUTURE WORK + a link to a ticket."
>
> "Do we have issues filed for this to make sure somebody comes back and addresses it?"

---

## 18. Constants and Magic Numbers

### No Magic Numbers

All magic numbers must be extracted into named constants with descriptive names.

> "What is 3 here. Defining a constant with a descriptive name will be helpful."

---

## 19. Javadoc and Documentation

### All Public Methods Must Have Javadoc

> "Please add javadocs for all the methods that are public."

### Comments Must Be Accurate

A misleading comment is worse than no comment. When code changes, update accompanying documentation.

> "Strictly speaking, this method lies."

### Document the "Why", Not Just the "What"

Explain domain separation in hashing, reasons for specific byte prefixes, and other non-obvious design decisions.

> "Be nice to add comment for why we are adding the 0x00 byte prefix."

### Use Javadoc Formatting Correctly

- Use `<br/>` for line breaks within a block, but don't add breaks between every line when you want lines to merge
- Paragraphs go between description and requirements, not between each item

---

## 20. Security

### Fail Closed, Not Open

Default security checks and validation to deny, not allow. Use `key -> false` not `key -> true` for stubs.

> "With `key -> true` I'm worried that maybe somebody won't see or respond to the TODO and we'll end up with the sig check always passing."

### Watch for Throttle Bypass

Ensure that rejected transactions properly refund consumed throttle capacity.

> "This lets an attacker consume ingest throttle capacity without paying fees at consensus for that usage."

### Verify Signatures Against Original Bytes

PBJ may not serialize a `TransactionBody` to the same bytes the user submitted. Always verify signatures against the original wire bytes.

> "There is no guarantee that PBJ will serialize this `TransactionBody` object to the same `byte[]` the user submitted."

### Use `char[]` Over `String` for Secrets

`char[]` can be zeroed after use, whereas `String` retains the value in memory until GC.

> "For the secret key, a `char[]` is more secure because it can be zeroed."

### Host Your Own Copies of External Scripts

Don't pull shell scripts from third-party GitHub repos at build/CI time. Fork and maintain your own copy.

> "Should we host our own fork of this bash script somewhere in the Hiero org, rather than pulling 'whatever's latest on github'?"

---

## 21. PR Discipline

### Keep PRs Focused

Each PR should address a single concern. Unrelated improvements, formatting fixes, or refactors go in separate tickets/PRs.

> "To minimize the size of the current PR, this change is not needed."
>
> "Revert." (on unrelated changes)

### Revert Unrelated Changes

If a PR contains changes outside its scope, those changes should be reverted and submitted separately.

### Don't Commit Debug/Local Artifacts

Editor configs belong in `~/.gitignore_global`, not the repository `.gitignore`. Debug logging, local configuration changes, and test modifications that aren't part of the feature must not be committed.

> "There are a bunch of changes to build, logging etc files that seem like they are local debug changes rather than should be committed."

---

## 22. Architecture Patterns

### Choose the Right Data Structure

Use the simplest data structure that fits. Don't use a `Map` when a `Set` suffices. Don't use a concurrent collection when external synchronization already exists. Use arrays when IDs are sequential.

> "This can be a set. Map elements are never queried by key."

### Extract Common Logic, Don't Duplicate

When multiple services do similar things, extract common utilities to base classes or shared modules.

> "Can all this code be moved to `AbstractSimpleFeeCalculator`? Since it will be the same for every handler."

### Push Validation into the Store Layer

Increment counters and validate invariants at the store layer rather than in individual handlers, to prevent missed cases.

> "We should increment entity counts here, instead of handler to make sure we won't miss any other places this can be called."

### Use Existing DI Mechanisms

Prefer injecting existing singletons via Dagger over creating new abstraction types.

> "Instead of creating another type for people to grasp, I would lean to passing the singleton into each `DispatchHandleContext` and using it directly."

### Specification Changes Require a HIP

Any change to the Hedera specification must go through the HIP (Hedera Improvement Proposal) process.

> "Since this is a change to the specification, it needs to be in a HIP."

---

## 23. Serialization

### `serialize()` and `deserialize()` Must Be Symmetrical

If a class supports serialization, the serialize and deserialize methods must be exact inverses.

> "`serialize()` and `deserialize()` must be symmetrical."

### Support Only One Version Back for Migration

If a node runs version N and needs to upgrade to N+2, it must first upgrade to N+1. Only one previous version needs to be supported in code.

> "The idea is only one previous version needs to be supported in code."

---

## 24. Modern JDK APIs

Prefer modern JDK APIs over legacy patterns or third-party libraries:

- `java.util.HexFormat` over manual hex conversion
- `java.time.InstantSource` over custom time abstractions
- `ScopedValue` over `ThreadLocal` (when available)
- `record` types over plain classes for data holders

> "JDK 17 now has `HexFormat`."
>
> "This guy would be a great candidate for a `record` type instead of a `class` type."

---

## 25. Build and CI

### Use Major Java Version Only in CI

Let the runner image control the specific patch version.

> "We should always use the major version only."

### Never Import Generated Code in Gradle

Generated code is subject to change between caches and builds.

> "These should never be imported and will break the build."

### Always Double-Quote Shell Variables in CI Workflows

Prevent globbing and word splitting in GitHub Actions.

> "Should be double quoted ideally to prevent globbing in the event the path contains spaces or colons."

### Minimize CI Costs

Don't re-run expensive test suites for trivial changes.

> "I'm not sure it is worth running another hour of tests and incurring the costs for this change."

### Run `./gradlew qualityGate` Before Committing

Formatting is enforced by the quality gate. Fix issues before opening a PR.

---

## 26. Design Philosophy

### Minimize Configuration Knobs

The more knobs, the more likely one will be wrong. Prefer systems that auto-tune rather than requiring manual operational intervention.

> "We should strive for a design with the fewest knobs possible. The more knobs, the more likely we get one wrong."
>
> "I think we should strive for goal of automatic management, minimum operations effort needed."

### Keep Feature Flags Disciplined

New features must be behind feature flags. Don't merge config or code that isn't used yet. Don't add flags when existing config already conveys the same information.

> "Why do we need new feature flag for this? Shouldn't this be the default behavior if `hintsEnabled=false`?"

### On-Disk Config Files Should Be Human-Readable

Use YAML or similar formats for operational config, not protobuf.

> "On-disk startup files should be human readable and human editable to support operations teams."

### Avoid JSON in Production -- Prefer Protobuf

For data serialization, prefer protobuf with PBJ. JSON is slow and non-standard for this codebase.

### Imports

Always use module-level imports at the top of your Java files.Never use fully-qualified class names (e.g., java.util.List or java.io.IOException) in the body of the code.Resolve imports explicitly instead of relying on wildcards (e.g., import java.util.List; instead of import java.util.*;) to prevent namespace collisions.


## 27. Code format

When placing/organizing functions and methods, always order theme following the clean code STEP DOWN rule: code should read like a top-down narrative, where high-level abstract functions are written first, and each subsequent function descends exactly one level of abstraction.
