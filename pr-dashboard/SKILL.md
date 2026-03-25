---
name: pr-dashboard
description: List and classify open PRs from a GitHub organization updated in a specified time range (default 1 day), prioritizing those waiting for your review, with approval recommendations
invocation: /pr-dashboard <org-name> [days]
---

# PR Dashboard

This skill generates a comprehensive dashboard of open pull requests from a specified GitHub organization, focusing on PRs updated within a configurable time range (default: last 24 hours), with automated analysis to identify potential issues.

## Usage

```
/pr-dashboard <organization-name> [days]
```

**Parameters:**
- `organization-name` (required): The GitHub organization to search
- `days` (optional): Number of days to look back for updated PRs (default: 1)

**Examples:**
```
/pr-dashboard osac-project           # Last 24 hours (default)
/pr-dashboard osac-project 3         # Last 3 days
/pr-dashboard osac-project 7         # Last week
```

## What This Skill Does

1. **Searches for Recent PRs**: Finds all open pull requests in the specified organization that have been updated within the specified time range
2. **Fetches Detailed Information**: Gets PR details including line counts (additions/deletions), review requests, author, and update time
3. **Analyzes PRs for Issues**: Reviews each PR that requires your review for potential blockers
4. **Identifies Urgent PRs**: Prioritizes PRs that are explicitly waiting for YOUR review
5. **Classifies by Size**: Groups PRs into Small, Medium, and Large categories based on lines changed
6. **Provides Approval Recommendations**: Indicates whether each PR is safe to approve or has potential issues
7. **Displays Organized Report**: Presents results in a clean, actionable format

## Implementation Steps

### Step 0: Parse Parameters
Extract the organization name and optional days parameter:
- If days parameter is not provided or invalid, default to 1 day
- Calculate the cutoff date: `current_date - days`
- Format as `YYYY-MM-DD` for the GitHub API

### Step 1: Get Current User
First, identify the current GitHub user to check which PRs are waiting for their review:

```bash
gh api user --jq '.login'
```

Store this username for later comparison.

### Step 2: Search for PRs
Search for open PRs in the organization updated since the cutoff date:

```bash
gh search prs --owner <org-name> --state open --updated ">=YYYY-MM-DD" --json number,title,repository,updatedAt,url,author,isDraft --limit 100
```

**Note**: Replace `YYYY-MM-DD` with the calculated cutoff date (current_date - days).

### Step 3: Fetch Detailed PR Information
For each PR found, fetch detailed information including line counts and review requests:

```bash
for pr_data in $(gh search prs --owner <org-name> --state open --updated ">=YYYY-MM-DD" --json number,repository --jq '.[] | @base64'); do
  pr_info=$(echo "$pr_data" | base64 -d)
  repo=$(echo "$pr_info" | jq -r '.repository.nameWithOwner')
  number=$(echo "$pr_info" | jq -r '.number')

  gh pr view "$number" --repo "$repo" --json number,title,url,additions,deletions,reviewRequests,author,updatedAt --jq "{repo: \"$repo\", number: .number, title: .title, url: .url, additions: .additions, deletions: .deletions, reviewRequests: [.reviewRequests[].login], author: .author.login, updatedAt: .updatedAt}"
done | jq -s .
```

This will output a JSON array with all PR details.

### Step 4: Analyze PRs Requiring Your Review
For each PR where you are a reviewer, fetch additional context and analyze for potential issues:

```bash
gh pr view "$number" --repo "$repo" --json title,body,files,reviews,comments
gh pr diff "$number" --repo "$repo"
```

Analyze each PR for:
- **Missing tests**: New features or bug fixes without test coverage
- **Missing type definitions**: References to types not defined in the diff
- **Large changes**: PRs with >500 lines should have comprehensive tests
- **Incomplete implementation**: Test plan checkboxes not completed
- **Breaking changes**: API changes without migration guides
- **Security concerns**: Authentication, authorization, or data handling changes
- **Missing documentation**: New features without docs
- **Dependencies**: Whether this PR depends on others being merged first

