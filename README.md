\# CS Market Data API



A production-grade market data system that aggregates live pricing from four independent CS2 skin marketplaces, processing over 3.65 million listings for automated arbitrage detection and trade-up discovery. The system bypasses enterprise bot protection, unifies disparate APIs, and serves real-time data through a local REST server.



\## Markets Integrated



The system connects to four separate marketplaces, each requiring a different access strategy due to their unique security measures.



\*\*Skinport\*\* protects their 3.65 million listings behind Cloudflare. I used SeleniumBase in Undetected Chrome mode to navigate their JavaScript challenges and browser fingerprinting, then captured the authenticated session cookies. Once obtained, these cookies transfer to curl\_cffi for fast API calls without the overhead of a browser. This gives unlimited access to every listing with float values, rarity filters, and collection grouping.



\*\*CSFloat\*\* requires authentication and enforces a 200 request per hour rate limit. I extracted the JWT session token from an authenticated browser, then used curl\_cffi with Chrome 120 TLS fingerprint impersonation to bypass their JA3 fingerprint detection. To work within the rate limit, I built a 12 hour caching layer that rotates through different sort orders and price ranges, reducing API usage from 150 calls per cycle down to just 6 calls every 30 minutes, a 96 percent reduction. I also built a 700 entry name to ID lookup system that maps skin names to their internal def\_index and paint\_index values, enabling targeted queries that bypass the 40 skin limit on global searches.



\*\*Buff163\*\* is the largest CS skin market by volume but requires Chinese authentication. I reverse engineered their internal website API by replicating the exact browser headers including sec-ch-ua client hints, x-requested-with XMLHttpRequest markers, and CSRF tokens extracted from the authenticated session. This bypasses their paid developer API tiers and provides access to 34,000 items with full order book depth, supply and demand numbers, and individual listing float values with no monthly rate limits.



\*\*Steam\*\* provides public market data through their Community Market API. Using curl\_cffi browser impersonation, the system fetches real time prices, 24 hour trading volume, and item search results. The price overview endpoint returns the current lowest listing price, median price, and how many units sold in the past day.



\## How The Caching System Works



The caching layer is what makes the rate limited APIs sustainable. When the system fetches data from CSFloat, it stores every listing in a local SQLite database with a 12 hour expiry window. The cache rotates through six different price brackets and three sort orders across multiple pages, so each cycle discovers different skins instead of re-fetching the same ones. Once a price range is cached, subsequent requests return instantly from the database with zero API calls. The cache auto expires after 12 hours, at which point a fresh pull updates the data. This approach turned CSFloat from a bottleneck into a reliable data source while staying well within the 200 call per hour budget.



\## Float Verification System



One of the trickiest problems was ensuring honest float to price pairing. CSFloat listings include both auction items and buy now items. Auctions show artificially low prices because they are starting bids, not real sale prices. The system filters to only buy\_now listings, then pairs each float value directly with its actual price. This prevents phantom profit calculations where a rare low float skin gets incorrectly matched with an auction starting bid, which would show unrealistic arbitrage opportunities.



\## The REST API



All market data flows into a local REST server running on port 8080 with over 25 endpoints. The bulk items endpoint fetches prices from all four markets for multiple skins in a single request. Trade up discovery endpoints find viable collections, return input items sorted by lowest float, and check what the output item sells for. The arbitrage endpoint scans the entire watchlist and ranks opportunities by profit percentage. Float analysis tells you how rare a specific wear value is within its range. The CSFloat sweep endpoint uses the caching system to return hundreds of skins with real float and price data.



Everything is accessible through simple GET requests, making it easy for any trading bot or dashboard to consume.



\## Tech Stack



Python 3.14, SeleniumBase for undetected Chrome automation, curl\_cffi for TLS fingerprint impersonation, SQLite for local caching, and Python's built in HTTP server for the REST API. The system handles Cloudflare bypass, JWT session authentication, CSRF token extraction, browser header spoofing, and intelligent rate limit management.

S

\## Key Results



The system found a 421 percent arbitrage opportunity on AK-47 Redline Field Tested, priced at 8 dollars and 18 cents on CSFloat versus 42 dollars and 67 cents on Steam. API calls to CSFloat were reduced by 96 percent through caching. Four separate markets were unified under a single API where most competitors access only one or two. The entire system runs without any paid API subscriptions, all data extracted from internal website endpoints that standard HTTP libraries cannot reach.



