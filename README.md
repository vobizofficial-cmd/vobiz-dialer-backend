# Vobiz Dialer Backend

Backend service for Vobiz Dialer - handles inbound call routing via Vobiz Voice XML.

## Architecture

```
PSTN → Vobiz DID (+917965850027) 
    → Vobiz Application (webhook) 
    → This backend (/answer) 
    → Returns Dial XML 
    → Platform sends SIP INVITE to registered Android app
```

## Deployed URL

**Production**: `https://vobiz-dialer-backend.up.railway.app`

## Deploy to Railway (Free Tier)

1. **Fork/clone this repo**
2. **Create Railway project** → "Deploy from GitHub repo"
3. **Add Environment Variables** in Railway dashboard:
   ```
   FIREBASE_SERVICE_ACCOUNT=<full JSON from .env>
   VOBIZ_AUTH_ID=MA_4272LINL
   VOBIZ_AUTH_TOKEN=<your Vobiz API token>
   BACKEND_URL=https://vobiz-dialer-backend.up.railway.app
   ANSWER_MODE=dial
   PORT=3000
   REDIS_URL=<optional - for persistence>
   ```
4. **Deploy** → Get stable HTTPS URL
5. **Verify**: `curl https://your-app.railway.app/health`

## Local Development

```bash
npm install
cp .env.example .env
# Edit .env with your credentials
npm run dev
# Runs on http://localhost:3000
```

- Android debug builds expect backend at `http://10.0.2.2:3000` (emulator loopback)

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/health` | GET | Health check (`{status, devices, firebase, project, mode, redis, startedAt}`) |
| `/answer` | POST/GET | Vobiz Answer webhook — returns Dial XML |
| `/hangup` | POST | Vobiz Hangup webhook — logs call end |
| `/register-token` | POST | Register FCM device token (`{token, deviceId}`) |
| `/dial-status` | POST | Dial action callback (diagnostic) |
| `/dial-callback` | POST | Real-time B-leg lifecycle events (DialAnswer, DialConnected, hangup) |
| `/dial-log` | GET | Recent dial events (last 50) |
| `/probe-answer` | POST/GET | Probe route — returns Speak+Hangup for CDR visibility |

## Answer Webhook Response (Dial XML)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
    <Dial timeout="45" dialMusic="real" callerId="+917965850027"
          callbackUrl="https://vobiz-dialer-backend.up.railway.app/dial-callback"
          action="https://vobiz-dialer-backend.up.railway.app/dial-status"
          method="POST" redirect="false">
        <User>sip:sairam8391265128911238046@registrar.vobiz.ai</User>
    </Dial>
    <Speak>The customer is not available. Please try again later.</Speak>
    <Hangup/>
</Response>
```

### Key Dial Attributes
- `timeout="45"` — Ring timeout (seconds)
- `dialMusic="real"` — Caller hears real ringback (not silence)
- `callerId="+917965850027"` — DID shown to callee
- `callbackUrl` — Real-time events (DialAnswer, etc.)
- `action` — Final dial result (answered/failed/busy)
- `redirect="false"` — Don't follow redirects

## Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `FIREBASE_SERVICE_ACCOUNT` | Yes | — | Full Firebase Admin SDK JSON (from Console → Service Accounts) |
| `VOBIZ_AUTH_ID` | Yes | — | Vobiz Account Auth ID (`MA_4272LINL`) |
| `VOBIZ_AUTH_TOKEN` | Yes | — | Vobiz API Token |
| `BACKEND_URL` | Yes | — | Public HTTPS URL of this backend (for callback URLs) |
| `ANSWER_MODE` | No | `dial` | `dial` (direct SIP INVITE) or `conference` (FCM + conference) |
| `PORT` | No | `3000` | Server port (Railway sets) |
| `REDIS_URL` | No | — | Redis URL for persistence (tokens, dial log, conference room) |

## Vobiz Configuration (Pre-configured)

