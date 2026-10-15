# 🔎 AI Lost & Found Platform

A full-stack MERN application that helps people report lost and found items and
uses an AI-powered matching engine to detect when two independent reports likely
describe the **same physical object** — exactly like the JBL headphones example:

> Lost: *Black JBL headphones, near Central Library, Aug 12, small scratch on left earcup*
> Found: *Black wireless headphones, near Central Library, Aug 13, small scratch on left side*
> → **Possible Match — 91%** — 📍 Location matches · 🎧 Object type matches · 🎨 Color matches · 📝 Description similarity: High · 📅 Date proximity: 1 day

---

## Tech Stack

| Layer      | Tech |
|------------|------|
| Frontend   | React 18 (Vite), React Router, Tailwind CSS, Axios |
| Backend    | Node.js, Express, JWT auth, Multer (image uploads) |
| Database   | MongoDB + Mongoose |
| AI Matching| Local NLP (TF-IDF cosine similarity, Jaro-Winkler fuzzy matching via `natural`) + perceptual image hashing (`jimp`, dHash) — **no external API keys required** |

---

## How the AI Matching Works

For every new report, the backend compares it against all *open* reports of the
opposite type (`lost` ↔ `found`) and computes a weighted 0–100 confidence score
from six signals (see `backend/utils/matching.js`):

| Signal | Weight | Technique |
|---|---|---|
| Category / object type | 20 | Fuzzy string match (Jaro-Winkler + substring) |
| Color | 15 | Fuzzy string match |
| Location | 20 | Fuzzy string match |
| Description | 25 | TF-IDF weighted cosine similarity over stemmed tokens |
| Date proximity | 10 | Linear decay over a 14-day window |
| Image similarity (bonus, only if both have photos) | 10 | Perceptual dHash + average-color distance |

Matches scoring above `MATCH_THRESHOLD` (default 45%) are persisted, both
reporters are notified in-app, and the match — with its full breakdown and
human-readable flags (📍🎧🎨📝📅) — appears on each item's detail page.

This is genuinely "AI-powered" in the classical NLP/CV sense while staying
100% self-hosted: no OpenAI/Anthropic/Google API key is needed to run it.
If you'd like to swap in an LLM-based embedding similarity instead of TF-IDF,
that's a drop-in replacement inside `computeMatch()`.

---

## Project Structure

```
lost-found-platform/
├── backend/
│   ├── config/db.js              MongoDB connection
│   ├── models/                   User, Item, Match, Claim, Notification
│   ├── middleware/                auth (JWT), upload (Multer), error handler
│   ├── utils/
│   │   ├── matching.js           ⭐ the AI matching engine
│   │   ├── imageFingerprint.js   perceptual image hashing
│   │   ├── notify.js             notification helper
│   │   └── seed.js               demo data (incl. the JBL example)
│   ├── controllers/               business logic per resource
│   ├── routes/                    REST API routes
│   └── server.js                  app entry point
└── frontend/
    └── src/
        ├── context/AuthContext.jsx
        ├── components/            Navbar, ItemCard, MatchCard, route guards
        └── pages/                 Login, Register, Home, Report, ItemDetail,
                                    MyItems, Claims, Notifications, AdminDashboard
```

---

## Setup

### Prerequisites
- Node.js 18+
- MongoDB running locally (`mongodb://127.0.0.1:27017`) or a MongoDB Atlas URI

### 1. Backend

```bash
cd backend
cp .env.example .env      # edit JWT_SECRET / MONGO_URI if needed
npm install
npm run seed               # optional: creates demo users + the JBL example
npm run dev                 # starts on http://localhost:5000
```

### 2. Frontend

```bash
cd frontend
npm install
npm run dev                 # starts on http://localhost:5173
```

Open **http://localhost:5173**. The Vite dev server proxies `/api` and
`/uploads` to the backend on port 5000, so no CORS config is needed locally.

### Demo accounts (after `npm run seed`)
| Role | Email | Password |
|---|---|---|
| Admin | admin@lostfound.app | admin123 |
| User (lost headphones) | alex@student.edu | password123 |
| User (found headphones) | jordan@student.edu | password123 |

Log in as either Alex or Jordan and open their headphones item — you'll see
the AI match card with the full score breakdown immediately.

---

## REST API Overview

All authenticated routes require `Authorization: Bearer <token>`.

| Method | Route | Description |
|---|---|---|
| POST | `/api/auth/register` | Create account |
| POST | `/api/auth/login` | Login |
| GET | `/api/auth/me` | Current user profile |
| GET | `/api/items?type=&q=&mine=` | Search/list items |
| POST | `/api/items` | Report lost/found item (`multipart/form-data`, up to 5 images) — triggers AI matching |
| GET | `/api/items/:id` | Item detail + its matches |
| PUT / DELETE | `/api/items/:id` | Update / delete own item |
| GET | `/api/matches/mine` | All matches involving your items |
| PUT | `/api/matches/:id/reject` | Dismiss a suggested match |
| POST | `/api/claims` | Request a claim on a match |
| GET | `/api/claims/mine` | Claims where you're claimant or finder |
| PUT | `/api/claims/:id/decision` | Approve/reject a claim (finder only) |
| GET | `/api/notifications` | Your notifications |
| PUT | `/api/notifications/:id/read` / `/read-all` | Mark read |
| GET | `/api/admin/stats` \| `/users` \| `/items` \| `/claims` | Admin dashboard data (admin only) |
| PUT | `/api/admin/users/:id/toggle-active` | Activate/deactivate a user |
| DELETE | `/api/admin/items/:id` | Force-remove a listing |

---

## Core User Flow

1. **Report** — User submits a lost or found item report with photos.
2. **Match** — The AI engine immediately scores it against opposite-type
   open reports and creates `Match` records above the threshold.
3. **Notify** — Both reporters get an in-app notification with a link to
   the match.
4. **Review** — On the item page, users see AI match cards with the score
   ring, breakdown icons (📍🎧🎨📝📅), and a "Request Claim" button.
5. **Claim** — Claiming notifies the other party, who can **Approve** or
   **Decline** from the Claims page.
6. **Resolve** — On approval both items are marked `resolved`.

## Admin Dashboard

`/admin` (admin role required) shows platform-wide stats (users, lost/found
counts, resolved items, average match score, pending claims), a user table
with activate/deactivate controls, an item moderation table with force-delete,
and a claims overview.

---

## Notes on Production Hardening

This is a complete, runnable reference implementation. Before production use,
consider adding: rate limiting, email/SMS notifications, refresh tokens,
image storage on S3/Cloudinary instead of local disk, input sanitization
middleware (e.g. `express-mongo-sanitize`), and pagination on list endpoints.
