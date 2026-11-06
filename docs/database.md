# Database (Supabase Postgres)

Project ref `pdgfeniatfaktiwdabrh`. The live database is the source of truth. The migration history is listed below; to export the full schema to files, run `supabase db pull` with the Supabase CLI.

## Tables (`public`)

All tables have row-level security turned on. "Who" summarizes the policies.

### Applications

| Table | Purpose | Who |
|---|---|---|
| `productions` | One row per application: details, status, footprint figures, review fields (`assigned_reviewer`, `review_due`), certification (`cert_level`, `cert_score`, `certified_at`, `cert_token`), `form_version`, geocoded location (`geo_*`) | Submitters: own drafts. Delegates: read. Reviewers: non-drafts (plus returned drafts they're assigned to). Staff: all |
| `responses` | One row per application × question: `prep_answer`, `wrap_answer`, `explanation`, `answered_by`, reviewer `prep_points`/`wrap_points`, `reviewer_notes`, flags (`flag_note`, `flagged_at`, `flag_addressed_at`) | Read: whoever can see the application. Answers: `can_edit_question`. Scoring and flags: reviewers and staff (enforced by trigger) |
| `department_answerers` | "Answered by" contact per application × department | As answers, by department |
| `production_delegates` | People invited to answer departments on a draft (email, user once linked) | Owner (draft) and staff manage; delegates read their own |
| `application_comments` | Threaded comments (`parent_id`), `internal` notes, resolve state | Reviewers and staff: all. Submitters: shared comments on their applications |
| `application_events` | Activity log. Written only by triggers | Readers of the application; submitters don't see internal kinds |
| `application_deletions` | Record of deleted applications | Staff read |
| `notifications` | Bell items | Each person reads, marks and deletes their own; created only by triggers |
| `certificate_sends` | Every certificate email attempt | Readers of the application |

### Program setup

| Table | Purpose | Who |
|---|---|---|
| `questions` | PEACH actions: `code`, `question_text`, `category`, `department_id`, `answer_options` `[{label, points, na}]`, `is_innovation`, `documentation`, `parent_label`, `sort_order`, `active`, `form_version` | Signed-in users read; staff write |
| `departments` | PEACH departments, ordered | Signed-in users read; staff write |
| `emission_factors` | kg CO₂ per unit for footprint columns | Signed-in users read; staff write |
| `app_settings` | Key/value settings (below) | Everyone reads, including anonymous visitors (for the theme); staff write |

### People and directories

| Table | Purpose | Who |
|---|---|---|
| `profiles` | One per auth user: `role` (`submitter`/`reviewer`/`staff`), name, title, `deactivated_at` | Own row (read and edit name and title); reviewers and staff read all; only staff change role or status |
| `contacts` | Crew and department heads who answer PEACH sections | `private.can_view_contact`; creators and staff edit |
| `companies` | Studios, production companies, vendors | Signed-in users read and add; staff edit |

### Dashboard and other

| Table | Purpose | Who |
|---|---|---|
| `dashboard_prefs` | Per-person `layout`, `charts`, `report` (jsonb) | Own row |
| `shared_charts` | Charts staff share with everyone | Reviewers and staff read; staff write |
| `geocode_cache` | Cached place lookups | No client access (edge function only) |

## `app_settings` keys

| Key | Shape | Used by |
|---|---|---|
| `theme` | `{ org, name, tagline, logo, fonts, colors: { primary, highlight, accent, paper, ink } }` | Every page, sign-in, certificate email |
| `certification` | `{ basis: "wrap", levels: [{ key, name, min }], placeholder }` | `private.level_for`, seals, dashboard |
| `review` | `{ target_days }` | `private.set_review_due` |
| `form` | `{ current }` | New applications' `form_version` |
| `certificate` | Certificate design: `lead`, `body`, signer, `show` flags, orientation, border, number format | Certificate page and email |
| `dashboard_default` | `{ layout }` | Dashboard for people without their own layout |
| `report_default` | Report layout | Report for people without their own |

## Security helpers (`private` schema)

`private` isn't exposed through the API. The helpers are `security definer` with `search_path = ''`, and are called from RLS policies.

| Function | Returns |
|---|---|
| `current_app_role()` | The caller's role, or null if they're deactivated or have no profile |
| `can_view_production(pid)` | Owner, delegate, reviewer (non-draft or assigned) or staff |
| `can_edit_answers(pid)` | Owner of a draft, or staff |
| `is_delegate(pid)`, `is_assigned_reviewer(pid)` | As named |
| `can_edit_question(pid, qid)`, `can_edit_department(pid, did)` | Answer editing, including delegates for their departments |
| `can_view_contact(cid)` | Contact visibility |
| `final_score(pid)` | Final wrap score (verified points where entered) |
| `level_for(score)` | Certification level name for a score |

## Triggers

| Table | Trigger | Does |
|---|---|---|
| `auth.users` | `on_auth_user_created` → `handle_new_user` | Creates the profile and links delegate invites |
| `productions` | `guard_production_submitter` | Keeps protected columns unchanged for submitters |
| | `touch_production` | `updated_at`, `submission_date` on submit |
| | `set_review_due` | Due date on submit and resubmit |
| | `stamp_certification` | Certification on approval; cleared when leaving Approved |
| | `notify_status_change` | Bell notifications |
| | `log_production_events` | Activity log |
| | `record_deletion` | `application_deletions` |
| `responses` | `validate_response_answers` | Answers must be one of the question's option labels |
| | `validate_answered_by` | Answerer must be a visible contact |
| | `guard_response_scoring` | Scoring and flags only by reviewers and staff; marks flags addressed |
| | `log_response_events` | Activity log |
| `application_comments` | `prepare_comment` | Author, thread visibility, reopen on reply |
| | `notify_comment` | Notifications |
| | `guard_comment_update` | No editing comment text |
| `production_delegates` | `link_delegate`, `notify_delegate` | Link to an existing account; notify |
| `companies` | `propagate_company_rename` | Rename studio and company text on applications and contacts |
| `profiles` | `guard_profile_role` | Only staff change role or deactivate |
| `storage.objects` | `log_evidence_events` | Evidence upload and removal in the activity log |
| several | `touch_updated_at` | `updated_at` |

## RPCs (callable from the client)

Each public function is a thin `security invoker` wrapper around a `private` definer function. This keeps the privileged code out of the API surface while the definer function checks the caller.

| RPC | Who | Does |
|---|---|---|
| `recertify_all()` | staff | Re-stamps certification on approved applications after the cutoffs change |
| `start_form_version(new_version)` | staff | Copies the current questions to a new version and makes it current |
| `certificate_by_token(token)` | anyone, including anonymous visitors | Certificate fields for an approved, certified application |

## Storage

| Bucket | Public | Limit | Path and rules |
|---|---|---|---|
| `evidence` | no | 10 MB; images, PDF, CSV, Word, Excel | `<production_id>/<question_id>/<file>`. Read by whoever can see the application; written by whoever can edit that question |
| `branding` | yes | 1 MB; PNG, JPG, WebP, SVG | Logo and signature images; staff write |

## Migration history

Applied in this order:

1. `portal_roles_and_profiles`
2. `application_workflow`
3. `portal_rls_policies`
4. `secure_scores_view_and_dedupe_indexes`
5. `evidence_storage`
6. `production_footprint`
7. `status_notifications`
8. `peach_v2_schema`
9. `footprint_water_waste`
10. `people_and_answerers`
11. `branding_settings`
12. `user_delete_keeps_records`
13. `service_role_private_schema`
14. `companies_directory`
15. `companies_insert_policy_tidy`
16. `dashboard_customization`
17. `activity_log_and_comments`
18. `activity_hide_internal_notes`
19. `activity_evidence_events`
20. `comment_threads_and_application_delete`
21. `certification_flags_due_versions`
22. `certification_on_approval`
23. `start_form_version`
24. `start_form_version_identity`
25. `reviewers_follow_returned_drafts`
26. `review_due_on_resubmit`
27. `staff_rpcs_invoker_wrappers`
28. `questions_unique_per_version`
29. `questions_code_unique_per_version`
30. `certificate_design_and_sending`
31. `certificate_by_token_wrapper`
32. `report_customization`
33. `geocoding`

The original `productions`, `questions`, `responses` and `departments` tables were loaded before migrations were tracked. The PEACH v2.0 questions were imported as data, not as a migration.

## Changing the schema

- Make every change as a named migration (Supabase CLI `supabase migration new`, or the dashboard's SQL editor saved as a migration), never as ad-hoc SQL on the live database.
- New tables need RLS turned on and policies written in the same migration.
- Put new privileged logic in `private`. If clients must call it, expose it through a `security invoker` wrapper in `public`.
- Check access by simulating each role inside a transaction you roll back:
  ```sql
  begin;
  set local role authenticated;
  set local request.jwt.claims = '{"sub":"<user uuid>","role":"authenticated"}';
  -- queries as that user …
  rollback;
  ```
- Run Supabase's security advisor afterwards. The only expected warning is "leaked password protection", which is a dashboard setting.
