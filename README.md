# SecureVote

A face-authenticated electronic voting system, built as a UP Vidhan Sabha (Uttar Pradesh Legislative Assembly) election demo. Voters verify their identity with their face before casting a ballot, the vote itself is stored in a way that can't be traced back to the voter who cast it, and election officers control the voting lifecycle (registration → voting → results) through an admin dashboard.

## What it helps with

- Replacing manual ID checks at small-scale elections (student body, club, society, or demo-scale elections) with face-based verification
- Preventing one person from voting twice — both directly (same Voter ID can't vote twice) and indirectly (same face can't register under two different Voter IDs)
- Keeping vote choices anonymous even from whoever administers the system
- Giving election officers explicit control over the election lifecycle, so registration and voting don't stay open indefinitely or overlap
- Giving the public (not just admins) visibility into turnout and results once declared
- Demonstrating a complete authentication + liveness + secrecy pipeline rather than a bare face-match demo

## Election lifecycle

An election moves through three phases, controlled from the admin dashboard:

| Phase | What's open | What's locked |
|---|---|---|
| **Registration open** (default) | New voter registration | Voting, ballot casting |
| **Voting open** | Authentication + casting a ballot | Registration |
| **Results declared** | Public results page shows the tally | Registration, voting |

Registration and voting are mutually exclusive by design — once voting opens, the roll is closed. Visiting `/register` or `/vote` outside the matching phase shows a locked-out notice instead of the form; hitting the underlying API endpoints outside their phase returns an HTTP 403.

## How it works

**Registration** *(only during the "Registration open" phase)*
1. Voter enters a Voter ID (must match the UP format `UP/23/142/0001234` — state/year/constituency code/7-digit number, validated both client- and server-side), name, email (optional), and constituency, then captures one photo via webcam.
2. The photo is converted into a 128-dimension face encoding using `face_recognition`.
3. That new encoding is compared against every already-registered voter's encoding. If it matches an existing one, registration is rejected and logged to `flagged_duplicates` — this is what stops the same person enrolling under two different identities.
4. If no match is found, the encoding is encrypted (Fernet/AES) and stored against the Voter ID and constituency. The raw photo itself is discarded; only the encrypted encoding persists.

**Voting** *(only during the "Voting open" phase)*
1. Voter enters their Voter ID and looks at the camera. The frontend captures a burst of ~16 frames over roughly 2 seconds, using the front-facing camera by default (with a manual camera-switch option on devices with more than one camera).
2. Liveness check: eye landmarks are extracted from each frame and the Eye Aspect Ratio (EAR) is tracked across the burst, looking for an open → closed → open pattern (a real blink). This filters out static photos or a phone screen held up to the camera.
3. Identity check: the middle frame of the burst is encoded and compared (1-to-1) against the encryption-decrypted encoding stored for that specific Voter ID, scoped to that voter's own constituency ballot.
4. If both checks pass, a short-lived, single-use server-side session is opened and the voter is shown the ballot for their constituency.
5. A voter who has already voted gets a specific message — "Your vote was cast on [date] at [time] UTC. Each voter may vote only once." — rather than a generic error, using a `voted_at` timestamp that records *when* they voted without touching the anonymous ballot record.

**Casting a vote**
1. The voter selects a candidate (scoped to their constituency) and submits.
2. The vote is written to a `ballots` table that has no voter-identifying column at all — not nulled, not hidden, structurally absent. There's no query that joins a cast vote back to a voter.
3. The voter's record is flagged as having voted (with a timestamp), and the session is invalidated (single use).

**Public results & turnout**
- `/results` is public, no login required. While the phase is "Registration open" or "Voting open" it shows a "not yet declared" notice; once the admin declares results, it shows the live party-wise tally as a bar chart.
- The landing page (`/`), the results page, and the admin dashboard all show real-time turnout stats — total registered voters, total votes cast, and turnout % — safe to show publicly since it's headcounts only, never choices.

