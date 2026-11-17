# Feedback Flow

The checklists of this plugin learn from feedback in two stages. The first
stage is private and stays on the machine. The second changes a checklist,
and the user reviews it before it is committed.

`<skill>` below is `design` or `code-checks`.

## Stage 1: record (`/dsz-scylladb:<skill> feedback <note>`)

1. Work out what the feedback says was missed or flagged wrongly, and which
   checklist item or check should have caught it, or that none did.
2. Add an entry at the top of the skill's local log,
   `~/.claude/dsz-scylladb/feedback/<skill>.md`. Create the file if it is
   missing. An entry has:
   - a `###` heading: the date, the source (PR, Jira, who), a few words on
     the lesson
   - the quote
   - what was missed or flagged wrongly
   - the item or check it supports
3. Apply the lesson for the rest of the session.

The local log may hold real code, names, quotes, and ticket and PR numbers.
Never write an entry into the skill's own files: the installed copy is
replaced on every update, and the repository is shared.

When a skill runs, it reads its local log. Entries without a "Promoted" line
count as items.

Feedback given through `/dsz:design feedback` or `/dsz:review feedback`,
while a checklist of this plugin is in use, is recorded the same way, in the
log of this plugin.

## Stage 2: promote (`/dsz-scylladb:<skill> promote`)

1. Find the user's clone of this repository. Its path is the one line in
   `~/.claude/dsz-scylladb/clone-path`. If the file is missing, ask the user
   for the path and write it there.
2. For each log entry without a "Promoted" line, decide whether it changes
   the checklist. If it teaches something new, add or reword an item in the
   clone: `skills/design/design-checklist.md` for `design`,
   `skills/code-checks/code-checks.md` for `code-checks`. Keep the numbering,
   and fix the numbers that refer to it. The code checks point at design
   checklist items by number.
3. Write the item as a class of problem. Nothing from the log goes into the
   clone: no names of people or clients, no quotes, no ticket or PR numbers,
   no internal numbers (fleet or client data), and no description of a
   specific bug, fixed or not.
4. If the lesson is not specific to ScyllaDB, say so. It may also belong in
   the general checklist of the `dsz` plugin, which has its own log and its
   own `promote`.
5. Add a line under the heading of the log entry:
   `Promoted: <date>, <item or check>`. For an entry that changes nothing:
   `Promoted: <date>, no change`.
6. Show the diff of the clone. Do not commit or push: the user reviews the
   diff first. Leave `version` in `plugin.json` as it is: the maintainer
   raises it at a release.
