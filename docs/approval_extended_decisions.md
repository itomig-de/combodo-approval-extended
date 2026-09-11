# "One answer per person per level" — design decisions

This document records the behavioural decisions made for the
`levelN_single_answer_per_person` option. It exists so a reviewer, the repo owner, or a future
maintainer can see *what* was decided and *why*, without having to reconstruct it from the
implementation.

References to upstream below are to `approval-base/3.3.9`, file `main.approval-base.php`.

## What problem this solves

Each approval level's approver list comes from two independent OQL queries on `ApprovalRule`:
the approver query (`levelN_rule`) and the substitute query (`levelN_substitute_query`). Nothing
guarantees these produce disjoint sets of people, so one person can end up holding more than one
vote on the same level. Three distinct situations can occur:

- **Case 1 — the approver query itself returns the same person twice** (e.g. a duplicate join
  row). Harmless in practice: upstream's answer-recording loop (`OnAnswer`, `:1007-1039`) has no
  `break`, so a single click stamps every one of that person's own records at once. Only cosmetic
  side effects (duplicate case-log entries, duplicate notification mails).
- **Case 2 — a person is a main approver *and* a substitute inside a colleague's record.** A real
  defect: their own click stamps their own record, and their substitute link independently stamps
  the colleague's record. Two genuinely independent votes from one person.
- **Case 3 — a person is a substitute for two or more different approvers, with no approver
  record of their own.** Also a real defect. The most common real-world trigger is a substitute
  query like `SELECT Person WHERE id = :approver->manager_id`, which naturally puts the same
  manager into the substitute list of every one of their reports.

In both real-defect cases, a multi-approver level meant to require independent votes can end up
being carried by a single person, and an absent colleague's slot gets filled by someone who has
already voted.

## Where the duplicate is removed, and why there

Substitutes are materialised **exactly once**, at scheme creation: `AddSubstitutes()`
(`:238-260`) runs the substitute OQL per approver and stores the result under
`approvers[i]['forward'][]`, and `AddStep()` (`:129-180`) freezes that into the serialised `steps`
attribute. Nothing recomputes the list afterwards — `FindApprover()` (`:938`), `OnAnswer()`
(`:996`), `OnTimeout()` (`:1308-1325`) and `GetAwaitedReplies()` (`:1366-1385`) only ever read it.

The duplicate therefore arises at one known moment, and is removed at that same moment:
`ApplySingleAnswerPerPerson()` runs immediately after `AddStepFromQuery()` in
`ExtendedApprovalScheme::GetApprovalScheme()`, and drops the redundant `forward` entries before a
single passcode has been mailed.

The consequence is the point of the design: **the second capacity never exists.** It is never
mailed, never appears in a reminder, never shows up in the Portal, and cannot be answered. No
guard is needed at answer time, and no upstream method has to be overridden.

### Rejected alternative: enforcing at answer time

The obvious alternative is to leave both capacities in place and refuse the second answer in
`OnAnswer()`. It was implemented first and then discarded, because the guard drags in a chain of
further overrides:

- `DisplayApprovalForm()`, so the refused link shows a message instead of a form that silently
  does nothing.
- `FindApprover()`, because upstream returns the **first** matching record in step order
  (`:951-973`). In case 2, if the colleague's record happens to come first, the person is routed
  to it as a substitute and then refused — meaning they could never cast their own vote from the
  Portal at all. This override does not add a restriction; it repairs damage the guard itself
  causes.
- `IsActiveApprover()`, to stop offering a spent action in the Portal list.
- `GetAwaitedReplies()` and `OnTimeout()`, to suppress reminder and substitute mails for links
  that are already dead — the latter needing its own "mark as sent so `ComputeTimeout()` does not
  spin on a deadline nobody will act on" workaround.

That came to roughly 370 lines across six upstream overrides, versus roughly 70 lines in one
method here, none of which overrides upstream behaviour.

## Scope of the option

- **Opt-in, per `ApprovalRule`, per level** (`level1_single_answer_per_person` /
  `level2_single_answer_per_person`) — not a global setting. Different rules, and different levels
  of the same rule, can behave differently.
- **Default `no`.** Existing rules and existing serialised approval schemes are unaffected;
  behaviour is byte-identical to today unless a rule explicitly turns the option on.
- **Applies to schemes created after the option is turned on.** Because enforcement happens at
  scheme creation, switching the option on does not retro-fit approvals already in flight.
- **Scope is within one level only.** Approving at L1 and again at L2 remains allowed —
  `reuse_previous_answers` already governs whether an L1 answer is recycled at L2.
- **Only `forward` lists are modified, never `approvers`.** The number of votes the level requires
  is left untouched, so the option can only ever tighten the four-eyes requirement, never relax it.

## Precedence rules

- **A person's own approver record always wins over a substitute capacity.** Main approvers are
  seeded into the "already votes here" set before any `forward` list is examined, so a person who
  holds their own record is removed from every colleague's substitute list (case 2).
- **Otherwise, the first approver in the list keeps the shared substitute** (case 3). The
  substitute stays attached to exactly one approver instead of being able to pick a slot at answer
  time. The approvers left without a substitute fall back to their own answer or to the level's
  timeout default — which is the same outcome an answer-time guard produces, only with the
  uncovered slot chosen deterministically instead of by whoever clicks first.
- **Case 1 needs no handling**: it is already correct upstream, and the de-duplication is keyed on
  the person, so a person's own duplicate records are never mistaken for two different people.

## Interaction with `exit_condition`

The option is only meaningful when a level actually requires more than one vote:

- `first_reply` (`EXIT_ON_FIRST_REPLY`): `GetStepResult()` returns on the very first answer
  (`:1424-1428`), so the level is over before anybody could vote twice. The option is a no-op.
- `first_reject` (`EXIT_ON_FIRST_REJECT`): everybody must approve — this is where the reported
  defect occurs, and where the option matters.
- `first_approve` (`EXIT_ON_FIRST_APPROVE`): symmetrically, everybody must reject.

The option is deliberately *not* auto-derived from `exit_condition`: an administrator may switch
the exit condition later, and an explicit flag keeps the stored scheme's behaviour predictable.

## Traceability

Two keys are written into the step data purely for diagnostics; no other code reads them:

- `single_answer_per_person` — records that the option was active when this scheme was built.
- `single_answer_dropped_forwards` — the substitute capacities that were removed, each with the
  approver they would have covered, so the audit trail still shows in which capacities a person
  would have been contacted.

Removals are additionally logged via `IssueLog::Info()` under the `combodo-approval-extended`
channel.

## Explicitly out of scope

- **Patching upstream `approval-base`.** It is an install-time dependency, not vendored in this
  repository; any patch would be lost on the next update. The chosen design needs no patch.
- **Retro-fitting approvals already in flight.** See "Scope" above.
- **Bumping the module version.** Left for the repo owner to do as part of merging/releasing.
