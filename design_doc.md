# Mixxx Local Streaming Bridge Design Document

## Project Overview

The goal of this project is to build a fully local streaming bridge for Mixxx that allows users to:

1. Paste Spotify track/album/playlist links into a local application or API
2. Automatically download the corresponding audio using SpotiFLAC
3. Import the downloaded tracks into Mixxx
4. Notify the user on macOS when downloads are complete
5. Expose a clean local API for automation and scripting
6. Maintain playlists and metadata synchronization

The entire system runs locally on the user's computer.

There is NO hosted backend and NO cloud dependency required for the MVP.

The system is intended to function as a personal DJ workflow tool that integrates streaming discovery with local DJ library management.

---

# High-Level Architecture

```text
+----------------------+
| Local UI / CLI / API |
+----------+-----------+
           |
           | HTTP/WebSocket
           v
+----------------------+
| Local Bridge Server  |
|----------------------|
| REST API             |
| WebSocket API        |
| Download Queue       |
| Metadata Manager     |
| Playlist Manager     |
+----------+-----------+
           |
           v
+----------------------+
| Download Engine      |
|----------------------|
| SpotiFLAC Wrapper    |
| File Organizer       |
| Metadata Scanner     |
| Notification Service |
+----------+-----------+
           |
           v
+----------------------+
| Mixxx Integration    |
|----------------------|
| Watched Folders      |
| SQLite Integration   |
| Playlist Sync        |
+----------------------+
```

---

# Core Components

# 1. Local API Server

## Responsibilities

The local API server is the central orchestration layer.

It is responsible for:

* receiving Spotify URLs
* managing download queues
* exposing automation endpoints
* managing playlist synchronization
* tracking download status
* sending notifications

The API is intended to be easy to use from:

* Python scripts
* shell scripts
* external applications
* Mixxx integrations
* future GUI applications

---

## Suggested Stack

### Recommended

* FastAPI
* asyncio
* uvicorn
* websockets

---

## Local API Design

The local server runs on:

```text
http://localhost:3876
```

---

## Core Endpoints

### Download Track

```http
POST /download
```

Body:

```json
{
  "spotify_url": "https://open.spotify.com/track/..."
}
```

Response:

```json
{
  "job_id": "uuid",
  "status": "queued"
}
```

---

### Get Download Status

```http
GET /jobs/{job_id}
```

---

### List Active Jobs

```http
GET /jobs
```

---

### Create Playlist

```http
POST /playlist
```

---

### Trigger Mixxx Rescan

```http
POST /mixxx/rescan
```

---

## WebSocket Events

```text
ws://localhost:3876/ws
```

Events:

* download progress
* download completed
* import completed
* playlist updates
* notifications

---

## Example Python Usage

```python
import requests

requests.post(
    "http://localhost:3876/download",
    json={
        "spotify_url": "https://open.spotify.com/track/..."
    }
)
```

---

## Example curl Usage

```bash
curl -X POST http://localhost:3876/download \
-H "Content-Type: application/json" \
-d '{
  "spotify_url": "https://open.spotify.com/track/..."
}'
```

---

# 2. Local Download Queue

## Responsibilities

The queue manager coordinates:

* concurrent downloads
* retry handling
* playlist grouping
* status tracking
* import sequencing

Everything is stored locally.

---

## Suggested Storage

The MVP should avoid using SQLite or any dedicated database for application state.

The expected workload is extremely small:

* typically 3-4 tracks per session
* lightweight local-only operation
* no multi-user synchronization
* minimal queue complexity

A simple in-memory queue plus lightweight JSON configuration files is sufficient.

Recommended local configuration directory:

```text
~/.mixxx-streaming/
```

Suggested files:

```text
config.json
session.json
```

The application should primarily operate in-memory during runtime.

Session state can optionally be persisted temporarily to JSON for:

* crash recovery
* restoring active downloads
* tracking downloaded files for cleanup

This keeps the architecture lightweight and avoids unnecessary infrastructure.

---

## Session Tracking

The application should maintain a temporary session object containing:

* tracks downloaded this session
* original Spotify URLs
* local file paths
* duplicate-detection decisions
* import status

