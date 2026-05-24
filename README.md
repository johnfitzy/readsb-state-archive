# Readsb State Archive

## ADS-B Hobbyist Aircraft State Archive using [Bento](https://warpstreamlabs.github.io/bento/) and [DuckDB](https://duckdb.org/)

Collects live ADS-B aircraft state from an ultrafeeder (or any readsb-based feeder), batches it into time windows, and writes compressed Parquet files partitioned by hour. Query the results locally with DuckDB - no cloud, no cluster.

```
ultrafeeder :30047 (JSON stream)
    └── Bento (batch → Parquet encode)
            └── /data/bento/aircraft/hour=.../part-N.parquet
                    └── DuckDB view
```

## Prerequisites

- Docker
- An ultrafeeder (or readsb) instance with the JSON stream accessible on `localhost:30047`

Verify the stream is live before proceeding:
```bash
nc localhost 30047 | jq --unbuffered -c '.'
```

## Setup

### 1. Create the output directory

Bento runs as UID 10001 inside the container:

```bash
sudo mkdir -p /data/bento/aircraft
sudo chown 10001:10001 /data/bento/aircraft
```

### 2. Run Bento

```bash
docker run --net=host --rm \
  --name readsb-state-archive \
  -v $(pwd)/bento-config.yaml:/bento.yaml \
  -v /data/bento/aircraft:/data/bento/aircraft \
  ghcr.io/warpstreamlabs/bento:1.17.0
```

Bento will start collecting immediately. By default it uses a **15-minute tumbling window**, your first Parquet file will appear after the first window closes (15min intervals past the hour). This is configured with the `buffer.system_window.size` option in `bento-config.yaml`. 

Normally, in the "real" world, Parquet file size should be kept between 256MB to 1GB for best performance. With your own ADS-B data feed you won't get any where near this size for each hour partition. To increase the file size increase the tumbling window up to `1h`. But make sure you are not going to blow the Pi's memory out, eg: `docker stats` to see RAM usage of the container. 

To run in the background:
```bash
docker run --net=host -d --restart unless-stopped \
  --name readsb-state-archive \
  -v $(pwd)/bento-config.yaml:/bento.yaml \
  -v /data/bento/aircraft:/data/bento/aircraft \
  ghcr.io/warpstreamlabs/bento:1.17.0
```

### 3. Verify output

After 15 minutes, check for Parquet files:
```bash
ls /data/bento/aircraft/
```

You should see directories like `hour=1748000000/` containing `part-1.parquet`.

## Configuration

See [`bento-config.yaml`](bento-config.yaml) for the full pipeline config.

Key settings:

| Setting | Default | Description |
|---------|---------|-------------|
| `buffer.system_window.size` | `15m` | How long to accumulate records before writing a file. Larger = fewer bigger files, more data at risk if the container crashes. |
| `input.socket.address` | `localhost:30047` | readsb JSON output port |
| `output.file.path` | `/data/bento/aircraft/...` | Output location, partitioned by hour |

**Memory:** Bento holds up to two windows of raw messages in memory. To view how much memory the container is using run `docker stats`.

## Querying with DuckDB

### Install

**x86-64 (amd64):**
```bash
wget https://github.com/duckdb/duckdb/releases/latest/download/duckdb_cli-linux-amd64.zip
unzip duckdb_cli-linux-amd64.zip
sudo mv duckdb /usr/local/bin/
```

**Raspberry Pi (aarch64):**
```bash
wget https://github.com/duckdb/duckdb/releases/latest/download/duckdb_cli-linux-aarch64.zip
unzip duckdb_cli-linux-aarch64.zip
sudo mv duckdb /usr/local/bin/
```

### Create a persistent view

Create a DuckDB database file once, it stores only the view definition, not the data:

```bash
duckdb ~/aircraft.duckdb
```

```sql
CREATE VIEW aircraft AS
SELECT * FROM read_parquet('/data/bento/aircraft/hour=*/part-*.parquet', hive_partitioning=true);
```

Reconnecting later with `duckdb ~/aircraft.duckdb` will have the view ready to go.

### Example queries

```sql
-- How many distinct aircraft seen?
SELECT COUNT(DISTINCT hex) FROM aircraft;

-- All flights in the last hour
SELECT DISTINCT hex, flight, alt_baro_ft, gs
FROM aircraft
WHERE time > epoch(now()) - 3600
ORDER BY flight;

-- Highest aircraft seen
SELECT hex, flight, MAX(alt_baro_ft) AS max_alt
FROM aircraft
GROUP BY hex, flight
ORDER BY max_alt DESC
LIMIT 20;

-- Aircraft count by hour
SELECT hour, COUNT(DISTINCT hex) AS aircraft
FROM aircraft
GROUP BY hour
ORDER BY hour DESC;
```

### Disk use

Be sure to rotate the files (or mount the directory to your NAS) because eventually your Pi's disk will fill up! 