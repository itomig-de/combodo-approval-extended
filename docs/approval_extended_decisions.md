# "One answer per person per level" — design decisions

This document records the behavioural decisions made for the
`levelN_single_answer_per_person` option (branch `feature/single-answer-per-approval-level`,
commits `1c2e78e`, `3c42d27`, `f18e46a`). It exists so a reviewer, the repo owner, or a future
maintainer can see *what* was decided and *why*, without having to reconstruct it from the
implementation or the discussion that produced it.

## What problem this solves

Each approval level's approver list comes from two independent OQL queries on `ApprovalRule`:
the approver query (`levelN_rule`) and the substitute query (`levelN_substitute_query`). Nothing
guarantees these produce disjoint sets of people, so one person can end up holding more than one
vote on the same level. Three distinct situations can occur:

- **Case 1 — the approver query itself returns the same person twice** (e.g. a duplicate join
  row). Harmless in practice: upstream's answer-recording loop has no `break`, so a single click
  stamps every one of that person's own records at once. Only cosmetic side effects (duplicate
  case-log entries, duplicate notification mails).
- **Case 2 — a person is a main approver *and* a substitute inside a colleague's record.** A real
  defect: their own click stamps their own record, and their substitute link independently stamps
  the colleague's record. Two genuinely independent votes from one person.
- **Case 3 — a person is a substitute for two or more different approvers, with no approver
  record of their own.** Also a real defect, and structurally different from case 2: there is no
  "own record" to anchor on. The most common real-world trigger is a substitute query like
  `SELECT Person WHERE id = :approver->manager_id`, which naturally puts the same manager into the
  substitute list of every one of their reports.git

In both real-defect cases, a multi-approver level meant to require independent votes can end up
being carried by a single person, and an absent colleague's slot gets filled by someone who has
already voted.

## Scope of the option

- **Opt-in, per `ApprovalRule`, per level** (`level1_single_answer_per_person` /
  `level2_single_answer_per_person`) — not a global setting. Different rules, and different levels
  of the same rule, can behave differently.
- **Default `no`.** Existing rules and existing serialized approval schemes are unaffected;
  behaviour is byte-identical to today unless a rule explicitly turns the option on.
- **Scope is within one level only.** Approving at L1 and again at L2 remains allowed —
  `reuse_previous_answers` already governs whether an L1 answer is recycled at L2, and this option
  does not change that.
- **The approver/substitute list itself is left intact.** The option is enforced at answer time,
  not by filtering who gets solicited, so the audit trail still shows in which capacities a person
  was contacted.

## Precedence rule: the person's own slot always wins

When a person holds both a main approver record and a substitute record on the same level (case
2), their own approver slot always counts as their vote — regardless of which link they open
first:

- If they answer their own record first, their substitute link is refused afterwards.
- If they open their substitute link first, it is refused there and then, and their own slot
  remains fully answerable — they are not penalized for the order in which they happened to open
  their e-mails.

Their substitute capacity is treated as dead from the outset once the option is on; it is never a
race between the two links.

When a person holds **no** own record and is only a substitute for several approvers (case 3),
there is nothing to prefer, so a plain **first-answer-wins** rule applies: the first record they
answer keeps the vote, and every other record they cover remains open for its own approver,
another substitute, or the level's timeout default.

Case 1 needs no special rule: it is already handled correctly by upstream's no-`break` answer
loop, and the guard here is evaluated per *person*, so a person's own duplicate records are never
mistaken for two different people.

## What the refused person sees — e-mail / console path

This is the path this extension actually ships (approval links sent by e-mail, answered via
`approve.php`, including the console "Approve/Reject" actions). When a refused capacity is opened:

