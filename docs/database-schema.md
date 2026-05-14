# Keepy — Database Schema

## Design Decisions

### Room Type Belongs on Room, Not Game
Each Room is locked to a single game type (Dominoes or Phase Ten). All Games played within a Room inherit that type. This means:
- You never have to validate game type at game-start time — the room enforces it.
- Querying "all domino games" is a simple join through Room.
- The `Game` table does not need a `game_type` column.

### Score and Snapshot Are Separate Tables
A Score can exist without a Snapshot (manual entry). A Snapshot always belongs to a Score attempt. Keeping them separate avoids nullable columns and makes it easy to query "how many scores were ML-generated vs manual."

### Game_Player Is Separate from Room_Member
Room membership and game participation are different things. A player can be in a room but sit out a specific game. `Game_Player` tracks who is actually playing in a given game.

### Composite PKs on Junction Tables
`Room_Member` and `Game_Player` use composite primary keys (both foreign keys together) instead of a surrogate key. This naturally prevents duplicate memberships at the database level.

---

## Tables

### Player
Stores authenticated user accounts.

| Column          | Type         | Constraints              |
|-----------------|--------------|--------------------------|
| player_key      | UUID         | PK                       |
| email           | VARCHAR(255) | UNIQUE, NOT NULL         |
| username        | VARCHAR(50)  | UNIQUE, NOT NULL         |
| nickname        | VARCHAR(50)  |                          |
| password_hash   | VARCHAR(255) | NOT NULL                 |
| created_at      | TIMESTAMP    | NOT NULL, DEFAULT NOW()  |

**Notes:**
- `email` is used for login and password reset.
- `username` is the public-facing identifier shown in games.
- `nickname` is an optional display name override.
- `password_hash` stores a bcrypt hash — never plaintext.

---

### Room
A persistent shared space where games are played. Locked to one game type.

| Column          | Type         | Constraints                        |
|-----------------|--------------|------------------------------------|
| room_key        | UUID         | PK                                 |
| room_name       | VARCHAR(50)  | NOT NULL                           |
| invite_code     | VARCHAR(12)  | UNIQUE, NOT NULL                   |
| game_type       | ENUM         | NOT NULL — ('dominoes','phase_ten')|
| owner_key       | UUID         | FK → Player(player_key), NOT NULL  |
| created_at      | TIMESTAMP    | NOT NULL, DEFAULT NOW()            |

**Notes:**
- `invite_code` is a short, unique, human-readable code (e.g. "XKQT92") used to join the room.
- `game_type` is set at creation and never changes.
- `owner_key` tracks who can delete the room and manage members.

---

### Room_Member
Junction table — tracks which players belong to which rooms.

| Column      | Type      | Constraints                       |
|-------------|-----------|-----------------------------------|
| room_key    | UUID      | FK → Room(room_key), NOT NULL     |
| player_key  | UUID      | FK → Player(player_key), NOT NULL |
| joined_at   | TIMESTAMP | NOT NULL, DEFAULT NOW()           |

**Primary Key:** (room_key, player_key) — composite, prevents duplicate memberships.

**Notes:**
- `joined_at` is used to determine which membership to auto-evict when a player hits the 20-room limit (oldest `joined_at` is removed).
- Deleting a row removes membership but does NOT cascade-delete scores — historical data is preserved.

---

### Game
A single match played within a Room.

| Column       | Type      | Constraints                        |
|--------------|-----------|------------------------------------|
| game_key     | UUID      | PK                                 |
| room_key     | UUID      | FK → Room(room_key), NOT NULL      |
| started_by   | UUID      | FK → Player(player_key), NOT NULL  |
| status       | ENUM      | NOT NULL — ('active','completed')  |
| started_at   | TIMESTAMP | NOT NULL, DEFAULT NOW()            |
| ended_at     | TIMESTAMP | NULL until game is completed       |

**Notes:**
- Game type is not stored here — it is inherited from `Room.game_type`.
- `started_by` is the only player who can end the game.
- `status` drives UI state (active games show live leaderboard; completed games show history view).
- A room can have up to 5 concurrent active games (enforced at the application layer).

---

### Game_Player
Junction table — tracks which players are participating in a specific game.

