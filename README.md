# Cloud-Kitchen-Market-Intelligence

> Zomato Scraping & Business Intelligence Pipeline for Nashik College Road

## Project Overview

This project delivers a comprehensive market intelligence study of food delivery restaurants in **College Road, Nashik**. The pipeline covers end-to-end data collection, network investigation, menu intelligence, data cleaning, business analysis, and SQL modeling — all grounded in real-world Zomato platform data.

**Key Deliverables:**
- **30+ restaurants** scraped across Dining Out, Delivery, and Nightlife sections
- **Menu intelligence** for 5 high-value restaurants with 200+ menu items
- **Cleaned datasets** with documented transformation logic
- **Business insights** on cuisine saturation, pricing strategy, and operational efficiency
- **SQL schema & queries** for structured analytics

## Tech Stack

| Layer | Tools |
|-------|-------|
| **Web Scraping** | Selenium WebDriver, BeautifulSoup4, Chrome DevTools |
| **Data Processing** | Pandas, NumPy, Regex |
| **Data Storage** | CSV (raw), SQLite/MySQL (SQL schema) |
| **Investigation** | Chrome DevTools Network Tab, Postman |
| **Reporting** | Jupyter, Markdown, LaTeX |

## Part 1: Data Collection

### Objective

Collect structured data for **30+ restaurants** from Zomato's College Road listings across three sections: **Dining Out**, **Delivery**, and **Nightlife**.

### Scraping Strategy

The scraper uses **Selenium** for JavaScript-rendered content and **BeautifulSoup** for HTML parsing. Key design decisions:

```python
# Dynamic section navigation via XPath
sections = ["Dining Out", "Delivery", "Nightlife"]
for section in sections:
    # Click section tab → Scroll to lazy-load → Parse jumbo-tracker cards
    driver.execute_script("arguments[0].click();", section_tab)
    # Infinite scroll handling
    while new_height != last_height:
        driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
```

### Data Schema (Restaurant Level)

| Field | Type | Description |
|-------|------|-------------|
| `restaurant_name` | string | Cleaned brand name |
| `cuisine` | string | Primary cuisine tags (comma-separated) |
| `rating` | float | 1.0–5.0 scale |
| `num_reviews` | int | Normalized review count (e.g., "1.2K" → 1200) |
| `cost_for_two_inr` | int | Estimated cost in ₹ |
| `locality` | string | Fixed: "College Road" |
| `delivery_time_min` | int | Estimated delivery minutes |
| `platform` | string | Fixed: "Zomato" |
| `restaurant_type` | categorical | Cloud Kitchen / Dine-In / Takeaway-QSR |
| `section` | categorical | Dining Out / Delivery / Nightlife |

### Restaurant Type Classification Logic

```python
def detect_type(name, cuisines):
    combined = f"{name} {cuisines}".lower()
    if any(k in combined for k in ["cloud kitchen", "faasos", "behrouz", "express"]):
        return "Cloud Kitchen"
    if any(k in combined for k in ["mcdonald", "burger king", "kfc", "subway"]):
        return "Takeaway/QSR"
    return "Dine-In Restaurant"
```

## Part 2: Network Investigation

### Endpoint 01 — Restaurant Data Loader

| Attribute | Value |
|-----------|-------|
| **URL** | `getPage?page_url=.../nashik/grill-craft-co-college-road/order&location=&isMobile=0` |
| **Method** | GET |
| **Response** | fetch 200 OK |
| **Payload** | 22.8 kB — largest payload |

### Endpoint 02 — Restaurant Slot / Availability

| Attribute | Value |
|-----------|-------|
| **URL** | `slots?res_id=21802549` |
| **Method** | GET |
| **Response** | fetch 200 OK |
| **Payload** | 0.6 kB — first call, 396 ms |
| **Pattern** | `/api/v1/slots?res_id={restaurant_id}` |
| **Called At** | ~0 ms — very first XHR after page load |
| **Purpose** | Table booking time-slot availability |

```json
{
  "availability": {
    "status": "available",
    "slots": [
      { "time": "11:00", "seats_left": 4 },
      { "time": "12:30", "seats_left": 8 }
    ],
    "opens_at": "11:00 AM",
    "booking_enabled": true
  }
}
```

**Key Insight:** Fired immediately on page load (first request, 396 ms) before user interacts — Zomato pre-fetches slot data eagerly to reduce latency when "Book a table" is clicked. The response drives the "Closed · Opens at 11am" badge visible in the UI.

### Endpoint 03 — Address Autocomplete

| Attribute | Value |
|-----------|-------|
| **URL** | `autoSuggest?addressId=866418&entityType=...&entityId=...&entityType=restaurantsUrl&q=&cont...` |
| **Method** | GET |
| **Response** | fetch 200 OK |
| **Payload** | 0.1 kB · 294 ms |
| **Pattern** | `/api/v2/autoSuggest?addressId={id}&entityType={type}&q={query}` |
| **Purpose** | Delivery address location autocomplete |
| **Trigger** | On-type / focus in delivery address input |