- The normal Approve/Reject form is **not** rendered.
- A new message is shown instead: `Approbation:AlreadyAnsweredOnThisLevel` ("You have already
  answered on this approval level").
- Nothing is recorded — the refusal is a no-op, not a silent re-stamp.

This mirrors the existing upstream pattern where `approve.php` already ends the page with
`Approval:Form:AnswerGivenBy` when a record was answered by someone else; a new key was needed
here because the situation is different — it is the person's *own* answer that already exists, or
their own approver role that supersedes their substitute role, not somebody else's answer.

## What the refused person sees — Portal path, and why it's different

The Portal (`ApprovalBrickController` in the upstream `approval-base` module) needed separate
treatment, for a concrete reason found while implementing the guard:

- The Portal's `Approve()`/`Reject()` calls route through `FindApprover($oReplier)`, which returns
  the **first** matching record in step order. In case 2, if the colleague's record happens to
  come first, a bare answer-time guard would route the person to the colleague's record as a
  *substitute* and then refuse it — meaning that person could **never** cast their own vote from
  the Portal at all, no matter how many times they clicked.
- `ApprovalBrickController` always renders a fixed `Approval:Portal:Success` message regardless of
  outcome, and offers no hook to show a different message for a refused attempt. It is also a
  Symfony controller that lives in the upstream `approval-base` module, not in this extension —
  patching it was **explicitly ruled out**: it is install-time-dependency code, not vendored, and
  any patch would be silently overwritten by the next `approval-base` update. It was also judged
  out of scope for this pull request.

Given those constraints, the decision was:

1. **Override `FindApprover()`** so that, when the option is on, a person's own approver record in
   the step always wins over a colleague's record that happens to list them as a substitute. This
   makes the Portal route the person to their own slot, so their real vote is recorded correctly
   instead of being silently swallowed.
2. **Override `IsActiveApprover()`** so that once a person's single answer for the step has been
   recorded, they simply stop being offered the action: the ticket drops off their Portal list
   (`ListOngoingApprovals`) and off the "Ongoing approval" report page. There is no message because
   there is nothing left to refuse — the action is no longer offered, rather than offered and then
   rejected.

This was chosen over the alternative of leaving the Portal guard bare (which would have produced a
misleading fake-success click with nothing recorded, repeating indefinitely for a case-2 person)
and over patching upstream `approval-base` (rejected for the reasons above).

## Suppressing redundant notifications

Once an answer would be refused, the person should not keep receiving reminders or
substitute-notification e-mails for that dead capacity:

- **`GetAwaitedReplies()`** (used for reminder mails and the reminder dialog) filters out any
  substitute entry that the guard would refuse, so the person is not reminded about a link they
  cannot use.
- **`OnTimeout()`** (which mails substitutes once their forward becomes due) is pre-filtered so it
  never mails a forward that:
  - belongs to someone who also holds their own approver record in the step (case 2), or
  - belongs to someone who has already answered elsewhere in the step, or
  - would be a second live substitute token for the same person in the same step (case 3 — at most
    one live substitute link per person per step at any time).
- A suppressed forward is marked as **sent** (not left pending), specifically so the scheme's
  timeout computation never keeps waiting on a deadline that will never trigger a mail. Leaving it
  pending instead of marking it sent was considered and rejected for exactly this reason.
- Case 3 has an ordering subtlety: at the time a forward becomes due, the person usually has not
  answered yet, so "already answered" alone would not catch the second capacity in time. The rule
  that closes this gap is the "at most one live substitute token per person per step" check above,
  which looks at forwards already recorded in the step, not just the batch coming due in the
  current call — because different approvers' forwards can come due at different times.

## Explicitly out of scope

These were raised and deliberately not pursued as part of this option:

- **Patching upstream `approval-base`** (e.g. to add a Portal-specific message). Rejected: it is
  an install-time dependency, not vendored in this repository, and any patch would be lost on the
  next `approval-base` update.
- **A Portal message in addition to hiding the action.** Once `IsActiveApprover()` hides the spent
  action, there is no dead-end click left to explain — the "hide it" approach was chosen instead
  of "offer it, then explain why it failed."
- **Bumping the module version.** Left for the repo owner to do as part of merging/releasing; this
  change goes out as a pull request only.
