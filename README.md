# DailyBudget

**Live site:** https://shahriyajaman007-collab.github.io/dailybudget/

Created by **Md Sahria Jaman Shazib**.

A public, multi-user personal budgeting web app. Anyone can visit the site, create an account and keep a private dashboard of income, expenses, daily and monthly budgets, bills and savings goals.

- **Frontend:** one file, `index.html` (HTML, CSS and vanilla JavaScript). No npm, Node.js, build step or custom server.
- **Hosting:** GitHub Pages.
- **Backend:** Supabase Auth (sign-up, login, password reset) and Supabase PostgreSQL with Row Level Security.
- **Libraries (CDN):** `@supabase/supabase-js@2.45.4` and `chart.js@4.4.1` from jsDelivr, pinned to exact versions.

## Files

| File | Purpose |
|---|---|
| `index.html` | The entire app: landing page, auth, dashboard and all screens. |
| `supabase_setup.sql` | Tables, constraints, indexes, triggers, RLS policies and database functions. Run once in Supabase. |
| `README.md` | This guide. |
| `.nojekyll` | Empty file that makes GitHub Pages publish the site exactly as uploaded. |
| `.gitignore` | Keeps editor files, OS clutter and any local secrets out of the repository. |

---

## 1. Set up Supabase

1. Create a free project at <https://supabase.com>. Pick a strong database password and store it in a password manager. **It never goes in this repository.**
2. Open **SQL Editor → New query**, paste the whole of `supabase_setup.sql`, and click **Run**. The script is safe to run again later.
3. Confirm Row Level Security is on: run the check query at the bottom of the SQL file. Every table should show `true`.

### Configure Auth

In **Authentication**:

- **Sign In / Providers → Email:** keep Email enabled. Turn on **Confirm email** for a public site so people must prove they own the address.
- **URL Configuration:**
  - **Site URL:** your GitHub Pages address: `https://shahriyajaman007-collab.github.io/dailybudget/`
  - **Redirect URLs:** add the same address. Add `http://localhost:8000/` too if you test locally.
  Confirmation and password-reset links only redirect to URLs on this list.
- **Password settings:** set the minimum length to at least 8 (the app also checks this). Enable leaked-password protection if your plan offers it.
- **Rate limits and CAPTCHA:** keep the default rate limits. For a busy public site, enable CAPTCHA protection (hCaptcha or Turnstile) to slow down automated sign-ups. Note that turning CAPTCHA on also requires adding the CAPTCHA widget to the sign-up form.
- **SMTP:** Supabase's built-in email sender is limited to a handful of messages per hour and is meant for testing. Before launch, add your own SMTP provider under **Authentication → Emails → SMTP Settings**.

The app uses the PKCE flow, so links in emails carry a one-time code rather than tokens. Open confirmation and reset links in the **same browser** you used to request them. The app removes the code from the address bar as soon as it is used.

### Get your public keys

Go to **Project Settings → API Keys** (or **Data API**) and copy:

- the **Project URL** (`https://xxxx.supabase.co`)
- the **publishable key** (`sb_publishable_...`) or, on older projects, the legacy **anon public** key.

> Never use the `service_role` / secret key, a JWT secret or the database password in `index.html`. Those keys bypass Row Level Security. The publishable/anon key is designed to be public; security comes from the RLS policies.

## 2. Configure the app

Open `index.html`, find the `CONFIG` block near the top of the script, and fill in:

```js
const CONFIG = {
  SUPABASE_URL: 'https://xxxx.supabase.co',
  SUPABASE_PUBLIC_KEY: 'sb_publishable_xxxxxxxx',
  IDLE_TIMEOUT_MINUTES: 30,
};
```

Until these are set, the site shows a banner saying it is not connected.

If you use a **custom Supabase domain**, also add it to `connect-src` in the `Content-Security-Policy` meta tag at the top of `index.html`.

### Test locally (optional)

Opening the file directly from disk will not work well with auth redirects. From the folder, run any static server, for example:

```bash
python3 -m http.server 8000
```

