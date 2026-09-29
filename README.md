# ReachInbox Email Scheduler

A full-stack email scheduling application that accepts email send requests, schedules them using BullMQ and Redis, delivers emails through SMTP, and provides a dashboard for monitoring scheduled, sent, failed, and queued emails.

The application stores email and campaign records in PostgreSQL and BullMQ queue state in Redis. The API and BullMQ worker run as separate processes. BullMQ is designed to recover scheduled jobs from Redis after API or worker process restarts, provided the Redis datastore itself retains its data. The current free Render Key Value deployment is in-memory and does not provide Redis persistence across datastore restarts.

---

## 1. Features

### Authentication

- Google OAuth 2.0 login.
- Redis-backed, single-use OAuth state.
- HttpOnly Redis-backed sessions.
- Authenticated API requests are user-scoped.
- Users can log out and revoke their session.

### Email Scheduling

- Compose and schedule emails for a future time.
- Supports:
  - Sender selection
  - Campaign name
  - Subject
  - Body
  - Multiple recipients
  - CSV/TXT recipient import
  - Minimum send delay
  - Hourly sending limit
- Duplicate recipients are removed.
- Invalid email addresses are reported.
- Scheduling creates persistent database records and BullMQ jobs.

### Queue and Worker

- BullMQ is used for persistent job scheduling.
- Redis stores BullMQ queue state and delayed jobs.
- Worker concurrency is configurable.
- Queue state is stored in Redis, so BullMQ can recover jobs after API or worker restarts while Redis retains its data.
- Email jobs move through states such as:
  - `SCHEDULED`
  - `PROCESSING`
  - `SENT`
  - `FAILED`

### Rate Limiting

- Sender-level hourly limits.
- Minimum spacing between emails from the same sender.
- Redis Lua-based atomic reservation prevents concurrent workers from bypassing limits.
- A spacing deferral does not consume hourly quota.
- Deferred jobs remain scheduled and are rescheduled through BullMQ.
- Rate-limit notifications can be sent to Slack.

### Email Delivery

- SMTP delivery through Ethereal Email for development/testing.
- Successful delivery is persisted before Elasticsearch indexing.
- SMTP failures are recorded as failed jobs with failure reasons.
- Elasticsearch failure does not cause an already-sent email to be sent again.

### Slack Integration

- Per-user Slack OAuth connection.
- Slack connection status can be viewed from the integrations section.
- Rate-limit notifications are sent to a configured Slack channel.
- Slack notifications do not cause an email job to fail if Slack is unavailable.
- Slack OAuth uses the `chat:write` scope.

### Elasticsearch

- Successful email records are indexed in Elasticsearch.
- Search supports subject and recipient text.
- Search results are scoped to the authenticated user.
- Duplicate indexing of the same email job is prevented using its UUID.

### Queue Dashboard

Bull Board is available at:

`http://localhost:5000/admin/queues`

The dashboard is:

- Authenticated
- Read-only
- Connected to the existing `email-scheduler` BullMQ queue

It displays queue states such as:

- Waiting
- Active
- Delayed
- Completed
- Failed

### Feature coverage