**Admin**
- Separate authentication path: username + password (bcrypt-hashed), session via JWT.
- Dashboard shows the current election phase (with one-click controls to change it), turnout stats, the live tally, the voter roll, flagged duplicate-registration attempts, and an audit log of authentication events (register, auth success/fail, liveness fail, vote cast, phase changes). The audit log records that an event happened, never which candidate was selected.
- Candidates can be added or removed, scoped per constituency. Removing a candidate who already has votes is blocked at the database level (foreign key constraint), so a tally can't be silently erased.
- A voter locked out after repeated failed authentication attempts can be manually unlocked from the dashboard.
- Results can be exported as a one-page PDF report (party tally + turnout stats) via `reportlab`, downloadable straight from the dashboard.

## Technologies used

| Layer | Tech |
|---|---|
| Web framework | FastAPI, Uvicorn |
| Request validation | Pydantic (incl. custom Voter ID format validator) |
| Database | MySQL, accessed via PyMySQL |
| Face detection/encoding | face_recognition (dlib) |
| Image handling | OpenCV, Pillow |
| Encryption at rest | cryptography (Fernet / AES) |
| Admin auth | bcrypt (password hashing), PyJWT (sessions) |
| Sessions (voter side) | Starlette SessionMiddleware (signed cookie) |
| PDF export | reportlab |
| Frontend | Vanilla HTML/CSS/JS, browser `getUserMedia` API (with explicit front-camera constraints + switch-camera support for mobile) |
| Runtime | Python 3.11.9 |

No external/cloud APIs are called — face matching, encryption, PDF generation, and token signing all run locally on whichever machine hosts the app.

## Project structure

```
Secure-vote/
├── app.py                FastAPI routes (voter-facing + admin)
├── database.py           MySQL access layer, schema migrations on startup
├── face_engine.py        Face encoding, matching, liveness, duplicate detection
├── dlib_backend.py       dlib-based face recognition backend
├── auth.py               Admin password hashing + JWT
├── schemas.py            Pydantic request models (incl. Voter ID format validation)
├── schema.sql            Standalone DB schema (kept in sync with database.py migrations)
├── requirements.txt      Python dependencies
├── test_connection.py    Standalone MySQL connectivity check
├── LICENSE               MIT
├── templates/            Jinja2 HTML pages
│   ├── base.html
│   ├── index.html          Landing page + public turnout stats
│   ├── register.html       Voter registration (phase-gated)
│   ├── vote.html            Face verification (phase-gated)
│   ├── ballot.html          Ballot casting
│   ├── results.html         Public results page
│   ├── admin_login.html
│   └── admin_dashboard.html Election phase control, tally, voters, audit log, PDF export
└── static/
    ├── css/style.css
    └── js/capture.js        Camera capture, burst sampling, mobile camera handling
```

## Errors during face capture

| Error shown | Cause | Fix |
|---|---|---|
| "No face detected" | Face not in frame, too far from camera, or lighting too dark/bright | Center the face, improve lighting, move closer |
| "Multiple faces detected" | More than one face in frame (second person, poster, photo in background) | Make sure only one face is visible |
| "Could not extract face features" | Face at a steep angle, motion blur, or low-contrast lighting | Face the camera directly, hold still, improve lighting |
| "No blink detected. This may be a static photo." | Liveness check (voting step only) found no open-closed-open eye pattern across the captured frames | Look at the camera and blink naturally during the ~2 second capture window; this also triggers on a printed photo or phone screen held up to the camera |
| "Face does not match our records for this Voter ID" | Live face doesn't match the stored encoding for the claimed Voter ID closely enough | Recapture with better lighting/angle; if it persists, the registered photo may have been low quality |
| "This face is already registered under a different Voter ID" | The 1-to-N duplicate check at registration matched an existing voter | Each person can only register once; this is by design |
| "Too many failed attempts. Try again later." | Repeated failed match/liveness attempts on one Voter ID | Temporary lockout (default 5 minutes after 5 failures); resets automatically, or an admin can unlock manually from the dashboard |
| "Registration is closed for this phase of the election." | Registration attempted outside the "Registration open" phase | Wait for the admin to open registration, or this is expected if voting has already started |
| "Voting is not open right now." | Authentication or vote-casting attempted outside the "Voting open" phase | Wait for the admin to open voting |
| "Voter ID must match the format UP/23/142/0001234..." | Voter ID doesn't match the required UP format (state/year/constituency code/7-digit number) | Re-enter the Voter ID in the correct format |

