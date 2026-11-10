# バイリンガル-GitHub

## 概要
bilingual-github リポジトリは、GitHubのIssueやコメントを翻訳するための再利用可能なGitHub Actionsワークフローを提供します。 このドキュメントでは、以下のワークフローの使用方法について説明します：
- GitHubのIssueの翻訳
- GitHubのコメントの翻訳

## 翻訳の動作
- Issue に **「need translation」** ラベルが付いている場合のみ翻訳が実行されます。
- ラベルが付いている場合：
  - Issue の内容が翻訳される
  - その Issue 内のすべてのコメントが翻訳される
  - Issue やコメントが編集されると、新しい翻訳が実行される
- **「need translation」** ラベルがない場合:
  - Issue やコメントは翻訳されない
  - 翻訳ワークフローは処理をスキップする

## 対象リポジトリで再利用可能なワークフローを使用する手順

#### 1. GitHubのIssueを翻訳

Issueを翻訳するためのワークフローを使用するには、対象リポジトリに以下のファイルを作成してください：
```bash
.github/workflows/translate_issues.yml
```
#### 呼び出しワークフローのコード (Issueの翻訳)
```python
name: Translate GitHub Issues

on:
  issues:
    types:
      - opened
      - edited

jobs:
  call-translate-issues:
    uses: rimo/bilingual-github/.github/workflows/translate-issues.yml@main
    secrets:
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
    with:
      issue_number: ${{ github.event.issue.number }}
```
#### 2. GitHubのコメントを翻訳
コメントを翻訳するためのワークフローを使用するには、対象リポジトリに以下のファイルを作成してください：
```base
.github/workflows/translate_comments.yml
```
#### 呼び出しワークフローのコード (コメントの翻訳)
```python
name: Translate Issue Comments

on:
  issue_comment:
    types:
      - created
      - edited

jobs:
  call-translate-comments:
    uses: rimo/bilingual-github/.github/workflows/translate-comments.yml@main
    secrets:
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
    with:
      issue_number: ${{ github.event.issue.number }}
      comment_id: ${{ github.event.comment.id }}
```
    
#### 3. .mdファイルを翻訳
推奨構成では、再利用ワークフローを `output_mode: artifact` で呼び出し、GitHub App のシークレットは**渡しません**。
再利用ワークフローはリポジトリを読み取るだけで、翻訳結果を `translations` という名前の artifact としてアップロードします。
コミットと push は、呼び出し側ワークフローの `push-translations` ジョブが自リポジトリに限定したトークンで行います。

- 再利用ワークフローは `contents: read` で動作し、push 用の認証情報を受け取りません。
- `push-translations` は `actions/create-github-app-token` で、現在のリポジトリだけに限定した GitHub App インストールトークン
  (`repositories: ${{ github.event.repository.name }}`, `permission-contents: write`) を発行します。
- App トークンで push したコミットでは PR の CI が再実行されます。`GITHUB_TOKEN` での push では実行されません。
- コミットの author は `rimo-translation-bot[bot]` のままにしてください。両言語のファイルが同じ PR で変更されたとき、
  翻訳処理はこの名前でボットの翻訳と人の編集を区別します。
- 翻訳ジョブに `secrets: inherit` は使わず、`OPENAI_API_KEY` だけを渡してください。

`translations` artifact の中身:
- `files/<リポジトリ相対パス>`: 追加・変更されたファイル
- `deleted-files`: 削除すべきリポジトリ相対パスの NUL 区切りリスト (削除された原文に対応する翻訳や、`foo.md` → `foo.en.md` のリネーム元など)

