# IDEA: Budget Deck Builder (Yu-Gi-Oh!)

> **For Claude Code:** This file is the idea, not the spec. Use it to write `SPEC.md` for the **MVP only** (section 4). The full version (section 5) is context so the MVP's design doesn't block it later. Do not spec or build full-version features. Before writing the spec, interview me on every item in section 10 (Open questions) and anything else that's ambiguous.

---

## 1. One-line summary

Load a Yu-Gi-Oh! deck you want to build, and the app tells you the cheapest way to finish it from the cards you already own, suggests cheaper substitutes when you're over budget, and scores each version by how consistently it opens with a playable hand.

## 2. Why this exists

- **The problem:** competitive decks are expensive, and most players already own part of what they need. Working out what to buy, which printing is cheapest, what can be swapped for something cheaper, and how much those swaps hurt the deck is slow, manual work across several sites.
- **Why not just ask an AI:** a prompt can suggest substitutes, but it can't reliably find the cheapest combination of printings under a budget, and it can't calculate opening-hand odds correctly. This app's reason to exist is **optimization and simulation that must be mathematically right**, with AI used only for suggestions the user confirms.

### What already exists (checked Nov 2026)

Each individual piece exists somewhere. **This app does not compete on any single piece**; its angle is the combination.

| Existing tool | What it covers | What it doesn't do |
|---|---|---|
| **YGOPRODeck card pages** | Per-printing prices (TCGplayer, Cardmarket), 120-day price history, "similar cards," deck/meta usage stats, banlist history | Per-card only; nothing about completing a specific deck on a budget |
| **YGOPRODeck deck tool** | Deck prices; on their site, can filter to cards in your collection | No budget-driven substitutions, no consistency scoring |
| **TCGplayer, Dragon Shield, YuScan apps** | Scanning, collection tracking and value, deck building with legality checks | No substitutions or consistency scoring |
| **YGOrg Combo Rate Calculator, hypergeometric calculators** | Opening-hand odds (Monte Carlo and exact) | Not connected to a collection, prices, or substitutions |
| **MTGGoldfish Super Brew (Magic: The Gathering)** | Shows owned vs. missing cards for a deck, cost to finish, substitutions from your collection | Magic only; closest analog to this app |

**The gap:** for Yu-Gi-Oh!, nothing combines your collection, a budget, role-aware substitute suggestions, and a consistency score for each version. Pitch: "Magic players have this; Yu-Gi-Oh! players don't."

### Why "similar cards" isn't the same as substitutes

YGOPRODeck's "similar cards" are based on related names and effects, so they're mostly cards from the same archetype. Archetype siblings usually aren't substitutes: they're more pieces of the same engine, at their own prices. A real substitute fills the same **role** in **this specific deck** at a lower price. This is why substitution in this app is role-aware and confirmed by the user (see 5.5 and 5.7), and it's a good example for the design doc.

## 3. Goals

### Product goals
- I use it to plan my own decks and purchases.
- Useful enough that friends at locals would use it for budget builds.

### Portfolio goals
This is one of two portfolio projects for a job search targeting NYC fintech/commerce companies (e.g. Capital One, Whatnot). The other project (a live bill-splitting app) covers real-time and multi-user work, so this one should demonstrate:
- **Algorithms:** a real optimization problem (choosing printings and substitutes under a budget), not just summing prices.
- **Probability and simulation:** exact calculations where possible, simulation where not, and validation of one against the other.
- **Data engineering:** a nightly sync of an external API into Postgres, respecting the API's rate limits and rules.
- **AI with a clear, checkable job:** an LLM suggests card roles and substitutes; the user confirms. Never a black box.
- **Professional engineering:** tests, CI, AWS deployment, a design doc with reasoned decisions.
- **AI-assisted development:** built with Claude Code using a spec-first workflow; specs and `CLAUDE.md` kept in the repo.

### Required stack
- **Frontend:** React + TypeScript
- **Backend:** Node.js + TypeScript (API)
- **Compute worker:** Python (optimizer, simulation, LLM role/substitute suggestions)
- **Database:** Postgres (pgvector if used for substitute similarity)
- **Infra:** Docker (Docker Compose locally), AWS for deployment (S3 for card images, a queue for compute jobs, containers for services)
- **Version control/CI:** Git, GitHub Actions

