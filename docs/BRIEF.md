# Kizuna Members App — Build Brief

Version: 16 November 2026. Decisions confirmed by Priya and Benaisha on that date are folded in. Open questions are in section 10.

## 1. How to use this brief

This file is the product spec. `docs/rules/` is the only source of truth for Mahjong rules and scoring. Build one phase per session and do not start the next until the acceptance criteria in section 9 pass.

**The product in one paragraph.** Kizuna Members is a mobile-first web app for Kizuna Social Club, a Taiwanese Mahjong social club in North West London. Anyone can create an account, book and pay for sessions, and see everything they have booked. Paying subscribers get a members-only play space where they play Kizuna-rules Mahjong against bots or other members, with every game and score saved. The app also sells mahjong sets and accessories, hosts rules and guides, and shows promotional offers. Priya and Benaisha run it from an admin area on their phones.

**Kickoff prompt for a new Claude Code session**

```text
Read docs/BRIEF.md, CLAUDE.md and everything in docs/rules/ in full before doing anything.

You are building the Kizuna Members app described in the brief.
1. Enter plan mode. Produce a plan for Phase 0 only (section 9).
2. List any decision in the brief you think is wrong or underspecified,
   with your recommended alternative. Wait for my answer.
3. Then build Phase 0. Stop when its acceptance criteria pass and show me
   how to verify each one.

Never start a later phase without being asked. Never invent Mahjong rules:
if docs/rules/ does not answer a question, add it to docs/OPEN_QUESTIONS.md
and ask me.
```

## 2. Decisions and assumptions

| Area | Decision | Why |
| --- | --- | --- |
| Platform | Installable web app (PWA), mobile-first at 380px wide. No native iOS or Android app in v1. | One codebase, no app-store review, no app-store commission on the membership. |
| Front end | Next.js (App Router), TypeScript in strict mode, Tailwind CSS. | Deploys in one step. |
| Back end | Supabase: Postgres, Auth, Realtime, Storage. Row-level security on every table. | Auth, database and live multiplayer from one service. |
| Payments | Stripe: Checkout for bookings and shop orders, Billing for the membership, Customer Portal for cancelling a membership. | Kizuna already pays Stripe 1.5% + 20p per booking through Luma. |
| Hosting | Vercel for the app, Supabase cloud for data, both in a UK or EU region. | Lowest effort. |
| Email | Resend for booking confirmations, receipts and reminders. | Simple transactional email. |
| Booking | The app replaces Luma for all bookings from launch. Kizuna absorbs the Stripe fee; prices shown are what the customer pays. | Removes the Luma fee problem. |
| Refunds | No self-service refunds. Users move a booking or transfer it to a friend. Admin can refund by exception. | Confirmed policy. |
| Game engine | A pure TypeScript package with no UI or network code, shared by client and server. The server is the authority in every game. | Testable rules, no cheating, same engine for bots and multiplayer. |
| Money in games | Games score points only. No stakes, no cash prizes, no purchasable points. | Keeps the app clear of gambling rules. |
| Shop | Collection at a session only. No delivery in v1. Stock is tracked per product. | Confirmed. |
| Currency and locale | GBP, en-GB, Europe/London time. | |

Prices, perks and benefit text live in admin-editable settings, never in code.

## 3. Roles and access

Four roles. Membership is a paid state on top of an account, driven only by Stripe subscription status.

| Capability | Guest | Account | Member | Admin |
| --- | --- | --- | --- | --- |
| Browse events, shop, public guides, offers | Yes | Yes | Yes | Yes |
| Book and pay for events | No, prompted to sign up | Yes | Yes | Yes |
| Bookings and order history | No | Yes | Yes | Yes |
| Buy from the shop | No, prompted to sign up | Yes | Yes | Yes |
| Guides unlocked by attendance | No | Yes | Yes | Yes |
| Member-only guides and offers, early booking, shop discount | No | No | Yes | Yes |
| Play space: bots, multiplayer, history, stats | Preview and join prompt | Preview and join prompt | Yes | Yes |
| Admin area | No | No | No | Yes |

Rules:

