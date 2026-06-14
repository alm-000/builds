# uk-neighbourhood

**Theme:** scrape→enrich→score · full-stack

## Problem

When evaluating a property, the neighbourhood context — crime rate, nearby antisocial behaviour, and house price trends — is scattered across multiple government APIs with no convenient unified view. Manually checking each site for every candidate property is time-consuming and inconsistent.

## Approach

A Python module that calls four free public APIs, filters and weights the results, and returns a structured neighbourhood report with a 0–100 safety score.

**Data sources (all free, no API key required):**

| Source | Data |
|--------|------|
| postcodes.io | Lat/lng + admin district from postcode |
| data.police.uk | Street-level crime data (3-month window) |
| Land Registry UKHPI | House price trends by property type |
| OSRM (OpenStreetMap) | Driving time to a target location |

**Crime scoring logic:**

- Fetches all crimes within 500m of the property coordinates (Haversine filter — not just the postcode polygon, which can be large).
- Separates "residential concern" crimes (burglary, vehicle crime, ASB, robbery, violence) from town-centre crimes (shoplifting, drugs, public order) that inflate area stats without affecting home safety.
- Applies extra weight to antisocial behaviour within 200m — the closest proxy for problematic immediate neighbours.
- Returns a 0–100 score: 100 = very safe; deductions for residential crime rate, doorstep ASB, and burglary.

## Architecture

```python
get_neighbourhood_report(postcode)
  → postcode_lookup()         # postcodes.io → lat/lng + district
  → get_crimes()              # police API → filtered to CRIME_RADIUS_M (500m)
  → summarise_crimes()        # Haversine filter + residential concern separation
  → neighbourhood_score()     # 0-100 weighted score
  → get_price_trends()        # Land Registry UKHPI → price by type + annual change
  → format_for_notion()       # structured text summary
```

## Real results / metrics

- Processes a full neighbourhood report in ~3–5 seconds (three sequential API calls, each with a 200ms polite delay).
- RAM: negligible — pure Python with stdlib `urllib`, no pandas or heavyweight dependencies.
- Designed to handle ~50–100 postcodes in a single session with a `price_cache` dict to avoid redundant Land Registry calls for adjacent properties in the same district.

## Snippet

Haversine filter + crime summariser with residential/ASB weighting:

```python
CRIME_RADIUS_M = 500  # Only count crimes within this radius of the property

def _haversine(lat1, lng1, lat2, lng2):
    """Distance in metres between two coordinates."""
    R = 6371000
    dlat = math.radians(lat2 - lat1)
    dlng = math.radians(lng2 - lng1)
    a = (math.sin(dlat / 2) ** 2 +
         math.cos(math.radians(lat1)) * math.cos(math.radians(lat2)) *
         math.sin(dlng / 2) ** 2)
    return R * 2 * math.asin(math.sqrt(a))


def get_crimes(lat: float, lng: float, months: int = 3) -> list[dict]:
    """Fetch street-level crimes near a location for the last N months.

    Filters to crimes within CRIME_RADIUS_M of the property coordinates.
    Each crime gets a '_distance' and '_street' field added.
    """
    all_crimes = []
    from datetime import date, timedelta
    today = date.today()
    for i in range(2, 2 + months):  # 2-month data lag
        d = today.replace(day=1) - timedelta(days=30 * i)
        month_str = d.strftime('%Y-%m')
        time.sleep(0.2)
        url = f"{POLICE_API}/crimes-street/all-crime?lat={lat}&lng={lng}&date={month_str}"
        crimes = _get_json(url)
        if isinstance(crimes, list):
            for c in crimes:
                try:
                    clat = float(c['location']['latitude'])
                    clng = float(c['location']['longitude'])
                    dist = _haversine(lat, lng, clat, clng)
                    if dist <= CRIME_RADIUS_M:
                        c['_distance'] = round(dist)
                        c['_street'] = c.get('location', {}).get('street', {}).get('name', 'Unknown')
                        all_crimes.append(c)
                except (KeyError, ValueError):
                    pass
    return all_crimes


def neighbourhood_score(crime_report: dict) -> int:
    """Calculate a 0-100 neighbourhood score from crime data.

    100 = very safe, 0 = very high crime.
    """
    if not crime_report or crime_report['total'] == 0:
        return 80  # No data = assume decent (rural areas have no data)

    score = 100.0
    months = max(crime_report['months'], 1)
    cats = crime_report.get('by_category', {})

    # Residential crimes: -2 pts per incident per month
    res_avg = crime_report.get('residential_avg', 0)
    score -= res_avg * 2

    # ASB on doorstep (within 200m): -5 pts per incident per month
    asb_nearby_avg = crime_report.get('asb_nearby_avg', 0)
    score -= asb_nearby_avg * 5

    # Burglary: -4 pts per incident per month (very relevant)
    burglary = cats.get('Burglary', 0)
    score -= (burglary / months) * 4

    # Vehicle crime: -3 pts per incident per month
    vehicle = cats.get('Vehicle Crime', 0)
    score -= (vehicle / months) * 3

    return max(0, min(100, round(score)))
```
