\# CS Market Data API



A production-grade market data system that aggregates live pricing from four independent CS2 skin marketplaces, processing over 3.65 million listings for automated arbitrage detection and trade-up discovery. The system bypasses enterprise bot protection, unifies disparate APIs, and serves real-time data through a local REST server.



\## Markets Integrated



\*\*Skinport\*\* protects their 3.65 million listings behind Cloudflare. I used SeleniumBase in Undetected Chrome mode to navigate their JavaScript challenges and browser fingerprinting, then captured the authenticated session cookies. Once obtained, these cookies transfer to curl\_cffi for fast API calls without the overhead of a browser.



\*\*CSFloat\*\* requires authentication and enforces a 200 request per hour rate limit. I extracted the JWT session token from an authenticated browser, then used curl\_cffi with Chrome 120 TLS fingerprint impersonation to bypass their JA3 fingerprint detection. I built a 12 hour caching layer that rotates through different sort orders and price ranges, reducing API usage by 96 percent. I also built a 700 entry name to ID lookup system that maps skin names to their internal identifiers.



\*\*Buff163\*\* is the largest CS skin market by volume. I reverse engineered their internal website API by replicating the exact browser headers including sec-ch-ua client hints, x-requested-with XMLHttpRequest markers, and CSRF tokens extracted from the authenticated session. This bypasses their paid developer API tiers and provides access to 34,000 items with full order book depth and individual listing float values with no monthly rate limits.



\*\*Steam\*\* provides public market data through their Community Market API, returning real time prices, 24 hour trading volume, and item search results.



\## The REST API



All market data flows into a local REST server running on port 8080 with over 30 endpoints. The bulk items endpoint fetches prices from all four markets for multiple skins in a single request. Trade up discovery endpoints find viable collections, return input items sorted by lowest float, and check what the output item sells for. The arbitrage endpoint scans the watchlist and ranks opportunities by profit percentage.



\## Tech Stack



Python 3.14, SeleniumBase, curl\_cffi, SQLite, REST API. Handles Cloudflare bypass, JWT session authentication, CSRF token extraction, browser header spoofing, and rate limit management.



\## Key Results



The system found a 421 percent arbitrage opportunity on AK-47 Redline Field Tested. API calls to CSFloat were reduced by 96 percent through caching. Four separate markets were unified under a single API. The entire system runs without any paid API subscriptions.



\## Screenshots



!\[API Endpoints](api\_endpoints.png)

!\[Buff163 Search](buff163\_search.png)

!\[Buff163 Floats](buff163\_floats.png)

!\[Arbitrage](arbitrage.png)

!\[Steam Search](steam\_search.png)

!\[Steam Price](steam\_price.png)

!\[Buff163 Browse](buff163\_browse.png)



\## Full Source Code



Available upon request for verified recruiters and hiring managers.



Contact: yehiabrkt46@gmail.com

