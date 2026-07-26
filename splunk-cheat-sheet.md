# 🔍 Splunk SPL Cheat Sheet

A practical reference for Splunk Search Processing Language (SPL) — search basics, transforming commands, `eval`, `stats`, `rex`, lookups, and real-world query patterns.

---

## 1. Search Basics

```spl
index=web                                  # Search a specific index
index=web sourcetype=access_combined       # Filter by sourcetype
index=web host=web01                       # Filter by host
index=web status=500                        # Field=value filter
index=web "error"                           # Free-text keyword
index=web "login failed"                    # Phrase (quoted)
index=web status=500 OR status=503          # Boolean OR
index=web status>=400 AND status<500        # Range + AND
index=web NOT status=200                     # Negation
index=web status IN (500, 502, 503)          # IN operator
index=web status!=200                        # Not equal
```

**Wildcards & operators**
```spl
index=web uri=/api/*                         # Wildcard match
index=web user=admin*                        # Prefix match
index=* source="*.log"                       # Match any index, .log sources
```

> 💡 Always lead with `index=` and `sourcetype=` — narrowing early makes searches far faster.

---

## 2. Time Range Modifiers

```spl
index=web earliest=-24h                      # Last 24 hours
index=web earliest=-7d latest=now            # Last 7 days
index=web earliest=-1h@h latest=@h           # Previous full hour (snap)
index=web earliest=-1d@d latest=@d           # Yesterday (snap to day)
index=web earliest="10/01/2024:00:00:00"     # Absolute time
index=web earliest=-30m@m                    # Last 30 min, snapped
```

**Snap-to units:** `@s @m @h @d @w @mon @y` — e.g. `@w0` = start of week (Sunday).

---

## 3. Selecting & Shaping Fields

```spl
... | fields status, uri, clientip           # Keep only these fields
... | fields - _raw, _time                    # Remove fields
... | table _time, host, status, uri          # Tabular output (ordered)
... | rename clientip AS "Client IP"          # Rename a field
... | sort - _time                            # Sort desc by time
... | sort status, -bytes                     # Multi-field sort
... | head 20                                 # First 20 results
... | tail 20                                 # Last 20 results
... | dedup user                              # Remove duplicate users
... | dedup 3 host                            # Keep 3 events per host
... | reverse                                 # Reverse result order
```

---

## 4. `stats` — Aggregation

```spl
... | stats count                                    # Total event count
... | stats count BY status                          # Count grouped by status
... | stats count AS total BY host                   # Named field
... | stats dc(user) AS unique_users                 # Distinct count
... | stats sum(bytes) AS total_bytes BY host        # Sum
... | stats avg(response_time) AS avg_rt BY uri      # Average
... | stats min(bytes), max(bytes), avg(bytes)       # Multiple aggregates
... | stats p95(response_time) BY endpoint           # 95th percentile
... | stats values(status) BY host                   # List of distinct values
... | stats list(user) BY host                       # List (with duplicates)
... | stats first(status), last(status) BY session   # First/last value
... | stats count BY status, host                    # Group by multiple fields
```

**Common stats functions:** `count`, `dc` (distinct count), `sum`, `avg`, `min`, `max`, `median`, `mode`, `stdev`, `range`, `values`, `list`, `first`, `last`, `earliest`, `latest`, `perc95` / `p95`.

---

## 5. `chart` & `timechart`

```spl
... | timechart count                                # Count over time
... | timechart span=1h count BY status              # Hourly, split by status
... | timechart span=5m avg(response_time)           # Avg over 5-min buckets
... | timechart count BY host limit=10               # Top 10 hosts over time
... | chart count BY status, host                    # Matrix: status x host
... | chart avg(bytes) OVER host BY status           # OVER / BY split
... | timechart span=1d count usenull=f              # Exclude null groups
```

---

## 6. `top` & `rare`

```spl
... | top user                                # Most common users (+ %)
... | top limit=5 uri                          # Top 5 URIs
... | top status BY host                       # Top status per host
... | top user showperc=f countfield=hits      # Custom output
... | rare status                              # Least common values
```

---

## 7. `eval` — Compute Fields

