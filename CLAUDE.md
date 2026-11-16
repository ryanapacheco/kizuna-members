# Kizuna Members App

Mobile-first web app (PWA) for Kizuna Social Club: event booking and payment, members-only Mahjong play space, shop, guides and offers.

## Read first

- `docs/BRIEF.md` is the product spec. Build one phase at a time (section 9) and stop at its acceptance criteria.
- `docs/rules/` is the only source of truth for Mahjong rules and scoring. `00_confirmed_rulings.md` wins over the other files. Never fill a gap from general Mahjong knowledge; add it to `docs/OPEN_QUESTIONS.md` and ask.

## Stack

Next.js (App Router), TypeScript strict, Tailwind CSS, Supabase (Postgres, Auth, Realtime, Storage), Stripe, Resend, Vercel.

## Working rules

1. Plan first. For each phase, write the plan and wait for approval before coding.
2. TypeScript strict mode, no `any`. Validate every input on the server with a schema.
3. Every feature ships with tests. The game engine is built test-first.
4. All schema changes are migrations. Every new table gets row-level security in the same migration.
5. Money is integer pence. Payment state changes only in Stripe webhook handlers.
6. Secrets live in environment variables. Keep `.env.example` current. Never commit keys.
7. Build every screen at 380px first, following section 8 of the brief. No stock-template look.
8. All prices, perks and benefit text come from settings, never from constants in code.
9. Small commits with clear messages. Ask before adding a dependency.
10. Use the names Learning Tables, Social Tables, Sheung, Pong and Gong everywhere.
