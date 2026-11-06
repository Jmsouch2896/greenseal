# Edge functions

Four Deno functions run on Supabase. All have **verify JWT** on, so the caller must send a signed-in user's token; `sb.functions.invoke` does this automatically. Each one also checks the caller's role itself.

| Function | Version | Purpose |
|---|---|---|
| `admin-users` | v3 | Account management (staff) and delegate invitations |
| `notify` | v2 | Status-change emails |
| `send-certificate` | v1 | Email a certificate |
| `geocode` | v4 | Place lookup and saving a production's location |

The source lives in Supabase. Download a copy with `supabase functions download <name>`, or view it in the dashboard under Edge Functions.

## Secrets

Set these under Supabase → Edge Functions → Secrets:

| Secret | Needed by | Notes |
|---|---|---|
| `RESEND_API_KEY` | `notify`, `send-certificate` | Without it, both return a "not set up" result instead of failing |
| `EMAIL_FROM` | both | Defaults to `onboarding@resend.dev`, which only delivers to the Resend account owner until a domain is verified |
| `PORTAL_URL` | both | The live site URL, for links in emails. `send-certificate` also accepts it from the client |

`SUPABASE_URL`, `SUPABASE_ANON_KEY` and `SUPABASE_SERVICE_ROLE_KEY` are provided automatically.

## `admin-users`

Body: `{ action, ... }`. The caller must have an active profile.

| `action` | Who | Fields | Result |
|---|---|---|---|
| `create` | staff | `email`, `full_name`, `title?`, `role`, `method` (`password` or `invite`), `redirect_to?` | `{ ok, user_id, password? }`. The temporary password is returned once and never stored |
| `reset_password` | staff | `user_id` | `{ ok, password }` |
| `set_active` | staff | `user_id`, `active` | Bans or unbans the auth user and sets `profiles.deactivated_at` |
| `delete` | staff | `user_id`, `transfer_to?` | Optionally moves their applications and contacts first, then deletes the auth user (records are kept) |
| `invite_delegate` | owner of a draft, or staff | `production_id`, `email`, `redirect_to?` | Emails a sign-up invitation if the person has no account; rate-limited by Supabase Auth email |

Staff can't deactivate or delete themselves.

## `notify`

Body: `{ production_id }`. It reads the application **with the caller's own token**, so it can't be used to email about applications the caller can't see, and it uses the real current status.

- `submitted`: emails every reviewer and staff member except the caller.
- Other statuses: emails the submitter, unless the submitter made the change.
- Skips `@example.*` addresses (demo accounts).

The bell notifications don't depend on this function; a database trigger creates them.

## `send-certificate`

Body: `{ production_id, recipients[] (≤10), message?, portal_url? }`. Reviewers, staff and the application's owner can send, for approved and certified applications only.

- Builds an HTML email from the theme and certificate settings, with a link to the public `#verify-<token>` page.
- Sets the reply-to address to the sender.
- Logs every attempt in `certificate_sends`; successful sends also go into the activity log.
- Without `RESEND_API_KEY` it returns `{ ok: false, code: "no_email", link }`, and the portal offers to copy the link instead.

## `geocode`

| Body | Does |
|---|---|
| `{ query }` | Previews up to five matching places. Nothing is saved |
| `{ production_id, force? }` | Places the production's current location text. Skips the lookup if the text hasn't changed, unless `force` is set |
| `{ production_id, location, pick? \| place? }` | Changes the location text **as the caller**, so RLS decides who may. Then saves either result number `pick` from the server's search, or a `place` the browser already looked up (validated and cleaned) |

- **Lookups:** Photon (photon.komoot.io) first, then Nominatim. Results are cached in `geocode_cache`, keyed by the normalized query.
- **Nominatim is blocked from Supabase's servers**, which is why Photon goes first and why the browser can do the search itself (`searchPlaces` in the front end).
- **If the lookup fails** after the location text was changed, the text stays saved, the old place is cleared, and the response is `{ ok: true, found: false, note }`.
- **`geo_source`** records how the place was found: `osm` (automatic), `chosen` (picked by a person) or `seed` (preloaded).

## Changing a function

1. Edit the code. Download it first if you don't have it locally.
2. Deploy: `supabase functions deploy <name>`, keeping verify JWT on.
3. Check the logs (Supabase → Edge Functions → Logs) while trying it in the portal.

Keep every function to these rules:

- Check the caller, and their role where it matters, before doing anything with the service role.
- Read the caller's data with a client built from their own token where you can, so RLS applies.
- Return `{ error: "plain-language message" }` with a 4xx/5xx status. The front end shows that message through `fnError`.
