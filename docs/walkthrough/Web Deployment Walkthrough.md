# Web Deployment Walkthrough (VPS + PM2)

This guide walks through building the Notiguide admin web (`web/`) as a Next.js standalone bundle, shipping it to your own VPS, and (re)starting it under PM2 with zero downtime.

It assumes:

- You have shell access (SSH) to the VPS as a non-root user with `pm2` already installed (`npm i -g pm2`).
- Node.js **22 LTS** (or newer) is installed on the VPS — the same major as you build with locally.
- You build locally and ship the artifact. We do not run `yarn install` / `yarn build` on the VPS.

> **Why standalone?** With `output: "standalone"` (already set in `web/next.config.ts`), Next.js traces every file the server actually imports — including `node_modules` — into `.next/standalone/`. The VPS does not need `npm`/`yarn`, a lockfile, or a full `node_modules`. Just Node.js and the traced bundle.

---

## 1. Build the standalone bundle (local)

From `web/`:

```bash
yarn install --immutable   # only when deps changed
yarn build                 # next build
```

`yarn build` produces three things relevant to deployment:

| Path | What it is | Self-contained? |
|---|---|---|
| `web/.next/standalone/` | Minimal Node app: `server.js` + traced `node_modules/` + server-rendered code | No — missing `public/` and `.next/static/` |
| `web/.next/static/` | Hashed client-side JS/CSS chunks | Must be copied into the standalone tree |
| `web/public/` | Static assets (favicon, images, etc.) | Must be copied into the standalone tree |

Next.js intentionally leaves `public/` and `.next/static/` outside the standalone folder because they are commonly served by a CDN. Since we serve them from `server.js` directly, we copy them in:

```bash
# from web/
cp -r public        .next/standalone/
cp -r .next/static  .next/standalone/.next/
```

After this step, `.next/standalone/` is fully self-contained and runnable with `node server.js`.

> **Smoke-test locally** (optional):
>
> ```bash
> PORT=2312 node .next/standalone/server.js
> ```
>
> Then hit `http://localhost:2312`. If it serves the dashboard with styles and fonts, the bundle is good.

---

## 2. What to upload to the VPS

You only need to ship two things:

1. **The standalone bundle** — `web/.next/standalone/` (with `public/` and `.next/static/` already copied in).
2. **The PM2 ecosystem file** — `web/ecosystem.config.cjs`.

`ecosystem.config.cjs` resolves `cwd` from its own directory (`__dirname`), so as long as you place it one level above `.next/standalone/` it will keep working unchanged:

```js
// web/ecosystem.config.cjs (already in repo)
cwd: path.join(__dirname, ".next/standalone"),
script: "server.js",
instances: 2,
exec_mode: "cluster",
env: { NODE_ENV: "production", PORT: 2312 },
```

Everything else (`src/`, `node_modules/`, `yarn.lock`, `next.config.ts`, TypeScript configs, `.next/cache/`, `.next/server/`, `.next/types/`) is **not** needed at runtime.

### Recommended upload command

Pick a stable directory on the VPS, e.g. `/var/www/notiguide-admin/`. Then from your local `web/`:

```bash
# Sync the standalone tree (fast incremental upload, deletes stale files)
rsync -az --delete \
  .next/standalone/ \
  deploy@vps.example.com:/var/www/notiguide-admin/.next/standalone/

# Sync the ecosystem file
rsync -az \
  ecosystem.config.cjs \
  deploy@vps.example.com:/var/www/notiguide-admin/ecosystem.config.cjs
```

Note the trailing slash on `.next/standalone/` — it copies the *contents* into the target dir, not the folder itself.

---

## 3. Final directory layout on the VPS

After the first deploy, `/var/www/notiguide-admin/` looks like this (high-level only):

```
/var/www/notiguide-admin/
├── ecosystem.config.cjs          # PM2 config (resolves cwd from here)
└── .next/
    └── standalone/               # cwd for PM2; everything below is the app
        ├── server.js             # entry point, reads PORT env
        ├── package.json          # minimal, traced
        ├── node_modules/         # traced production deps only
        ├── public/               # copied from web/public
        └── .next/
            ├── static/           # copied from web/.next/static
            ├── server/           # SSR bundles + RSC payloads
            └── BUILD_ID, *.json  # build manifests
```

