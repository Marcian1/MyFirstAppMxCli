# ResourceMonitor module

A reusable Mendix module that shows a developer, in one page, every resource the running application is using:
runtime, JVM, CPU, memory, threads, garbage collection, database, web server and sessions.

Built and tested on Mendix 11.15.0 (Java 25). Package: `dist/ResourceMonitor.mpk`.

## What the dashboard shows

| Tab | Content |
|---|---|
| Overview | Mendix runtime version, active and anonymous sessions, JVM name/version/uptime, process and system CPU load, load average, processors, open file descriptors, heap and non-heap memory, physical memory, swap, disk space, web server (Jetty) thread pool, JVM arguments |
| Threads | Every live JVM thread: id, name, state, CPU time, blocked/waited counts, the lock it is blocked on, its top frame. **Stack** opens the full stack trace. Deadlocked threads are named. |
| Memory and GC | Every JVM memory pool (used, committed, max, peak, % of max) and every garbage collector (collections, time), loaded classes |
| Database | Product, version, JDBC driver, URL, user, isolation level, connection validity check time, **connection pool** (active, idle, max, threads waiting, longest wait) and **data storage activity** (selects, inserts, updates, deletes, transactions) |
| Sessions | Each user session: user, last active, roles, user agent |
| JMX metrics | Every other numeric or text attribute published over JMX by the runtime, connection pools, web server and buffer pools. Filterable. |
| System properties | JVM system properties, secrets masked |

The page is a point-in-time capture. **Refresh** re-captures into the same snapshot.

## Using it in an app

1. In Studio Pro: *App > Import module package* and pick `ResourceMonitor.mpk`.
   The module needs `Atlas_Core` (any standard Mendix app has it) for its layouts.
2. Map the module role `ResourceMonitor.Admin` into the user role(s) that may look at this data, for example your Administrator role.
3. Add a menu item or button that opens `ResourceMonitor.Dashboard`.

For this repository's demo app steps 2 and 3 are done in `mdlsource/resourcemonitor_05_navigation.mdl`.

## Public microflows

| Microflow | Use |
|---|---|
| `SUB_CaptureSnapshot(IncludeStackTraces, IncludeJmxMetrics)` | Returns a `Snapshot` with child lists. Call it from your own logic, e.g. to alert when `DbPoolWaiters > 0` or `DeadlockedThreadCount > 0`. |
| `ACT_LogSnapshot` | Logs a one-line summary to log node `ResourceMonitor`. Point a scheduled event at it to keep a resource history in the application log. |
| `DS_Snapshot` / `ACT_Refresh` | Used by the dashboard. |

The Snapshot and its samples are **non-persistent**: nothing is written to your database.

## Security

- Everything is restricted to the module role `ResourceMonitor.Admin` (entities, microflows and pages).
- The module reads JVM and runtime state only. It never reads environment variables.
- Masked: values of system properties whose name suggests a secret, `password=`/`token=`/`key=` style values in the JVM arguments and the database URL, and JMX attributes whose name suggests a secret.
- Stack traces and thread names can still reveal internals. Give the role only to people allowed to see them.

## Platform notes

- System load average and open file descriptors are not available on Windows and show empty there.
- Which JMX metrics exist depends on the runtime and database in use. The database connection pool, data storage and web server fields are read from the Mendix runtime's own MBeans and stay empty if a future runtime renames them. The JMX tab always shows what is really published.
- Each section is collected separately, so a probe that fails on a platform leaves the rest of the dashboard intact.

## Testing

`tests/resourcemonitor.test.mdl` holds 7 tests that run against the real runtime:

```
mxcli test tests/resourcemonitor.test.mdl -p MyFirstApp.mpr --local --db-type hsqldb
```

Manual check in the demo app: run it (`mxcli run --local --db-type hsqldb --detach -p MyFirstApp.mpr`, with `JAVA_HOME` pointing at JDK 25), open http://127.0.0.1:8080/ and choose **Resource Monitor**.

## Rebuilding the package

Close Studio Pro, then:

```
mxcli marketplace package ResourceMonitor -p MyFirstApp.mpr -o dist
```

## Source

The module is defined by the scripts in `mdlsource/resourcemonitor_*.mdl`; the Java is in
`mdlsource/resourcemonitor_02_java.mdl` (generated to `javasource/resourcemonitor/actions/JA_Capture.java`).
