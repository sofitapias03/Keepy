# Keepy — Smart Score Tracker

Keepy is a mobile application that eliminates manual score-keeping for tabletop and card games. Players snap a photo of their cards or tiles at the end of each round, and a machine learning algorithm automatically calculates the point value. Scores are tracked in real time, persist across sessions, and are organized into shared Rooms so groups can maintain a full game history.

---

## Problem

Keeping score manually during games like Dominoes or Phase Ten is tedious and error-prone. Counting tiles or cards takes time, scores get lost between sessions, and there is no easy way to look back at past games.

---

## Solution

Keepy solves this with three core ideas:

1. **Camera-based scoring** — At the end of each round, each player photographs their hand. An ML model reads the tiles or cards and calculates the point total automatically. Players can override the result if the model gets it wrong.
2. **Persistent Rooms** — Players create or join Rooms. Every game played in a Room is saved to a Room History, so nothing is lost between sessions.
3. **Live Leaderboard** — Scores update in real time over a WebSocket connection so everyone sees the standings the moment a round ends.

---

## Features

### User Authentication
Players register with an email and password (8–128 characters). Sessions are token-based and valid for 7–30 days. Password reset is supported via a time-limited email link (expires in 1 hour). Google and Apple OAuth sign-in are supported when configured. Accounts are locked for 15 minutes after 5 consecutive failed login attempts.

**Technology:** JWT-based session tokens, bcrypt password hashing, OAuth 2.0 (Google / Apple).

---

### Room Management
A player creates a Room and shares its unique invite code with others. Each player can belong to up to 20 Rooms simultaneously (oldest membership is auto-removed when the limit is exceeded). Room owners can remove members or transfer ownership. Members can leave voluntarily without losing their historical score data.

**Technology:** Server-generated unique invite codes, relational membership table in the Database.

---

### Game Lifecycle
Any Room member can start a game by selecting a game type (Dominoes or Phase Ten) and choosing 2–10 participants. The game tracks rounds until the initiator ends it, at which point the final leaderboard is saved to Room History. Recording a score of 0 for any player who did not submit.

**Technology:** REST API for game state management, WebSocket for real-time round progression.

---

### Camera-Based Score Capture
At the start of each round's scoring phase, players choose between taking a Snapshot with their camera or entering a score manually. The app displays a framing overlay and guidance text to help players capture a clear image. Players can retake the photo before confirming. If the ML service does not respond within 15 seconds, the app falls back to manual entry automatically.

**Technology:** Native device camera API (iOS / Android), image upload to ML_Service backend.

---

### ML-Based Point Recognition
The ML_Service analyzes Snapshots and returns a calculated point total. For Dominoes, it sums all visible pip values. For Phase Ten, it applies the game's official card point values. The model must achieve ≥90% accuracy on well-lit, in-focus images. Results with a confidence score below 80% are flagged as low-confidence, prompting the player to verify manually. Images must be JPEG or PNG, at least 640×480 pixels, and no larger than 10 MB.

**Technology:** Pre-trained or fine-tuned computer vision model (e.g., a CNN or object detection model such as YOLO or EfficientDet) served via a Python-based ML backend (e.g., FastAPI + TensorFlow/PyTorch). The model can be imported from a pre-trained checkpoint or trained from scratch on a labeled dataset of domino tiles and Phase Ten cards.

---

### Score Override
Before confirming a ML-calculated score, players can edit the value directly in a numeric field. After a round ends, players can freely edit their own score within 5 minutes. Edits made after the 5-minute window are applied immediately but trigger a notification to the game initiator showing the player's name, the affected round, and the old and new values. All manually overridden scores are marked with an indicator in the round-by-round breakdown.

**Technology:** Timestamped score records in the Database, server-side edit window enforcement, in-app notification delivery.

---

### Real-Time Leaderboard
The leaderboard updates within 3 seconds of any score being confirmed. It shows each player's cumulative score and rank (ties broken by submission order). A round summary is displayed for at least 3 seconds after all players submit before the next round begins. Pending submission status is also shown in real time.

**Technology:** WebSocket (or equivalent persistent connection) for push-based leaderboard updates.

---

### Room History and Game Records
Every completed and in-progress game is stored in the Room's history, ordered by start date. Completed games show the final leaderboard, the winner(s), and a round-by-round score breakdown with override indicators. In-progress games show the current leaderboard and completed rounds so far. Room History is retained indefinitely unless the Room owner deletes the Room (which requires explicit confirmation and permanently removes all associated data).

**Technology:** Relational Database with cascading delete on Room removal, indexed by Room ID and start date.

---

### Notifications
In-app notifications inform players when their Snapshot has been processed (success, low-confidence warning, or timeout). Notifications appear within 3 seconds of the ML_Service response and remain visible until dismissed. Late score edits also trigger a notification to the game initiator.

**Technology:** In-app notification system (no push notification dependency for MVP).

---

### Multi-Platform Mobile Support
Keepy runs on iOS 15+ and Android 10+ (API level 29+). Camera permissions are requested on first use. If denied, Snapshot capture is disabled and manual entry is the fallback. All core screens are designed to render correctly on devices with screen widths between 360dp and 430dp.

**Technology:** React Native (or Flutter) for cross-platform mobile development.

---

### Data Persistence and Offline Resilience
Confirmed scores are always saved, even without a network connection. The app queues up to 100 scores locally (persisted across app restarts) and syncs them when connectivity returns. If sync fails, the app retries with exponential backoff (up to 5 attempts) before prompting the player to retry manually. An offline banner is shown whenever the device is disconnected.

**Technology:** Local SQLite or AsyncStorage queue, background sync service, exponential backoff retry logic.

---

### Security and Privacy
All API endpoints require a valid session token (HTTP 401 if missing/invalid). Room membership is enforced at the API level — a valid token is not enough; the user must also be a member of the relevant Room (HTTP 403 otherwise). All traffic uses HTTPS with TLS 1.2+. Passwords are stored as salted bcrypt hashes. Locally cached Snapshots are deleted within 24 hours of successful upload, or within 7 days if upload never succeeds. Snapshots in cloud storage are accessible only by the ML_Service and the submitting user.

**Technology:** HTTPS/TLS 1.2+, bcrypt, JWT, access-controlled cloud storage bucket (e.g., AWS S3 with IAM policies or equivalent).

---

## Tech Stack Summary

| Layer | Technology |
|---|---|
| Mobile App | React Native (iOS & Android) |
| Backend API | Node.js / Express or Python / FastAPI |
| Real-Time | WebSocket (Socket.IO or native WS) |
| ML Service | Python (FastAPI + TensorFlow or PyTorch) |
| ML Model | Pre-trained CNN / YOLO / EfficientDet, fine-tuned on domino and card datasets |
| Database | PostgreSQL (relational data) |
| File Storage | Cloud object storage (e.g., AWS S3) with access control |
| Auth | JWT + bcrypt, OAuth 2.0 (Google / Apple) |
| Offline Queue | Local SQLite or AsyncStorage |

---

## Supported Games

| Game | Scoring Method |
|---|---|
| Dominoes | Sum of all pip values on visible tiles |
| Phase Ten | Sum of card point values per Phase Ten rules |

---

## License

See [LICENSE](LICENSE).