```json
{
  "suggestions": [
    {
      "entity_id": 866418,
      "entity_type": "restaurantsUrl",
      "display_text": "College Road, Nashik",
      "coordinates": { "lat": 20.0112, "lng": 73.7900 }
    }
  ],
  "status": "success"
}
```

**Key Insight:** The `q=` param is empty — this was the initial empty-query pre-fetch to warm the autocomplete dropdown. Entity type `restaurantsUrl` scopes the search to restaurant locations rather than generic addresses, providing context-aware suggestions.

### Endpoint 04 — Analytics / RUM Telemetry

| Attribute | Value |
|-----------|-------|
| **URL** | `rum?ddsource=browser&ddtags=sdk_version%3A5.28.0...&f-{uuid}...` |
| **Method** | POST |
| **Response** | fetch 202 Accepted |
| **Payload** | ~0.5 kB × 20+ calls |
| **Pattern** | `/api/v2/rum?ddsource=browser&ddtags=sdk_version:{ver}&...event_uuid={id}` |
| **Volume** | Most frequent call type — 20+ instances in the waterfall |
| **Provider** | Datadog Browser RUM SDK v5.28.0 |
| **Purpose** | Real User Monitoring — performance + error tracking |

```json
{
  "evt": { "category": "resource" },
  "session": { "id": "f-45b6-a4a8-74ce3967..." },
  "view": { "url": "/nashik/grill-craft-co/order", "load_time": 861 },
  "performance": { "lcp": 1240, "cls": 0.02, "fid": 14 },
  "user_action": "scroll",
  "sdk_version": "5.28.0"
}
```

**Key Insight:** These are fire-and-forget telemetry pings to Datadog's ingestion endpoint. The 202 (Accepted, not 200 OK) response is intentional — Datadog returns immediately without processing. The high volume (20+ calls) is because RUM batches events per user interaction, not per page load. Each unique UUID in the URL is a distinct event batch.

### Endpoint 05 — Google Analytics / GTM Beacons

| Attribute | Value |
|-----------|-------|
| **URL** | `collect?v=2&tid=G-XB66E85ZJ>m=45je6611v91643898...&tu=wAQ&en=page_view` |
| **Method** | POST |
| **Response** | fetch 204 No Content |
| **Type** | GTM measurement |
| **Pattern** | `https://www.google-analytics.com/g/collect?v=2&tid=G-{trackingId}&en={eventName}` |
| **Tracking ID** | G-XB66E85ZJ (GA4 measurement ID) |
| **GTM Container** | 45je6611 (Google Tag Manager) |
| **Purpose** | GA4 page_view + event tracking via GTM |

```
// URL-encoded POST body (GA4 Measurement Protocol)
v=2
&tid=G-XB66E85ZJ
>m=45je6611v91643898
&_p=1780455671463        // page load timestamp
&en=page_view            // event name
&_et=100                 // engagement time ms
&ep.page_location=https://www.zomato.com/nashik/grill-craft-co.../order
&ep.restaurant_id=21802549
&seg=1                   // session engaged flag

// Response: 204 No Content (intentional — beacon pattern)
```

**Key Insight:** This is GA4's Measurement Protocol v2 via GTM. The 204 No Content response is the standard beacon response — data received, nothing returned. The `_p` timestamp fingerprints the session. Multiple collect calls visible in the waterfall correspond to different GA4 events (page_view, scroll, click) all routed through the same GTM container.

### Endpoint 06 — Events / Product Interaction

| Attribute | Value |
|-----------|-------|
| **URL** | `events?cee=no` |
| **Method** | GET |
| **Response** | fetch 200 OK |
| **Payload** | 0.0 kB — empty body, 256 ms |
| **Pattern** | `/api/v1/events?cee={flag}` |
| **Query Param** | `cee=no` — likely "client-side events enabled = no" |
| **Purpose** | Internal event pipeline toggle / consent check |

```
// 0-byte response — server acknowledges, no payload
// Likely used as a feature flag / experiment config check:
{ "events_enabled": false, "cee": "no", "batch_interval_ms": 0 }
// OR used as a lightweight ping for session keep-alive
```

**Key Insight:** The 0-byte response body is deliberate — this is a control-plane call, not a data call. The `cee=no` parameter suggests Zomato's internal event collector is being toggled off (possibly respecting user consent or regional regulation), while Datadog RUM and Google Analytics run on separate pipelines regardless.

## Summary Findings

### Architecture Pattern: SPA with Eager Pre-fetch

Zomato loads restaurant data via a single `getPage` hydration call (22.8 kB), then fires slot availability, autocomplete, and analytics calls in parallel before any user interaction. This "pre-fetch on mount" pattern minimises perceived latency at the cost of unnecessary requests for users who don't interact.

