# Alight

A faith-first social platform for youth groups. Milestone one: the safety spine — real accounts,
invite-only circles, moderated posting, and a leader review queue, with every rule enforced by
row-level security in Postgres rather than by the application layer.

## What is here

| Route | What it does |
|---|---|
| `/` | Landing page. Redirects to `/app` when signed in. |
| `/join` | Invite-code entry. The only door into the platform. |
| `/signup` | Account creation, gated by a validated code. Age tier is derived from date of birth. |
| `/login` | Sign in. |
| `/app` | Circle feed, composer, reactions. Shows your own posts while they await review. |
| `/app/review` | Leader-only moderation queue, with the audit trail. |
| `/app/circles` | Your circles, members, and — for leaders — invite code generation. |
| `/app/safety` | Age-tier matrix and safeguarding commitments, read from the live account. |

## Stack

- Next.js 15, App Router, React 19, server components and server actions. No client-side data
  fetching, so nothing about another member reaches the browser unless a policy allowed it.
- Supabase Postgres for data and auth (`@supabase/ssr` cookie sessions, refreshed in middleware).
- Hand-written CSS in `app/globals.css`, driven by design tokens. Light and dark are both defined.

## Environment

Both values are public by design — every protection lives in row-level security, not in these keys.
They are compiled in as defaults in `lib/supabase.ts` and can be overridden:

```
NEXT_PUBLIC_SUPABASE_URL=https://rjcuwaktwzzicxgbhfvn.supabase.co
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=sb_publishable_sf---yseSJjTSsCBiJeT7w_YUuCIBMB
```

## Running locally

```bash
npm install
npm run dev
```

## Deploying

```bash
npm i -g vercel
vercel link      # create or pick the "alight" project
vercel --prod
```

Vercel auto-detects Next.js; no build configuration is needed.

## The database

The schema, policies and functions live in the Supabase project `alight` (`rjcuwaktwzzicxgbhfvn`,
region eu-west-2). Pull the migration history into this repo with:

```bash
npx supabase link --project-ref rjcuwaktwzzicxgbhfvn
npx supabase db pull
```

### The guarantees, and where they are enforced

- **A post cannot skip review.** `posts` has no `UPDATE` policy at all. Status only changes through
  `review_post()`, which refuses anyone who is not a leader of that circle, and refuses a leader
  approving their own post. The `posts_force_pending` trigger overwrites status on every insert.
- **A pending post is invisible.** `posts_select` shows a row only to its author, to leaders of that
  circle, or — once approved — to circle members.
- **There is no user directory.** `profiles_select` requires a shared circle, or a confirmed parent
  link. Two members of different circles cannot see each other at all.
- **There is no way in but a code.** Sign-up validates against `circle_invites` before an account is
  made, and `redeem_invite()` checks uses and expiry inside a row lock.
- **Age tier is derived, never chosen.** `private.tier_for_dob()` runs in the sign-up trigger.
- **Membership helpers are not a public API.** They live in the `private` schema, which PostgREST does
  not expose, so they are usable inside policies but not callable over HTTP.
- **Every review decision is recorded.** `post_reviews` gets a row with reviewer, decision and time in
  the same transaction as the status change.

### Verified

Run against three throwaway accounts in one circle:

| Viewer | Pending post visible | Profiles visible | Invite codes visible |
|---|---|---|---|
| Author | yes | — | 0 |
| Another member of the same circle | **no** | 3 (their circle) | 0 |
| Leader | yes | — | all for their circle |

A member attempting `UPDATE posts SET status='approved'` and a member calling `review_post()` both
left the post pending.

## Before this takes real users

1. Turn on email confirmation and configure a sender in Supabase Auth. The sign-up flow already
   handles the confirm-then-join path.
2. Rate-limit `check_invite` at the edge; it is deliberately callable while signed out.
3. Add the platform-moderator tier as a second pair of eyes. `profiles.is_platform_moderator` exists
   and is not yet used by any policy.
4. Decide the Bible translation licence before scripture is stored in the database.
5. Commission an independent safeguarding review of the policies above.
