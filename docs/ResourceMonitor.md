# ResourceMonitor module

A reusable Mendix module that gives a developer a Grafana-style view of the running application: **history charts
for the last 14 days, minute by minute**, plus a live view of runtime, JVM, CPU, memory, threads, GC, database,
web server and sessions.

Built and tested on Mendix 11.15.0 (Java 25). Package: `dist/ResourceMonitor.mpk`.

## How it works

1. A scheduled event, `ResourceMonitor.SE_CollectPoint`, runs **every minute** and stores one `ResourcePoint`
   (about 30 numbers: CPU, heap, threads, DB pool, sessions, runtime counters ...) in your application's database.
2. Samples older than `ResourceMonitor.RetentionDays` (default **14**) are deleted automatically.
3. The dashboard reads the samples for the period you pick, groups them into time buckets and draws them with
   the Time series chart widget. Counters (selects, inserts, GC ...) are turned into per-minute rates.

Storage: one row per minute is about 20,000 rows for 14 days.

## The dashboard

**Overview tab** (charts)
- Period buttons: 1 hour, 6 hours, 24 hours, 3 days, 7 days, 14 days, or a **custom From / To period**.
- Eight charts: CPU, memory (heap, committed, non-heap), threads (JVM and web server), database connections
  (active, idle, waiting), database activity per minute (selects, inserts, updates, deletes, transactions),
  sessions, garbage collection, and database response time. Each chart has a legend and zoom.

**Resolution**

| View | Points per chart |
|---|---|
| 1 hour, 6 hours | one per minute |
| 24 hours | one per 2 minutes |
| 3 / 7 / 14 days | one per 6 / 14 / 28 minutes (averages; CPU, heap, threads, pool and sessions also show the maximum) |
| **Max detail** toggle | up to 1440 points: 24 hours is every minute, 14 days is 14-minute buckets |
| **Custom period** up to 24 hours | **one point per minute**, for any day inside the retention period |

Showing every minute of 14 days at once would be 20,160 points in each of eight charts, which a browser cannot
draw comfortably (it crashed the test browser). Pick a custom period to look at any stretch of the 14 days minute by minute.

**Other tabs:** Live now (current values), Threads (with stack traces), Memory and GC, Database (connection pool,
storage activity), Sessions, JMX metrics, System properties.

**History page** (link on the Overview tab): how many samples are stored and from when to when, store a sample now,
and **demo data**: generate 14 days of synthetic, flagged history so the charts can be tried immediately, and delete it again.
Demo data fills the time before the oldest real sample, so real and demo data never overlap.

## Using it in an app

1. In Studio Pro: *App > Import module package* and pick `ResourceMonitor.mpk`.
   Requirements: `Atlas_Core` and the **Time series** (Charts), **Data grid 2** and filter widgets, which a standard
   Mendix project already contains.
2. Map the module role `ResourceMonitor.Admin` into the user role(s) that may see this data.
3. Add a menu item or button that opens `ResourceMonitor.Dashboard`.
4. Make sure scheduled events run in your environment. Without them no history is collected.

Constants: `RetentionDays` (default 14) and `CollectionEnabled` (default true).

For this repository's demo app steps 2 and 3 are done in `mdlsource/resourcemonitor_05_navigation.mdl`.

## Public microflows

| Microflow | Use |
|---|---|
| `SUB_CaptureSnapshot(IncludeStackTraces, IncludeJmxMetrics)` | Returns a live `Snapshot` with child lists. Call it from your own logic. |
| `SE_CollectPoint` | The per-minute collector. Can also be called from your own scheduled event. |
| `ACT_LogSnapshot` | Logs a one-line summary to log node `ResourceMonitor`. |

## Security

- Everything is restricted to the module role `ResourceMonitor.Admin` (entities, microflows and pages).
- The module reads JVM and runtime state only. It never reads environment variables.
- Masked: system properties whose name suggests a secret, `password=`/`token=`/`key=` style values in the JVM arguments
  and the database URL, and JMX attributes whose name suggests a secret.
- Stack traces and thread names can still reveal internals. Give the role only to people allowed to see them.

## Limits and notes

- **Several runtime instances** (cluster): each instance would store its own sample every minute, so the history
  would hold one point per instance. Disable collection on all but one (`CollectionEnabled`) if that matters.
- Counters are per runtime start. A restart shows as a gap in the rates, not as a spike.
- A gap longer than 30 minutes (collector off or app stopped) is not turned into a rate.
- System load average and open file descriptors are not available on Windows and show empty there.
- Database pool, storage and web server numbers come from the Mendix runtime's own JMX beans
  (`com.mendix:type=DataStorage`, `JettyThreadPool`, and the DBCP pool). They stay empty if a future runtime renames them;
  the JMX tab always shows what is really published.
- In Studio Pro's / `mxcli run` development mode the browser client logs every object and long ranges feel slower
  than they will in a deployed app.

## Testing

`tests/resourcemonitor.test.mdl` holds 12 tests that run against the real runtime (live capture, collector, retention,
minute resolution, buckets, demo data, deletion):

```
mxcli test tests/resourcemonitor.test.mdl -p MyFirstApp.mpr --local --db-type hsqldb
```

Manual check in the demo app: run it (`mxcli run --local --db-type hsqldb --detach -p MyFirstApp.mpr`, with `JAVA_HOME`
pointing at JDK 25), open http://127.0.0.1:8080/, choose **Resource Monitor**, then **History settings and demo data**
and **Generate demo history**, and go back to the dashboard.

## Rebuilding the package

Close Studio Pro, then:

```
mxcli marketplace package ResourceMonitor -p MyFirstApp.mpr -o dist
```

## Source

The module is defined by the scripts `mdlsource/resourcemonitor_01..09*.mdl`, applied in order. The Java is in
`resourcemonitor_02_java.mdl` and `resourcemonitor_07_history_java.mdl`.
