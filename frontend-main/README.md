# 🌿 LeafScan AI — Deployment Guide

LeafScan is a rice leaf disease detector with three services:

| Service | Technology | Deploy to |
|---|---|---|
| `frontend` | React (CRA) | Netlify |
| `auth-backend` | Node.js + Express + MongoDB | Render |
| `backend` | Python Flask + YOLO | Render |

---

## Architecture

```
Browser (Netlify)
  │  Login / Register / Detect
  ▼
auth-backend (Render - Node)   ──POST /predict──▶   backend (Render - Python/Flask)
  │  Saves scan to MongoDB              YOLO model returns label + annotated image
  ▼
MongoDB Atlas
```

---

## 1. Deploy Flask Model Backend to Render

1. Push `backend/` folder to a **separate GitHub repo** (or use monorepo with root dir setting).
2. Create a new **Web Service** on [render.com](https://render.com):
   - **Runtime:** Python 3
   - **Root Directory:** `backend`
   - **Build Command:** `pip install -r requirements.txt`
   - **Start Command:** `gunicorn app:app --bind 0.0.0.0:$PORT --timeout 120 --workers 1`
3. Add environment variable:
   - `PORT` = `10000`
4. **Important:** The `yolo_model/best.pt` file (~14MB) must be committed to the repo (already included in the zip). Git LFS is recommended for this.
5. Note the deployed URL, e.g. `https://leafscan-model.onrender.com`

---

## 2. Deploy Auth Backend to Render

1. Create another Web Service:
   - **Runtime:** Node
   - **Root Directory:** `auth-backend`
   - **Build Command:** `npm install`
   - **Start Command:** `node server.js`
2. Add environment variables:
   | Key | Value |
   |---|---|
   | `MONGO_URI` | Your MongoDB Atlas connection string |
   | `JWT_SECRET` | A strong random secret (e.g. 32+ char string) |
   | `JWT_EXPIRES_IN` | `7d` |
   | `MODEL_API_URL` | URL from step 1 (e.g. `https://leafscan-model.onrender.com`) |
   | `CLIENT_URL` | Your Netlify URL (add after step 3) |
3. Note the deployed URL, e.g. `https://leafscan-auth.onrender.com`

---

## 3. Deploy Frontend to Netlify

1. In `frontend/.env`, set:
   ```
   REACT_APP_API_URL=https://leafscan-auth.onrender.com/api
   ```
   Or set this as a Netlify **Environment Variable** in the dashboard (preferred).

2. Connect your GitHub repo on [netlify.com](https://netlify.com):
   - **Base directory:** `frontend`
   - **Build command:** `npm run build`
   - **Publish directory:** `frontend/build`
   - **Environment variable:** `REACT_APP_API_URL` = `https://leafscan-auth.onrender.com/api`

3. After deploy, go back to Render → auth-backend → update `CLIENT_URL` to your Netlify URL.

---

## Local Development

### Flask backend
```bash
cd backend
pip install -r requirements.txt
python app.py
# Runs on http://localhost:10000
```

### Auth backend
```bash
cd auth-backend
cp .env.example .env   # fill in values
npm install
node server.js
# Runs on http://localhost:5001
```

### Frontend
```bash
cd frontend
# set REACT_APP_API_URL=http://localhost:5001/api in frontend/.env
npm install
npm start
# Runs on http://localhost:3000
```

---

## Bugs Fixed (from original)

1. **`app.py` crashed**: Mixed Flask with FastAPI's `CORSMiddleware` — Flask can't use FastAPI middleware. Replaced with `flask-cors`.
2. **Wrong model path**: `best.pt` was being looked up in project root; moved to `yolo_model/best.pt` where it actually lives.
3. **Wrong YOLO API**: Used `torch.hub.load` (YOLOv5 hub API) but file uses `ultralytics` package. Switched to `from ultralytics import YOLO`.
4. **Missing `package.json`**: `auth-backend` had no `package.json` so `npm install` failed on Render.
5. **Missing `server.js`**: `auth-backend` had routes/models but no entry point. Created `server.js`.
6. **`node-fetch` import**: Used `fetch` in `scans.js` without importing it (Node 18 has native fetch, but added `node-fetch` as dep for safety).
7. **Bloated `requirements.txt`**: Had 100+ packages including Jupyter, dev tools. Replaced with minimal set of 9 packages.
8. **Missing Netlify `_redirects`**: React SPA returns 404 on page refresh without this.
9. **Missing `netlify.toml`**: Added for zero-config Netlify deploy.
10. **`CLIENT_URL` CORS**: Hardened to support comma-separated origins.
