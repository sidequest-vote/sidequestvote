# Updating the live board

`tracker.html` polls **`/data/quest.json`** every 20 s (cache-busted, `no-store`), pauses while the tab is hidden, and refreshes immediately when it becomes visible. Numbers and bars animate to the new values. There is no admin UI, password, or localStorage on the public site: totals change only by committing `data/quest.json`. Vercel serves `/data/*` with `Cache-Control: no-cache, must-revalidate, max-age=0` (see `vercel.json`).

## Privacy rules for the public file
`data/quest.json` is public. Never write payer names, Venmo handles, emails, message IDs, or payment IDs into it. A feed entry may carry at most an optional `who` of first name + last initial (e.g. `"Sam K."`) or `"Someone"`, and `amount` is optional. Preferred: totals and counts only, empty `feed`.

## `data/quest.json` schema
```json
{
  "questNum": 1,
  "title": "The Cybertruck: love it or hate it?",
  "status": "pre-launch",            // "pre-launch" | "live" | "closed"
  "sideA": { "label": "Love it", "color": "#52B788", "link": "/a", "venmoNote": "SIDE-A_Q001", "total": 0, "voters": 0 },
  "sideB": { "label": "Hate it", "color": "#D4A017", "link": "/b", "venmoNote": "SIDE-B_Q001", "total": 0, "voters": 0 },
  "flips": 0,                        // lead changes
  "feed": [],                        // optional, newest first, max 8: { "side": "a"|"b", "amount": 5, "who": "Someone", "flip": false, "at": "<ISO UTC>" }; amount and who optional
  "startedAt": null,                 // ISO UTC of first counted vote
  "lastVoteAt": null,                // ISO UTC of newest counted payment (drives "LAST VOTE Xs AGO")
  "updatedAt": "2026-10-22T19:58:31Z", // ISO UTC of this file update
  "history": []                      // finished quests, see "New quest"
}
```

### Per counted payment, the counter changes
| Field | Change |
|---|---|
| `sideA.total` **or** `sideB.total` (by note) | `+= gross amount paid` in dollars (before Venmo fees; number, cents allowed) |
| `sideA.voters` **or** `sideB.voters` | `+= 1` |
| `flips` | `+= 1` if the leader (higher `total`) changed because of this payment (ties are not a flip) |
| `lastVoteAt` | payment time, ISO UTC |
| `startedAt` | set to the first payment's time if `null` |
| `status` | `"live"` if it was `"pre-launch"` |
| `feed` (optional) | prepend `{side, amount?, who?, flip, at}`; keep max 8 |
| `updatedAt` | now, ISO UTC (always set on any commit) |

Note -> side: `SIDE-A_Q001` = side A, `SIDE-B_Q001` = side B (the note is `sideX.venmoNote`). Match case-insensitively. Anything else is not counted.

## `data/processed.json` (counter bookkeeping, `{"processedMessageIds": [], "needsReview": []}`)
- `processedMessageIds`: Gmail message IDs already counted. Check before counting, append after counting, in the same commit as the `quest.json` change, so no payment is counted twice.
- `needsReview`: Venmo emails the counter could not place (no/unknown note, personal payment, refund, odd amount). Entries like `{ "messageId": "...", "receivedAt": "<ISO UTC>", "reason": "no note" }`. No payer names or handles. These are NOT counted. Also add them to `processedMessageIds` once logged so they are not re-queued.
- This file is public in the repo. Keep it free of personal data (message IDs only).

## Update steps
1. Count new payments to @SideQuestVote by note as above.
2. Edit `data/quest.json` and `data/processed.json`.
3. Validate: `python3 -m json.tool data/quest.json && python3 -m json.tool data/processed.json`
4. Commit only those files (`Board: A $X / B $Y`), push to `main`. Vercel redeploys in ~30-60 s and open boards pick it up on their next 20 s poll.

## New quest
- Move the finished quest into `history` (`questNum, title, nameA, nameB, colorA, colorB, amtA, amtB, totalVoters, totalAmt, flips, startedAt, endedAt`).
- Set the new title, labels, colors, notes, zero totals, `startedAt`/`lastVoteAt` null, `flips` 0, empty `feed`.
- Change the two redirect destinations in **`vercel.json`** (`/a`, `/b`) to the new note, e.g. `note=SIDE-A_Q002`. Printed QR codes point to sidequest.vote/a and /b and never change.
- Never prefill an amount in the Venmo link.