then visit `http://localhost:8000/` (and make sure that URL is in Supabase's Redirect URLs).

## 3. Deploy on GitHub Pages

1. Create a repository, for example `dailybudget`.
2. Upload `index.html`, `README.md`, `supabase_setup.sql`, `.nojekyll` and `.gitignore` to the repository root (not inside a folder). `index.html` is the only file the website needs; `.nojekyll` tells GitHub to publish the files as they are without running its Jekyll site builder.
3. Go to **Settings → Pages**. Under **Build and deployment**, choose **Deploy from a branch**, select `main` and `/ (root)`, and save.
4. After a minute the site is live at `https://shahriyajaman007-collab.github.io/dailybudget/`.
5. Make sure exactly that URL is the Site URL and a Redirect URL in Supabase.
6. Under **Settings → Pages**, tick **Enforce HTTPS**.

GitHub Pages runs no server code. Everything sensitive happens in Supabase.

---

## How the money is calculated

These definitions are shown in the app and kept consistent everywhere, so the same money is never counted twice.

- **Balance on a date** = opening balance + all income − all expenses dated on or before that day. The opening balance is set in Settings.
- **Set aside for goals** = the saved amounts of goals marked *inside my tracked balance*.
- **Available after goals** = balance − set aside. Goal money marked *in a separate account* is not part of the balance, so it is neither added nor subtracted.
- **Saved this month** = income − expenses for the month (cash flow). It is not the same as goal deposits.
- **Daily budget:** only expenses dated that day count. Income never reduces spending. Expenses created by paying a bill are left out by default (toggle on the Budgets page) so rent does not wreck one day.
- **Rollover (optional):** unused budget from earlier days in the same month carries into today.
  `carry = max(0, Σ (daily limit − counted spending))` over each earlier day this month, starting from the later of the 1st and your sign-up date. Overspending reduces the carry, the carry never goes negative, and it restarts each month.
- **Warnings:** 50% (half used), 80%, 100% (limit reached), and above 100% (over the limit). The same levels apply to monthly and category budgets.
- **Currencies:** each record keeps its own currency. Totals, budgets and charts include only records in your display currency; other-currency records are listed and flagged, never converted or mixed.

### Bills and expenses

A bill is a reminder that money is due. It is not spending. When you choose **Mark paid**, the database function `pay_bill()`:

1. optionally records **one** expense, linked to the bill by `bill_id` and to that cycle by `bill_due_date`;
2. moves a recurring bill to its next due date (keeping the same day of the month where possible), or marks a one-time bill paid.

A unique index on `(bill_id, bill_due_date)` makes recording the same cycle twice impossible. Untick *Record this payment as an expense* if you already entered it yourself. Deleting a bill keeps its past expenses and just clears the link.

### Savings goals

Goals are amounts you enter and set aside. They are not bank accounts and are not verified by any financial institution. Adding to or withdrawing from a goal does not create a transaction, because moving money between your own pots is not spending.

---

## Security model

- **Authentication** is handled by Supabase Auth. The app never sees or stores password hashes.
- **Authorization is enforced in the database.** Every table with user data has RLS enabled with separate SELECT, INSERT, UPDATE and DELETE policies for the `authenticated` role that compare the row owner to `auth.uid()`. Anonymous visitors have no policies and no table privileges, so they cannot read any financial data.
- **Ownership cannot be changed.** UPDATE policies check ownership both before (`USING`) and after (`WITH CHECK`) the change, and a trigger rejects any change to `user_id`.
- **Linked records stay within one account.** A transaction's bill link is a composite foreign key on `(bill_id, user_id)`, so it can only point to a bill owned by the same user.
- **Database functions:** `pay_bill()` runs as the caller (`SECURITY INVOKER`), so RLS still applies. `delete_my_account()` must run as `SECURITY DEFINER` to remove the login, and it only ever deletes `auth.uid()`. Both are callable only by signed-in users and use a fixed `search_path`.
- **Validation** happens twice: the app rejects invalid, non-finite, zero or negative amounts and impossible dates, and the database has `CHECK` constraints on amounts, dates, lengths and allowed values.
- **Output safety:** all user text is HTML-escaped before display. CSV exports quote every cell and prefix text that starts with `=`, `+`, `-`, `@`, tab or carriage return with an apostrophe to block spreadsheet formula injection.
- **Sessions** are managed by the Supabase client (short-lived access token plus refresh token in the browser's storage, under Supabase's own key). Expired or revoked sessions return the user to the landing page. The app also signs out after 30 minutes of inactivity (change `IDLE_TIMEOUT_MINUTES`) and offers *Sign out everywhere*.
- **No secrets or data in URLs.** Routes such as `#/bills` contain no data, and auth codes are removed from the address bar immediately.
- **Logging:** the app never logs transaction contents or tokens; it logs at most a short error code.
- **No third parties for data:** transaction details are not sent to AI services, analytics or advertisers. The only outside requests are to your Supabase project, jsDelivr (scripts) and Google Fonts (typefaces). To avoid Google Fonts, delete the three font `<link>` tags; the app falls back to system fonts.
- **Content Security Policy:** a CSP meta tag limits scripts to the page and jsDelivr, and network connections to Supabase.
- **What users should never enter:** bank passwords, full card numbers, account numbers or other financial credentials. Notes fields remind them.

### GitHub Pages is public

Anyone can read `index.html`, including the Supabase URL and publishable key. That is expected and safe **only** because RLS protects the data. Never commit service-role keys, database passwords or `.env` files. If a secret is ever committed, rotate it in Supabase immediately; deleting the commit is not enough.

### Optional hardening

- Add [Subresource Integrity](https://www.srihash.org/) hashes (`integrity="sha384-..." crossorigin="anonymous"`) to the two CDN `<script>` tags so a tampered CDN file is refused.
- GitHub Pages cannot send custom HTTP headers. To add `frame-ancestors`, HSTS preload or stricter headers, put the site behind a CDN such as Cloudflare.
- Enable MFA for your own Supabase dashboard account.

---

## Account deletion and data export

In **Settings**, users can:

- download **all data (JSON)**: profile, preferences, transactions, budgets, bills and goals (amounts in integer cents);
- download **transactions (CSV)**; the Transactions and Reports pages also export the current filter or date range;
- **delete their account**, which removes the login and, through `ON DELETE CASCADE`, every row they own.

## Maintenance

- **Library updates:** to upgrade Supabase JS or Chart.js, change the pinned version in the two `<script>` tags, choose a release at least two weeks old, test, then update SRI hashes if you use them.
- **Schema changes:** add new SQL to a new migration file and run it in the SQL editor. Keep RLS enabled on every new table and add all four policies.
- **Backups:** paid Supabase plans include daily backups. On the free plan, export regularly (Database → Backups or `pg_dump`), and note that free projects pause after a week of inactivity.
- **Monitoring:** check **Authentication → Users** and **Logs** for unusual sign-up spikes. Review **Advisors → Security** in Supabase; it flags tables without RLS.
- **Scale:** the app loads a user's records in pages of 1,000 at sign-in, which comfortably handles many years of personal transactions. For far larger datasets, move summaries into SQL views or RPC functions.

## Troubleshooting

| Problem | Fix |
|---|---|
| Banner says the site is not connected | Fill in `CONFIG` in `index.html`. |
| "Could not reach the server" | Check the Project URL, your connection, and that the project is not paused. |
| Email links go to localhost or the wrong page | Set Site URL and Redirect URLs in Supabase to your Pages URL. |
| "Link has expired or was already used" | Request a new link and open it in the same browser. |
| Confirmation emails never arrive | Check spam, then configure custom SMTP (the built-in sender is rate limited). |
| "You do not have permission" | The SQL file was not run fully, or policies were edited. Re-run `supabase_setup.sql`. |
| Charts say they could not load | jsDelivr is blocked on that network. The rest of the app still works. |

DailyBudget is a planning tool, not financial advice.

© Md Sahria Jaman Shazib