| Column      | Type | Constraints                        |
|-------------|------|------------------------------------|
| game_key    | UUID | FK → Game(game_key), NOT NULL      |
| player_key  | UUID | FK → Player(player_key), NOT NULL  |

**Primary Key:** (game_key, player_key) — composite.

**Notes:**
- A player must be a Room_Member to be added to a Game_Player row (enforced at application layer).
- Between 2 and 10 players per game (enforced at application layer).

---

### Round
One scoring cycle within a Game.

| Column        | Type      | Constraints                    |
|---------------|-----------|--------------------------------|
| round_key     | UUID      | PK                             |
| game_key      | UUID      | FK → Game(game_key), NOT NULL  |
| round_number  | INT       | NOT NULL                       |
| status        | ENUM      | NOT NULL — ('active','completed') |
| started_at    | TIMESTAMP | NOT NULL, DEFAULT NOW()        |
| ended_at      | TIMESTAMP | NULL until round is completed  |

**Notes:**
- `round_number` is sequential within a game (1, 2, 3…).
- `started_at` is used to enforce the 5-minute auto-advance timeout.
- A round is marked `completed` when all players submit OR the timeout fires.

---

### Score
The point value recorded for a player in a specific round.

| Column        | Type      | Constraints                          |
|---------------|-----------|--------------------------------------|
| score_key     | UUID      | PK                                   |
| round_key     | UUID      | FK → Round(round_key), NOT NULL      |
| player_key    | UUID      | FK → Player(player_key), NOT NULL    |
| score_value   | INT       | NOT NULL, CHECK (score_value >= 0)   |
| is_override   | BOOLEAN   | NOT NULL, DEFAULT FALSE              |
| submitted_at  | TIMESTAMP | NOT NULL, DEFAULT NOW()              |
| updated_at    | TIMESTAMP | NULL — set when a score is edited    |

**Notes:**
- `is_override` is set to TRUE whenever a player manually edits the value (either before or after confirming).
- `submitted_at` is used for tie-breaking on the leaderboard (earlier submission ranks higher).
- `updated_at` is used to enforce the 5-minute free-edit window (compare against `Round.ended_at`).
- One row per (round_key, player_key) — uniqueness enforced at application layer.

---

### Snapshot
A photo submitted by a player for ML-based scoring.

| Column           | Type      | Constraints                       |
|------------------|-----------|-----------------------------------|
| snapshot_key     | UUID      | PK                                |
| score_key        | UUID      | FK → Score(score_key), NOT NULL   |
| image_url        | TEXT      | NOT NULL — path in cloud storage  |
| confidence_score | FLOAT     | NULL until ML returns result      |
| upload_status    | ENUM      | NOT NULL — ('pending','uploaded','failed') |
| captured_at      | TIMESTAMP | NOT NULL, DEFAULT NOW()           |
| uploaded_at      | TIMESTAMP | NULL until successfully uploaded  |

**Notes:**
- `image_url` points to the access-controlled cloud storage bucket.
- `confidence_score` is a value between 0 and 1. Values below 0.80 trigger a low-confidence warning.
- `upload_status` drives the cleanup logic: uploaded snapshots are deleted from device within 24h; failed snapshots are deleted from device after 7 days.
- A Score row can exist with no Snapshot (manual entry path).

---

## Entity Relationship Summary

```
Player ──< Room_Member >── Room
                              │
                           Game ──< Game_Player >── Player
                              │
                           Round
                              │
                           Score ──── Snapshot
                              │
                           Player
```

---

## Indexes to Consider

| Table       | Index On                        | Reason                                      |
|-------------|---------------------------------|---------------------------------------------|
| Room        | invite_code                     | Fast lookup when a player joins via code    |
| Room_Member | player_key                      | Fast query "all rooms for a player"         |
| Room_Member | joined_at                       | Fast eviction of oldest membership         |
| Game        | room_key, status                | Fast query "active games in a room"         |
| Round       | game_key, round_number          | Fast lookup of current round                |
| Score       | round_key, player_key           | Fast leaderboard aggregation                |
| Score       | submitted_at                    | Tie-breaking sort on leaderboard            |
| Snapshot    | upload_status, captured_at      | Cleanup job queries                         |
