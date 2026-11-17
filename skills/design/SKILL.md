---
name: design
description: >
  ScyllaDB design checklist for the dsz:design skill (shards and stalls,
  mixed-version clusters, group0, tablets, all code paths, CQL compatibility,
  config, metrics, test layer, backports). This skill should be used together
  with dsz:design whenever a ScyllaDB change is designed, planned or thought
  through, when a ScyllaDB PR, branch or plan gets a design or architecture
  review, and when a ScyllaDB design missed something.
argument-hint: "<task, Jira key or description> | review <PR|branch|plan file> | feedback <note> | promote | help"
---

# dsz-scylladb:design

The ScyllaDB checklist for `dsz:design`. This skill has no design procedure
of its own, and it needs the `dsz` plugin.

Files next to this one:

- `design-checklist.md`: the numbered ScyllaDB checklist, and the `docs/dev/`
  files to read per subsystem.

## Usage

```
/dsz-scylladb:design <task>             design a ScyllaDB change, no code yet
/dsz-scylladb:design review <target>    check a PR, branch or plan at the design level
/dsz-scylladb:design feedback <note>    record a design miss in the local log
/dsz-scylladb:design promote            turn local log entries into checklist changes
/dsz-scylladb:design help               print this card
```

`/dsz:design` gives the same result in a ScyllaDB checkout: it finds this
checklist on its own.

## Design and review

1. Invoke the `dsz:design` skill if it is not loaded yet, with the arguments
   this skill was called with. If it is not installed, say so and stop.
2. Follow its procedure and its hard rules, with `design-checklist.md` of this
   skill in place of its general checklist.
3. "The project's design docs" in that procedure are the `docs/dev/` files in
   the "Where to read" table of the checklist. Read the ones for the area
   before going through the items.
4. Read the local log `~/.claude/dsz-scylladb/feedback/design.md` before the
   checklist, if it exists. Its entries without a "Promoted" line count as
   items.
5. Claims about what Cassandra does are checked like claims about the code, or
   marked "unverified".
6. Tickets are Jira keys (`SCYLLADB-<n>`).

## Feedback

For `feedback` and `promote`, follow `${CLAUDE_PLUGIN_ROOT}/FEEDBACK.md`, the
`FEEDBACK.md` in this plugin's root directory. `dsz:design` is not needed for
these two.

- Local log: `~/.claude/dsz-scylladb/feedback/design.md`. An entry has the
  date, the source (PR, Jira, who), the quote, what was missed, and the
  checklist item that should have caught it, or that none did.
- On `promote`, a new or reworded item goes to `design-checklist.md` in the
  clone. Keep the items numbered and fix the references to them, also in
  `../code-checks/code-checks.md`.
