# Friendship Analyzer — Project Document

## Overview

Friendship Analyzer is a personal, local-first application for systematically evaluating the depth and quality of your relationships. It combines structured self-assessment with automated analysis of real interaction data to produce an honest, data-driven picture of your social network. The goal is to move beyond vague intuition and identify which relationships are genuinely mutual, which are acquaintances, and where the meaningful connections in your life actually are.

---

## Motivation

Most people carry an inaccurate mental model of their friendships—overestimating some, undervaluing others, and pattern-matching on recent or emotionally salient experiences rather than actual behavior over time. This tool applies a more systematic framework:

- Rate people across multiple dimensions of friendship quality
- Ingest real interaction data to supplement or validate subjective ratings
- Use dimensionality reduction (PCA) in parallel with the manual dimension structure to surface latent patterns
- Track how relationships evolve over time, not just their current state
- Visualize your social network as a radar chart or scatter plot rather than a flat list

The scoring framework is intentionally exploratory — weights, thresholds, and dimensions are expected to be revised as more people are rated and intuitions about closeness become clearer.

---

## Core Concepts

### Friendship Dimensions

All contacts — regardless of context — are rated on the same five dimensions. Context-specific scoring profiles adjust how each dimension is weighted, rather than using different question sets.

**Mutuality** — Do they initiate? Do they show up when needed? Do they remember things about your life?

**Authenticity** — Can you be honest with them? Do they push back? Do you feel like yourself around them?

**Investment** — Do they know what matters to you? Have they sacrificed time or convenience for you? Would they notice if you went quiet?

**Consistency** — Is the relationship consistent or only convenient? Does it pick up naturally after time apart? Do you feel better or drained after spending time with them?

**Gut Check** — Would you call them with genuinely bad news? Would they describe you as a friend if asked?

### Scoring

- Each question rated 1–5
- Dimension scores are averages of constituent questions
- Overall score is a weighted L2 norm across dimensions
- Weights and thresholds vary by context (see Scoring Profiles)
- Some questions serve as threshold criteria — a very low score can cap classification regardless of overall score
- Ratings are versioned over time — each rating session is timestamped and preserved, never overwritten

### Scoring Profiles

The same questions are used for all contexts. Weights are adjusted per context to reflect the different nature of each relationship type. These weights are initial defaults and should be revised as more people are rated and intuitions sharpen.

| Dimension | Social | Family | Professional |
|---|---|---|---|
| Mutuality | 1.5 | 0.8 | 1.2 |
| Authenticity | 1.3 | 1.3 | 0.7 |
| Investment | 1.2 | 1.5 | 1.0 |
| Consistency | 1.0 | 1.5 | 1.2 |
| Gut Check | 1.5 | 1.2 | 0.5 |

Gut Check is downweighted for professional contacts since the work context structurally suppresses it. Mutuality is downweighted for family since low initiation frequency doesn't necessarily indicate low closeness.

### Classification

| Score Range | Context | Classification |
|---|---|---|
| High across all dimensions | Social | Close Friend |
| High mutuality + authenticity, moderate elsewhere | Social | Good Friend |
| Moderate overall, low gut check | Social | Casual Friend |
| Low mutuality, low investment | Social | Acquaintance |
| High investment + consistency, high authenticity | Family | Close Family |
| Moderate investment, low mutuality + gut check | Family | Extended Family |
| Low overall despite shared history | Family | Obligatory Family |
| High investment + consistency, low authenticity + gut check | Professional | Trusted Colleague |
| Moderate across dimensions, context-bound | Professional | Professional Contact |
| Low overall, purely transactional | Professional | Work Acquaintance |

### PCA Analysis — Parallel Layer

PCA runs alongside the manual dimension structure as an analytical layer, not a replacement. The manual dimensions are the working classification framework; PCA surfaces latent patterns that the manual structure may not capture.

PCA is run per context group to avoid conflating different relationship dynamics. Over time, divergence between PCA components and manual dimensions is a signal to revisit the question set or weights.

Future direction: running PCA on rating snapshots over time to detect whether the latent dimensions themselves shift as your social world changes.

---

## Friendship Evolution Over Time

A core feature of the application is tracking how relationships change, not just classifying their current state.

### Design Principles

- Ratings are never overwritten — each rating session creates a new versioned snapshot
- Interaction-derived metrics are computed over configurable time windows (30-day, 90-day, 1-year, all-time)
- A per-person timeline view shows how dimension scores and overall classification have shifted across rating sessions
- Decay is a valid and meaningful signal — a friendship that has cooled is different from one that was always shallow

### What This Enables

- Identifying relationships that are trending positive or negative
- Correlating changes in ratings with changes in interaction frequency
- Reviewing your own scoring calibration over time — did your weights or intuitions shift?
- Eventually: running PCA on rating snapshots over time to detect structural shifts in your social world

---

