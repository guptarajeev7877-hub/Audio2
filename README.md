# Reissue

Upload an old recording, get the hiss and hum stripped off it, then pull the mix
apart into vocals / drums / bass / instruments and rebalance them in the browser.

Two pieces:

- **backend** — FastAPI. Runs the ffmpeg restoration chain and calls Demucs for
  source separation.
- **frontend** — one HTML file. Loads the stems into Web Audio, gives you a fader
  per instrument with a live VU meter, and renders your balance to a WAV.

---

## The one API key you need

Separation needs a GPU. You have two options:

**Replicate (recommended to start).** No GPU, no setup, roughly 2c per track.

1. Sign up at replicate.com
2. Grab a token from `replicate.com/account/api-tokens`
3. Put it in `backend/.env` as `REPLICATE_API_TOKEN=r8_...`

**Local.** Free, but a 4-minute song takes ~1 minute on a decent GPU and
15+ minutes on CPU.

```
pip install demucs torch
# in .env:
SEPARATION_BACKEND=local
```

The **Clean only** mode needs no key at all — that's pure ffmpeg and runs anywhere.

---

## Run it locally

```bash
cd backend
cp .env.example .env          # add your Replicate token
pip install -r requirements.txt
uvicorn main:app --reload --port 8000
```

ffmpeg must be on the path (`brew install ffmpeg` / `apt install ffmpeg`).

Then serve the frontend:

```bash
cd frontend
python -m http.server 5173
```

Open `http://localhost:5173`. If your API isn't on `localhost:8000`, set it once
in the browser console:

```js
localStorage.setItem("reissue_api", "https://api.yourdomain.com")
```

---

## Deploy

**Backend** — the Dockerfile is ready. Render, Railway, Fly, or your own VPS all
work. Set the env vars from `.env.example` in the platform's dashboard, and set
`ALLOWED_ORIGINS` to your frontend's real domain rather than `*`.

```bash
docker build -t reissue ./backend
docker run -p 8000:8000 --env-file backend/.env reissue
```

**Frontend** — it's a static file. Netlify, Vercel, Cloudflare Pages, GitHub
Pages, or nginx.

Note that separation jobs hold a GPU for a minute or two, so the free tier of most
platforms will time out on long tracks. Replicate does the heavy lifting on their
hardware, which is why it's the default.

---

## Running it from an iPhone

**Setting it up** is much faster on a computer — about ten minutes. **Using it**
afterwards works properly on the phone. If you only have the iPhone, the whole
setup is still possible; see the notes at each step.

### 1. Get the code onto GitHub

On a computer: `git init`, commit, push to a new repo.

On iPhone only: install **Working Copy** from the App Store. It's a full git
client. Create a repo, paste the files in, push. The GitHub website in Safari can
also create files one at a time, but with seven files across two folders that's
tedious.

### 2. Backend on Render

1. render.com → New → Web Service → connect the repo
2. Root Directory: `backend`, Runtime: **Docker**
3. Add environment variables:
   - `REPLICATE_API_TOKEN` = your token
   - `SEPARATION_BACKEND` = `replicate`
   - `ALLOWED_ORIGINS` = your frontend URL from step 3 (come back and set this)
4. Deploy. You get something like `https://reissue-api.onrender.com`

Render's dashboard works fine in mobile Safari.

Two things about the free tier: it sleeps after 15 minutes, so the first request
after a gap takes ~50 seconds to wake, and it has 512 MB of RAM which is enough
because Replicate does the separation elsewhere. Don't set
`SEPARATION_BACKEND=local` on a free instance — it will run out of memory.

### 3. Frontend on Cloudflare Pages

1. pages.cloudflare.com → Create → connect the same repo
2. Build command: leave empty. Output directory: `frontend`
3. Deploy. You get `https://reissue.pages.dev`

Go back to Render and set `ALLOWED_ORIGINS` to that exact URL, then redeploy.

### 4. On the phone

Open the Pages URL in Safari, type your Render URL into **Server address** at the
top, tap Save. The line underneath tells you whether it connected. That setting
persists, so you do it once.

Then Share → **Add to Home Screen**. It launches without Safari chrome and
behaves like an app.

### iPhone-specific behaviour

- **Export runs on the server on mobile.** Four decoded stems plus an offline
  render buffer is roughly 400 MB for a 4-minute song, past what iOS gives a tab.
  The phone sends its fader positions to `/api/jobs/{id}/mix` and downloads a
  320 kbps MP3 instead. Desktop still renders locally.
- **Playback still decodes all four stems in memory.** Tracks past about six
  minutes may still reload the tab. Nothing to be done short of streaming
  playback from the server.
- **Audio needs a tap to start.** iOS won't let a page make sound without a
  gesture; the Play button is that gesture.
- **Files app works, Apple Music doesn't.** Purchased and streamed tracks are
  DRM-protected and won't open in the picker. Anything in Files, iCloud Drive,
  Dropbox or Google Drive is fine.
- Keep an eye on Render's upload limit — `MAX_UPLOAD_MB` defaults to 60, which
  covers a WAV of about 5 minutes.

---

## How the restoration actually works

The filter chain in `audio.py` runs in the order a restoration engineer would:

| Step | Filter | What it's fixing |
|---|---|---|
| 1 | `highpass=f=32` | Turntable rumble, tape wow |
| 2 | `equalizer` notches at 50/100/150/200 Hz | Mains hum and its harmonics |
| 3 | `afftdn` | Broadband tape hiss, by spectral subtraction |
| 4 | `adeclick` | Vinyl clicks and pops |
| 5 | `adeclip` | Clipping — 60s masters were often driven hard |
| 6 | `equalizer` at 3.2k / 9.5k / 110 Hz | Presence, air, and low-end weight |
| 7 | `loudnorm` | EBU R128 to −14 LUFS, streaming level |

The presence lift is deliberately gentle. Cranking the top end on a 1965 master
mostly just amplifies whatever hiss survived step 3.

---

## What this can't do

Restoration removes noise. It does not add information that was never captured.
If the file you upload is a 128 kbps rip that dies at 15 kHz, everything above
15 kHz stays gone — the app tells you when it detects this, because at that point
the fix is a better source file, not more processing.

Stem separation is also imperfect by nature. Expect some bleed between stems and
some smearing on cymbals, especially on dense mono recordings from the 60s where
the instruments were bounced to the same tape track in the first place.

---

## Where to take it next

- Swap Demucs for BS-RoFormer / Mel-RoFormer, which now beat it on most benchmarks
- Cache jobs by file hash so re-processing the same track is free
- Add a per-stem EQ before the fader, not just a gain
- Move the in-memory job dict to Redis before you run more than one worker
