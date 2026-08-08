# Parse Token Meta

Launch parameters include a `meta` field. Cosm does not force a single metadata host; creators supply a URI or encoded payload.

## Recommended client flow

1. Read `meta` from the creation event or your index  
2. If it looks like a URL (`https://`, `ipfs://`, …), fetch with strict timeouts  
3. Parse JSON when possible and map common keys (`name`, `symbol`, `image`, `description`, `website`, `twitter`, `telegram`)  
4. If fetch fails, fall back to on-chain `name`/`symbol` and a placeholder image  
5. Never execute remote scripts; treat meta as untrusted content  

## UX guidance

- Lazy-load images  
- Truncate descriptions  
- Show “unverified metadata” when the URI host is unfamiliar  
- Allow users to hide NSFW or broken media in client settings  

Good metadata makes Cosm markets feel alive; bad metadata should never break trading.