- A user is a member when their Stripe subscription status is `active` or `trialing`. When it lapses, the play space locks at the end of the paid period and their game history is kept.
- Admin is a flag on the profile, set by hand in the database for Priya, Benaisha and Ryan. There is no self-service route to admin.
- Every access rule is enforced on the server and in row-level security. Hiding a button is not access control.

## 4. Feature spec

The app has five bottom tabs: **Home**, **Events**, **Bookings**, **Play**, **Club**. Club holds Shop, Learn, Offers and the member's profile. Shop and Offers are also surfaced as cards on Home.

### 4.0 Home

- A greeting with the member's first name and, for members, a small gold "Member" mark.
- **Hero card:** the next booking, with date, time, venue, a "View ticket" button and directions. With no booking, the hero is the next event with places left.
- **Series progress:** Table One to Table Four as a four-step strip, with the next step bookable in one tap.
- **What's on:** a horizontal row of upcoming event cards led by photography.
- **From the club:** one offer card, one shop card, one guide card. Never more than three.
- Guests see the same screen without the personal parts, and a single "Join Kizuna" button.

### 4.1 Sign-up and login

- Email magic link and Google sign-in. No passwords.
- First login collects: first name, last name, a display name for the play space, and a separate opt-in tick for marketing emails (unticked by default).
- Profile screen: edit details, membership status, manage subscription, order history, delete account.

### 4.2 Events and booking

- Event types: **Learning Tables** (Table One to Table Four, £15 per session or £60 for all four), **Social Tables** social play (£20), **Private or corporate** (enquiry form only, no online payment in v1), and **Other** for non-Mahjong gatherings such as Book Circle, Mindfulness and Talks.
- The four Learning Tables: Table One (Beginners) the ABCs, rules, tiles and hand formations; Table Two (Beginners) hand building, strategising and game etiquette; Table Three (Intermediate) melds, scoring and tactical play; Table Four (Intermediate) game prep to play independently on Social Tables.
- Each event has: title, type, date, start and end time (default 7:30pm to 9:30pm), venue and address, description, price, capacity, optional prerequisite, status (draft, published, cancelled).
- Default venue: Fire & Ice Bar Lounge, 256 Uxbridge Rd, Pinner / Hatch End, HA5 4HS.
- Capacity is set in tables of 4. Default 8 places (2 tables); admin can raise it to 12, 16 or 20.
- Event list shows upcoming published events with places left. "Last 2 places" shows when 2 or fewer remain. Full events show "Join waitlist".
- A Learning Tables series can be booked whole (one payment, four bookings created) or one session at a time.
- Social Tables is open to anyone marked attended at Table Four, and to anyone else who ticks "I can play a full hand unaided" when booking.
- Members can book during an early booking window before general sale. The window length is a setting.
- Booking flow: choose event, choose quantity (1 to 4, with a name for each guest), apply promo code, pay by Stripe Checkout, land on a confirmation screen, receive a confirmation email with a calendar file.
- A place is held for 15 minutes while checkout is open, then released. Two people must never be sold the last place.
- No refunds. From Bookings the user can move a booking to another date of the same type with places left, or transfer the place to a friend by email, up to the start time. Admin can refund by exception.
- Waitlist: when a place frees up, the first person on the waitlist gets an email with a 12-hour hold.
- Reminder email 24 hours before each session.

### 4.3 Bookings

- Two lists: **Upcoming** and **Past**. Each card shows event, date, time, venue, number of places, amount paid and status (confirmed, moved, transferred, refunded, attended).
- Tapping a card opens the ticket: map link, add to calendar, receipt link, "Move" and "Transfer" buttons, and a QR booking code the host can scan or tick off on the night.
- Series progress strip at the top: Table One to Table Four, showing booked and attended.

### 4.4 Membership

- One plan in v1, billed monthly through Stripe Billing. Price and benefit text are editable in admin; the price is not yet decided.
- Benefits: the play space, member-only guides, an early booking window, and a discount in the shop. No discount on events.
- Join screen lists the benefits, then goes to Stripe Checkout in subscription mode.
- Members manage or cancel through the Stripe Customer Portal. Access continues to the end of the paid period.

