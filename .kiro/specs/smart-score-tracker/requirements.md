# Requirements Document

## Introduction

Keepy is a smart score tracking mobile application designed for tabletop and card games such as Dominoes and Phase Ten. Instead of manually counting and recording scores, players use their device camera to photograph their cards or tiles at the end of each round. A machine learning algorithm analyzes the image and automatically calculates the point value, adding it to the player's running total. Games are organized into Rooms, which persist across sessions so players can resume or review past games. A single player can belong to multiple rooms, and each room maintains a history of all games played within it.

---

## Glossary

- **App**: The Keepy mobile application (iOS and Android).
- **User**: An authenticated individual using the App.
- **Room**: A persistent shared space created by a User where one or more Games are played. A Room has a unique invite code and a collection of Members.
- **Member**: A User who has joined a Room.
- **Game**: A single match of a supported game type (e.g., Dominoes, Phase Ten) started within a Room.
- **Round**: One scoring cycle within a Game. Each Round ends when all Players submit their score for that cycle.
- **Player**: A Member who is participating in a specific Game.
- **Score**: The numeric point value assigned to a Player for a given Round.
- **Snapshot**: A photograph taken by a Player of their cards or tiles at the end of a Round, submitted for ML-based scoring.
- **ML_Service**: The backend machine learning service responsible for analyzing Snapshots and returning a calculated point value.
- **Score_Override**: A manual correction a Player applies to a Score calculated by the ML_Service.
- **Leaderboard**: A ranked display of Players' cumulative scores within a Game.
- **Room_History**: The collection of all Games ever started within a Room.
- **Auth_Service**: The backend service responsible for user registration, login, and session management.
- **API**: The backend REST/WebSocket API that mediates communication between the App and backend services.
- **Database**: The persistent data store for Users, Rooms, Games, Rounds, and Scores.
- **Notification**: An in-app message displayed to a User to inform them of ML scoring results.

---

## Requirements

### Requirement 1: User Authentication

**User Story:** As a new user, I want to register and log in securely, so that my rooms, games, and scores are tied to my identity and persist across sessions.

#### Acceptance Criteria

1. THE Auth_Service SHALL support user registration with a unique email address and a password of at least 8 characters and no more than 128 characters.
2. WHEN a User submits valid registration credentials, THE Auth_Service SHALL create a new User account and return a session token.
3. IF a User submits a registration request with an email address already associated with an existing account, THEN THE Auth_Service SHALL return an error message indicating the email is already in use.
4. WHEN a User submits valid login credentials, THE Auth_Service SHALL return a session token valid for at least 7 days and no more than 30 days.
5. IF a User submits invalid login credentials, THEN THE Auth_Service SHALL return an error message, SHALL NOT return a session token, and SHALL NOT grant any form of authentication success.
6. IF a User submits 5 consecutive invalid login attempts for the same account, THEN THE Auth_Service SHALL lock that account for 15 minutes and SHALL reject all further login attempts for that account during the lockout period.
7. IF a User's session token has expired, THEN THE Auth_Service SHALL require the User to log in again before accessing protected resources.
8. WHEN the Auth_Service receives a password reset request for a registered email address, THE Auth_Service SHALL send a single-use reset link to that address that expires after 1 hour, and SHALL invalidate the link immediately after it is used.
9. WHERE a third-party OAuth provider (Google or Apple) is configured, THE Auth_Service SHALL allow Users to register and log in using that provider.
10. IF no third-party OAuth provider is configured, THEN THE Auth_Service SHALL reject OAuth login attempts and return an error indicating the provider is unavailable.

---

### Requirement 2: Room Management

**User Story:** As a player, I want to create and join rooms, so that I can organize games with specific groups of people and keep a shared history.

#### Acceptance Criteria