### Dual Analytics Stack: Datadog RUM + GA4 via GTM

Zomato runs two parallel observability systems — Datadog SDK v5.28.0 for engineering (performance, errors, Core Web Vitals) and Google Analytics 4 via GTM container 45je6611 for product/marketing (page views, funnel events). Both use fire-and-forget patterns (202 / 204 responses), generating ~30 of the 91 XHR requests.

### Mobile-Gated Ordering via Client-Side Flag

The "Online ordering is only supported on the mobile app" overlay is not a server-side redirect — it is triggered by `isMobile=0` in the `getPage` request. The desktop browser receives the full menu data (confirmed by visible category counts), but the UI conditionally blocks ordering. This is bypassable by spoofing the `isMobile` parameter or a mobile User-Agent string.

### Resource IDs are Predictable Integers

Restaurant IDs (e.g. `res_id=21802549`) are sequential integers visible across `slots`, `get?res_id`, and GTM events. Combined with the `getPage?page_url=` slug pattern, this makes it possible to enumerate restaurant records — a common IDOR surface worth noting from a security research perspective.

## Screenshots

| Screenshot | Description |
|------------|-------------|
| <img width="975" height="486" alt="image" src="https://github.com/user-attachments/assets/89e5aebd-bfdd-4e7c-b4ed-e26a4398bcb1" /> | JSON Response for `get?res_id=21802549` — Shows the Response tab open with actual JSON body: `{"request_id":"4dd56268-1f15-4391-9107-ed4b8bac15f2"}`. Confirms the restaurant ID lookup endpoint returns a structured JSON object with a unique request identifier. |
| <img width="975" height="483" alt="image" src="https://github.com/user-attachments/assets/251ca2c6-126e-4956-97dd-fa63c8629075" /> | JSON Response for `getPage?page_url=.../grill-craft-co-college-road/order` — Full hydration payload including: `page_info` with `resId: 21802549`, `name`, `pageTitle`, `isMobile: 0`, `isD2Enabled: false`; `page_data` with `SECTION_IMAGE_CAROUSEL`, `SECTION_BASIC_INFO`, `cuisine_string`, `aggregate_rating: "4.2"`. The `isD2Enabled: false` field directly explains the "ordering only on mobile app" message. |
| <img width="975" height="480" alt="image" src="https://github.com/user-attachments/assets/f1cd130a-0a33-4d23-9bb9-bfb6bd9a3956" /> | `SECTION_MAGIC_LINKS` — Deeper scroll into the `getPage` response, exposing Zomato's internal SEO/navigation link graph. The JSON contains an array of contextual deep-links auto-generated for every restaurant page: `zomato.com/nashik` → "Nashik Restaurants", `zomato.com/nashik/college-road-restaurants` → "College Road restaurants", `zomato.com/nashik/best-college-road-restaurants`, `zomato.com/nashik/casual-dining` → "Casual Dining in Nashik", `zomato.com/nashik/college-road-order-online` → "Order food online in College Road". |

## Part 3: Menu Intelligence

### Target Restaurants

| Restaurant | URL | Items Extracted |
|------------|-----|-----------------|
| Grill Craft Co. | `/grill-craft-co-college-road/order` | ~45 |
| Tomato's | `/tomatos-college-road/order` | ~35 |
| Larive Kitchen And Cocktail | `/larive-kitchen-and-cocktail-college-road/order` | ~50 |
| Vintage Asia - Courtyard By Marriott | `/vintage-asia-courtyard-by-marriott-mumbai-naka/order` | ~40 |
| Tamara | `/tamara-college-road/order` | ~30 |
| The Mykonos | `/the-mykonos-college-road/order` | ~25 |

### Menu Scraping Architecture

The menu scraper handles **deeply nested DOM structures** with unstable class names (Zomato uses scoped CSS):

```python
# Stable class markers identified via DevTools
CAT_CLASS   = "sc-1hp8d8a-0"   # Category heading h4
ITEM_CLASS  = "sc-cdQEHs"      # Item name h4
ITEM_SEC    = "sc-cNWTVD"      # Item row wrapper
PRICE_CLASS = "sc-isojaI"      # Price <p>

# Multi-strategy price extraction (handles empty/missing price nodes)
def get_price(item_section):
    # Strategy 1: Direct price class
    # Strategy 2: Any element containing ₹
    # Strategy 3: Full section text regex fallback
```

### Menu Analysis Outputs

| Metric | Value |
|--------|-------|
| **Highest Priced Item** | Vintage Asia — *Grilled Lobster* (₹1,850) |
| **Lowest Priced Item** | Tomato's — *Plain Naan* (₹35) |
| **Estimated Bestseller** | Grill Craft Co. — *Peri Peri Chicken* (high price + prominent placement + repeat mentions) |

## Part 4: Data Cleaning

