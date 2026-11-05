# Live call transcription + foul-language flagging (test scaffold)

Single-number test setup: places one Twilio call to your own phone, forks the
live audio to this server via Twilio Media Streams, transcribes it in ~2s
chunks with Gemini, runs a Gemini moderation check on each chunk, stores
every entry in Redis, and pushes it live to an HTML dashboard over a
websocket.

## How it works

1. `start_call.py` calls your phone via Twilio, pointing at `/twiml`.
2. `/twiml` returns TwiML that (a) forks your live audio to `/media` over a
   websocket, and (b) bridges the call into a conference so it stays open.
3. `/media` receives raw audio frames from Twilio, buffers ~2 seconds,
   converts mu-law → PCM16 WAV, and sends it to Gemini for transcription.
4. Each transcribed line is sent to Gemini again as a yes/no moderation
   check (backed by a fast local keyword list as a fallback), then written
   to Redis (`transcript:<call_sid>`, a list of JSON entries) and broadcast
   to any connected dashboard clients.
5. `dashboard.html`, served at `/`, opens a websocket to `/ws/dashboard` and
   renders each line live, highlighting flagged ones in red.

This chunks audio every ~2s rather than doing continuous low-latency
streaming — good enough to prove out the pipeline. If you want sub-second
latency later, swap `transcribe_chunk` for the Gemini Live API's
bidirectional audio websocket instead of polling `generate_content`.

## Setup

```bash
pip install -r requirements.txt
cp .env.example .env
# fill in .env: Twilio creds + your Twilio number, your own phone number,
# a Gemini API key, and your Redis URL
```

You need a public HTTPS/WSS URL for Twilio to reach this server. Easiest is
ngrok:

```bash
ngrok http 8000
```

Copy the `https://xxxx.ngrok-free.app` domain (without the protocol) into
`PUBLIC_HOST` in `.env`.

Start Redis if it isn't already running:

```bash
redis-server
```

## Run

Terminal 1 — the server:

```bash
uvicorn server:app --host 0.0.0.0 --port 8000
```

Terminal 2 — open the dashboard in your browser:

```
http://localhost:8000/
```

Terminal 3 — place the test call:

```bash
python start_call.py
```

Answer your phone and talk. Say a normal sentence, then say something with
a swear word in it — you should see both lines appear on the dashboard
within a couple seconds, with the second one highlighted and marked
FLAGGED.

## Notes for going beyond a self-test

- Recording/transcribing a call with someone else (your friend, a
  supervisor) may require their consent depending on your jurisdiction —
  worth checking before you point this at real calls with other people.
- `dashboard_clients` and the in-memory broadcast are per-process; for
  multiple server workers you'd fan out via the Redis `transcript_updates`
  pub/sub channel instead (already published to, just not yet subscribed
  to — one line of work to wire up `redis_client.pubsub()` and forward to
  `broadcast()`).
- If you want a searchable/queryable history later (the "RAG" part), the
  Redis lists already give you the raw transcript per call; embedding each
  entry into a vector store (e.g. `redis` itself supports vector search via
  RediSearch, or use a dedicated store) is a natural next step once the
  live path is working.
