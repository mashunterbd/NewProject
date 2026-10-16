বার্তা ১: প্রথম ডেভেলপারের জন্য (সেটআপ)

এর সাথে বার্তা ২-ও পাঠাবেন, কারণ সে ওটাকে CONTRIBUTING.md হিসেবে রিপোজিটরিতে রাখবে।

``` Subject: New project setup, Part 1 (FIRST developer): build the CI + auto-merge pipeline

ROLE
You are the first developer on this repository. Other developers start only
after you finish this task and I confirm. The pipeline you create does this:
when anyone pushes a branch, it checks the code (lint, build, tests) after
merging in the latest main. If everything passes, it opens a PR and merges it
into main automatically, then deletes the branch.

RULES
1. Use YOUR OWN GitHub account and your own name for every commit. Never use
   the owner's or anyone else's account.
2. Never use --force.
3. Pushing directly to main is allowed ONLY for the setup files in Step 4.
4. Do not edit application code to make checks pass. If a check fails, stop and
   report it.
5. If any result differs from what is described here, stop and report.

STEP 1: Identity (one time)

  git config --global user.name "Your Full Name"
  git config --global user.email "the-email-linked-to-your-GitHub-account"
  git config --global --get user.name
  git config --global --get user.email

Log in to GitHub with your own account (gh auth login, or sign in through the
Git credential manager). On a shared computer, make sure no other person's
GitHub login is saved. Then confirm:

  gh auth status

The account shown must be yours.

STEP 2: Get the repository and inspect it

  git clone <repository-url>
  cd <repository-folder>
  git status
  git branch -a

Check which scripts exist. For a Node project, open package.json and note
which of these are defined: lint, build, test. Only include steps in the
workflow for scripts that really exist.

STEP 3: Create the workflow file

Create the file .github/workflows/ci-merge.yml with the content between
BEGIN FILE and END FILE. Copy the YAML exactly; indentation matters.
Replace only the block marked PROJECT-SPECIFIC STEPS to match the project
(examples are given after the file).

----- BEGIN FILE: .github/workflows/ci-merge.yml -----
name: CI and Auto-merge

on:
  push:
    branches: ['**']

permissions:
  contents: write
  pull-requests: write

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Merge latest main into this branch (test the final result)
        if: github.ref != 'refs/heads/main'
        run: |
          git config user.name "ci-bot"
          git config user.email "ci-bot@users.noreply.github.com"
          git fetch origin main
          git merge --no-edit origin/main

      # ===== PROJECT-SPECIFIC STEPS (replace this block) =====
      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Build
        run: npm run build

      - name: Test
        run: npm test
      # ===== END PROJECT-SPECIFIC STEPS =====

  auto-merge:
    needs: build-and-test
    if: >
      github.ref != 'refs/heads/main' &&
      !contains(github.event.head_commit.message, '[no-merge]')
    runs-on: ubuntu-latest
    concurrency:
      group: auto-merge
      cancel-in-progress: false
    env:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      GH_REPO: ${{ github.repository }}
      BRANCH: ${{ github.ref_name }}
      PUSHER: ${{ github.actor }}
    steps:
      - name: Create PR if missing, then merge
        run: |
          set -euo pipefail

          AHEAD=$(gh api "repos/$GH_REPO/compare/main...$BRANCH" -q .ahead_by)
          if [ "$AHEAD" -eq 0 ]; then
            echo "Nothing new compared with main. Nothing to merge."
            exit 0
          fi

          PR=$(gh pr list --head "$BRANCH" --base main --state open --json number -q '.[0].number')
          if [ -z "$PR" ]; then
            gh pr create --base main --head "$BRANCH" \
              --title "[$PUSHER] Merge $BRANCH into main" \
              --body "CI passed. Pushed by $PUSHER."
            PR=$(gh pr list --head "$BRANCH" --base main --state open --json number -q '.[0].number')
          fi

          DRAFT=$(gh pr view "$PR" --json isDraft -q .isDraft)
          HOLD=$(gh pr view "$PR" --json labels -q '[.labels[].name] | index("hold") != null')
          if [ "$DRAFT" = "true" ] || [ "$HOLD" = "true" ]; then
            echo "PR #$PR is a draft or has the hold label. Not merging."
            exit 0
          fi

          gh pr merge "$PR" --merge --delete-branch
----- END FILE -----

Examples for the PROJECT-SPECIFIC STEPS block (use only what fits, and only
commands that exist in the project):

  Python:
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install -r requirements.txt
      - run: ruff check .
      - run: pytest

  Flutter:
      - uses: subosito/flutter-action@v2
        with:
          channel: stable
      - run: flutter pub get
      - run: flutter analyze
      - run: flutter test

  PHP / Laravel:
      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.3'
      - run: composer install --no-interaction --prefer-dist
      - run: php artisan test

If the Actions log shows a deprecation warning about Node.js 20, update the
actions/checkout and actions/setup-node versions to their current major
versions.

STEP 4: Push the setup files to main

Add the workflow file and the CONTRIBUTING.md file. CONTRIBUTING.md must
contain the "Developer Workflow" text that the owner sent with this message.

  git switch main
  git add .github/workflows/ci-merge.yml CONTRIBUTING.md
  git commit -m "chore: add CI and auto-merge workflow and contributing guide"
  git push

Open the repository's Actions tab and confirm the run on main is green.
On main the auto-merge job is skipped. That is expected.

STEP 5: Test the pipeline (required before anyone else starts)

Test A, success case:
  git switch -c <your-username>/ci-smoke-test
  (add one harmless line to README.md)
  git commit -am "docs: smoke test the pipeline"
  git push -u origin <your-username>/ci-smoke-test

Expected in the Actions tab: build-and-test green, auto-merge green, a PR
titled "[<your-username>] Merge ... into main" merged, and the branch deleted.
Then: git switch main; git pull; git fetch --prune
Check that the commit on main shows YOUR name as author.

Test B, failure case:
  git switch -c <your-username>/ci-fail-test
  (introduce a deliberate lint or build error)
  git commit -am "test: deliberate failure [no-merge]"
  git push -u origin <your-username>/ci-fail-test

Expected: build-and-test red, auto-merge skipped, nothing merged. Then delete
the test branch on GitHub and locally, and make sure main is unchanged.
(Because of [no-merge], this test shows only that the check goes red. To also
prove a failure blocks the merge, repeat once without [no-merge]: the run must
end red and no PR must be merged.)

Troubleshooting:
  - Error "GitHub Actions is not permitted to create or approve pull
    requests": the owner must tick that option in Settings > Actions > General.
  - Error about merge commits not being allowed: the owner must enable
    "Allow merge commits" in Settings > General > Pull Requests.

REPORT BACK
  - Output of: git config --global --get user.name, and gh auth status
  - The final ci-merge.yml you pushed (and which steps you changed)
  - Test A result with the run link, and the author name shown on main
  - Test B result with the run link
  - Anything unusual

Do not tell other developers to start until I confirm. ``` 