```spl
... | eval total = bytes_in + bytes_out
... | eval kb = round(bytes/1024, 2)
... | eval is_error = if(status>=400, "yes", "no")
... | eval level = case(status<300, "ok", status<400, "redirect", status<500, "client_err", 1=1, "server_err")
... | eval full = host . ":" . port                    # String concat
... | eval user = lower(user)                          # Lowercase
... | eval domain = upper(domain)
... | eval msg = coalesce(error_msg, "none")           # First non-null
... | eval flag = if(match(uri, "^/admin"), 1, 0)      # Regex match
... | eval elapsed = latest_time - earliest_time
```

**Numeric/util functions:** `round()`, `ceiling()`, `floor()`, `abs()`, `sqrt()`, `pow()`, `exact()`, `tonumber()`, `tostring()`.
**Conditional:** `if()`, `case()`, `coalesce()`, `nullif()`, `validate()`.

---

## 8. String Functions in `eval`

```spl
... | eval len = len(uri)                              # Length
... | eval sub = substr(uri, 1, 10)                    # Substring
... | eval pos = if(like(uri, "/api/%"), "api", "web") # SQL-style LIKE
... | eval clean = trim(name)                          # Trim whitespace
... | eval parts = split(path, "/")                    # Split to multivalue
... | eval joined = mvjoin(tags, ",")                  # Join multivalue
... | eval replaced = replace(uri, "\d+", "N")         # Regex replace
... | eval upper_host = upper(host)
... | eval formatted = printf("%05d", id)              # Formatted string
```

---

## 9. Field Extraction with `rex`

```spl
# Named capture groups create new fields
... | rex field=_raw "user=(?<username>\w+)"
... | rex field=uri "/(?<endpoint>[^/?]+)"
... | rex field=_raw "(?<ip>\d+\.\d+\.\d+\.\d+)"

# Extract multiple fields at once
... | rex field=_raw "status=(?<status>\d+)\s+bytes=(?<bytes>\d+)"

# Sed mode — substitute/mask (e.g. redact card numbers)
... | rex field=_raw mode=sed "s/\d{16}/XXXX-XXXX-XXXX-XXXX/g"

# max_match for repeated patterns -> multivalue field
... | rex max_match=0 field=_raw "id=(?<ids>\d+)"
```

**`extract` / `kv`** auto-extracts key=value pairs:
```spl
... | extract pairdelim="," kvdelim="="
```

---

## 10. Filtering with `where` & `search`

```spl
... | search status=500                        # Post-pipeline keyword filter
... | where bytes > 10000                       # Numeric comparison
... | where response_time > avg_rt              # Compare two fields
... | where like(uri, "/api/%")                 # SQL LIKE
... | where match(user, "^admin")               # Regex
... | where isnotnull(error_msg)                # Null check
... | where status>=400 AND host="web01"
... | where cidrmatch("10.0.0.0/8", clientip)   # IP in subnet
```

> `search` matches keywords/fields like the base search; `where` uses eval-style expressions and comparisons between fields.

---

## 11. Multivalue Fields

```spl
... | makemv delim="," tags                     # Split field into multivalue
... | mvexpand tags                             # One row per value
... | eval count = mvcount(tags)                # Number of values
... | eval first_tag = mvindex(tags, 0)         # Nth value
... | eval joined = mvjoin(tags, "; ")          # Join back to string
... | where mvcount(errors) > 1                 # Filter on MV count
... | eval filtered = mvfilter(match(tags, "prod"))
... | nomv tags                                 # Convert MV back to single
```

---

## 12. Lookups

```spl
# Enrich events using a lookup table
... | lookup user_roles user OUTPUT role, department
... | lookup geo_ip ip AS clientip OUTPUT country, city

# Inline lookup listing
| inputlookup user_roles.csv
| inputlookup user_roles.csv WHERE department="IT"

# Write results to a lookup
... | outputlookup my_results.csv

# Show available lookups
| rest /services/data/lookup-table-files
```

---

## 13. Subsearches & Joins