| Resource | Value |
|----------|-------|
| **Account** | `MA_4272LINL` |
| **Endpoint** | `sairam8391265128911238046@registrar.vobiz.ai` (ID: `269321988178847`) |
| **DID** | `+917965850027` |
| **Voice Application** | `85076776948601220` (Vobiz WebRTC Playground) |
| **Outbound Trunk** | `08ecd76e.sip.vobiz.ai` (ID: `08ecd76e-75aa-460f-bed0-3228030070fe`) |
| **Trunk Credential** | `dialer_trunk_auth` / `omsairamji` |

### Voice Application Webhook URLs
```
Answer URL:  https://vobiz-dialer-backend.up.railway.app/answer
Answer Method: POST
Hangup URL:  https://vobiz-dialer-backend.up.railway.app/hangup
Public URI:  Enabled
Attached Numbers: +917965850027
```

## Call Flow (Standard Vobiz Pattern — Dial Mode)

```
1. User calls DID +917965850027
2. Vobiz routes to Application → POST /answer
3. Backend returns <Dial><User>sip:sairam...@registrar.vobiz.ai</User></Dial>
4. Platform sends SIP INVITE to registrar.vobiz.ai
5. Android app (via WSS relay) receives INVITE
6. App rings → User answers → Two-way audio (DTLS-SRTP)
7. Call ends → Vobiz calls POST /hangup
```

This is the **standard Vobiz pattern** used by rtc-demo, Vapi, Retell, and all production apps.

## Conference Mode (ANSWER_MODE=conference)

When `ANSWER_MODE=conference`:
1. Backend returns `<Conference>` XML instead of `<Dial>`
2. Caller parked in conference (hears hold music)
3. Backend sends FCM to all registered devices
4. App receives FCM → joins conference via SIP INVITE
5. Two-way audio through Vobiz conference bridge

**Default is `dial` mode** — no FCM needed for inbound.

## Quick Test

```bash
# Health check
curl https://vobiz-dialer-backend.up.railway.app/health

# Expected: {"status":"ok","devices":N,"firebase":true,"project":"vobiz-fire","mode":"dial","redis":"connected","startedAt":"..."}

# Test Answer webhook (simulates inbound call)
curl -X POST https://vobiz-dialer-backend.up.railway.app/answer \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "CallUUID=test123&From=+919876543210&To=+917965850027&Direction=inbound"

# Expected: Dial XML with <User>sip:sairam8391265128911238046@registrar.vobiz.ai</User>

# Test Probe
curl -X POST https://vobiz-dialer-backend.up.railway.app/probe-answer \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "CallUUID=probe123&From=+919876543210&To=+917965850027"
```

## Trigger Test Call via Vobiz API

```bash
curl -X POST "https://api.vobiz.ai/api/v1/Account/MA_4272LINL/Call/" \
  -H "X-Auth-ID: MA_4272LINL" \
  -H "X-Auth-Token: YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"from": "+917965850027", "to": "+919123151351", "answer_url": "https://vobiz-dialer-backend.up.railway.app/answer"}'
```

## Redis Persistence (Optional)

When `REDIS_URL` is configured:
- FCM tokens survive redeploys
- Dial log persists (last 50 events)
- Conference room name persists
- App starts faster (hydrates from Redis)

Without Redis: state is in-memory only (lost on restart).

## Known Issues

### Inbound Call Disconnects at ~32 Seconds
**Symptom**: Inbound call connects, audio works, but call ends at ~32 seconds.

**Likely Cause**: DTLS-SRTP renegotiation failure or media timeout in WSS relay / Linphone.

**Backend Impact**: The `/dial-callback` and `/dial-status` endpoints will receive hangup events with `HangupCause=MEDIA_TIMEOUT` or similar.

**Investigation**: See Android app docs → [Known Issues](https://github.com/vobizofficial-cmd/vobiz-dialer-android#known-issues)

## Monitoring

- **Health**: `GET /health` — includes `startedAt` to verify redeploy
- **Dial Log**: `GET /dial-log` — recent call events
- **Logs**: Railway dashboard → Deployments → View Logs

## Repository

https://github.com/vobizofficial-cmd/vobiz-dialer-backend

## Related

- **Android App**: https://github.com/vobizofficial-cmd/vobiz-dialer-android
- **Architecture Docs**: Android repo → `docs/ARCHITECTURE.md`
- **Inbound Debugging**: Android repo → `docs/INBOUND_DEBUG.md`