# GridTools Team Guidelines

## Preamble

These are guidelines. There is no guideline without exception. This is a living document, nothing is written in stone.

## Pull Requests

### Keep PRs small

Rationale:
- Smaller PRs are easier to review, increasing code quality. Reviews on large PRs tend to focus on details as the overall design is lost in the size. The possiblity to [learn from the review](#review-is-a-platform-for-learning-and-knowledge-transfer) is reduced as patterns are harder to understand.
- Small PRs encourage [faster review](#Review-in-a-timely-manner).

Reviewer and reviewee should decide together if it makes sense to split a large PR into smaller self-contained units. If a PR is conjectured to be inevitably large, the required time for the corresponding review should be taking into account in the sprint planning.

### PRs are squash merged

PRs are squash merged. Take the PR description as commit message of the squashed commit. This requires a complete, concise description of the changes in the description of the PR. Add the description when the PR is opened, it is valuable input for the reviewer.

Rationale:
- We see advantage of a clean, linear history where every commit passes tests. (Don't forget to [keep PRs small](#Keep-PRs-small)!)

## Code Reviews

### Review in a timely manner

Reviews are first priority. If you have a pending review, do it first before you start a new task.
Use context switches to do a review instead of going back to your previous (development) task (e.g. in the morning, after a meeting, after a break). Ideally, you get a first feedback within hours.

When a PR is opened, assign a reviewer and ping them on slack. Ping again if you have the feeling they forgot. As a reviewer, embrace being pinged. If you cannot review in a timely manner, reassign the review.

Rationale:
- The longer the feedback cycle, the harder the context switch.
- Fast feedback encourages [smaller PRs](#Keep-PRs-small).

### Review is a platform for learning and knowledge transfer

... for both the reviewer and the reviewee.

Often, it is helpful to do a pairing session to
- introduce the reviewer to the new changes.
- to discuss the review comments in more detail.

As a reviewer
- Give constructive feedback. If you know the right answer, explain it or use the "Suggest changes" feature in GitHub.
- Use "Suggest changes" for small/nitpicking comments (e.g. typos, formatting issues in comments).

Examples:

- Instead of

  > You have a tyop in typo.

  just fix the typo with a suggestion.

- Instead of

  > Please add a test.

  use

  > Please add a test, see test_file_x.py for an example of a similar test.


## Stand-up

### Stand-ups are not for justifying your time

Instead focus on work that is relevant for the team.
Especially, communicate if you need feedback/help.

### Take 1-on-1 discussions offline

Longer discussions on a particular topic are moved to after the stand-up with only the people involved.
