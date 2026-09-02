# Bugs found

Add one section per issue. Bug 1 is filled in to show the format — fix it, then write what you changed. Copy the blank template for the rest.

Keep this file in the repo and **commit it** with your fixes.

---

## Bug 1

**How to reproduce:** Open the app. The expense list says "Newest first". The first row is Wine (7 Mar). Board game (15 Mar) is further down.

**What is wrong:** The list is showing oldest expenses first. Newest should be at the top.

**What I changed:** In `src/components/ExpenseList.jsx`, the sort comparator was `dateValue(a.date) - dateValue(b.date)` which puts the smallest (oldest) date first. I reversed it to `dateValue(b.date) - dateValue(a.date)` so the most recent expense appears at the top of the list.

---

## Bug 2

**How to reproduce:** Open the app. In the Filters section, select any member from the "Paid by" dropdown. The list does not filter — all expenses remain visible regardless of which member you pick.

**What is wrong:** The `paidBy` value from the `<select>` element is a string (e.g. `"1"`), but each expense's `paidBy` field is a number (e.g. `1`). The strict `!==` comparison always returns `true` because a string never strictly equals a number, so no expenses are ever filtered out.

**What I changed:** In `src/App.jsx`, inside the `filtered` useMemo, I changed `e.paidBy !== paidBy` to `e.paidBy !== Number(paidBy)` so the comparison is number-to-number.

---

## Bug 3

**How to reproduce:** Open the app. Look at the Balances panel. Alice paid for most things and should be shown as "is owed" money, but she is labeled "owes". The people who owe money are labeled "is owed".

**What is wrong:** The label logic in `BalancesPanel.jsx` was backwards. A positive balance means the person paid more than their share — they are owed money. A negative balance means they consumed more than they paid — they owe money. The original code had `bal > 0` labeled as `"owes"` and `bal < 0` labeled as `"is owed"`, which is the opposite of the sign convention used throughout the app.

**What I changed:** In `src/components/BalancesPanel.jsx`, I swapped the two branches so that `bal > 0` shows `"is owed"` (with class `"owed"`) and `bal < 0` shows `"owes"` (with class `"owe"`).

---

## Bug 4

**How to reproduce:** Load the app, add an expense, then refresh the page. After the refresh the expense list reorders incorrectly (dates stop sorting properly) because dates come back as plain strings instead of Date objects.

**What is wrong:** In `loadState()`, when data is already stored in localStorage the function just did `return JSON.parse(raw)`. JSON serialization turns `Date` objects into ISO strings, and `JSON.parse` gives them back as plain strings — not Dates. The fresh-seed path correctly calls `hydrate()` to convert those strings back to `Date` objects, but the localStorage branch skipped this step entirely.

**What I changed:** In `src/state/store.js`, I changed `return JSON.parse(raw)` to `return hydrate(JSON.parse(raw))` so that dates are always converted back to `Date` objects when loading from localStorage, matching what the initial seed path already did.

---

## Bug 5

**How to reproduce:** Open the app. Look at the Balances panel for the Uber expense (paid by member 4, split only between members 1 and 2 — the payer is not in splitWith). Member 4's balance is wildly wrong; they appear to owe money even though they were the one who paid.

**What is wrong:** In `computeBalances`, after the share-subtraction loop correctly debits each person in `splitWith`, there was an extra `if` block that ran whenever `paidBy` was not in `shares`. That block subtracted `amount / n` from the payer's balance again, double-charging them even though the loop above had already handled everything correctly.

**What I changed:** In `src/lib/balances.js`, I deleted the entire `if (!(exp.paidBy in shares) && !(String(exp.paidBy) in shares))` block. The share-subtraction loop is sufficient on its own.

---

## Bug 6

**How to reproduce:** Add an expense for $10.00 split equally 3 ways. Check the balances — the numbers don't add up to $10.00 (they total $9.99 because each of the three shares is rounded down to $3.33).

**What is wrong:** `splitEqual` called `.toFixed(2)` on each share independently. When the division doesn't produce an exact cent (e.g. `10 / 3 = 3.3333…`), every share gets truncated to `$3.33`, and the three shares only sum to `$9.99` — one cent is lost.

**What I changed:** In `src/lib/money.js`, I updated `splitEqual` to calculate the base rounded share, then compute the difference between `amount` and `baseShare * n`, and add that leftover to the last person's share. This guarantees the shares always sum exactly to the original amount.

---

## Bug 7

**How to reproduce:** Create a scenario where one debtor owes exactly the same amount as one creditor is owed (e.g. Alice owes $15 and Bob is owed $15). Open the Settle Up panel — that transfer is completely missing from the list.

**What is wrong:** In `suggestSettlements`, the algorithm has three branches: `d.amount > c.amount`, `d.amount < c.amount`, and `else` (equal). The `else` branch was only incrementing both pointers but never pushing the transfer. So when a debtor and creditor have equal amounts, the settlement is silently dropped and those people are left with non-zero net balances.

**What I changed:** In `src/lib/settle.js`, I added a `transfers.push(...)` call in the `else` branch before the `i += 1; j += 1` lines, so the exact-match pair produces a transfer just like the other two cases do.

---

## Bug 8

**How to reproduce:** Add several expenses. Apply a filter (e.g. filter by category). Then try to delete or edit an expense amount in the filtered list. The wrong expense gets modified — often one that is not even visible in the current filtered view.

**What is wrong:** `ExpenseList` sorts and filters a copy of the expenses array, then passes the *index within that sorted/filtered array* to `onDeleteAt`/`onUpdateAt`. `App.jsx` then dispatches that index directly into `state.expenses`, which is an entirely different array (unfiltered, unsorted). When the two arrays are in different orders, the index points to the wrong item.

**What I changed:** I switched from index-based to id-based addressing across three files:
- In `src/state/store.js`, `DELETE_EXPENSE` now filters by `action.id` and `UPDATE_EXPENSE` maps by `action.id`.
- In `src/App.jsx`, the dispatch calls pass `id` instead of `index`.
- In `src/components/ExpenseList.jsx`, I replaced `index` with `expense.id` in the key, `onDelete`, and `onSaveAmount` callbacks.

---

## Bug 9

**How to reproduce:** Open the app. In the Summary card, note the "Paid so far" list of members. Click "Add member", type a name, and hit Add. The new member does not appear in the "Paid so far" list until you also add or delete an expense.

**What is wrong:** The `perPerson` useMemo in `SummaryCards.jsx` had `[expenses]` as its dependency array. The computation also maps over `members`, but since `members` wasn't listed as a dependency, React didn't re-run the memo when a new member was added — only when expenses changed.

**What I changed:** In `src/components/SummaryCards.jsx`, I changed the useMemo dependency array from `[expenses]` to `[expenses, members]` so the "Paid so far" list recalculates immediately whenever a member is added.

---
