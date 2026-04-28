# Merge Queue Validation Test

This repo is being used to empirically validate the two-tier required-checks model proposed in `Cogni-DAO/node-template`'s `docs/spec/merge-queue-config.md`.

## What we're testing

The spec claims GitHub's merge queue can express two distinct gates:

- **Tier 1 (PR gate)**: required to merge — includes checks whose workflows fire only on `pull_request`.
- **Tier 2 (queue gate)**: required to complete the queue — implicit subset whose workflows fire on `merge_group`.

The central uncertainty: when a required check's workflow has no `merge_group:` trigger, does GitHub's merge queue:

- **(a)** Wait forever for a status report that will never come (= the bug observed in `node-template` PR #1083), or
- **(b)** Skip the check on the queue ref because no workflow is registered to produce it?

## Test setup

Two diagnostic workflows in `.github/workflows/`:

- `mq-test-both.yml`: fires on `pull_request` AND `merge_group`. Always passes. Emits `mq-test-both` status check.
- `mq-test-pr-only.yml`: fires on `pull_request` only. Always passes. Emits `mq-test-pr-only` status check.

Branch protection (via Rulesets):

- Required-status-checks: **both** `mq-test-both` AND `mq-test-pr-only`
- Merge queue: enabled

## Expected outcomes

| Hypothesis | What we'd observe | Implication for spec |
|---|---|---|
| **(a) waits forever** | Open PR → CI green → click "Merge when ready" → queue stays in `AWAITING_CHECKS` indefinitely | Spec needs to pivot to stub-job pattern |
| **(b) skips PR-only** | Same flow → queue accepts after `mq-test-both` reports green on the merge_group ref | Spec is correct as written |

## How to reproduce

1. Open a docs PR against `main` (any tiny change).
2. Wait for both check workflows to report green on the PR.
3. Click "Merge when ready" or `gh pr merge --auto --squash`.
4. Observe queue state via:
   ```bash
   gh api graphql -f query='query { repository(owner:"Cogni-DAO", name:"test-repo") { mergeQueue(branch:"main") { entries(first:5) { totalCount nodes { state pullRequest { number } } } } } }'
   ```
5. Watch `Actions → MQ Test Both Events` — does a `merge_group` event run appear?
6. After ~15 minutes:
   - Queue accepted + merged → outcome (b), spec correct.
   - Queue still `AWAITING_CHECKS` → outcome (a), spec wrong.
