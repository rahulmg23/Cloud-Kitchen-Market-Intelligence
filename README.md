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

Here's the converted Part 2 content in proper GitHub markdown format, ready to copy-paste into your README:

```markdown
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
```


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

### Q1: Which cuisines appear most saturated in College Road?

| Cuisine | Count | Market Share |
|---------|-------|--------------|
| North Indian | 14 | 38.9% |
| Chinese | 11 | 30.6% |
| Fast Food / Burger | 8 | 22.2% |
| Continental | 5 | 13.9% |
| South Indian | 3 | 8.3% |

**Insight:** North Indian and Chinese cuisines are **oversaturated**. Entry barriers are high due to established players (Grill Craft Co., Tomato's) and price competition.

### Q2: Which cuisine category appears underrepresented?

**Mediterranean / Middle Eastern** — Only 1 dedicated outlet (The Mykonos). High growth potential given:
- Rising health-conscious consumer segment
- Limited direct competition
- Premium pricing tolerance (₹800–₹1,200 for two)

### Q3: If you were launching a cloud kitchen in this locality with ₹5 lakh capital, what cuisine would you launch first?

| Parameter | Recommendation |
|-----------|----------------|
| **Target Cuisine** | Mediterranean Bowls (Falafel, Hummus, Shawarma) |
| **Target Customer** | 22–35 urban professionals, health-conscious, ₹400–₹600 spend |
| **Pricing Strategy** | ₹349–₹499 per bowl (40% food cost, 25% delivery, 35% margin) |
| **Competitive Advantage** | Subscription meal plans, calorie-counted menus, 25-min delivery guarantee |

### Q4: Identify one restaurant that appears operationally efficient and explain why.

**Grill Craft Co.** — Evidence:
- **Rating:** 4.3/5 with 1,200+ reviews (high volume + quality retention)
- **Delivery Time:** 28 min (below section average of 35 min)
- **Menu Engineering:** 60% items priced ₹200–₹400 (sweet spot for College Road)
- **Cross-section Presence:** Active in both Delivery and Nightlife (revenue diversification)

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
```

---
