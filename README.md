# Prime Saas POS

A multi-tenant point-of-sale platform built with Next.js (App Router) and a Neon/Postgres backend (own session-cookie auth in `src/lib/neonAuth.ts`, data access via `neonQuery`/`neonPool` in `src/lib/db/neon.ts`). Multiple organizations (cafes, shops — any single-location retail business) each get their own isolated menu, inventory, sales history, and settings, under one shared app.

## Roles

- **Super Admin** — manages the platform itself: creates organizations and their first Admin account, can suspend/reactivate any organization. Doesn't belong to any single organization and never touches its day-to-day data.
- **Admin** — full control of their own organization: POS, Inventory, Analytics, History, Settings, and the **Team** page, where they create Moderator accounts and grant each one a precise set of permissions.
- **Moderator** — sees and can act on exactly the pages/actions their Admin has granted, nothing more. Permissions are set per page (POS, Inventory, Analytics, History, Settings) and per action (View / Add / Edit / Delete).

## Features

- **Real login** — email + password, verified against a bcrypt hash in the Neon `users` table, with a signed httpOnly session cookie (`src/lib/neonAuth.ts`). No more hardcoded credentials; every person has their own account, created by a Super Admin (for an org's first Admin) or an Admin (for their Moderators).
- **Multi-tenant data isolation** — every organization's categories, items, orders, and settings carry an `org_id` column in Postgres. Every `/api/data/*` route re-checks the caller's session and org membership server-side before reading or writing — see **Security model** below.
- **Per-page, per-action permissions** — an Admin can give a Moderator, say, View + Add on Inventory but no Edit/Delete, and View-only on Analytics, with History hidden entirely. The sidebar only shows pages a Moderator can view; buttons for actions they can't perform are hidden.
- **Public landing page** (`/`, logged out) — explains the platform and the three roles (Super Admin / Admin / Moderator), each with its own "Sign in" link — all leading to the same `/login` form, since one account always has exactly one role and the app routes you automatically. Visiting any other page while logged out redirects here (via `/login`).
- **Home dashboard** (`/`, logged in) — the same route shows a role-aware dashboard instead: an overview row (today's sales/orders, low/out-of-stock counts — each tile only shown if the signed-in Admin/Moderator can view the relevant page) plus quick-access cards to whichever pages they can view. Super Admin is redirected straight to `/super-admin`, since they don't belong to an organization.
- **Self-service account management** — click your name at the bottom of the sidebar to change your own password or permanently delete your own account (both require re-entering your current password, verified server-side against the stored bcrypt hash — see `src/app/api/auth/change-password/route.ts` and `src/app/api/auth/delete-account/route.ts`).
- **POS screen** (`/pos`) — search or tap items by category, build a cart, apply a discount, choose **Paid/Unpaid** and, if paid, **Cash/Online**, complete the sale. Cash shows a "Cash received"/"Change" calculator; Online skips it (assumed paid in full elsewhere); Unpaid skips payment collection entirely and prints a bold **UNPAID** marker on the receipt instead of a payment line — useful for tab/credit sales settled later. Every order stores `paymentMethod` and `paymentStatus` (old orders without these default to `cash`/`paid`, matching how checkout always worked before this).
- **Custom price & custom line items** — in the cart, clicking a line's price (shown with a dotted underline) turns it into an editable field, so an operator can override an item's price for just this one sale (a negotiated discount, a damaged-item markdown) without touching its real catalog price in Inventory. A **Custom** button above the cart adds an entirely off-catalog line — a name and amount for something not in the menu at all. Custom lines carry a `custom-` prefixed id and are deliberately invisible to all stock logic (checkout, refund, cancel, permanent delete) since there's no inventory item behind them to adjust.
- **Inventory** (`/inventory`) — categories + menu items with stock/low-stock tracking, photos, search, and Import/Export to Excel. Adding/renaming an item blocks exact-name duplicates (case-insensitive); the table scrolls internally with a sticky header once it gets long, instead of growing the whole page.
- **Image uploads (Cloudflare R2)** — the shop logo (Settings) and menu item photos (Inventory) upload to your R2 bucket via a server-side API route, not straight from the browser, so the R2 secret key never reaches client JS. See **Image uploads** below.
- **Analytics** (`/analytics`) — a read-only dashboard: stat tiles with deltas and sparklines, a 7-day revenue chart, all-time top sellers.
- **History** (`/history`) — order lookup filterable by Today/Week/Month/All/custom date range, search by order #, Export to Excel/Word, and Refund/Cancel actions. **Delete record** permanently removes a single order (restocking any items it still holds); **Clear history** does the same for every order currently shown in the filtered view — both need the same permission as Cancel, and both are irreversible, unlike Cancel/Refund which just change an order's status and keep the record.
- **Toast notifications** (`src/context/ToastContext.tsx`) — success/error toasts in the bottom-right corner, used across Inventory (add/edit/delete, duplicate-name checks, Excel import) and History (cancel/delete) so actions give clear feedback beyond an inline banner.
- **Confirmation modals, not browser popups** (`src/context/ConfirmContext.tsx`) — every destructive action app-wide (deleting an item, a category, an order record, clearing history, removing a team member or an org's admin, revoking a token/license) asks via a styled in-app modal (`useConfirm()`, `await confirm("…")`) instead of the native `window.confirm()` dialog, which looks and behaves inconsistently across browsers.
- Categories (Inventory) also block exact-name duplicates and scroll internally past a fixed height, same treatment as the item table.
- **Receipts & QR verification** — a printable 80mm receipt per sale with a QR code linking to a public `/verify/[orgId]/[id]` page that looks the order up live in Neon — no login needed to verify a receipt is genuine.
- **Item QR tags & camera scanner** — from Inventory, each item has a **Tag** link (`/tag/[orgId]/[itemId]`) that prints a small 80mm label with the item's name, price, and a QR code. In the POS screen, the **Scan** button opens the device camera and adds the scanned item straight to the cart — handy on a phone or a USB/camera-based scanner, no app install needed. See **QR item tags & scanning** below for how the QR payload works and a browser camera caveat.
- **Pakistani Rupee formatting**, **tax & service charge** — see inline comments in `src/lib/currency.ts`, `src/lib/pricing.ts`.
- **Works offline, including checkout and printing** — see **Offline checkout** below for how, and what's different about an offline sale.
- **Cash drawer support** — a "Drawer" button in the POS cart panel can kick open a till wired into the receipt printer, via WebUSB. See **Cash drawer** below for the real hardware/browser constraints before relying on it.
- **Prime Saas POS / Primecodia Tech branding** — the platform name appears on the login page and the Super Admin area; each organization's own shop name/logo (set in their Settings) appears everywhere else for that org, including receipts. "Built by Primecodia Tech" appears on login, the sidebar footer, and every receipt.

## Setting up a fresh Neon database

1. Create a Neon Postgres database and put its connection string in `.env.local` as `DATABASE_URL`.
2. Run the schema once: `psql "$DATABASE_URL" -f scripts/neon-schema.sql` — additive `create table if not exists` only, safe to re-run.
3. Set `AUTH_SECRET` in `.env.local` (signs the session cookie — see `src/lib/neonAuth.ts`).
4. Visit `/setup-super-admin` once to create the Super Admin account, then `/super-admin` to create your first organization and its Admin.

If you're migrating off an existing Firestore-backed install, `scripts/migrate-firestore-to-neon.mjs` (and the Super Admin "migrate-to-neon" page) does a one-way, idempotent sync of the old Firestore data into these Neon tables — it still needs `FIREBASE_ADMIN_*` env vars to read the old project, and is a temporary tool kept around only for that purpose.

## Security model — what's enforced where

- **Tenant isolation (hard boundary):** every `/api/data/*` route checks the caller's session (httpOnly cookie, `getSessionUser`) and compares their `org_id` against the row(s) being read/written — a non-Super-Admin can never touch another organization's data, enforced server-side on every request, not just hidden in the UI.
- **Role gates:** only Super Admin can create/suspend organizations or promote accounts; only an Admin can create Moderators in their own org — checked in the relevant API route, not left to the frontend.
- **Per-action Moderator permissions (UI only, documented limitation):** whether a Moderator sees an "Add item" button or a "Delete" link is decided by the app's frontend code reading their `permissions` object. The API routes do **not** separately re-check "does this Moderator have delete permission on Inventory" — a Moderator's database access is "full read/write on their own org," same as an Admin, same as the UI just doesn't expose everything. For an internal team tool this is a reasonable, common trade-off — just don't treat the permission matrix as a hard security boundary against a technically sophisticated Moderator.
- **Public receipt verification:** an individual order row and the shop settings row can be read by anyone who has the exact `orgId`/id (random ids — unguessable, not published anywhere except in the QR code itself) via a public GET route. *Listing* orders or settings is still fully private to org members.

## Licensing — trials and tokens

Every organization needs a valid trial or license to write data (reads always work, so a lapsed org never loses visibility into its own history):

- **Trial**: a brand-new organization gets a 14-day trial automatically from the moment Super Admin creates it (`TRIAL_DAYS` in `src/lib/licensing.ts`).
- **License tokens**: from `/super-admin`, Super Admin picks an organization and clicks **Generate token**, choosing a duration (1/3/6/12 months, a custom number of days, or **Lifetime**). This creates a one-time code — shown once, copy it immediately — stored in the `license_tokens` table, tied to that organization and unused until redeemed.
- **Lifetime licenses**: "Lifetime" isn't a special case in the data model — it's just a 100-year token (`LIFETIME_DAYS` in `src/lib/licensing.ts`), reusing the exact same generate/redeem code as any other duration. Anywhere an expiry date would normally show, `isLifetimeExpiry()` checks if there's more than 10 years of runway left and displays "Lifetime" instead of a year-2126-ish date.
- **Redeeming**: once the trial is within 3 days of ending (or has already ended), the organization's Admin sees a banner at the top of every page with a field to paste the token code. Redeeming extends `licenseExpiresAt` to *now + the token's duration* (not stacked on top of any previous expiry) and immediately unblocks writes. Moderators see the same banner but without a redeem field — only an Admin can redeem.
- **Enforcement note**: license-expiry write-blocking used to be enforced in Firestore security rules at the database level; the Neon/Postgres rewrite enforces it in the `/api/data/*` route handlers instead (application-level, not database-level) — functionally the same outcome for this app, but worth knowing if you're auditing where the check actually lives. A token can only be redeemed by that same organization's Admin, only once, and only if it hasn't already been used (enforced transactionally in `redeemLicenseToken`, `src/lib/licensing.ts`).
- Super Admin's own access, and everyone's ability to *read* existing data, is never gated by licensing.
- **Demo account print cap**: while an organization has no active paid license (on trial, or expired), printing a receipt is capped at `TRIAL_PRINT_LIMIT` (5) lifetime prints — after that, the Print button on `/receipt/[orgId]/[id]` is disabled and the page won't auto-print either, with a message telling Admins to add a token (Moderators are told to ask their Admin). The count lives on the `organizations.trial_print_count` column, bumped by `recordTrialPrint` (`src/lib/licensing.ts`). Once a license is active, printing is unlimited and the counter stops moving.
- **Always-visible trial reminder**: unlike the "expires soon" banner for a paid license, the trial banner shows for the entire trial (not just the final days) — "You're on a trial — using a demo account" plus days left and prints remaining — so a demo organization is never mistaken for a fully licensed one.
- **Admin's own view** (`/users`, bottom of the page): a "Licensing" card shows the org's current status plus every token ever generated for it (code, duration, redeemed/unused, dates), with an **Apply** button right there for any unused one — no need to retype a code given verbally.
- **Super Admin's view** (`/super-admin`): click **Details** on any organization row to expand admin email, created date, license status, demo prints remaining, the full member list (every Admin/Moderator with role and active/disabled status), and that org's token history.
- **Revoking a token**: in that same Details panel, Super Admin can click **Revoke** on any unused token to permanently delete it so it can never be redeemed — for a code generated by mistake or one that needs to be invalidated before it's handed over. A redeemed token shows **Delete** instead, which only clears it from the history list — deleting it can't claw back access it already granted.
- **Revoking an organization's license** ("banning" the org itself): also in the Details panel, **Revoke license now** immediately sets the org's access to expired — trial or paid, redeemed token or not — regardless of what granted it. The Admin sees the same expired banner/redemption flow as a naturally-lapsed org and needs a new token from Super Admin to use the app again.
- **License & Legal, in Settings** (Admin only): a card showing "This product is licensed to {Admin name} ({email})" with the current status badge, a standard anti-piracy/unauthorized-use notice, and links to Privacy Policy and Terms of Service. The same two links are also in the sidebar footer on every authenticated page, not just the logged-out landing page.
- **Creating and editing organizations** (`/super-admin`): both are proper modals (`CreateOrgModal`, `EditOrgModal`) rather than an inline form on the page. Create sets up the organization and its first Admin account together; Edit only touches the organization's name — the admin's email/login is deliberately not editable there, since changing it wouldn't actually update their real login credentials and would be misleading to expose as if it did.

## Activity log

A platform-wide, append-only audit trail — **Activity log** in the Super Admin sidebar, its own page at `/super-admin/logs` with a paginated table (20 rows per page, Next/Previous — keyset-paginated over the Neon `audit_logs` table by `created_at`, so it scales past what a single query could hold).

- **What's logged**: organization created, suspended, or activated; an org's admin removed; a license revoked by Super Admin; a token generated, redeemed, or revoked; a moderator created or removed. Each entry records the organization, a human-readable message, who did it (email), and when.
- **Who can read it**: Super Admin only (checked in `/api/data/audit-log`'s route handler) — it's a platform-level trail, not something an org's own Admin can browse.
- **Logging is best-effort**: `logEvent()` in `src/lib/auditLog.ts` swallows its own errors — a logging failure never blocks or fails the actual action it's recording.

## Contact form

The landing page (`/`, logged out → **Contact** section) has a real contact form — name, email, message — alongside the direct email/phone/website cards.

- **Submissions go to Neon** (`contact_messages` table, `src/lib/contactMessages.ts`) — no email service involved, so nothing to configure.
- **Anyone can submit**, signed in or not — that's the point of a contact form for prospective customers who don't have an account yet. The `/api/data/contact-messages` POST handler validates the shape of what's written (required non-empty `name`/`email`/`message`, size caps) since there's no signed-in identity to check against.
- **Only Super Admin can read them**: a new **Messages** page at `/super-admin/messages` (linked in the sidebar, with an unread-count badge) lists every submission, newest first. Clicking one expands the full message and marks it read; a **Reply by email** link opens a `mailto:` to that sender; **Delete** removes it permanently.
- Marking read/unread and deleting are both Super-Admin-only at the rules level too — an unauthenticated visitor can create a message but never read, update, or delete one (including their own, once submitted).

## Offline checkout (behavior changed in the Neon rewrite)

Postgres/Neon has no client-side offline cache or offline write queue the way the Firestore SDK did, so this works differently now:

- **Two code paths for creating an order** (`src/lib/orders.ts`): `createOrder` (online) runs a real Postgres transaction (stock check + order-number assignment + insert, all-or-nothing) via a new `/api/data/orders` route. `createOrderOffline` is for genuinely offline checkout — it skips the network entirely and writes the order straight into the same `localStorage` holding pen used for retries (`src/lib/pendingOrders.ts`), using a locally-generated id and a best-guess order number.
- **The tradeoff is the same as before**: the offline path can't ask the server "is there actually enough stock" before committing — it trusts whatever stock number is already showing in the UI. Two offline sales on different devices racing for the last unit of something is the one scenario this doesn't protect against; for a single-register setup this isn't a practical concern.
- **Syncing back**: `flushPendingOrders` (`src/lib/orders.ts`) pushes anything queued in `pendingOrders.ts` through to Neon once back online, oldest-first, via `/api/data/orders/flush`.
- **No more Firestore-style "quota exceeded" fallback**: the old code had a special case for Firestore's daily write-quota errors that silently queued a sale locally even on a nominally online register. Postgres has no equivalent per-project write quota, so that fallback was removed — an online write now either succeeds or fails with a normal error; only genuinely offline checkout uses the local queue.
- A toast ("Saved offline — this sale will sync once you're back online") tells the cashier when a sale took this path, and `OfflineBanner` (`src/components/OfflineBanner.tsx`, shown app-wide from `AppShell`) tells them up front that sales and printing both still work while offline.
- **What needs a live connection**: signing in (session verification is a server round-trip), and flushing the pending-order queue. Everything that used to be a live Firestore `onSnapshot` listener (POS, Inventory, History, Settings, Analytics, Kitchen) is now a short-interval poll (`fetch` every ~5s, ~3-4s for Kitchen) against `/api/data/*` — see each lib file's `subscribeX` function — so there's no live push update anymore, just a small delay (one poll interval) between a change and it showing up elsewhere.

## Cash drawer

A **Drivers** button next to "Current Order" in the POS cart panel (`src/components/pos/CartItemsPanel.tsx`, `src/lib/cashDrawer.ts`) can kick open a till wired into the receipt printer's drawer port, by talking to the printer directly over **WebUSB** and sending the standard ESC/POS "kick drawer" byte sequence. Two real constraints worth knowing before relying on it:

- **Browser support**: WebUSB only exists in Chromium browsers (Chrome, Edge) on desktop — not Firefox, not Safari, not any mobile browser. The button simply doesn't render if `navigator.usb` isn't available.
- **The OS printer driver usually wins the fight for the USB connection.** Once Windows/macOS has a driver installed for the printer (which it needs anyway, for the receipt printing this app already does via the browser print dialog), that driver often holds the printer's USB interface exclusively — so WebUSB's `claimInterface()` call can fail with a permission/already-claimed error that has no code-side workaround. If that happens, the **reliable, zero-code fallback** is almost always available: most thermal printers can be configured (via a DIP switch or the manufacturer's utility) to auto-open the drawer on every print job, which needs nothing from this app since a receipt is already sent to that printer on every sale.
- The first click opens the browser's device-picker to authorize the printer once; Chrome remembers that authorization afterward, so later clicks (and the automatic kick after a cash sale, if a drawer's already paired) don't re-prompt.
- This pairing is per-browser-profile on one device, not something that syncs across your team's machines — each register needs its own one-time "Drawer" click to authorize its printer.

## QR item tags & scanning

- **Generating a tag**: Inventory → **Tag** on any item opens `/tag/[orgId]/[itemId]`, a printable 80mm label (name, price, QR code) — same print-dialog flow as receipts, with its own **Print tag** button rather than auto-printing.
- **What's in the QR code**: not a URL — a short internal string, `primepos-item:<orgId>:<itemId>` (`src/lib/itemQr.ts`). The scanner checks both the prefix and that the `orgId` matches the org you're signed into, so a tag from a different organization (or an unrelated QR code) is just ignored rather than mis-adding the wrong item.
- **Scanning in POS**: the **Scan** button next to the search bar opens a camera modal (`src/components/pos/ScannerModal.tsx`) using `getUserMedia` for the camera feed and the `jsqr` library to decode frames client-side — nothing is uploaded anywhere. A matched tag adds that item to the cart and shows a brief confirmation; an unrecognized code just keeps scanning.
- **Camera access needs a secure context** — same constraint as any website requesting camera/microphone. It works on `localhost` on the same machine, but a phone on your network hitting `http://<LAN-IP>:3000` will likely have the camera blocked by the browser. To test scanning from a phone before deploying, either deploy the app (real HTTPS), or tunnel your dev server over HTTPS temporarily (e.g. `ngrok http 3000` or a Cloudflare Tunnel) and open that HTTPS URL on the phone.
- Only one QR tag format is supported right now (no barcode/EAN scanning) — printing existing barcoded products' own barcodes isn't handled, only tags generated by this app.

## Settings: Receipt Settings vs. POS Setting

Settings is two separate sidebar pages, not one, so branding/receipt content and checkout behavior never end up mixed on the same screen:

- **Receipt Setting** (`/settings/receipt`) — everything that gets printed: shop name, logo, phone/email/website, and the receipt footer message.
- **POS Setting** (`/settings/pos`) — everything that controls checkout math and account access: currency symbol, tax/service-charge percentages, the device-login limit (see below), and **License & Legal** (admin-only — it's about the account's licensing/legal standing, not receipts).
- Both pages share one underlying doc (`ShopSettings`) and the same dirty-tracking/save logic (`src/hooks/useShopSettingsForm.ts`) — each just renders and saves its own slice of it.
- The old `/settings` route still exists as a redirect to `/settings/receipt`, so any old links/bookmarks keep working.
- Both pages sit behind the same `settings` permission key — a Moderator with Settings access (view/edit) gets both pages; there's no separate permission for Receipt vs. POS.

## Device-login limit & session tracking

Each account (Admin or Moderator) can only be signed into a limited number of browsers/devices at once — **2 by default**, adjustable per organization from **Settings → POS Setting → Login Security**.

- **How a "device" is identified**: a random id (`crypto.randomUUID()`) generated on first login and persisted in that browser's `localStorage` (`src/lib/deviceSessions.ts`). It identifies a browser, not a person — clearing site data or using a different browser/profile counts as a new device.
- **Where sessions are stored**: the `device_sessions` table, keyed by `(uid, device_id)` — each row has a human-readable label (parsed from the user agent, e.g. "Chrome on Windows"), a best-effort IP/city/country (a client-side lookup against `ipapi.co`, skipped silently if offline or unreachable), and first/last-seen timestamps.
- **Enforcement happens at login** (`AuthContext.login`, right after `/api/auth/login` succeeds): if the device signing in is already known, its session is just refreshed; if it's a new device and the account is already at its limit, the just-created session is immediately signed back out and a `DeviceLimitError` is thrown, which `LoginScreen` turns into a toast ("This account is already signed in on 2 devices…") rather than the usual inline error banner.
- **Viewing locations/devices**: Team page → **Devices** button on any member opens `UserSessionsModal`, listing every known device for that account with its location and last-active time, and a **Sign out** action that deletes the session row — freeing a slot without needing that person to sign out themselves. Super Admin gets the same view per organization — the Members list inside an org's **Details** modal (`OrgDetailsPanel`) has its own **Devices** button per member, reusing the identical modal.
- **Who can see/revoke sessions**: the account itself, its org's Admin, or Super Admin (checked in `/api/data/device-sessions`) — never another org.
- This is app-level bookkeeping, not a hard auth-provider restriction — the Neon session layer itself doesn't cap concurrent sign-ins, so the limit only applies to logins that go through this app's own `login()` flow.

## Platform security (Super Admin) — the device-limit cap no org can override

**Security** in the Super Admin sidebar (`/super-admin/security`, `src/lib/platformSecurity.ts`) is the platform-level backstop for the per-org device-login limit above — it exists so an org's own Admin can't quietly raise their org's limit to, say, 999 and let one license get shared across an unbounded number of devices.

- A single `platform_security` row holds two numbers: **max devices per account (any organization)** — the ceiling every org's own `maxDevicesPerUser` is clamped against — and **max devices for Super Admin accounts**, since a Super Admin has no org settings row of its own.
- **Enforced in two places**: the org Settings page's "Max devices per account" field can't be typed or saved past the cap (`Math.min` against the polled cap), and `AuthContext.login()` independently takes `Math.min(orgSetting, platformCap)` as the effective limit at login time — so even an org settings row written directly (bypassing the UI) can't exceed the platform cap.
- **Only Super Admin can write it** (checked in `/api/data/platform-security`); any signed-in user can read it, since every org's Settings page needs the current cap to validate against.
- Defaults (5 devices per account, 2 for Super Admin) apply automatically if the row has never been saved — no migration needed on existing installs.
- Changes apply to the next login, not to sessions already signed in.

## Kitchen tickets (Super-Admin-gated per organization)

A separate **Kitchen** sidebar tab (`/kitchen`) lists today's orders, synced live from the same `orders` collection as POS/History — no separate data path, so a new sale shows up immediately. Each order has a **Print ticket** button opening `/kitchen-ticket/[orgId]/[id]` in a new tab, which auto-prints a minimal 80mm ticket: order #, date/time, item names + quantities only (no prices/totals), and the same Primecodia footer used on receipts.

- **Off by default, per organization** — Super Admin turns it on/off from the Organizations list (`OrgActionsMenu` → **Enable/Disable Kitchen**), which just flips `kitchenEnabled` on that org's document (`src/lib/organizations.ts`). An org's own Admin can't enable this for themselves; only Super Admin writes that field (already covered by the existing `organizations` update rule — no rules change needed).
- **Sidebar visibility** needs both: the org has `kitchenEnabled` AND the signed-in account has View access to the new `kitchen` page permission (Admins always do; a Moderator needs it granted from Team → Permissions, same as any other page).
- If an Admin/Moderator opens `/kitchen` directly while it's disabled for their org, they see an explanatory message instead of the order list — the route itself isn't hidden, just inert.

## Daily email reports (Super-Admin-gated, same pattern as Kitchen)

An org can get a short daily sales-summary email — revenue, order count, top 5 sellers — sent automatically to whatever address it chooses, at a time of day it picks. Configure it from **Settings → POS Setting → Daily Email Report**.

- **Two-layer opt-in, same as Kitchen**: Super Admin must first allow it for the organization (`OrgActionsMenu` → **Enable/Disable Daily Reports**, flips `reportsEnabled` on the org doc). Only once that's on does the org's own Admin see the settings card to turn it on, pick a recipient email, a custom send time (`<input type="time">`, any minute — not limited to whole hours), and an IANA timezone (defaults to `Asia/Karachi`).
- **Sending is a real email, not a test one** — it goes through [Resend](https://resend.com) (`src/lib/reports/sendReportEmail.ts`). Needs `RESEND_API_KEY` in your environment, and ideally `RESEND_FROM_EMAIL` set to an address on a domain you've verified in Resend (falls back to Resend's own `onboarding@resend.dev` sandbox sender otherwise, which works but looks less trustworthy to recipients). **testmail.app is not usable here** — it only provides disposable inboxes for *receiving* test mail in QA, not a sending/delivery API, so it can't actually get a report into anyone's real inbox.
- **Delivery is manual or externally scheduled**, through `GET /api/cron/daily-reports` (`src/app/api/cron/daily-reports/route.ts`): each call checks every `reportsEnabled` org and sends for any whose configured local time (`dailyReportTime`, "HH:mm") has **already passed today** and hasn't been sent yet (tracked via `lastReportSentAt` on the org doc). Time-of-day math goes through `src/lib/reports/timezone.ts`, computed with `Intl` so real IANA/DST rules apply — no date library needed.
  - The route is protected by a `CRON_SECRET` env var — it 401s without an exact `Authorization: Bearer <CRON_SECRET>` match. No Vercel cron is configured by default; an external scheduler can call this URL with that header if automatic delivery is needed.
- Report data is computed server-side with a direct `neonQuery` call (no Next.js request object available in a cron context — see `src/lib/reports/generateDailyReport.ts`) against that org's `orders` rows for "today" in its chosen timezone — it doesn't reuse the client-side `src/lib/analytics.ts` helpers, since those expect an already-fetched `Order[]` from a live listener.

## Site map & legal pages

The public side of the site now has a proper footer and a few standard pages, all sharing `PublicHeader`/`PublicFooter` (`src/components/home/`) for a consistent look:

- **About** (`/about`), **Privacy Policy** (`/privacy`), **Terms of Service** (`/terms`) — real content specific to this product (the 14-day trial, Super-Admin-issued license tokens, per-org data isolation), not generic placeholder text. These are a reasonable starting draft, not a substitute for legal review if you need specific regulatory compliance (GDPR, etc.).
- **Footer site map**: every public page now ends in a multi-column footer (Product / Company / Legal + contact icons) linking all of the above, plus the home page's Pricing/Contact sections.
- **`/sitemap.xml`** (`src/app/sitemap.ts`) lists the public, unauthenticated pages for search engines. It uses `https://www.primepos.tech` by default and can be overridden with `NEXT_PUBLIC_SITE_URL`.
- All three new pages are registered in `AppShell`'s `NO_SHELL_PREFIXES`, so they render standalone (no sidebar, no login required) regardless of auth state, same as `/login`.
- Both views share `src/components/licensing/TokenHistoryList.tsx`, which queries tokens by `orgId` only (sorted client-side instead of via `ORDER BY`) — a holdover from the Firestore-index-avoidance days that's harmless now, just no longer necessary.

## Account deletion — what "delete" actually does

- **Deleting your own account** (via the sidebar's Account modal) is a full, real deletion: there's no separate "auth provider" account anymore — the Neon `users` row **is** the whole account (profile + credentials), so deleting it removes everything in one step.
- **An Admin deleting a Moderator**, or **a Super Admin deleting an org's Admin**, removes that person's `users` row entirely (`Team` → Delete, or `/super-admin` → Delete admin). This instantly revokes all app access — they can no longer sign in, and (unlike the old Firestore-profile-only deletion) their email is immediately free to be reused for a new account, since there's no separate auth-provider registration left behind.
- Both deletion paths require typing your current password (self-delete) or a confirmation prompt (deleting someone else) before anything happens.

## Image uploads

Logo and item photo uploads go through `POST /api/upload` (`src/app/api/upload/route.ts`), a server-side route that:

1. Verifies the caller is a real signed-in user via `getSessionUser` (`src/lib/neonAuth.ts`), which reads and verifies the signed session cookie sent automatically on this same-origin request — no bearer token to attach client-side anymore. Unauthenticated requests get `401`.
2. Validates the file server-side (JPEG/PNG/WebP/GIF only, 5MB max) — never trust client-side checks alone.
3. Uploads it to your Cloudflare R2 bucket using the S3-compatible API (`@aws-sdk/client-s3`), under `uploads/<uid>/<random-id>.<ext>`.
4. Returns the public URL (`R2_PUBLIC_URL/uploads/...`), which gets saved on the item or shop settings document.

The R2 credentials live in `.env.local` **without** a `NEXT_PUBLIC_` prefix, so Next.js never bundles them into client-side JS — only this one server route can see them. Don't rename these to `NEXT_PUBLIC_R2_*`; that would ship your bucket's write/delete credentials to every visitor's browser.

Known gap: the R2 bucket itself has no lifecycle/cleanup — deleting an item or replacing a logo doesn't delete the old object from R2, it just stops being referenced. Harmless for a while, but worth knowing if storage usage matters to you long-term.

## Setup

```bash
npm install
npm run dev
```

`DATABASE_URL` (Neon) and `AUTH_SECRET` live in `.env.local` — see **Setting up a fresh Neon database** above.

### Login

Go to `/` — logged out, that's the public landing page, with "Sign in" links to `/login` for each role. After signing in:
- **Super Admin** lands on `/super-admin` to manage organizations.
- **Admin / Moderator** lands on the Home dashboard (also `/` — it shows different content once you're authenticated), with quick links to whichever pages they have View access to.

Visiting any other page (e.g. `/pos`, `/inventory`) while logged out redirects you to `/login` automatically.

### QR verification in development

The QR code encodes `window.location.origin + /verify/<orgId>/<id>`, so it only resolves when scanned from a phone if that origin is reachable — not the case for `localhost`. Deploy the app, or open it via your machine's LAN IP, for QR codes to work from another device.

This is a *different* constraint from the item-tag scanner's camera access (see **QR item tags & scanning** above) — receipt QR codes just need to be network-reachable to open; the in-app camera scanner needs HTTPS (or `localhost`) to even turn the camera on in the first place.

### Printer setup (Black Copper BC-88AC)

Printing goes through the browser's native print dialog rather than raw ESC/POS:

1. Install the BC-88AC's Windows driver so it shows up as a normal printer.
2. In the print dialog, select it, set paper size to **80mm receipt** (or "Custom"), margins **None**.
3. The receipt page sets `@page { size: 80mm auto; margin: 0 }` for print, so once paper size matches, every sale prints correctly without manual resizing.
