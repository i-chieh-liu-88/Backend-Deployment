# Part 4: Deploying an Express + Mongoose API with Render and MongoDB Atlas

**Stack:** Node.js/Express (hosted on **Render**) + MongoDB (hosted on **MongoDB Atlas**) + Mongoose ODM. Both have free plans that need no credit card.

**You will need:** a GitHub account, a project with an Express server that uses Mongoose, and free accounts on Render and MongoDB Atlas.

_Free-tier details and UI labels change over time. Check the official docs if something looks different._

---

## Step 1: Provision the Atlas database and get the connection string

1. Sign up at cloud.mongodb.com and create a project.
2. Click **Build a Database** and choose the **Free** (shared) option. Pick a cloud provider and a region close to where you will host on Render, then create the cluster.
3. **Create a database user.** Under **Database Access**, add a user with a username and a strong password. Save both somewhere safe.
4. **Allow network access.** Under **Network Access**, add an IP access entry. Atlas only accepts connections from allowed IP addresses, and Render's free instances do not have a fixed outbound IP. For a learning or portfolio project, the common approach is to allow `0.0.0.0/0` (any IP). Because this means the username and password are your only protection, use a strong password and never commit the connection string.
5. **Get the connection string.** Click **Connect**, choose **Drivers**, and copy the URI. It looks like this:

```
mongodb+srv://USERNAME:PASSWORD@cluster0.xxxxx.mongodb.net/mydatabase?retryWrites=true&w=majority
```

- Replace `USERNAME` and `PASSWORD` with your database user's credentials. If the password contains special characters such as `@` or `#`, URL-encode them.
- Put your database name after `.net/` (for example `mydatabase`), otherwise data may land in a default database.

6. Save it in your local `.env` file (this file stays on your computer):

```env
MONGODB_URI="mongodb+srv://USERNAME:PASSWORD@cluster0.xxxxx.mongodb.net/mydatabase?retryWrites=true&w=majority"
```

---

## Step 2: Prepare the project for deployment

1. **Keep secrets out of Git.** Make sure `.gitignore` contains:

   ```
   node_modules
   .env
   ```

   Commit a `.env.example` with placeholder values so classmates know which variables exist.

2. **Connect Mongoose using the environment variable.** Lowering `maxPoolSize` from its default of 100 keeps you well under the free cluster's connection limit:

   ```js
   const mongoose = require("mongoose");

   mongoose
     .connect(process.env.MONGODB_URI, { maxPoolSize: 10 })
     .then(() => console.log("MongoDB connected"))
     .catch((err) => console.error("MongoDB connection error:", err));
   ```

3. **Use the platform's port.** Render services must listen on the port given by the `PORT` environment variable.

   ```js
   const PORT = process.env.PORT || 3000;
   app.listen(PORT, () => console.log(`Server running on ${PORT}`));
   ```

4. **Add a start script** in `package.json` (adjust the path to your entry file):

   ```json
   "scripts": { "start": "node src/index.js" }
   ```

5. **Add a health-check route** (useful for testing later):

   ```js
   app.get("/health", (req, res) => res.json({ status: "ok" }));
   ```

---

## Step 3: Push the project to GitHub

```bash
git add .
git commit -m "Prepare for deployment"
git push origin main
```

Double-check on github.com that `.env` does **not** appear in the repository. If you ever pushed it by accident, change your Atlas database user's password right away, because deleting the file later does not remove it from Git history.

---

## Step 4: Create the Render web service and link GitHub

1. Sign up at render.com (signing in with GitHub is easiest).
2. In the dashboard, click **New +** then **Web Service**.
3. Authorize Render to access your GitHub repositories, either all of them or only selected ones, then pick your project repo.
4. Fill in the settings:

| Field             | Value                                                                     |
| ----------------- | ------------------------------------------------------------------------- |
| **Name**          | Short and simple, since it becomes your URL (`https://name.onrender.com`) |
| **Branch**        | `main`                                                                    |
| **Runtime**       | Node                                                                      |
| **Build Command** | `npm install`                                                             |
| **Start Command** | `npm start`                                                               |
| **Instance Type** | **Free** (no credit card is needed for the free instance type)            |

**Why these commands?** On every deploy, Render runs the build command, then an optional pre-deploy command, then the start command.

- `npm install` downloads your dependencies. Unlike Prisma, Mongoose needs no code-generation or migration step, so nothing else is required.
- `npm start` launches Express.

---

## Step 5: Set environment variables on Render (not in Git)

Before clicking **Create Web Service**, open **Advanced** (or later go to your service's **Environment** tab) and add:

| Key           | Value                                                                  |
| ------------- | ---------------------------------------------------------------------- |
| `MONGODB_URI` | The full Atlas connection string from Step 1                           |
| `JWT_SECRET`  | A long random string (generate a new one, do not reuse your local one) |
| `NODE_ENV`    | `production`                                                           |

- Do **not** add `PORT`. Render supplies it, and your code reads it from `process.env.PORT`.
- Copy values in from your local `.env`, but never commit that file.
- If your frontend is hosted elsewhere (for example Netlify), also add its URL to your CORS allowlist, ideally through an env variable such as `CLIENT_URL`.

Click **Create Web Service**. Render starts the first deployment right away.

---

## Step 6: Confirm automatic deployments

With Auto-Deploy enabled, every push to the watched branch triggers a new build and deploy. Check it under **Settings → Build & Deploy → Auto-Deploy** (it should be **On Commit**).

Test it: make a small change (for example, change the message in `/health`), then `git commit` and `git push`. In the Render dashboard, a new deploy should start by itself.

---

## Step 7: Test that the live URL can read and write to the database

1. **Watch the logs.** In Render, open the **Logs** tab and wait for a "Live" status. You should also see your "MongoDB connected" message. Errors during build or start show up here first.
2. **Check the server is up** (the first request after idle may take 30 to 60 seconds because free instances sleep):

   ```bash
   curl https://YOUR-APP.onrender.com/health
   ```

3. **Write** a document through your public API (replace the route and fields with your own):

   ```bash
   curl -X POST https://YOUR-APP.onrender.com/api/items \
     -H "Content-Type: application/json" \
     -d '{"name": "Deploy test"}'
   ```

4. **Read** it back:

   ```bash
   curl https://YOUR-APP.onrender.com/api/items
   ```

5. **Verify in the database itself.** In Atlas, open your cluster and click **Browse Collections**. Your new document should appear in the collection. If it is there, your deployed app really wrote to the remote database.

---

## Free-tier warning: idle clusters get paused

According to the Atlas documentation, a Free cluster with no activity for 30 days is paused automatically, and all connections are refused until you resume it. Atlas emails you seven days before pausing. If your API suddenly cannot connect after a long quiet period, open the Atlas dashboard and resume the cluster. Free clusters have no automatic backups, so export your data if it matters.

---

## Troubleshooting

| Symptom                                              | Likely cause and fix                                                                                |
| ---------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `MongooseServerSelectionError` or connection timeout | Atlas **Network Access** does not allow Render's IP. Add `0.0.0.0/0` (or check the entry is active) |
| `Authentication failed`                              | Wrong username or password, or special characters in the password that need URL-encoding            |
| Data saves to an unexpected database                 | The database name is missing after `.net/` in the URI                                               |
| "Application failed to respond"                      | The app is not listening on `process.env.PORT`                                                      |
| Build fails on `npm install`                         | Check that all dependencies are listed in `package.json`, not just installed locally                |
| Browser shows a CORS error                           | Add your frontend URL to the backend's allowed origins                                              |
| First request is very slow                           | Normal cold start on Render's free tier                                                             |
| Connection refused after a long break                | The free cluster was paused for inactivity. Resume it in Atlas                                      |
