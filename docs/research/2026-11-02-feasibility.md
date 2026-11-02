# Commute optimizer feasibility

Research date: 2026-11-02. Status: findings for discussion; provider and platform remain undecided.

## Finding

The proposed Google-based historical estimate store should not be implemented under the published standard terms without explicit permission. A free request allowance does not establish permission to retain estimates. TomTom is worth investigating next, but its historical-storage rights have not been verified. No provider has passed the brief's first-phase exit criterion, and no live traffic request has been tested.

This note follows the supplied product brief's next step: validate the provider and platform before building the application. The repository currently contains no application, dependencies, CI configuration, or active validation hooks.

## Google Routes

Google's general terms prohibit caching Maps Content except where expressly permitted and restrict creating content from it (§3.2.3). The Routes-specific exception (§19.3) permits caching latitude and longitude for 30 days. It does not grant the same exception for duration estimates. General Google ID caching is a separate exception. Our assessment is that neither the proposed raw duration history nor retaining only derived statistics is a verified permitted use. Obtain written clarification covering both before depending on Google. [General terms](https://cloud.google.com/maps-platform/terms), [service-specific terms](https://cloud.google.com/maps-platform/terms/maps-service-terms).

Billing must be enabled, and requests need an API key or OAuth token. Traffic-aware driving uses the Pro pricing tier; two-wheeler routing triggers Enterprise. A narrow response field mask does not make a traffic-aware request an Essentials request. Daily quota controls are available. [Usage and billing](https://developers.google.com/maps/documentation/routes/usage-and-billing), [SKU details](https://developers.google.com/maps/billing-and-pricing/sku-details).

Compute Routes supports traffic-aware estimates, requires a response field mask, and defaults departure time to the request time. Past driving departures are unsupported. Initially, `routes.duration,routes.distanceMeters` would be enough for the proposed duration analysis if storage permission is obtained. Future departures blend live and historical traffic according to how far ahead they are. This API does not supply an archive of actual journeys. [Compute Routes reference](https://developers.google.com/maps/documentation/routes/reference/rest/v2/TopLevel/computeRoutes), [traffic configuration](https://developers.google.com/maps/documentation/routes/config_trade_offs).

| Compute Routes Pro | Monthly free requests | First paid tier, USD per 1,000 requests |
| --- | ---: | ---: |
| Global pricing | 5,000 | $10 |
| India pricing, if eligible | 35,000 | $3 |

These are published pay-as-you-go allowances, checked on the research date. Usage is aggregated across projects on the billing account. India eligibility depends on billing and primary usage in India; a Bengaluru route alone does not establish it. Verify account eligibility and other usage before testing. [Global pricing](https://developers.google.com/maps/billing-and-pricing/pricing), [India pricing](https://developers.google.com/maps/billing-and-pricing/pricing-india), [India eligibility FAQ](https://developers.google.com/maps/billing-and-pricing/india).

## Request volume

The brief's estimate of roughly 90 requests per day needs correction. Four requests per hour across 24 hours is 96. Sampling every 15 minutes over the half-open interval 07:00 to 21:00 is 56 requests per day, or 1,736 in a 31-day month, for one route direction. Two directions double that to 3,472. These calculations assume one departure time per request and exclude retries, manual tests, and other API consumers.

If a permitted provider is selected, count every attempt against a local request budget, reserve capacity for retries, and apply provider-side limits where available. Local counters alone cannot bound use by other clients sharing a key. Keep paid use disabled where the account supports that choice; confirm the provider's exhaustion behavior before claiming a zero-cost configuration.

## Other data sources

TomTom currently advertises 20,000 free Routing API requests per month without an upfront credit card. Its routing coverage table lists India with real-time traffic support. That makes it a plausible technical candidate, not a confirmed replacement: this research did not establish permission to retain duration estimates or build historical statistics from them. Its legal page exposed only navigation through both the research interface and a browser visit. Confirm the applicable account agreement, retention rights, derived-analysis rights, attribution, and what happens at the free limit before an integration test. [Pricing](https://docs.tomtom.com/pricing), [market coverage](https://docs.tomtom.com/routing-api/documentation/tomtom-maps/v1/product-information/market-coverage), [legal page requiring further review](https://docs.tomtom.com/legal/terms-and-conditions).

The owner confirmed during this research that automatic traffic estimates are essential. Manually recorded journeys are not an acceptable MVP substitute. Provider permission and a bounded request test remain prerequisites; the project must not quietly switch to journey logging to avoid them.

## Platform tradeoffs

| Approach | What it supports | Collection limit |
| --- | --- | --- |
| Local-first web app/PWA | Browser-local storage and a dashboard accessible on multiple device types | Cannot promise unattended collection every 15 minutes |
| Native Android app | Local database and recurring background work | WorkManager has a 15-minute minimum repeat interval and execution depends on constraints and system optimizations |
| Native iOS app | Local persistence and system-managed background opportunities | An earliest start date does not guarantee execution at that time |

Chrome explicitly leaves periodic background sync timing to the browser and ties availability to installation, engagement, connectivity, and power. A PWA could collect while actively open. It is a poor basis for promising complete automatic coverage of commute windows. [Chrome background sync](https://developer.chrome.com/docs/capabilities/periodic-background-sync).

Native mobile also requires accepting missed or delayed samples. Platform selection should follow the target device and whether unattended collection is essential. [Android work requests](https://developer.android.com/develop/background-work/background-tasks/persistent/getting-started/define-work), [Apple earliest start date](https://developer.apple.com/documentation/backgroundtasks/bgtaskrequest/earliestbegindate).

SQLite in a browser is feasible through its WebAssembly implementation and origin-private filesystem (OPFS). Persistence requires a compatible browser and a worker; concurrency and header requirements depend on the selected storage implementation. Private browsing can restrict persistence. A browser choice therefore needs an explicit persistence check and an export/restore path. [SQLite persistence documentation](https://sqlite.org/wasm/doc/trunk/persistence.md).

## Decisions needed next

1. Investigate TomTom's applicable terms first, including the account-specific agreement if required. Automatic estimates are required; no provider has been approved.
2. Resolve provider permission for local raw history and derived analytics, then test one request with a user-provided key, confirmed quota limits, and the intended Bengaluru route. Record the result without retaining prohibited payloads.
3. Choose the target device and collection lifecycle. Finalize the schema and sampling semantics after those choices.

Before implementation, the design must distinguish observation time from requested departure time and actual journey departure; preserve route timezone and direction; define route identity, duplicate handling, failures, retention, and analysis boundaries. Estimate requests made for the same departure at different forecast horizons should not silently become independent evidence. Record the number of distinct days as well as observation count before setting an eligibility threshold. The score, minimum evidence, weekday grouping, and departure-window size remain proposals to resolve in the design.

Also decide whether a route means a pair of endpoints and routing options, or one fixed physical path. Provider-selected roads can change between requests, so endpoint-only sampling may mix path changes with traffic changes. The request contract permits intermediate waypoints, but a fixed-path guarantee has not been validated. [Compute Routes reference](https://developers.google.com/maps/documentation/routes/reference/rest/v2/TopLevel/computeRoutes).

## Work record

Owner: this Codex chat. Checkout: `/Users/gauravaditya/work/projects/sides/commute-optimizer`. Branch: `docs/commute-feasibility`, based on freshly fetched `origin/main` at `5b446fa74cf81f4e439dc8386200da65f87a3fef`. Intended delivery is one documentation commit and an open PR; merging needs separate authorization.

Validation is documentation-only: inspect the complete diff, check Markdown and local links, and run `git diff --check`. Runtime tests do not apply because no executable files change. No API key was used, no billing settings were changed, and no traffic request was made. Phase 1 remains incomplete pending provider rights and a successful bounded request.
