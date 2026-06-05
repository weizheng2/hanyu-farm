# Hanyu Farm

Gamified Chinese learning: study with spaced repetition, earn coins, spend them on a small farm.

**Learning goal:** practice DDD, hexagonal architecture, CQRS, Symfony and Angular.

---

## MVP scope (2 weeks)

**In:**
- Single-player, no auth (one default learner in the backend)
- Seeded beginner deck (~20–40 cards) + minimal add-card form
- Study screen: show due card → reveal answer → rate (`again` / `know it`)
- Simple SRS scheduler: due cards come back; interval grows when you get them on the first try
- Farm screen: earn coins from reviews, plant/harvest crops on a few plots

**Out (for now):**
- Login, multi-user, social
- Import/export, audio, handwriting, advanced quiz modes
- XP / leveling, real-time farm sim, heavy animations, mobile app
- Event sourcing, complex read models

---

## Domain glossary

| Term | Meaning |
|------|---------|
| Card | Hanzi + pinyin + english translation + example phrase usage |
| Review | One rating of a card during study |
| Due card | Card whose next review date is today or earlier |
| Rating | `again` or `know it` — what the learner taps; drives SRS |
| Review state | Per-card counters the scheduler keeps: review count, current interval, next due date |
| Wallet | Coins balance |
| Plot | One slot on the farm (empty, growing, or ready) |
| Crop | Plantable item with cost and grow duration |

**SRS (v1, keep it simple):**
- A card becomes *due* when `nextDueDate <= today`.
- Study shows the prompt only; learner reveals the answer, then rates.
- **`know it`** (first try, no `again` in this session) → lengthen interval (show less often).
- **`again`** → reset or shorten interval; card stays in the session queue until `know it`.
- Track review count + interval on the card; no `hard` / `good` / `easy` buttons in the UI for now.

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
- [ ] Create backend folder structure (empty dirs + `messenger.yaml` buses)
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

## Decisions log

| Decision | Choice | Why |
|----------|--------|-----|
| Auth | None in v1 | Focus on DDD/CQRS, not security |
| Content | Seeded deck + add-card | Enough to learn without building a full CMS |
| Messenger | Sync buses | Simpler debugging; async later |
| SRS | Binary rating + interval growth | Simple UX (`again` / `know it`); scheduler uses review count and first-try success; SM-2 optional later |

---

## Progress

_Current step:_ **Create backend folder structure** (Learning, Farm, Shared, UserInterface/Api).

_Update this line as you go._

---

## How to use the tutor

1. Pick the next unchecked task.
2. Try it yourself first.
3. Ask: *"I'm on step X, here's what I did — does this fit hexagonal/CQRS?"*
4. Request hints, not full implementations: *"What should go in the Domain for SubmitReview?"*
