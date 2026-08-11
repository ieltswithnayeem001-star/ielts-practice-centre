# IELTS Practice Centre — Cloudflare Pages deployment

This folder is a ready-to-deploy Cloudflare Pages project:
- `index.html` — the frontend (static)
- `functions/api/auth.js` — register/login (Cloudflare KV)
- `functions/api/evaluate.js` — calls Gemini server-side, keeps your API key(s) private
- `functions/api/history.js` — reads past evaluations from KV

## 1. Get a Gemini API key
Create one for free at https://aistudio.google.com/apikey — no credit card needed.

Important: Gemini's free-tier rate limit is **per Google Cloud project, not per key**. If
you generate 10 keys inside the same project, they all share one quota pool — rotating
between them will *not* give you 10x throughput. To actually multiply your quota you'd
need keys from 10 separate Google accounts/projects. The backend here supports a
comma-separated list of keys either way (useful for resilience even with one project).

## 2. Create the KV namespace
In the Cloudflare dashboard: **Workers & Pages → KV → Create namespace**, name it
anything (e.g. `ielts-kv`). You'll bind it in step 4.

## 3. Push this folder to a GitHub repo
Cloudflare Pages deploys from a git repo (or you can use `wrangler pages deploy` directly
— see step 6 for the CLI alternative).

## 4. Create the Pages project
In the Cloudflare dashboard: **Workers & Pages → Create → Pages → Connect to Git**,
pick the repo, and use these build settings:
- Framework preset: **None**
- Build command: *(leave empty)*
- Build output directory: `/` (the repo root, since `index.html` is at the top level)

After the first deploy, go to the project's **Settings**:
- **Functions → KV namespace bindings**: add a binding named `IELTS_KV` pointing to the
  namespace you created in step 2.
- **Settings → Environment variables**: add `GEMINI_API_KEYS` as a **secret** (encrypted),
  value = your key, or multiple keys separated by commas (`key1,key2,key3`).

Redeploy after adding the bindings (Cloudflare Pages needs a redeploy to pick up new
bindings/env vars on some plans — trigger one from the dashboard if needed).

## 5. Visit your site
Cloudflare gives you a `*.pages.dev` URL automatically. Register a candidate account and
try an evaluation — you can watch requests to Gemini in Google AI Studio's usage page if
you want to confirm it's working.

## 6. Alternative: deploy from the command line
If you'd rather skip GitHub:
```bash
npm install -g wrangler
wrangler login
wrangler pages project create ielts-practice-centre
wrangler pages deploy . --project-name=ielts-practice-centre
wrangler pages secret put GEMINI_API_KEYS --project-name=ielts-practice-centre
```
Then bind the KV namespace to the project from the dashboard (Settings → Functions →
KV namespace bindings) as in step 4, since `wrangler pages deploy` alone won't wire that
up for you.

## Notes
- Passwords are hashed (SHA-256 + username as salt) before being stored in KV — better
  than plaintext, but this is still a lightweight demo auth system, not a vetted identity
  provider. Don't reuse a sensitive password here.
- The Gemini model used is `gemini-2.5-flash-lite` (free tier: 15 RPM / 1,000 requests
  per day, the most generous of the three free models). For higher-quality but lower-
  throughput evaluations instead, change `GEMINI_MODEL` in `functions/api/evaluate.js`
  to `gemini-2.5-flash` (10 RPM / 250 requests per day).
- Everything server-side (KV data, your Gemini keys) never reaches the browser — the
  frontend only ever talks to your own `/api/*` routes.
