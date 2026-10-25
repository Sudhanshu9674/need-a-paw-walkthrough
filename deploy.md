# Deploying Need a Paw?

Written for whoever is putting the web app online — likely you, once, and then
again in six months having forgotten all of it.

The mobile app is not covered here; it ships through EAS and TestFlight, which
is a separate exercise and not on the critical path for a beta.

---

## What talks to what

```
GitHub repo ──► Vercel (Next.js, root: apps/web) ──► Supabase (Postgres, Auth, Storage)
                                                        └──► SMTP provider (auth email)
```

Vercel hosts only the web app. All data, auth and file storage stay in Supabase,
which is already running — deploying does not move or copy any of it.

---

## 1. Vercel project settings

Create the project from the GitHub repo, then set:

| Setting | Value |
|---|---|
| Framework Preset | Next.js |
| **Root Directory** | **`apps/web`** |
| Include files outside root directory | **on** |
| Build / Install command | leave as the defaults |

Root Directory is the one that matters. This is an npm-workspaces monorepo and
`apps/web` depends on `packages/shared`, which ships TypeScript source and is
compiled through `transpilePackages`. With the checkbox off, the install cannot
see `packages/shared` and the build fails at module resolution.

## 2. Environment variables

Three, all public, all needed in **Production, Preview and Development**:

| Variable | Value |
|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | `https://<project-ref>.supabase.co` |
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | the `sb_publishable_…` key |
| `NEXT_PUBLIC_SITE_URL` | the deployed origin, no trailing slash |

Take the first two from your local `apps/web/.env.local` — they are the same
project. There is no service-role key here and there must never be one: the web
app talks to Supabase as the signed-in user and relies on row-level security. A
service-role key in a `NEXT_PUBLIC_` variable would be shipped to every browser
and would bypass every policy in the database.

> **Set these before the first build, not after.** `next.config.ts` reads
> `NEXT_PUBLIC_SUPABASE_URL` at build time to whitelist the image host. If it is
> missing during the build, the config falls back to a placeholder hostname, the
> image optimizer refuses every Supabase URL, and `/browse` renders with all
> photos broken — while the build itself reports success. Adding the variable
> afterwards changes nothing until you redeploy.

`NEXT_PUBLIC_SITE_URL` is a chicken-and-egg: you do not know the domain until
the project exists. Deploy once with the `*.vercel.app` domain, then set it to
the final domain and redeploy. It is used to build auth email links, so a wrong
value sends people to the wrong site from their confirmation email.

## 3. Supabase redirect allow-list

**Authentication → URL Configuration**:

- **Site URL**: the deployed origin
- **Redirect URLs**: add `https://<domain>/auth/callback` and
  `https://<domain>/**`

Keep `http://localhost:3000/**` in the list so local development still works.

Skipping this is the classic first-deploy failure: signup appears to work, the
email arrives, and the link bounces to an error because Supabase refuses to
redirect to an origin it was never told about.

## 4. SMTP — the one that actually blocks the beta

Supabase's built-in email sender is for development. It is rate-limited to a
handful of messages per hour across the whole project and sends from a shared
domain that lands in spam. With it, a shelter signing up does not receive a
confirmation code, and the signup dead-ends with no visible error.

Set a real sender under **Project Settings → Authentication → SMTP Settings**.
[Resend](https://resend.com) is the usual choice — free to 3,000 messages a
month, which is far more than a local beta needs. Any SMTP provider works.

Then raise the rate limit under **Authentication → Rate Limits**, which stays at
the development default until you change it.

Verify by signing up as a real address you control and confirming the email
arrives, not just that the form submits.

## 5. Before inviting anyone

- [ ] Fill `OPERATOR` in `apps/web/src/lib/legal.ts`. While the legal name,
      address and contact email are blank, `isLegalDraft` is true and a draft
      banner shows on `/terms`, `/privacy` and `/welfare`. That banner is doing
      its job — people should not be ticking six acknowledgements agreeing with
      an unnamed party — but it has to go before real shelters sign up.
- [ ] Sign up end to end as an adopter, on a phone, on cellular rather than
      office Wi-Fi.
- [ ] Open `/browse` signed out and confirm photos load. This is what catches
      the build-time image-host mistake above.
- [ ] Confirm an admin account exists. Admins are created by SQL against the
      database, never through the site.

## 6. Going live

The site ships behind a pre-launch wall. Opening it takes two steps, and the
first on its own leaves `/browse` returning 500 to every visitor:

1. Set `PRELAUNCH=0` and **redeploy**. The flag is baked into prerendered HTML
   at build time, so changing it in the dashboard without a rebuild leaves the
   holding page and the empty sitemap exactly where they were. Only route
   gating, which runs in the proxy, follows the variable immediately.
2. Re-grant the `anon` role, using the reverse migration written out at the top
   of `supabase/migrations/20260927090000_prelaunch_revoke_anon.sql`.

Step 2 is required because the public pages are server-rendered with the anon
key — `/browse` calls `public_pets()` — so while the grants are revoked the
page throws instead of rendering.

Check both afterwards: `/` should be the real landing page, and `/browse`
should list animals with photos.

---

## 7. Redeploying

Vercel builds on every push to the default branch. Database changes do not ride
along: migrations are applied with `npm run db:push` against the linked project,
separately and deliberately.