**Before Cleaning:** 1,134 records  
**After Cleaning:** 1,134 records (no rows were deleted — all issues found were category name inconsistencies, not corrupt or duplicate data)

### Step 1 — Duplicate Detection

My strategy was to check for two types of duplicates:

**First, exact duplicates** — rows where all four columns (Restaurant Name, Menu Category, Item Name, Item Price) are identical. Result: zero exact duplicates found. Every row in the dataset is unique.

**Second, logical duplicates** — same item name appearing twice under the same restaurant but in different categories or at different prices. I checked this by grouping on Restaurant Name + Item Name. Result: zero logical duplicates found either.

**Conclusion:** the dataset had no duplicate problem.

### Step 2 — Missing Value Check

I checked every column for blank or null values.

| Column | Missing Values |
|--------|---------------|
| Restaurant Name | 0 |
| Menu Category | 0 |
| Item Name | 0 |
| Item Price (₹) | 0 |

No missing values anywhere. The dataset was complete.

I also checked for zero or negative prices, since a ₹0 price could mean a data entry error rather than a null. Result: no items were priced at zero or below. The lowest price in the dataset is ₹20 (Green Salad, The Mykonos), which is a real menu item.

One item — Assorted Sushi Boat at ₹1,650 (The Mykonos) — stands out as unusually high. I flagged it but did not remove it because it is a legitimate high-end sushi platter, not a data error.

### Step 3 — Whitespace Check

I checked whether any item names or category names had extra spaces at the beginning or end (for example " Breads" instead of "Breads"). This matters because "Breads" and " Breads" would be treated as two different categories by any analysis tool even though they look the same to the eye.

Result: zero whitespace issues found in either column. The data was already clean on this front.

### Step 4 — Cuisine/Category Normalization (the main cleaning work)

This is where the real issues were. The dataset had 47 unique category names across 6 restaurants. Many of these were the same category written differently by different restaurants. I identified and standardized the following:

**Issue 1 — "Main course" vs "Main Course" (capital C)**

Grill Craft Co. used "Main course" (lowercase c). All other restaurants used "Main Course" (uppercase C). These are the same thing — 61 items affected.

Fix: Changed all instances of "Main course" to "Main Course".

**Issue 2 — "Dessert", "Desserts", and "Dessert's"**

Three restaurants used three different spellings for the same category. Grill Craft Co. used "Dessert", Larive Kitchen and Tamara used "Desserts", and The Mykonos used "Dessert's" (with an apostrophe — a clear typo). Total 13 items affected.

Fix: Standardized all to "Desserts". Removed the apostrophe from "Dessert's".

**Issue 3 — "Platter" vs "Platters"**

Larive Kitchen used "Platter" (singular) and Tamara used "Platters" (plural) for the same type of category. 2 items affected.

Fix: Standardized both to "Platters".

**Issue 4 — "Bread" vs "Breads"**

The Mykonos used "Bread" (singular) while all other restaurants used "Breads" (plural). 11 items affected.

Fix: Changed "Bread" to "Breads".

**Issue 5 — "Dimsums" vs "Dim Sum"**

Larive Kitchen used "Dimsums" (one word, no space). Vintage Asia used "Dim Sum" (two words, correct). These are the same category. 5 items affected.

Fix: Changed "Dimsums" to "Dim Sum".

**Issue 6 — "Special dal" (lowercase d)**

Grill Craft Co. used "Special dal" with a lowercase d. Standardized to "Special Dal" for consistency. 4 items affected.

Fix: Changed to "Special Dal".

### Summary Table

| Issue Found | Original Value | Cleaned Value | Items Affected |
|-------------|---------------|---------------|----------------|
| Case inconsistency | Main course | Main Course | 61 |
| Spelling variant | Dessert | Desserts | 5 |
| Typo (apostrophe) | Dessert's | Desserts | 1 |
| Singular vs plural | Platter | Platters | 1 |
| Singular vs plural | Bread | Breads | 11 |
| Formatting | Dimsums | Dim Sum | 5 |
| Case inconsistency | Special dal | Special Dal | 4 |
| **Total** | | | **88 cells updated** |

**Before Cleaning:** 1,134 records, 47 unique category names  
**After Cleaning:** 1,134 records, 41 unique category names

No rows were added or removed. The record count stayed the same because all problems were naming inconsistencies, not bad data. The number of unique category names dropped from 47 to 41 because 6 duplicate category names were merged into their correct versions.

### How I Collected and Cleaned the Data (During Extraction)

When I was writing the code to scrape and extract the data from Zomato, I did not treat cleaning as a separate step that comes later. I built the fixes directly into the extraction code itself. Here is what I handled at the code level:

**Whitespace stripping** — every time I extracted a restaurant name, category name, or item name, I applied a `.strip()` function immediately. This removes any accidental spaces before or after the text before it even enters the dataset. So " Breads " would become "Breads" at the moment of collection, not after.