```base
.github/workflows/translate-markdown.yml
```
```python
name: Trigger Markdown Translation
 
on:
  pull_request:
    types: [opened, synchronize, reopened, labeled, unlabeled]
    paths:
      - '**.md'

# Prevent concurrent runs for the same PR
concurrency:
  group: translation-${{ github.event.pull_request.number }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  debug-info:
    runs-on: ubuntu-latest
    steps:
      - name: Debug Information
        run: |
          echo "Event Name: ${{ github.event_name }}"
          echo "Ref: ${{ github.ref }}"
          echo "PR Number: ${{ github.event.number }}"
          echo "Base Branch: ${{ github.base_ref }}"
          echo "Head Branch: ${{ github.head_ref }}"
          echo "Commit Hash: ${{ github.sha }}"

  check-markdown-changes:
    needs: debug-info
    runs-on: ubuntu-latest
    outputs:
      has_changes: ${{ steps.check.outputs.has_changes }}
      is_initial_setup: ${{ steps.check.outputs.is_initial_setup }}
      changed_files: ${{ steps.check.outputs.changed_files }}
      deleted_files: ${{ steps.check.outputs.deleted_files }}
      skip_translation: ${{ steps.check.outputs.skip_translation }}
      is_pr: ${{ steps.check.outputs.is_pr }}
      pr_number: ${{ steps.check.outputs.pr_number }}
      target_branch: ${{ steps.check.outputs.target_branch }}
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 0  

      - name: Check for changes
        id: check
        run: |
          # Check if this is a PR event and if it has the translation label
          if [[ "${{ github.event_name }}" == "pull_request" ]]; then
            HAS_LABEL=$(echo '${{ toJson(github.event.pull_request.labels) }}' | jq -r '.[] | select(.name == "needs_markdown_translation") | .name')
            
            if [[ "$HAS_LABEL" != "needs_markdown_translation" ]]; then
              echo "should_translate=false" >> $GITHUB_OUTPUT
              echo "skip_reason=missing_label" >> $GITHUB_OUTPUT
              echo "skip_translation=true" >> $GITHUB_OUTPUT
              echo "has_changes=false" >> $GITHUB_OUTPUT
              echo "Translation label 'needs_markdown_translation' not found on PR"
              exit 0
            fi
          fi
          
          # Check if commit message contains [skip-translation] or is a merge commit
          COMMIT_MSG='${{ github.event.head_commit.message || github.event.pull_request.title }}'
          if [[ "$COMMIT_MSG" == *"[skip-translation]"* ]] || [[ "$COMMIT_MSG" == "Merge pull request"* ]] || [[ "$COMMIT_MSG" == "Merge branch"* ]]; then
            echo "skip_translation=true" >> $GITHUB_OUTPUT
            echo "has_changes=false" >> $GITHUB_OUTPUT
            echo "Skipping translation due to [skip-translation] flag or merge commit"
            exit 0
          fi
          
          echo "skip_translation=false" >> $GITHUB_OUTPUT
          
          # This is always a PR event now
          echo "is_pr=true" >> $GITHUB_OUTPUT
          echo "pr_number=${{ github.event.number }}" >> $GITHUB_OUTPUT
          echo "target_branch=${{ github.head_ref }}" >> $GITHUB_OUTPUT
          
          # Get changed files from the PR
          CHANGED_FILES=$(git diff --name-only origin/${{ github.base_ref }}...HEAD | grep -E '\.md$|\.en\.md$|\.ja\.md$' || true)
            
          # Remove duplicates and format
          CHANGED_FILES=$(echo "$CHANGED_FILES" | tr ',' '\n' | sort -u | tr '\n' ',' | sed 's/,$//')
          
          echo "Changed markdown files: $CHANGED_FILES"
          
          # Check for deleted markdown files in the PR
          DELETED_FILES=$(git diff --name-only --diff-filter=D origin/${{ github.base_ref }}...HEAD | grep -E '\.md$|\.en\.md$|\.ja\.md$' || true)
          DELETED_FILES=$(echo "$DELETED_FILES" | tr '\n' ',' | sed 's/,$//')
          
          echo "Deleted files: $DELETED_FILES"
          
          if [[ -n "$CHANGED_FILES" ]] || [[ -n "$DELETED_FILES" ]]; then
            echo "has_changes=true" >> $GITHUB_OUTPUT
          else
            echo "has_changes=false" >> $GITHUB_OUTPUT
          fi
          
          echo "changed_files=$CHANGED_FILES" >> $GITHUB_OUTPUT
          echo "deleted_files=$DELETED_FILES" >> $GITHUB_OUTPUT
          echo "is_initial_setup=false" >> $GITHUB_OUTPUT

  trigger-markdown-translation:
    needs: check-markdown-changes
    if: needs.check-markdown-changes.outputs.has_changes == 'true' && needs.check-markdown-changes.outputs.skip_translation == 'false'
    uses: rimo/bilingual-github/.github/workflows/translate-markdown.yml@main
    permissions:
      contents: read
    secrets:
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
    with:
      is_initial_setup: ${{ needs.check-markdown-changes.outputs.is_initial_setup == 'true' }}
      changed_files: ${{ needs.check-markdown-changes.outputs.changed_files }}
      deleted_files: ${{ needs.check-markdown-changes.outputs.deleted_files }}
      is_pr: ${{ needs.check-markdown-changes.outputs.is_pr == 'true' }}
      pr_number: ${{ needs.check-markdown-changes.outputs.pr_number }}
      target_branch: ${{ needs.check-markdown-changes.outputs.target_branch }}
      output_mode: artifact

  push-translations:
    needs: [check-markdown-changes, trigger-markdown-translation]
    if: >-
      needs.trigger-markdown-translation.outputs.has_changes == 'true' &&
      github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - name: Generate repo-scoped GitHub App token
        id: app-token
        uses: actions/create-github-app-token@64ab729a6294000f5aa4a4bc0b73d1f32bc45510 # v3.3.0
        with:
          app-id: ${{ secrets.RIMO_GITHUB_APP_ID }}
          private-key: ${{ secrets.RIMO_GITHUB_APP_PRIVATE_KEY }}
          repositories: ${{ github.event.repository.name }}
          permission-contents: write

      - name: Checkout PR branch
        uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
        with:
          ref: ${{ needs.check-markdown-changes.outputs.target_branch }}
          token: ${{ steps.app-token.outputs.token }}

      - name: Download translations
        uses: actions/download-artifact@9000827ccba6bdab643e8b6fd33ac0654aef8333 # v8.0.2
        with:
          name: translations
          path: ${{ runner.temp }}/translations

      - name: Apply, commit and push translations
        env:
          TRANSLATIONS: ${{ runner.temp }}/translations
          TARGET_BRANCH: ${{ needs.check-markdown-changes.outputs.target_branch }}
        run: |
          set -euo pipefail
          # Apply only regular .md files so the artifact cannot touch anything else in the repository.
          (cd "$TRANSLATIONS/files" && find . -type f -name '*.md' -print0) |
            while IFS= read -r -d '' f; do
              mkdir -p "$(dirname "$f")"
              cp "$TRANSLATIONS/files/$f" "$f"
            done
          while IFS= read -r -d '' f; do
            case "$f" in *.md) git rm -q --ignore-unmatch -- "$f" ;; esac
          done < "$TRANSLATIONS/deleted-files"

          git config user.name "rimo-translation-bot[bot]"
          git config user.email "rimo-translation-bot[bot]@users.noreply.github.com"
          git add -A -- '*.md'
          if git diff --cached --quiet; then
            echo "No changes to commit"
            exit 0
          fi
          git commit -m "Update markdown translations"
          git pull --rebase origin "$TARGET_BRANCH"
          git push origin "HEAD:$TARGET_BRANCH"
```

