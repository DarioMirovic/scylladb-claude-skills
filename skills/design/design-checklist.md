# ScyllaDB Design Checklist

Questions to answer before writing code, and to ask of a change in review.
Most items will not apply to a given change; say so in one word and move on.
Items come from `docs/dev/review-checklist.md`, `.github/instructions/`, and
real misses recorded in the local feedback log (see `SKILL.md`).

## Checklist

### A. Problem and scope

1. **What does a user see change?** CQL results, error messages, defaults,
   config, metrics, system tables, performance. A visible change needs docs
   and a line in the PR description; docs that stop being true are fixed or
   removed.
2. **Root cause or defense?** Name the line where the behaviour first goes
   wrong. A check where the symptom shows up is a defense layer: fine as an
   extra, but say which of the two the change is.
3. **Every path into the changed code.** Same bug, all paths, same PR. Pairs
   that usually differ: prepared and unprepared, local and forwarded,
   coordinator and replica, tablets and vnodes, LWT and plain writes, strong
   and eventual consistency, CQL and Alternator, first page and later pages,
   top-level and nested expressions. Special paths that copy shared logic (a
   dump statement, an internal query) miss later fixes: move them onto one
   helper. Shortcut predicates miss variants.
4. **A new path repeats every check of the old one.** Validation,
   guardrails, paging, limits, routing. A cheap way to find the gaps: run the
   existing test suites with the new path as the default. Where the new path
   differs by design, write it down.
5. **Is there a realistic scenario?** A fix for a case no real workload or
   operator hits gets rejected. Say how a real cluster gets there.
6. **Does it need to exist, in this shape?** If a maintainer may answer
   "Won't Do", ask before building. Is it the best solution, or just one
   that works? Reuse existing machinery over a parallel implementation, even
   at a few cycles' cost. Every guard and special case
   must be needed: is the combination it rejects planned to be supported, and
   does rejecting it break normal clients?
7. **Is it worth it?** For an optimization: the gain on production-like
   data, the memory cost, and a fair benchmark. Prefer a better worst case
   over a better average when the bad case outlives its cause.

### B. Seastar: shards, stalls, memory

8. **Which shard owns the state?** Cross-shard access goes through
   `sharded<>`, `invoke_on` or `foreign_ptr`. No shared mutable state.
9. **Can it stall the reactor?** Every loop over user-sized or cluster-sized
   input yields (`maybe_yield`), including parsing, the error path, and
   freeing large structures. Nothing quadratic on user input, including
   hidden linear costs inside a loop.
10. **Is recursion bounded on the stack it runs on?** A `seastar::async`
    thread has a 128 KiB stack by default (`seastar/core/thread.hh`), not
    8 MiB. User-controlled nesting (types, expressions) needs a depth limit
    with a clear error.
11. **Large contiguous allocations.** Anything above 128 KiB warns
    (`large_memory_allocation_warning_threshold`, `db/config.cc`). Use
    `utils::chunked_vector` or another fragmented container for input-sized
    data. Count the copies of large user values too (linearizing,
    concatenation).
12. **Is concurrency bounded?** Concurrent work needs a limit on how much
    memory it can hold (a semaphore, the reader concurrency semaphore).
    Backpressure works both ways: a bounded request queue with unbounded
    responses piling up still runs out of memory.
13. **Which scheduling group runs it?** User requests, maintenance,
    streaming, compaction and gossip have their own groups. Work handed to a
    shared long-lived thread or queue runs in the caller's group, not in the
    group of whoever started the thread.
14. **Background work and shutdown.** Background work is tracked (a gate),
    `stop()` or `close()` waits for it, and unbounded work can be aborted.
    Services drain in the right order. What happens if startup aborts
    halfway, a restart follows an interrupted one, or a request arrives
    during shutdown? No new global singleton: pass dependencies in, so the
    start order stays visible.
15. **Errors on the hot path.** An error that is part of normal operation
    (validation failures, a connection storm) should not throw on the data
    path: return it (`std::expected`, `utils/result.hh`). Throwing is fine
    for errors that almost never happen.

