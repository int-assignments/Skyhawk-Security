# skyhawk-security

# Skyhawk Security — NBA Statistics Service

A **high-availability, horizontally-scalable** micro-service that ingests NBA box-score data and serves season-average statistics for players and teams.

---


## Objectives

* **Ingest** per-game stat lines for every NBA player.
* **Compute & expose** season-average stats:

  * Per *player*
  * Per *team*
* **Deliver** rock-solid, maintainable code in **Golang *or* Java** (no Spring/ORMs).
* **Package** via Docker Compose *or* Minikube for one-command startup.

---

## Functional Scope

| Capability                     | Endpoint                                | Consumer      |
| ------------------------------ | --------------------------------------- | ------------- |
| **Log player stats**           | `POST /games`                           | Machine / ETL |
| **Season averages per player** | `GET /stats/player/{season}/{playerId}` | Human / UI    |
| **Season averages per team**   | `GET /stats/team/{season}/{teamId}`     | Human / UI    |

---

## Non-Functional Requirements

* **Throughput** Handle bursts of *tens → hundreds* of concurrent requests.
* **Freshness** Read-after-write consistency; new data is instantly queryable.
* **Availability & Scalability** No single points of failure; stateless services scale horizontally.
* **Maintainability** Clean layering, clear boundaries, strong test coverage.

---

## High-Level Design

### Core Components

| # | Component                       | Responsibility                                   | Scalability Notes                                   |
| - | ------------------------------- | ------------------------------------------------ | --------------------------------------------------- |
| 1 | **API Gateway / Load Balancer** | Routes all HTTP traffic.                         | Stateless—add replicas behind ELB / Nginx.          |
| 2 | **Write Service**               | Validates and stores each game’s stat line.      | Stateless; horizontal scaling handles write bursts. |
| 3 | **Read Service**                | Executes aggregate SQL queries for season stats. | Stateless; scales independently of writes.          |
| 4 | **PostgreSQL**                  | Single source of truth (`game_stats` table).     | Can evolve to master + read-replicas.               |

### Data Flow (plain English)

1. **Ingestion Path**
   *Producer →* **`POST /games`** → Gateway → **Write Service** → *INSERT →* **DB**.
2. **Query Path**
   *Browser / BI Tool →* **`GET /stats/…`** → Gateway → **Read Service** → *SELECT + AVG →* **DB** → Response.

---

## API Specification

### 1 · Log Player Stats

```
POST /games
Content-Type: application/json
```

```jsonc
{
  "gameId"   : "2024-11-15-LAL-DEN",
  "playerId" : "201939",
  "teamId"   : "GSW",
  "season"   : "2024-2025",
  "points"   : 34,
  "rebounds" : 7,
  "assists"  : 12,
  "steals"   : 2,
  "blocks"   : 1,
  "fouls"    : 2,
  "turnovers": 4,
  "minutes"  : 36.5
}
```

*`201 Created`* on success   *`400 Bad Request`* on validation failure.

---

### 2 · Season Averages

```
GET /stats/player/{season}/{playerId}
GET /stats/team/{season}/{teamId}
```

**Response**

```jsonc
{
  "season"  : "2024-2025",
  "entityId": "201939",
  "games"   : 72,
  "avg": {
    "points"   : 28.3,
    "rebounds" : 6.1,
    "assists"  : 7.8,
    "steals"   : 1.4,
    "blocks"   : 0.5,
    "fouls"    : 2.3,
    "turnovers": 3.1,
    "minutes"  : 34.2
  }
}
```

---

## Data Model

### Table `game_stats`

| Column               | Type       | Constraints     |
| -------------------- | ---------- | --------------- |
| `id`                 | UUID       | PK              |
| `game_id`            | VARCHAR    | nullable        |
| `season`             | CHAR(9)    | `YYYY-YYYY`     |
| `player_id`          | VARCHAR    | FK → `players`  |
| `team_id`            | VARCHAR    | FK → `teams`    |
| `points` … `minutes` | INT / REAL | see rules below |
| `created_at`         | TIMESTAMP  | default = NOW() |

#### Indexes

```sql
CREATE INDEX idx_player_season ON game_stats(season, player_id);
CREATE INDEX idx_team_season   ON game_stats(season, team_id);
```

---

## Validation Rules

* `points`, `rebounds`, `assists`, `steals`, `blocks`, `turnovers` ≥ 0 (integers)
* `fouls` ∈ 0-6 (integer)
* `minutes` ∈ 0.0-48.0 (float)

Invalid payloads never reach the database.

---

## Running the Project

```bash
# 1 · Clone
git clone https://github.com/<your-handle>/skyhawk-nba-stats.git
cd skyhawk-nba-stats

# 2 · Start (Docker Compose example)
docker-compose up --build

# 3 · Verify
curl -X POST http://localhost:8080/games -d @sample.json -H "Content-Type: application/json"
curl http://localhost:8080/stats/player/2024-2025/201939
```

*(Minikube users: run `make k8s-up` instead.)*

---

## Testing

```bash
# Unit + integration
make test

# Simple load test (≈500 concurrent writes + reads)
make load-test
```

---

## Design Decisions

| Area             | Choice                                | Rationale                                            |
| ---------------- | ------------------------------------- | ---------------------------------------------------- |
| **Language**     | *Go (or plain Java)*                  | Small binary, easy concurrency, no heavy frameworks. |
| **Database**     | PostgreSQL                            | ACID, strong SQL, ready Docker image.                |
| **Architecture** | Split write/read services (CQRS-lite) | Isolates concerns; scale independently.              |
| **Consistency**  | Single primary DB                     | Guarantees immediate read-after-write without cache. |
| **Deployment**   | Docker Compose / Minikube             | One-command local run; mirrors AWS ECS/EKS layout.   |
| **Validation**   | Hand-rolled structs                   | Keeps binary light; avoids reflection/ORM overhead.  |

---

## Future Work

1. Promote Postgres read-replicas + connection-pooling for even higher read QPS.
2. Add Redis cache for most-requested player stats.
3. Emit Kafka CDC events for downstream analytics.
4. Integrate Prometheus + Grafana dashboards.
5. Harden CI/CD: GitHub Actions → security scan → push to ECR.

---

## Submission

1. Push the code to a **public GitHub repo**.
2. Grant access to ****.
3. Ensure `docker-compose up` (or `make k8s-up`) works out-of-the-box.

Happy coding!
