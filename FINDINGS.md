# Findings

- customer_nav.mdl is not re-runnable: ALTER NAVIGATION INSERT adds a duplicate "Customers" item if executed twice. Run once.
- Lint after adding Customer: SEC001/SEC006/MPR007 (no access rules or page roles), ARCH002/ARCH003 (no save microflow, no business key). Not addressed yet.
- Verified: `mxcli check`, `exec` (second run unchanged), `lint` (0 errors), `docker check` (0 errors) on Mendix 11.15.0.
- Added customer_security.mdl: User role gets full CRUD on Customer and view on both Customer pages (re-run: unchanged). Cleared SEC001, SEC006, MPR007. Remaining: SEC008 (Email readable by every User without an XPath constraint; acceptable for a demo), ARCH002, ARCH003.
- mxcli refuses to write while Studio Pro has the project open (lock file). Close the project first.
- Ran the app locally (JDK 25, `mxcli run --local --db-type hsqldb`, needs JAVA_HOME pointed at JDK 25 since the machine default is Java 8). Customer overview loads, "Name is required" validation fires, a created customer saves and lists. First version had no New button; added via customer_new_button.mdl (not re-runnable: would insert a second button).
- ResourceMonitor module: 7/7 `mxcli test` pass; docker check 0 errors; browser-verified every dashboard tab, Refresh (in place) and the thread Stack popup. Package: dist/ResourceMonitor.mpk (needs Atlas_Core).
- The runtime publishes useful MBeans: com.mendix:type=DataStorage (selects/inserts/updates/deletes/transactions), type=JettyThreadPool (Threads, IdleThreads, MaxThreads, QueueSize), org.apache.commons.pool2 GenericObjectPool (DBCP connection pool). Attribute names were read from the running app, not guessed.
- Local dev sessions are named `Anonymous_<uuid>`; anonymous detection matches that prefix.
- Accepted lint: DESIGN001 on Snapshot (72 attributes, one non-persistent snapshot by design), CONV013/CONV010 (Java action never throws, every section is try/catch).
- Mendix runtime default on this PC is Java 8; mxcli run/test need JAVA_HOME at JDK 25.
- History (14 days, minute by minute): scheduled event SE_CollectPoint stores one persistent ResourcePoint per minute; retention constant RetentionDays=14 enforced in JA_RecordPoint. Charts use the Time series widget bound to non-persistent ChartPoint buckets (Snapshot_ChartPoint). 12/12 mxcli tests pass; docker check 0 errors.
- Browser limit measured: 20,160 points x 8 charts crashed the browser tab. Defaults are 720 points per chart (1440 with Max detail); true minute resolution for any day via the custom From/To period (up to 24 h = one point per minute). Dev-mode client logs every object (328k console messages), so long ranges are slower here than in a deployed app (server answers in ~0.8 s, 2.9 MB for 1440 points).
- mxcli quirk: `create or modify microflow` could not splice a restructured SUB_LoadHistory (variables inside a new branch "not declared"); `drop microflow if exists` + `create microflow` works and callers (by name) reconnect.
- Demo data is flagged IsDemo and only fills the time before the oldest real sample.
- Import test (fresh blank 11.15.0 app, `mxcli marketplace install --file dist/ResourceMonitor.mpk`): installs 10 entities, 3 pages, 15 microflows, 6 Java actions, 2 constants and the SE_CollectPoint scheduled event; docker check 0 errors; the collector stored its first sample on its own; demo history + 14-day charts verified. Only wiring needed: map ResourceMonitor.Admin into a user role and add a menu item. The module does not ship the Time series / Data grid 2 widgets; the blank template already contains them.
