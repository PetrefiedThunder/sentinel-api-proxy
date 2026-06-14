# sentinel-api-proxy

Reverse-proxy in front of the Sentinel API backend.

- `api.pauseapi.app/*` → `origin.pauseapi.app/*` (the Railway origin)
- TLS handshake happens at Vercel's edge (anycast, close to user)
- Vercel handles cert issuance via Let's Encrypt — no Railway cert dance needed
- Rewrites are CDN-level routing, not function invocations, so long-poll (30s)
  works without function-timeout issues

## Origin configuration (single source of truth)

The rewrite destination in [`vercel.json`](./vercel.json) points at a **stable
custom domain**, `origin.pauseapi.app`, **not** the auto-generated Railway
subdomain (`*.up.railway.app`). This is deliberate:

- Railway can rotate/regenerate the `*.up.railway.app` subdomain. If the proxy
  hardcoded it, that change would silently break **all** of `api.pauseapi.app` —
  a single point of failure.
- `origin.pauseapi.app` is a CNAME you control, pointed at the Railway service.
  Swapping the underlying Railway host (or moving to a load balancer) is a DNS
  change with **zero** changes to this repo.

**To change the origin, update the `origin.pauseapi.app` DNS record — do not
edit the Railway host in `vercel.json`.** `vercel.json` is the only place the
destination appears; nothing else in this repo hardcodes the origin URL.

> Why not a Vercel env var in the rewrite? Static `vercel.json` rewrites do
> **not** support environment-variable interpolation in `destination`; the field
> accepts only absolute paths, external URLs, and `source` path parameters
> (`:slug`, `:match*`). Env-var-driven config requires `vercel.ts` + a build
> step (`@vercel/config`), which this zero-dependency static repo intentionally
> avoids. A stable custom domain achieves the same de-SPOF goal with no build.
> See https://vercel.com/docs/project-configuration/vercel-json#rewrites and
> https://vercel.com/docs/project-configuration/vercel-ts