## Architecture

### Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.11+ with FastAPI |
| Database | SQLite via SQLAlchemy ORM |
| Data Analysis | pandas, scikit-learn (PCA), numpy |
| Frontend | HTML + vanilla JS |
| Data Ingestion | Pluggable connector architecture |

### Why This Stack

- **FastAPI** — async-native, clean API design, auto-generates OpenAPI docs
- **SQLite** — single file, zero server overhead, easy to back up, sufficient for personal use
- **Local-first** — all data stays on your machine; nothing is sent to any external service
- **Pluggable connectors** — interaction history from multiple sources feeds into a unified model without tight coupling

---

## Database Schema

```sql
CREATE TABLE people (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    context TEXT NOT NULL DEFAULT 'social',  -- 'social' | 'family' | 'professional'
    is_family BOOLEAN DEFAULT FALSE,
    notes TEXT,
    source TEXT,
    external_id TEXT,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

CREATE TABLE questions (
    id INTEGER PRIMARY KEY,
    dimension TEXT NOT NULL,
    text TEXT NOT NULL,
    active BOOLEAN DEFAULT TRUE
);

-- Ratings are versioned — never overwritten
CREATE TABLE ratings (
    id INTEGER PRIMARY KEY,
    person_id INTEGER REFERENCES people(id),
    question_id INTEGER REFERENCES questions(id),
    score INTEGER CHECK(score BETWEEN 1 AND 5),
    session_id INTEGER REFERENCES rating_sessions(id),
    notes TEXT
);

-- A rating session groups all question scores for a person at a point in time
CREATE TABLE rating_sessions (
    id INTEGER PRIMARY KEY,
    person_id INTEGER REFERENCES people(id),
    rated_at TIMESTAMP NOT NULL,
    notes TEXT
);

CREATE TABLE interactions (
    id INTEGER PRIMARY KEY,
    person_id INTEGER REFERENCES people(id),
    source TEXT NOT NULL,     -- 'imessage', 'gmail', 'whatsapp', etc.
    direction TEXT,           -- 'inbound' | 'outbound'
    channel TEXT,             -- 'message' | 'email' | 'call'
    occurred_at TIMESTAMP,
    metadata JSON
);

-- Computed scores are versioned by session and time window
CREATE TABLE computed_scores (
    id INTEGER PRIMARY KEY,
    person_id INTEGER REFERENCES people(id),
    session_id INTEGER REFERENCES rating_sessions(id),
    dimension TEXT,
    score REAL,
    context TEXT,
    window TEXT,              -- '30d' | '90d' | '1y' | 'all_time'
    computed_at TIMESTAMP
);

-- Context-specific scoring weight profiles — tunable over time
CREATE TABLE scoring_profiles (
    id INTEGER PRIMARY KEY,
    context TEXT NOT NULL,    -- 'social' | 'family' | 'professional'
    dimension TEXT NOT NULL,
    weight REAL DEFAULT 1.0,
    is_threshold BOOLEAN DEFAULT FALSE,
    effective_from TIMESTAMP NOT NULL,
    UNIQUE(context, dimension, effective_from)
);
```

---

## Pluggable Data Source Architecture

Data sources implement a common connector interface. New sources can be added without modifying core logic.

### Connector Interface

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime
from typing import Iterator, Literal

Context = Literal["social", "family", "professional"]

@dataclass
class Contact:
    name: str
    external_id: str
    source: str
    context: Context = "social"
    is_family: bool = False
    metadata: dict = field(default_factory=dict)

@dataclass
class Interaction:
    external_person_id: str
    source: str
    direction: str        # 'inbound' | 'outbound'
    channel: str          # 'message' | 'email' | 'call'
    occurred_at: datetime
    metadata: dict = field(default_factory=dict)

class DataSourceConnector(ABC):

    @abstractmethod
    def get_contacts(self) -> Iterator[Contact]: ...

    @abstractmethod
    def get_interactions(self, since: datetime = None) -> Iterator[Interaction]: ...

    @abstractmethod
    def health_check(self) -> bool: ...
