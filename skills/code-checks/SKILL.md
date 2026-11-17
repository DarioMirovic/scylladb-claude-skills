---
name: code-checks
description: >
  ScyllaDB and Seastar code checks for the dsz:review skill (lifetimes in
  continuations and coroutines, futures, reactor stalls, allocations, error
  types, invariants). This skill should be used together with dsz:review
  whenever a ScyllaDB or Seastar PR is reviewed, by hand or with
  /code-review, and when a ScyllaDB review missed something or flagged
  something wrongly.
argument-hint: "feedback <note> | promote | help"
---

# dsz-scylladb:code-checks

The ScyllaDB and Seastar checks for `dsz:review`. This skill has no review
procedure of its own, and it needs the `dsz` plugin.

Files next to this one:

- `code-checks.md`: 20 line-level checks for Seastar and ScyllaDB code.

## Usage

```
/dsz:review <PR URL> [--light]            review a ScyllaDB PR, with these checks
/dsz-scylladb:code-checks feedback <note> record a missed or wrongly flagged finding
/dsz-scylladb:code-checks promote         turn local log entries into check changes
/dsz-scylladb:code-checks help            print this card
```

A review starts from `/dsz:review`. It finds these checks on its own when the
PR is a ScyllaDB or Seastar one.

## In a review

1. The review procedure, the hard rules and the "before flagging" rules are
   the ones of `dsz:review`. Invoke it if it is not loaded yet. If it is not
   installed, say so and stop.
2. Use the checks in `code-checks.md` of this skill in place of the general
   "Checks" section of `dsz:review`.
3. Read the local log `~/.claude/dsz-scylladb/feedback/code-checks.md` at the
   start of the review, if it exists. Its entries without a "Promoted" line
   count as checks.
4. "Design checklist item N" in the checks points at
   `../design/design-checklist.md`.

## Feedback

For `feedback` and `promote`, follow `${CLAUDE_PLUGIN_ROOT}/FEEDBACK.md`, the
`FEEDBACK.md` in this plugin's root directory. `dsz:review` is not needed for
these two.

- Local log: `~/.claude/dsz-scylladb/feedback/code-checks.md`. An entry has
  the date, the source (PR, who), the quote, what was missed or flagged
  wrongly, and the check it supports.
- On `promote`, a new or reworded check goes to `code-checks.md` in the
  clone. Keep the checks numbered.