### 4.5 Play space (members only)

- **Play a bot game:** one member against three bots, at Easy, Medium or Hard. Starts instantly.
- **Play with members:** create a table and share a 6-character code, or join the open lobby. Empty seats can be filled with bots. A member who disconnects for more than 60 seconds is replaced by a bot and can rejoin.
- **Game length:** a single hand, one round (East), or a full game (East, South, West, North). Turn timer 30 seconds by default in multiplayer, with auto-discard of the drawn tile on timeout.
- **Learning aids**, each a toggle: highlight legal claims, show which tiles complete the hand, show the running points of the current hand against the round minimum. On by default in bot games, off in multiplayer.
- **After each hand:** a score sheet listing every scoring element by its Kizuna name with points, the total, and who pays whom.
- **History:** every finished game is saved with date, opponents, result, final scores and each hand's score sheet.
- **Stats:** games played, hands won, win rate, highest-scoring hand, most frequent hand patterns, a hand catalog showing which patterns the member has completed.
- **Leaderboard:** monthly, among members, by total points. Display names only.
- **Score calculator:** the member taps in a hand and gets the Kizuna score sheet. Useful at the real table.
- No chat in v1. A fixed set of friendly reactions only, so no moderation is needed.

Guests and non-members see a preview of this tab and the join prompt. The engine spec is in section 6.

### 4.6 Shop

- Products: mahjong sets and accessories. Each has name, photos, description, price, stock count, active flag. Members get a percentage discount set in admin.
- Basket, then Stripe Checkout. Collection only: the buyer picks an upcoming session to collect at.
- Stock is reduced when payment succeeds. A product at zero stock shows "Sold out" with a "Tell me when it is back" email capture.
- Order history sits in Club with status: paid, ready to collect, collected, refunded.

### 4.7 Learn (rules and guides)

- Articles are written in Markdown in admin, with images, grouped into categories: Getting started, Tiles, How a hand works, Etiquette, Scoring, Hand catalog.
- Each article has an access level: **public**, **account**, **attended** (unlocks after the user is marked attended at a named session), or **member**.
- Release plan: basics are public; scoring unlocks after Table Three; the full hand catalog unlocks after Table Four. Locked articles show their title and how to unlock them.
- Seed content: convert the files in `docs/rules/` into draft articles. Admin sets the access level and publishes.
- Search across article titles and text the user can access.

### 4.8 Offers

- An offer has: title, text, image, optional Stripe promotion code, where it applies (events, shop, membership), audience (everyone, account holders, members), start and end dates.
- Active offers show as a card on Home and on an Offers screen. Tapping one applies its code at checkout.
- The app does not send marketing email in v1. It only records consent, so a list can be exported.

### 4.9 Admin area

Built for a phone, since it will be used at the venue.

- **Events:** create, edit, duplicate, publish, cancel (cancelling refunds every booking and emails attendees). Create a whole four-part series in one form.
- **On the night:** attendee list per event with a tick or QR scan to mark attended, guest names, and a walk-in button to add someone who paid in person.
- **Bookings:** search, refund in full or part by exception, move a booking to another date.
- **Members:** list with status, join date and games played.
- **Shop:** products, stock, orders and status changes.
- **Learn and Offers:** create and edit content.
- **Settings:** membership benefit text, early booking window, shop discount, default venue.
- **Reports:** revenue by month split by events, membership and shop; Stripe fees; occupancy per event; CSV export of each.

## 5. Data model

Postgres tables in Supabase. All ids are UUIDs, all money is integer pence, all times are `timestamptz`. Treat this as the starting schema and write it as versioned migrations.