1. WHEN an authenticated User requests to create a Room, THE App SHALL create a Room with a unique invite code and designate that User as the Room owner.
2. WHEN a Room owner sets a Room name, THE App SHALL accept names between 1 and 50 characters and reject names outside that range with an error message.
3. WHEN an authenticated User submits a valid Room invite code for a Room they are not already a Member of, THE App SHALL add that User as a Member of the corresponding Room.
4. IF a User submits an invite code that does not correspond to any existing Room, THEN THE App SHALL display an error message indicating the code is invalid.
5. IF a User submits a valid Room invite code for a Room they are already a Member of, THEN THE App SHALL display an error message indicating they are already a Member of that Room and SHALL NOT add a duplicate membership.
6. THE App SHALL allow a single User to be a Member of up to 20 Rooms simultaneously.
7. WHEN an authenticated User who is already a Member of 20 Rooms submits a valid Room invite code, THE App SHALL add the User as a Member of the new Room and automatically remove the User from the Room they joined least recently.
8. WHEN a Member views a Room, THE App SHALL display the Room name, the list of Members, and the Room_History.
9. WHEN a Room owner removes a Member from a Room, THE App SHALL revoke that Member's access to the Room and its Games and redirect the removed Member away from any Room or Game view with an error message indicating their access has been revoked.
10. WHEN a Member who is not the Room owner requests to leave a Room, THE App SHALL remove that Member's membership without deleting their historical score data.
11. IF a Room owner attempts to leave a Room, THEN THE App SHALL reject the leave action and display a message requiring the owner to transfer ownership to another Member before leaving.

---

### Requirement 3: Game Lifecycle

**User Story:** As a room member, I want to start, play, and end games within a room, so that each match is tracked separately and contributes to the room's history.

#### Acceptance Criteria

1. WHEN a Room Member starts a new Game, THE App SHALL prompt the Member to select a supported game type (Dominoes or Phase Ten) and select which Room Members will participate as Players.
2. IF a Room Member attempts to start a Game with fewer than 2 or more than 10 Players selected, THEN THE App SHALL reject the request and display an error message indicating the valid Player count range.
3. WHEN a Game is started, THE App SHALL initialize a Leaderboard with all Players at a score of 0.
4. WHEN all Players in a Round have submitted their Scores, THE App SHALL advance the Game to the next Round and update the Leaderboard.
5. WHEN a Round has been active for 5 minutes and at least one Player has not submitted a Score, THE App SHALL advance the Game to the next Round, record a Score of 0 for each Player who did not submit, and update the Leaderboard.
6. WHEN a Player who started a Game requests to end it, THE App SHALL mark the Game as completed, record the final Leaderboard, and add the Game to the Room_History.
7. IF a Player who did not start a Game requests to end it, THEN THE App SHALL reject the request and display an error message indicating only the Game initiator can end the Game.
8. WHILE a Game is active, THE App SHALL update the displayed Leaderboard within 3 seconds of any Player's Round Score being confirmed.
9. WHEN a completed Game is viewed from the Room_History, THE App SHALL display the final Leaderboard and a round-by-round score breakdown for each Player.
10. THE App SHALL support up to 5 concurrent active Games within the same Room.

---

### Requirement 4: Camera-Based Score Capture

**User Story:** As a player, I want to photograph my cards or tiles at the end of a round, so that the app can automatically calculate my score without manual counting.

#### Acceptance Criteria

1. WHEN a Round begins and a Player has not yet submitted a Score for that Round, THE App SHALL present an option to capture a Snapshot using the device camera and an option to enter the score manually.
2. WHEN a Player captures a Snapshot, THE App SHALL transmit the image to the ML_Service for analysis.
3. WHEN the ML_Service returns a calculated point value, THE App SHALL display the value to the Player for confirmation before recording it as the Round Score.
4. WHILE a Player has not confirmed a calculated value, THE App SHALL allow the Player to retake a Snapshot, replacing the previous calculated value with the new result.
5. IF the ML_Service fails to return a result within 15 seconds, THEN THE App SHALL notify the Player of the timeout and present the manual score entry field.
6. WHEN a Player initiates Snapshot capture, THE App SHALL display on-screen capture instructions (framing overlay and guidance text) to improve ML_Service accuracy.
7. IF a Player dismisses the Snapshot confirmation screen without confirming or retaking, THEN THE App SHALL discard the calculated value and return the Player to the score submission screen with both the Snapshot and manual entry options available.

---

### Requirement 5: ML-Based Point Recognition

**User Story:** As a player, I want the app to automatically recognize the value of my cards or tiles from a photo, so that scoring is fast and accurate.

#### Acceptance Criteria

