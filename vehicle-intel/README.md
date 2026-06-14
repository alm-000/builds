# vehicle-intel

**Theme:** scrape→enrich→score · full-stack

## Problem

Buying a used vehicle without proper due diligence risks purchasing a stolen, written-off, or finance-encumbered car. The data to answer these questions exists across multiple sources (NHTSA, Vincario, UK vehicle registries) but requires piecing together multiple paid and free APIs, normalising inconsistent schemas, and surfacing alerts clearly.

## Approach

A multi-source VIN decode pipeline that layers data sources by cost and coverage:

1. **Structural decode** — pure Python, no API: WMI (manufacturer + country), model year from position 10 (with 30-year cycle disambiguation), VW Group assembly plant lookup. Always runs, zero cost.
2. **NHTSA vPIC** — free US government API. Good for US-spec vehicles; limited for EU VINs (returns partial data).
3. **Vincario** — paid API: full European spec decode, stolen vehicle check, market value range. Runs if `VINCARIO_API_KEY` is configured.
4. **ukvehicledata.co.uk** — paid UK-specific API: finance outstanding, write-off category, keeper history, mileage anomaly flag.

Each source fills in gaps from earlier sources — later sources override only where they have a value.

## Architecture

```python
decode_vin(vin)
  → decode_vin_structure()  # Pure Python: WMI, year, plant — always runs
  → _fetch_nhtsa()          # Free: spec fields, merge into result
  → _fetch_vincario()       # Paid: decode + stolen + market value
  → _fetch_ukvehicledata()  # Paid UK: finance, write-off, keepers
  # Returns merged dict; alerts list populated on stolen/finance/write-off flags
```

## Real results / metrics

- Structural decode: instant, handles all 17-char VINs; covers 30+ WMIs across DE, UK, FR, IT, JP, KR, SE, CZ, ES.
- Full pipeline (all 4 sources): ~4–6 seconds, returning a merged dict with up to 35 fields.
- Alert system surfaces `STOLEN`, `FINANCE OUTSTANDING`, `WRITE-OFF Cat N`, `MILEAGE ANOMALY` — the highest-risk flags for private buyers.

## Snippet

Pure-Python structural decoder — no API, no key required:

```python
# WMI → (manufacturer, country) lookup table (partial)
_WMI = {
    "WVW": ("Volkswagen",    "Germany"),
    "WAU": ("Audi",          "Germany"),
    "WBA": ("BMW",           "Germany"),
    "WDD": ("Mercedes-Benz", "Germany"),
    "SAJ": ("Jaguar",        "UK"),
    "SAL": ("Land Rover",    "UK"),
    "SCC": ("Aston Martin",  "UK"),
    "JHM": ("Honda",         "Japan"),
    "JT2": ("Toyota",        "Japan"),
    "KMH": ("Hyundai",       "South Korea"),
    "YV1": ("Volvo",         "Sweden"),
    # ... 30+ WMIs total
}

# Model year codes: position 10 cycles every 30 years (I, O, Q, U, Z not used)
_YEAR_CODES: dict[str, list[int]] = {
    'A': [1980, 2010], 'B': [1981, 2011], 'C': [1982, 2012],
    'D': [1983, 2013], 'E': [1984, 2014], 'F': [1985, 2015],
    # ... full table in repo
}


def decode_vin_structure(vin: str) -> dict:
    """Decode VIN into structural components. Pure Python, no API.

    Returns dict with: wmi, manufacturer, country_of_manufacture,
    model_year, assembly_plant, serial_number, is_european_vin.
    """
    vin = vin.upper().strip()
    if len(vin) != 17:
        return {"error": f"VIN must be 17 characters, got {len(vin)}"}

    wmi        = vin[:3]
    vds        = vin[3:9]
    check_char = vin[8]
    year_char  = vin[9]
    plant_char = vin[10]
    serial     = vin[11:]

    manufacturer, country = _WMI.get(wmi, (None, None))

    # European VINs use ZZZ filler in VDS and don't implement US check digit
    is_european = vds[:3] == 'ZZZ' or check_char == 'Z'

    model_year = _resolve_year(year_char)

    # Plant decode (VW Group); others return raw code
    plant = _VW_PLANTS.get(plant_char) if wmi[:2] in ('WV', 'WA', 'WB', 'WD', 'WP', 'TM', 'VS') else None

    out = {
        "vin":                    vin,
        "wmi":                    wmi,
        "manufacturer":           manufacturer,
        "country_of_manufacture": country,
        "vds":                    vds,
        "model_year":             model_year,
        "assembly_plant_code":    plant_char,
        "serial_number":          serial,
        "is_european_vin":        is_european,
    }
    if plant:
        out["assembly_plant"] = plant

    return {k: v for k, v in out.items() if v is not None}
```
