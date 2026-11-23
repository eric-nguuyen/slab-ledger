# Slab Ledger

A phone web app for logging trading-card purchases (mostly PSA graded slabs), built for Eric. It is installed from Safari with "Add to Home Screen" and used on an iPhone.

- Live: https://eric-nguuyen.github.io/slab-ledger/
- Hosting: GitHub Pages, from the `main` branch, root folder. A push to `main` goes live in about a minute.
- Backend: Firebase project `slab-ledger` (free Spark plan, no card on file). Eric does not want anything that costs money; stay inside the free tier and never suggest the paid plan.

## Files

- `index.html`: the whole app. Markup, CSS and JavaScript in one file, no build step.
- `icon.png`: home-screen icon, 180×180.
- No package.json, no framework. Keep it a single file unless there is a strong reason not to.

## Shipping a change

1. Edit `index.html`.
2. **Bump `const APP_VERSION="x.y"`** (search for `APP_VERSION`). On open, and when the app returns to the foreground, it fetches the live page, compares the version and reloads itself if newer. Without a bump, phones keep the cached copy for about 10 minutes (GitHub Pages cache). The version is also shown at the bottom of the History tab, so Eric can confirm he's on the latest.
3. Check the script parses: extract the last `<script>` block and run `node --check` on it.
4. Commit and push to `main`.
5. Tell Eric the new version number to look for.

## Libraries (loaded from CDNs)

- Firebase 10.12.2 compat builds: `firebase-app-compat`, `firebase-auth-compat`, `firebase-firestore-compat` from gstatic.
- `@zxing/library@0.21.3` UMD from unpkg, for the barcode scanner.
- Offline persistence is on (`enablePersistence`), so writes work offline and sync later.

## Auth

Email/password sign-in (Google sign-in was avoided because it is unreliable in iPhone home-screen apps). Sign up, sign in, password reset and sign out are in the app. `eric-nguuyen.github.io` is an authorized domain in Firebase Auth. Anyone with the link can create their own account, and each account sees only its own data. Eric may share the link with friends.

## Data model (Firestore)

Everything lives under `users/{uid}/...`. Security rules (set in the Firebase console, not in this repo):

```
match /users/{uid}/{document=**} {
  allow read, write: if request.auth != null && request.auth.uid == uid;
}
```

- `users/{uid}/purchases/{id}`: `{cert, name, cat, cents, qty, ts, day, photo?, photoBytes?, photoTs?}`
  - `cents`: total paid for the purchase, in cents (not per card). `qty`: number of cards, default 1.
  - `ts`: ms timestamp. `day`: local date string `YYYY-MM-DD` (used for all daily grouping).
  - `cat`: one of `pokemon`, `basketball`, `baseball`, `football`, `misc` (the `CATS` array; colors are `--c-<key>` CSS tokens).
  - `photo`, `photoBytes`, `photoTs`: set only when a photo exists.
- `users/{uid}/photos/{purchaseId}`: `{data, bytes, ts}`. One photo per purchase, a JPEG data URL shrunk on the phone to about 100–150 KB (max side 1000px). Stored in Firestore because Firebase Storage needs the paid plan. Loaded only when a purchase is opened.
- `users/{uid}/payouts/{YYYY-MM-DD}`: `{items:[{c, t}]}`. Payout amounts in cents for that day.
- `users/{uid}/meta/cash`: `{cents, since}`. Cash on hand.
- `users/{uid}/meta/settings`: `{autoClearDays}`. 0 = off, or 7, 14, 30.

## Features and the rules behind them

- **Log a buy** tab: PSA cert number (scan or type), card name (optional), category, total price paid, number of cards (stepper, resets to 1), optional photo. Warns if a cert number was already logged.
- **Scanner**: live camera via `getUserMedia`, decodes the area inside the corner brackets. A fast Code 128/39 pass runs every frame, with QR and a slower thorough pass every few frames. Opens at 1×; pinch to zoom up to 2.5×; zoom persists until the app is fully closed. "Use a photo instead" fallback. PSA cert numbers are extracted from the barcode text (a `cert/NNNN` URL or the digits).
- **History** tab: date picker, that day's buys and category breakdown, search by cert number, list of days with buys, Export/Import CSV, Photos section (usage of the free 1 GB, Clear all photos with confirmation, auto-clear Off/7/14/30 days, off by default), sign out, version label.
- **Profit** tab: adding machine for payouts per day (one at a time, or `120 + 45`), remove with Undo. **Profit = that day's payouts − that day's purchases** (always same day). Week/month/all-time totals count only days that have payouts entered. Profit CSV export.
- **History and Profit share one selected date** (`histDay`). Changing it on either tab changes both.
- **Top bar** (red, sticky): date row; tap to collapse/expand with a height animation (Web Animations API). Collapsed state is remembered (`localStorage bandMin`). Body shows Spent today, Profit today (once payouts exist), week/month/all-time spend, and Cash on hand.
- **Cash on hand** = `cash.cents − sum of purchases with ts >= cash.since`. "Set exact amount" resets `since` to now; "Add cash" adds to `cents`. Editing or deleting a purchase adjusts it automatically. **Payouts do not change cash on hand** (Eric's decision).
- **Entries**: tap to edit; swipe left to reveal Delete, long swipe deletes. Every delete has a 5-second Undo, which restores the photo too.
- Auto-update reloads only when nothing is in progress (no scanner, sheet or viewer open, no unsaved text in the forms).

## Style and conventions

- Phone-first, one column, max width 560px, safe-area insets respected. System fonts.
- Colors are CSS tokens on `:root`, with a dark-mode set under `prefers-color-scheme: dark`. Accent red `--accent`; `--gain` green and `--loss` red for profit.
- Money is stored and computed in integer cents; display with `money()`.
- Copy is plain and short. Errors say what to do next. No `alert`/`confirm`; confirmations are built into the page.

## Working with Eric

- He tests on his iPhone from the home-screen app and reports what he sees. Explain steps plainly and avoid jargon.
- For bigger features he likes a short plan first, then "execute". For small changes, just ship them.
- After shipping, tell him the version number to look for at the bottom of History.
