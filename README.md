# ⚡ SpeedTest — Production Internet Speed Measurement Platform

A premium, production-grade internet speed test web application measuring **download speed**, **upload speed**, **ping latency**, and **jitter** with real-time visualization.

---

## 📁 Project Structure

```
speedtest/
├── backend/
│   ├── src/
│   │   ├── routes/
│   │   │   ├── download.js     # GET /api/download — streams random binary data
│   │   │   ├── upload.js       # POST /api/upload — receives upload payload
│   │   │   ├── ping.js         # GET /api/ping — latency measurement
│   │   │   └── info.js         # GET /api/info — server/client metadata
│   │   ├── ws/
│   │   │   └── wsHandler.js    # WebSocket coordination server
│   │   ├── utils/
│   │   │   ├── generateData.js # Efficient random binary data generator
│   │   │   └── logger.js       # Structured JSON logger
│   │   └── server.js           # Express entry point
│   ├── Dockerfile
│   ├── package.json
│   └── .env.example
│
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── SpeedGauge.jsx      # SVG circular animated gauge
│   │   │   ├── LiveGraph.jsx       # Real-time Recharts area chart
│   │   │   ├── MetricCards.jsx     # Download/Upload/Ping/Jitter cards
│   │   │   ├── ServerCard.jsx      # Server location + client IP
│   │   │   ├── PhaseIndicator.jsx  # Test phase progress steps
│   │   │   ├── StartButton.jsx     # Animated Start/Stop CTA
│   │   │   └── ThemeToggle.jsx     # Dark/light mode switcher
│   │   ├── hooks/
│   │   │   ├── useSpeedTest.js     # State machine orchestrating all phases
│   │   │   └── useWebSocket.js     # WS connection with auto-reconnect
│   │   ├── utils/
│   │   │   └── speedTest.js        # Core measurement engine (download/upload/ping)
│   │   ├── App.jsx                 # Root layout component
│   │   ├── main.jsx                # React entry point
│   │   └── index.css               # Tailwind + custom CSS variables + animations
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── vite.config.js
│   ├── tailwind.config.js
│   └── package.json
│
├── docker-compose.yml
└── README.md
```

---

## 🚀 Quick Start

### Prerequisites
- Node.js ≥ 20
- npm ≥ 9
- Docker + Docker Compose (for containerized deployment)

### Development

```bash
# 1. Start backend
cd backend
cp .env.example .env
npm install
npm run dev          # → http://localhost:3001

# 2. Start frontend (new terminal)
cd frontend
npm install
npm run dev          # → http://localhost:5173
```

### Docker (Production)

```bash
# Build and start everything
docker-compose up --build -d

# View logs
docker-compose logs -f

# Stop
docker-compose down
```

Frontend served at **http://localhost** · Backend API at **http://localhost:3001**

---

## ⚙️ Environment Variables

### Backend (`backend/.env`)

| Variable               | Default                    | Description                          |
|------------------------|----------------------------|--------------------------------------|
| `PORT`                 | `3001`                     | HTTP server port                     |
| `NODE_ENV`             | `development`              | Environment mode                     |
| `CORS_ORIGIN`          | `http://localhost:5173`    | Allowed frontend origins (CSV)       |
| `MAX_UPLOAD_SIZE`      | `52428800`                 | Max upload bytes (50 MB)             |
| `RATE_LIMIT_MAX`       | `100`                      | Requests per window per IP           |
| `RATE_LIMIT_WINDOW_MS` | `60000`                    | Rate limit window (ms)               |
| `LOG_LEVEL`            | `info`                     | `error`/`warn`/`info`/`debug`        |
| `SERVER_CITY`          | `Local`                    | Display name in server card          |
| `SERVER_COUNTRY`       | `US`                       | Server country code                  |

### Frontend (`frontend/.env`)

| Variable        | Default                  | Description               |
|-----------------|--------------------------|---------------------------|
| `VITE_API_URL`  | *(empty = relative)*     | Backend base URL          |
| `VITE_WS_URL`   | *(auto-detected)*        | WebSocket URL             |

---

## 📐 Speed Calculation Logic

### Download Mbps
```
Mbps = (totalBytes × 8) / elapsedSeconds / 1,000,000
```
- Multiple parallel streams (1–4) to saturate bandwidth
- Progressive phases: 250 KB → 1 MB → 5 MB → 10 MB
- Weighted moving average of recent samples
- AbortController cancels immediately on stop

