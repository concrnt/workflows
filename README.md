# concrnt/workflows

concrnt org の各 repo から呼び出す共通(reusable)ワークフロー置き場。

## dependabot-auto-merge.yml

Dependabot の PR を自動マージする。

- semver-major の bump は `dependencies:major` ラベルを付けて人手に残す
- それ以外は `gh pr merge --auto --squash` で auto-merge を有効化する
  (ブランチ保護の required status check が通った時点で merge される)
- `approve: true` の repo では先に approve する(review 必須ブランチ用)

### 前提(repo 側)

1. default branch に required status check を含むブランチ保護があること。
   **required check が無いと `gh pr merge --auto` は CI を待たずその場で merge する。**
2. Settings > General > Pull Requests > "Allow auto-merge" が ON
   (`gh api -X PATCH repos/concrnt/<repo> -F allow_auto_merge=true`)
3. `approve: true` を使う repo は Settings > Actions > General >
   "Allow GitHub Actions to create and approve pull requests" が ON
   (`gh api -X PUT repos/concrnt/<repo>/actions/permissions/workflow -F default_workflow_permissions=read -F can_approve_pull_request_reviews=true`)

### caller(各 repo の `.github/workflows/dependabot-auto-merge.yml`)

```yaml
name: Dependabot auto-merge

on:
  pull_request:
    types: [opened, reopened, synchronize, ready_for_review]

permissions:
  contents: write
  pull-requests: write
  issues: write

concurrency:
  group: dependabot-auto-merge-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  dependabot:
    if: github.event.pull_request.user.login == 'dependabot[bot]'
    uses: concrnt/workflows/.github/workflows/dependabot-auto-merge.yml@main
    secrets: inherit
    # review 必須の repo だけ:
    # with:
    #   approve: true
```

### inputs

| name | default | 説明 |
|---|---|---|
| `merge-method` | `squash` | `squash` / `merge` / `rebase` |
| `allow-major` | `false` | true なら semver-major も merge する |
| `major-label` | `dependencies:major` | 人手に残す PR に付けるラベル |
| `approve` | `false` | auto-merge の前に approve する |

### 既存 PR への適用

caller 追加前に open だった PR はイベントが無いので動かない。
`@dependabot rebase` をコメントして synchronize を起こす
("already up to date" なら `@dependabot recreate`)。
