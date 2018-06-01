---
title: Logging and tracing in InfluxDB

menu:
  influxdb_1_6:
    name: Logging and tracing
    weight: 40
    parent: Administration
---

**Content**

* [Logging locations](#logging-locations)
* [Redirecting HTTP request logging](#redirecting-http-request-logging)
* [Structured logging](#structured-logging)
* [Tracing](#tracing)


## Logging locations

InfluxDB writes log output, by default, to `stderr`.
Depending on your use case, this log information can be written to another location.

### Running InfluxDB directly

When you run InfluxDB directly, using `influxd`, all logs are written to `stderr`.
You can also redirect the log output, as you would any output to `stderr`, like in this example:

```
influxd 2>$HOME/my_log_file
```

### Launched as a service

#### sysvinit

If InfluxDB was installed using a prebuilt package, and then launched
as a service, `stderr` is redirected to `/var/log/influxdb/influxd.log` and all log data will be written to that file.
You can override this location by setting the environment variable `STDERR` in the InfluxDB configuration file `/etc/default/influxdb`.

>**Note:** Mac OS X logs are stored, by default, in the file `/usr/local/var/log/influxdb.log`.

For example, if `/etc/default/influxdb` contains:

```
STDERR=/dev/null
```

all log data will be discarded.  Likewise, you can direct output to
`stdout` by setting `STDOUT` in the same file.  Output to `stdout` is
sent to `/dev/null` by default when InfluxDB is launched as a service.

InfluxDB must be restarted to use any changes to `/etc/default/influxdb`.

#### systemd

Starting with version 1.0, InfluxDB on `systemd` systems will no longer
write files to `/var/log/influxdb` by default, and will now use the
system configured default for logging (usually `journald`).  On most
Linux systems, the logs will be directed to the `systemd` journal and can be accessed with the command:

```
sudo journalctl -u influxdb.service
```

See the `systemd` `journald` documentation about configuring `journald`.

### Using logrotate

You can use [logrotate](http://manpages.ubuntu.com/manpages/hardy/man8/logrotate.8.html) to rotate the log files generated by InfluxDB on systems where logs are written to flat files.
If using the package install on a `sysvinit` system, the config file for logrotate is installed in `/etc/logrotate.d`.
You can view the file [here](https://github.com/influxdb/influxdb/blob/master/scripts/logrotate).


## Redirecting HTTP request logging

InfluxDB 1.5 introduces the option to log HTTP request traffic separately from the other InfluxDB log output. When HTTP request logging is enabled, the HTTP logs are intermingled by default with internal InfluxDB logging. By redirecting the HTTP request log entries to a separate file, both log files are easier to read, monitor, and debug.

**To redirect HTTP request logging:**

Locate the `[http]` section of your InfluxDB configuration file and set the `access-log-path` option to specify the path where HTTP log entries should be written.

**Notes:**

* If `influxd` is unable to access the specified path, it will log an error and fall back to writing the request log to `stderr`.
* The `[httpd]` prefix is stripped when HTTP request logging is redirected to a separate file, allowing access log parsing tools (like [lnav](https://lnav.org)) to render the files without additional modification.
* To rotate the HTTP request log file, use the `copyrotate` method of `logrotate` or similar to leave the original file in place.


## Structured logging

With InfluxDB 1.5, structured logging is supported and enable machine-readable and more developer-friendly log output formats. The two new structured log formats, `logfmt` and `json`, provide easier filtering and searching with external tools and simplifies integration of InfluxDB logs  with Splunk, Papertrail, Elasticsearch, and other third party tools.

The InfluxDB logging configuration options (in the `[logging]` section) now include the following options:

* `format`: `auto` (default) | `logfmt` | `json`
* `level`: `error` | `warn` | `info` (default) | `debug`
* `suppress-logo`: `false` (default) | `true`

For details on these logging configuration options and their corresponding environment variables, see [Logging options](/influxdb/v1.6/administration/config/#logging-settings-logging) in the configuration file documentation.

### Logging formats

Three logging `format` options are available: `auto`, `logfmt`, and `json`. The default logging format setting, `format = "auto"`, lets InfluxDB automatically manage the log encoding format:

* When logging to a file, the `logfmt` is used
* When logging to a terminal (or other TTY device), a user-friendly console format is used.

The `json` format is available when specified.

### Examples of log output:

**Logfmt**

```
ts=2018-02-20T22:48:11.291815Z lvl=info msg="InfluxDB starting" version=unknown branch=unknown commit=unknown
ts=2018-02-20T22:48:11.291858Z lvl=info msg="Go runtime" version=go1.10 maxprocs=8
ts=2018-02-20T22:48:11.291875Z lvl=info msg="Loading configuration file" path=/Users/user_name/.influxdb/influxdb.conf
```

**JSON**

```
{"lvl":"info","ts":"2018-02-20T22:46:35Z","msg":"InfluxDB starting, version unknown, branch unknown, commit unknown"}
{"lvl":"info","ts":"2018-02-20T22:46:35Z","msg":"Go version go1.10, GOMAXPROCS set to 8"}
{"lvl":"info","ts":"2018-02-20T22:46:35Z","msg":"Using configuration at: /Users/user_name/.influxdb/influxdb.conf"}
```

**Console/TTY**

```
2018-02-20T22:55:34.246997Z     info    InfluxDB starting       {"version": "unknown", "branch": "unknown", "commit": "unknown"}
2018-02-20T22:55:34.247042Z     info    Go runtime      {"version": "go1.10", "maxprocs": 8}
2018-02-20T22:55:34.247059Z     info    Loading configuration file      {"path": "/Users/user_name/.influxdb/influxdb.conf"}
```

### Logging levels

The `level` option sets the log level to be emitted. Valid logging level settings are `error`, `warn`, `info`(default), and `debug`. Logs that are equal to, or above, the specified level will be emitted.

### Logo suppression

The `suppress-logo` option can be used to suppress the logo output that is printed when the program is started. The logo is always suppressed if `STDOUT` is not a TTY.

## Tracing

Logging has been enhanced to provide tracing of important InfluxDB operations. Tracing is useful for error reporting and discovering performance bottlenecks.

### Logging keys used in tracing

#### Tracing identifier key

The `trace_id` key specifies a unique identifier for a specific instance of a trace. You can use this key to filter and correlate all related log entries for an operation.

All operation traces include consistent starting and ending log entries, with the same message (`msg`) describing the operation (e.g., "TSM compaction"), but adding the appropriate `op_event` context (either `start` or `end`). For an example, see [Finding all trace log entries for an InfluxDB operation](#finding-all-trace-log-entries-for-an-influxdb-operation).

**Example:** `trace_id=06R0P94G000`

#### Operation keys

The following operation keys identify an operation's name, the start and end timestamps, and the elapsed execution time.

##### `op_name`
Unique identifier for an operation. You can filter on all operations of a specific name.

**Example:** `op_name=tsm1_compact_group`

##### `op_event`
Specifies the start and end of an event. The two possible values, `(start)` or `(end)`, are used to indicate when an operation started or ended. For example, you can grep by values in `op_name` AND `op_event` to find all starting operation log entries. For an example of this, see [Finding all starting log entries](#finding-all-starting-operation-log-entries).

**Example:** `op_event=start`

##### `op_elapsed`
Amount of time the operation spent executing. Logged with the ending trace log entry. Time unit displayed depends on how much time has elapsed -- if it was seconds, it will be suffixed with `s`. Valid time units are `ns`, `µs`, `ms`, and `s`.

**Example:** `op_elapsed=0.352ms`


#### Log identifier context key

The log identifier key (`log_id`) lets you easily identify _every_ log entry for a single execution of an `influxd` process. There are other ways a log file could be split by a single execution, but the consistent `log_id` eases the searching of log aggregation services.

**Example:** `log_id=06QknqtW000`

#### Database context keys

`db_instance`: Database name

`db_rp`: Retention policy name

`db_shard_id`: Shard identifier

`db_shard_group` Shard group identifier

### Tooling

Here are a couple of popular tools available for processing and filtering log files output in `logfmt` or `json` formats.

#### [hutils](https://blog.heroku.com/hutils-explore-your-structured-data-logs)

The [hutils](https://blog.heroku.com/hutils-explore-your-structured-data-logs), provided by Heroku, is a collection of command line utilities for working with logs with `logfmt` encoding, including:

* `lcut`: Extracts values from a `logfmt` trace based on a specified field name.
* `lfmt`: Prettifies `logfmt` lines as they emerge from a stream, and highlights their key sections.
* `ltap`: Accesses messages from log providers in a consistent way to allow easy parsing by other utilities that operate on `logfmt` traces.
* `lviz`: Visualizes `logfmt` output by building a tree out of a dataset combining common sets of key-value pairs into shared parent nodes.

#### [lnav (Log File Navigator)](http://lnav.org)

The [lnav (Log File Navigator)](http://lnav.org) is an advanced log file viewer useful for watching and analyzing your log files from a terminal. The lnav viewer provides a single log view, automatic log format detection, filtering, timeline view, pretty-print view, and querying logs using SQL.

### Operations

The following operations, listed by their operation name (`op_name`) are traced in InfluxDB internal logs and available for use without changes in logging level.

#### Initial opening of data files

The `tsdb_open` operation traces include all events related to the initial opening of the `tsdb_store`.


#### Retention policy shard deletions

The `retention.delete_check` operation includes all shard deletions related to the retention policy.

#### TSM snapshotting in-memory cache to disk

The `tsm1_cache_snapshot` operation represents the snapshotting of the TSM in-memory cache to disk.

#### TSM compaction strategies

The `tsm1_compact_group` operation includes all trace log entries related to TSM compaction strategies and displays the related TSM compaction strategy keys:

* `tsm1_strategy`: `level` | `full`
* `tsm1_level`: `1` | `2` | `3`
* `tsm1_optimize`: `true` | `false`

#### Series file compactions

The `series_partition_compaction` operation includes all trace log entries related to series file compactions.

####  Continuous query execution (if logging enabled)

The `continuous_querier_execute` operation includes all continuous query executions, if logging is enabled.

#### TSI log file compaction

The `tsi1_compact_log_file`

#### TSI level compaction

The `tsi1_compact_to_level` operation includes all trace log entries for TSI level compactions.


### Tracing examples

#### Finding all trace log entries for an InfluxDB operation

In the example below, you can see the log entries for all trace operations related to a "TSM compaction" process. Note that the initial entry shows the message "TSM compaction (start)" and the final entry displays the message "TSM compaction (end)". \[Note: Log entries were grepped using the `trace_id` value and then the specified key values were displayed using `lcut` (an hutils tool).\]

```
$ grep "06QW92x0000" influxd.log | lcut ts lvl msg strategy level
2018-02-21T20:18:56.880065Z	info	TSM compaction (start)	full
2018-02-21T20:18:56.880162Z	info	Beginning compaction	full
2018-02-21T20:18:56.880185Z	info	Compacting file	full
2018-02-21T20:18:56.880211Z	info	Compacting file	full
2018-02-21T20:18:56.880226Z	info	Compacting file	full
2018-02-21T20:18:56.880254Z	info	Compacting file	full
2018-02-21T20:19:03.928640Z	info	Compacted file	full
2018-02-21T20:19:03.928687Z	info	Finished compacting files	full
2018-02-21T20:19:03.928707Z	info	TSM compaction (end)	full
```


#### Finding all starting operation log entries

To find all starting operation log entries, you can grep by values in `op_name` AND `op_event`. In the following example, the grep returned 101 entries, so the result below only displays the first entry. In the example result entry, the timestamp, level, strategy, trace_id, op_name, and op_event values are included.

```
$ grep -F 'op_name=tsm1_compact_group' influxd.log | grep -F 'op_event=start'
ts=2018-02-21T20:16:16.709953Z lvl=info msg="TSM compaction" log_id=06QVNNCG000 engine=tsm1 level=1 strategy=level trace_id=06QV~HHG000 op_name=tsm1_compact_group op_event=start
...
```

Using the `lcut` utility (in hutils), the following command uses the previous `grep` command, but adds an `lcut` command to only display the keys and their values for keys that are not identical in all of the entries. The following example includes 19 examples of unique log entries displaying selected keys: `ts`, `strategy`, `level`, and `trace_id`.

```
$ grep -F 'op_name=tsm1_compact_group' influxd.log | grep -F 'op_event=start' | lcut ts strategy level trace_id | sort -u
2018-02-21T20:16:16.709953Z	level	1	06QV~HHG000
2018-02-21T20:16:40.707452Z	level	1	06QW0k0l000
2018-02-21T20:17:04.711519Z	level	1	06QW2Cml000
2018-02-21T20:17:05.708227Z	level	2	06QW2Gg0000
2018-02-21T20:17:29.707245Z	level	1	06QW3jQl000
2018-02-21T20:17:53.711948Z	level	1	06QW5CBl000
2018-02-21T20:18:17.711688Z	level	1	06QW6ewl000
2018-02-21T20:18:56.880065Z	full		06QW92x0000
2018-02-21T20:20:46.202368Z	level	3	06QWFizW000
2018-02-21T20:21:25.292557Z	level	1	06QWI6g0000
2018-02-21T20:21:49.294272Z	level	1	06QWJ_RW000
2018-02-21T20:22:13.292489Z	level	1	06QWL2B0000
2018-02-21T20:22:37.292431Z	level	1	06QWMVw0000
2018-02-21T20:22:38.293320Z	level	2	06QWMZqG000
2018-02-21T20:23:01.293690Z	level	1	06QWNygG000
2018-02-21T20:23:25.292956Z	level	1	06QWPRR0000
2018-02-21T20:24:33.291664Z	full		06QWTa2l000
2018-02-21T21:12:08.017055Z	full		06QZBpKG000
2018-02-21T21:12:08.478200Z	full		06QZBr7W000
```