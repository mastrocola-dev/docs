# Repository rulesets

Every repository protects its default branch with the same ruleset: changes land only through pull requests, history cannot be rewritten, and the branch cannot be deleted. Nobody bypasses it, the organization owner included.

## Why

- `docs`: the ADR index is generated and reviewed inside the pull request that changes `adr/` ([ADR-005](../adr/005-agent-generated-content.md)). A direct push skips generation and human review, and breaks the `verify` job on `main`
- `infra`: a push to `main` runs `apply`; without a pull request, no `plan` is ever reviewed
- Everywhere else: CI runs on pull requests, so a direct push ships unchecked code

Approvals are not required: a single maintainer cannot approve their own pull request. The gate is that every change goes through one, with its checks visible before merge. Status checks are not required either: path-filtered workflows never report on pull requests outside their paths, which would block those merges.

## Ruleset

```json
{
  "name": "main",
  "target": "branch",
  "enforcement": "active",
  "conditions": { "ref_name": { "include": ["~DEFAULT_BRANCH"], "exclude": [] } },
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    {
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 0,
        "dismiss_stale_reviews_on_push": false,
        "require_code_owner_review": false,
        "require_last_push_approval": false,
        "required_review_thread_resolution": false
      }
    }
  ]
}
```

## Applying

Save the JSON above as `ruleset.json`, then, for each repository without it:

```bash
gh api repos/mastrocola-dev/<repo>/rulesets --jq '.[].name'
gh api -X POST repos/mastrocola-dev/<repo>/rulesets --input ruleset.json
```

The first command lists existing rulesets — `POST` does not deduplicate, so run it only when `main` is absent. A new repository gets the ruleset right after its first push.