### C. Cluster: upgrades, raft, stored state

16. **Mixed-version cluster.** During a rolling upgrade, old and new nodes
    talk to each other. Is new behaviour gated by a cluster feature
    (`gms/feature_service.hh`), and is the behaviour right before the
    feature is enabled? What does an old node do with a new RPC field, verb
    or system table column? Can the cluster roll back? Does it need an
    upgrade test?
17. **Wire and on-disk formats.** IDL, sstables, commitlog, system table
    schemas. A type change in an IDL struct is a format change unless both
    types encode the same. New IDL fields get a `[[version X.Y]]`
    annotation.
18. **Data that exists today.** Never reject what is accepted today:
    existing schemas and stored values that break a new rule keep working.
    Nodes that already hit the bug: does the fix help them, or do they need
    a manual step and a release note?
19. **Can a user value crash nodes later?** Values stored in schema, group0
    or the commitlog are read back at flush, load or replay, often on every
    replica. Validate at write time, where a bad value is a CQL error and not
    an abort.
20. **Where does the state live?** Group0 (raft), a local system table,
    gossip, or memory only. After a group0 change, does a reader on another
    node need a `read_barrier` first? A group0 apply or snapshot load must
    not do slow synchronous I/O: a joining node's `read_barrier` has a
    timeout.
21. **Windows of empty or stale state.** Between startup steps (sstable
    load to the first group0 reload, schema merge to state reload) in-memory
    state can be empty. Prefer a positive list ("known safe") so an empty set
    means unsafe. A snapshot copied from live state freezes it. Does the fix
    turn off an existing optimization?
22. **Committed or applied?** A raft-backed (strongly consistent) write
    returns when committed, not when applied everywhere. Readers and tests
    that need applied state wait for it.
23. **Topology changes during the operation.** Tablet migration, node join,
    decommission, replace. Is the replica set read once and then stale? The
    tablet balancer can move a tablet between two steps of a test that
    samples placement.
24. **Failures.** A node down, a timeout, a network partition, multiple DCs.
    Are retries safe to repeat? Which error does the client get?
25. **Races.** A decision made on a snapshot and acted on later without a
    recheck (read, then unconditional delete). State captured at request
    start (current keyspace, schema version, prepared entry) used after
    suspension points. A table or keyspace dropped or altered during the
    operation.

### D. Interfaces

26. **Cassandra compatibility.** Does Cassandra do the same, and in which
    version? A difference is deliberate and written down.
27. **Errors.** The same exception type as the neighbouring checks
    (`invalid_request_exception`, `configuration_exception`), so the client
    gets the right CQL error code, never SERVER_ERROR for a user mistake.
    The same goes for timeouts and lost connections on internal RPCs. A
    specific message, the same on every path. A refactor of validation or
    resolution keeps the specific message on the top-level path. Two
    different failures get two messages a user can tell apart. "Not yet
    supported" promises support: use it only when it is planned.
28. **Broken invariants.** What happens when "can't happen" happens? Abort
    (`SCYLLA_ASSERT`) only when going on would corrupt data;
    otherwise `on_internal_error()` or `throwing_assert()` (`utils/assert.hh`).
    Log only what can safely be ignored.
29. **Untrusted input.** Input from CQL, REST, Alternator or a config
    template is validated, and escaped where it is inserted into another
    language (an LDAP filter, a URL, JSON). Sizes are checked before buffers
    are used.
30. **Name and overload resolution.** Several exact matches are ambiguous;
    duplicates of one candidate are not. A new built-in name or cast can
    hide a user function with the same name.
31. **Config.** Is an option needed at all? Its default; the same on all
    nodes or per node; live (`liveness::LiveUpdate`); a bad value at boot
    fails startup, a bad live value keeps the last valid one. A new default
    for a table property applies to new tables only, and an explicit value
    wins. Tools that read the config (`scylla sstable`) know the option.