The face-match threshold is controlled by `MATCH_TOLERANCE` in `face_engine.py` (default `0.5`). Lower values are stricter (fewer false matches, more false rejections); higher values are looser. `face_recognition`'s own default is `0.6`, so `0.5` is already on the stricter side — relevant if rejections seem more frequent than expected on a typical laptop webcam.

## Security design notes

- **Encrypted encodings, not photos.** Only a 128-d vector is stored, encrypted with Fernet before it touches the database. Raw photos are never persisted.
- **Vote secrecy is structural.** No column or table links a cast ballot to a voter ID.
- **1-to-1 vs 1-to-N.** Voting authentication is 1-to-1 (does this face match the claimed Voter ID's stored encoding). Registration is 1-to-N (does this face match *any* existing voter). This split is what prevents duplicate enrollment without making every vote a full database scan.
- **Liveness via blink detection (EAR).** Effective against printed photos and screens. Not effective against pre-recorded video of the real person blinking — that requires depth sensors or challenge-response checks, which this project doesn't implement.
- **Lockout on repeated failure.** Prevents unlimited brute-force attempts against the match threshold for a given Voter ID.
- **Phase enforcement is server-side.** Every phase restriction (registration/voting/results) is checked on the API endpoints themselves, not just hidden in the UI — so the phase can't be bypassed by calling the API directly.
- **`voted_at` doesn't weaken secrecy.** The re-vote timestamp lives on the voter's own record (which already tracks `has_voted`), never on the anonymous `ballots` table, so it reveals *when* someone voted, never *what* they voted for.

## Limitations

- Cannot reliably distinguish identical twins or very close lookalikes — a fundamental limit of face recognition, not a software bug.
- Duplicate-face checking at registration is O(N) — it compares against every existing voter. Workable at college/demo scale (hundreds to low thousands); not how this would be implemented for a national-scale system (would require an indexed vector search, e.g. FAISS).
- Liveness detection is webcam-only and does not defeat video replay or deepfakes.
- A voter has no way to independently verify their specific vote was counted without revealing their choice (no cryptographic receipt system) — results can only be checked in aggregate via the public results page.
- Not designed for legally-binding elections (state/national). That scale requires audited secret-ballot guarantees, accessibility accommodations, supervised capture environments, cryptographically verifiable tallying, and avoids centralizing biometric data the way this project does for convenience at demo scale.

## Setup

Requires Python 3.11.9 and a running MySQL server.

```bash
git clone https://github.com/Chirag0071/Secure-vote.git
cd Secure-vote
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env             # fill in real DB/admin credentials
python test_connection.py        # verify MySQL connectivity
uvicorn app:app --reload
```

Tables, columns, and the admin account are created/migrated automatically on first run and on every subsequent startup, using whatever is set in `.env`. The election phase defaults to **"Registration open"** on a fresh database — switch it to "Voting open" from the admin dashboard once registrations are in.

## Deployment

Deployed on [Render](https://render.com) (app hosting) with [Aiven](https://aiven.io) (managed MySQL). In broad strokes:

1. Provision a MySQL instance on Aiven and note the host, port, user, password, database name, and CA certificate.
2. On Render, create a new Web Service pointing at this repo.
   - **Build command:** `pip install -r requirements.txt`
   - **Start command:** `uvicorn app:app --host 0.0.0.0 --port $PORT`
3. Set environment variables on Render to match `.env` (DB connection details, `SECUREVOTE_JWT_SECRET`, Fernet encryption key, admin bootstrap credentials).
4. `requirements.txt` is UTF-16 encoded — keep it that way if editing manually, or Render's pip install may fail to parse it.

## License

MIT — see [`LICENSE`](./LICENSE). Note this covers SecureVote's own code only; bundled third-party dependencies (e.g. `dlib-bin`, under the Boost Software License) carry their own separate licenses.
