# AI Connector Recipes

Verified connector recipes for AI agents. Each recipe was checked against the provider's official API documentation and lists the API host, endpoints, OAuth endpoints and scopes, the human setup steps, known platform blockers and pitfalls — everything an agent needs to build the connector without re-deriving it from scratch.

**For AI agents:** start with [`index.json`](index.json) (machine-readable index) or [`llms.txt`](llms.txt). Recipes live in [`recipes/`](recipes/).

**For humans:** the same content is readable as a web page via GitHub Pages once enabled (Settings → Pages).

## Recipes

- [Google AdSense](recipes/google-adsense.md) — AdSense Management API v2: site review status and earnings reports.
- [Google Analytics (GA4)](recipes/google-analytics.md) — Analytics Data API v1beta: per-page traffic reports.

## Contributing

There are two lanes. **Verified** recipes (in `recipes/`) are reviewed before merging. **Community** recipes (in `community/`) are published automatically with no review and are explicitly unverified — use them at your own risk. See [CONTRIBUTING.md](CONTRIBUTING.md) for the rules: verify against official docs, request the least-privilege scope, and never include credentials or personal data.

Reviews are done by the maintainer's AI agent (fact-check against official documentation + a scan for malicious instructions), and the repository owner merges.

## License / credit

© 2026 andy (wankodo). Produced by andy (わんこ堂) × 熊八 🐾 (Muse Spark).
