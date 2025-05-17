# Project description

Enhanced MCP Server with Merge Detection
Core Problem
When developers frequently merge master into their feature branches, the MCP server needs to distinguish between:

Developer changes: Actual code changes made by the developer
Merge changes: Changes introduced through merging other branches

Additional Tools for Merge Detection
Merge Analysis Tools:

```json
json{
  "tools": [
    {
      "name": "analyze_commit_authorship",
      "description": "Differentiate between author's changes and merge-introduced changes",
      "inputSchema": {
        "type": "object",
        "properties": {
          "base_branch": {
            "type": "string",
            "description": "Base branch to compare against (e.g., 'origin/master')"
          },
          "exclude_merges": {
            "type": "boolean",
            "default": true,
            "description": "Exclude merge commits from analysis"
          },
          "author_filter": {
            "type": "string",
            "description": "Filter by specific author (current user by default)"
          }
        }
      }
    },
    {
      "name": "get_merge_base",
      "description": "Find the common ancestor between current branch and target",
      "inputSchema": {
        "type": "object",
        "properties": {
          "target_branch": {
            "type": "string",
            "description": "Target branch for comparison"
          },
          "current_branch": {
            "type": "string",
            "description": "Current branch (defaults to HEAD)"
          }
        }
      }
    },
    {
      "name": "get_developer_only_changes",
      "description": "Get changes excluding those from merges",
      "inputSchema": {
        "type": "object",
        "properties": {
          "base_commit": {
            "type": "string",
            "description": "Base commit to compare from"
          },
          "ignore_merge_commits": {
            "type": "boolean",
            "default": true
          },
          "author_email": {
            "type": "string",
            "description": "Email of the developer to focus on"
          }
        }
      }
    },
    {
      "name": "classify_changes_by_source",
      "description": "Classify each change as developer-made or merge-introduced",
      "inputSchema": {
        "type": "object",
        "properties": {
          "detailed_analysis": {
            "type": "boolean",
            "default": false,
            "description": "Include detailed commit information for each change"
          }
        }
      }
    }
  ]
}
```

## Workflow

### Change Detection Phase:

1. Find merge base with target branch
2. Identify all commits since merge base
3. Classify commits as:
   - Author commits (developer changes)
   - Merge commits (external changes)
   - Mixed commits (merge conflicts resolved by developer)

### Change Classification

For each file change:

- Track which commits introduced the change
- Determine if change originated from:
  * Developer's direct edits
  * Automatic merge resolution
  * Manual merge conflict resolution
  * External branch merges

### Focused Review

- Primary focus: Developer-authored changes
- Secondary review: Merge conflict resolutions
- Ignore: Automatic merge changes from other branches

## Configuration for Merge Handling

```yaml
yaml# .mcp-pr-review.yml
merge_detection:
  enabled: true
  base_branch: "origin/master"
  focus_on_author_changes: true
  include_merge_conflict_resolution: true
  exclude_automatic_merges: true
  
review_scope:
  developer_changes:
    weight: 100  # Full review
    include_tests: true
    include_docs: true
  
  merge_resolutions:
    weight: 75   # Focused review on conflict resolution
    check_logic_preservation: true
    verify_no_functionality_loss: true
  
  automatic_merges:
    weight: 10   # Minimal review, mainly for context
    check_for_unexpected_changes: true
```

## Enhanced Output Format

```json
json{
  "review_id": "uuid",
  "timestamp": "2025-05-14T10:30:00Z",
  "change_classification": {
    "total_files_changed": 12,
    "developer_authored": 8,
    "merge_introduced": 3,
    "conflict_resolutions": 1
  },
  "developer_changes": {
    "files_changed": 8,
    "lines_added": 120,
    "lines_removed": 45,
    "focus_areas": [
      "src/components/Button.tsx",
      "src/utils/api.ts",
      "tests/Button.test.tsx"
    ]
  },
  "merge_analysis": {
    "merge_base": "abc123def",
    "merged_branches": ["feature/user-auth", "hotfix/security-patch"],
    "conflict_resolutions": [
      {
        "file": "src/config/database.ts",
        "resolution_quality": "good",
        "notes": "Properly preserved both authentication methods"
      }
    ]
  },
  "file_reviews": [
    {
      "file_path": "src/components/Button.tsx",
      "change_source": "developer",
      "review_priority": "high",
      "issues": [...],
      "strengths": [...]
    },
    {
      "file_path": "src/config/database.ts",
      "change_source": "merge_resolution",
      "review_priority": "medium",
      "merge_context": {
        "conflicted_with": "hotfix/security-patch",
        "resolution_approach": "manual"
      },
      "issues": [...]
    }
  ],
  "recommendations": [
    "Focus review on Button.tsx - significant logic changes",
    "Verify database config merge didn't break authentication",
    "Consider adding integration tests for merged security features"
  ]
}
```

## Implementation Strategy

Git Command Examples:

```bash
bash# Find merge base
git merge-base HEAD origin/master

# Get author-only changes
git log --author="developer@email.com" --no-merges merge-base..HEAD

# Analyze file changes by commit
git log -p --follow -- path/to/file merge-base..HEAD

# Identify merge commits
git log --merges merge-base..HEAD
```

## Advanced Features

### Intelligent Conflict Detection

Detect when merge conflicts were resolved
Analyze quality of conflict resolution
Flag potentially problematic merges

### Change Attribution

Track original author of each line
Identify when developer modified existing code vs. added new code
Highlight areas where developer changed merged code

### Review Optimization

Skip review of unchanged merged code
Focus on areas with developer input
Provide context about what was merged

This approach ensures that code reviews focus on the actual developer contributions while still maintaining awareness of the broader changes introduced through merges.
