---
title: "How Does Google Review Code?"
datePublished: 2026-06-07T16:33:20.268Z
cuid: cmq403yis00071spq03jh6bec
slug: how-does-google-review-code
cover: https://cdn.hashnode.com/uploads/covers/625b81432f84e0d4fc33dca5/fa7693e6-e00c-4e94-b619-8fe86c6bfd7c.png
tags: code-review, github, google, pull-requests

---

Google processes tens of thousands of code changes every day across a codebase of hundreds of millions of lines. The secret isn't magic tooling — it's a documented, principled approach to code review that any team can adopt.

In 2019, Google open-sourced its internal *Engineering Practices* documentation — the same guides used by its 25,000+ engineers. This article distills those principles into an actionable playbook, adapted for teams using GitHub Pull Requests (PRs) instead of Google's internal "Change Lists" (CLs).

* * *

## Key Terminology

Before diving in, here's how Google's internal vocabulary maps to standard GitHub terms:

| Google term | GitHub / common equivalent |
| --- | --- |
| CL (Change List) | Pull Request (PR) |
| LGTM | Approve / "Looks Good To Me" |
| Nit | Minor, non-blocking comment |
| Critique / Gerrit | GitHub PR interface |
| Readability review | Language-specific style approval |

* * *

## The One Overriding Principle

Every rule in Google's guide flows from one foundational idea:

> **Approve a PR once it *definitely improves* the overall code health of the system — even if it isn't perfect. There is no such thing as perfect code. There is only better code.**

This is a deliberate counterweight to two failure modes:

*   Reviewers who block PRs over minor style preferences.
    
*   Reviewers who approve anything to avoid conflict.
    

The standard is not "does this code meet my ideal?" but rather "does this move the codebase forward?" The reviewer owns a stake in that outcome — they share responsibility for code health with the author.

* * *

## What to Look for in Every PR

Google's guide identifies eight review dimensions. These are ordered by priority — design and functionality matter far more than style.

### 1\. Design *(highest priority)*

Is the code well-designed and appropriate for the system? Does the approach fit the existing architecture?

**Ask yourself:** Would you have designed this differently? If so, is your way clearly better, or just different?

### 2\. Functionality

Does the code behave as the author intended? Is the behavior actually good for users?

```csharp
// Example: reviewer checks edge cases the author may have missed

public User GetUser(int userId)
{
    // What happens when userId is 0 or negative?
    // What if the DB returns multiple rows?
    return _db.Query<User>("SELECT * FROM Users WHERE Id = @id", new { id = userId });
}
```

### 3\. Complexity

Could the code be simpler? Will another developer easily understand and use it six months from now?

> "Complexity is expensive. Simplicity scales."

Watch for: over-engineering, unnecessary abstractions, and clever tricks that are hard to follow.

### 4\. Tests

Are there correct, well-designed automated tests? Do they cover the important cases, including edge cases and failure paths?

Note: reviewers do **not** manually test functionality — that is the author's responsibility. Reviewers evaluate whether the tests themselves are correct and sufficient.

### 5\. Naming

Did the developer choose clear names for variables, classes, methods, and files?

```csharp
// Bad naming
public List<Dictionary<string, object>> Proc(List<Dictionary<string, object>> d, string f)
{
    return d.Where(x => (int)x[f] > 0).ToList();
}

// Good naming
public List<Dictionary<string, object>> FilterPositiveEntries(
    List<Dictionary<string, object>> dataset, string fieldName)
{
    return dataset.Where(entry => (int)entry[fieldName] > 0).ToList();
}
```

### 6\. Comments

Are comments clear, useful, and explaining **why** rather than **what**?

```csharp
// Bad comment — explains the "what", which is already obvious from the code
// Loop through users and increment counter
foreach (var user in users)
{
    counter++;
}

// Good comment — explains the "why"
// We count active users separately from total to exclude soft-deleted accounts,
// which still appear in the Users table with Status = "Deleted".
foreach (var user in users)
{
    if (user.Status == UserStatus.Active)
        activeCount++;
}
```

### 7\. Style

Does the code follow the team's style guide? Style is the **lowest** priority — it should never block a PR unless it violates a written standard.

### 8\. Documentation *(lowest priority)*

Are READMEs, changelogs, and API docs updated to match the change? Documentation updates must be in the same PR — not deferred to a follow-up.