### Upload Mbps
- Generates in-browser Uint8Array blobs (random, non-compressible)
- POSTs in parallel streams to backend
- Server discards data; measures transfer duration
- Same Mbps formula applied on client side

### Ping & Jitter
```
Ping   = median(RTT samples)    // HEAD requests, 12 samples
Jitter = mean |RTT[n] - RTT[n-1]|  // Mean absolute deviation
```
- First request discarded (TCP warm-up)
- Cache-busting query params on every request

---

## 🌐 Deployment

### Option A: Docker Compose (Recommended)
```bash
# Edit docker-compose.yml: update CORS_ORIGIN to your domain
docker-compose up -d --build
```

### Option B: Manual (PM2 + Nginx)
```bash
# Backend
cd backend && npm ci --omit=dev
pm2 start src/server.js --name speedtest-api -i max

# Frontend
cd frontend && npm run build
# Copy dist/ to your web root, configure nginx to proxy /api and /ws
```

### Option C: Cloud Platforms
- **Railway / Render**: Point backend to `backend/` directory, set env vars
- **Vercel**: Deploy frontend (static), connect to hosted backend
- **AWS ECS / GCP Cloud Run**: Use provided Dockerfiles

---

## 🔒 Security Best Practices

1. **Rate limiting** — 100 req/min per IP on all `/api/*` routes
2. **Helmet.js** — sets 11 security HTTP headers automatically
3. **CORS allow-list** — only configured origins accepted
4. **Non-root Docker user** — backend container runs as UID 1001
5. **Max upload size** — rejects payloads over 50 MB to prevent DoS
6. **No sensitive data logging** — only bytes/timing metadata logged
7. **HTTPS in production** — terminate TLS at load balancer (ELB, Cloudflare, etc.)
8. **Content Security Policy** — add to nginx for production deployments

---

## 📈 Performance Tuning

### Backend
- **Node.js cluster mode** (`pm2 -i max`) — utilizes all CPU cores
- **`compression` disabled for download route** — prevents CPU waste on incompressible data
- **Pre-allocated random data pool** — avoids GC pressure from repeated `crypto.randomBytes`
- **Streaming responses** — no full-file buffering; chunks flow directly to client
- **Keepalive connections** — nginx upstream `keepalive 32`

### Frontend
- **Recharts `isAnimationActive={false}`** — critical for 60fps live graph updates
- **requestAnimationFrame gauge animation** — smooth needle without layout thrash
- **Weighted moving average** — stable Mbps display without jitter
- **React code splitting** — recharts in separate chunk (`manualChunks`)

### Network
- **nginx buffering off** for `/api/download` — true streaming
- **nginx gzip** on static assets only
- **HTTP/1.1 keepalive** between nginx and backend

---

## 📊 Scaling Advice

| Traffic Level | Architecture |
|---------------|-------------|
| <1K users/day | Single Docker Compose host |
| 1K–50K/day    | PM2 cluster + PostgreSQL result storage |
| 50K+/day      | Kubernetes + horizontal pod autoscaling + CDN for download chunks |
| Global        | Multi-region test servers + GeoDNS routing + CloudFront/Cloudflare |

**For maximum accuracy at scale:**
- Host large test files (50 MB+) on a CDN edge close to users
- Add multiple test server locations, auto-select nearest via GeoDNS or latency probe
- Use Redis to cache server metadata and rate-limit state across instances

---

## 🧪 Testing Strategy

```bash
# Backend unit tests (Jest + Supertest)
cd backend && npm test

# Load testing (requires k6)
k6 run --vus 50 --duration 30s k6/load-test.js

# Frontend component tests (Vitest + Testing Library)
cd frontend && npm test
```

**What to test:**
- `generateBuffer(n)` returns exact byte count
- Download route sets `Content-Length` correctly
- Upload route rejects `>50MB` payloads with 413
- `calcMbps(bytes, ms)` formula accuracy
- Gauge angle calculation at boundary values (0, 500, 1000 Mbps)
- AbortController stops all in-flight fetches

---

## 🎨 UI Design System

| Token      | Value       | Usage           |
|------------|-------------|-----------------|
| `--accent` | `#00d4ff`   | Download, primary |
| Upload     | `#a78bfa`   | Upload phase    |
| Ping       | `#34d399`   | Latency/success |
| Jitter     | `#fbbf24`   | Warning         |
| Error      | `#f87171`   | Stop/error      |
| Font: Display | Syne 800 | Values, headings |
| Font: Mono    | Space Mono | Labels, units  |
| Font: Body    | DM Sans    | Descriptions    |

---

## 📜 License

MIT — free for personal and commercial use.
