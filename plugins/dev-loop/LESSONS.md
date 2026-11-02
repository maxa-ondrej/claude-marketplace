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

## Stronger lessons beat weaker ones

`Review level` says how much weight a lesson carries. From strongest to weakest:

1. `second lvl` — a second-level reviewer asked for it
2. `first lvl` — a first-level reviewer asked for it
3. `self review` — the user wrote it for themselves
4. `bot / AI` — a bot raised it

Use this order every time, not only when two lessons disagree:

- **A conflict is settled by level.** The stronger lesson wins. Say which one you followed and why.
- **Read strongest first** when there are more lessons than you can hold. A `second lvl` lesson you
  missed becomes a review comment; a `bot / AI` lesson you missed usually does not.
- **`reviewer` grades by level too.** A broken `second lvl` lesson is a Must Fix. A broken
  `bot / AI` lesson is a Nit, unless the code is wrong for some other reason as well.
- `AGENTS.md` still beats every level.

`AGENTS.md` has control if a lesson does not agree with it.

## Add a lesson

One row for one lesson. Only from `/review --pr`. Only after the user approves these words.

```bash
notion db add <db-id> \
  "Name=<short rule, one line>" \
  "Category=<a category that exists>" \
  "Repo=<group>/<name>" \
  "Lesson=<the full rule>" \
  "Authors=<name>[,<name>]" \
  "Review level=<bot / AI | self review | first lvl | second lvl>" \
  "Date=$(date +%F)"
```

Use a category that exists. A new category needs the user to agree first.

`Authors` is everyone who asked for the rule. `Review level` is the **highest** level among them:
a lesson from the user and a second-level reviewer together is `second lvl`. Use `self review` only
when the user is the one author, and `bot / AI` only when every author is a bot.

`Created at` and `Last updated` fill in on their own. Do not set them.

To make a lesson better, change that row. Do not add a row that is almost the same.
