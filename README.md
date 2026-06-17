# Calendar-Text-to-Picto-Translation

A hybrid pipeline that translates calendar event data into [ARASAAC](https://arasaac.org) pictograms. Built as a full-stack application with an **Angular** frontend, a **Python / FastAPI** backend, and **MongoDB** for pictogram storage.

The system follows the tiered architecture described in the AAC research literature: a curated keyword-matching core ensures deterministic, reliable output for accessibility-critical users, extended by stemming and WordNet synset traversal for broader coverage.

---

## Architecture

```
┌────────────┐        ┌────────────────┐        ┌──────────┐
│  Angular   │──────▶│  FastAPI        │──────▶│ MongoDB  │
│  Frontend  │ REST   │  Backend       │ query  │          │
│  (Nginx)   │◀──────│  /api/translate │◀──────│ 25 000+  │
│  :3000     │  JSON  │  :8080         │        │pictograms│
└────────────┘        └────────────────┘        └──────────┘
                                                   ▲
                                                   │ seed
                                              ┌────┴───────┐
                                              │ Data Loader │
                                              │ (Python)    │
                                              └─────────────┘
                                                   ▲
                                                   │ HTTP
                                              ARASAAC API
```

| Service        | Technology              | Port  | Purpose                                       |
|----------------|-------------------------|-------|-----------------------------------------------|
| `mongodb`      | MongoDB 7               | 27017 | Stores all ARASAAC pictogram records          |
| `data-loader`  | Python 3.12             | —     | Fetches pictograms from ARASAAC API → MongoDB |
| `backend`      | Python 3.12, FastAPI    | 8080  | Translation pipeline                          |
| `frontend`     | Angular 19              | 3000  | Calendar event input & pictogram display      |

## Translation Pipeline

The backend implements a five-tier hybrid matching pipeline. Each input field is tokenized and processed through the tiers in order until a match is found:

**Tier 1 — Multi-word sliding window (exact match)**
Starting from each token position, the pipeline tries the longest possible phrase first, then shrinks the window. This handles ARASAAC multi-word keywords like *"an einem Seil gehen"* as a single pictogram match.

**Tier 2 — Exact single-token match**
Each remaining token is checked against the pre-built index of lowercased keywords, plurals, tags, and categories from the ARASAAC database.

**Tier 3 — Stemmed match**
The token is stemmed using the NLTK Snowball German stemmer, then matched against a stem index built from all ARASAAC keywords. This handles inflections like *"Hunde"* → *"hund"* → pictogram for *"Hund"*.

**Tier 4 — Synset match (WordNet hypernym traversal)**
The token's synsets are looked up via the ARASAAC keyword-to-synset mapping, then WordNet hypernyms are traversed up to 4 levels. This allows *"Lachs"* (salmon) to match the pictogram for *"Fisch"* (fish) via the hypernym chain.

**Tier 5 — Unmatched**
Tokens that fail all tiers are flagged in the response as `unmatchedTokens`.

Each match includes a `matchType` field (`EXACT`, `STEMMED`, or `SYNSET`) so the frontend can indicate how the pictogram was found.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and [Docker Compose](https://docs.docker.com/compose/install/)
- Internet connection (first run fetches ~25 000 pictograms from the ARASAAC API)

## Quick Start

```bash
# Clone the repository
git clone https://github.com/flohoch/Calendar-Text-to-Picto-Translation.git
cd Calendar-Text-to-Picto-Translation

# Build and start all services
docker compose up --build

# Wait for the data-loader to finish (watch its logs):
#   "Inserting 25xxx pictograms into MongoDB ..."
#   "Data loading complete."

# Open the frontend
open http://localhost:3000
```

The data loader runs once and exits. On subsequent starts, it detects the existing collection and skips the import.

## Usage

1. Open **http://localhost:3000** in your browser.
2. Enter calendar event details in the three fields:
    - **Summary** — what the event is (e.g., `Essen Frühstück`)
    - **Location** — where it takes place (e.g., `Schule`)
    - **Participants** — who is involved (e.g., `Mutter Vater`)
3. Click **Translate to Pictograms**.
4. Each field shows matched pictograms with images and labels, a badge indicating the match type, plus any tokens that couldn't be matched.

## API Reference

### `POST /api/translate`

Translate a calendar event into pictograms.

**Request body:**
```json
{
  "summary": "Essen Frühstück",
  "location": "Schule",
  "participants": "Mutter Vater"
}
```

**Response:**
```json
{
  "summary": {
    "originalText": "Essen Frühstück",
    "matches": [
      {
        "matchedTerm": "essen",
        "pictogramId": 6456,
        "imageUrl": "https://static.arasaac.org/pictograms/6456/6456_500.png",
        "keywords": [...],
        "matchType": "EXACT"
      }
    ],
    "unmatchedTokens": []
  },
  "location": { ... },
  "participants": { ... }
}
```

### `GET /api/pictograms/{id}`

Retrieve a single pictogram by its ARASAAC ID.

### `GET /api/status`

Health check returning `{ "status": "ok", "pictogramCount": 25000 }`.

## Configuration

| Environment Variable        | Default                                | Description                        |
|-----------------------------|----------------------------------------|------------------------------------|
| `SPRING_DATA_MONGODB_URI`   | `mongodb://localhost:27017/pictograms` | MongoDB connection string (backend)|
| `MONGO_URI`                 | `mongodb://localhost:27017`            | MongoDB connection string (loader) |
| `MONGO_DB`                  | `pictograms`                           | Database name (loader)             |
| `ARASAAC_LANGUAGE`          | `de`                                   | Language code for ARASAAC API      |

To change the pictogram language, set `ARASAAC_LANGUAGE` in `docker-compose.yml` (e.g., `en`, `es`, `fr`).

## Development

### Run services individually

```bash
# Start MongoDB
docker compose up mongodb -d

# Run data loader
docker compose up data-loader

# Backend (requires Python 3.12+)
cd backend
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8080

# Frontend (requires Node 20+)
cd frontend
npm install
ng serve --proxy-config proxy.conf.json    # → http://localhost:4200 with API proxy to :8080
```

### Re-seed the database

```bash
# Drop the collection and re-import
docker compose exec mongodb mongosh pictograms --eval "db.pictograms.drop()"
docker compose up data-loader
```

## Project Structure

```
Calendar-Text-to-Picto-Translation/
├── docker-compose.yml
├── data-loader/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── load_pictograms.py            # ARASAAC API → MongoDB
├── backend/
│   ├── Dockerfile
│   ├── requirements.txt
│   └── app/
│       ├── main.py                    # FastAPI app with startup lifecycle
│       ├── models/
│       │   └── schemas.py             # Pydantic models (request/response/DB)
│       ├── routers/
│       │   └── api.py                 # POST /api/translate, GET /api/status
│       └── services/
│           ├── database.py            # PyMongo connection
│           ├── stemmer.py             # NLTK Snowball German stemmer
│           ├── synset_service.py      # WordNet hypernym traversal
│           ├── index_service.py       # In-memory indices built at startup
│           └── translation_service.py # 5-tier matching pipeline
└── frontend/
    ├── Dockerfile
    ├── nginx.conf
    ├── angular.json
    ├── tsconfig.json
    ├── tsconfig.app.json
    ├── package.json
    ├── proxy.conf.json
    └── src/
        ├── main.ts
        ├── index.html
        ├── styles.css
        └── app/
            ├── app.component.ts
            ├── app.component.html
            ├── app.component.css
            ├── app.config.ts
            ├── models/
            │   └── translation.model.ts
            ├── services/
            │   └── translation.service.ts
            └── components/
                ├── event-field-input/
                │   └── event-field-input.component.ts
                └── pictogram-results/
                    └── pictogram-results.component.ts
```

## License & Attribution

Pictograms are property of the Government of Aragón and are created by Sergio Palao for [ARASAAC](https://arasaac.org), distributed under the [Creative Commons BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) license.

Application code is licensed under the [MIT License](LICENSE). 
