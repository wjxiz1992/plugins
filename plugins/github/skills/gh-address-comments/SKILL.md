---
name: gh-address-comments
description: Address actionable GitHub pull request review feedback. Use when the user wants to inspect the latest comment, top-level conversation comments, review summaries, unresolved review threads, requested changes, or inline review comments on a PR, then implement selected fixes. Use the GitHub app for PR metadata and patch context, and use the bundled GraphQL script via `gh` to read all review surfaces and preserve thread-level state.
---

# GitHub PR Comment Handler

Use this skill when the user wants to work through requested changes on a GitHub pull request. Use the GitHub app from this plugin for PR metadata and patch context, but treat thread-aware review data as a `gh api graphql` problem because the connector comment surface is flat and does not preserve full review-thread state.

Run all `gh` commands with elevated network access. If CLI auth is required, confirm `gh auth status` first and ask the user to authenticate with `gh auth login` if it fails.

## Workflow

1. Resolve the PR.
   - If the user provides a repository and PR number or URL, use that directly.
   - If the request is about the current branch PR, use local git context plus `gh auth status` and `gh pr view --json number,url` to resolve it.
2. Inspect every review surface.
   - Use the GitHub app from this plugin to fetch PR metadata and patch context when the repo and PR are known.
   - Use the bundled `scripts/fetch_comments.py` workflow whenever the task depends on the latest comment, actionable feedback, unresolved review threads, inline review locations, or resolution state.
   - Read its `ordered_feedback` list first. The script builds this list from three independent feedback surfaces: `conversation_comments`, non-empty `reviews[].body`, and `review_threads[].comments.nodes`.
   - Do not assume actionable feedback is inline. A review body can request PR title or description changes and has no `isResolved` field.
   - Use connector-only comment reads only for lightweight top-level PR comment summaries.
3. Normalize and order feedback before filtering.
   - Use `ordered_feedback`, which normalizes `createdAt` for conversation and inline comments and `submittedAt` for review bodies, sorted newest first.
   - When the user asks for the "latest" or "newest" comment, select from this globally ordered list before applying thread-only filters such as `isResolved` or `isOutdated`.
   - When the request means the latest reviewer feedback, skip entries authored by `pull_request.author` but do not skip any feedback surface.
   - Report the surface and timestamp of the selected comment so the choice is auditable.
4. Cluster actionable feedback.
   - Group comments by file or behavior area.
   - Separate actionable change requests from informational comments, approvals, already-addressed feedback, and duplicates.
   - Use `isResolved` and `isOutdated` only for inline-thread classification; never use them to exclude conversation comments or review bodies.
5. Confirm scope before editing.
   - Present numbered actionable comments with their surface and a one-line summary of the required change.
   - If the user did not ask to fix everything, ask which threads to address.
   - If the user asks to fix everything, interpret that as all outstanding actionable feedback across all three surfaces and call out anything ambiguous.
6. Implement the selected fixes locally.
   - Keep each code change traceable back to the thread or feedback cluster it addresses.
   - If a comment calls for explanation rather than code, draft the response rather than forcing a code change.
7. Reply without resolving the conversation.
   - When the user says "resolve a comment", interpret it as making the code or documentation change, validating it, and replying with the commit and validation evidence.
   - Never call `resolveReviewThread` or otherwise mark a review conversation resolved. Conversation resolution belongs exclusively to the original comment author.
8. Summarize the result.
   - List which comments were addressed, which were intentionally left open, what tests or checks support the change, and which conversations remain for their authors to resolve.

## Write Safety

- Do not reply on GitHub or submit a review unless the user explicitly asks for that write action.
- Never resolve review threads, even when the user says "resolve comment"; address and reply only, leaving conversation resolution to the original comment author.
- If review comments conflict with each other or would cause a behavioral regression, surface the tradeoff before making changes.
- If a comment is ambiguous, ask for clarification or draft a proposed response instead of guessing.
- Do not treat flat PR comments from the connector as a complete representation of review-thread state.
- If `gh` hits auth or rate-limit issues mid-run, ask the user to re-authenticate and retry.

## Fallback

If neither the connector nor `gh` can resolve the PR cleanly, tell the user whether the blocker is missing repository scope, missing PR context, or CLI authentication, then ask for the missing repo or PR identifier or for a refreshed `gh` login.
