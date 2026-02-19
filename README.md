# Booking Notifications: LINE OA + Facebook Messenger

Static site + Express backend for booking alerts.

When customer submits the booking form:

- Sends to `LINE OA` (Messaging API)
- Sends to `Facebook Page Messenger` (to configured PSID list)
- Returns `2xx` only when all notifications succeed
- Returns non-`2xx` on any failure so frontend can fallback to opening LINE

## 1) Install and run

```bash
npm install
npm start
```

Server starts at `http://localhost:3000`.

## Deploy on Render (Recommended)

### Option A: Blueprint (uses `render.yaml`)

1. Push this project to GitHub.
2. In Render, click `New` -> `Blueprint`.
3. Select your repo and create service.
4. In Render service `Environment`, fill secret values:
   - `LINE_CHANNEL_ACCESS_TOKEN`
   - `LINE_TO_IDS`
   - `FB_PAGE_ACCESS_TOKEN`
   - `FB_RECIPIENT_PSIDS`
   - `ALLOWED_ORIGINS` (optional but recommended when frontend is on another domain)
5. Deploy and wait until status is `Live`.

### Option B: Manual Web Service

1. `New` -> `Web Service` -> connect your repo.
2. Runtime: `Node`
3. Build command: `npm install`
4. Start command: `node server.js`
5. Add the same environment variables as above.

After deployment:

- Health URL: `https://<your-service>.onrender.com/health`
- Config check: `https://<your-service>.onrender.com/api/config-check`
- LINE webhook URL: `https://<your-service>.onrender.com/api/line-webhook`
- Booking webhook URL: `https://<your-service>.onrender.com/api/booking-notify`

If your booking page is served by this same Render service, no extra change is needed because `index.html` already uses `/api/booking-notify`.
If frontend is hosted elsewhere, set `data-webhook-url` in `index.html` to the full Render URL above.

## 2) Configure environment

Edit `.env`:

```env
PORT=3000

LINE_CHANNEL_ACCESS_TOKEN=YOUR_LINE_CHANNEL_ACCESS_TOKEN
LINE_TO_IDS=Uxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

FB_PAGE_ACCESS_TOKEN=YOUR_FB_PAGE_ACCESS_TOKEN
FB_RECIPIENT_PSIDS=1234567890123456

# Optional: lock CORS to your frontend domains
# ALLOWED_ORIGINS=https://your-site.com,https://www.your-site.com
```

## 3) Frontend webhook URL

`index.html` booking form already points to:

- `data-webhook-url="/api/booking-notify"`

So no extra change is needed when serving this site from the same server.

## 4) API endpoint

### `POST /api/booking-notify`

- Requires `Content-Type: application/json`
- Example payload:

```json
{
  "name": "สมชาย",
  "phone": "0812345678",
  "service": "งานรีโนเวท",
  "date": "2026-02-20",
  "time": "10:00",
  "area": "บางนา",
  "note": "เข้าบ้านหลัง 9 โมง",
  "raw_message": "(optional)"
}
```

Success:

- `200` when LINE delivery succeeds (Facebook may still fail; details returned in `failed`)

Failure:

- `400` invalid JSON body
- `415` unsupported content type (must be `application/json`)
- `500` missing config
- `502` when LINE delivery fails

This behavior is intentional so frontend fallback can trigger.

Quick test:

```bash
curl -X POST http://localhost:3000/api/booking-notify \
  -H "Content-Type: application/json" \
  -d "{\"name\":\"ทดสอบ\",\"phone\":\"0812345678\",\"service\":\"งานรีโนเวท\",\"date\":\"2026-02-20\",\"time\":\"10:00\",\"area\":\"บางนา\",\"note\":\"test\"}"
```

## 5) Health checks

- `GET /health`
- `GET /api/config-check`

## 6) Where to get IDs/tokens

### LINE

- `LINE_CHANNEL_ACCESS_TOKEN`: from LINE Developers console (Messaging API channel)
- `LINE_TO_IDS`: target `userId/groupId/roomId`
  - For team alerts, add your OA bot into the target chat (group or user) and use that ID

### Facebook

- `FB_PAGE_ACCESS_TOKEN`: from Meta for Developers (Page token with `pages_messaging`)
- `FB_RECIPIENT_PSIDS`: PSID(s) of people who have already messaged your Page
  - Team members should message the Page first
  - Then use their PSID as recipient targets

## Make Team See Alerts Immediately

1. LINE: create one team chat (group) and set that `groupId` in `LINE_TO_IDS`.
2. Facebook: every team member must message your Page at least once.
3. Put all team PSIDs in `FB_RECIPIENT_PSIDS` (comma-separated).
4. Run `GET /api/config-check` and ensure token/recipient counts are present before going live.

## Notes

- LINE Notify is deprecated; this project uses LINE Messaging API.
- Keep `.env` secrets private and never commit production tokens.