* * *

## How to Navigate a PR: The 3-Step Approach

Google's guide provides a concrete reading order that avoids getting lost in line-by-line details before understanding the big picture.

**Step 1 — Read the description**

Understand the intent. Does the PR description explain what problem this solves and why this approach was chosen?

**Step 2 — Find and review the main files first**

Identify the files with the most logical changes. Review those for design issues before reading anything else.

> If there is a major design flaw in the core files, a large portion of the remaining code will be rewritten anyway. Catching this early saves everyone's time.

**Step 3 — Review remaining files**

After confirming no major design problems exist, go through the remaining files in order.

**Bonus tip:** Read the tests before the implementation. Tests describe what the code is *supposed* to do — they give you a mental model before you see the business logic.

* * *

## Writing Review Comments That Actually Help

The quality of a comment determines whether a review is productive or demoralizing. Google's guide is explicit: **comment on the code, never on the developer.**

### Use comment prefixes to signal intent

| Prefix | Meaning |
| --- | --- |
| `Nit:` | Minor polish — not a blocker. The author can ignore it. |
| `Optional:` | A suggestion worth considering, but not required for approval. |
| `FYI:` | Informational only. No action required. |
| *(no prefix)* | A required change. Blocking feedback. Always explain why. |

### Show the principle, not just the fix

**Unhelpful:**

```plaintext
Change this to use a map.
```

**Actionable:**

```plaintext
This loop is O(n²). Using a map here would make the lookup O(1),
which matters at the volume this function is called.
See the same pattern in payments/utils.go:47.
```

The better comment explains the engineering principle. The developer learns a pattern they'll apply in future PRs — not just a one-time fix. Code review is also a mentoring channel.

### Flag problems, don't always prescribe solutions

It is often better to point out a problem and let the developer solve it — the author knows the code's context best.

> "There's an unhandled edge case when the list is empty."

is often more useful than rewriting the code for them. The author will find a solution that fits the surrounding context better.

### Give credit where it's due

Code reviews often focus only on mistakes, but Google's guide is explicit: recognizing good work explicitly reinforces best practices and balances the inevitably critical tone of corrections.

```csharp
// Great use of Polly's retry policy with exponential backoff here.
// Clean and exactly what this service needed.
```

* * *

## Review Speed: The One-Business-Day Rule

Google optimizes for **team velocity**, not individual velocity. Slow reviews create compounding problems: blocked developers, growing frustration, and pressure to accept lower-quality code.

### The rules

*   **First response within one business day.** This is the maximum — not the target. Responding before end of day keeps the author unblocked.
    
*   **Don't interrupt focused work.** If you're in deep work, finish it first. Batch your review sessions.
    
*   **Emergency PRs get priority.** A PR that fixes a production outage, closes a security hole, or unblocks a major launch should skip the queue. Speed matters more than perfection in genuine emergencies.
    

### The paradox of strict reviews

> Counter-intuitively, consistent review standards make reviews faster over time. Developers internalize what's expected and submit higher-quality PRs, requiring fewer round trips.

Do not lower your standards to speed up a review. Approving a bad PR to avoid delay creates far more rework later.

* * *

## Keep PRs Small

One of the most impactful — and most resisted — practices in Google's guide is the insistence on small PRs.

**The data:** 90% of code reviews at Google involve fewer than 10 files, and the median review completes in under 4 hours. The median at most other large companies is 15–24 hours. The primary difference is PR size.

### Why small PRs win

*   **Reviewed faster** — Finding 5 minutes twice is easier than blocking half an hour.
    
*   **Reviewed more deeply** — Feedback quality degrades with volume. A reviewer's attention is finite.
    
*   **Easier to roll back** — A focused, single-purpose change is trivially reverted.
    
*   **Fewer merge conflicts** — Small PRs land fast and don't go stale while the rest of the codebase moves.
    

### Splitting large changes

The usual objection is "but this feature requires a big change." Google's answer: split it.

```plaintext
# Example: a new feature requiring DB schema changes + API + tests

PR 1: Add DB migration and schema change (no business logic yet)
PR 2: Add the repository/data layer
PR 3: Add the API endpoint + integration tests
PR 4: Update documentation
```

Each unit is reviewable on its own merits and independently deployable.

