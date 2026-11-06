# Open questions for Jonathan

Answer these before or during the milestone listed. Each answer becomes a
`DECISIONS.md` entry. "Blocks" = the earliest milestone that needs the answer.

| # | Question | Blocks | My recommendation |
|---|---|---|---|
| Q1 | **Which domain will you buy, and from which registrar?** Needed for `app.`/`api.` (D-02), `WEBAUTHN_RP_ID`, and the native passkey files (D-05). | M9 (local dev doesn't need it) | Buy it early (≈ $10–15/yr) so the Vercel rewrite fallback is only needed for preview URLs. Cloudflare Registrar sells at cost and puts DNS next to R2. |
| Q2 | **Do you accept the correction to the local-dev fallback?** You asked for "Vercel rewrites for local dev". Rewrites only run on Vercel or under `vercel dev`, and local dev doesn't need them, because `localhost` ports are the same site (D-02). | M2 | Yes: plain localhost for daily dev; rewrites for Vercel preview deploys; `vercel dev` only to reproduce the preview setup. |
| Q3 | **P1 has no keyword rules to port.** SPEC 3 and 7.2 say built-in rules are "ported from the P1 categorizer", but `categorize.py` only sums a `category` column that was filled in by hand. Should the spec say "written new in M4, seeded from P1's merchant → category pairs" instead? | M4 | Yes. P1's `transactions.csv` gives 15 starter pairs (MTA, LIRR → Transport, …). Reword the spec honestly. |
| Q4 | **Is `transactions.csv` real data?** It includes $3,000 rent to a named person. The spec forbids committing real statements. | M1 (moving P1 to `legacy/`) | Swap in invented names and amounts when it moves to `legacy/`. The file is already in public git history, so if it's real, decide whether the history needs rewriting too. |
| Q5 | **What is the SEWE Finance design canvas URL?** The v1.0 PDF linked it but the link didn't survive. | M1 (theme), M5 (reports UI) | Paste it into SPEC 9. |
| Q6 | **Which email provider for verification codes?** `EMAIL_PROVIDER_KEY` exists but no provider is named. | M2 | Pick one with a free tier and a Python SDK. Free-tier limits weren't verified for this spec; check them before choosing. |
| Q7 | **Will SEWE ever charge money?** Vercel Hobby is personal and non-commercial only. | M9 | Stay on Hobby while it's a portfolio project; budget Vercel Pro ($20/mo) the day it goes commercial. |
| Q8 | **When should the repo be renamed to `sewe-finance`?** Also, this session can read the repo but **can't push**. The Claude GitHub App isn't installed for it. | Before the first push | Install the GitHub App (github.com/apps/claude) and rename before M1 starts, so all V1 history lands under the new name. |
| Q9 | **Do you approve a small paid live-model eval for M7?** CI uses recorded mocks, but "chat matches Reports to the cent" should be checked once against the real model. | M7 | Yes. A 20-question run costs well under $1 at Sonnet 5.5 prices. |
| Q10 | **Should masking cover more P2P services?** D-09 covers Zelle, Venmo and Cash App. PayPal, Apple Cash and bank-to-bank transfers ("TRANSFER TO J DOE") also carry names. | M4 | Add PayPal and generic "TRANSFER TO/FROM <name>" patterns, and grow the list from your real statements during M3. |
| Q11 | **Is ≈ 8 weeks (7–10) at 20 h/week acceptable?** (D-12) | Now | If you need it faster, cut in this order: PDF import → voice → native passkeys (move past V1). |
| Q12 | **Is the M1 exercise in TypeScript OK?** The other eight are Python. M1's backend is just `/health`, so the meaningful function is on the frontend. | M1 | Yes. It's a gentle first contact with TS. |
