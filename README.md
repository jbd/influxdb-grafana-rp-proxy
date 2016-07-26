# InfluxDB Grafana Retention Policy Proxy

Proxy inserted between Grafana and InfluxDB which auto-selects an
InfuxDB retention policy based on the size of the `GROUP BY` interval
specified in queries from Grafana.

Until InfluxDB either

a) intelligently merges data points from different retention policies
at query time (perhaps through layers for RPs?)

b) implements downsampling internally

a proxy such as this one will be required to effectively use Grafana
 with an InfluxDB containing downsampled data.  Watch the
 [GitHub issue](https://github.com/influxdata/influxdb/issues/5750)
 for progress.

This proxy

- assumes series names and tags are identical across retention
  policies.
- assumes database and retention policies do not have periods ('.') in
  their names (it's OK if series do).

Original code: @PaulKuiper https://github.com/influxdata/influxdb/issues/2625#issuecomment-161716174

## Installation

Ubuntu 14.04 setup:

```
$ sudo apt-get install python-regex python-requests mitmproxy
```

Or use [`pip`](https://docs.python.org/3.6/installing/index.html) one
you've cloned this repo:

```
$ sudo pip install -r requirements.txt
```

## Configuration

The [default configuration file](default.yml) is simple and
documented.  Modify it as you need.

## Usage

This proxy is intended to be run as a script for the `mitmdump` tool.

```
$ mitmdump --reverse "http:/localhost:8086" --port 3004 --script 'proxy.py default.yml'
```

Now configure Grafana to use an "InfluxDB server" at `localhost:3004`,
instead of the actual InfluxDB server (`localhost:8086` in this
example).

## Database Preparation

The following is just an example, written to match the
[default configuration](default.yml).  It works well with data being
sourced from collection systems at a raw frequency of 5-10 seconds and
being viewed from Grafana.

1) Create database with retention policies:

```
CREATE DATABASE IF NOT EXISTS operations WITH DURATION 24h REPLICATION 1 NAME for_1d_raw
CREATE RETENTION POLICY for_7d_at_1m ON operations DURATION 7d REPLICATION 1
CREATE RETENTION POLICY for_90d_at_10m ON operations DURATION 90d REPLICATION 1
CREATE RETENTION POLICY forever_at_1h ON operations DURATION INF REPLICATION 1
```

2) Create Continuous Queries

```
CREATE CONTINUOUS QUERY "for_1d_raw->for_7d_at_1m" ON operations RESAMPLE EVERY 30s BEGIN SELECT mean(value) AS value INTO operations."for_7d_at_1m".:MEASUREMENT FROM operations."for_1d_raw"./.*/ GROUP BY time(1m), * END
CREATE CONTINUOUS QUERY "for_7d_at_1m->for_90d_at_10m" ON operations RESAMPLE EVERY 5m BEGIN SELECT mean(value) AS value INTO operations."for_90d_at_10m".:MEASUREMENT FROM operations."for_7d_at_1m"./.*/ GROUP BY time(10m), * END
CREATE CONTINUOUS QUERY "for_90d_at_10m->forever_at_1h" ON operations RESAMPLE EVERY 30m BEGIN SELECT mean(value) AS value INTO operations."forever_at_1h".:MEASUREMENT FROM operations."for_90d_at_10m"./.*/ GROUP BY time(1h), * END
```

The `, *` and the end of the `GROUP BY` clause ensures that InfluxDB
uses the full series name (including tags) for grouping.

3) Backfill historical data 90 days (only if needed).

```
SELECT mean(value) as value INTO operations."for_7d_at_1m".:MEASUREMENT  FROM operations."for_1d_raw"./.*/  WHERE time > now() - 90d GROUP BY time(1m), *
SELECT mean(value) as value INTO operations."for_90d_at_10m".:MEASUREMENT FROM operations."for_7d_at_1m"./.*/ WHERE time > now() - 90d GROUP BY time(10m), *
SELECT mean(value) as value INTO operations."forever_at_1h".:MEASUREMENT FROM operations."for_90d_at_10m"./.*/ WHERE time > now() - 90d GROUP BY time(1h), *
```
