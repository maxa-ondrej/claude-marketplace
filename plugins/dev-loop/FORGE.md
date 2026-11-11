# Forge commands

dev-loop works against GitHub or GitLab. Nothing in the skills assumes one — they name an
operation, and this file says how to run it on the forge the repo actually uses.

## Detect the forge

```bash
git remote get-url origin
```

`github.com` → GitHub, `gh`. `gitlab.com` or a self-hosted GitLab host → GitLab, `glab`. Neither,
or both CLIs missing? Ask the user rather than guessing — every command below writes to someone
else's view of the PR.

Terminology differs and the skills use the GitHub words: a GitLab **MR** is a PR, a **discussion**
is a thread, and an MR is identified by its `iid`, not a number.

## Sign everything you post

Everything this plugin posts to a forge is signed. A colleague who reads it must be able to see
that a person did not type it. They answer it differently when they know.

**A comment — a thread reply or a plain note** — ends with this, after a blank line:

````markdown

---
_Sent using `@Claude`_
````

**A PR or MR description** ends with this instead:

```markdown

---
Co-Authored-By: Claude Code
```

A description is the work's own record, so it names the author the way a commit trailer does. A
comment is one side of a conversation, so it says who is speaking.

Three rules about both:

- **The backticks are not decoration.** They stop the forge from turning `@Claude` into a mention.
  On gitlab.com, `@Claude` is a real person — Francisco Claude-Faust — and a bare mention would
  notify a stranger on every single reply. Inside backticks it renders as code and links to
  nobody.
- **Never write a bare `@` in anything you post.** Not in a reply, not in a note, not in a
  description. A name that looks like a bot on one forge is somebody's account on another.
- **Not on commits.** A commit message carries no footer at all, not even `Co-Authored-By`. That
  rule is in `/dev-loop:ship` and this does not change it. Only the PR or MR description gets the
  trailer.

Count the footer as part of the text the user approves. Show it in the draft, so nobody is
surprised by what went up.

## Operations

`gh` substitutes `{owner}` and `{repo}` in REST paths; `glab` substitutes `:fullpath`. Neither
substitutes into GraphQL, so GitHub's thread queries need the owner and name passed explicitly —
`gh repo view --json owner,name -q '.owner.login, .name'`.

Each Bash call is a fresh shell. Substitute the real number, id, and text into these commands
rather than setting variables and expecting them to survive.

### The PR for the current branch

```bash
gh pr view --json number,url -q '.number, .url'                 # GitHub
glab mr view -F json --jq '.iid, .web_url'                      # GitLab
```

No PR/MR for the branch: both exit non-zero. That is a stop condition, not an error to work around.

### Read review comments

```bash
gh api repos/{owner}/{repo}/pulls/<number>/reviews              # GitHub — review bodies
gh api repos/{owner}/{repo}/pulls/<number>/comments             # GitHub — inline comments
glab mr view <iid> --comments                                   # GitLab — readable form
```

### Threads, with the ids needed to reply or resolve

GitHub keeps resolution in GraphQL only:

```bash
gh api graphql -f query='
  query($owner:String!,$repo:String!,$pr:Int!) {
    repository(owner:$owner, name:$repo) {
      pullRequest(number:$pr) {
        reviewThreads(first:100) { nodes {
          id isResolved isOutdated path line
          comments(first:1) { nodes { databaseId author { login } body } }
        } }
      }
    }
  }' -F owner=<owner> -F repo=<repo> -F pr=<number>
```

GitLab returns the same shape from the discussions API:

```bash
glab api "projects/:fullpath/merge_requests/<iid>/discussions" --paginate
```

Per discussion: `id` is the thread, `notes[0]` the original comment (with
`position.new_path` / `position.new_line` for file:line), and `notes[].resolvable` / `resolved`
say whether it can be and already is resolved. System notes are not resolvable — skip them.

### Reply into a thread

```bash
gh api "repos/{owner}/{repo}/pulls/<number>/comments/<databaseId>/replies" -f body='<message>'
glab api --method POST "projects/:fullpath/merge_requests/<iid>/discussions/<id>/notes" --raw-field "body=<message>"
```

### Resolve a thread

```bash
gh api graphql -f query='mutation($t:ID!){ resolveReviewThread(input:{threadId:$t}){ thread { isResolved } } }' -f t=<thread-id>
glab api --method PUT "projects/:fullpath/merge_requests/<iid>/discussions/<id>" --field resolved=true
```

### Comment on the PR itself (not a thread)

```bash
gh pr comment --body '<message>'
glab mr note create <iid> -m '<message>'              # `glab mr comment` is an alias
```

### Open a PR

```bash
gh pr create --title '<title>' --body '<body>'
glab mr create --title '<title>' --description '<body>'
```

### Set the PR description

```bash
gh pr edit <number> --body-file <file>
glab mr update <iid> --description "$(cat <file>)"
```

Write the body to a file first. Both CLIs mangle a long body passed inline.

### Who you are posting as

```bash
gh api user --jq .login
glab api user --jq .username
```

Compare it against a thread's author to tell the user's own comments from a colleague's.

### Reviewer signals

Code owners for the changed files:

```bash
# GitHub: owners are requested on the PR, plus the CODEOWNERS file itself
gh api "repos/{owner}/{repo}/pulls/<number>/requested_reviewers" --jq '.users[].login'
# GitLab: the MR's own code-owner approval rules
glab api "projects/:fullpath/merge_requests/<iid>/approval_state" \
  | jq -r '.rules[] | select(.rule_type=="code_owner") | .eligible_approvers[].username'
```

Both can come back empty. Fall back to the `CODEOWNERS` file — `CODEOWNERS`, `.github/CODEOWNERS`,
`.gitlab/CODEOWNERS` or `docs/CODEOWNERS` — matched against the changed paths.

Who reviews your work most:

```bash
gh search prs --author=@me --state=closed --json number --jq '.[].number'   # then read each PR's reviews
gh pr list --author=@me --state=merged --limit 30 --json reviews --jq '.[].reviews[].author.login'
glab api "projects/:fullpath/merge_requests?author_username=<me>&state=merged&per_page=30" \
  | jq -r '.[].reviewers[]?.username'
```

How much review work a person already has:

```bash
gh search prs --review-requested=<user> --state=open --json number --jq 'length'
glab mr list --reviewer=<user> --output json --jq 'length'
```

Assign reviewers:

```bash
gh pr edit <number> --add-reviewer "user1,user2"
glab mr update <iid> --reviewer "user1,user2"
```

### CI

```bash
gh run watch ; gh run view --log-failed                         # GitHub
glab ci status --live ; glab ci view                            # GitLab
```
