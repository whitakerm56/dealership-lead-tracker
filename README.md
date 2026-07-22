# Commercial Lead Tracker

Replaces the shared spreadsheet with:
- a public form (QR code destination) that captures a lead and emails the Sales Manager
- an internal "board" (`dashboard.html`) styled after a physical repair-order rack, where Sales, the Service Advisor, Parts, and the follow-up salesperson each move a lead to the next stage — each move fires the next person's email automatically
- a live ROI panel showing leads captured, RO revenue, program cost, and return
- a "Manage Users" page (`users.html`) where you add/edit/remove who's in each role — that list drives who actually gets the notification emails

**Pipeline:** New Lead → Ready for Advisor → Appointment Scheduled → Parts Ordered → **Service Complete (RO value entered)** → **Follow-Up Assigned** → **Follow-Up Done**

**ROI math:** `ROI% = (Total RO revenue − Program cost) ÷ Program cost × 100`, where Program cost = (number of visits that reached Service Complete) × (Cost / Redemption, editable at the top of the dashboard). That cost figure is stored in Supabase, not the browser, so every screen shows the same ROI.

**Roles:** Sales Manager, Service Advisor, Parts, and Salesperson (the follow-up pool). Add people to each on `users.html`. If a role has nobody added yet, notifications for that role fall back to the single legacy email address set in the Worker's `SALES_MANAGER_EMAIL` / `SERVICE_ADVISOR_EMAIL` / `PARTS_EMAIL` variables — so nothing breaks if you haven't populated the list yet. If a role has multiple people added, everyone in that role gets the notification (whoever's free can grab it). The Salesperson role instead populates a dropdown on the "Assign Follow-Up" step, since that's a single named handoff rather than a broadcast.

**VIN capture:** the public form requires a 17-character VIN and decodes it live against NHTSA's free vPIC API (no key needed) as the person types, showing the year/make/model/trim right there before they submit — or a clear warning if the VIN doesn't check out. The decoded description gets stored on the lead and shows up on the dashboard immediately, and pre-fills the vehicle field when the Service Advisor later logs the appointment.

## 1. Supabase

1. Create a new Supabase project (separate from any other project you run).
2. Open the SQL editor and run `supabase/schema.sql`.
3. Under Project Settings → API, copy:
   - **Project URL** → `SUPABASE_URL`
   - **service_role key** (not the anon key) → `SUPABASE_SERVICE_KEY`

The service_role key bypasses RLS and is only ever used inside the Worker — never expose it in the frontend.

## 2. Resend (email)

1. Sign up at resend.com and verify a sending domain (e.g. `yourdealership.com`). Free tier covers this easily (3,000 emails/month, 100/day) — plenty for lead-volume traffic.
2. Create an API key → `RESEND_API_KEY`.
3. Decide the `FROM_EMAIL` (must be on the verified domain, e.g. `leads@yourdealership.com`).

## 3. Cloudflare Worker (API)

You can deploy this two ways — pick whichever's easier. Both end up in the same place.

### Option A: Dashboard only (no installs, no terminal)

1. Log into dash.cloudflare.com → **Workers & Pages** → **Create application** → **Create Worker**.
2. Pick the **Hello World** template, name it (e.g. `dealership-lead-tracker-api`), and **Deploy**.
3. On the Worker's page, click **Edit code**. Delete the placeholder and paste in the entire contents of `worker/src/index.js`. Deploy again.
4. Go to **Settings → Variables and Secrets** and add:
   - Plain variables: `ALLOWED_ORIGIN`, `ENFORCE_IP_ALLOWLIST`, `ALLOWED_IPS`, `FROM_EMAIL`, `SALES_MANAGER_EMAIL`, `SERVICE_ADVISOR_EMAIL`, `PARTS_EMAIL` (see values below)
   - Encrypted/secret variables: `SUPABASE_URL`, `SUPABASE_SERVICE_KEY`, `RESEND_API_KEY`
5. Save (this redeploys automatically). Your Worker URL is shown at the top of the page, e.g. `https://dealership-lead-tracker-api.<your-subdomain>.workers.dev`.

### Option B: Wrangler CLI

```
cd worker
npm install -g wrangler   # if you don't have it
wrangler login
wrangler secret put SUPABASE_URL
wrangler secret put SUPABASE_SERVICE_KEY
wrangler secret put RESEND_API_KEY
```

Edit `wrangler.toml` and fill in the `[vars]` block, then deploy:

```
wrangler deploy
```

### Values to set either way

- `ALLOWED_ORIGIN` — your GitHub Pages URL (e.g. `https://yourusername.github.io`)
- `ENFORCE_IP_ALLOWLIST` — `false` for now (demo mode); `true` at go-live
- `ALLOWED_IPS` — blank for now; the dealership's static IP at go-live
- `FROM_EMAIL` — `onboarding@resend.dev` for now (demo mode); a verified-domain address at go-live
- `SALES_MANAGER_EMAIL`, `SERVICE_ADVISOR_EMAIL`, `PARTS_EMAIL` — your own inbox for now (demo mode, per Resend's sandbox limits); real staff addresses at go-live

## 4. Restricting the dashboard to the dealership network

This works the same way as your CCC Co-Pilot setup: the Worker checks the caller's IP against an allowlist before serving `GET`/`PATCH /api/leads` — but only when it's turned on.

**Demo mode (default):** `ENFORCE_IP_ALLOWLIST = "false"` in `wrangler.toml` means the dashboard is reachable from anywhere — no IP check at all. This is deliberate while you're building and presenting the demo, so you're not locked out of your own tool.

**Go-live:** once the dealership signs on, switch it over:
1. From a computer on the dealership Wi-Fi/network, visit `https://whatismyipaddress.com` and note the IPv4 address. Confirm with your ISP that it's a *static* business IP — if it's dynamic (common on consumer/residential-grade plans), it can change without notice and silently lock everyone out until you update `ALLOWED_IPS` and redeploy. Most business-class internet plans include a static IP; it's worth confirming rather than assuming.
2. Set `ALLOWED_IPS` in `wrangler.toml` to that address (comma-separated if there's more than one location/circuit).
3. Set `ENFORCE_IP_ALLOWLIST = "true"`.
4. `wrangler deploy` again.

The public lead form (`POST /api/leads`) is always exempt from this check, since it needs to work from anyone's phone regardless of mode.

## 5. Frontend (GitHub Pages)

1. In `public/index.html`, replace `https://api.yourdealership-leads.workers.dev/api/leads` with your Worker URL + `/api/leads`.
2. In `public/dashboard.html`, replace `https://api.yourdealership-leads.workers.dev/api` with your Worker URL + `/api` (no trailing `/leads` — the dashboard calls `/api/leads`, `/api/settings`, and `/api/users`).
3. In `public/users.html`, set the same `API_URL` as `dashboard.html`.
4. Push all three files in `public/` to a GitHub repo and enable Pages (Settings → Pages → deploy from branch, root or `/public`).
4. You'll end up with two URLs:
   - `.../index.html` — put this behind the QR code on the flyer
   - `.../dashboard.html` — bookmark this on the dealership computers/tablets each department will use

## 6. Generate the QR code

Any free QR generator (e.g. `qr-code-generator.com`) pointed at your `index.html` URL works — no need to build this into the app.

## Notes / things to double check before go-live

- **Follow-up assignment is by email, not a fixed roster:** since your commercial salesmen aren't hardcoded into the app, the Sales Manager types in the salesperson's name and email each time a visit reaches "Service Complete." That email is who gets the follow-up notification. If the sales team is a small, fixed group, it'd be easy to turn this into a dropdown later — just say the word.
- **ROI only counts visits that reached Service Complete.** A lead sitting in "New" or "Appointment Set" hasn't cost you anything yet, so it's excluded from both the revenue and the cost side of the calculation until an RO value is entered.
- **Accountability without login:** since there's no auth, each stage-advance asks the person to type their name (`assigned_by`, `scheduled_by`, `parts_by`, `closed_by`, `followup_completed_by`) so there's a record of who did what.
- **Email delivery isn't guaranteed:** if Resend fails to send, the lead's status still updates — the board itself is always the source of truth, the emails are just the "someone needs to check the board" nudge. Worth checking the Resend dashboard occasionally for bounces.
- **Polling, not real-time:** the dashboard re-fetches every 30 seconds and on any action. If you want instant updates across screens, Supabase Realtime could be layered in later — not necessary for an MVP with this volume.
- **Duplicate leads:** the schema doesn't currently dedupe by company/email, so the same business scanning twice creates two cards. Easy to add a check in `createLead` later if it becomes an issue.
- **VIN validation is strict on bad VINs, lenient on a down VIN service.** If NHTSA reports the VIN itself is invalid, submission is blocked until it's corrected. If NHTSA's API is simply unreachable, submission is still allowed (with no decoded vehicle info) rather than blocking a customer because a third-party government API is briefly down.
