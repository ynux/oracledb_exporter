# Oracle DB Exporter

[![Build Status](https://travis-ci.org/iamseth/oracledb_exporter.svg)](https://travis-ci.org/iamseth/oracledb_exporter)
[![GoDoc](https://godoc.org/github.com/iamseth/oracledb_exporter?status.svg)](http://godoc.org/github.com/iamseth/oracledb_exporter)
[![Report card](https://goreportcard.com/badge/github.com/iamseth/oracledb_exporter)](https://goreportcard.com/badge/github.com/iamseth/oracledb_exporter)

##### Table of Contents  

[Description](#description)  
[Installation](#installation)  
[Running](#running)  
[Grafana](#grafana)  
[Troubleshooting](#troubleshooting)  
[Operating principles](operating-principles.md)

# Description

A [Prometheus](https://prometheus.io/) exporter for Oracle modeled after the MySQL exporter. I'm not a DBA or seasoned Go developer so PRs definitely welcomed.

The following metrics are exposed currently.

- oracledb_exporter_last_scrape_duration_seconds
- oracledb_exporter_last_scrape_error
- oracledb_exporter_scrapes_total
- oracledb_up
- oracledb_activity_execute_count
- oracledb_activity_parse_count_total
- oracledb_activity_user_commits
- oracledb_activity_user_rollbacks
- oracledb_sessions_activity
- oracledb_wait_time_application
- oracledb_wait_time_commit
- oracledb_wait_time_concurrency
- oracledb_wait_time_configuration
- oracledb_wait_time_network
- oracledb_wait_time_other
- oracledb_wait_time_scheduler
- oracledb_wait_time_system_io
- oracledb_wait_time_user_io
- oracledb_tablespace_bytes
- oracledb_tablespace_max_bytes
- oracledb_tablespace_bytes_free
- oracledb_process_count
- oracledb_resource_current_utilization
- oracledb_resource_limit_value

# Installation

## Docker

You can run via Docker using an existing image. If you don't already have an Oracle server, you can run one locally in a container and then link the exporter to it.

```bash
docker run -d --name oracle -p 1521:1521 wnameless/oracle-xe-11g-r2:18.04-apex
docker run -d --name oracledb_exporter --link=oracle -p 9161:9161 -e DATA_SOURCE_NAME=system/oracle@oracle/xe iamseth/oracledb_exporter
```

Since 0.2.1, the exporter image exist with Alpine flavor. Watch out for their use. It is for the moment a test.

```bash
docker run -d --name oracledb_exporter --link=oracle -p 9161:9161 -e DATA_SOURCE_NAME=system/oracle@oracle/xe iamseth/oracledb_exporter:alpine
```

## Binary Release

Pre-compiled versions for Linux 64 bit and Mac OSX 64 bit can be found under [releases](https://github.com/iamseth/oracledb_exporter/releases).

In order to run, you'll need the [Oracle Instant Client Basic](http://www.oracle.com/technetwork/database/features/instant-client/index-097480.html)
for your operating system. Only the basic version is required for execution.

# Running

Ensure that the environment variable DATA_SOURCE_NAME is set correctly before starting. For Example:

```bash
# export Oracle location:
export DATA_SOURCE_NAME=system/password@oracle-sid
# or using a complete url:
export DATA_SOURCE_NAME=user/password@//myhost:1521/service
# Then run the exporter
/path/to/binary/oracledb_exporter --log.level error --web.listen-address 0.0.0.0:9161
```

# Integration with System D

Create file **/etc/systemd/system/oracledb_exporter.service** with the following content:

    [Unit]
    Description=Service for oracle telemetry client
    After=network.target
    [Service]
    Type=oneshot
    #User=oracledb_exporter
    ExecStart=/path/of/the/oracledb_exporter --log.level error --web.listen-address 0.0.0.0:9161
    [Install]
    WantedBy=multi-user.target

Then tell System D to read files:

    systemctl daemon-reload

Start this new service:

    systemctl start oracledb_exporter

Check service status:

    systemctl status oracledb_exporter

## Usage

```bash
Usage of oracledb_exporter:
  --log.format value
       	If set use a syslog logger or JSON logging. Example: logger:syslog?appname=bob&local=7 or logger:stdout?json=true. Defaults to stderr.
  --log.level value
       	Only log messages with the given severity or above. Valid levels: [debug, info, warn, error, fatal].
  --custom.metrics string
        File that may contain various custom metrics in a TOML file.
  --default.metrics string
        Default TOML file metrics.
  --web.listen-address string
       	Address to listen on for web interface and telemetry. (default ":9161")
  --web.telemetry-path string
       	Path under which to expose metrics. (default "/metrics")
  --database.maxIdleConns string
        Number of maximum idle connections in the connection pool. (default "0")
  --database.maxOpenConns string
        Number of maximum open connections in the connection pool. (default "10")
```

# Default metrics

This exporter comes with a set of default metrics defined in **default-metrics.toml**. You can modify this file or
provide a different one using ``default.metrics`` option.

# Custom metrics

This exporter does not have the metrics you want? You can provide new one using TOML file. To specify this file to the
exporter, you can:
- Use ``--custom.metrics`` flag followed by the TOML file
- Export CUSTOM_METRICS variable environment (``export CUSTOM_METRICS=my-custom-metrics.toml``)

This file must contain the following elements:
- One or several metric section (``[[metric]]``)
- For each section a context, a request and a map between a field of your request and a comment.

Here's a simple example:

```
[[metric]]
context = "test"
request = "SELECT 1 as value_1, 2 as value_2 FROM DUAL"
metricsdesc = { value_1 = "Simple example returning always 1.", value_2 = "Same but returning always 2." }
```

This file produce the following entries in the exporter:

```
# HELP oracledb_test_value_1 Simple example returning always 1.
# TYPE oracledb_test_value_1 gauge
oracledb_test_value_1 1
# HELP oracledb_test_value_2 Same but returning always 2.
# TYPE oracledb_test_value_2 gauge
oracledb_test_value_2 2
```

You can also provide labels using labels field. Here's an example providing two metrics, with and without labels:

```
[[metric]]
context = "context_no_label"
request = "SELECT 1 as value_1, 2 as value_2 FROM DUAL"
metricsdesc = { value_1 = "Simple example returning always 1.", value_2 = "Same but returning always 2." }

[[metric]]
context = "context_with_labels"
labels = [ "label_1", "label_2" ]
request = "SELECT 1 as value_1, 2 as value_2, 'First label' as label_1, 'Second label' as label_2 FROM DUAL"
metricsdesc = { value_1 = "Simple example returning always 1.", value_2 = "Same but returning always 2." }
```

This TOML file produce the following result:

```
# HELP oracledb_context_no_label_value_1 Simple example returning always 1.
# TYPE oracledb_context_no_label_value_1 gauge
oracledb_context_no_label_value_1 1
# HELP oracledb_context_no_label_value_2 Same but returning always 2.
# TYPE oracledb_context_no_label_value_2 gauge
oracledb_context_no_label_value_2 2
# HELP oracledb_context_with_labels_value_1 Simple example returning always 1.
# TYPE oracledb_context_with_labels_value_1 gauge
oracledb_context_with_labels_value_1{label_1="First label",label_2="Second label"} 1
# HELP oracledb_context_with_labels_value_2 Same but returning always 2.
# TYPE oracledb_context_with_labels_value_2 gauge
oracledb_context_with_labels_value_2{label_1="First label",label_2="Second label"} 2
```

Last, you can set metric type using **metricstype** field.

```
[[metric]]
context = "context_with_labels"
labels = [ "label_1", "label_2" ]
request = "SELECT 1 as value_1, 2 as value_2, 'First label' as label_1, 'Second label' as label_2 FROM DUAL"
metricsdesc = { value_1 = "Simple example returning always 1 as counter.", value_2 = "Same but returning always 2 as gauge." }
# Can be counter or gauge (default)
metricstype = { value_1 = "counter" }
```

This TOML file will produce the following result:

```
# HELP oracledb_test_value_1 Simple test example returning always 1 as counter.
# TYPE oracledb_test_value_1 counter
oracledb_test_value_1 1
# HELP oracledb_test_value_2 Same test but returning always 2 as gauge.
# TYPE oracledb_test_value_2 gauge
oracledb_test_value_2 2
```

You can find [here](./custom-metrics-example/custom-metrics.toml) a working example of custom metrics for slow queries, big queries and top 100 tables.

# Customize metrics in a docker image

If you run the exporter as a docker image and want to customize the metrics, you can use the following example:

```Dockerfile
FROM iamseth/oracledb_exporter:latest

COPY custom-metrics.toml /

ENTRYPOINT ["/oracledb_exporter"]
CMD ["--custom.metrics", "/custom-metrics.toml"]
```

# TLS connection to database

First, set the following variables:

    export WALLET_PATH=/wallet/path/to/use
    export TNS_ENTRY=tns_entry
    export DB_USERNAME=db_username
    export TNS_ADMIN=/tns/admin/path/to/use

Create the wallet and set the credential:

    mkstore -wrl $WALLET_PATH -create
    mkstore -wrl $WALLET_PATH -createCredential $TNS_ENTRY $DB_USERNAME

Then, update sqlnet.ora:

    echo "
    WALLET_LOCATION = (SOURCE = (METHOD = FILE) (METHOD_DATA = (DIRECTORY = $WALLET_PATH )))
    SQLNET.WALLET_OVERRIDE = TRUE
    SSL_CLIENT_AUTHENTICATION = FALSE
    " >> $TNS_ADMIN/sqlnet.ora

To use the wallet, use the wallet_location parameter. You may need to disable ssl verification with the
ssl_server_dn_match parameter.

Here a complete example of string connection:

    DATA_SOURCE_NAME=username/password@tcps://dbhost:port/service?ssl_server_dn_match=false&wallet_location=wallet_path

For more details, have a look at the following location: https://github.com/iamseth/oracledb_exporter/issues/84

# Integration with Grafana

An example Grafana dashboard is available [here](https://grafana.com/dashboards/3333).

# Build

## Docker build

To build Ubuntu and Alpine image, run the following command:

    make docker

You can also build only Ubuntu image:

    make ubuntu-image

Or Alpine:

    make alpine-image

## Linux binaries

Retrieve Oracle RPMs (version 18.5):

    make download-rpms

Then run build:

    make linux

## Windows binaries

*Stollen from https://github.com/iamseth/oracledb_exporter/issues/40*

First, download Oracle Instant Client 64-Bit version basic and sdk versions.

Extract client (for example: **C:\oracle\instantclient_18_5**) and extract SDK to the same folder (**C:\oracle\instantclient_18_5\sdk**)

Set the environment variables:

    setx CGO_CFLAGS "C:\oracle\instantclient_18_5\sdk\include"
    setx CGO_LDFLAGS "-LC:\oracle\instantclient_18_5 -loci"

Then install GCC (like MSYS2 64 bit in **c:\msys64**)

Run the MSYS2 MINGW64 terminal and set dependencies packages:

- Update pacman:

    pacman -Su

- Close terminal and open a new terminal
- Update all other packages:

    pacman -Su

- Install pkg-config and gcc:

    pacman -S mingw64/mingw-w64-x86_64-pkg-config mingw64/mingw-w64-x86_64-gcc

Go to the pkg-config dir **c:/msys64/mingw64/lib/pkgconfig/** and create **oci8.pc** with the following content:

    prefix=C:\oracle\instantclient_18_5/sdk/
    version=18.5
    build=client64
    libdir=C:\oracle\instantclient_18_5/sdk/lib/msvc
    includedir=C:\oracle\instantclient_18_5/sdk/include
    glib_genmarshal=glib-genmarshal
    gobject_query=gobject-query
    glib_mkenums=glib-mkenums
    Name: oci8
    Description: Oracle database engine
    Version: ${version}
    Libs: -L${libdir} -loci
    Libs.private:
    Cflags: -I${includedir}

Set **%PKG_CONFIG_PATH%** as the environment variable:

    setx PKG_CONFIG_PATH "C:\msys64\mingw64\lib\pkgconfig"

Ensure, that **%PATH%** includes path to the msys64 binares, if not set it: setx path "%path%;C:\msys64\mingw64\bin"

Everything must compile, including mattn driver for oracle.

Next build ./... in oracledb-exporter dir, or install it.

# FAQ/Troubleshooting

## Unable to convert current value to float (metric=par,metri...in.go:285

Oracle is trying to send a value that we cannot convert to float. This could be anything like 'UNLIMITED' or 'UNDEFINED' or 'WHATEVER'.

In this case, you must handle this problem by testing it in the SQL request. Here an example available in default metrics:

```toml
[[metric]]
context = "resource"
labels = [ "resource_name" ]
metricsdesc = { current_utilization= "Generic counter metric from v$resource_limit view in Oracle (current value).", limit_value="Generic counter metric from v$resource_limit view in Oracle (UNLIMITED: -1)." }
request="SELECT resource_name,current_utilization,CASE WHEN TRIM(limit_value) LIKE 'UNLIMITED' THEN '-1' ELSE TRIM(limit_value) END as limit_value FROM v$resource_limit"
```

If the value of limite_value is 'UNLIMITED', the request send back the value -1.

You can increase the log level (`--log.level debug`) in order to get the statement generating this error.

## error while loading shared libraries: libclntsh.so.xx.x: cannot open shared object file: No such file or directory

This exporter use libs from Oracle in order to connect to Oracle Database. If you are running the binary version, you
must install the Oracle binaries somewhere on your machine and **you must install the good version number**. If the
error talk about the version 18.3, you **must** install 18.3 binary version. If it's 12.2, you **must** install 12.2.

An alternative is to run this exporter using a Docker container. This way, you don't have to worry about Oracle binaries
version as they are embedded in the container.

Here an example to run this exporter (to scrap metrics from system/oracle@//host:1521/service-or-sid) and bind the exporter port (9161) to the global machine:

```docker run -it --rm -p 9161:9161 -e DATA_SOURCE_NAME=system/oracle@//host:1521/service-or-sid iamseth/oracledb_exporter:0.2.6a```


## Error scraping for wait_time

If you experience an error `Error scraping for wait_time: sql: Scan error on column index 1: converting driver.Value type string (",01") to a float64: invalid syntax source="main.go:144"` you may need to set the NLS_LANG variable.

```bash

export NLS_LANG=AMERICAN_AMERICA.WE8ISO8859P1
export DATA_SOURCE_NAME=system/oracle@myhost
/path/to/binary --log.level error --web.listen-address 9161
```

If using Docker, set the same variable using the -e flag.
