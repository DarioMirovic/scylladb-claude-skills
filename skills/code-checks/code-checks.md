# ScyllaDB Code Checks

Line-level checks for Seastar and ScyllaDB code, used by the light review of
`dsz:review` and to judge Seastar claims in `/code-review` findings.
Design-level questions (scope, upgrades, raft, compatibility, tests) are in
`../design/design-checklist.md`. "Design checklist item N" below points
there.

The "before flagging" rules are in the `code-checks.md` of `dsz:review` and
always apply.

## Checks

### Lifetimes

1. **Captures in continuations.** A `.then()` lambda that captures a local
   by reference (or `this` of an object that may die) dangles once the
   scope ends. This holds for `.then()` chains inside a coroutine too.
   Move or copy into the capture. Coroutine locals live in the frame and are
   safe.
2. **Reference parameters of coroutines.** A coroutine keeps only the
   reference. The caller's object must live until the coroutine finishes,
   which is not true when the caller passes a temporary and does not await
   at once.
3. **References across `co_await`.** An iterator, pointer or reference into
   shared state (a map, a list, the schema) can be invalidated by another
   fiber while this one is suspended.
4. **Moves of inline storage.** Moving a `small_vector`, `std::array` or a
   short string moves the elements, so views and pointers into the old
   object dangle. Watch structs that hold both the storage and views into it.

### Futures and coroutines

5. **Dropped futures.** A future that nobody waits for loses its error
   ("Exceptional future ignored" in the log), and shutdown does not wait for
   the work. Background work needs a gate (design checklist item 14).
6. **Throwing from a future-returning function.** A function that is not a
   coroutine and returns `future<>` must return a failed future
   (`make_exception_future`), not throw: callers that chain
   `.handle_exception()` will not see a throw.
7. **Failed futures in coroutines.** `co_await f` and `f.get()` rethrow.
   On a hot path, propagate without a throw: `co_await
   coroutine::try_future(f)`, or `coroutine::as_future` then check
   `failed()` and `co_await coroutine::return_exception_ptr(...)`.
8. **Throwing on the hot path.** In a coroutine, `co_await
   coroutine::return_exception(...)` instead of `throw`. Errors that happen
   in normal operation are returned, not thrown (design checklist item 15).
9. **Ready-future fast path.** A hot function that usually has its result
   at once should not pay for a coroutine frame or a continuation: return
   `make_ready_future()` on the fast path. Extra wrappers like
   `futurize_invoke` cost on hot paths too.
10. **Long loops yield.** `co_await coroutine::maybe_yield()` in coroutines,
    `thread::maybe_yield()` in a `seastar::thread`, or a `need_preempt()`
    check. Loops over user-sized input without one stall the reactor.

### Memory and copies

11. **Hidden allocations on hot paths.** `std::function` (type-erased,
    allocates), lambdas capturing large objects by value, copies where a
    move or a `const&` would do, a `std::move` on a trivially copyable type
    that suggests a misunderstanding.
12. **Large contiguous allocations.** Containers that grow with user input
    above 128 KiB use `utils::chunked_vector` (design checklist item 11).
13. **Pointer types.** `lw_shared_ptr` within a shard, `seastar::shared_ptr`
    when polymorphism needs it, `foreign_ptr` across shards. `std::shared_ptr`
    only at external APIs. Objects are destroyed on the shard that owns them.

### Errors and invariants

14. **Exception type.** User errors use the same exception type as the
    neighbouring checks, so the client gets the right CQL error code (design
    checklist item 27).
15. **Invariant checks.** `SCYLLA_ASSERT` only when going on would corrupt
    data; otherwise `on_internal_error()` or `throwing_assert()`
    (`utils/assert.hh`) (design checklist item 28).

### Safety

16. **Bounds.** Sizes from input checked before use, correct lengths in
    `memcpy` and friends, no out-of-bounds index.
17. **Casts.** `reinterpret_cast` and C-style casts need a reason;
    narrowing conversions of sizes and counts are checked.

### Types and templates

18. **Templates.** Type erasure in code that is not performance sensitive;
    templates constrained by concepts; no template only to save a few lines.
19. **Identifiers.** A UUID-based id is a `utils::tagged_uuid` with its own
    tag, not a bare `utils::UUID`.
20. **Move operations.** Move constructors and move assignment are
    `noexcept`.