| Table | Key fields | Notes |
| --- | --- | --- |
| `profiles` | id (matches auth user), first_name, last_name, display_name, is_admin, marketing_opt_in, stripe_customer_id | Created on first login. |
| `venues` | name, address, map_url | Seed with Fire & Ice Bar Lounge. |
| `series` | title, description, full_price_pence | Groups four Learning Tables events. |
| `events` | series_id, type, session_number, title, description, venue_id, starts_at, ends_at, price_pence, capacity, prerequisite_session, member_presale_at, status | type: learning, social, private, other. |
| `bookings` | user_id, event_id, quantity, guest_names, amount_paid_pence, status, stripe_checkout_session_id, stripe_payment_intent_id, booking_code, held_until, moved_from_booking_id, transferred_to_user_id | status: held, confirmed, moved, transferred, refunded. |
| `attendance` | booking_id, user_id, event_id, marked_by, marked_at | Drives guide unlocks and prerequisites. |
| `waitlist` | user_id, event_id, position, offered_at, expires_at | |
| `subscriptions` | user_id, stripe_subscription_id, status, current_period_end, cancel_at_period_end | Written only by the Stripe webhook. |
| `products` | name, description, images, price_pence, stock, active | |
| `orders` | user_id, total_pence, collection_event_id, status, stripe_payment_intent_id | |
| `order_items` | order_id, product_id, quantity, unit_price_pence | |
| `articles` | slug, title, category, body_markdown, access_level, unlock_session, sort_order, published | |
| `offers` | title, body, image, stripe_promotion_code, applies_to, audience, starts_at, ends_at | |
| `games` | mode, length, ruleset_version, seed, status, started_at, finished_at | mode: bots, members. |
| `game_players` | game_id, seat, user_id or bot_level, final_score, placing | |
| `game_hands` | game_id, hand_number, round_wind, dealer_seat, winner_seat, win_type, score_sheet (jsonb), payments (jsonb), action_log (jsonb) | The action log allows a full replay. |
| `game_state` | game_id, state (jsonb), updated_at | Live games only. Server access only. |
| `settings` | key, value (jsonb) | Perks, benefit text, defaults. |
| `stripe_events` | stripe_event_id, type, processed_at | Makes webhooks safe to receive twice. |

Row-level security:

- Users read and write only their own profile, bookings, orders, waitlist rows and game records.
- Events, products, offers and published articles are readable by all, with article bodies filtered by access level in a database function.
- `subscriptions`, `attendance`, `game_state`, `stripe_events` and all payment status fields are written only by server code using the service role.
- Admin access is checked in a single `is_admin()` function used by every admin policy.

## 6. Game engine spec

The game is Kizuna's 16-tile Taiwanese Mahjong exactly as written in `docs/rules/`. Never substitute Hong Kong, Riichi or American rules, terms or scoring. This is the largest and riskiest part of the build, which is why it comes last.

### 6.1 Rules the engine must implement

- **Tiles:** 144. Three suits of 1 to 9 (Circles, Bamboo, Characters), four Winds, three Dragons, four Flowers and four Seasons.
- **Hand:** each player holds 16 tiles; East takes a 17th and discards first. A winning hand is 5 melds and 1 pair, 17 tiles, plus one extra tile per Gong.
- **Turn order:** after East, play passes to the player on East's right and continues anticlockwise.
- **Melds and names:** Sheung (run, claimed only from the player who plays immediately before you), Pong (from anyone), Gong (from anyone, or concealed, or by promoting an open Pong with a self-drawn tile only). Use these names in the UI, not Chow, Pung or Kong.
- **Claim priority:** Mahjong, then Pong or Gong, then Sheung.
- **Bonus tiles:** Flowers and Seasons are shown face-up at once and replaced from the Flower Wall. A Gong also draws its replacement from the Flower Wall.
- **Rob the Gong:** a player waiting on the tile used to promote a Pong may win on it.
- **Minimum points to win:** East round 20, South 22, West 24, North 26. The Mahjong button appears only when the hand meets the minimum.
- **Dealer streak:** East stays dealer after winning. The streak lowers East's minimum and adds a streak bonus, using the table in the scoring document.
- **Payment:** on a discard win only the discarder pays the winner. On a self-pick win all three players each pay the full hand value.
- **Missed discard rule, chasing tiles penalty, the 4th West Wind exception, shared win on the same discard, and the drawn-game pot:** as written in the rules document.
- **7 and 8 Flowers instant wins:** as written.
- **Scoring:** every entry in the scoring and hand catalog, including the concealed and exposed variants, Chicken Hand, Nico Nico, Orphans, Gem hands and Pure Honour.

