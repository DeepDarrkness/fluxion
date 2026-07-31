# Issue Tracker: GitHub

Issues and planning artifacts for this repository live in GitHub Issues on `DeepDarrkness/fluxion`. Use the `gh` CLI for all operations.

## Conventions

- **Create an issue**: `gh issue create --repo DeepDarrkness/fluxion --title "..." --body "..."`.
- **Read an issue**: `gh issue view <number> --repo DeepDarrkness/fluxion --comments`.
- **List issues**: `gh issue list --repo DeepDarrkness/fluxion --state open` with appropriate label and state filters.
- **Comment on an issue**: `gh issue comment <number> --repo DeepDarrkness/fluxion --body "..."`.
- **Apply or remove labels**: `gh issue edit <number> --repo DeepDarrkness/fluxion --add-label "..."` or `--remove-label "..."`.
- **Close an issue**: `gh issue close <number> --repo DeepDarrkness/fluxion --comment "..."`.

## Pull Requests As A Triage Surface

**PRs as a request surface: no.**

## Publishing And Fetching

When a skill says "publish to the issue tracker", create a GitHub issue in `DeepDarrkness/fluxion`.

When a skill says "fetch the relevant ticket", run `gh issue view <number> --repo DeepDarrkness/fluxion --comments`.

## Wayfinding Operations

Wayfinder uses one map issue with child decision tickets.

- **Map**: a single issue labelled `wayfinder:map`, holding Destination, Notes, Decisions so far, Not yet specified, and Out of scope.
- **Child ticket**: an issue linked to the map as a GitHub sub-issue. Label it with one of `wayfinder:research`, `wayfinder:prototype`, `wayfinder:grilling`, or `wayfinder:task`. If sub-issues are unavailable, add the child to a task list in the map body and put `Part of #<map>` at the top of the child body.
- **Blocking**: use GitHub's native issue dependencies. Add an edge with `gh api --method POST repos/DeepDarrkness/fluxion/issues/<child>/dependencies/blocked_by -F issue_id=<blocker-database-id>`. Obtain the database ID with `gh api repos/DeepDarrkness/fluxion/issues/<number> --jq .id`. If native dependencies are unavailable, use a `Blocked by: #<number>` line at the top of the child body.
- **Frontier**: list the map's open child issues and exclude tickets with open blockers or assignees. The first remaining ticket in map order is next.
- **Claim**: assign the ticket before starting work with `gh issue edit <number> --repo DeepDarrkness/fluxion --add-assignee @me`.
- **Resolve**: post the answer as a resolution comment, close the ticket, then append a one-line gist and ticket link to the map's Decisions so far section.

Do not create a Wayfinder map unless the user explicitly invokes Wayfinder.
