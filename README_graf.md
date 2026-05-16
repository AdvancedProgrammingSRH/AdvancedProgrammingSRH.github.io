# Python -> InfluxDB -> Grafana: Setup Steps

End-to-end flow that runs entirely on your laptop. The shape is:
**your Python script -> InfluxDB (storage) -> Grafana (display)**.
InfluxDB and Grafana run in Docker so there's nothing to install on
Windows beyond Docker Desktop.

---

## Step 1 — Start InfluxDB and Grafana

Save this as `docker-compose.yml` in any folder:

```yaml
services:
  influxdb:
    image: influxdb:2.7
    container_name: influxdb
    ports: ["8086:8086"]
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME: admin
      DOCKER_INFLUXDB_INIT_PASSWORD: admin12345
      DOCKER_INFLUXDB_INIT_ORG: lecture
      DOCKER_INFLUXDB_INIT_BUCKET: factory
      DOCKER_INFLUXDB_INIT_ADMIN_TOKEN: my-super-secret-token
    volumes: ["influx-data:/var/lib/influxdb2"]

  grafana:
    image: grafana/grafana:11.0.0
    container_name: grafana
    ports: ["3000:3000"]
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    depends_on: [influxdb]
    volumes: ["grafana-data:/var/lib/grafana"]

volumes:
  influx-data:
  grafana-data:
```

In a terminal in that folder:

```powershell
docker compose up -d
```

InfluxDB is now on <http://localhost:8086>, Grafana on
<http://localhost:3000> (login `admin` / `admin`).

---

## Step 2 — Python script that generates data

Install the client once:

```powershell
pip install influxdb-client
```

Save as `producer.py`:

```python
import time, math, random
from datetime import datetime, timezone
from influxdb_client import InfluxDBClient, Point
from influxdb_client.client.write_api import SYNCHRONOUS

URL    = "http://localhost:8086"
TOKEN  = "my-super-secret-token"
ORG    = "lecture"
BUCKET = "factory"

client = InfluxDBClient(url=URL, token=TOKEN, org=ORG)
write  = client.write_api(write_options=SYNCHRONOUS)

print("Sending data to InfluxDB. Ctrl+C to stop.")
t = 0
parts_total = 0
states = ["Productive", "Standby", "Unscheduled Downtime"]

try:
    while True:
        # Fake a temperature curve with noise
        temperature = 60 + 5 * math.sin(t / 10) + random.uniform(-0.3, 0.3)

        # Fake an occasional finished part
        if random.random() < 0.4:
            parts_total += 1

        # Fake an e10-style state
        state = random.choices(states, weights=[0.85, 0.10, 0.05])[0]

        point = (
            Point("machine")
            .tag("line", "A")
            .field("temperature", temperature)
            .field("parts_total", parts_total)
            .field("state", state)
            .time(datetime.now(timezone.utc))
        )
        write.write(bucket=BUCKET, record=point)

        print(f"t={t:4d}  T={temperature:5.2f}  parts={parts_total}  state={state}")
        t += 1
        time.sleep(1)
except KeyboardInterrupt:
    print("Stopped.")
finally:
    client.close()
```

Run it:

```powershell
python producer.py
```

It prints a row per second and writes the same row to InfluxDB.

---

## Step 3 — Wire Grafana to InfluxDB (one-time)

1. Open <http://localhost:3000>, log in.
2. Left sidebar -> **Connections -> Data sources -> Add data source -> InfluxDB**.
3. Settings:
   - **Query language**: `Flux`
   - **URL**: `http://influxdb:8086` (the container name; from inside Docker network)
   - **Organization**: `lecture`
   - **Token**: `my-super-secret-token`
   - **Default Bucket**: `factory`
4. Click **Save & test** — should say "datasource is working".

---

## Step 4 — Build a panel

Left sidebar -> **Dashboards -> New -> Add visualization -> pick the InfluxDB datasource**.

Paste this Flux query for the temperature panel:

```flux
from(bucket: "factory")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "machine" and r._field == "temperature")
  |> filter(fn: (r) => r.line == "A")
```

Hit **Refresh** — the line chart should start moving as long as
`producer.py` is running. Set the panel auto-refresh (top right) to
`5s` for a live feel.

For the "parts produced" panel, change the field filter to
`parts_total` and pick the **Stat** visualization. For the e10 state
panel, filter on `state` and pick the **State timeline** visualization.

---

## How the pieces connect

```
producer.py (your laptop)
      |  HTTP POST /api/v2/write   (port 8086)
      v
   InfluxDB  (Docker container)
      |  Flux queries
      v
   Grafana   (Docker container, port 3000) --> your browser
```

Your script never talks to Grafana directly — it only writes to
InfluxDB. Grafana pulls from InfluxDB on every refresh. That decoupling
is exactly the pattern used in real factories.