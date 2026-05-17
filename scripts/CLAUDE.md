# Data Sources & Pipeline

## Geographic Scope

All scripts target the **Raleigh-Durham, NC** region, spanning two Redfin metros:
- **Raleigh-Cary MSA** (Redfin metro code 39580)
- **Durham-Chapel Hill MSA** (Redfin metro code 20500)

Configured in `config.json`:
- `metro` — primary metro descriptor (display name, map center, market slug used by city-level fetchers)
- `redfin_metros` — list of all metros pulled for city/zip-level **market trend** data
- `cities[]` — every city pulled for individual sold + active listings (spans both metros)
- `metro.baseline_cities` — cities shown with full Buyer Favorability breakdown on Market Pulse

The sold and active fetchers iterate `cities[]` by region_id, so they pull every configured city regardless of which metro it belongs to. Only `fetch_market_trends.py` filters by metro code, and it iterates over `redfin_metros`.

## Entry Point

```
python3 scripts/run_pipeline.py                 # run all tiers
python3 scripts/run_pipeline.py --tier active    # just active listings
python3 scripts/run_pipeline.py --tier sold      # just sold homes
python3 scripts/run_pipeline.py --tier trends    # just market trends
python3 scripts/run_pipeline.py --if-stale       # only run tiers that are due
python3 scripts/run_pipeline.py --sold-days 30   # override sold-homes window
```

## Tiered Update Cadences

Data sources refresh at different rates, tracked in `data/pipeline_state.json`:

| Tier | Cadence | Script |
|------|---------|--------|
| `market_trends` | ~2 weeks (336h) | `fetch_market_trends.py` |
| `sold_homes` | Daily (24h) | `fetch_sold_listings.py` |
| `active_listings` | Twice daily (12h) | `fetch_active_listings.py` |

`start.sh` runs `--if-stale` before serving the dashboard, so data refreshes automatically when you open the dashboard and data is due.

## Sources

### 1. Redfin Market Data (Market Trends)
- **Source:** S3 bulk TSV downloads
  - City: `redfin_market_tracker/city_market_tracker.tsv000.gz`
  - Zip: `redfin_market_tracker/zip_code_market_tracker.tsv000.gz`
- **Update cadence:** Weekly (city/county), monthly (zip)
- **Key fields:** median_sale_price, median_list_price, median_ppsf, homes_sold, pending_sales, new_listings, inventory, months_of_supply, median_dom, avg_sale_to_list, price_drops
- **Access:** Public S3, no key required.
- **Script:** `fetch_market_trends.py`

### 2. Redfin Sold Homes (Transaction Prices)
- **Endpoint:** `https://www.redfin.com/stingray/api/gis-csv` with query params
- **Update cadence:** As sales close
- **Geographic scope:** Cities listed in `config.json → cities[]` (iterated by region_id)
- **Key fields:** sale_price, sold_date, address, beds, baths, sqft, lot_size, year_built, price_per_sqft, days_on_market, hoa, property_type, mls_number, lat/lng
- **Limit:** 350 rows/request — splits by city + property type to stay under cap
- **Access:** Public, no key required.
- **Script:** `fetch_sold_listings.py`

### 3. Redfin Active Listings (For-Sale Homes)
- **Endpoint:** Same `gis-csv` endpoint as sold, with `status=1` (active)
- **Update cadence:** Twice daily + on-demand
- **Geographic scope:** Same cities
- **Key fields:** list_price, address, beds, baths, sqft, lot_size, year_built, price_per_sqft, days_on_market, hoa, property_type, mls_number, lat/lng
- **Limit:** 350 rows/request — splits by city + property type + price band
- **Price tracking:** `data/active_listings_tracker.json` records first_seen, price history, and delisting per MLS#
- **Access:** Public, no key required.
- **Script:** `fetch_active_listings.py`

## How They Combine

```
Redfin Sold (sale prices + DOM)
       |
       | CONTEXT from
       |
Redfin Market (zip/city trends)
       |
       | Sold comps used for
       |
Redfin Active (current asking prices)
```

- **Redfin Market:** Provides macro context per zip/city.
- **Sold <-> Active (in dashboard):** Sold comps inform whether an active listing's asking price is reasonable.

## Output

All data lands in `data/` as CSV:
- `data/redfin_market_city.csv`
- `data/redfin_market_zip.csv`
- `data/redfin_sold.csv`
- `data/redfin_active.csv`
- `data/active_listings_tracker.json` (price history sidecar — for-sale)
- `data/pipeline_state.json` (tier freshness tracker)
- `data/dashboard_data.json` (assembled for dashboard consumption)