```spl
# Subsearch (runs first, feeds the outer search)
index=web [ search index=security action=blocked | fields clientip ]

# Format a subsearch as a filter
index=web [ search index=alerts | fields user | format ]

# join (use sparingly — stats is usually faster)
index=web | join type=inner user [ search index=hr | fields user, dept ]

# append / appendcols
... | append [ search index=other | stats count ]
... | appendcols [ search ... | stats avg(x) ]

# Better alternative to join in many cases:
index=web OR index=hr | stats values(dept) AS dept, count BY user
```

---

## 14. `transaction` — Group Related Events

```spl
... | transaction sessionid                            # Group by session
... | transaction clientip maxspan=30m                 # Time-bounded
... | transaction user startswith="login" endswith="logout"
... | transaction sessionid maxpause=5m maxevents=50
```

Adds fields: `duration`, `eventcount`. Powerful but expensive — prefer `stats` when possible.

---

## 15. Formatting Output

```spl
... | fieldformat bytes = tostring(bytes, "commas")    # 1,234,567
... | eval _time = strftime(_time, "%Y-%m-%d %H:%M:%S") # Format time
... | eval epoch = strptime(datestr, "%Y-%m-%d")        # Parse string to epoch
... | convert ctime(_time) AS readable                  # Epoch -> readable
... | addcoltotals                                      # Add a totals row
... | addtotals                                         # Row-wise total column
... | fillnull value=0                                  # Replace nulls with 0
... | fillnull value="N/A" status
```

---

## 16. Real-World Query Examples

**Top 10 URLs returning server errors**
```spl
index=web status>=500
| stats count BY uri
| sort -count
| head 10
```

**Response time 95th percentile per endpoint over time**
```spl
index=web
| timechart span=5m p95(response_time) BY endpoint
```

**Failed login attempts by source IP (last 24h)**
```spl
index=security action=failure earliest=-24h
| stats count AS attempts BY src_ip, user
| where attempts > 5
| sort -attempts
```

**Bandwidth usage per host**
```spl
index=network
| stats sum(bytes) AS total_bytes BY host
| eval GB = round(total_bytes/1024/1024/1024, 2)
| sort -GB
```

**Error rate percentage over time**
```spl
index=web
| timechart span=1h count(eval(status>=500)) AS errors, count AS total
| eval error_rate = round(errors/total*100, 2)
| fields _time, error_rate
```

**Unique users per day**
```spl
index=app
| timechart span=1d dc(user) AS unique_users
```

**Detect a spike vs. previous period**
```spl
index=web
| timechart span=1h count
| delta count AS change
| where change > 1000
```

**Extract and count API endpoints**
```spl
index=web sourcetype=access_combined
| rex field=uri "/api/(?<endpoint>[^/?]+)"
| stats count BY endpoint
| sort -count
```

**Slowest 20 requests**
```spl
index=web
| sort -response_time
| head 20
| table _time, host, uri, response_time, status
```

**Geographic breakdown of traffic**
```spl
index=web
| iplocation clientip
| stats count BY Country
| sort -count
| geostats count                        # Or map it directly
```

---

## 17. Useful Generating & Utility Commands

```spl
| makeresults                                  # Create a single dummy event
| makeresults count=5                          # Create N dummy events
| gentimes start=-7                            # Generate time ranges
| tstats count WHERE index=web BY host         # Fast metadata-based stats
| metadata type=hosts index=web                # Hosts/sources/sourcetypes
| eventcount index=*                           # Event counts per index
| dbinspect index=web                          # Bucket-level detail
| rest /services/server/info                   # Query Splunk's REST API
```

**`tstats`** is dramatically faster than `stats` for accelerated data models / indexed fields:
```spl
| tstats count WHERE index=web BY _time span=1h
```

---

## 18. Common Aliases & Cheats

| Task | Command |
|------|---------|
| Count events | `stats count` |
| Unique count | `stats dc(field)` |
| Over time | `timechart` |
| Most common | `top` |
| Least common | `rare` |
| Remove dupes | `dedup` |
| New field | `eval` |
| Extract field | `rex` |
| Enrich | `lookup` |
| Keep columns | `table` / `fields` |
| Filter after search | `where` / `search` |

---

### Search pipeline mental model
```
index=... (find events)  →  transform (eval/rex)  →  aggregate (stats/timechart)  →  format (sort/table)
```
Each `|` passes results to the next command, left to right. Narrow early, aggregate late.

Happy searching! 🔎
