# Discovery Report — Email correction for Sanni Koponen (DRYEYE_FI_KUIVASILMA)

**Status:** DISCOVERY ONLY — nothing has been modified. Awaiting PO approval.
**Date:** 2026-09-20
**Requested by:** PO work order "SANNI KOPONEN — EMAIL CORRECTION DISCOVERY ONLY"

| Field | Value |
|---|---|
| Customer | Sanni Koponen |
| Wrong email (current) | `snnhtnn@gmail.com` |
| Correct email (target) | `sanni.koponen1@gmail.com` |
| Course | `DRYEYE_FI_KUIVASILMA`, scope `CH01-CH06` |
| Entitlement | Real Shopify purchase, `access_type = purchased` — must stay `purchased` |

---

## 0. Blocking finding — the data is not reachable from this session

Questions 1–4 and 7 ask for live records (Shopify order/customer, WordPress user,
entitlement row, mapping tables). **None of these are available here**, so they are
reported as open, not answered. I will not guess IDs for a record that will later be
edited in production.

### What was checked

| Source | Result |
|---|---|
| `leatest-app/vision-academy-preview` (the only repo in this session) | Static landing-page prototype only: `README.md` + a single `index.html`. After stripping embedded base64 images it is ~23 KB of markup with one `<script>` block that drives a modal, an accordion and a scroll-reveal observer. |
| Keyword sweep over `index.html` — `shopify`, `wordpress`, `wp_`, `entitlement`, `access_type`, `valid_until`, `commerce`, `DRYEYE`, `kuivasilma`, `koponen`, `snnhtnn`, `order_id`, `customer_id` | **0 hits for every term.** |
| Network/storage surface — `fetch(`, `XMLHttpRequest`, `localStorage`, `sessionStorage`, any `http(s)://` endpoint | **0 hits.** No backend calls at all. |
| Full git history (`--all`, both commits, both branches) | Only `README.md` and `index.html` have ever existed. |
| Other repos on the account — `leatest-app/leatest-valitsin`, `leatest-app/leatest-simulaattori` | Exist, but are **out of scope for this session** and access is denied. They are not added, because the work order did not ask for them. |

**Conclusion:** the Shopify store, the WordPress installation, the entitlement store and
any Commerce Bridge / mapping table live outside this repository. This repo is a marketing
prototype and holds no commerce, identity or entitlement code.

### Access needed to close questions 1–4 and 7

1. **Shopify admin** (or Admin API read scopes `read_customers`, `read_orders`) — to find
   the customer by email `snnhtnn@gmail.com` and the order carrying `DRYEYE_FI_KUIVASILMA`.
2. **WordPress admin** (or DB/WP-CLI read access) — to resolve `ID`, `user_login`,
   `user_email`, `user_registered` for that customer.
3. **The entitlement store** — the plugin/table holding `course_id`, `scope`,
   `access_type`, `valid_from`, `valid_until`, and what its foreign key is
   (`wp_user_id` vs. `email`). This single fact decides the whole risk profile.
4. **The Commerce Bridge repo/config** — to see whether it persists an email column and
   which direction it syncs.

If the platform lives in a repo I can be given, add it to the session and I will close
1–4 and 7 with real values in a follow-up pass. Everything below is then confirmed rather
than assumed.

---

## 1–4. Open items (no values available)

| # | Question | Status |
|---|---|---|
| 1 | Shopify order/customer record connected to this purchase | **UNKNOWN** — needs Shopify access |
| 2 | Current Shopify customer/order email | **UNKNOWN** — expected `snnhtnn@gmail.com` on both, but customer email and per-order contact email are separate fields and must be read separately |
| 3 | WordPress user ID, login, email | **UNKNOWN** — needs WP access |
| 4 | Entitlement ID, `course_id`, `scope`, `access_type`, `valid_from`, `valid_until` | **UNKNOWN** — needs entitlement store access |

---

## 5. Is changing the WordPress user email alone safe?

