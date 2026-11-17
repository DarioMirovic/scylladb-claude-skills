# ScyllaDB checklists for the dsz skills

A personal collection, not an official ScyllaDB project.

The plugin `dsz-scylladb` adds two checklists to the
[dsz](https://github.com/DarioMirovic/dsz-claude-skills) plugin, which it
needs:

| Skill | What it is |
|---|---|
| `dsz-scylladb:design` | The ScyllaDB design checklist for `/dsz:design`: 39 items, and the `docs/dev/` files to read per subsystem. |
| `dsz-scylladb:code-checks` | 20 Seastar and ScyllaDB line-level checks for `/dsz:review`. |

With both plugins installed, `/dsz:design` and `/dsz:review` pick these
checklists up in a ScyllaDB checkout, in place of their general ones.

## Install

```
/plugin marketplace add DarioMirovic/dsz-claude-skills
/plugin install dsz@dsz
/plugin marketplace add DarioMirovic/scylladb-claude-skills
/plugin install dsz-scylladb@dsz-scylladb
```

## What is in here, and what is not

The checklists name public identifiers and files of the ScyllaDB and Seastar
repositories. They hold no ScyllaDB code.

They take no names of people or clients, no quotes, no ticket or PR numbers,
and no description of a specific bug.

## Feedback

The checklists learn in two stages, described in [FEEDBACK.md](FEEDBACK.md):

1. `/dsz-scylladb:<skill> feedback <note>` records a design miss, or a missed
   or wrongly flagged finding, in a local log. The log may hold real code,
   names, quotes and ticket numbers. It stays on your machine, and the skill
   reads it on every run.
2. `/dsz-scylladb:<skill> promote` turns log entries into checklist changes
   in your clone of this repository, written as a class of problem. You
   review the diff and commit.

`<skill>` is `design` or `code-checks`.

## Local files

| Path | What it is |
|---|---|
| `~/.claude/dsz-scylladb/feedback/design.md` | The local feedback log of the design checklist. |
| `~/.claude/dsz-scylladb/feedback/code-checks.md` | The local feedback log of the code checks. |
| `~/.claude/dsz-scylladb/clone-path` | One line: the path of your clone, used by `promote`. |

## Releases

The plugin has a `version` in `.claude-plugin/plugin.json`. Users get changes
only when that version changes. A PR does not change it. The maintainer raises
it when a set of changes is ready, and users then run
`claude plugin update dsz-scylladb@dsz-scylladb`.

## License

MIT with the Commons Clause, see [LICENSE](LICENSE). In short: free to use,
change and share, also at work. Not to be sold.
