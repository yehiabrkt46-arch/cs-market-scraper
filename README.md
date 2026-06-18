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



All market data flows into a local REST server running on port 8080 with over 30 endpoints. The bulk items endpoint fetches prices from all four markets for multiple skins in a single request. Trade up discovery endpoints find viable collections, return input items sorted by lowest float, and check what the output item sells for. The arbitrage endpoint scans the entire watchlist and ranks opportunities by profit percentage. Float analysis tells you how rare a specific wear value is within its range. The CSFloat sweep endpoint uses the caching system to return hundreds of skins with real float and price data.



Everything is accessible through simple GET requests, making it easy for any trading bot or dashboard to consume.



\## Tech Stack



Python 3.14, SeleniumBase for undetected Chrome automation, curl\_cffi for TLS fingerprint impersonation, SQLite for local caching, and Python's built in HTTP server for the REST API. The system handles Cloudflare bypass, JWT session authentication, CSRF token extraction, browser header spoofing, and intelligent rate limit management.



\## Key Results



The system found a 421 percent arbitrage opportunity on AK-47 Redline Field Tested, priced at 8 dollars and 68 cents on CSFloat versus 42 dollars and 70 cents on Steam. API calls to CSFloat were reduced by 96 percent through caching. Four separate markets were unified under a single API where most competitors access only one or two. The entire system runs without any paid API subscriptions, all data extracted from internal website endpoints that standard HTTP libraries cannot reach.



\## Code Samples



\### Cloudflare Bypass and Session Transfer



The system launches an undetected browser to bypass Cloudflare challenges, captures authenticated session cookies, then transfers them to a TLS-impersonating HTTP client:

from seleniumbase import Driver

from curl\_cffi import requests

import time



driver = Driver(uc=True, headless=True)

driver.get("https://skinport.com/market")

time.sleep(8)



cookies = {}

for c in driver.get\_cookies():

cookies\[c\["name"]] = c\["value"]



resp = requests.get(

"https://skinport.com/api/browse/730",

params={"sort": "price", "order": "asc", "limit": 5},

cookies=cookies,

impersonate="chrome110"

)



items = resp.json().get("items", \[])



text



\### Intelligent Caching Layer



A 12-hour SQLite cache with price range rotation reduces API usage by 96 percent:

if db.is\_range\_cached(min\_price, max\_price, cache\_minutes=720):

return get\_cached\_listings(min\_price, max\_price)

else:

items = csfloat.get\_listings(

min\_price=min\_price,

max\_price=max\_price,

sort="price",

order="asc"

)

cache\_listings(items)



text



\### Internal API Access



Reverse-engineers Buff163's internal website API by replicating exact browser headers and CSRF tokens:

from curl\_cffi import requests



headers = {

"accept": "application/json, text/javascript, \*/\*; q=0.01",

"referer": "https://buff.163.com/market/csgo",

"x-requested-with": "XMLHttpRequest",

"sec-ch-ua": ""Google Chrome";v="149", "Chromium";v="149"",

}



cookies = {

"session": "1-3HpJXIT5oWZhYOdOor\_jdR1iQ...",

"csrf\_token": "IjdlODVjOTBiNTQ3MjNlMW...",

}



resp = requests.get(

"https://buff.163.com/api/market/goods",

params={"game": "csgo", "search": "AK-47 Redline"},

headers=headers,

cookies=cookies,

impersonate="chrome120"

)



data = resp.json()

items = data\["data"]\["items"]



text



\### Cross-Market Arbitrage Engine



One API call compares prices across all four markets:

resp = requests.get("http://localhost:8080/bulk/items", params={

"skins": "AK-47 | Redline (Field-Tested),AWP | Asiimov (Field-Tested)",

"sources": "skinport,steam,csfloat,buff163"

})



data = resp.json()



for skin\_name, skin\_data in data\["results"].items():

arb = skin\_data.get("arbitrage", {})

if arb.get("profit\_percent", 0) > 50:

print(f"Buy on {arb\['buy\_from']} for 

a

r

b

\[

′

b

u

y

p

r

i

c

e

′

]

"

)

p

r

i

n

t

(

f

"

S

e

l

l

o

n

a

r

b

\[

′

s

e

l

l

t

o

′

]

f

o

r

arb\[ 

′

&#x20;buy 

p

​

&#x20;rice 

′

&#x20;]")print(f"Sellonarb\[ 

′

&#x20;sell 

t

​

&#x20;o 

′

&#x20;]for{arb\['sell\_price']}")

print(f"Profit: {arb\['profit\_percent']}%")



text



\## Screenshots



\### API Endpoints - 30+ routes across 4 markets

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



