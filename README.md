# InfluxDB Proxy

This project adds a basic high availability and consistent hash layer to InfluxDB v2.

NOTE: influx-proxy must be built with Go 1.14+ with Go module support, don't implement udp.

NOTE: [InfluxDB Cluster](https://github.com/chengshiwen/influxdb-cluster) for replacing [InfluxDB Enterprise](https://docs.influxdata.com/enterprise_influxdb/v1.8/) is coming, which is better than InfluxDB Proxy.

## Why

We used [InfluxDB Relay](https://github.com/influxdata/influxdb-relay) before, but it doesn't support some demands.
We use grafana for visualizing time series data, so we need add datasource for grafana. We need change the datasource config when influxdb is down.
We need transfer data across idc, but Relay doesn't support gzip.
It's inconvenient to analyse data with connecting different influxdb.
Therefore, we made [InfluxDB Proxy](https://github.com/shell909090/influx-proxy). More details please visit [https://github.com/shell909090/influx-proxy](https://github.com/shell909090/influx-proxy).

Forked from the above InfluxDB Proxy, after many improvements and optimizations, [InfluxDB Proxy v1](https://github.com/chengshiwen/influx-proxy/tree/branch-1.x) has released, which no longer depends on python and redis, and supports more features.

Since the InfluxDB Proxy v1 is limited by the only `ONE` database and the `KEYMAPS` configuration, we refactored [InfluxDB Proxy v2](https://github.com/chengshiwen/influx-proxy) with high availability and consistent hash, which supports multiple databases and tools to rebalance, recovery, resync and cleanup.

[InfluxDB Proxy v3](https://github.com/chengshiwen/influx-proxy/tree/influxdb-v2) is aimed at [InfluxDB v2](https://docs.influxdata.com/influxdb/v2.1/).

## Features

* Support query and write.
* Transparent for client, like cluster for client.
* Cache data to file when write failed, then rewrite.
* Support data sharding with consistent hash.
* Load config file and no longer depend on python and redis.
* Support influxdb-java, and influxdb shell.
* Support authentication and https.
* Support health status query.
* Support version display.
* Support gzip.

## Requirements

* Golang >= 1.14 with Go module support
* InfluxDB >= 2.0 (For 1.x, please visit branch [master](https://github.com/chengshiwen/influx-proxy/tree/master))

## Usage

#### Quickstart

```sh
$ git clone https://github.com/chengshiwen/influx-proxy.git
$ cd influx-proxy
$ make
$ ./bin/influx-proxy -config proxy.json
```

#### Usage

```sh
$ ./bin/influx-proxy -h
Usage of ./bin/influx-proxy:
  -config string
        proxy config file with json/yaml/toml format (default "proxy.json")
  -version
        proxy version
```

#### Build Release

```sh
$ # build current platform
$ make build
$ # build linux amd64
$ make linux
```

## Tutorial

[Chinese](https://git.io/influx-proxy-wiki)

## Description

The architecture is fairly simple, one InfluxDB Proxy instance and two consistent hash circles with two InfluxDB instances respectively.
The Proxy should point HTTP requests with organization, bucket and measurement to the two circles and the four InfluxDB servers.

The setup should look like this:

```
        ┌──────────────────┐
        │ writes & queries │
        └──────────────────┘
                 │
                 ▼
        ┌──────────────────┐
        │                  │
        │  InfluxDB Proxy  │
        │   (only http)    │
        │                  │
        └──────────────────┘
                 │
                 ▼
        ┌──────────────────┐
        │    org,bucket    │
        │   & measurement  │
        │ consistent hash  │
        └──────────────────┘
          |              |
        ┌─┼──────────────┘
        │ └────────────────┐
        ▼                  ▼
     Circle 1          Circle 2
  ┌────────────┐    ┌────────────┐
  │            │    │            │
  │ InfluxDB 1 │    │ InfluxDB 3 │
  │ InfluxDB 2 │    │ InfluxDB 4 │
  │            │    │            │
  └────────────┘    └────────────┘
```

## Proxy Configuration

The configuration file supports format `json`, `yaml` and `toml`, such as [proxy.json](proxy.json), [proxy.yaml](conf/proxy.yaml) and [proxy.toml](conf/proxy.toml).

The configuration settings are as follows:

* `circles`: circle list
  * `name`: circle name, `required`
  * `backends`: backend list belong to the circle, `required`
    * `name`: backend name, `required`
    * `url`: influxdb addr or other http backend which supports influxdb line protocol, `required`
    * `token`: influxdb token, `required`
    * `write_only`: whether to write only on the influxdb, default is `false`
* `dbrp`: dbrp mapping config for 1.x compatibility
  * `separator`: the separator of the key-value pair mapping, for 1.x compatibility, default is `/`
  * `mapping`: the key-value pair mapping from `db/rp` to `org/bucket`, for 1.x compatibility, default is `nil`
* `listen_addr`: proxy listen addr, default is `:7076`
* `data_dir`: data dir to save .dat .rec, default is `data`
* `flush_size`: default is `10000`, wait 10000 points write
* `flush_time`: default is `1`, wait 1 second write whether point count has bigger than flush_size config
* `check_interval`: default is `1`, check backend active every 1 second
* `rewrite_interval`: default is `10`, rewrite every 10 seconds
* `conn_pool_size`: default is `20`, create a connection pool which size is 20
* `write_timeout`: default is `10`, write timeout until 10 seconds
* `write_tracing`: enable logging for the write, default is `false`
* `query_tracing`: enable logging for the query, default is `false`
* `token`: proxy token, default is `empty` which means no auth
* `https_enabled`: enable https, default is `false`
* `https_cert`: the ssl certificate to use when https is enabled, default is `empty`
* `https_key`: use a separate private key location, default is `empty`

## Write

* [/api/v2/write](https://docs.influxdata.com/influxdb/v2.1/api/#operation/PostWrite) v2 supported
* [/write](https://docs.influxdata.com/influxdb/v2.1/api/v1-compatibility/#operation/PostWriteV1) v1 compatibility supported

## Query

* [/api/v2/query](https://docs.influxdata.com/influxdb/v2.1/api/#operation/PostQuery) v2 supported
* [/query](https://docs.influxdata.com/influxdb/v2.1/api/v1-compatibility/#operation/PostQueryV1) v1 compatibility supported

### /api/v2/query

Note: `_measurement` must be specified

```
from(bucket: "example-bucket")
    |> range(start: -1h)
    |> filter(fn: (r) => r._measurement == "example-measurement" and r.tag == "example-tag")
    |> filter(fn: (r) => r._field == "example-field")
```

### /query

Note: `dbrp mapping` must be specified like

```
"dbrp": {
    "separator": "/",
    "mapping": {
        "mydb/myrp": "myorg/mybucket"
    }
}
```

Only support match the following commands, more details please see [InfluxQL support](https://docs.influxdata.com/influxdb/v2.1/query-data/influxql/#influxql-support).

* `select from` (read-only)
* `show databases`
* `show measurements`
* `show tag keys`
* `show tag values`
* `show field keys`
* `delete from`
* `drop measurement`
* `on clause` (the `db` parameter takes precedence when the parameter is set in `/query` http endpoint)

## HTTP Endpoints

[HTTP Endpoints](https://github.com/chengshiwen/influx-proxy/wiki/HTTP-Endpoints)

## Benchmark

There are three tools for benchmarking InfluxDB, which can also be applied to InfluxDB Proxy:

* [influx-stress](https://github.com/chengshiwen/influx-stress) is a stress tool for generating artificial load on InfluxDB.
* [influxdb-comparisons](https://github.com/influxdata/influxdb-comparisons) contains code for benchmarking InfluxDB against other databases and time series solutions.
* [tsbs](https://github.com/timescale/tsbs) (Time Series Benchmark Suite) is a tool for comparing and evaluating databases for time series data.

## Tool

There is a tool for InfluxDB and InfluxDB Proxy:

* [influx-tool](https://github.com/chengshiwen/influx-tool): high performance tool to rebalance, recovery, resync, cleanup and compact. most commands do not require InfluxDB to start

## License

MIT.