\## Code Samples



\### Cloudflare Bypass \& Session Transfer (Skinport)



The system launches an undetected browser to bypass Cloudflare's JavaScript challenges, captures authenticated session cookies, then transfers them to a TLS-impersonating HTTP client:





```python

from seleniumbase import Driver

from curl\_cffi import requests

import time



driver = Driver(uc=True, headless=True)

driver.get("https://skinport.com/market")

time.sleep(8)



cookies = {}

for c in driver.get\_cookies():

&#x20;   cookies\[c\['name']] = c\['value']



resp = requests.get(

&#x20;   "https://skinport.com/api/browse/730",

&#x20;   params={"sort": "price", "order": "asc", "limit": 5},

&#x20;   cookies=cookies,

&#x20;   impersonate="chrome110"

)



items = resp.json().get("items", \[])

&#x09;

&#x09;





Caching Layer (CSFloat)

A 12-hour SQLite cache with price range and sort order rotation reduces API usage by 96 percent:



```python

if db.is\_range\_cached(min\_price, max\_price, cache\_minutes=720):

&#x20;   return get\_cached\_listings(min\_price, max\_price)

else:

&#x20;   items = csfloat.get\_listings(

&#x20;       min\_price=min\_price, 

&#x20;       max\_price=max\_price,

&#x20;       sort="price",

&#x20;       order="asc"

&#x20;   )

&#x20;   cache\_listings(items)





Internal API Access (Buff163)

Reverse-engineers Buff163's internal website API by replicating exact browser headers and CSRF tokens, bypassing their paid developer tiers:





```python

from curl\_cffi import requests



headers = {

&#x20;   'accept': 'application/json, text/javascript, \*/\*; q=0.01',

&#x20;   'referer': 'https://buff.163.com/market/csgo',

&#x20;   'x-requested-with': 'XMLHttpRequest',

&#x20;   'sec-ch-ua': '"Google Chrome";v="149", "Chromium";v="149"',

}



cookies = {

&#x20;   'session': '1-3HpJXIT5oWZhYOdOor\_jdR1iQ...',

&#x20;   'csrf\_token': 'IjdlODVjOTBiNTQ3MjNlMW...',

}



resp = requests.get(

&#x20;   'https://buff.163.com/api/market/goods',

&#x20;   params={'game': 'csgo', 'search': 'AK-47 Redline'},

&#x20;   headers=headers,

&#x20;   cookies=cookies,

&#x20;   impersonate='chrome120'

)



data = resp.json()

items = data\['data']\['items']





Cross-Market Arbitrage Engine

One API call compares prices across all four markets and returns ranked profit opportunities:





```python

resp = requests.get('http://localhost:8080/bulk/items', params={

&#x20;   'skins': 'AK-47 | Redline (Field-Tested),AWP | Asiimov (Field-Tested)',

&#x20;   'sources': 'skinport,steam,csfloat,buff163'

})



data = resp.json()



for skin\_name, skin\_data in data\['results'].items():

&#x20;   arb = skin\_data.get('arbitrage', {})

&#x20;   if arb.get('profit\_percent', 0) > 50:

&#x20;       print(f"Buy on {arb\['buy\_from']} for ${arb\['buy\_price']}")

&#x20;       print(f"Sell on {arb\['sell\_to']} for ${arb\['sell\_price']}")

&#x20;       print(f"Profit: {arb\['profit\_percent']}%")













\## Screenshots



\### API Endpoints (30+ routes across 4 markets)

!\[API Endpoints](api\_endpoints.png)



\### Buff163 Search - 10 AK-47 Redline variants with supply/demand

!\[Buff163 Search](buff163\_search.png)



\### Buff163 Listings - Individual items with float values

!\[Buff163 Floats](buff163\_floats.png)



\### Cross-Market Arbitrage - 391% profit detected

!\[Arbitrage](arbitrage.png)



\### Steam Search - Real-time market data

!\[Steam Search](steam\_search.png)



\### Steam Price Overview - Current price + 24h volume

!\[Steam Price](steam\_price.png)



\### Buff163 Browse - 34,000 items across 11,000 pages

!\[Buff163 Browse](buff163\_browse.png)



\## Full Source Code



Available upon request for verified recruiters and hiring managers.



Contact: yehiabrkt46@gmail.com

