# Findings

- customer_nav.mdl is not re-runnable: ALTER NAVIGATION INSERT adds a duplicate "Customers" item if executed twice. Run once.
- Lint after adding Customer: SEC001/SEC006/MPR007 (no access rules or page roles), ARCH002/ARCH003 (no save microflow, no business key). Not addressed yet.
- Verified: `mxcli check`, `exec` (second run unchanged), `lint` (0 errors), `docker check` (0 errors) on Mendix 11.15.0.
- Added customer_security.mdl: User role gets full CRUD on Customer and view on both Customer pages (re-run: unchanged). Cleared SEC001, SEC006, MPR007. Remaining: SEC008 (Email readable by every User without an XPath constraint; acceptable for a demo), ARCH002, ARCH003.
- mxcli refuses to write while Studio Pro has the project open (lock file). Close the project first.
- Ran the app locally (JDK 25, `mxcli run --local --db-type hsqldb`, needs JAVA_HOME pointed at JDK 25 since the machine default is Java 8). Customer overview loads, "Name is required" validation fires, a created customer saves and lists. First version had no New button; added via customer_new_button.mdl (not re-runnable: would insert a second button).
