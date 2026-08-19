# Maritime Cruise Agency — full scaffold

Next.js + Supabase app covering **auth, database schema, and all three
portals**: the applicant experience, the staff review console, and the
partner/employer portal, plus a public marketing landing page.
Everything reads and writes real data — nothing here is mock data.

## 1. Create a Supabase project
1. Go to https://supabase.com, create a new project.
2. In the SQL editor, paste and run `supabase/migrations/0001_init.sql`.
   This creates all tables, enums, row-level security policies, and a
   private `documents` storage bucket.
3. Then run `supabase/migrations/0002_exams_and_accommodations.sql`. This
   adds exam PDF uploads, an accommodation listings catalog with photos,
   and a `pending` travel status. If your Postgres version complains
   about the `ALTER TYPE ... ADD VALUE` line running in the same
   transaction as later statements that use it, just run that one line
   first, then run the rest of the file.
4. Then run `supabase/migrations/0003_inquiries.sql`. This adds the
   in-platform messaging/consult system described below.
5. Then run `supabase/migrations/0004_registration_fields.sql`. This adds
   the expanded registration fields (name, nationality, phone, date of
   birth, gender, referral source) to `profiles`.
6. Then run `supabase/migrations/0005_screening_ships_verification.sql`.
   This adds the language accreditation document type, the ships
   catalog, the screening questionnaire, and employee/ship verification.
7. Then run `supabase/migrations/0006_profile_email.sql`. This adds an
   `email` column to `profiles` (needed for notification emails) and
   backfills it for any accounts created before this migration.

## 2. Configure environment variables
Copy `.env.local.example` to `.env.local` and fill in your project's URL
and anon key (Project Settings → API).

## 3. Install and run
```bash
npm install
npm run dev
```