> **Red flag:** A PR titled "Various fixes and improvements" touching 40 files is not a PR — it's a rubber-stamp waiting to happen because no reviewer has the energy to read it properly.

* * *

## How to Handle Reviewer–Author Conflicts

Disagreements are normal. Google's escalation path is clear and sequential.

**Level 1 — Resolve in PR comments**

Reference the Engineering Practices guide or team standards to ground the discussion in shared principles, not personal opinions. Technical facts and data overrule preferences.

**Level 2 — Move to a synchronous conversation**

A 10-minute call resolves in minutes what async comments drag out for days. Record the outcome as a comment on the PR so future readers have context.

**Level 3 — Escalate to a technical lead or team**

If consensus still can't be reached, bring in a wider audience: a team discussion, a tech lead's ruling, or an engineering manager's decision.

**The hard rule:** A PR should never just stall. Paralysis is worse than an imperfect decision. Never let a change sit around because the reviewer and author can't come to an agreement.

* * *

## Style: The Lowest-Priority Dimension

Style is where many code reviews go wrong — reviewers spend disproportionate time on whitespace and naming conventions while missing design flaws.

### The rules on style

*   The **style guide is the only authority**. Personal preferences don't override it, even from senior engineers.
    
*   Style issues **not in the guide** are personal preference — accept the author's choice.
    
*   If no prior style exists in the codebase, **accept the author's style**.
    
*   Style feedback should always be a `Nit:` — never a blocker on its own.
    

```csharp
// Correct: style violation with a nit prefix
// Nit: Our convention is PascalCase for method names.
//      processUserData() → ProcessUserData()

// Correct: no prior convention exists
// LGTM — no existing convention here, your choice is fine.

// Wrong: blocking a PR over personal preference
// Request changes: I prefer var over explicit types everywhere. Please change.
// ↑ Reject this. Style not in the guide = personal preference.
```

* * *

## The PR Author's Responsibilities

The guide isn't one-sided. Authors have responsibilities too.

### Author checklist before opening a PR

*   The PR has a clear description explaining **why** this change is needed, not just what was done.
    
*   The change is self-contained and as small as reasonably possible.
    
*   Tests and documentation are included in the same PR — not deferred to a follow-up.
    
*   The author has reviewed their own diff before requesting review.
    
*   The best-suited reviewers are assigned — not just whoever is most available.
    
*   No speculative functionality: only implement what is needed **now**, not what might be needed later.
    

* * *

## Applying This to Your Team

You don't need to overhaul your process at once. Three changes deliver most of the improvement:

**1\. Adopt the comment prefixes immediately**

Add `Nit:`, `Optional:`, and `FYI:` to your team's vocabulary. It costs nothing and eliminates a major source of review friction within the first week.

**2\. Set a one-business-day SLA**

Agree as a team that PRs get a first response within one business day. Most complaints about "strict reviews" disappear when reviews are also *fast* reviews.

**3\. Enforce small PRs**

Set a soft limit (e.g. 400 lines of non-generated code). Large PRs require a written justification in the description. This single change has the biggest impact on review quality and velocity.

Once those habits are established, add the eight-dimension review checklist to your PR template, and ensure your style guide is documented so reviewers have a shared authority instead of arguing personal preferences.

* * *

## Summary

| Principle | Practical rule |
| --- | --- |
| Improve code health | Approve PRs that move the codebase forward, even if imperfect |
| Review order matters | Design first, style last |
| Navigate top-down | Description → main files → remaining files |
| Comment clearly | Use `Nit:` / `Optional:` / `FYI:` prefixes |
| Move fast | First response within one business day |
| Keep it small | Target under 10 files; always under 400 lines |
| Resolve conflicts | Comments → sync call → escalate. Never stall. |
| Authors own quality | Self-review, small scope, include tests |

* * *

## Further Reading

*   [solmaz.io/google-eng-practices-github](https://solmaz.io/google-eng-practices-github) — Full guide adapted for GitHub/PR terminology (single-page, recommended starting point)
    
*   [The Standard of Code Review](https://google.github.io/eng-practices/review/reviewer/standard.html) — The core standard, canonical source
    
*   [How To Do A Code Review](https://google.github.io/eng-practices/review/reviewer/) — Full reviewer guide
    
*   [The CL Author's Guide](https://google.github.io/eng-practices/review/developer/) — Full author guide
    

Thanks, and happy coding.