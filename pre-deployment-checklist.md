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

## 中文筆記

**比喻：** 部署前的檢查就像餐廳開幕前的安全檢查。門要上鎖（Security）、倉庫的改建要按流程（Database）、廚房失火時客人不該看到內部線路圖，但老闆要有監視器（Errors & Logs）、開店只帶必要的設備，不搬整間工作室（Environment）。

### 1. Security

| 檢查項目                 | 怎麼做                                                                                                                   | 為什麼                                                                                                                 |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| **API keys 與 secrets**  | 全部放環境變數，`.env` 加進 `.gitignore`，只提交 `.env.example`（放假值）。JWT_SECRET 在 production 用新產生的長隨機字串 | secrets 一旦推上 GitHub，刪檔案也刪不掉歷史紀錄，要立刻換掉密碼或 key                                                  |
| **CORS**                 | 只允許你前端的網址（例如 Netlify 網址），不要用 `*`                                                                      | 設成允許全部，等於任何網站都能呼叫你的 API                                                                             |
| **Security headers**     | `app.use(helmet())`                                                                                                      | 一次設定多個安全性 HTTP headers，也能減少被探測出你在用 Express 的機會                                                 |
| **隱藏技術資訊**         | `app.disable('x-powered-by')`                                                                                            | 不讓攻擊者一眼看出你的技術架構                                                                                         |
| **Rate limiting**        | 用 `express-rate-limit`，登入、註冊路由限制更嚴                                                                          | 防止暴力破解密碼與濫用                                                                                                 |
| **Render 的 proxy 設定** | 在建立 app 後加 `app.set('trust proxy', 1)`                                                                              | 不設的話 rate limiter 抓到的是 proxy 的 IP，變成全站共用同一個上限，還會出現 `ERR_ERL_UNEXPECTED_X_FORWARDED_FOR` 錯誤 |
| **輸入驗證**             | 每個路由先驗證 request body 再處理                                                                                       | 避免壞資料與 injection                                                                                                 |
| **Cookie**               | 有用 cookie 時設 `httpOnly`、`secure`（production）、`sameSite`                                                          | 降低被竊取的風險                                                                                                       |
| **套件更新**             | 跑 `npm audit`，更新有漏洞的套件                                                                                         | 舊版套件常有已知漏洞                                                                                                   |

### 2. Database Management

**Prisma (PostgreSQL)：**

| 指令                    | 用在哪              | 說明                                                                                                                        |
| ----------------------- | ------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| `prisma migrate dev`    | **只用在本機開發**  | 會建立 migration，必要時可能要求 reset 資料庫，不能用在 production                                                          |
| `prisma migrate deploy` | **Production 使用** | 只套用已經存在的 migration，不會建立新的，也不會 reset。建議放進自動化流程（例如 Render 的 build command）                  |
| `prisma db push`        | 只用在本機快速實驗  | 跳過 migration 紀錄，沒有版本歷史也無法 rollback，可能造成資料遺失。**例外：** Prisma 搭配 MongoDB 時，官方就是用 `db push` |

- `prisma/migrations/` 資料夾**要提交到 Git**。
- **不要修改已經套用過的 migration 檔案**，要改就新增一個 migration。
- 套用 migration 之前，先備份 production 資料。
- `migrate deploy` 需要 `prisma` CLI，如果它在 devDependencies 而平台會移除 devDependencies，就要把 `prisma` 移到 `dependencies`。

**Mongoose (MongoDB) 的 index：**

- Mongoose 預設 `autoIndex: true`。想在 production 謹慎管理時，常見寫法是：
  ```js
  mongoose.set("autoIndex", process.env.NODE_ENV !== "production");
  ```
- 關掉之後，要在部署時用 `Model.syncIndexes()` **刻意執行一次**。注意它會**刪除**資料庫裡有、但 schema 裡沒有的 index。
- `unique: true` 其實是靠 index 實現的。index 沒建好，重複資料就擋不住；資料庫已有重複資料時，建立 unique index 會失敗。
- 幫常用來查詢、篩選、排序的欄位建立 index。

### 3. Error Handling & Logs

**防止 stack trace 外洩：**

1. 設定 `NODE_ENV=production`。
2. 寫一個集中的 error-handling middleware（放在所有路由之後）：stack trace 只記錄在 server 端，回給使用者的只有通用訊息。
3. 自訂 404 handler。
4. async 路由用 `try/catch` 並呼叫 `next(error)`。
5. 不要把密碼、API keys、個資寫進 log。

**去哪裡看 live logs（Render）：**

| 方式                                | 說明                                                                       |
| ----------------------------------- | -------------------------------------------------------------------------- |
| **Dashboard → 你的 service → Logs** | Render 會收集 stdout 和 stderr，可以即時看、搜尋、篩選                     |
| **Live tail**                       | 在 Dashboard 或用 CLI 指令 `render logs` 即時串流                          |
| **Deploy logs**                     | 每次部署都可以另外看 build 過程的 logs                                     |
| **外部服務**                        | Render 依方案保留 logs 一段時間，要長期保存可串到 Better Stack、Datadog 等 |

HTTP request logs 要 Pro workspace 以上才有，免費方案主要看你自己 `console.log` 出來的內容。

### 4. Environment Setup

| 檢查項目                        | 怎麼做                                                                                                      |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| **只安裝需要的套件**            | `nodemon`、`jest`、`supertest`、`eslint` 放在 `devDependencies`。production 的 build 用 `npm ci --omit=dev` |
| **build 需要的工具要放對位置**  | build 或 migration 需要的套件（例如 `prisma`、`typescript`）必須在 `dependencies`                           |
| **start script 不要用 nodemon** | `"start": "node src/index.js"`，`nodemon` 只放在 `"dev"` script                                             |
| **移除本機測試設定**            | 刪掉寫死的 `localhost` 網址、測試用帳號、seed 假資料、多餘的 `console.log`                                  |
| **鎖定版本**                    | 提交 `package-lock.json`，並在 `engines` 指定 Node 版本                                                     |
| **設定 NODE_ENV**               | Render 環境變數設 `NODE_ENV=production`                                                                     |

> **重要提醒（Part 4 Prisma 版教學）：** 那份教學要你在 Render 設定 `NODE_ENV=production`，同時 build command 有跑 `npx prisma generate` 和 `migrate deploy`。如果 `prisma` 放在 `devDependencies`，build 時可能被略過而出錯，請確認 `prisma` 在 `dependencies` 裡。