32. **New limits.** Applied to user input only, or also to internal queries,
    schema load and tools? In a mixed or forwarded setup, whose value
    applies? A cluster-wide config option is simpler than passing the
    sender's value along.
33. **Observability.** Metrics: name, labels, per-shard or per-table cost,
    and the metrics docs. Tracing for the request path. Log levels that do
    not flood.
34. **Other surfaces.** REST API, nodetool, virtual tables, `DESCRIBE`
    output, the Python driver and cqlsh.

### E. Tests and delivery

35. **Test layer.** cqlpy for CQL behaviour, even when Scylla-only. Boost
    only for what cqlpy cannot reach. Cluster tests only for more than one
    node. The test owns its signal: a per-table metric, not a node-wide one
    that other tests move too. A case that an existing error injection can
    reach gets a test. A case that needs a new injection is never dropped
    silently: ask the user, who decides per case whether to add one. Tests
    check observable state, never log lines. One test per distinct code
    path, not one per input value that takes the same path.
36. **Reproducer first.** For a bug: a commit with the test marked as
    expected to fail, then the fix removes the marker. The repro run on a
    binary without the fix. Preparatory commits (item 37) may come before
    the reproducer or between it and the fix.
37. **Commit split.** Every commit builds and passes its tests. Where does
    each part of the design go? Aim for a series that tells a story: a few
    small preparatory commits (a refactor with no behaviour change,
    plumbing, making a service safe for a new order), each reviewable on its
    own, so that the fix at the end is small and obvious. Reproducer plus
    fix is the minimum shape, not the default one; not every change needs
    preparation. No commit changes code that a later commit of the same
    series changes again. Moving tests between suites (scylla-dtest to
    `test/cluster/dtest` to `test/cluster`) goes in stages: copy unchanged
    and disabled, then change, then enable. No commit leaves a test in
    neither suite.
38. **Backport.** By severity: a crash any user with MODIFY can trigger,
    yes; performance, mostly no. Which branches have the bug, and do the
    code path and the test exist there? A backport wants a small change.
39. **Performance.** Is it on a hot path? If the design is about speed,
    plan the benchmark before coding.

## Where to read

The `docs/dev/` files per area. Read the ones for the area before step 3 of
a design.

| Area | `docs/dev/` files |
|---|---|
| Raft, group0, topology | `raft-in-scylla.md`, `topology-over-raft.md` |
| Tablets | `tablets-routing-v2.md`, `size-based-load-balancing.md` |
| Strong consistency | `strong_consistency.md` |
| RPC and IDL | `IDL.md`, `advanced_rpc_compression.md` |
| Reads and paging | `paged-queries.md`, `reverse-reads.md`, `reader-concurrency-semaphore.md`, `row_cache.md`, `mvcc.md` |
| Aggregation | `parallel_aggregations.md` |
| Storage formats | `sstable-scylla-format.md`, `sstables-directory-structure.md`, `commitlog-file-format.md`, `tombstone.md`, `timestamp-conflict-resolution.md` |
| System tables | `system_keyspace.md`, `system-keyspaces.md`, `system_schema_keyspace.md`, `virtual-tables.md` |
| Repair, node ops, hints | `repair_based_node_ops.md`, `row_level_repair.md`, `hinted_handoff_design.md` |
| Views, indexes, CDC | `secondary_index.md`, `view-building-coordinator.md`, `cdc.md`, `vector_index.md` |
| Scheduling, isolation | `isolation.md`, `service_levels.md`, `per-partition-rate-limit.md` |
| CQL protocol and types | `protocols.md`, `protocol-extensions.md`, `cql-extensions-internal.md`, `cql3-type-mapping.md` |
| Observability | `metrics.md`, `tracing.md`, `logging.md` |
| Background tasks | `task_manager.md` |
| Config | `cluster_config.md` |
| Backports | `backport.md` |
| Where code lives | `modules.md`, `repository_layout.md` |

Also `docs/dev/review-checklist.md` (what maintainers check) and
`.github/instructions/cpp.instructions.md` (Seastar patterns).
