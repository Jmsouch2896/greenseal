# Development

## Running locally

```bash
python3 -m http.server 8000     # from the repo root
# open http://localhost:8000
```

Opening `index.html` directly from disk also mostly works, but some browsers block fetches from `file://` pages, which affects maps and Excel import.

The page connects to the **live** Supabase project (`SUPABASE_URL` and `SUPABASE_KEY` in the Connection section). Anything you do while signed in changes real data. For safer development:

- **Use a Supabase branch or a second project.** Point the two constants at it temporarily, and don't commit that change.
- **Use test accounts** with `@example.com` addresses. The email functions skip them.

## Making changes

**Front end:** edit `index.html`. The sections are listed in [frontend.md](frontend.md). Some habits that have kept the code stable:

- Escape everything that came from the database or a person with `esc()`.
- Add colors as CSS tokens, never literal values, so the theme and dark mode keep working.
- After a change, check the page at phone width (about 400px) and in dark mode.
- Keep library versions pinned.

**Database:** add a migration (see [database.md](database.md#changing-the-schema)). Every new table needs RLS and policies in the same migration. Check each role's access with the rolled-back transaction pattern.

**Edge functions:** see [edge-functions.md](edge-functions.md#changing-a-function).

## Testing

There's no automated test suite in the repo yet. Features so far were checked two ways:

1. **Browser tests (Playwright) against a mock Supabase client.**
   - A small script stands in for `supabase-js`. It serves in-memory tables (productions, responses, comments, settings…) and records calls to `functions.invoke`.
   - A test copy of `index.html` loads that script instead of the CDN library.
   - Each role is picked with `?as=staff|reviewer|submitter|delegate`.
   - Tests click through pages at desktop and phone widths, check for script errors and horizontal scrolling, and take screenshots.
2. **SQL role simulation.** Policies and triggers were checked with the `set local role authenticated` pattern in [database.md](database.md#changing-the-schema), always inside a transaction that's rolled back.

Adding the mock client and Playwright tests to this repo (for example `dev/mock.js` and `tests/`) would make this repeatable for anyone.

## Deploying

| Part | How |
|---|---|
| Front end | Commit and push to `main`. GitHub Pages republishes in about a minute. Hard-refresh to see changes |
| Database | Apply the migration to the Supabase project (CLI `supabase db push`, or the dashboard) |
| Edge functions | `supabase functions deploy <name>` |

Deploy database changes **before** the front end that depends on them.

## Configuration checklist (production readiness)

- [ ] `RESEND_API_KEY`, `EMAIL_FROM` (verified domain) and `PORTAL_URL` secrets for email
- [ ] SMTP set up in Supabase Auth, so invitations aren't rate-limited
- [ ] Leaked-password protection turned on (Auth settings)
- [ ] EMA's real certification cutoffs (Questions page). The current 60% / 80% are placeholders
- [ ] Custom domain for GitHub Pages, if wanted; then update `PORTAL_URL` and the Auth redirect URLs