### 6.2 What changes because it is digital

- Shuffling, wall building, dice, seat draw and dealing are automatic, with a short animation.
- Flower replacement is automatic.
- Dead hands, False Mahjong, falsely called discards and a forgotten streak die cannot happen, because the engine only offers legal actions. Do not build their penalties.
- The winning tile is shown sideways against the set it completes, to mirror table etiquette.

### 6.3 Engine design

- A pure TypeScript package at `packages/engine`: no UI, no network, no randomness except a seeded generator passed in.
- Core shape: `applyAction(state, action)` returns the new state or a typed error; `legalActions(state, seat)` lists what a seat may do now; `viewFor(state, seat)` returns the state with other players' concealed tiles removed.
- Scoring is data-driven. Each catalog entry is one object with an id, the Kizuna name, points (exposed and concealed where they differ), a detector function, and the list of entries it replaces or cannot combine with.
- Where the rules documents give two values, put both in a `ruleset` config. Confirmed defaults are in `docs/rules/00_confirmed_rulings.md`; anything not confirmed there defaults to the first value listed and is recorded in `docs/OPEN_QUESTIONS.md`.
- How scoring entries combine is not fully specified. Do not guess. See section 10.
- Store `ruleset_version` on every game so old score sheets stay correct if rules are tuned later.

### 6.4 Bots

- **Easy:** keeps tiles that form pairs and runs, claims when it can, slow to win.
- **Medium:** discards to minimise tiles needed for 5 melds and a pair, and steers toward a pattern that reaches the round minimum.
- **Hard:** adds safe-discard reading of other players' exposed sets and discards, and weighs hand value against speed.
- Bots must respect the minimum-points rule; a bot that builds hands it cannot win with is a bug.
- Bots act after a random 0.6 to 1.5 second delay and have names in the Kizuna tone, not "Bot 1".
- Bots run on the server through the same `legalActions` and `applyAction` as people.

### 6.5 Multiplayer

- The server holds the only full game state, in a `game_state` row per live game. Clients send intended actions; the server validates with the engine, saves, and broadcasts each seat its own redacted view over Supabase Realtime.
- A client never receives another player's concealed tiles or the wall order.
- Claim windows: after each discard the server waits up to 8 seconds for claims, resolves by priority, then continues. It closes early when every eligible seat has passed.
- Turn timers run on the server. Reconnecting rebuilds the screen from the saved view.

### 6.6 Table UI

- Portrait phone layout: the member's hand along the bottom, sorted automatically; own exposed sets and flowers above it; three opponents shown compactly with their exposed sets and flower counts; discards in the centre; round wind, dealer marker, streak count and wall count always visible.
- Tap a tile to select, tap again to discard. Claim buttons appear in a bar: Mahjong, Pong, Gong, Sheung, Pass.
- Tiles are SVG. Use an openly licensed tile set that includes Flowers and Seasons, check the licence allows commercial use, and record it in `docs/LICENCES.md`.

### 6.7 Tests that must exist

- A fixtures file with at least one worked hand per catalog entry and its expected score sheet. Priya signs this off before the play space launches.
- Tile conservation: 144 tiles accounted for after every action.
- Determinism: the same seed and actions give the same game.
- A soak test that plays 10,000 all-bot games with no illegal state, no stuck game and no win below the round minimum.

## 7. Payments

Stripe is the only source of truth for money. The app never marks anything paid from the browser; only a verified webhook does.

- **Bookings and shop orders:** Stripe Checkout in payment mode, with Apple Pay and Google Pay enabled. Create the booking or order as `held` first, pass its id in the Checkout metadata, and confirm it in the webhook.
- **Membership:** Stripe Checkout in subscription mode, then the Customer Portal for card changes and cancelling.
- **Promo codes:** Stripe promotion codes, enabled on Checkout. Offers in the app reference them by code.
- **Webhooks to handle:** `checkout.session.completed`, `checkout.session.expired` (release the hold), `charge.refunded`, `customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.payment_failed`.
- **Webhook rules:** verify the signature, record the event id in `stripe_events`, and skip any id already processed.
- **Refunds:** admin only, through the Stripe API. Booking status changes only when the refund webhook arrives.
- **Capacity safety:** taking a place and creating the hold happen in one database transaction that locks the event row.
- **Receipts:** Stripe's own receipt emails, plus the app's confirmation email.
- **Keys:** test mode keys until launch. Secrets only in environment variables, never in the repo or client code.
- **Account:** the Stripe account belongs to Kizuna (Priya and Benaisha) and pays out to the Kizuna business bank account.

