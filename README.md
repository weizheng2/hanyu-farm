# Hanyu Farm

Gamified Chinese learning: study with spaced repetition, earn coins, spend them on a small farm.

**Learning goal:** practice DDD, hexagonal architecture, CQRS, Symfony and Angular.

---

## MVP scope (2 weeks)

**In:**
- Single-player, no auth (one default learner in the backend)
- Seeded beginner deck (~HSK 1-6 cards)
- Study screen: show due card → reveal answer → rate (`again` / `know it`)
- Simple SRS scheduler: due cards come back; interval grows when you get them on the first try
- Farm screen: earn coins from reviews, plant/harvest crops on a few plots

**Out (for now):**
- Login, multi-user, social
- Let user Add-cards (owner_user_id column in cards)
- Card phrases, categories
- Import/export, audio, handwriting, advanced quiz modes
- XP / leveling, real-time farm sim, heavy animations, mobile app
- Event sourcing, complex read models
- Example phrases (`phrases`, `phrase_translations`, `card_phrases`) — cards + translations first

---

## Review process
**In:**
Option 1: Two buttons

Today's Progress
---------------
New Words: 1/2
Reviews Due: 18

[ Learn New Words ]
[ Review ]

Why?

*  Easy for users to understand. 
*  Easy to implement. 
*  Easy to debug and tune. 
*  Lets users choose based on available time. 
This is especially useful when you're still figuring out your learning algorithm.

**Out (for now):**
Option 2: One "Study" button (better once the product matures)

Today's Progress
---------------
2 new words
18 reviews due

[ Start Studying ]

The app decides:
Review
Review
New word
Review
Review
New word
..

The user doesn't need to understand the distinction between learning and reviewing.
This is what many polished learning apps do because it's simpler from the user's perspective.

---

## Spaced Repetition Algorithm

**In:**
Option A: Simple staged intervals (easier to build/debug first, no ease_factor needed)
A fixed ladder of intervals — "Know it" moves up a stage, "Learn it" drops back to stage 0 or 1.

stages = [10min, 1day, 3days, 7days, 14days, 30days, 90days]
max_stage = 6  (index of last element)

On "Know it":
  stage = min(stage + 1, max_stage)
  next_review_at = now + stages[stage]
  times_correct += 1
  status = 'known' if stage >= 3 else status

On "Learn it":
  stage = max(stage - 2, 0)
  next_review_at = now + stages[stage]
  status = 'learning'

times_seen += 1 (always)
last_reviewed_at = now (always)
New card, first time: stage = 0, status = 'new', row created on first answer.

**Out (for now):**
Option B: Simplified SM-2 (most common)
This is what Anki and most flashcard apps use, adapted for 2 buttons instead of 4.
interval (days), ease_factor (starts at 2.5)

On "Know it":
  if status == 'new' or 'learning':
    interval = 1 day (first time), then 6 days (second correct)
  else:
    interval = interval * ease_factor
  ease_factor = ease_factor + 0.1  (small bump, optional)
  status = 'known' (or stays, once interval is large enough)

On "Learn it":
  interval = max(interval * 0.5, 10 minutes)  -- shrink, don't fully reset
  ease_factor = max(ease_factor - 0.2, 1.3)   -- penalize, with a floor
  status = 'learning'
  next_review_at = now + interval
next_review_at = now() + interval. Reasoning: each correct answer grows the gap multiplicatively (so easy cards quickly get spaced way out), each miss shrinks it and slightly lowers ease_factor so that card grows more slowly in future. This is why ease_factor exists in the schema — without it, all cards grow at the same rate regardless of how often a user struggles with them.

---

## Tables
**In:**
- cards
    id
    hanzi 
    pinyin

- cards_translations
    id
    card_id
    language
    text

- user_card_progress
    id
    user_id
    card_id          (FK -> cards.id)
    status           -- 'new' | 'learning' | 'known'
    stage            -- integer index into the stage ladder (replaces interval + ease_factor)
    times_seen
    times_correct
    last_reviewed_at
    next_review_at
    created_at

- user
    id
    login etc
    review_quantity_cap
    daily_goal

- app_settings

**Out (for now):**
- user_settings
- user_card_reviews (analytics, stats)
  id
  user_id
  card_id
  reviewed_at
  result            -- 'know' / 'learn'
  interval_before
  interval_after

- phrases (id, chinese_text, pinyin)
- phrase_translations (phrase_id, lang, translation)
- card_phrases (card_id, phrase_id)
- categories (id, name   -- e.g. 'Animals', 'HSK1', 'Food', 'Verbs')
- card_categories (card_id (FK -> cards.id), category_id (FK -> categories.id))

---

## Domain glossary

**Domain modeling (hexagonal)**
Card — value object/entity: id, hanzi, pinyin, translations
ReviewRating — enum: Again | KnowIt
ReviewState — value object: stage, status, timesSeen, timesCorrect, lastReviewedAt, nextReviewAt. Holds the staged-interval logic as a method, e.g. ReviewState::apply(ReviewRating): ReviewState (pure, returns new state)
SrsScheduler — could just be a static method/policy living on ReviewState, or a separate domain service if you want it swappable later (e.g. to SM-2). For MVP, simplest is a method on ReviewState itself — less ceremony, still pure domain logic with no framework deps.

 
**Application layer**
SubmitReview command: { cardId, rating } (userId implicit/default)

