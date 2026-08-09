---
name: merge-to-test
description: 将当前开发分支合并到本地 feature/test，推送 origin/feature/test，并发布 Go 测试环境。
---

# Merge To Test

把当前开发分支发布到 Go 测试环境：

`source branch -> feature/test -> push -> deploy`

## Flow

1. Preflight:

   - Confirm repo is `fincore`.
   - Record current branch as `<source-branch>`.
   - Stop if already on `feature/test`.
   - Check worktree; if dirty, use `git-commit` skill first (it will commit and push `<source-branch>`), then continue.

2. Confirm before checkout, pull, merge, push, or deploy:

   - source: `<source-branch>`
   - target: `feature/test`
   - push: `origin/feature/test`
   - deploy: `.githooks/post-merge --after-push --source-branch <source-branch>`
   - deploy root: `$FINCORE_DEPLOY_ROOT`, or `../fincore-deploy` if unset

3. Run:

```bash
git checkout feature/test
git pull origin feature/test
git merge <source-branch>
git push origin feature/test
.githooks/post-merge --after-push --source-branch <source-branch>
```

Deployment must run after push because the deploy script pulls remote `feature/test`.
Do not call interactive deploy scripts such as `bin/fincore-deploy`.

## Conflicts

If `git merge <source-branch>` conflicts:

1. Keep conflict state; do not push, deploy, abort, rebase, or squash.
2. After the user fixes conflicts, verify no conflict markers remain.
3. Run `git add <resolved-files>` and `git commit --no-edit`.
4. Continue with:

```bash
git push origin feature/test
.githooks/post-merge --after-push --source-branch <source-branch>
```

## Logs

- Error logs: `.githooks/logs/error_log/{yyyy_mm_dd}-{hh_mm_ss}.log`
- Deploy log: `.githooks/logs/deploy_log/post-merge-test-deploy.log`

Error logs must include: failed command, source branch, current branch, timestamp,
exit output, and relevant `git status`.

## Stop Rules

- `checkout` or `pull` failed: do not merge, push, or deploy.
- `merge` conflicted: stop until the user fixes conflicts.
- `push` failed: keep local `feature/test`; do not deploy.
- `deploy` failed after push: report that local and remote `feature/test` were updated, but Go deploy failed.

## Success Output

Report that:

- `<source-branch>` was merged into local `feature/test`
- `origin/feature/test` was updated
- Go test deployment completed
- integration testing can start

## Rules

- Use this exact merge workflow; do not use rebase, squash, or fast-forward-only.
- Deploy only via `.githooks/post-merge --after-push --source-branch <source-branch>`.
- Do not auto-create an MR.