> 再利用ワークフローの `output_mode: commit` (既定値) と `RIMO_GITHUB_APP_ID` / `RIMO_GITHUB_APP_PRIVATE_KEY` シークレットは非推奨です。
> すべての呼び出し元が `output_mode: artifact` に移行した後に削除します。`commit` を使い続ける場合は、呼び出し側ジョブに `contents: write` が必要です。
## 対象リポジトリを設定する手順

#### 1. シークレットを追加
- GitHubで対象リポジトリの設定ページに移動します。
- **セキュリティ > シークレットと変数 > Actions**の下に、以下のシークレットを追加します：
  - **OPENAI_API_KEY:** 翻訳用のOpenAI APIキー。
  - **RIMO_GITHUB_APP_ID / RIMO_GITHUB_APP_PRIVATE_KEY** (.mdファイルの翻訳のみ): 呼び出し側の `push-translations` ジョブで使います。再利用ワークフローには渡さないでください。

#### 2. ワークフローを有効化
対象リポジトリでGitHub Actionsが有効化されていることを確認してください。以下のイベントでワークフローがトリガーされます：
- **Issueの翻訳:**
  - Issueが作成または編集されたとき。
- **コメントの翻訳:**
  - コメントが追加または編集されたとき。

## 統合のテスト

#### 1. Issueの翻訳
- 対象リポジトリでIssueを作成または編集します。
- 「need translation」 というラベルを作成します。
- ```call-translate-issues``` ジョブがトリガーされ、Issueが翻訳されます。

#### 2. コメントの翻訳
- 対象リポジトリのIssueでコメントを追加または編集します。
- ```call-translate-comments``` ジョブがトリガーされ、コメントが翻訳されます。

#### 3. ワークフローの監視
- 対象リポジトリの**Actions**タブに移動します。
- ```Translate GitHub Issues```と```Translate Issue Comments```ワークフローのステータスを確認します。
- 成功した場合、翻訳された内容がIssueまたはコメントに表示されます。
