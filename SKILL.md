---
name: "jira-ticket-writer"
description: "Turn rough notes or context into a well-structured Jira ticket (description, tasks, acceptance criteria, test notes) written so Claude Code can implement it; creates the ticket in Jira after approval."
---

# Jira Ticket Writer

## Purpose

The tickets this skill produces have two readers: a human teammate skimming the board, and an AI coding agent (Claude Code) that will be handed the ticket text as its prompt. A vague ticket forces the agent to guess — and guesses become wrong file edits, invented requirements, and rework. So every section of the ticket exists to remove a guess: the description removes "why and where", the tasks remove "what order", the acceptance criteria remove "when am I done", and the test notes remove "how do I prove it".

Write for the agent the way you would write for a capable new engineer on their first day: they are smart, they have the codebase in front of them, but they know nothing you don't tell them.

## Workflow

1. **Collect the context.** Use everything the user gave you: their notes, pasted messages, earlier conversation, attached files. If they reference an existing Jira issue, fetch it with the Atlassian tools to build on it rather than starting blind.

2. **Check for missing essentials — ask before writing.** If any of the following is missing and cannot be safely inferred, ask the user first (at most 3 short questions in one go, via AskUserQuestion when available):
   - Which project/product and, if relevant, which repo or part of the system the work belongs to
   - For bugs: how to reproduce, and what the expected vs. actual behavior is
   - For features: who it's for and what "done" looks like from the user's point of view
   - Which Jira project and issue type to file it under (only if it's time to create and this is unclear)

   Don't interrogate the user about nice-to-haves. One round of questions, then write.

3. **Draft the ticket** using the template below and show it to the user in full.

4. **On approval, create it in Jira.** Never create before the user has seen and approved the draft. See "Creating the ticket in Jira" below.

## Ticket template

Use this structure for every ticket. Keep the summary under ~80 characters, specific, and action-oriented ("Add CSV export to candidate list" — not "Export feature").

```
Summary: <imperative, specific, ≤80 chars>

## Description
<2–6 sentences: the why and the where.
- What problem this solves or what behavior is broken, and for whom.
- Where in the system it lives: name the repo, service, module, page, or endpoint exactly as it's known.
- Any constraints or decisions already made (library choices, API contracts, designs).
For bugs: steps to reproduce, expected behavior, actual behavior, environment.>

## Tasks
<A numbered list of concrete implementation steps in a sensible order.
Each task is one verifiable action an engineer/agent can do and check off.
Name real things: files, components, endpoints, tables, env vars — when known.
5–10 tasks is typical; split the ticket if you need many more.>

## Acceptance Criteria
<A checklist of observable outcomes. Each item:
- states ONE behavior,
- is testable (a reviewer can answer yes/no),
- describes the result, not the implementation ("Given/When/Then" phrasing is welcome).
Include the unhappy paths: invalid input, empty states, permissions.>

## Things to Test
<What to verify before calling it done:
- the specific manual checks or user flows to walk through,
- edge cases worth probing (boundary values, concurrency, roles/permissions),
- what automated tests to add or update, and where they live if known,
- any regression risk: existing features this change could break.>

## Out of Scope (optional)
<Only when there's a real risk of scope creep: name what this ticket deliberately does NOT cover.>

## Attachments / References (only if there are real ones)
<Links or files the user actually provided or referenced: designs, logs, call recordings,
related tickets, docs. Skip this section entirely if there's nothing concrete to list —
never pad it with placeholders like "recordings to be attached".>

## Assumptions (only if any — and only about the feature/task itself)
<Inferences about the WORK: behavior, scope, defaults, limits (e.g. "default page size 25").
Each on one line so the user can correct it at a glance. Do NOT put guesses about repo
names, file paths, or infrastructure here — if you're not sure about those, ask the user
or leave them out of the ticket; a speculative line about them just adds noise.
Skip this section entirely when there are no assumptions worth flagging.>
```

## Writing rules

- **Never invent facts.** File paths, endpoint names, library versions, repo names — use them only if the user provided them, they're in the referenced Jira issue, or you verified them. If a technical detail like this matters but is unknown, ask (step 2) or fold "locate X" into the first task — don't speculate about it in the Assumptions section. A wrong "fact" is far more damaging to a coding agent than an honest unknown.
- **One behavior per acceptance criterion.** "Export works and handles errors" is two criteria pretending to be one; the agent can't check it off halfway.
- **Make criteria observable.** "Code is clean" is not checkable; "the export completes in under 5 seconds for 1,000 rows" is.
- **Tasks are steps, criteria are outcomes.** Don't repeat the task list in the acceptance criteria with different words.
- **Keep the user's own terminology** for products, features, and teams (e.g. their internal names), so the ticket reads naturally to their team.
- **Bugs get reproduce steps first.** A bug ticket without "steps to reproduce / expected / actual" is not done. If the user can't reproduce it reliably, say so in the ticket and capture whatever signal exists (logs, screenshots, frequency).
- **Right-size it.** If the notes describe a week of work, say so and offer to split it into 2–4 tickets instead of writing one monster.

## Creating the ticket in Jira

After the user approves the draft:

1. If you don't already know the Jira project: call `getVisibleJiraProjects` and confirm the project with the user if more than one is plausible.
2. Call `getJiraProjectIssueTypesMetadata` for that project to pick the right issue type (Story/Task/Bug) and learn any required fields.
3. Check `getContentFormatGuide` if you're unsure how to format the description field (Jira uses its own markup/ADF — don't paste raw Markdown headings if the instance expects wiki markup).
4. Call `createJiraIssue` with the summary and the full ticket body (Description, Tasks, Acceptance Criteria, Things to Test as formatted sections of the description field).
5. Report back with the issue key and link.

If the Atlassian tools aren't available or the creation fails, deliver the finished ticket text in the conversation so the user can paste it — the ticket content is the real deliverable; Jira creation is a convenience.

## Example

**Input (user's notes):** "candidates page in continuity connect is slow when there are lots of candidates, maybe pagination? backend is the node api"

**Output (abridged):**

```
Summary: Paginate the candidates list API and page in Continuity Connect

## Description
The candidates page in the Continuity Connect admin app loads all candidates in a
single request, which makes the page slow for organisations with large candidate
lists. Add server-side pagination to the Node API endpoint that serves this list,
and update the candidates page to fetch and render pages.

## Tasks
1. Add `page` and `pageSize` query params to the candidates list endpoint in the Node API (default pageSize 25).
2. Return `totalCount` in the response so the UI can render page controls.
3. Update the candidates page to request page 1 on load and render pagination controls.
4. Preserve existing search/filter behavior so filters apply across pages, not just the current page.
5. Add/update API tests for the paginated endpoint.

## Acceptance Criteria
- [ ] Given 1,000+ candidates, when the page loads, only the first page (25) is fetched and rendered.
- [ ] When the user navigates to page N, the list shows the correct slice and the URL/API call reflects page N.
- [ ] When a search filter is active, pagination applies to the filtered result set.
- [ ] When there are zero candidates, the page shows the existing empty state (no broken pager).

## Things to Test
- Manual: load with a large seeded candidate set; navigate first/middle/last pages; apply a filter then page through.
- Edge cases: exactly one page of results; pageSize boundary (25/26 items); invalid page number in the URL.
- Regression: candidate detail links from the list still open the right candidate.

## Assumptions
- Default page size of 25 (adjust if the team has a standard).
- Pagination is offset-based, not cursor-based.
```