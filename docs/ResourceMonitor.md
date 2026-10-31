# ResourceMonitor module

A reusable Mendix module that gives a developer a Grafana-style view of the running application: **history charts
for the last 14 days, minute by minute**, plus a live view of runtime, JVM, CPU, memory, threads, GC, database,
web server and session count.

Built and tested on Mendix 11.15.0 (Java 25). Package: `dist/ResourceMonitor.mpk`.

## How it works

1. A scheduled event, `ResourceMonitor.SE_CollectPoint`, runs **every minute** and stores one `ResourcePoint`
   (about 30 numbers: CPU, heap, threads, DB pool, sessions, runtime counters ...) in your application's database.
2. Samples older than `ResourceMonitor.RetentionDays` (default **14**) are deleted automatically.
3. The dashboard reads the samples for the period you pick, groups them into time buckets and draws them with
   the Time series chart widget. Counters (selects, inserts, GC ...) are turned into per-minute rates.

Storage: one row per minute is about 20,000 rows for 14 days.

## The dashboard

**Overview tab**
- Six status tiles (process CPU, heap, DB connections, web request queue, sessions, deadlocked threads), coloured green, amber or red.
  Thresholds: CPU 70/90 %, heap 75/90 %, pool 70/90 %, any queued request or deadlock is red.
- Range buttons (1 hour, 6 hours, 24 hours, 3 days, 7 days, 14 days) with the active one highlighted, and a custom From / To period.
- Six charts (Time series): CPU, memory (heap used, committed, non-heap, heap max line), database connections (active, idle,
  waiters, pool size line), web server threads (busy, queued, max line), database activity per minute (selects, updates), sessions.
  Waiters and queued requests keep the bucket **maximum** so a short spike is not averaged away.
- Every tile, chart and table title has an "i" Tooltip with what the metric is, why it matters, what to watch for and its JMX source.

| View | Points per chart |
|---|---|
| 1 hour, 6 hours | one per minute |
| 24 hours | one per 2 minutes |
| 3 / 7 / 14 days | one per 6 / 14 / 28 minutes |
| Custom period up to 24 hours | one point per minute |

**Other tabs:** Threads (with stack traces in a popup), Memory and GC (Eden, Survivor, Old Gen, Metaspace; collectors), Environment (runtime, OS, database product).

**History page** (link on the Overview tab): sample count and age, store a sample now, and demo data: 14 days of synthetic, flagged history
that fills the time before the oldest real sample.

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
| `SUB_CaptureSnapshot(IncludeDetails)` | Returns a live `Snapshot`. With `IncludeDetails` it also lists threads (with stack traces), memory pools and collectors. |
| `SE_CollectPoint` | The per-minute collector (numbers only). Can also be called from your own scheduled event. |
| `ACT_LogSnapshot` | Logs a one-line summary to log node `ResourceMonitor`. |

## Security

- Everything is restricted to the module role `ResourceMonitor.Admin` (entities, microflows and pages).
- The module reads JVM and runtime state only. It never reads environment variables, user names, user agents, JVM arguments,
  system properties, the database URL or the database user. Sessions are only counted.
- Thread names and stack traces can still reveal internal class names. Give the role only to people allowed to see them.

## Limits and notes

- **Several runtime instances** (cluster): each instance would store its own sample every minute, so the history
  would hold one point per instance. Disable collection on all but one (`CollectionEnabled`) if that matters.
- Counters are per runtime start. A restart shows as a gap in the rates, not as a spike.
- A gap longer than 30 minutes (collector off or app stopped) is not turned into a rate.
- Database pool, storage and web server numbers come from the Mendix runtime's own JMX beans
  (`com.mendix:type=DataStorage`, `JettyThreadPool`, and the DBCP pool). They stay empty if a future runtime renames them;
  the JMX tab always shows what is really published.
- In Studio Pro's / `mxcli run` development mode the browser client logs every object and long ranges feel slower
  than they will in a deployed app.

## Testing

`tests/resourcemonitor.test.mdl` holds 11 tests that run against the real runtime (live capture with and without details, collector, retention,
minute resolution, buckets, chart limits, demo data, deletion; run with JAVA_HOME at JDK 25):

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

The module is defined by the scripts `mdlsource/resourcemonitor_01..09*.mdl`, applied in order. The Overview redesign is `10_overview_java`, `11_overview_microflows`, `12_overview_page` and `13_overview_cleanup`
(applied last; they replace the dashboard and drop what is no longer collected). The Java is in `02`, `07` and `10`.