**Conditionally yes for the entitlement, but not sufficient on its own.**

- **Entitlement safety** — if the entitlement row's foreign key is the numeric
  `wp_user_id` (the normal design), then updating `user_email` touches nothing on the
  entitlement: `entitlement_id`, `course_id`, `scope`, `access_type`, `valid_from` and
  `valid_until` are all untouched. The purchase stays `purchased`.
- **Entitlement risk** — if the entitlement (or a pending/unclaimed entitlement) is keyed
  on **email**, a WP-side-only change orphans it instantly. **This must be verified before
  any edit** — it is the single highest-risk unknown.
- **System-wide** — a WP-only change leaves Shopify still holding `snnhtnn@gmail.com`.
  That split is the real danger: a later re-sync or webhook that matches by email will
  either fail to find the user or **provision a second user**, which violates the
  "no duplicate user / no duplicate entitlement" requirement.

**Two hard rules for the WP edit:**
- Change `user_email` only. **Do not change `user_login`** — WP treats it as immutable and
  many integrations key on it.
- **Do not delete and recreate the user.** That is what would destroy the entitlement.

---

## 6. Does the Shopify email also need to change?

**Yes for the Customer record. No for the historical order.**

- **Customer record** — should be updated to `sanni.koponen1@gmail.com` so future orders,
  marketing and account flows resolve to the right person and match WordPress.
- **Historical order** — Shopify stores a contact email **on the order itself**, snapshotted
  at purchase. Changing the customer's email does *not* rewrite it. Leaving it is normally
  correct: the order is a financial record and the requirement is that "Shopify order
  history remains intact". Edit the order's email **only** if the Commerce Bridge matches
  on order email rather than customer/order ID — otherwise leave it alone.
- **Pre-check** — Shopify enforces unique customer emails. If a customer already exists at
  `sanni.koponen1@gmail.com`, the update will be rejected and this becomes a customer-merge
  task, not an email edit. Check before scheduling.

---

## 7. Does Commerce Bridge or a mapping table store the email separately?

**UNKNOWN — must be verified, and it is the second highest-risk unknown.**

Bridges of this kind typically persist a mapping row along the lines of
`(shopify_customer_id, shopify_order_id, wp_user_id, email, synced_at)`. If `email` is
stored denormalised there, it must be updated in the same maintenance window, or the next
webhook mismatches and may re-provision.

Also needs answering: **which side is the source of truth**, and does the sync run
one-way (Shopify → WP) or both ways? That determines the safe ordering in §9.

---

## 8. What would an email change affect?

| Area | Impact | Notes |
|---|---|---|
| **Login** | Low, if WP login is by username. Medium if by email. | The user must use the new address if they sign in by email. `user_login` stays as-is and keeps working either way. |
| **Activation / password links** | **Real impact.** | Any activation or set-password email already sent went to `snnhtnn@gmail.com` and is unrecoverable. Existing tokens stay valid but sit in the wrong inbox. A **fresh set-password link must be sent** to the new address after the change — treat this as part of the work, not a follow-up. |
| **Order mapping** | None, if mapped by Shopify customer/order ID. **Breaks, if mapped by email** and only one side is updated. | Decided by §7. |
| **Entitlement validity** | **None**, provided the user record is updated in place and not recreated. `valid_from` / `valid_until` are attributes of the entitlement row. | |
| **`access_type`** | **None.** No step in the plan below writes this column. It stays `purchased`. | |
| **Future emails** | Go to the new address once the WP user and the Shopify customer are aligned. | A resend of the *historical* Shopify order confirmation still goes to that order's snapshotted email unless the order is also edited. |
| **Duplicate risk** | The main threat, and it comes from the gap between the two edits, not from either edit. | Mitigated by the maintenance window in §9. |

---

## 9. Proposed change plan (for approval — not executed)

### Pre-checks (all must pass before any write)
- **P1.** Confirm no WP user already exists at `sanni.koponen1@gmail.com`. WP enforces
  unique emails; a collision means a merge, not an edit. **Stop if a collision exists.**