## 8. Brand and UI

The target is Apple-clean structure carrying Kizuna's editorial brand: calm cream screens, one confident serif headline per screen, forest green for action, gold as a thin accent. The reference for brand feel is Kizuna's own Instagram artwork (@kizunasocialclub); the reference for structure is Apple's own apps (Wallet, Fitness, App Store Today).

### 8.1 Principles

1. **Function before polish.** Booking a place takes three taps from Home and never fails silently.
2. **One primary action per screen**, in a full-width green button fixed above the tab bar.
3. **Content is the decoration.** Photography, tiles and type do the work. No gradients, no drop-shadow stacks, no icons used as ornament.
4. **Gold is a whisper.** Under a tenth of any screen: hairlines, small marks, the active tab.
5. **Feels native.** Sheets, large titles, grouped lists and spring motion behave the way an iPhone user expects.

### 8.2 Colour tokens

| Token | Value | Use |
| --- | --- | --- |
| `green-900` | `#0B3B24` | Primary buttons, headlines, active tab icon, game table |
| `gold-500` | `#C6A87C` | Hairlines, small marks, active indicators, text on green only |
| `cream-50` | `#FAF8F5` | App background |
| `white` | `#FFFFFF` | Cards and sheets on cream |
| `ink-900` | `#1C1C1A` | Body text |
| `ink-500` | `#6B6B66` | Secondary text |
| `line` | `#E9E4DB` | Dividers and card borders |
| `danger` | `#B3261E` | Errors and destructive actions only |

Rules: gold is never used for text or icons on cream or white, because it fails contrast; use it on green, or as a line. Cream text on green is the standard for buttons. Light theme only in v1; the Play table is the one green-background screen.

### 8.3 Type

- **Display:** a high-contrast serif with a true italic, matching the Instagram artwork (Playfair Display or Bodoni Moda from Google Fonts unless Kizuna supplies its brand font). Used only for screen titles, hero headlines and prices. One per screen.
- **Text and UI:** the system font stack, so it renders as San Francisco on iPhone and Roboto on Android.
- **Labels:** small capitals in the UI font with wide letter-spacing (11px, +0.12em), as in "LEARNING TABLES" on the artwork. Used for section labels and pill tags.
- Scale: large title 34, title 24, headline 17 semibold, body 17, secondary 15, label 11. Line height 1.3 for body.

### 8.4 Layout and components

- 8-point spacing grid. Screen margins 20px. Cards: white, 16px radius, 1px `line` border, no shadow or a single very soft one.
- **Navigation:** large left-aligned title that collapses on scroll; translucent tab bar with five icons and labels; back by swipe from the left edge.
- **Sheets:** booking, basket, filters and promo codes open as bottom sheets with a grab handle, not new pages.
- **Event card:** 4:3 photo, label (for example "TABLE TWO · BEGINNERS"), serif title, date and time, venue, and a places-left pill.
- **Pills:** the green capsule with a gold hairline from the artwork, for event type and status.
- **Progress strip:** gold dots joined by a hairline, as on the Learning Tables artwork, for the four-part series.
- **Ticket:** a booking opens as a pass-style card with event, date, guest names and a QR code the host scans or ticks.
- **Member card:** in Club, a green card with the member's name, "Member since" and the gold Kizuna mark.
- **Lists:** grouped inset lists for settings, order history and admin, in the iOS style.
- **Buttons:** 52px tall, 14px radius. Primary green with cream text; secondary outlined green; text buttons for minor actions.
- **Forms:** one column, 52px fields, labels above, inline errors, correct keyboard type per field.

### 8.5 Payment and booking feel

