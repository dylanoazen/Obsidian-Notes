# Git Avançado

Além do add-commit-push — entendendo os internos do Git e workflows avançados.

---

## Modelo de Objetos do Git

Tudo no Git é armazenado como objetos em `.git/objects/`:

```
blob    → file content (no filename, just data)
tree    → directory listing (maps names → blobs/trees)
commit  → snapshot: points to a tree + parent commit(s) + metadata
tag     → named reference to a commit (annotated tags)
```

```bash
# Inspect any object
git cat-file -t <hash>    # type
git cat-file -p <hash>    # content

# See the commit graph
git log --oneline --graph --all
```

Um commit não armazena diffs — ele armazena um snapshot completo (tree). O Git calcula diffs sob demanda.

## Refs e HEAD

- **Refs**: ponteiros legíveis por humanos para commits (branches, tags)
- **HEAD**: ponteiro para o branch atual (ou para um commit em estado detached)
- Armazenados como arquivos em `.git/refs/`

```bash
cat .git/HEAD                      # ref: refs/heads/main
cat .git/refs/heads/main           # commit hash
git reflog                         # history of where HEAD has been
```

## Rebase vs Merge

```
Merge:                          Rebase:
  A─B─C (main)                   A─B─C (main)
       \                              \
        D─E (feature)                  D'─E' (feature, replayed)
             \
              M (merge commit)

merge: preserves history, creates merge commit
rebase: linear history, rewrites commits (new hashes)
```

```bash
# Interactive rebase — rewrite last N commits
git rebase -i HEAD~5
# pick, squash, fixup, reword, edit, drop

# Rebase onto main
git checkout feature
git rebase main

# Abort if conflicts are messy
git rebase --abort
```

## Cherry-pick

Aplicar um commit específico em outro branch:

```bash
git cherry-pick <commit-hash>
git cherry-pick A..B              # range
git cherry-pick --no-commit <hash>  # stage without committing
```

## Reset vs Revert

```bash
# Reset — moves HEAD backward (destructive)
git reset --soft HEAD~1     # undo commit, keep changes staged
git reset --mixed HEAD~1    # undo commit, keep changes unstaged
git reset --hard HEAD~1     # undo commit, DELETE changes

# Revert — creates a NEW commit that undoes a previous one (safe)
git revert <hash>
```

## Stash

```bash
git stash                     # save work-in-progress
git stash pop                 # restore and remove from stash
git stash list                # view all stashes
git stash apply stash@{2}     # apply specific stash
git stash drop stash@{0}      # delete a stash
```

## Worktrees

Múltiplos diretórios de trabalho a partir de um único repositório:

```bash
git worktree add ../hotfix hotfix-branch
# now you have two directories, two branches, one repo
git worktree list
git worktree remove ../hotfix
```

## Bisect

Busca binária pelo commit que introduziu um bug:

```bash
git bisect start
git bisect bad                 # current commit is broken
git bisect good v1.0           # this tag was working
# Git checks out middle commit → you test → mark good/bad
git bisect good                # or: git bisect bad
# Repeat until Git finds the culprit
git bisect reset
```

## Hooks

Scripts que executam em eventos do Git (`.git/hooks/`):

```bash
pre-commit      # lint, format, run tests before commit
commit-msg      # validate commit message format
pre-push        # run tests before push
post-merge      # install deps after pull
```

## Configurações Avançadas

```bash
# Sign commits with GPG
git config --global commit.gpgsign true

# Default branch name
git config --global init.defaultBranch main

# Reuse recorded resolution (auto-resolve repeated conflicts)
git config --global rerere.enabled true

# Better diff algorithm
git config --global diff.algorithm histogram
```

## Related

- [[DevOps/CICD/GitHubActions]]
- [[DevOps/CICD/CICD]]

## Recursos

- https://git-scm.com/book/en/v2 (Pro Git — gratuito)
- https://learngitbranching.js.org (interativo)

#### Meus comentários
- 
