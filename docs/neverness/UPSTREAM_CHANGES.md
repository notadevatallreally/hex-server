# Upstream integration log

Neverness keeps `main` as the upstream-tracking branch and `neverness` as the maintained deployment branch.

## Integration policy

When IanUtley/hex-server changes:

1. Update the fork's `main` to the new upstream commit without adding Neverness-only changes there.
2. Compare the new upstream range against `neverness`.
3. Classify each relevant upstream change as:
   - `TAKE`
   - `TAKE WITH ADAPTATION`
   - `ALREADY FIXED LOCALLY`
   - `CONFLICTS WITH NEVERNESS`
   - `NOT RELEVANT`
   - `DEFER`
4. Merge/cherry-pick only after reviewing interactions with Neverness changes and persistent data.
5. Record the decision here, including the upstream commit/range and resulting Neverness commit.
6. Deploy from a specific Neverness commit/tag, not from an unreviewed moving branch.

## Baseline

- Initial upstream baseline: `ee0f06b3ee04b4eb678fe9e464e4d207271f67da`
- Initial maintained branch: `neverness`
- Existing local Neverness commits to preserve:
  - `3fd0d24` — `Fix Docker bootstrap SQLite self-lock`
  - `b8e9e9d` — `Fix unopened chest import during profile login`

## Change log

| Date | Upstream range | Classification / decision | Neverness result |
| --- | --- | --- | --- |
| 2026-09-16 | through `ee0f06b` | Establish fork baseline before further review/fixes | `neverness` branch created from exact upstream baseline |

Future entries should describe why a change was taken or deferred, especially when database migrations, protocol behavior, or persistent player state are involved.
