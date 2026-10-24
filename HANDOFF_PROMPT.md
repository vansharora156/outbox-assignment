Continue a project I (Vansh Arora) started. Upload the attached zip (reachinbox-scheduler.zip) and read README.md first.

Project: ReachInbox assignment - email scheduler service + dashboard.
Stack: TypeScript, Express, BullMQ + Redis (delayed jobs, NO cron), Postgres, Ethereal SMTP, Elasticsearch, bull-board, React + Vite + Tailwind.

Already done (backend/ and frontend/ both typecheck and frontend builds):
- Google OAuth login (JWT cookie), Slack OAuth (incoming-webhook) + notify on hourly limit
- POST /api/emails/schedule, scheduled/sent lists, Elasticsearch search
- BullMQ worker: concurrency, global limiter (min delay), Redis Lua hourly counter per sender, reschedule to next hour on limit, atomic DB claim for idempotency, boot-time reconcile
- React dashboard: login, tabs, compose modal with CSV email-count, tables with loading/empty/error, toasts

Not verified: I could not run Docker/Redis/Postgres or real Google/Slack OAuth in the build sandbox, so nothing has been run end to end.

Your tasks:
1. Run docker compose up -d, start backend and frontend, and fix any runtime bugs (SQL, BullMQ DelayedError flow, ES mapping, cookie/proxy issues).
2. Test: schedule with low hourly limit -> reschedule + Slack message; restart server mid-schedule -> no loss/duplicate.
3. Match the frontend to my Figma: https://www.figma.com/design/kOTwGlESjijCYnMgtHfvfU/Outbox-Labs-Assignment?node-id=59-4050&p=f&m=dev (I will paste screenshots).
4. Add a short demo checklist and finish README.
Keep the hard constraints: no cron, no duplicate sends, config via env only.