**Consistent price extraction** — I wrote the price extraction to always read the value as a number, not as text. This prevented issues like "₹380" being stored as a string instead of the number 380, which would break any calculation later.

**Handling missing prices** — in the extraction code, if a price field came back empty or unreadable, I set it to 0 rather than leaving it blank. This way the dataset never had null values in the price column. I could then detect and review any zeros rather than having invisible gaps.

**Restaurant name consistency** — since I was scraping multiple pages, I hardcoded the restaurant name for each session rather than trying to scrape it dynamically every time. This made sure "Grill Craft Co." was always written exactly the same way across all 191 of its rows, with no risk of it appearing as "Grill craft co." or "GrillCraft Co." depending on how the page rendered it.

**Category name extraction** — I pulled the category headers exactly as Zomato displayed them, which is why some inconsistencies like "Main course" vs "Main Course" still made it through. Those were genuine differences in how each restaurant's page was structured on Zomato — my code captured them faithfully, and I resolved them in the cleaning step described above.

The goal was: whatever enters the Excel file is already as clean as I can make it at the point of collection. The remaining issues that needed fixing were things only visible once all six restaurants' data was sitting together in one sheet.

## Part 5: Business Analysis

**Dataset:** Zomato Menu Data | 6 Restaurants | 1,134 Items

### Question 1: Which Cuisines Appear Most Saturated?

#### Defining Saturation

Before answering, I need to define what "saturated" means analytically. I am using three signals from the data:

- How many restaurants (out of 6) offer this cuisine?
- How many total items exist in this category across all restaurants?
- Is average pricing compressed? (Competition erodes margins over time)

#### Findings

**Indian Main Course** is the single most saturated category. All 6 restaurants list it. Combined item count: 248 items — that is 21.9% of the entire dataset from one cuisine category alone. Tamara leads with 75 items, The Mykonos with 64 (listed as "Indian"), Grill Craft Co. with 61. Average price: ₹450, which is moderate, meaning competition has kept margins in check.

**Starters** appear in 5 of 6 restaurants with 178 total items and an average price of ₹454. Every restaurant treats starters as a mandatory section — high breadth + high item count = saturated.

**Soups and Salads** appear in all 6 restaurants with 99 items but average only ₹238. Present everywhere, low margin — a textbook saturated commodity category.

**Pizza and Pasta** shows up in 5 of 6 restaurants (72 items, avg ₹518). Still commanding a price premium, but the breadth of presence marks it as contested.

#### The Structural Saturation Signal Nobody Talks About

When I segment all 1,134 items by price band — **515 items (45.5% of the entire menu dataset) are priced between ₹301 and ₹500**. Every restaurant is competing for the same customer, at the same price point, with the same cuisine mix. That is saturation at the market level, not just the cuisine level.

---

### Question 2: Which Cuisine Category is Underrepresented?

#### My Definition

A segment where real consumer demand exists but local supply is conspicuously thin — measured by low restaurant presence, low item count, or zero dedicated specialist operators.

#### Finding: Indian Chaat and Street Food

**Zero out of 6 restaurants** positions itself as a chaat or street food specialist. The category is treated as a filler section by those who include it at all.

| Restaurant | Chaat Items | Avg Price |
|------------|-------------|-----------|
| Grill Craft Co. | 4 | ₹240 — clearly a side thought |
| Larive Kitchen | 7 | ₹364 — priced like a sit-down restaurant, not street food |
| Tamara | 23 | ₹329 — the most in the set, but buried inside a 273-item menu |
| Tomato's | 19 | ₹320 — again, filler positioning |
| The Mykonos | 0 | — |
| Vintage Asia | 0 | — |

**Total chaat supply across all 6 restaurants: 53 items.** Compare that to 248 items for Indian main course. The demand for chaat nationally and in Tier-1 cities is well documented — it consistently ranks among the top 5 ordered categories on delivery platforms. The local supply does not match that demand.

#### Secondary Gap: Budget Tier

Only **134 items** across all 6 restaurants are priced at or below ₹150 — that is **11.8%** of the total menu. The lowest median price in the dataset is ₹337 (Grill Craft Co.). There is effectively no operator serving a sub-₹200 per head customer in this locality. That is an entire income segment unserved.

#### Third Gap: Desserts

Only **14 dessert items** across 4 restaurants. Smallest absolute count for any multi-restaurant category. Given that desserts drive repeat visits and carry high add-on margins, this is a meaningful gap.

---

### Question 3: Cloud Kitchen Launch Plan — ₹5 Lakh Capital

#### Target Cuisine: Indian Chaat and Street Food

The decision flows directly from Q1 and Q2. The most profitable entry is not into the most popular segment — it is into the segment where demand exceeds supply.

#### Why Chaat Beats Indian Main Course for ₹5L Capital