- Apple Pay and Google Pay offered first through Stripe, card second.
- After payment: a full-screen confirmation with a brief tick animation, the ticket, "Add to calendar" and "Share with a friend".
- Skeleton placeholders while loading, never spinners on a blank screen.
- Every empty state has a line of friendly copy and one action. Example for Bookings: "Nothing booked yet. Table One is where everyone starts."

### 8.6 Motion

- Spring transitions of 200 to 350ms for sheets and page pushes. Tiles deal, draw and discard with short eased movement.
- Respect the reduced-motion setting.

### 8.7 Voice and imagery

- Name in the app: **Kizuna Social Club**. Tagline: "The art of coming together". Hook for guests: "Staying local just got exciting."
- Warm, local, upbeat, never intimidating. "Save my place", not "Purchase ticket". No hard sell in the shop or the membership prompt.
- Real photos from Kizuna nights and the brand's green velvet, ivory tile and gold styling. No casino imagery. Admin uploads every image.

### 8.8 Installed-app behaviour and accessibility

- Web app manifest with standalone display, cream splash screen, green theme colour, and icons built from the Kizuna mark.
- Respect iPhone safe areas at the top and above the home indicator. No rubber-band scrolling on fixed screens such as the game table.
- An "Add to home screen" prompt after the first booking, with instructions for iPhone and Android.
- WCAG AA contrast, visible focus states, 44px minimum tap targets, screen-reader labels on every tile and button, and tile faces that do not rely on colour alone.

### 8.9 How to work on design

- Build the tokens and components in 8.2 to 8.4 first, as a small component library with a `/styleguide` page, before any feature screen.
- Build Home, an event card, the booking sheet and the ticket as the first four screens and stop for review at 380px before going further.
- No UI kit defaults left visible. If a screen looks like a stock template, it is not done.

## 9. Build phases and working rules

Six phases, each shippable on its own. Phase 1 is the launch: it replaces Luma and starts taking bookings. The game arrives in Phases 4 and 5.

### Phase 0: Foundation

Next.js app, Supabase migrations for the whole schema, auth, brand tokens and component library with `/styleguide`, the five-tab shell with empty screens, deploy pipeline, Stripe in test mode.

- [ ] A new user can sign up by magic link and by Google, complete their profile, and log out.
- [ ] The shell and styleguide match section 8 and work at 380px and on a laptop.
- [ ] Row-level security is on for every table, with a test proving one user cannot read another's rows.
- [ ] A preview deployment is live and `npm test`, lint and type-check pass in CI.

### Phase 1: Events, booking and Bookings (launch)

Sections 4.0, 4.2, 4.3, the events and on-the-night parts of 4.9, and section 7 for one-off payments.

- [ ] Admin creates a four-part Learning Tables series and a Social Tables night from a phone.
- [ ] A user books one session, books a whole series, and books two places with a guest name.
- [ ] Payment confirms only through the webhook; closing the browser after paying still confirms the booking.
- [ ] Two users racing for the last place: exactly one succeeds.
- [ ] A user moves a booking to another date and transfers a place to a friend; there is no self-service refund.
- [ ] Social Tables booking is allowed for a Table Four graduate, and for others only after the self-check tick.
- [ ] Confirmation and reminder emails arrive with correct date, time and venue.
- [ ] Admin ticks attendance on the night and adds a walk-in.
- [ ] Bookings shows upcoming, past and series progress correctly.

### Phase 2: Membership, Learn and Offers

Sections 4.4, 4.7, 4.8.

- [ ] A user subscribes, is shown as a member within seconds, and can cancel in the Customer Portal.
- [ ] A failed or cancelled subscription removes member access at period end.
- [ ] Members can book during the early booking window; non-members cannot.
- [ ] The rules documents exist as draft articles; an "attended" article unlocks only after attendance is ticked.
- [ ] An offer with a promotion code appears to the right audience and applies at checkout.

### Phase 3: Shop

Section 4.6 and the shop parts of 4.9.

- [ ] A user buys a set and an accessory for collection at a chosen session.
- [ ] Stock never goes below zero under simultaneous purchases.
- [ ] Admin moves an order through its statuses and refunds one.
- [ ] The member discount applies only to members.

