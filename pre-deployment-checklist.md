# Part 5: Pre-Deployment Checklist

_Tool behavior changes between versions, so check the official docs for your exact Prisma, Mongoose, and Express versions._

---

## English Answer

### Security

- [ ] **Secrets:** Keep all keys in environment variables. Add `.env` to `.gitignore`, commit only a `.env.example` with placeholders, and use a fresh, long `JWT_SECRET` in production. If a secret is ever committed, rotate it immediately.
- [ ] **CORS:** Allow only your real frontend origin(s). Never use `*` in production.
- [ ] **Security headers:** Add `app.use(helmet())` early in the middleware stack. It sets security-related HTTP headers such as Content-Security-Policy and Strict-Transport-Security.
- [ ] **Hide fingerprints:** Add `app.disable('x-powered-by')`.
- [ ] **Rate limiting:** Use `express-rate-limit`, with stricter limits on login and register routes.
- [ ] **Proxy setting:** On Render, add `app.set('trust proxy', 1)` right after creating the app. Without it, the limiter cannot see real client IPs and treats everyone as one user.
- [ ] **Input and cookies:** Validate request bodies before business logic. Use `httpOnly`, `secure`, and `sameSite` on cookies.
- [ ] **Dependencies:** Run `npm audit` and update vulnerable packages.

### Database Management

- [ ] **Prisma:** Use `prisma migrate dev` locally only and `prisma migrate deploy` in production, ideally as part of the build/CI pipeline. `migrate deploy` applies existing migrations only; it never creates them or resets the database.
- [ ] **Avoid `db push` in production for SQL databases:** it bypasses migration history and offers no rollback. (For MongoDB with Prisma, `db push` is the official workflow.)
- [ ] Commit `prisma/migrations/`, never edit an already-applied migration, and back up production data before migrating.
- [ ] Make sure the `prisma` CLI is available at build time. If your platform prunes `devDependencies`, move `prisma` to `dependencies`.
- [ ] **Mongoose indexes:** `autoIndex` is on by default. To manage indexes deliberately in production, set `mongoose.set('autoIndex', process.env.NODE_ENV !== 'production')` and build indexes explicitly with `Model.syncIndexes()` during deployment. Be careful: `syncIndexes()` drops indexes that exist in MongoDB but not in your schema.
- [ ] Remember that `unique: true` is enforced by an index. If the index is not built, duplicates are not blocked, and building it fails if duplicates already exist.
- [ ] Index the fields you frequently filter or sort by (for example `email` or `userId`).

### Error Handling & Logs

- [ ] Set `NODE_ENV=production`; otherwise Express includes stack traces in error responses.
- [ ] Add a central error-handling middleware after all routes: log the full error server-side, and return only a generic message (e.g. `"Internal Server Error"`) to the client.

  ```js
  app.use((err, req, res, next) => {
    console.error(err.stack); // stays in server logs only
    res
      .status(err.statusCode || 500)
      .json({ message: "Internal Server Error" });
  });
  ```

- [ ] Add a custom 404 handler and pass async errors to it with `next(err)`.
- [ ] Never log passwords, API keys, or personal data.
- [ ] **Where to read logs on Render:** Dashboard, then your service, then **Logs**. Render captures everything written to stdout and stderr. You can live-tail in the dashboard or with the CLI (`render logs`), and view build output on each deploy's page. Retention depends on your plan, so stream logs to a provider like Better Stack or Datadog if you need long-term storage. HTTP request logs require a Pro workspace or higher.

### Environment Setup

- [ ] Keep `nodemon`, test runners, and linters in `devDependencies`; install production dependencies only where possible (`npm ci --omit=dev`).
- [ ] Anything needed at build or migration time (e.g. `prisma`, `typescript`) must be in `dependencies`.
- [ ] Use `node` (not `nodemon`) in the `start` script.
- [ ] Remove hardcoded `localhost` URLs, test accounts, seed data, and leftover debug `console.log` calls.
- [ ] Commit `package-lock.json` and pin the Node version in `engines`.
- [ ] Set `NODE_ENV=production` and all secrets in the hosting dashboard, not in the repo.

---
