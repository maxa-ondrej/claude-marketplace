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

### CI

```bash
gh run watch ; gh run view --log-failed                         # GitHub
glab ci status --live ; glab ci view                            # GitLab
```