- **P2.** Confirm no Shopify customer already exists at `sanni.koponen1@gmail.com`. Same
  reasoning. **Stop if a collision exists.**
- **P3.** Confirm the entitlement's foreign key (`wp_user_id` vs. `email`) — §5.
- **P4.** Confirm whether the bridge/mapping table stores email, and its sync direction — §7.
- **P5.** Snapshot and record current values of every field to be touched, plus the full
  entitlement row (ID, `course_id`, `scope`, `access_type`, `valid_from`, `valid_until`).
  This snapshot **is** the rollback source.
- **P6.** Take a WP database backup (or at minimum export `wp_users`, `wp_usermeta` for
  that ID, and the entitlement + mapping rows).

### Execution — one maintenance window, all steps back to back
Pause the Shopify→WP sync / webhook consumer first, if the bridge has that control. The
duplicate-user risk lives entirely in the gap between step 1 and step 3; the window closes it.

| Step | Action | Verification |
|---|---|---|
| **1** | Shopify: update **Customer** record email → `sanni.koponen1@gmail.com`. Leave the historical order untouched. | Customer shows new email; order still shows original email and its line items. |
| **2** | Commerce Bridge / mapping table: update the stored email **only if** §7 found one. | Mapping row still points at the same `wp_user_id` and `shopify_order_id`. |
| **3** | WordPress: update **`user_email` only** on the existing user ID. Via `wp user update <ID> --user_email=sanni.koponen1@gmail.com` or WP admin. Do **not** touch `user_login`, do **not** create a user. | Same user ID; `user_login` unchanged. |
| **4** | Re-enable the sync / webhook consumer. | No re-provisioning fires. |
| **5** | Re-read the entitlement row and diff against the P5 snapshot. | `entitlement_id`, `course_id`, `scope`, `access_type = purchased`, `valid_from`, `valid_until` **all byte-identical**. |
| **6** | Send a fresh set-password / login link to `sanni.koponen1@gmail.com`. | Customer confirms login and sees CH01–CH06. |
| **7** | Duplicate sweep: search WP users and the entitlement store for **both** addresses. | Exactly one user, exactly one entitlement for `DRYEYE_FI_KUIVASILMA`. No complimentary/reviewer row created. |

### Rollback
No row is ever created or deleted — every step is a single-field update — so rollback is a
field-level revert from the P5 snapshot, applied **in reverse order** (3 → 2 → 1), then
re-enable sync and re-verify the entitlement row against the snapshot.

**Rollback triggers:** entitlement row differs from snapshot in any column; `access_type`
is no longer `purchased`; a second user or second entitlement appears; the customer cannot
log in after step 6 and step 3 is the cause.

**Not recoverable by rollback** (so treat as one-way): emails already dispatched to either
address during the window, and any Shopify order-email edit — which is why §6 recommends
leaving the historical order alone.

### Requirements traceability

| Required outcome | How the plan satisfies it |
|---|---|
| Sanni uses `sanni.koponen1@gmail.com` | Steps 1–3 + step 6 |
| Purchased entitlement unchanged | No step writes the entitlement table; step 5 proves it |
| Validity unchanged | `valid_from` / `valid_until` never touched; step 5 proves it |
| Shopify order history intact | Historical order deliberately not edited (§6) |
| No duplicate user | P1 pre-check + sync paused during the window + step 7 |
| No duplicate entitlement | Entitlement never re-created; step 7 sweep |
| No complimentary/reviewer access | No provisioning path is invoked at any step |

---

## Recommendation

Approve **P1–P6 first** as a read-only pass — they need Shopify, WordPress and Commerce
Bridge access that this session does not have. P3 and P4 in particular can still change the
plan: if the entitlement or the bridge keys on email, the ordering in the execution table
matters much more, and P1/P2 can turn this into a merge task instead of an edit.

**No changes will be made until the PO approves.**