A `package-lock.json` is committed, so Vercel (and `npm install` locally)
will use npm rather than defaulting to pnpm. `npm audit` will flag 3
high-severity issues in `postcss`/`sharp` — these are inside Next.js's
own build tooling, not reachable through normal use of this app (we
don't run untrusted images or CSS through them), and only clear by
jumping to Next 16, a breaking major version. Left as-is for now.

## 4. Create your first staff and partner accounts
New sign-ups via `/login` always start as `applicant`. To promote someone
to `staff` or `partner`, run this in the Supabase SQL editor after they
sign up once:

```sql
update profiles set role = 'staff' where id = '<their-user-id>';

-- for a partner, also set the company name shown in their portal
update profiles set role = 'partner', partner_company = 'Meridian Voyages'
where id = '<their-user-id>';
```

You can find their user id in Authentication → Users, or:
```sql
select id, full_name from profiles where full_name = 'Jane Doe';
```

## 5. Set up email notifications (optional but recommended)
1. Create a free account at [resend.com](https://resend.com).
2. Verify a sending domain (Resend walks you through the DNS records) —
   or use their test domain while you're just trying things out.
3. Create an API key at resend.com/api-keys.
4. In `.env.local`, set `RESEND_API_KEY` to that key, and
   `RESEND_FROM_EMAIL` to an address on your verified domain (e.g.
   `Maritime Cruise Agency <notifications@maritimecruiseagency.com>`).
5. Set `NEXT_PUBLIC_APP_URL` to your real domain once deployed — it's
   used to build the links inside notification emails.

If you skip this, the app works exactly the same — emails are just
logged to the console instead of sent, so nothing breaks in local dev
without a Resend account.

## What's included
- `supabase/migrations/0001_init.sql` — full schema: `profiles`, `jobs`,
  `applications`, `exam_results`, `documents`, `travel_requests`, plus RLS
  policies for applicant / staff / partner and a private storage bucket.
- `lib/supabase/client.ts`, `lib/supabase/server.ts` — Supabase client
  helpers for Client and Server Components.
- `middleware.ts` — refreshes the auth session on every request and
  redirects users away from `/staff` or `/partner` if their role doesn't
  match.
- `app/login/page.tsx` — sign in / sign up, routes users to the right
  area based on role.
- `app/staff/*` — staff review console:
  - `/staff` — counts of what's pending
  - `/staff/applications` — change an application's stage
  - `/staff/documents` — approve/reject uploaded documents, view the file
    via a short-lived signed URL
  - `/staff/exams` — upload the exam paper for each category as a PDF;
    applicants always see the most recent one
  - `/staff/screening` — read-only view of every applicant's screening
    answers
  - `/staff/accommodations` — post accommodation listings with photos and
    toggle availability
  - `/staff/ships` — add/manage the ships under your agency's management;
    this is the list applicants pick from when confirming their ship
    assignment
  - `/staff/travel` — update visa/flight status, and for accommodation
    requests, assign a specific listing. Assigning only sets the request
    to `confirmed` if the applicant's documents are all approved —
    otherwise it stays `pending` with a note explaining why
- `app/app/*` — applicant portal (requires `role = 'applicant'`):
  - `/app` — boarding-pass style progress overview, built from real counts
  - `/app/welcome` — one-time "thank you for registering" page shown
    right after sign-up, with the document checklist
  - `/app/jobs` — browse open jobs by category, view requirements, apply
  - `/app/applications` — the applicant's own applications, with a form
    to enter an employee number + ship once HR issues them (see
    verification note below)
  - `/app/exams` — download the current exam PDF per category, or take
    the scored online quiz (70% to pass); both are saved to the record
  - `/app/documents` — upload CV, language accreditation (TOEFL/IELTS/
    TOEIC/Cambridge/JLPT only), qualification certificates, medical exam
    results, and police clearance — DOC/DOCX/PDF/TXT, 3MB max, validated
    client-side before upload
  - `/app/screening` — the yes/no + short-answer questionnaire; answers
    upsert into one row per applicant so it can be revisited and updated
  - `/app/travel` — request visa/flights, and for accommodation, browse
    posted listings (with photos) and request one specifically or leave
    it open — status shows `pending` until staff confirm, with a note if
    it's waiting on document review specifically
- `app/partner/*` — partner/employer portal (requires `role = 'partner'`):
  - `/partner` — counts of open postings, total postings, total applicants
  - `/partner/jobs` — post new jobs (category, title, requirements) and
    toggle them open/closed
  - `/partner/applicants` — read-only list of applicants to their own
    postings and current stage (exam/document review stays with staff)
  - `/partner/accommodations` — read-only view of the accommodation
    catalog staff maintain
  - `/partner/messages` — send and read replies to their own inquiries
- `app/page.tsx` — the public landing page: hero slideshow (photos in
  `public/hero/`), job category overview, "how it works", the WhatsApp
  scam notice, and a CTA into `/login`.
- `app/login/page.tsx` — sign in, and full applicant registration
  (first/last name, country of residence, nationality, phone with
  country code, date of birth, gender, how they heard about us). All of
  it is stored on `profiles` via the sign-up trigger in migration 4;
  `full_name` is auto-derived so existing pages that read it keep working.
- **Theme**: navy blue, white, and warm wood tones throughout — navy
  sidebars, white content cards, wood-toned buttons/accents. All colors
  are defined per-page as a small `const C = {...}` object at the top of
  each file for easy tweaking.
- `app/contact` — public page, no login required. Anyone (prospective
  applicant, partner, or someone who just wants to verify a job offer)
  can send a message. Always shows the scam notice.
- `app/app/messages`, `app/partner/messages`, `app/staff/inquiries` —
  the in-platform messaging system: applicants/partners send messages
  and see staff replies inline; staff see everything (including
  anonymous public submissions) in one queue and can reply + set status.
- `components/ScamNotice.tsx` — the "we never use WhatsApp" warning,
  shown on `/login`, `/contact`, the applicant dashboard, the partner
  dashboard, and both messaging pages.
- `lib/actions/staff.ts`, `lib/actions/applicant.ts`, `lib/actions/partner.ts`,
  `lib/actions/inquiries.ts`
  — Server Actions for each portal (RLS is the actual security boundary;
  these are ergonomic wrappers around it).
- `lib/data/exams.ts` — exam question content, kept in code for now since
  there's no admin UI to author questions yet.

## Design notes
- `travel_requests.provider` (`manual` | `api`) means the same table and
  the same staff UI keep working once you plug in a real visa or booking
  API — you'd add a background job that syncs `provider_ref` and updates
  `status` automatically, no schema change needed.
- Documents and exam PDFs are stored in **private** Storage buckets, only
  reachable via signed-URL routes (`/api/documents/[id]/file`,
  `/api/exams/[category]/file`) — nothing is public by default.
- Accommodation photos are the one exception: that bucket is public,
  since they're pictures of bookable places rather than personal
  documents, which keeps the photo grid simple (no signed URLs needed).
  Only staff can upload or delete them.
- Accommodation status logic: a request starts `pending`. Staff assigning
  a listing only flips it to `confirmed` if the applicant's documents are
  *all* approved — otherwise it stays `pending`, and the applicant sees a
  note explaining it's waiting on document review specifically.
- **Employee/ship verification is intentionally narrow.** An applicant
  can move their own application to `approved` in exactly one way: by
  calling the `verify_employment` Postgres function (via
  `lib/actions/applicant.ts`) with an employee number and a ship they
  select from the ships you manage. That function is `SECURITY DEFINER`
  and only ever touches the caller's own row, setting exactly
  `employee_number`, `ship_id`, `verified_at`, and `stage` — there's no
  general-purpose "applicant can update their own application" policy,
  so this can't be used to alter anything else on the application.
- Anti-scam design: `inquiries.sender_id` is nullable specifically so a
  member of the public (not signed in) can reach staff through `/contact`
  instead of a WhatsApp number. Staff reply in-platform, and the sender
  gets the reply by email (with the full text included, so even an
  anonymous visitor sees it without needing an account) as well as
  in-app on `/app/messages` or `/partner/messages` if they're signed in.
- Role changes are manual (via SQL) for now, which is normal for a small
  team. Once you have several staff/partner accounts to manage, the next
  step is a simple internal "invite by email" flow instead of hand-editing
  the database.

## Not yet built (next steps)
- The online quiz questions (`lib/data/exams.ts`) still live in code —
  an admin UI to author/edit them isn't built yet (the PDF upload covers
  the "post the official exam paper" need in the meantime)
- A proper invite flow for staff/partner accounts instead of hand-editing
  `profiles.role` via SQL
- Real visa/accommodation/flight provider integrations — the `provider`
  and `provider_ref` columns on `travel_requests` are ready for this
