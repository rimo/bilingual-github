# Bilingual-GitHub 

## Overview
The bilingual-github repository provides reusable GitHub Actions workflows for translating GitHub issues and comments.
This documentation explains how to use the workflows for:
- Translating GitHub Issues
- Translating GitHub Comments

## Translation Behavior
- Translation only occurs when an issue has the "**need translation**" label
- When the label is present:
  - The issue content will be translated
  - All comments on that issue will be translated
  - Any edits to the issue or its comments will trigger new translations
- If an issue doesn't have the **"need translation"** label:
  - No translation will occur for the issue or its comments
  - The translation workflows will skip processing

## Steps to Use the Reusable Workflows in a Target Repository

#### 1. Translate GitHub Issues
To use the workflow for translating issues, create the following file in the target repository:
```bash
.github/workflows/translate-issues.yml
```
#### Code for the Calling Workflow (Translate Issues)
```python
name: Translate GitHub Issues

on:
  issues:
    types:
      - labeled
      - edited

jobs:
  call-translate-issues:
    if: |
      github.event.label.name == 'need translation' ||
      (github.event.action == 'edited' && contains(github.event.issue.labels.*.name, 'need translation'))
    uses: rimo/bilingual-github/.github/workflows/translate-issues.yml@main
    secrets:
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
    with:
      issue_number: ${{ github.event.issue.number }}
```
#### 2. Translate GitHub Comments
To use the workflow for translating comments, create the following file in the target repository:
```base
.github/workflows/translate-comments.yml
```
#### Code for the Calling Workflow (Translate Comments)
```python
name: Translate Issue Comments

on:
  issue_comment:
    types:
      - created
      - edited
  issues:
    types:
      - labeled

jobs:
  call-translate-comments:
    if: |
      (github.event_name == 'issues' && github.event.label.name == 'need translation') ||
      github.event_name == 'issue_comment'
    uses: rimo/bilingual-github/.github/workflows/translate-comments.yml@main
    secrets:
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
    with:
      issue_number: ${{ github.event.issue.number }}
      comment_id: ${{ github.event.comment.id || '' }}
```
#### 3. Translate .md files
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
  contents: write
  pull-requests: write
  
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
    secrets:
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
      RIMO_GITHUB_APP_ID: ${{ secrets.RIMO_GITHUB_APP_ID }}
      RIMO_GITHUB_APP_PRIVATE_KEY: ${{ secrets.RIMO_GITHUB_APP_PRIVATE_KEY }}
    with:
      is_initial_setup: ${{ needs.check-markdown-changes.outputs.is_initial_setup == 'true' }}
      changed_files: ${{ needs.check-markdown-changes.outputs.changed_files }}
      deleted_files: ${{ needs.check-markdown-changes.outputs.deleted_files }}
      is_pr: ${{ needs.check-markdown-changes.outputs.is_pr == 'true' }}
      pr_number: ${{ needs.check-markdown-changes.outputs.pr_number }}
      target_branch: ${{ needs.check-markdown-changes.outputs.target_branch }}  
```

## Steps to Configure the Target Repository

#### 1. Add Secrets
- Navigate to target repository's settings on GitHub.
- Under **Security > Secrets and variables > Actions**, add the following
  secret:
  - **OPENAI_API_KEY:** Your OpenAI API key for translations.

#### 2. Enable the Workflows
Ensure that GitHub Actions are enabled for the repository. The workflows will trigger on the following events:
- **Translate Issues:**
  - When an issue is created or edited.
- **Translate Comments:**
  - When a comment is added or edited.

#### 3. Create the Required Label
- Go to your repository's **Issues > Labels**
- Create a new label named **"need translation"**
- This label will be used to control which issues get translated
  
## Testing the Integration

#### 1. Translate Issues
- Open or edit an issue in the target repository.
- Add the "need translation" label to the issue
- The ```call-translate-issues``` job will trigger and translate the issue.

#### 2. Translate Comments
- Add or edit a comment in an issue in the target repository.
- The ```call-translate-comments``` job will trigger and translate the comment.

#### 3. Monitor the Workflow
- Go to the **Actions** tab in the target repository.
- Check the status of the ```Translate GitHub Issues``` and ```Translate Issue Comments``` workflows.
- If successful, the translated content will appear in the issue or 
  comment.