1. WHEN the ML_Service receives a Snapshot of Domino tiles, THE ML_Service SHALL identify each visible tile and return the sum of all pip values in the image.
2. WHEN the ML_Service receives a Snapshot of Phase Ten cards, THE ML_Service SHALL identify each visible card and return the sum of all card point values according to Phase Ten scoring rules.
3. THE ML_Service SHALL return a calculated point value within 15 seconds of receiving a Snapshot.
4. IF the ML_Service produces a confidence score below 80% for a Snapshot, THEN THE ML_Service SHALL return a low-confidence flag alongside the best-effort calculated value.
5. WHEN the ML_Service returns a low-confidence flag, THE App SHALL display a warning to the Player and recommend manual verification before confirming the Score.
6. THE ML_Service SHALL use a pre-trained or fine-tuned model that achieves a recognition accuracy of at least 90% on well-lit, in-focus Snapshots of standard Domino tile pip patterns and Phase Ten card face values and suits.
7. THE ML_Service SHALL accept JPEG or PNG images with a minimum resolution of 640×480 pixels and a maximum file size of 10 MB.
8. IF a Snapshot does not meet the format or resolution requirements, THEN THE ML_Service SHALL return an error response indicating the image is invalid and SHALL NOT attempt recognition.

---

### Requirement 6: Score Override

**User Story:** As a player, I want to manually correct a score if the ML algorithm gets it wrong, so that the final recorded score is always accurate.

#### Acceptance Criteria

1. WHEN a Player reviews an ML_Service-calculated Score before confirming, THE App SHALL provide an editable numeric field pre-populated with the calculated value.
2. IF a Player enters a Score_Override value that is not a non-negative integer, THEN THE App SHALL reject the input and display an error message indicating the valid value format.
3. WHEN a Player submits a valid Score_Override, THE App SHALL record the overridden value as the Player's Round Score and log that a manual correction was made.
4. WHILE the most recently completed Round ended less than 5 minutes ago, THE App SHALL allow a Player to edit their own Score for that Round without requiring approval from the Game initiator.
5. IF a Player attempts to edit a Score from a Round that ended more than 5 minutes ago, THEN THE App SHALL apply the change immediately and send a Notification to the Game initiator indicating the Player's name, the affected Round, and the old and new Score values.
6. IF a Score has been manually overridden, THEN THE App SHALL display an override indicator on that Score in the round-by-round breakdown, distinguishing it from ML-calculated Scores.

---

### Requirement 7: Real-Time Leaderboard

**User Story:** As a player, I want to see live score updates during a game, so that everyone knows the current standings without waiting for the game to end.

#### Acceptance Criteria

1. WHILE a Game is active, THE App SHALL update the Leaderboard within 3 seconds of a Player's Round Score being confirmed.
2. WHILE a Game is active, THE App SHALL display each Player's cumulative score and their rank on the Leaderboard, where ties are broken by the order in which Players submitted their most recent Round Score (earlier submission ranks higher).
3. WHEN a new Round begins, THE App SHALL update the pending submission status for each Player within 3 seconds, displaying which Players have submitted their Scores and which are still pending.
4. WHEN all Players have submitted their Scores for a Round, THE App SHALL display a Round summary showing each Player's Score for that Round for at least 3 seconds before advancing to the next Round.
5. THE App SHALL deliver Leaderboard updates to all Players in the same Game using a persistent connection (WebSocket or equivalent).

---

### Requirement 8: Room History and Game Records

**User Story:** As a room member, I want to view the history of all games played in a room, so that I can track long-term performance and revisit past results.

#### Acceptance Criteria

1. THE App SHALL maintain a Room_History containing all completed and in-progress Games for a Room, ordered by start date descending.
2. WHEN a Member views the Room_History, THE App SHALL display each Game's game type, start date, end date (if completed), and the winning Player; if two or more Players share the highest score, THE App SHALL display all tied Players as joint winners.
3. WHEN a Member selects a completed Game from the Room_History, THE App SHALL display the final Leaderboard and a round-by-round score breakdown showing each Player's Score per Round and an override indicator for any manually corrected Score.
4. WHEN a Member selects an in-progress Game from the Room_History, THE App SHALL display the current Leaderboard and the round-by-round score breakdown for all completed Rounds so far.
5. THE App SHALL retain Room_History data indefinitely unless the Room is deleted by the owner.
6. WHEN a Room owner initiates Room deletion, THE App SHALL display a confirmation prompt listing the number of Games and Members that will be permanently deleted.
7. WHEN a Room owner confirms the deletion prompt, THE App SHALL permanently delete the Room and all associated Games, Rounds, Scores, and Snapshots.
8. IF a Room owner dismisses the deletion confirmation prompt without confirming, THEN THE App SHALL cancel the deletion and return the owner to the Room view.