This enables cleanup and rollback workflows.

---

## Session Cleanup Feature

The user should be able to wipe all tracks downloaded during the current session.

Example workflow:

```text
1. Start session
2. Download tracks
3. Import into Mixxx
4. End session
5. Optionally remove all imported files
```

Suggested API endpoint:

```http
POST /session/wipe
```

Behavior:

* remove downloaded files from configured music folder
* optionally remove tracks from Mixxx library if future DB integration exists
* clear temporary session state

This feature is especially useful for:

* temporary DJ sets
* testing workflows
* avoiding library clutter
* short-lived imports

---

## Job Model

```json
{
  "id": "uuid",
  "spotify_url": "https://open.spotify.com/...",
  "status": "queued",
  "created_at": "timestamp"
}
```

### Device

```json
{
  "id": "uuid",
  "user_id": "uuid",
  "device_name": "Tony-MacBook",
  "last_seen": "timestamp",
  "status": "online"
}
```

### DownloadJob

```json
{
  "id": "uuid",
  "spotify_url": "https://open.spotify.com/...",
  "status": "queued",
  "device_id": "uuid",
  "created_at": "timestamp"
}
```

---

# 3. Local Download Engine

## Responsibilities

The local download engine is the most important system component.

It is responsible for:

* Receiving download jobs
* Running SpotiFLAC
* Organizing downloaded files
* Updating Mixxx library
* Triggering macOS notifications
* Synchronizing playlists

---

# Download Engine Architecture

```text
+----------------------+
| Local Agent          |
+----------------------+
| REST/WebSocket API   |
| Download Manager     |
| SpotiFLAC Adapter    |
| Metadata Scanner     |
| Mixxx DB Connector   |
| Notification Service |
| Playlist Sync Engine |
+----------------------+
```

---

# 4. SpotiFLAC Integration

Repository:

[https://github.com/spotbye/SpotiFLAC](https://github.com/spotbye/SpotiFLAC)

## Responsibilities

The agent wraps SpotiFLAC and:

* Passes Spotify URLs
* Tracks download progress
* Detects output files
* Extracts metadata
* Reports status back to backend

---

## Suggested Wrapper Flow

```python
async def process_job(job):
    spotify_url = job.spotify_url

    proc = await asyncio.create_subprocess_exec(
        "python",
        "spotiflac.py",
        spotify_url
    )

    await proc.wait()

    if proc.returncode == 0:
        import_into_mixxx()
        send_notification()
```

---

# 5. Mixxx Integration

## Approach Options

### Option A — Watched Music Folder (Recommended MVP)

Mixxx watches:

```text
~/Music/MixxxStreaming/
```

The agent downloads directly into this folder.

Advantages:

* Simple
* Stable
* Minimal DB manipulation
* Works across Mixxx versions

Disadvantages:

* Slower library refresh
* Less playlist control

---

## Option B — Direct SQLite Manipulation

The agent updates:

```text
mixxxdb.sqlite
```

Advantages:

* Immediate playlist insertion
* Better metadata control
* Automatic crate generation

Disadvantages:

* DB schema changes across versions
* Potential corruption risk
* Requires careful transactions

---

## Recommended Strategy

The preferred MVP architecture is:

1. Download directly into a user-selected local music folder
2. Let Mixxx monitor that folder using its existing library system
3. Trigger optional Mixxx library refresh events
4. Keep the integration layer lightweight and stable

The goal is to minimize complexity while providing a seamless workflow.

The system should behave similarly to rekordbox Spotify-style integrations where tracks quickly appear inside the DJ software after download.

---

## Future Integration Path

In the future, an optional advanced integration layer may:

* directly modify `mixxxdb.sqlite`
* automatically create playlists
* immediately insert tracks into crates/playlists
* synchronize metadata and playlist ordering

However, this is NOT part of the MVP because:

* Mixxx schema compatibility may change
* direct SQLite manipulation increases corruption risk
* the watched-folder approach is significantly more stable

---

## Preferred Music Storage Location

```text
~/Music/MixxxStreaming/
```

The application should allow the user to choose ANY folder on their machine.

Examples:

```text
~/Music/MixxxStreaming/
```

```text
~/DJ/Music/
```

```text
/Volumes/DJSSD/Music/
```

The application should NOT automatically organize tracks by:

* artist
* album
* genre
* playlist

This keeps the system lightweight and avoids unnecessary file-management complexity.

The storage path should be configurable in:

```text
~/.mixxx-streaming/config.json
```

---

## Mixxx Folder Workflow

Example flow:

1. User pastes Spotify playlist URL
2. Tracks are downloaded locally
3. Files are moved into the configured music folder
4. Mixxx automatically detects new tracks
5. User receives macOS notification

---

## Optional Future SQLite Integration

Future versions may support:

* automatic playlist insertion
* automatic crate generation
* direct SQLite integration

This would allow downloaded tracks to immediately appear in specific playlists.

However, this is intentionally deferred until after the MVP is stable.

````

---

## Important Safety Requirement

The importer MUST:

- use SQLite transactions
- avoid modifying Mixxx while it is actively writing
- validate schema compatibility
- support rollback on failure

Future enhancement:

- file locking
- schema migration handling
- automatic DB backups

---

# 6. macOS Notifications

## Requirements

When:

- download finishes
- import succeeds
- playlist sync completes

The user receives:

- macOS notification
- optional sound
- optional clickable action

---

## Recommended Implementation

### Native Notification

Use:

```bash
osascript
````

Example:

```bash
osascript -e 'display notification "Track Imported" with title "Mixxx Bridge"'
```

---

## Python Example

```python
import subprocess

subprocess.run([
    "osascript",
    "-e",
    'display notification "Download Complete" with title "Mixxx Bridge"'
])
```

---

# 7. Local WebSocket Synchronization

## Why WebSockets

Needed for:

* real-time queue updates
* live progress bars
* local app synchronization
* future GUI support
* Mixxx integration events

---

## Flow

```text
CLI / GUI
    ↕
Local API Server
    ↕
Download Engine
```

---

## Example Events

### Client → Server

```json
{
  "type": "download_progress",
  "progress": 72
}
```

---

# 8. File Organization

## Recommended Structure

```text
~/Music/MixxxStreaming/
    Artists/
        Artist Name/
            Album Name/
                Track.flac
```

---

## Playlist Export

Optional:

```text
~/Music/MixxxStreamingPlaylists/
```

Generate:

* M3U8 playlists
* Mixxx-compatible crates

---

# 9. Duplicate Detection and Track Matching

One important workflow feature is duplicate prevention.

Before downloading a track, the system should first inspect the Mixxx library to determine whether the track already exists locally.

---

## Duplicate Detection Flow

```text
Spotify URL
    ↓
Extract Metadata
    ↓
Search Mixxx Library
    ↓
Exact Match?
    ├── YES → Notify User + Skip Download
    └── NO
         ↓
    Similar Match?
         ├── YES → Download In Background + Ask User
         └── NO → Download Track Normally
```

---

## Exact Match Detection

The system should search the Mixxx library for:

* exact track title
* exact artist name
* optional duration comparison

If an exact match exists:

* do NOT redownload by default
* notify the user immediately
* skip import
* optionally allow force-download

Example notification:

```text
Track already exists in Mixxx library.
```

````

---

## Similar Match Detection

The system should also detect approximate matches.

Examples:

- remix versions
- extended mixes
- radio edits
- slightly different metadata formatting
- alternate spellings

Suggested techniques:

- fuzzy string matching
- normalized metadata comparison
- duration similarity thresholds

Potential Python libraries:

- rapidfuzz
- difflib

---

## Background Download Strategy

If a similar match is detected:

1. The track should STILL begin downloading immediately in the background
2. The user should receive a warning notification
3. The user decides whether to keep or delete the downloaded track

This minimizes waiting time during DJ workflows.

The assumption is:

- download time is more expensive than temporary disk usage
- DJs care more about responsiveness than perfect duplicate prevention

---

## Similar Match Workflow

```text
1. Detect possible duplicate
2. Begin background download immediately
3. Notify user
4. User chooses:
    - Keep download
    - Delete download
    - Ignore warning
````

---

Example notification:

```text
Possible similar track detected:
Daft Punk - One More Time (Extended Mix)

Downloading in background...
Keep or delete?
```

---

## Temporary Download Handling

While awaiting user confirmation:

* the downloaded track should be marked as temporary
* session tracking should record its status
* cleanup logic should support immediate deletion

Suggested temporary states:

```text
pending_keep
confirmed
pending_delete
```

---

## Deferred Deletion

If the user chooses delete:

* remove downloaded file
* remove temporary session entry
* optionally notify Mixxx to refresh library
* preserve original duplicate warning log

````

---

## Mixxx Library Access

The system should inspect:

```text
mixxxdb.sqlite
````

for duplicate checking ONLY.

The MVP should avoid modifying the database.

Relevant tables may include:

* library
* track_locations

---

## Recommended Matching Pipeline

```text
1. Extract Spotify metadata
2. Normalize strings
3. Search Mixxx DB
4. Compute exact matches
5. Compute fuzzy similarity scores
6. Notify user
7. Decide whether to download
```

---

## User Configuration

The user should be able to configure:

* duplicate threshold
* fuzzy matching sensitivity
* auto-skip exact matches
* whether similar matches require confirmation

---

# 10. Metadata Management

Metadata management is intentionally minimal.

The system is only responsible for:

* downloading audio files
* placing files into the configured folder
* optionally exposing basic metadata through the local API

Mixxx itself is responsible for:

* BPM analysis
* waveform generation
* beatgrid analysis
* key detection
* library metadata management

This keeps the architecture simpler and more aligned with Mixxx's native analysis pipeline.

---

# 10. Security Model

## Device Authentication

Each local agent receives:

```text
Device Token
```

stored securely in:

```text
~/Library/Application Support/MixxxBridge/
```

---

## Preventing Unauthorized Downloads

The backend only routes jobs to:

* authenticated users
* registered devices

This prevents random users from pushing downloads to someone else's machine.

---

# 11. Future Features

## Mobile Queueing

Allow:

* queue tracks from phone
* import remotely
* synchronize to desktop

---

## Optional Future SQLite Integration

Future versions may support:

* automatic playlist insertion
* automatic crate generation
* direct Mixxx database integration

---

# 12. MVP Scope

## MVP Features

### Required

* Paste Spotify URL
* Download via SpotiFLAC
* Save locally
* Import into Mixxx watched folder
* macOS notification
* Queue status UI
* WebSocket sync

### Not Required Yet

* Direct Spotify streaming
* Live deck injection
* Playlist DB insertion
* Mobile app
* Multi-user support

---

# 13. Recommended Tech Stack

## Frontend

* Next.js
* TailwindCSS
* React Query
* Socket.IO

## Backend

* FastAPI
* PostgreSQL
* Redis
* WebSockets

## Local Agent

* Python
* asyncio
* watchdog
* sqlite3
* websockets

---

# 14. Recommended Development Order

## Phase 1

* Local-only prototype
* Paste URL
* Trigger SpotiFLAC
* Save file
* Notification

## Phase 2

* Mixxx auto-import
* Playlist management
* Metadata parsing

## Phase 3

* Web frontend
* User auth
* Remote synchronization

## Phase 4

* Streaming decks
* Rekordbox export
* Cloud sync

---

## Project Context

This project is intended for:

* academic purposes
* personal experimentation
* local DJ workflow research
* Mixxx ecosystem prototyping

---

# 16. Long-Term Vision

The long-term goal is to create a modern open-source DJ ingestion pipeline that:

* bridges streaming discovery with local DJ workflows
* integrates deeply with Mixxx
* supports cloud queueing
* enables collaborative playlist workflows
* eventually supports real-time streaming decks

The system should feel similar to modern DJ streaming integrations such as rekordbox Spotify-style workflows where tracks quickly appear inside the DJ software after download.

The long-term focus is:

* fast ingestion
* lightweight local architecture
* automation-friendly APIs
* Mixxx-native workflows
* local-first operation