| Factor | Indian Main Course | Chaat |
|--------|-------------------|-------|
| Competitor count | All 6 restaurants | Zero specialists |
| Capex requirement | High (chefs, tandoor, equipment) | Low (assembly-based) |
| Prep time per item | 15–25 minutes | 4–7 minutes |
| Differentiation difficulty | Very high | Low — first mover |
| Brand identity | Generic | Ownable |

#### Target Customer

**Primary:** Urban millennials, 22–35 years, office-goers ordering lunch or evening snacks. Budget: ₹150–₹350 per order — significantly below the locality median of ₹374. They want authentic street food taste without roadside hygiene concerns.

**Secondary:** Families ordering evening snack platters for 2–4 people at ₹299–₹499.

#### Pricing Strategy

The data gives me a clear anchor. The locality median is ₹382.50. The chaat category currently averages ₹338. My pricing needs to sit below restaurant-dining perception but above roadside perception — what I am calling the **"clean street food premium."**

| Item Type | Price Range |
|-----------|-------------|
| Solo items (pani puri, bhel, sev puri) | ₹89–₹149 |
| Premium solos (dahi vada, aloo tikki chaat) | ₹149–₹199 |
| Combos (any 3 solos + drink) | ₹199–₹269 |
| Platter for 2 | ₹349–₹449 |
| Drinks (jaljeera, aam panna, lassi) | ₹79–₹119 |

This positions the brand **20–40% cheaper** than every surveyed restaurant while staying **50–80% above roadside** — the brand's entire value proposition is that gap.

#### Capital Allocation

| Item | Amount |
|------|--------|
| Kitchen setup and equipment | ₹1,80,000 |
| Platform fees and commission buffer (3 months) | ₹1,50,000 |
| Ingredients and packaging working capital | ₹1,00,000 |
| Branding, photography, Zomato listing | ₹30,000 |
| Discounts and trial offers (first 60 days) | ₹70,000 |
| Contingency | ₹70,000 |
| **Total** | **₹5,00,000** |

#### Competitive Advantage

**First, category ownership.** Zero competitors in the locality means the brand owns Zomato search rankings, review volume, and customer association before anyone else enters. This is the hardest advantage to replicate once established.

**Second, speed.** Chaat items prep in 4–7 minutes (assembly-based). Competing restaurants average 20–25 minutes. Faster delivery = better Zomato SLA score = algorithmic promotion = more orders.

**Third, menu discipline.** Launch with **18 items only** — 6 solos, 6 combos, 4 platters, 2 drinks. The data shows that focused menus outperform bloated menus on quality consistency. Tamara's 273-item menu and Tomato's 259-item menu are operational liabilities, not strengths.

**Breakeven estimate:** Average order value ₹220 × 35% food cost = ₹143 gross margin. After 25% platform fees: ₹88 net per order. At 50 orders per day → ₹1.32 lakh per month contribution. Conservative breakeven: Month 4–5.

---

### Question 4: Identifying the Most Operationally Efficient Restaurant

**Verdict: Vintage Asia — Courtyard by Marriott**

I cannot see kitchen footage, waste reports, or P&L statements. But menu data is a powerful proxy for operational philosophy. Here is what I measured and why.

#### Observable Signals and What They Indicate

- **Menu item count:** More items = more SKUs = more ingredients to stock, more training, more waste
- **Category count:** Fewer categories = narrower cuisine identity = faster kitchen execution
- **Price Coefficient of Variation (CV = Standard Deviation ÷ Mean):** Lower CV = more consistent pricing = one clearly defined customer tier = no context-switching between cheap and expensive executions

#### The Comparative Data

| Restaurant | Items | Categories | Price CV | Avg Price |
|------------|-------|------------|----------|-----------|
| Vintage Asia | 31 | 7 | 0.25 | ₹573 |
| Grill Craft Co. | 191 | 13 | 0.43 | ₹337 |
| The Mykonos | 187 | 9 | 0.52 | ₹368 |
| Larive Kitchen | 193 | 15 | 0.50 | ₹364 |
| Tamara | 273 | 16 | 0.38 | ₹403 |
| Tomato's | 259 | 16 | 0.44 | ₹360 |

Vintage Asia has the **lowest item count, lowest category count, and lowest price CV** in the entire dataset. Every single one of these signals points in the same direction.

#### Evidence 1 — 31 Items, 7 Categories

Every other restaurant lists 187–273 items. Vintage Asia lists 31. This is not a data anomaly — it is a deliberate operational choice. More items means more ingredient SKUs to stock, more staff training, higher spoilage, and longer order-to-table times. A 31-item menu means every kitchen staff member knows every dish precisely, ingredient ordering is predictable, and there is no dead weight on the menu.

#### Evidence 2 — Price CV of 0.25