The only two paths that must exist for the server to render correctly are:

- `/var/www/notiguide-admin/.next/standalone/public/`
- `/var/www/notiguide-admin/.next/standalone/.next/static/`

If either is missing, you'll see broken images and unstyled pages — that's the symptom of forgetting the `cp -r` step in section 1.

> **Runtime env**: anything the server reads at runtime (e.g. backend base URL, FCM server keys) should be set on the PM2 process — either in `ecosystem.config.cjs` under `env`, or via a `.env` file next to `ecosystem.config.cjs` and loaded with `--env`. Build-time `NEXT_PUBLIC_*` vars are already baked into the bundle and cannot be changed without rebuilding.

---

## 4. (Re)start under PM2

### First-time start

On the VPS:

```bash
cd /var/www/notiguide-admin
pm2 start ecosystem.config.cjs
pm2 save                        # persist process list across reboots
pm2 startup                     # one-time: print a sudo command to enable boot autostart
```

You should see the app listed as `notiguide-admin` running 2 cluster workers on port 2312.

### Subsequent deploys (zero downtime)

After uploading a new build (section 2), reload — don't restart:

```bash
pm2 reload ecosystem.config.cjs
```

`pm2 reload` does a rolling restart of the cluster workers: it spins up a new worker, waits until it's listening, then kills the old one, repeating per instance. With `instances: 2` you keep serving traffic throughout. `pm2 restart` would kill all workers at once and cause a brief 502 window — prefer `reload`.

### Useful follow-ups

```bash
pm2 status                      # is it up? how many restarts?
pm2 logs notiguide-admin        # tail stdout/stderr
pm2 logs notiguide-admin --err  # only errors
pm2 describe notiguide-admin    # full config + env actually loaded
pm2 monit                       # live CPU/mem dashboard
```

---

## 5. End-to-end deploy script (optional)

Once the layout is set up, a full deploy reduces to four commands. You can wrap them in a `web/scripts/deploy.sh` if you want:

```bash
# local
yarn build
cp -r public .next/standalone/ && cp -r .next/static .next/standalone/.next/
rsync -az --delete .next/standalone/ deploy@vps:/var/www/notiguide-admin/.next/standalone/

# remote
ssh deploy@vps 'cd /var/www/notiguide-admin && pm2 reload ecosystem.config.cjs'
```

That's the entire deploy loop: build → copy assets → rsync → `pm2 reload`. No `npm install` on the server, no downtime.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Page renders but no CSS/JS, 404s on `/_next/static/...` | Forgot `cp -r .next/static .next/standalone/.next/` | Re-run the cp step, redeploy |
| Favicon and images 404 | Forgot `cp -r public .next/standalone/` | Re-run the cp step, redeploy |
| `Error: Cannot find module 'next'` | Uploaded `.next/` but not `.next/standalone/` (which has its own traced `node_modules`) | Make sure rsync source is `.next/standalone/`, not `.next/` |
| `EADDRINUSE :::2312` after `pm2 reload` | Stale process from a manually-started `node server.js` | `pm2 delete notiguide-admin && pm2 start ecosystem.config.cjs` |
| Locale strings missing (English only / Vietnamese only) | `next-intl` messages didn't get traced | Verify `src/messages/*.json` are statically imported from `src/i18n/request.ts`. Dynamic `import(\`./messages/${locale}.json\`)` traces correctly; `fs.readFile` does not |
| Server starts but binds to `:3000` instead of `:2312` | `PORT` env not picked up | Check `ecosystem.config.cjs` is the one PM2 loaded: `pm2 describe notiguide-admin` |

---

## References

- Next.js — [Output File Tracing & `output: "standalone"`](https://nextjs.org/docs/app/api-reference/config/next-config-js/output)
- PM2 — [Cluster mode & zero-downtime reload](https://pm2.keymetrics.io/docs/usage/cluster-mode/)
- PM2 — [Ecosystem file reference](https://pm2.keymetrics.io/docs/usage/application-declaration/)