## 4. Data source: YGOPRODeck API

- **API:** YGOPRODeck v7 (`https://db.ygoprodeck.com/api/v7/cardinfo.php`). Free, no API key. Docs: https://ygoprodeck.com/api-guide/
- **Rules that shape the design:**
  - Rate limit: 20 requests/second. Exceeding it blocks access for 1 hour.
  - Data must be downloaded and stored locally rather than queried repeatedly.
  - Card images must be downloaded and self-hosted; no hotlinking.
- **Prices:** `card_prices` gives the lowest price across vendors and versions. Per-printing prices (needed to pick the cheapest printing) come from per-set data (e.g. the `tcgplayer_data` option / `card_sets` with `set_price`). Verify exact fields during the spec phase.
- **Design consequence:** a **nightly sync job** pulls the full card database and current prices into Postgres. The app serves everything from its own database, never from the API at request time.
- **Copyright:** card text and images belong to Konami. Display them in the app only; don't offer bulk exports of card data or images.
- **Dependency risk:** the app relies on YGOPRODeck's free API while partly overlapping their site. Follow their rules strictly (rate limits, local storage, self-hosted images) and credit YGOPRODeck visibly in the app as the data source.

## 5. MVP scope (build this first)

### 5.1 Card database sync
- Nightly job syncs all cards, printings (set codes, rarities), and the current price of each printing into Postgres. Prices are overwritten each sync; no price history in the MVP.
- Card images downloaded to S3 (or local storage in development).
- Sync is idempotent (safe to re-run) and stays within the rate limit.

### 5.2 Collection
- Add cards by searching the name or typing the set code (e.g. `LOB-EN001`), with quantity, edition, and condition.
- Shows current total collection value (no history; other apps already cover that).
- Single user with simple sign-in (multi-user is fine later, but nothing fancy now).

### 5.3 Deck import and validation
- Import a deck from a `.ydk` file (the standard decklist format) or a pasted list.
- Validate the deck: main deck 40–60 cards, extra deck 0–15, side deck 0–15, copy limits, and the current Forbidden & Limited list.

### 5.4 Completion plan
- Compare the deck against the collection: which cards I own, which are missing, and how many copies of each.
- For each missing card, pick the **cheapest printing** (all printings of the same card play identically).
- Output a shopping list with per-card cost and total cost to finish the deck.

### 5.5 Card roles
- For each card in a deck, assign a role: e.g. starter, extender, hand trap, board breaker, non-engine, brick.
- **Roles are per deck, not per card:** the same card can be a starter in one deck and filler in another.
- An LLM suggests roles from card text; the user confirms or changes them. Confirmed roles are saved with the deck.

### 5.6 Consistency score
- Calculate how often an opening hand contains what the deck needs (e.g. "at least one starter", "a starter plus a hand trap").
- Cover both going first (5-card hand) and going second (6 cards after the first draw).
- **Exact math** (hypergeometric / multivariate hypergeometric) for conditions it can handle.
- **Monte Carlo simulation** for complex conditions, with a fixed random seed (reproducible results) and a confidence interval shown.
- Simulation results are validated against exact results in tests wherever both apply.

### 5.7 Budget mode with substitutes
- User sets a budget for completing the deck.
- If the completion cost is over budget, the app proposes substitutions: expensive missing cards swapped for cheaper cards that fill the same role.
- Substitute candidates: same role, similar function, legal, affordable. Found via card-text similarity (e.g. embeddings + pgvector) plus filters, with LLM re-ranking. **The user confirms every substitution.**
- The optimizer chooses which substitutions to make to fit the budget while keeping the consistency score as high as possible.
- Show versions side by side: full deck vs. budget version(s), each with cost and consistency score. Example: "Full deck: $240, opens a starter 88% of the time. Budget version: $65, 81%."

### 5.8 Explicitly out of scope for the MVP
- Photo scanning / card recognition
- Price history and collection value over time (YGOPRODeck and Dragon Shield already cover this)
- Live trading or trade sessions
- Meta deck library or tournament data
- Price alerts
- Deck sharing or social features
- Full drag-and-drop deck editor (basic edits are enough)
- Combo-line simulation (playing out actual turns)
- Native mobile apps

## 6. Full version (future, context only; do not build yet)