| Area | Implemented functionality |
|---|---|
| Backend | Express API; Google OAuth and Redis-backed sessions; user-scoped PostgreSQL/Prisma records; SMTP sender management; campaign and email scheduling; BullMQ delayed jobs; separate worker; sender hourly limits and spacing; email status/history; Slack integration; Elasticsearch search; authenticated read-only Bull Board dashboard |
| Frontend | Google sign-in; dashboard and email statistics; compose/schedule with sender selection; CSV/TXT recipient import; scheduled/sent/failed email views; search; campaign views; sender settings; Slack integration status; logout |
| Infrastructure / deployment | Vercel frontend and Render API/PostgreSQL/Redis are deployed. Ethereal SMTP sender setup is configured in the app. Production BullMQ worker hosting is pending; see [Current Production Deployment](#current-production-deployment). |

---

## 2. Architecture

```text
                         ┌─────────────────────┐
                         │      Frontend       │
                         │   localhost:5173    │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │   Express API       │
                         │   localhost:5000    │
                         └──────┬──────┬───────┘
                                │      │
                 ┌──────────────┘      └──────────────┐
                 ▼                                    ▼
        ┌─────────────────┐                  ┌─────────────────┐
        │   PostgreSQL    │                  │      Redis      │
        │ Users/Campaigns │                  │ BullMQ/Sessions │
        │   Email Jobs    │                  │ Rate Limiting   │
        └─────────────────┘                  └────────┬────────┘
                                                       │
                                                       ▼
                                             ┌─────────────────┐
                                             │  BullMQ Worker  │
                                             │ separate process│
                                             └────────┬────────┘
                                                      │
                                                      ▼
                                             ┌─────────────────┐
                                             │   SMTP/Ethereal │
                                             └─────────────────┘
                                                      │
                                                      ▼
                                                   Email

        Successful delivery
                 │
                 ▼
        ┌─────────────────┐
        │  Elasticsearch  │
        │ Email search    │
        └─────────────────┘

        Rate-limit events
                 │
                 ▼
        ┌─────────────────┐
        │      Slack      │
        │   Notification  │
        └─────────────────┘
```

---

## 3. Technology Stack

### Frontend
- HTML
- CSS
- JavaScript

### Backend
- Node.js
- TypeScript
- Express
- Prisma

### Data and Infrastructure
- PostgreSQL
- Redis
- BullMQ
- Elasticsearch

### Authentication / Integrations
- Google OAuth 2.0
- Slack OAuth
- Ethereal Email SMTP

### Queue Dashboard
- Bull Board

### Development
- Docker Compose
- npm
- TypeScript
- tsx

---

## 4. Project Structure

```text
ReachInbox-Scheduler/
│
├── backend/
│   ├── src/
│   │   ├── lib/
│   │   ├── routes/
│   │   ├── services/
│   │   ├── workers/
│   │   └── scripts/
│   ├── prisma/
│   │   ├── migrations/
│   │   └── schema.prisma
│   ├── package.json
│   └── .env.example
│
├── frontend/
│   ├── src/
│   ├── test/
│   └── package.json
│
├── docker-compose.yml
└── README.md
```

---

## 5. Prerequisites

Install:

- Node.js
- npm
- Docker Desktop (optional if PostgreSQL, Redis, and Elasticsearch are available another way)
- Git

The project uses Docker Compose for the local PostgreSQL, Redis, and Elasticsearch services.

---

## 6. Start Infrastructure

From the project root:

```bash
docker-compose up -d
```

Check the containers:

```bash
docker-compose ps
```

The backend requires PostgreSQL and Redis. Elasticsearch is optional for best-effort email indexing and search. The supplied Compose file starts all three services for local development:

- PostgreSQL
- Redis
- Elasticsearch

The local Redis service is configured with:

```text
maxmemory-policy noeviction
```

This avoids Redis evicting queue keys when memory is full. The Compose file uses named volumes for local data.

---

## 7. Backend Setup

Open a terminal:

```bash
cd backend
npm install
```

Create:

```text
backend/.env
```

using:

```text
backend/.env.example
```

as the template.

Apply the checked-in Prisma migrations to the local database:

```bash
npx prisma migrate dev
```

`prisma.config.ts` reads the connection string from `DATABASE_URL`. `npm run build` runs `prisma generate` and then TypeScript compilation.

Build the backend:

```bash
npm run build
```

Start the development server:

```bash
npm run dev
```

Backend:

`http://localhost:5000`

Health check:

`http://localhost:5000/health`

Production API start command (used by the Render web service):

```bash
npm start
```

This starts the API only. It does not start the worker.

---

## 8. Start the Worker

Open another terminal:

```bash
cd backend
npm run worker
```

The development worker processes scheduled BullMQ email jobs. The API and worker must run as separate processes. For a built production worker, use:

```bash
npm run build
npm run worker:prod
```

Keep the worker running while testing scheduled email delivery.

---

## 9. Frontend Setup

Open another terminal:

```bash
cd frontend
npm install
npm run dev
```

Frontend:

`http://localhost:5173`

The frontend chooses `http://localhost:5000` when opened on localhost; otherwise it uses the deployed Render API URL embedded in `frontend/src/lib/api.js`. Build the static frontend with `npm run build` (output: `frontend/dist/`).

---

## 10. Environment Variables

The backend reads configuration from `backend/.env` locally and from the hosting service's environment in production. The following names are used in source code; provide real values only in a private environment, never in this README or Git.

Important variables include:

### Database
- `DATABASE_URL` — required by the API and worker; Prisma uses it for PostgreSQL.

### Redis
- `REDIS_URL` — required by the API and worker; accepts `redis://` or `rediss://`.

### Elasticsearch
- `ELASTICSEARCH_URL` — optional; successful email indexing/search uses it when configured.

### Google OAuth
- `GOOGLE_CLIENT_ID`
- `GOOGLE_CLIENT_SECRET`
- `GOOGLE_REDIRECT_URI`

For local development, the Google callback is:

`http://localhost:5000/api/auth/google/callback`

### Frontend
```text
FRONTEND_URL=http://localhost:5173
```
`FRONTEND_URL` is the allowed browser origin and OAuth return destination; set it to the deployed frontend origin in production.

### Slack OAuth
- `SLACK_CLIENT_ID`
- `SLACK_CLIENT_SECRET`
- `SLACK_REDIRECT_URI`
- `SLACK_NOTIFICATION_CHANNEL_ID`

Local Slack callback:

`http://localhost:5000/api/slack/callback`

### SMTP

- `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASSWORD`, and `SMTP_SECURE` are supported fallback settings by the mailer. Normal scheduled worker delivery uses the SMTP details saved on the selected sender record in the app.

### Worker

- `WORKER_CONCURRENCY` — optional positive integer; defaults to `5`.
- `PORT` — optional API listen port; defaults to `5000`.

Google and Slack variables above are required only when those integrations are used. SMTP fallback variables are not needed when each sender is configured in Settings.

Example names and placeholder-only values (replace locally; never commit real credentials):

```dotenv
PORT=5000
DATABASE_URL=<your-postgresql-connection-url>
REDIS_URL=<your-redis-connection-url>
ELASTICSEARCH_URL=<your-elasticsearch-url>
FRONTEND_URL=http://localhost:5173
GOOGLE_CLIENT_ID=<your-google-client-id>
GOOGLE_CLIENT_SECRET=<your-google-client-secret>
GOOGLE_REDIRECT_URI=http://localhost:5000/api/auth/google/callback
SLACK_CLIENT_ID=<your-slack-client-id>
SLACK_CLIENT_SECRET=<your-slack-client-secret>
SLACK_REDIRECT_URI=http://localhost:5000/api/slack/callback
SLACK_NOTIFICATION_CHANNEL_ID=<your-slack-channel-id>
WORKER_CONCURRENCY=5
SMTP_HOST=smtp.ethereal.email
SMTP_PORT=587
SMTP_USER=<your-ethereal-username>
SMTP_PASSWORD=<your-ethereal-password>
SMTP_SECURE=false
```

---

## 10.1 Ethereal Email Testing

Create a test mailbox at [Ethereal Email](https://ethereal.email/) and use the generated SMTP credentials in the app's **Settings → Sender accounts → Add an SMTP sender** form:

```text
SMTP host: smtp.ethereal.email
SMTP port: 587
SMTP username: <your-ethereal-username>
SMTP password: <your-ethereal-password>
```

Use the mailbox credentials Ethereal generated for you; do not add them to source control or this README. The application stores sender SMTP credentials on the backend and omits the password from sender API responses. Ethereal captures test messages instead of delivering them to real recipients; use the preview link or Ethereal's web interface to inspect them.

---

## 11. Google OAuth Setup

Create a Google OAuth client for a web application.

For local development configure:

**Authorized JavaScript origin:**
`http://localhost:5173`

and:

**Authorized redirect URI:**
`http://localhost:5000/api/auth/google/callback`

The application requests:

- `openid`
- `email`
- `profile`

Google access tokens are used for profile lookup and are not stored as persistent application credentials.

---

## 12. Slack Setup

Create a Slack application and enable OAuth.

The application requires:

- `chat:write`

Configure the redirect URI:

`http://localhost:5000/api/slack/callback`

Set:

`SLACK_NOTIFICATION_CHANNEL_ID`

to the ID of the Slack channel where notifications should be posted.

The Slack app must be installed in the workspace and have access to the configured channel.

---

## 13. Scheduling an Email

1. The frontend sends the authenticated scheduling request to the Express API.
2. The API creates the campaign and its `EmailJob` rows in a PostgreSQL transaction.
3. It enqueues one BullMQ `send-email` job per email using the email job UUID and calculated delay in the `email-scheduler` queue.
4. Redis holds BullMQ's delayed queue state until the scheduled time.
5. The separate worker consumes due jobs, loads the email, sender, and campaign from PostgreSQL, reserves the sender's rate-limit slot in Redis, and sends through that sender's SMTP settings.
6. The worker records `SENT` or `FAILED` in PostgreSQL. Elasticsearch indexing and Slack notification are separate best-effort integrations.

In the UI, sign in with Google, configure/select an SMTP sender, open Compose, enter recipients (or upload CSV/TXT), subject and body, choose the start time, minimum delay, and hourly limit, then click **Schedule emails**.

The backend creates the campaign and email jobs atomically in PostgreSQL and enqueues deterministic BullMQ jobs.

After successful scheduling:

- The compose form is reset.
- The recipient list is cleared.
- Scheduled email data is refreshed.

If scheduling fails, the draft remains available for retry.

---

## 14. Recipient Import

CSV and TXT recipient files are processed in the browser.

The parser:

- Normalizes addresses.
- Removes duplicates.
- Ignores blank content.
- Counts malformed addresses.
- Shows the valid unique recipient count before scheduling.

---

## 15. Rate Limiting and Send Spacing

The worker reserves sender capacity in Redis before SMTP delivery.

Two limits are enforced:

### Hourly limit

The campaign's `hourlyLimit` controls the maximum number of sends allowed for the sender during a UTC-hour window.

### Minimum delay

The campaign's `delayMs` specifies the minimum spacing between actual sends in milliseconds.

Redis performs the reservation atomically so concurrent workers cannot bypass the limits.

When a job cannot send immediately:

```text
EmailJob → SCHEDULED
        ↓
BullMQ delayed job
        ↓
Future processing
```

A deferral does not count as a failed attempt.

---

## 16. Slack Rate-Limit Notifications

When an hourly sender limit is reached, the worker can send a Slack notification.

Notifications are deduplicated using a Redis sender/hour key so concurrent rate-limit events do not produce multiple alerts for the same sender in the same UTC hour.

Minimum-delay deferrals do not generate Slack rate-limit alerts.

A Slack API failure does not fail or retry the email job.

---

## 17. Email Status and History

The application tracks:

- `SCHEDULED`
- `PROCESSING`
- `SENT`
- `FAILED`

### Scheduled Emails

Displays:

- Scheduled jobs
- Processing jobs

### Sent Emails

Displays successful delivery history.

### Failed Emails

Displays:

- Recipient
- Subject
- Failure time
- Failure reason

Email list queries are authenticated and user-scoped.

Client-provided user IDs are not trusted for ownership checks.

---

## 18. Elasticsearch Search

Successful email delivery is indexed in Elasticsearch.

Search can use:

- Recipient
- Subject

Search requests require authentication and are scoped to the authenticated user.

Email jobs are indexed using their UUID, preventing duplicate documents for the same job.

If Elasticsearch is unavailable after SMTP delivery:

```text
SMTP delivery → SENT
             ↓
      Elasticsearch failure
```

The email remains `SENT` and is not delivered again.

---

## 19. Bull Board Queue Dashboard

Open:

`http://localhost:5000/admin/queues`

The dashboard displays the existing:

`email-scheduler`

BullMQ queue.

The dashboard is authenticated and read-only.

---

## 20. Restart Persistence

Scheduled BullMQ jobs are stored in Redis.

The application was verified with the following flow:

```text
Schedule email
      ↓
Stop backend + worker
      ↓
Redis remains running
      ↓
Restart backend + worker
      ↓
BullMQ recovers scheduled job
      ↓
Worker processes email
      ↓
Email delivered
      ↓
Slack notification received
      ↓
Email appears in Sent Emails
```

BullMQ stores delayed queue state in Redis rather than only in API/worker memory. Therefore API and worker restarts do not themselves erase queue state, as long as Redis retains its data. The development Compose setup uses a named Redis volume. Persistence depends on the Redis host's plan and configuration; the current free Render Key Value deployment has persistence disabled and can lose queue data when it restarts. This means the current production deployment does not guarantee jobs survive a Redis restart.

---

## 21. Testing

### Backend build
```bash
cd backend
npm run build
```

### Frontend tests
```bash
cd frontend
npm test
```

### Backend authentication/scheduling tests
```bash
npm run auth-scheduling:test
```

### Google OAuth tests
```bash
npm run google:test
```

### Slack tests
```bash
npm run slack:test
```

### Email delay/rate-limit tests
```bash
npm run delay:test
npm run rate-limit:test
npm run campaign-scheduler:test
npm run test:idempotency
```

### Elasticsearch tests
```bash
npm run elasticsearch:test
```

### Queue dashboard tests
```bash
npm run queue-dashboard:test
```

Other backend checks available in `backend/package.json` include `npm run queue:test` and `npm run email:test` (which create test queue/email jobs). `npm run build` is the backend compile check; `npm test` in `frontend/` runs the frontend test suite.

The frontend uses Node's built-in test runner (`node --test`) for UI rendering, CSV/TXT parsing, form validation, and API client behavior. Backend package scripts provide focused runnable checks for authentication/scheduling, Google OAuth, Slack, delayed jobs, rate limiting, idempotency, Elasticsearch, and the queue dashboard. These backend scripts exercise different paths and may require local services and integration credentials; they do not represent complete automated coverage of every feature.

---

## 22. Manual Verification

The following functionality has also been manually verified during development:

- Google sign-in
- Email scheduling through the browser
- Ethereal email delivery
- Sent Emails display
- Slack OAuth connection
- Real Slack notification delivery
- Backend and worker restart recovery when Redis remains running

---

## 23. Failure Handling

### SMTP failure

SMTP errors cause the email job to be recorded as `FAILED` with the failure reason.

### Slack failure

Slack notification failures are isolated from email delivery.

A Slack failure does not cause the successfully delivered email to fail or be retried.

### Elasticsearch failure

An Elasticsearch indexing failure does not cause a successfully sent email to be sent again.

### Rate-limit deferral

Rate-limited jobs remain scheduled and are moved to a future BullMQ delay instead of being marked as failed.

---

## 24. Security Considerations

- OAuth states are stored in Redis and consumed only once.
- OAuth state has a limited lifetime.
- Sessions use HttpOnly cookies.
- User ownership is resolved server-side.
- Client-provided user IDs are not trusted for authorization.
- Slack access tokens are not exposed by the status endpoint.
- Sender API responses omit SMTP passwords; credentials are stored server-side and should be entered only through the authenticated sender settings form.
- Keep Google/Slack client secrets, PostgreSQL and Redis credentials, SMTP passwords, and all other secrets in private environment configuration.
- Do not commit `.env` files; use `backend/.env.example` as the variable-name template.
- Queue dashboard access requires authentication.
- The queue dashboard is read-only.

For production deployment, additional role-based authorization should be considered for the global queue dashboard.

---

## 25. Known Limitations

- The queue dashboard currently does not have a separate admin role. Authenticated users may view the global queue status.
- Ethereal Email is intended for development/testing rather than production email delivery.
- Real third-party OAuth credentials are required for Google and Slack integrations.
- Production deployment should use HTTPS.
- Redis should use `maxmemory-policy noeviction`; job-queue deployments should also enable durable Redis persistence.
- The production API and database are deployed, but the separate production worker is pending a continuously running host.
- Free Render Key Value does not persist its data across restarts, so production queue restart durability is limited until a persistent Redis configuration is used.

---

## Current Production Deployment

**Application implementation:** The deployed application currently uses Vercel for the frontend and Render for the API, PostgreSQL, and Key Value datastore. The production BullMQ worker requires a continuously running worker process. The local development environment includes the API and worker and has been tested end-to-end with Ethereal SMTP.

**Current free hosting limitation:** The current free Render Key Value deployment does not provide persistence across Redis datastore restarts. A persistent Redis deployment is therefore required to provide the full restart-persistence guarantee in production.

| Component | Production endpoint / state |
|---|---|
| Vercel frontend | [https://reach-inbox-scheduler-kohl.vercel.app/](https://reach-inbox-scheduler-kohl.vercel.app/) — deployed and working |
| Render backend API | [https://reachinbox-scheduler-03lq.onrender.com](https://reachinbox-scheduler-03lq.onrender.com) — deployed and working |
| Health check | [ `/health` ](https://reachinbox-scheduler-03lq.onrender.com/health) — reports API status and PostgreSQL connectivity; it does not check Redis or worker status |
| PostgreSQL | Connected to the Render backend |
| Redis / Key Value | Configured; free-plan persistence is unavailable |
| Google OAuth | Working |
| SMTP sender setup | Working; sender can be selected in Compose |
| Scheduling API | Working; jobs appear in Scheduled |
| Production BullMQ worker | Pending a suitable continuously running worker host |
| End-to-end production delivery | Pending worker deployment; do not infer delivery from successful scheduling |

The deployed frontend and backend URLs are unchanged. The backend production start command is `npm start`, which starts only the Express API. The production worker command is `npm run worker:prod`, but it is not currently running in production.

### Worker hosting limitation

The current free Render Web Service is not a continuously available worker host: it spins down after idle time and may restart. Render also does not offer a free Background Worker, and free web services block outbound SMTP ports 25, 465, and 587. The worker should be deployed separately to a host that can run a persistent Node.js process and connect to the existing production Redis and PostgreSQL. BullMQ must remain the scheduler; cron or polling is not used as a replacement.

---

## 26. Quick Start

From the project root:

```bash
docker-compose up -d
```

**Terminal 1:**
```bash
cd backend
npm install
npx prisma migrate dev
npm run dev
```

**Terminal 2:**
```bash
cd backend
npm run worker
```

**Terminal 3:**
```bash
cd frontend
npm install
npm run dev
```

Then open:

`http://localhost:5173`

Sign in with Google, configure an SMTP sender, and schedule an email.

---

## 27. Project Status

| Feature | Status |
|---|---|
| Vercel frontend | Deployed and working |
| Render backend API | Deployed and working |
| PostgreSQL | Connected |
| Redis / Key Value | Configured; free-plan restart persistence unavailable |
| Google OAuth | Working |
| SMTP sender configuration | Working |
| Scheduling API / jobs in Scheduled | Working |
| BullMQ worker code | Implemented (`npm run worker:prod`) |
| Production BullMQ worker | Pending hosting |
| End-to-end production email delivery | Pending worker deployment |

---

## License

This project was developed as part of the ReachInbox email scheduler assignment.