Handler: load user_card_progress row for (userId, cardId) (or create if missing) → call ReviewState::apply(rating) → persist updated row → dispatch CardReviewed event (for Farm later)

GetDueCards query: returns cards where next_review_at <= now() for the default user, joined with cards_translations for the requested language, ordered by next_review_at ASC, capped at review_quantity_cap

---

## Architecture

Two bounded contexts + thin API layer.

```
backend/src/
├── Learning/          # cards, reviews, SRS
│   ├── Domain/
│   ├── Application/   # commands, queries, handlers
│   └── Infrastructure/
├── Farm/              # wallet, plots, crops, rewards
│   ├── Domain/
│   ├── Application/
│   └── Infrastructure/
├── Shared/            # IDs, clock — keep tiny
└── UserInterface/Api/ # thin controllers only

frontend/src/app/
├── core/              # HTTP client, config
├── layout/shell/      # nav: Study | Farm | Cards
├── shared/components/ # hanzi card, rating buttons, resource bar
├── models/
└── features/
    ├── study/
    ├── farm/
    └── vocabulary/
```

**Layer rules (hexagonal):**
- **Domain** — pure PHP/TS logic, no Symfony/Doctrine/HttpClient
- **Application** — use cases; depends on domain + repository *interfaces*
- **Infrastructure** — Doctrine entities/repos, Messenger wiring
- **UserInterface** — HTTP in, DTO out; dispatch commands/queries

**CQRS (Symfony Messenger):**
- **Commands** (write): `AddCard`, `SubmitReview`, `PlantCrop`, `HarvestCrop`
- **Queries** (read): `GetDueCards`, `GetCards`, `GetFarmState`
- **Event**: `CardReviewed` → Farm handler grants rewards (keeps Learning decoupled from Farm)

---

## API checklist

- [ ] `GET  /api/learning/due`
- [ ] `GET  /api/learning/cards`
- [ ] `POST /api/learning/cards`
- [ ] `POST /api/learning/reviews`
- [ ] `GET  /api/farm/state`
- [ ] `POST /api/farm/plots/{plotId}/plant`
- [ ] `POST /api/farm/plots/{plotId}/harvest`

---

## Task checklist

### Week 1 — Learning slice

- [x] Fix PHP version mismatch (`composer.json` wants 8.4, Docker uses 8.4-cli)
- [x] Create backend folder structure (empty dirs + `messenger.yaml` buses: `command.bus`, `query.bus`, `event.bus`)
- [ ] **Learning Domain:** `Card`, `ReviewRating` (`Again` / `KnowIt`), SRS scheduler (interval + due date from rating history)
- [ ] **Application:** `AddCard`, `SubmitReview`, `GetDueCards`, `GetCards` + handlers
- [ ] **Infrastructure:** Doctrine mapping, migration, seed command for starter deck
- [ ] **API:** one controller, wire first endpoint, test with curl/Postman
- [ ] **Angular:** shell + routes, `provideHttpClient`, study feature (due → reveal → rate)

### Week 2 — Farm slice + polish

- [ ] **Farm Domain:** `Wallet`, `Plot`, `Crop`, plant/harvest rules
- [ ] **Application:** `PlantCrop`, `HarvestCrop`, `GetFarmState` + `CardReviewed` event handler
- [ ] **Infrastructure:** migration, default wallet + plots on first run or seed
- [ ] **API:** farm endpoints
- [ ] **Angular:** farm view, vocabulary add-card form
- [ ] Tests: SRS scheduler unit test, one command handler test, one API smoke test
- [ ] Update progress below

---

## Learning checkpoints

After each checkpoint, ask the tutor to review before moving on.

1. **Domain first** — Can you describe `Card` and `SubmitReview` without mentioning HTTP or Doctrine?
2. **One command end-to-end** — `AddCard`: command → handler → repository → DB → API
3. **One query** — `GetDueCards`: query → handler → DTO → JSON response
4. **One event** — `SubmitReview` publishes `CardReviewed`; Farm listens and adds coins
5. **One Angular feature** — Study page calls API, shows card, posts rating

**Questions to ask yourself before each slice:**
- What belongs in Domain vs Application vs Infrastructure?
- Is this a command (mutates) or a query (reads)?
- What interface does the handler need from persistence?

---

## Local dev

```bash
# From repo root — copy env first
cp .env.example .env

docker compose up -d          # postgres :5432, backend :8000, frontend :4200
```

Backend (without Docker): `cd backend && composer install && symfony server:start`  
Frontend: `cd frontend && npm install && npm start`

---

## Progress

_Current step:_ **Learning Domain** — `Card`, `ReviewRating`, `ReviewState`, `SrsScheduler` in `backend/src/Learning/Domain/`.

_Update this line as you go._

---

## How to use the tutor

1. Pick the next unchecked task.
2. Try it yourself first.
3. Ask: *"I'm on step X, here's what I did — does this fit hexagonal/CQRS?"*
4. Request hints, not full implementations: *"What should go in the Domain for SubmitReview?"*