Roughly in priority order:

1. **Photo scanning:** identify cards from photos using image matching (e.g. CLIP embeddings), with OCR of the printed set code as a fallback. Must distinguish different printings of the same card. Estimated 3–5 extra weeks.
2. **Multi-deck planning:** plan several decks at once and account for cards shared between them.
3. **Price alerts:** notify when missing cards drop below a target price.
4. **Meta awareness:** import popular decklists (only from sources that allow it); flag Forbidden & Limited changes that affect my decks or collection.
5. **Collection analytics:** daily price snapshots, value over time, biggest gainers/losers, cost basis.
6. **Sharing:** public deck pages with cost and consistency score.
7. **Deeper simulation:** model actual combo lines, not just which cards are in hand.
8. **Side deck planning** for specific matchups.

**Design implication for the MVP:** keep the collection model printing-aware (set code, rarity, edition, condition) so photo scanning and analytics can be added later without a migration headache.

## 7. Correctness rules (non-negotiable)

- **Money:** all prices stored and computed as integer cents. No floating-point arithmetic for money.
- **Determinism:** the optimizer and simulation produce the same result for the same inputs (fixed seeds, documented tie-breaks).
- **Legality:** no suggested deck or substitute may break copy limits, deck size rules, or the current Forbidden & Limited list.
- **Transparency:** every number shown (cost, probability, score) can be traced to its inputs. Probabilities from simulation always show their margin of error.
- **AI output is never trusted blindly:** role and substitute suggestions are proposals until the user confirms them.

## 8. Testing strategy

- **Optimizer:** compare against brute-force search on small decks and budgets where every combination can be checked; property tests (totals within budget, all results legal).
- **Probability:** exact formulas tested against known values; Monte Carlo results must land within their confidence interval of the exact answer wherever both apply.
- **Sync:** run against a saved snapshot of API responses in CI (never hit the live API in tests); verify idempotency and rate-limit compliance.
- **Deck import:** fixture `.ydk` files, including malformed ones.
- **End-to-end:** import deck → completion plan → budget version → consistency comparison.

## 9. Milestones (≈10 hours/week; ~6–7 weeks best case)

| Week | Milestone |
|---|---|
| 1 | Repo, Docker Compose, schema, nightly card sync with image storage and current prices |
| 2 | Collection (add by name/set code, current value), deck import, validation, completion plan with cheapest printings |
| 3 | Card roles (LLM suggestions + confirmation), exact probability math |
| 4 | Monte Carlo simulation, validated against exact math; consistency score UI |
| 5–6 | Substitute candidates, budget optimizer, side-by-side version comparison |
| 7 | AWS deployment, CI/CD, monitoring, README and design doc |

This project is planned to be built **after** the bill-splitting app, reusing its Docker/AWS/CI setup.

## 10. Open questions (resolve during spec interview)

1. **Optimizer formulation:** integer linear programming, a heuristic search, or greedy with local improvements? How is "keep consistency high" turned into something the optimizer can work with (e.g. role-count constraints vs. calling the simulator in the loop)?
2. **Which consistency conditions** are built in, and can users define their own ("starter + extender", "any 2 of these 5 cards")?
3. **Exact vs. simulated:** where exactly is the line between the two, and how many simulated hands are enough?
4. **Price per printing:** which price field and vendor to use, and how to handle printings with no price data.
5. **Condition and edition:** does the MVP price by condition, or assume one market price per printing?
6. **Substitute search:** embeddings + pgvector, rule-based filters, LLM only, or a combination? How many candidates per card?
7. **Where compute runs:** Python worker behind a queue (async jobs with progress) or a synchronous call? How are results cached?
8. **Authentication:** simplest approach that works for a single user now and doesn't block multi-user later.
9. **Forbidden & Limited source:** confirm the API's banlist fields and how often to refresh them.
10. **Sync schedule and failure handling:** what happens if the nightly sync fails or the API is down?

## 11. Notes for the spec

- Keep the MVP small. If it isn't needed for "import a deck, see what it costs to finish from my collection, get a cheaper version, and see how much consistency it loses," it's out.
- Record each section 10 decision in the spec with the chosen option and a one-line reason; these become interview talking points.
- Break the spec into thin, testable milestones that each leave the app in a working state.
