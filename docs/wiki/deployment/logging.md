The osquery daemon uses a default **filesystem** logging plugin. Like the config, output from the filesystem plugin is written as JSON. Results from the query schedule are written to **/var/log/osquery/osqueryd.results.log**.

There are two types of logs:

- Status logs (info, warning, error, and fatal)
- Query schedule results logs

If you run osqueryd in a verbose mode then peek at **/var/log/osquery/**:

```
$ ls -l /var/log/osquery/
total 24
lrwxr-xr-x   1 root  wheel    77 Sep 30 17:37 osqueryd.INFO -> osqueryd.INFO.20140930
-rw-------   1 root  wheel  1226 Sep 30 17:37 osqueryd.INFO.20140930
-rw-------   1 root  wheel   388 Sep 30 17:37 osqueryd.results.log
```

### Status logs

Status logs are generated by the [glog logging framework](https://code.google.com/p/google-glog/). The default **filesystem** logger plugin writes these logs to disk the same way glog would. Logging plugins may intercept these status logs and write them to system or otherwise.

As the above directory listing reveals,
*osqueryd.INFO* is a symlink to the most recent execution's INFO log.
The same is true for the WARNING, ERROR and FATAL logs. For more information on the format of glog logs, please refer to the [glog documentation](http://google-glog.googlecode.com/svn/trunk/doc/glog.html).

### Results logs

The results of your scheduled queries are logged to the "results log".
Each log line is a JSON string that indicates what data has been added/removed by which query.
There are two format options, *single*, or event, and *batched*.

## Schedule results

### Event format

Event is the default result format. Each log line represents a state change.
This format works best for log aggregation systems like Logstash or Splunk.

Example output of `SELECT name, path, pid FROM processes;` (whitespace added for readability):

```json
{
  "action": "added",
  "columns": {
    "name": "osqueryd",
    "path": "/usr/local/bin/osqueryd",
    "pid": "97830"
  },
  "name": "processes",
  "hostname": "hostname.local",
  "calendarTime": "Tue Sep 30 17:37:30 2014",
  "unixTime": "1412123850"
}
```

```json
{
  "action": "removed",
  "columns": {
    "name": "osqueryd",
    "path": "/usr/local/bin/osqueryd",
    "pid": "97650"
  },
  "name": "processes",
  "hostname": "hostname.local",
  "calendarTime": "Tue Sep 30 17:37:30 2014",
  "unixTime": "1412123850"
}
```

This tells us that a binary called "osqueryd" was stopped and a new binary with the same name was started (note the different pids). The data is generated by keeping a cache of previous query results and only logging when the cache changes. If no new processes are started or stopped, the query won't log any results.

### Batch format

If a query identifies multiple state changes, the batched format will include all results in a single log line. If you're programmatically parsing lines and loading them into a backend datastore, this is probably the best solution.

To enable batch log lines, launch osqueryd with the `--log_result_events=false` argument.

Example output of `SELECT name, path, pid FROM processes;` (whitespace added for readability):

```json
{
  "diffResults": {
    "added": [
      {
        "name": "osqueryd",
        "path": "/usr/local/bin/osqueryd",
        "pid": "97830"
      }
    ],
    "removed": [
      {
        "name": "osqueryd",
        "path": "/usr/local/bin/osqueryd",
        "pid": "97650"
      }
    ]
  },
  "name": "processes",
  "hostname": "hostname.local",
  "calendarTime": "Tue Sep 30 17:37:30 2014",
  "unixTime": "1412123850"
}
```

Most of the time the **Event format** is the most appropriate. The next section in the deployment guide describes [log aggregation](deployment/log-aggregation) methods. The aggregation methods describe collecting, searching, and alerting on the results from a query schedule.

## Unique host identification

If you need a way to uniquely identify hosts embedded into osqueryd's results log, then the `--host_identifier` flag is what you're looking for.
By default, `--host_identifier` is set to "hostname".
The host's hostname will be used as the host identifier in results logs.
If hostnames are not unique or consistent in your environment, you can launch osqueryd with `--host_identifier=uuid`.

On Linux, a new UUID will be generated and stored in RocksDB so that it persists across reboots. On OS X, this will attempt to use the hardware UUID and fail back to using a custom generated UUID if that fails.