```

### Connector Registry

```python
CONNECTOR_REGISTRY = {
    "google_contacts": GoogleContactsConnector,
    "imessage": IMessageConnector,
    "gmail": GmailConnector,
    "whatsapp": WhatsAppConnector,
    "csv": CSVConnector,
}
```

### Planned Connectors

| Connector | Method | Notes |
|---|---|---|
| Google Contacts | Contacts API or vCard export | Seeds people list; can infer context from contact groups |
| iMessage | Local SQLite at `~/Library/Messages/chat.db` | Mac only; no API needed |
| Gmail | Gmail API (OAuth2) | Requires Google Cloud project setup |
| WhatsApp | Chat export (.txt parsing) | Manual export required |
| SMS (Android) | SMS Backup & Restore XML | Manual export required |
| CSV/Manual | File upload | Fallback for any source |

---

## Derived Metrics from Interaction Data

Computed over configurable time windows: 30-day, 90-day, 1-year, all-time.

| Metric | Maps To | How Computed |
|---|---|---|
| Initiation ratio | Mutuality | % of conversations started by them vs. you |
| Response latency | Investment | Average time to respond to your messages |
| Interaction frequency | Consistency | Messages/interactions per month over time |
| Recency | Consistency | Days since last interaction |
| Channel diversity | Depth | Do you interact across multiple channels? |
| Conversation length | Authenticity (proxy) | Average message count per thread |

---

## Scoring Logic

```python
def compute_score(person, ratings, profiles, window="all_time"):
    profile = get_active_profile(profiles, person.context)

    dimension_scores = {}
    for dimension, questions in grouped_by_dimension(ratings):
        raw = mean(q.score for q in questions)
        weight = profile[dimension].weight

        # Threshold breach caps classification regardless of overall score
        if profile[dimension].is_threshold and raw < THRESHOLD_MIN:
            return Classification.DOWNGRADE

        dimension_scores[dimension] = raw * weight

    overall = l2_norm(dimension_scores.values())
    return classify(overall, person.context)
```

---

## API Design

**People**
- `GET /people` — list all people with latest computed scores, filterable by context
- `POST /people` — add a person manually
- `GET /people/{id}` — get person with full rating history and interaction summary
- `DELETE /people/{id}` — remove a person

**Ratings**
- `GET /people/{id}/ratings` — get all rating sessions for a person
- `POST /people/{id}/ratings` — create a new rating session
- `GET /questions` — list all active questions by dimension

**Sources**
- `GET /sources` — list configured connectors and their status
- `POST /sources/{source}/sync` — trigger a sync for a specific connector
- `GET /sources/{source}/preview` — preview what would be imported

**Analysis**
- `GET /analysis/pca?context={context}` — run PCA for a context group, return component loadings and person coordinates
- `GET /analysis/scores?window={window}` — return computed scores for all people at a given time window
- `GET /analysis/interaction-stats/{id}?window={window}` — interaction frequency, initiation ratio, recency
- `GET /analysis/evolution/{id}` — time series of dimension scores and classification across rating sessions

**Scoring Profiles**
- `GET /profiles` — list all scoring profiles with effective dates
- `POST /profiles` — create a new profile version (preserves history)

---

## Frontend Views

1. **Dashboard** — all people sorted by score, filterable by context and classification, showing trend indicators
2. **Rate a Person** — guided questionnaire across all dimensions, adapted to context, creates a new versioned session
3. **Person Detail** — radar chart of latest dimension scores + interaction timeline + derived metrics
4. **Evolution View** — per-person line chart of dimension scores and overall classification over time, overlaid with interaction frequency
5. **Network View** — scatter plot using first two PCA components, colored by context and classification
6. **PCA Explorer** — component loadings per context group, showing which questions drive each dimension; comparison between manual structure and PCA components
7. **Source Management** — configure, test, and sync data source connectors
8. **Profile Editor** — view and adjust scoring weights per context; changes are versioned with effective dates

---

## Privacy Considerations

- All data is stored locally in a single SQLite file
- No telemetry, no external API calls except those explicitly configured by the user
- The database file can optionally be encrypted at rest using SQLCipher
- Google API credentials (if used) are stored locally and never transmitted elsewhere

---

## Phased Implementation Plan

**Phase 1 — Core Rating System**
SQLite schema with versioned ratings and rating sessions, FastAPI backend, question/dimension model, context-aware person entry, basic scoring and classification with configurable profiles.

**Phase 2 — Analysis**
PCA implementation per context group running in parallel with manual dimensions, radar charts, scatter plot visualization, weighted scoring and threshold logic, profile configuration UI.

**Phase 3 — Evolution Tracking**
Rating session history, per-person evolution timeline view, interaction frequency overlaid with score changes, configurable time windows for interaction-derived metrics.

**Phase 4 — Data Ingestion**
CSV connector (baseline), iMessage connector (local SQLite, Mac), Google Contacts connector (vCard export first, API second), interaction-derived metrics feeding into scores.

**Phase 5 — Extended Sources + Advanced Analysis**
Gmail connector, WhatsApp export parser, Android SMS export parser, automated score suggestions from interaction data, PCA over rating snapshots to detect structural shifts over time.

---

## Open Questions

- How should blended relationships be handled — e.g., a family member who is also a close friend, or a colleague who has become a genuine friend? Should a person support multiple contexts simultaneously?
- For professional contacts, should interaction data from work email be weighted differently than personal channels?
- What should trigger a prompt to re-rate someone — elapsed time, a significant change in interaction frequency, or manual only?
- How should scoring profile changes be handled retroactively — should historical computed scores be recalculated under new weights, or preserved as-is?
