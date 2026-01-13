# EnergizeCT to Home Assistant Cheapest Rate Automation

This project automatically collects electric supplier rates from EnergizeCT, determines the cheapest available offer, and pushes the results into Home Assistant as sensors. It runs fully unattended once per day using cron.

EnergizeCT blocks normal API scraping, so this uses Playwright with a real browser running under Xvfb to capture the site’s internal API response.

---

## What This Does

Every day the system:

1. Launches a real browser via Xvfb
2. Loads the EnergizeCT rate board page
3. Captures the internal API response
4. Saves raw rate data to disk
5. Determines the cheapest offer
6. Pushes the results into Home Assistant sensors

Home Assistant sensors created:

- sensor.ct_electric_rate
- sensor.ct_electric_supplier
- sensor.ct_electric_term

---

## Requirements

Python VM (Debian 12 recommended)

System packages:
- python3
- python3-venv
- xvfb
- curl

Home Assistant:
- Accessible via HTTP
- Long-Lived Access Token created

All files are stored in:

/opt/energizect/

---

## Install System Packages

```bash
apt update
apt install -y python3 python3-venv xvfb curl
```

---

## Create Project Directory

```bash
mkdir -p /opt/energizect
cd /opt/energizect
```

---

## Create Python Virtual Environment

```bash
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install playwright requests
playwright install firefox
```

---

## fetch_rates.py

```python
import json
from playwright.sync_api import sync_playwright

RATE_BOARD_URL = "https://www.energizect.com/rate-board/compare-energy-supplier-rates?customerClass=1201&monthlyUsage=750&planTypeEdc=1191"
API_URL_PART = "/ectr_search_api/offers"
OUTPUT_FILE = "/opt/energizect/rates_raw.json"

def main():
    print("Launching browser and waiting for API response")
    captured = []

    with sync_playwright() as p:
        browser = p.firefox.launch(headless=False)
        context = browser.new_context()
        page = context.new_page()

        def handle_response(resp):
            if API_URL_PART in resp.url and resp.status == 200:
                try:
                    captured.append(resp.json())
                except:
                    pass

        page.on("response", handle_response)

        page.goto(RATE_BOARD_URL, wait_until="domcontentloaded", timeout=60000)
        page.wait_for_timeout(15000)

        if not captured:
            print("API did not return data")
            return

        out = {
            "last_updated": page.evaluate("new Date().toISOString()"),
            "data": captured[0]
        }

        with open(OUTPUT_FILE, "w") as f:
            json.dump(out, f, indent=2)

        print("Rates captured successfully")
        browser.close()

if __name__ == "__main__":
    main()
```

---

## process_rates.py

```python
import json

RAW_FILE = "/opt/energizect/rates_raw.json"

def main():
    with open(RAW_FILE) as f:
        raw = json.load(f)

    offers = raw.get("data", {}).get("results", [])
    parsed = []

    for o in offers:
        try:
            parsed.append({
                "supplier": o.get("supplier"),
                "rate": float(o.get("rate")),
                "term": o.get("termOfOffer"),
            })
        except:
            pass

    if not parsed:
        print("No offers parsed")
        return

    cheapest = min(parsed, key=lambda x: x["rate"])

    print(f"Cheapest: {cheapest['supplier']} @ {cheapest['rate']}")
    print("All offers:")

    for p in sorted(parsed, key=lambda x: x["rate"]):
        print(f"- {p['supplier']}: {p['rate']} ({p['term']})")

if __name__ == "__main__":
    main()
```

---

## ha_read_cheapest.py

```python
import json

RAW_FILE = "/opt/energizect/rates_raw.json"

with open(RAW_FILE) as f:
    raw = json.load(f)

offers = raw.get("data", {}).get("results", [])

cheapest = None

for o in offers:
    try:
        rate = float(o.get("rate"))
        if not cheapest or rate < cheapest["rate"]:
            cheapest = {
                "rate": rate,
                "supplier": o.get("supplier"),
                "term": o.get("termOfOffer")
            }
    except:
        pass

if cheapest:
    print(f"{cheapest['rate']:.5f}|{cheapest['supplier']}|{cheapest['term']}")
else:
    print("0|None|None")
```

---

## push_to_ha.py

```python
import requests
import subprocess

HA_URL = "http://HOME_ASSISTANT_IP:8123"
HA_TOKEN = "PASTE_LONG_LIVED_TOKEN_HERE"

def get_cheapest():
    result = subprocess.check_output(
        ["python", "/opt/energizect/ha_read_cheapest.py"],
        text=True
    ).strip()
    rate, supplier, term = result.split("|")
    return rate, supplier, term

def update_sensor(entity_id, state, attributes=None):
    url = f"{HA_URL}/api/states/{entity_id}"
    headers = {
        "Authorization": f"Bearer {HA_TOKEN}",
        "Content-Type": "application/json",
    }
    payload = {
        "state": state,
        "attributes": attributes or {}
    }
    r = requests.post(url, headers=headers, json=payload)
    r.raise_for_status()

def main():
    rate, supplier, term = get_cheapest()

    update_sensor(
        "sensor.ct_electric_rate",
        rate,
        {
            "unit_of_measurement": "$/kWh",
            "supplier": supplier,
            "term": term
        }
    )

    update_sensor("sensor.ct_electric_supplier", supplier)
    update_sensor("sensor.ct_electric_term", term)

    print("HA updated successfully")

if __name__ == "__main__":
    main()
```

---

## run_daily.sh

```bash
#!/bin/bash
set -e

source /opt/energizect/venv/bin/activate

xvfb-run -s "-screen 0 1920x1080x24" python /opt/energizect/fetch_rates.py
python /opt/energizect/process_rates.py
python /opt/energizect/push_to_ha.py
```

Make executable:

```bash
chmod +x /opt/energizect/run_daily.sh
```

---

## Manual Test

```bash
/opt/energizect/run_daily.sh
```

---

## Cron Automation (Daily at 9 AM)

```bash
crontab -e
```

Add:

```cron
0 9 * * * /opt/energizect/run_daily.sh >> /opt/energizect/cron.log 2>&1
```

---

## Debugging

Check cron output:

```bash
cat /opt/energizect/cron.log
```

Check last update time:

```bash
stat /opt/energizect/rates_raw.json
```

---

## Home Assistant Dashboard Card

```yaml
type: entities
title: CT Electric Rate
entities:
  - entity: sensor.ct_electric_rate
    name: Current Rate
  - entity: sensor.ct_electric_supplier
    name: Supplier
  - entity: sensor.ct_electric_term
    name: Term Length
  - entity: sensor.ct_electric_rate
    name: Last Updated
    secondary_info: last-changed
```

---

## Notes

EnergizeCT actively blocks automated scraping. If this breaks, open DevTools on the rate board page and look for requests to:

/ectr_search_api/offers

Update capture logic accordingly.
