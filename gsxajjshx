/**
 * Ascii Bloom: garden API (Vercel serverless function + Neon Postgres)
 *
 * GET  /api/garden?id=<viewId>    -> read-only garden
 * GET  /api/garden?key=<editKey>  -> garden + plants (caller may edit)
 * POST /api/garden {action: create | plant | remove | date, ...}
 *
 * Access model: no accounts. Every garden has two random secrets.
 *   id  = public view link  (read only)
 *   key = shared edit link  (anyone holding it can plant and remove)
 */
import { neon } from '@neondatabase/serverless';
import { randomBytes } from 'node:crypto';

const TYPES = ['lily', 'rose', 'tulip', 'daisy', 'cherry', 'sunflower'];
const PALS = ['blush', 'peach', 'lilac', 'snow'];
const MAX_PLANTS = 80;
const MAX_GARDENS_PER_HOUR = 30; // crude global brake against spam

let sql;
let ready;

function db() {
  const url = process.env.DATABASE_URL || process.env.POSTGRES_URL;
  if (!url) throw new HttpError(503, 'The garden database is not connected yet. Add DATABASE_URL in Vercel and redeploy.');
  return (sql ??= neon(url));
}

function ensureSchema() {
  return (ready ??= (async () => {
    const q = db();
    await q`create table if not exists gardens (
      id text primary key,
      edit_key text unique not null,
      name text not null default '',
      start_date date,
      created_at timestamptz not null default now()
    )`;
    await q`create table if not exists plants (
      id bigserial primary key,
      garden_id text not null references gardens(id) on delete cascade,
      type text not null,
      pal text not null,
      x real not null,
      seed integer not null,
      msg text not null default '',
      author text not null default '',
      created_at timestamptz not null default now()
    )`;
    await q`create index if not exists plants_garden_idx on plants (garden_id, created_at)`;
  })().catch((err) => {
    ready = undefined; // allow a retry on the next request
    throw err;
  }));
}

class HttpError extends Error {
  constructor(status, message) {
    super(message);
    this.status = status;
  }
}

const token = (bytes) => randomBytes(bytes).toString('base64url');
const text = (v, max) => String(v ?? '').replace(/[\u0000-\u001f<>]/g, '').trim().slice(0, max);

async function gardenBy({ id, key }) {
  const q = db();
  const rows = key
    ? await q`select id, name, start_date from gardens where edit_key = ${String(key).slice(0, 64)}`
    : await q`select id, name, start_date from gardens where id = ${String(id ?? '').slice(0, 32)}`;
  if (!rows.length) throw new HttpError(404, 'Garden not found');
  return rows[0];
}

async function snapshot(row) {
  const plants = await db()`
    select id::text as id, type, pal, x, seed, msg, author, extract(epoch from created_at)::bigint as ts
    from plants where garden_id = ${row.id} order by created_at, id`;
  return {
    garden: { id: row.id, name: row.name, startDate: row.start_date ? new Date(row.start_date).toISOString().slice(0, 10) : '' },
    plants,
  };
}

const actions = {
  async create({ name }) {
    const q = db();
    const [{ n }] = await q`select count(*)::int as n from gardens where created_at > now() - interval '1 hour'`;
    if (n >= MAX_GARDENS_PER_HOUR) throw new HttpError(429, 'Lots of gardens are being planted right now. Try again soon.');
    const id = token(9);
    const key = token(18);
    await q`insert into gardens (id, edit_key, name) values (${id}, ${key}, ${text(name, 40)})`;
    return { id, key };
  },

  async plant({ key, plant = {} }) {
    const row = await gardenBy({ key });
    const q = db();
    const [{ n }] = await q`select count(*)::int as n from plants where garden_id = ${row.id}`;
    if (n >= MAX_PLANTS) throw new HttpError(409, 'The garden is full. Remove a flower first.');
    if (!TYPES.includes(plant.type) || !PALS.includes(plant.pal)) throw new HttpError(400, 'Unknown flower');
    const x = Math.min(0.96, Math.max(0.04, Number(plant.x) || 0.5));
    const seed = Math.abs(parseInt(plant.seed, 10) || 1) % 2147483647;
    await q`insert into plants (garden_id, type, pal, x, seed, msg, author)
            values (${row.id}, ${plant.type}, ${plant.pal}, ${x}, ${seed}, ${text(plant.msg, 80)}, ${text(plant.author, 24)})`;
    return { ok: true };
  },

  async remove({ key, plantId }) {
    const row = await gardenBy({ key });
    await db()`delete from plants where id = ${String(plantId).replace(/\D/g, '').slice(0, 18) || '0'} and garden_id = ${row.id}`;
    return { ok: true };
  },

  async date({ key, date }) {
    const row = await gardenBy({ key });
    if (!/^\d{4}-\d{2}-\d{2}$/.test(date)) throw new HttpError(400, 'Invalid date');
    await db()`update gardens set start_date = ${date} where id = ${row.id}`;
    return { ok: true };
  },
};

export default async function handler(req, res) {
  res.setHeader('Cache-Control', 'no-store');
  try {
    await ensureSchema();
    if (req.method === 'GET') {
      const { id, key } = req.query;
      if (!id && !key) throw new HttpError(400, 'Missing id or key');
      return res.status(200).json(await snapshot(await gardenBy({ id, key })));
    }
    if (req.method === 'POST') {
      const body = typeof req.body === 'string' ? JSON.parse(req.body || '{}') : req.body || {};
      const run = actions[body.action];
      if (!run) throw new HttpError(400, 'Unknown action');
      return res.status(200).json(await run(body));
    }
    res.setHeader('Allow', 'GET, POST');
    throw new HttpError(405, 'Method not allowed');
  } catch (err) {
    const status = err.status || 500;
    if (status === 500) console.error(err);
    res.status(status).json({ error: status === 500 ? 'The garden database could not be reached. Check the Neon connection in Vercel.' : err.message });
  }
}