### Phase 4: Game engine, bot play, history

Sections 6.1 to 6.4, 6.6, 6.7 and the single-player parts of 4.5. Build the engine and its tests before any table UI.

- [ ] Every open rules question in section 10 is answered and recorded in `docs/rules/00_confirmed_rulings.md`.
- [ ] The scoring fixtures file is signed off by Priya and all fixtures pass.
- [ ] The 10,000-game soak test passes.
- [ ] A member plays a full game against three bots on a phone, sees a correct score sheet after each hand, and finds the game in History.
- [ ] The score calculator returns the same result as the engine for the same hand.
- [ ] A non-member cannot start a game by any route, including calling the API directly.

### Phase 5: Multiplayer

Section 6.5 and the rest of 4.5.

- [ ] Four members on four phones complete a full game with no desync.
- [ ] Inspecting network traffic never reveals another player's concealed tiles.
- [ ] A player who drops is replaced by a bot after 60 seconds and can rejoin.
- [ ] Two members plus two bots works. The monthly leaderboard updates.

### Working rules

1. Plan first. For each phase, write the plan and wait for approval before coding.
2. `docs/rules/` is the only source of truth for Mahjong. If it does not answer a question, add it to `docs/OPEN_QUESTIONS.md` and ask. Never fill a gap from general Mahjong knowledge.
3. TypeScript strict mode, no `any`. Validate every input on the server with a schema.
4. Every feature ships with tests. The engine is built test-first.
5. All schema changes are migrations. Never edit the database by hand.
6. Money is integer pence. Payment state changes only in webhook handlers.
7. Every new table gets row-level security policies in the same migration.
8. Secrets live in environment variables. Keep `.env.example` current.
9. Build every screen at 380px first and check it in a browser before calling it done.
10. Small commits with clear messages. Ask before adding a dependency.
11. All prices, perks and benefit text come from settings, never from constants in code.
12. At the end of each phase, update `README.md` with setup steps and write a short manual test script Priya or Benaisha could follow.

## 10. Decisions and open questions

### Confirmed on 16 November 2026

| Topic | Decision |
| --- | --- |
| Session names | Learning Tables (Table One to Table Four) and Social Tables. |
| Luma | The app replaces Luma for all bookings from launch. |
| Cancellations | No refunds. Users can move a booking or transfer the place to a friend. Admin can refund by exception. |
| Social Tables access | Table Four graduates, plus anyone who ticks "I can play a full hand unaided" when booking. |
| Membership price | Not set yet. Build it as an admin setting. |
| Member perks | Play space, member-only guides, an early booking window, a discount in the shop. No discount on events. |
| Shop fulfilment | Collection at a session only. |
| Guides | Unlock by session attended: basics public, scoring after Table Three, full hand catalog after Table Four. |
| Turn order | After East, play passes to the player on East's right and continues anticlockwise. |
| Win on a discard | Only the discarder pays the winner. |
| Win by self-pick | All three players each pay the full hand value. |
| Point values | Everyday values: self-pick +2, self-pick from the Flower Wall +5, drawn-game pot 5 each. |

### Still open

**Business**

- [ ] Open Kizuna's own Stripe account, paying out to the business bank account.
- [ ] Membership price per month.
- [ ] Length of the early booking window, and the shop discount percentage.
- [ ] Sale price for a set, and which accessories to stock.
- [ ] Terms, privacy policy and the no-refund wording, checked before taking live payments.
- [ ] The brand display font, if Kizuna has one.

**Rules for the engine (needed before Phase 4)**

- [ ] How scoring entries combine: which patterns stack, and which replace each other beyond the cases the catalog states.
- [ ] The remaining two-value entries: chasing tiles 5 and 10 or 3 and 5, Brothers four sets 40 or 50, Uncle Pongs four sets 50 or 40, 7 Flowers flat 20 or the round minimum.
- [ ] Does East pay or receive extra as dealer, beyond the streak bonus?
- [ ] Does the dealer streak continue past a fourth win, and what is the minimum then?
- [ ] After a drawn game, does East pass the deal? The rules document says yes; confirm.
- [ ] Starting points per player and how a full game ends.