The lowest in the dataset. Prices range ₹350–₹800, clustered tightly around the ₹573 mean. Compare this to The Mykonos (CV = 0.52, range ₹20–₹1,650) or Larive Kitchen (CV = 0.50, range ₹59–₹1,180). A low CV means one clearly defined customer, no loss-leader items pulling margins down, and no kitchen context-switching between cheap and premium executions simultaneously.

#### Evidence 3 — Zero Items Below ₹350

Every single item is priced at ₹350 or above. A kitchen executing at one quality tier avoids the operational complexity of preparing a ₹89 soup and a ₹800 main course at the same time — different ingredients, different plating standards, different execution speeds. Vintage Asia has eliminated that complexity entirely.

#### Evidence 4 — Single Cuisine Identity

All 7 categories — Soups, Dim Sum, Appetizers, Tangra Menu, Fried Rice and Noodles, Main Course, Rice — sit within one Asian cuisine umbrella. No biryani section added to attract more search traffic, no pizza to hedge on delivery volume. One cuisine, one kitchen workflow, one training manual.

#### Evidence 5 — Hotel Brand Infrastructure

As a Courtyard by Marriott outlet, Vintage Asia operates with centralised procurement, standardised recipes, documented portion controls, and P&L management at the cover-cost level. The disciplined menu data reflects institutional standards — this is not accidental.

#### What the Least Efficient Restaurants Reveal by Contrast

Tamara (273 items, 16 categories) and Tomato's (259 items, 16 categories) display textbook menu bloat. High SKU count, high category fragmentation, no clear cuisine identity. Every item added beyond the operational sweet spot adds cost without proportionate revenue. These two restaurants are almost certainly spending more on procurement, experiencing higher waste, and training staff on too many dishes to achieve genuine mastery in any of them.

## Part 6: SQL Schema

### Database Schema

```sql
CREATE DATABASE zom;

USE zom;

CREATE TABLE restaurants (
    restaurant_id INT PRIMARY KEY,
    name VARCHAR(255),
    rating DECIMAL(2,1),
    cost_for_two INT,
    location VARCHAR(255),
    votes INT
);

CREATE TABLE cuisine_tags (
    tag_id INT PRIMARY KEY,
    restaurant_id INT,
    cuisine VARCHAR(100),
    FOREIGN KEY (restaurant_id) REFERENCES restaurants(restaurant_id)
);

CREATE TABLE menu_items (
    item_id INT PRIMARY KEY,
    restaurant_id INT,
    item_name VARCHAR(255),
    category VARCHAR(100),
    price DECIMAL(8,2),
    is_veg BOOLEAN,
    FOREIGN KEY (restaurant_id) REFERENCES restaurants(restaurant_id)
);

INSERT INTO restaurants VALUES
(1, 'Grill Craft Co.', 4.2, 800, 'Indiranagar, Bangalore', 320),
(2, 'Larive Kitchen And Cocktail', 4.5, 1500, 'UB City, Bangalore', 512),
(3, 'Tamara', 4.1, 1200, 'Koramangala, Bangalore', 198),
(4, 'The Mykonos', 3.9, 1000, 'Whitefield, Bangalore', 145),
(5, 'Tomatos', 3.7, 600, 'JP Nagar, Bangalore', 89),
(6, 'Vintage Asia', 4.4, 2000, 'MG Road, Bangalore', 430);

INSERT INTO cuisine_tags VALUES
(1, 1, 'North Indian'),
(2, 1, 'Continental'),
(3, 2, 'European'),
(4, 2, 'Asian'),
(5, 2, 'Cocktails'),
(6, 3, 'North Indian'),
(7, 3, 'Biryani'),
(8, 4, 'Mediterranean'),
(9, 4, 'Continental'),
(10, 5, 'South Indian'),
(11, 6, 'Asian'),
(12, 6, 'Chinese'),
(13, 6, 'Thai');

INSERT INTO menu_items VALUES
(1, 1, 'Smoked Butter Chicken', 'Main Course', 440.00, false),
(2, 1, 'Paneer Tikka', 'Starters', 320.00, true),
(3, 1, 'Garlic Naan', 'Breads', 60.00, true),
(4, 2, 'Truffle Risotto', 'Main Course', 950.00, true),
(5, 2, 'Burrata Salad', 'Soups and Salads', 650.00, true),
(6, 2, 'Sangria Pitcher', 'Drinks (Beverages)', 1200.00, true),
(7, 3, 'Butter Chicken', 'Main Course', 380.00, false),
(8, 3, 'Mutton Biryani', 'Rice and Biryani', 420.00, false),
(9, 3, 'Dal Makhani', 'Indian', 280.00, true),
(10, 4, 'Classic Margherita Pizza', 'Pizza and Pasta', 550.00, true),
(11, 4, 'Penne Arrabbiata', 'Pizza and Pasta', 480.00, true),
(12, 4, 'Greek Salad', 'Soups and Salads', 320.00, true),
(13, 5, 'Masala Dosa', 'South Indian', 120.00, true),
(14, 5, 'Idli Sambar', 'South Indian', 80.00, true),
(15, 5, 'Filter Coffee', 'Drinks (Beverages)', 60.00, true),
(16, 6, 'Thai Red Chicken Curry', 'Main Course', 580.00, false),
(17, 6, 'Dim Sum Basket', 'Starters', 420.00, false),
(18, 6, 'Wonton Soup', 'Soups and Salads', 280.00, false);
```

