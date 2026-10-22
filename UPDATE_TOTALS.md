# Updating the live board (Phase 1)

The board (`tracker.html`) reads **`/data/quest.json`** on load and every 30 s. There is no admin UI on the public site; totals change only by committing that file.

## How the bot updates totals
1. Count payments to @SideQuestVote by note: `SIDE-A_Q001` -> side A, `SIDE-B_Q001` -> side B. Anything else goes to a manual review list (not counted).
2. Edit `data/quest.json`:
   - `sideA.total` / `sideB.total`: dollar totals (gross, before Venmo fees)
   - `sideA.voters` / `sideB.voters`: number of payments
   - `flips`: +1 each time the leader changes
   - `feed`: newest first, max ~8: `{ "side": "a"|"b", "amount": 5, "flip": false, "at": "<ISO UTC>" }` (no names/handles)
   - `status`: `pre-launch` | `live` | `closed`
   - `updatedAt`: ISO UTC timestamp of this update (drives "UPDATED X MIN AGO"; the dot pulses only if < 15 min old)
3. Validate: `python3 -m json.tool data/quest.json`
4. Commit `data/quest.json` only (`Board: A $X / B $Y`), push to `main`. Vercel redeploys in ~30-60 s.

## New quest
- Move the finished quest into `history` (`questNum, title, nameA, nameB, colorA, colorB, amtA, amtB, totalVoters, totalAmt, flips, startedAt, endedAt`).
- Set the new title, labels, colors, notes, zero totals.
- Change the two redirect destinations in **`vercel.json`** (`/a`, `/b`) to the new note, e.g. `note=SIDE-A_Q002`. Printed QR codes point to sidequest.vote/a and /b and never change.
- Never prefill an amount in the Venmo link.
