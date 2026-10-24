# PeerNet - Peer Learning & Doubt Resolution Network
Full-stack peer-learning platform for asking doubts, matching with helpers, and sharing answers.
Full Stack Development-II (24CSP-337) capstone project.
**Frontend:** React (Vite) | **Backend:** Spring Boot 3 (Java 25) | **DB:** PostgreSQL (H2 for quick local run) | **Cache:** Redis | **Real-time:** WebSocket (STOMP) | **Auth:** JWT + RBAC

## 1. What you need to install (once)
| Tool | Version | Check with |
|---|---|---|
| JDK | 25 | `java -version` |
| Maven | 3.9+ | `mvn -v` |
| Node.js | 20 or newer | `node -v` |
| VS Code extensions | "Extension Pack for Java", "Spring Boot Extension Pack", "ES7+ React snippets" (optional), "Lombok Annotations Support" | |

## 2. Run it locally in VS Code
Open the **peernet** folder in VS Code (`File > Open Folder`). Open two terminals (`Terminal > New Terminal`).

**Terminal 1 - backend**
```
cd backend
```
Set a local-only JWT signing key in this terminal (PowerShell). To load sample data for a local demo, also set the seeding variables:
```
$env:JWT_SECRET="local-only-signing-key-change-before-deployment-123456"
$env:SEED_DEMO_DATA="true"
$env:ADMIN_PASSWORD="Admin@123"
mvn spring-boot:run
```
Wait for "Started PeerNetApplication". It runs on http://localhost:8080 with an in-memory H2 database (no database install needed). `JWT_SECRET` is required and must be at least 32 bytes. Demo accounts and sample content are created only when `SEED_DEMO_DATA=true`; seeding requires an `ADMIN_PASSWORD`.

**Terminal 2 - frontend**
```
cd frontend
npm ci
npm run dev
```
Open http://localhost:5173

### Demo accounts
| Role | Email | Password |
|---|---|---|
| Student | student@peernet.com | Student@123 |
| Student | diya@peernet.com | Student@123 |
| Student | rohan@peernet.com | Student@123 |
| Mentor | mentor@peernet.com | Mentor@123 |
| Admin (local default) | admin@peernet.com | Admin@123 |

These accounts exist only when local demo seeding is enabled as shown above. Production deployments do not seed demo users by default. Never enable demo seeding in production; configure a secure `JWT_SECRET` and other production environment variables.

### Quick demo flow (for the viva)
1. Login as **rohan@peernet.com** (open a second browser/incognito for another user), ask a doubt in "Java" with *anonymous* ticked.
2. Login as **student@peernet.com**: a live notification pops up (WebSocket) and the doubt shows in "Matched for me".
3. Student posts an answer (+5 pts). Rohan opens the doubt and clicks **Accept answer** (+15 pts, badge "Problem Solver").
4. Login as mentor: Mentor Panel, verify an answer (+10 pts).
5. Login as admin: Admin > Matching shows the weights, Stats shows the chart, Reports lets you moderate. Open `GET /api/admin/doubts/{id}/matching` in Postman to show the score breakdown of the algorithm.

## 3. Run the unit tests
```
cd backend
mvn test
```

## 4. Deploy online (free tier) - required for evaluation
Order: **Database -> Backend -> Frontend -> connect them**. Push the whole project to a public GitHub repository first.

### 4.1 Database + Backend on Render (https://render.com)
1. New > **PostgreSQL** (free). After it is created copy: Hostname, Port, Database, Username, Password.
2. New > **Web Service** > connect your GitHub repo. Settings:
   - Root Directory: `backend`
   - Runtime: **Docker** (it uses `backend/Dockerfile`)
3. Add these **Environment Variables**:

| Key | Value |
|---|---|
| `DB_URL` | `jdbc:postgresql://<hostname>:5432/<database>` (use the *internal* hostname from Render) |
| `DB_USER` | database username |
| `DB_PASSWORD` | database password |
| `JWT_SECRET` | any long random text (at least 40 characters) |
| `ADMIN_PASSWORD` | the admin password you want |
| `CORS_ORIGINS` | your frontend URL, e.g. `https://peernet.vercel.app` (add after step 4.2, no trailing slash) |

4. Deploy. Test: open `https://<your-backend>.onrender.com/api/health` - you should see `{"status":"UP"...}`.
   (Free Render services sleep after inactivity; open the health URL a minute before the evaluation.)

### 4.2 Frontend on Vercel (https://vercel.com) or Netlify
1. New Project > import the same GitHub repo.
2. Root Directory: `frontend`. Framework: **Vite**. Build command `npm run build`, output `dist`.
3. Environment variable: `VITE_API_URL` = your Render backend URL (e.g. `https://peernet-api.onrender.com`, no trailing slash).
4. Deploy, copy the frontend URL, then put it into `CORS_ORIGINS` on Render and redeploy the backend.

### 4.3 Redis (optional but matches the synopsis)
Create a free Redis (Upstash or Render Key Value). On the Render backend add:
`CACHE_TYPE=redis`, `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`, and `REDIS_SSL=true` if the provider requires TLS.
Without these the app uses an in-memory cache and still works.

## 5. Project structure
```
peernet/
  backend/   Spring Boot (controller > service > repository), JWT security, WebSocket, scheduler, tests, Dockerfile
  frontend/  React + Vite (pages, components, API client, live notifications)
  .github/workflows/ci.yml   GitHub Actions: build + test on every push
```

## 6. Features mapped to the synopsis
| Synopsis module | Where in the code |
|---|---|
| Authentication + RBAC (Student/Mentor/Admin) | `security/`, `config/SecurityConfig`, `AuthService` |
| Anonymous doubt posting + duplicate suggestions | `DoubtService.create/similar`, `DtoMapper.doubt` |
| Smart matching (0.40/0.25/0.20/0.15) + escalation | `MatchingService`, `EscalationScheduler`, `SettingsService` |
| Answers, votes, accepted, mentor verified | `AnswerService` |
| Reputation, badges, leaderboard (cached) | `ReputationService`, `LeaderboardService` |
| Real-time notifications | `NotificationService`, `NotificationPusher`, `WebSocketConfig` |
| Knowledge base + search + pagination | `DoubtService.search`, `KnowledgeBase.jsx` |
| Mentor panel, admin dashboard, analytics, reports | `MentorController`, `AdminController`, `Admin.jsx` |
| Indexes, caching, rate limiting | `model/*` (@Index), `@Cacheable`, `RateLimiter` |
| Tests + CI + Docker | `src/test`, `.github/workflows/ci.yml`, `backend/Dockerfile` |

## 7. Small differences from the synopsis (be ready to explain)
- Notifications are queued with Spring `@Async` (background thread pool) and delivered by WebSocket; the synopsis says "Redis-backed queue".
- The leaderboard is cached with Spring Cache (Redis when `CACHE_TYPE=redis`) and computed from PostgreSQL; the synopsis says Redis sorted sets.
- Subjects are used as topics (no separate topic table).
If your evaluators compare strictly, either mention this in your report or ask for the Redis queue/sorted-set version.
