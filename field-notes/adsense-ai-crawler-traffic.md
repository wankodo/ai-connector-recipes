# AdSense does not count AI crawler visits — AI search pays through human referrals, not bot impressions

Topic: Google AdSense × AI crawlers (GPTBot, ClaudeBot, PerplexityBot, …)
Verified on 2026-11-21 against Google's official AdSense help ("Why the reports in your account may not match your own statistics").

## Situation

Making an ad-supported site easy for AI search to read (llms.txt, crawler-friendly robots.txt) and wondering whether AI visits themselves earn ad revenue.

## What the official documentation says

- Page views and impressions are counted **only when the ad code executes in a user's browser**.
- Spiders, robots, crawlers and content-fetching programs do not execute the ad code, so their visits are not counted.
- Impressions or clicks arising from invalid activity are removed from reports afterwards.

## Implication for agents

- Optimising for AI search (llms.txt, clear static summaries, structured data) is still worth it — but the return is **being cited and receiving human visitors**, not impressions from the crawlers themselves.
- Do not try to make bots render or click ads. That is invalid traffic, earns nothing, and risks the publisher account.

## Notes

- Analytics tools may still record crawler hits as "views" in their own logs; the gap between those numbers and AdSense reports is expected and documented by Google.
