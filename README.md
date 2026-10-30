# Ascii Bloom

Flowers made of code, and shared gardens for people who are far apart.

## Files

```
index.html        the whole front end
api/garden.js     serverless API (Vercel + Neon Postgres)
package.json
```

## Deploy

1. Push this folder to GitHub.
2. In Vercel: **Add New → Project**, import the repo (no build settings needed).
3. In the project, open **Storage → Create → Neon (Postgres)** and connect it. Vercel adds `DATABASE_URL` automatically.
   (Or paste your own Neon connection string as an environment variable named `DATABASE_URL`.)
4. Redeploy. Tables are created automatically on the first request.

## How gardens work

- Creating a garden makes two random secrets: a **view id** and an **edit key**.
- **My Garden link** (`?view=…`) is read-only and shows a "Make your own garden" button.
- **Our Garden link** (`?join=…`) lets anyone holding it plant and remove flowers. Share it only with your person.
- There are no accounts, so anyone with the edit link has full access. The browser remembers it on each device.
- Data stored: garden name, anniversary date, and each flower (type, colour, position, note, a random per-device id, time). No emails or names.
- Limits: 80 flowers per garden, short notes, validated input.

## Local dev

```
npm i -g vercel
vercel env pull
vercel dev
```