For each PR, assign one of these statuses:
- **✅ SAFE TO APPROVE**: No blocking issues found, tests present, changes are reasonable
- **⚠️ NEEDS REVIEW**: Minor concerns that should be verified (e.g., missing tests for new types)
- **🚫 BLOCKING ISSUES**: Major problems that must be addressed before approval (e.g., security issues, missing critical tests)

### Step 5: Classify PRs
Classify PRs into the following buckets:

#### Urgent Bucket
PRs where the current user appears in the `reviewRequests` array. These require immediate attention.

#### Size Buckets
Calculate total lines changed as `additions + deletions`, then classify:
- **Small**: < 100 lines changed
- **Medium**: 100-500 lines changed
- **Large**: > 500 lines changed

### Step 6: Display Results
Format the output as follows (update the title to reflect the actual time range):

```markdown
# Open PRs in <org-name> (Updated in Last X day(s))

Found **N open PRs** across repositories.

---

## 🔴 URGENT - Waiting for Your Review

**N PRs requiring your review:**

1. ✅ **[SIZE - X lines]** repo#number
   `title` by @author
   https://github.com/org/repo/pull/number
   **Status**: Safe to approve - [brief reason]

2. ⚠️ **[SIZE - X lines]** repo#number
   `title` by @author
   https://github.com/org/repo/pull/number
   **Concerns**: [list of concerns to verify]

3. 🚫 **[SIZE - X lines]** repo#number
   `title` by @author
   https://github.com/org/repo/pull/number
   **Blocking Issues**: [list of issues that must be addressed]

---

## 📊 By Size - Other PRs

### Small (< 100 lines)
1. **[X lines]** repo#number
   `title` by @author
   https://github.com/org/repo/pull/number

### Medium (100-500 lines)
1. **[X lines]** repo#number
   `title` by @author
   https://github.com/org/repo/pull/number

### Large (> 500 lines)
1. **[X lines]** repo#number
   `title` by @author
   https://github.com/org/repo/pull/number
```

## Analysis Guidelines

When analyzing PRs, look for these patterns:

### ✅ Safe to Approve Indicators
- Clear, focused changes with good test coverage
- Well-documented PRs with completed test plans
- Simple RBAC or config changes that are well-tested
- All dependencies are satisfied
- No security concerns
- Passes all CI checks

### ⚠️ Needs Review Indicators
- Missing tests for new code (but existing tests pass)
- Referenced types not visible in diff (may exist elsewhere)
- Large PRs without proportional test coverage
- Incomplete test plan checkboxes
- Minor code quality concerns
- Documentation could be improved

### 🚫 Blocking Issues
- Security vulnerabilities (SQL injection, XSS, authentication bypass)
- Breaking changes without migration path
- No tests for new features or bug fixes (>100 lines)
- Dependencies on unmerged PRs
- Critical functionality broken
- Destructive operations without safeguards

## Tips

- Use a timeout of 300000ms (5 minutes) for the detailed PR fetch command in case there are many PRs
- For PRs requiring your review, fetch the full diff and analyze it
- PRs in the URGENT bucket should include their approval status
- Sort PRs within each size bucket by line count (ascending)
- If no PRs are found, inform the user clearly
- Include the total count of PRs found at the top of the report
- Output URLs as plain text (not markdown links) so they are clickable in the terminal
- When days > 1, there may be many PRs - consider limiting analysis to only PRs awaiting your review

## Error Handling

- If the organization doesn't exist or isn't accessible, `gh search prs` will fail - inform the user
- If rate limits are hit, suggest trying again later
- If no PRs are found, this is a valid result - report it clearly
- If PR analysis fails for any PR, mark it as "⚠️ NEEDS MANUAL REVIEW" and continue with others
- If days parameter is invalid (non-numeric or negative), default to 1 day