---

### Requirement 9: Notifications

**User Story:** As a player, I want to receive in-app notifications about ML scoring results, so that I am immediately informed when my Snapshot has been processed.

#### Acceptance Criteria

1. WHEN the ML_Service returns a calculated point value for a Player's Snapshot, THE App SHALL display an in-app Notification to that Player showing the calculated value within 3 seconds of receiving the ML_Service response.
2. WHEN the ML_Service returns a low-confidence flag for a Player's Snapshot, THE App SHALL display an in-app Notification to that Player indicating the result may be inaccurate and recommending manual verification within 3 seconds of receiving the ML_Service response.
3. IF the ML_Service fails to process a Snapshot within 15 seconds, THEN THE App SHALL display an in-app Notification to the Player indicating the timeout and SHALL present the manual score entry field within the same Notification or on the score submission screen.
4. WHEN a Notification is displayed, THE App SHALL keep it visible until the Player explicitly dismisses it or navigates away from the score submission screen.

---

### Requirement 10: Multi-Platform Mobile Support

**User Story:** As a player, I want to use Keepy on both iOS and Android, so that all players in a group can participate regardless of their device.

#### Acceptance Criteria

1. THE App SHALL be available on iOS 15 and later.
2. THE App SHALL be available on Android 10 (API level 29) and later.
3. WHEN a Player first initiates Snapshot capture, THE App SHALL request camera permissions from the operating system before accessing the device camera.
4. IF a User denies camera permissions, THEN THE App SHALL, within one UI transition, display an in-app message informing the User that Snapshot capture is unavailable and that manual score entry is the only available option, and SHALL disable the Snapshot capture option.
5. WHEN a User grants camera permissions after previously denying them, THE App SHALL re-enable the Snapshot capture option on the score submission screen.
6. THE App SHALL display all core game screens (score submission, Leaderboard, Room_History) without horizontal scrolling or overlapping UI elements on devices with screen widths between 360dp and 430dp.

---

### Requirement 11: Data Persistence and Offline Resilience

**User Story:** As a player, I want my scores and game data to be saved reliably, so that no progress is lost if I lose connectivity mid-game.

#### Acceptance Criteria

1. THE App SHALL persist all confirmed Round Scores to the Database regardless of network conditions.
2. WHEN a Player's device loses network connectivity during a Game, THE App SHALL queue confirmed Scores locally (persisting the queue across app restarts, up to a maximum of 100 queued Scores) and synchronize them to the Database when connectivity is restored.
3. WHILE a Player's device is offline, THE App SHALL display the last known Leaderboard state and show a visible offline banner indicating that the view may not reflect the latest scores.
4. THE Database SHALL store User, Room, Game, Round, and Score records in a backed-up data store with at least daily backups retained for a minimum of 30 days.
5. IF a synchronization attempt fails after connectivity is restored, THEN THE App SHALL retry the synchronization using exponential backoff with a maximum of 5 retry attempts before notifying the Player that their scores could not be synced and prompting them to retry manually.

---

### Requirement 12: Security and Privacy

**User Story:** As a user, I want my personal data and game history to be protected, so that only authorized people can access my information.

#### Acceptance Criteria

1. THE API SHALL require a valid session token for all endpoints that access or modify User, Room, Game, Round, Score, or Snapshot data.
2. IF a request to a protected endpoint is made with a missing or invalid session token, THEN THE API SHALL reject the request with an HTTP 401 Unauthorized response.
3. THE App SHALL transmit all data between the App and the API over HTTPS using TLS 1.2 or later.
4. THE Auth_Service SHALL store User passwords as salted cryptographic hashes and SHALL NOT store plaintext passwords.
5. IF a User presents a valid session token but is not a Member of the Room associated with the requested resource, THEN THE API SHALL reject the request with an HTTP 403 Forbidden response and SHALL NOT return or modify any Room, Game, Round, Score, or Snapshot data for that Room.
6. WHEN a Snapshot is successfully uploaded to the ML_Service, THE App SHALL delete the locally cached copy of that Snapshot from the device within 24 hours.
7. IF a Snapshot has not been successfully uploaded within 7 days of capture, THEN THE App SHALL delete the locally cached Snapshot from the device and notify the Player that the Snapshot was removed due to an upload failure.
8. THE Database SHALL store Snapshots in an access-controlled storage bucket accessible only by the ML_Service and the authenticated User who submitted the Snapshot.