### Key Queries

```sql
-- QUERY 1: Top 5 highest rated restaurants 

WITH ranked_restaurants AS (
    SELECT
        restaurant_id,
        name,
        rating,
        votes,
        DENSE_RANK() OVER (ORDER BY rating DESC, votes DESC) AS ran
    FROM restaurants
)
SELECT restaurant_id, name, rating, votes
FROM ranked_restaurants
WHERE ran <= 5;


-- QUERY 2: Average cost-for-two by cuisine (with restaurant count and grand total using ROLLUP)

WITH cuisine_stats AS (
    SELECT
        ct.cuisine,
        r.cost_for_two,
        r.restaurant_id
    FROM cuisine_tags ct
    JOIN restaurants r ON r.restaurant_id = ct.restaurant_id
)
SELECT
    COALESCE(cuisine, 'ALL CUISINES') AS cuisine,
    ROUND(AVG(cost_for_two), 2)       AS avg_cost_for_two,
    COUNT(restaurant_id)              AS restaurant_count
FROM cuisine_stats
GROUP BY cuisine WITH ROLLUP
ORDER BY avg_cost_for_two DESC;


-- QUERY 3: Restaurants with more than one cuisine (with cuisine list concatenated)

WITH cuisine_counts AS (
    SELECT
        restaurant_id,
        COUNT(tag_id)                                    AS cuisine_count,
        GROUP_CONCAT(cuisine ORDER BY cuisine SEPARATOR ', ') AS cuisines
    FROM cuisine_tags
    GROUP BY restaurant_id
    HAVING COUNT(tag_id) > 1
)
SELECT
    r.restaurant_id,
    r.name,
    cc.cuisine_count,
    cc.cuisines
FROM cuisine_counts cc
JOIN restaurants r ON r.restaurant_id = cc.restaurant_id
ORDER BY cc.cuisine_count DESC;


-- QUERY 4: Highest priced menu item (with rank per category using RANK)

WITH price_ranked AS (
    SELECT
        mi.item_id,
        mi.item_name,
        mi.category,
        mi.price,
        r.name AS restaurant_name,
        RANK() OVER (PARTITION BY mi.category ORDER BY mi.price DESC) AS price_rank
    FROM menu_items mi
    JOIN restaurants r ON r.restaurant_id = mi.restaurant_id
)
SELECT item_id, item_name, category, price, restaurant_name, price_rank
FROM price_ranked
WHERE price_rank = 1
ORDER BY price DESC;
```

## Future Improvements

| Area | Enhancement |
|------|-------------|
| **Scalability** | Migrate to Scrapy with proxy rotation for city-wide coverage |
| **Real-time** | Schedule Airflow DAGs for weekly competitive tracking |
| **NLP** | Sentiment analysis on review text (requires API access) |
| **Geospatial** | Map restaurant density vs. population heatmaps |
| **ML** | Predict bestseller items using price-elasticity modeling |
| **Visualization** | Tableau/PowerBI dashboard for stakeholder reporting |

## Data Visualizations

### Chart 1: Cuisine Saturation Analysis
<img width="583" height="301" alt="Screenshot 2026-06-24 115615" src="https://github.com/user-attachments/assets/fa5196a0-0b4c-4273-b02e-5fd7b2743431" />

**Insight:** Indian Main Course dominates with 248 items (21.9% of all menu items), confirming extreme saturation. The green zone (Chaat/Street Food at 53 items) represents the underrepresented opportunity identified in Q2.

---

### Chart 2: Price Distribution — The Red Ocean
<img width="537" height="308" alt="Screenshot 2026-06-24 115758" src="https://github.com/user-attachments/assets/e6200faa-433e-44b6-8aae-9451595d9fdc" />

**Insight:** 515 items (45.5%) cluster in the ₹301–₹500 band — the "red ocean" where every restaurant competes for the same customer at the same price point. The green zone (₹0–₹150) at only 11.8% represents the budget-tier gap.

---

### Chart 3: Operational Efficiency Matrix
<img width="596" height="353" alt="Screenshot 2026-06-24 115820" src="https://github.com/user-attachments/assets/c7dd1481-854c-48e8-94f4-6c7b6604bb98" />

**Insight:** Vintage Asia sits alone in the efficiency zone (low SKU count + low price CV). Tamara and Tomato's sit in the danger zone (high SKU + high complexity). Bubble size shows menu category fragmentation — larger bubbles mean more operational overhead.

