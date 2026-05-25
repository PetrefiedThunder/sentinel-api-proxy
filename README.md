# sentinel-api-proxy

Reverse-proxy in front of the Sentinel API backend.

- `api.pauseapi.app/*` → `sentinel-api-production-9c76.up.railway.app/*`
- TLS handshake happens at Vercel's edge (anycast, close to user)
- Vercel handles cert issuance via Let's Encrypt — no Railway cert dance needed
- Rewrites are CDN-level routing, not function invocations, so long-poll (30s)
  works without function-timeout issues
