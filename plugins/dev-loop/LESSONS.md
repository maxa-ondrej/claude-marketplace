# Review lessons

Durable reviewer preferences — the rules a reviewer must not have to give two times. They live in
a Notion database, not in the repo. They outlive the branch, and they apply to every clone.

`architect` reads them when it plans. `developer` and `tester` read them when they write.
`reviewer` applies them. `/review --pr` is the only writer, and only when the user approves the
words.

All commands use the `notion` CLI. It is authenticated for one workspace.

## Find the database

```bash
notion auth status
notion search "Review Lessons" --type database -f json
```

Use the database with the title **Review Lessons**. Keep its ID for the other commands.

The CLI is missing, or not authenticated, or the search finds nothing? Say this one time and
continue. No lessons is a normal condition, not an error. Do not stop the work.

Two or more databases with this title? Ask the user which one. Do not guess.

## Find the repository name

```bash
git remote get-url origin
```

Use `<group>/<name>` from the URL, for example `shipmonk-company/monorepo`. Do not use a local
path. Each worktree and each clone of a repository then reads the same lessons.

## Read the lessons

```bash
notion db query <db-id> --filter 'Repo=<group>/<name>' --all -f md
```

Lessons with the repository `(all repos)` apply to every repository. Read them too.

`Name` is the short rule. `Lesson` is the full text. One lesson of 2000 characters or more
continues in the page body — read the page only when you must.

`AGENTS.md` has control if a lesson does not agree with it.

## Add a lesson

One row for one lesson. Only from `/review --pr`. Only after the user approves these words.

```bash
notion db add <db-id> \
  "Name=<short rule, one line>" \
  "Category=<a category that exists>" \
  "Repo=<group>/<name>" \
  "Lesson=<the full rule>" \
  "Reviewer=<name>" \
  "Date=$(date +%F)"
```

Use a category that exists. A new category needs the user to agree first.

To make a lesson better, change that row. Do not add a row that is almost the same.
