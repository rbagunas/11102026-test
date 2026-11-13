Core charts (C1–C5)


/*
 - Name: C1 Devices by Type and Subtype
 - Purpose: Counts in-service devices by type (physical, virtual) and subtype (laptop, desktop,
   switch, router, firewall, server). Where the hardware model has no subtype, infers one from
   the network-device flag, hardware model name and OS name, labeled "(inferred)".
 - Chart: Sunburst (hierarchy device_type > device_subtype, metric SUM(devices))
 - Device42 version: v19.11
 - Date Created: 11/10/26
 - Changes: 11/10/26 Added inferred subtype for devices whose hardware model has no subtype
*/
SELECT device_type, device_subtype, COUNT(*) AS devices
FROM (
  SELECT COALESCE(d.type, 'unknown') AS device_type,
         COALESCE(NULLIF(d.physicalsubtype, ''), NULLIF(d.virtualsubtype, ''),
           CASE
             WHEN d.network_device = true AND x.hint ~* '(fortigate|fortios|pan-os|palo alto|firepower|\masa\M|check ?point)' THEN 'Firewall (inferred)'
             WHEN d.network_device = true AND x.hint ~* '(router|\misr|\masr\M)' THEN 'Router (inferred)'
             WHEN d.network_device = true AND x.hint ~* '(switch|catalyst|nexus|procurve|aruba|ws-c)' THEN 'Switch (inferred)'
             WHEN d.network_device = true THEN 'Network device (inferred)'
             WHEN x.hint ~* '(latitude|thinkpad|elitebook|probook|zbook|macbook|surface (laptop|pro|book)|vivobook|zenbook|travelmate)' THEN 'Laptop (inferred)'
             WHEN x.hint ~* '(optiplex|thinkcentre|prodesk|elitedesk|imac|mac mini|veriton)' THEN 'Desktop (inferred)'
             WHEN x.hint ~* '(server|linux|aix|esxi|solaris)' THEN 'Server (inferred)'
             ELSE 'not set'
           END) AS device_subtype
  FROM view_device_v2 d
  LEFT JOIN view_hardware_v1 h ON h.hardware_pk = d.hardware_fk
  CROSS JOIN LATERAL (SELECT COALESCE(h.name, '') || ' ' || COALESCE(d.os_name, '') AS hint) x
  WHERE d.in_service = true
) t
GROUP BY 1, 2
ORDER BY 1, 3 DESC







/*
 - Name: C2 Devices by Days Since Last Update
 - Purpose: Groups in-service physical devices (excluding network devices) by days since the
   record was last updated, to surface idle or missing hardware.
 - Chart: Bar (x-axis last_update, metric SUM(devices))
 - Device42 version: v19.11
 - Date Created: 11/10/26
 - Changes: 11/10/26 Replaced 30-day cut-off with buckets so the chart returns rows early in the project
*/
SELECT CASE
         WHEN CURRENT_DATE - d.last_edited::date <= 7  THEN '1 Updated in last 7 days'
         WHEN CURRENT_DATE - d.last_edited::date <= 14 THEN '2 8-14 days ago'
         WHEN CURRENT_DATE - d.last_edited::date <= 30 THEN '3 15-30 days ago'
         ELSE '4 Over 30 days ago'
       END AS last_update,
       COUNT(*) AS devices
FROM view_device_v2 d
WHERE d.type = 'physical'
  AND d.in_service = true
  AND d.network_device = false
GROUP BY 1
ORDER BY 1








/*
 - Name: C2b Idle Hardware Candidates
 - Purpose: Lists in-service physical devices with no update for more than 14 days, for IT to
   confirm before redeploying or retiring them.
 - Chart: Table (all columns, sorted by days_unchanged descending)
 - Device42 version: v19.11
 - Date Created: 11/10/26
 - Changes: none
*/
SELECT d.name AS device,
       COALESCE(NULLIF(d.physicalsubtype, ''), 'not set') AS subtype,
       d.serial_no,
       d.last_edited::date AS last_updated,
       CURRENT_DATE - d.last_edited::date AS days_unchanged
FROM view_device_v2 d
WHERE d.type = 'physical'
  AND d.in_service = true
  AND d.network_device = false
  AND d.last_edited < CURRENT_DATE - 14
ORDER BY days_unchanged DESC









/*
 - Name: C3 Operating Systems by Support Status
 - Purpose: Lists every OS and version on in-service devices with a support status. Uses entered
   OS EOL/EOS dates first; where none are entered, flags known end-of-support versions by name.
 - Chart: Table (sorted by support_status) + Big Number (devices past end of support)
 - Device42 version: v19.11
 - Date Created: 11/10/26
 - Changes: 11/10/26 Lists all operating systems instead of only end-of-support matches
*/
SELECT d.os_name,
       COALESCE(d.os_version, '') AS os_version,
       CASE
         WHEN ose.eos < CURRENT_DATE                THEN '1 Past end of support'
         WHEN ose.eos <= CURRENT_DATE + 365         THEN '2 Support ends within 12 months'
         WHEN ose.eos IS NOT NULL                   THEN '4 Supported'
         WHEN (d.os_name || ' ' || COALESCE(d.os_version, '')) ~* '(windows (xp|vista|7|8)|windows 10|server 2003|server 2008|server 2012|centos|red hat.*\m(5|6|7)\.|aix.*\m(5|6)\.|aix.*\m7\.1)'
                                                    THEN '1 Past end of support (by name)'
         ELSE '3 No end-of-support date entered'
       END AS support_status,
       COUNT(*) AS devices
FROM view_device_v2 d
LEFT JOIN view_oseoleos_v1 ose ON ose.os_fk = d.os_fk AND ose.version = d.os_version
WHERE d.in_service = true
  AND COALESCE(d.os_name, '') <> ''
GROUP BY 1, 2, 3
ORDER BY 3, 4 DESC











/*
 - Name: C4 Certificates by Expiry Window
 - Purpose: Counts discovered SSL certificates by days until expiry, to prevent outages from
   expired certificates.
 - Chart: Bar (x-axis expiry_window, metric SUM(certificates))
 - Device42 version: v19.11
 - Date Created: 11/10/26
 - Changes: none
*/
SELECT CASE
         WHEN valid_to < CURRENT_DATE            THEN '0 Expired'
         WHEN valid_to <= CURRENT_DATE + 30      THEN '1 Within 30 days'
         WHEN valid_to <= CURRENT_DATE + 60      THEN '2 31-60 days'
         WHEN valid_to <= CURRENT_DATE + 90      THEN '3 61-90 days'
         ELSE '4 More than 90 days'
       END AS expiry_window,
       COUNT(*) AS certificates
FROM view_certificate_v1
GROUP BY 1
ORDER BY 1










/*
 - Name: C4b Certificates Expiring Within 90 Days
 - Purpose: Lists certificates already expired or expiring within 90 days, soonest first, for
   the owners to renew.
 - Chart: Table (all columns, sorted by valid_to ascending)
 - Device42 version: v19.11
 - Date Created: 11/10/26
 - Changes: none
*/
SELECT *
FROM view_certificate_v1
WHERE valid_to <= CURRENT_DATE + 90
ORDER BY valid_to









/*
 - Name: C5 Hardware by Warranty Status
 - Purpose: Counts in-service physical devices by warranty status, using the latest warranty end
   date per device. Splits devices with no record into ready for warranty sync (Dell, Lenovo,
   IBM, Cisco, Meraki), HP (manual import), other manufacturer, and missing serial number.
 - Chart: Bar, horizontal (y-axis warranty_status, metric SUM(devices))
 - Device42 version: v19.11
 - Date Created: 11/10/26
 - Changes: 11/10/26 Split "No warranty record" by what is needed to fill it
*/
SELECT CASE
         WHEN w.warranty_end >= CURRENT_DATE + 90 THEN '1 Active'
         WHEN w.warranty_end >= CURRENT_DATE      THEN '2 Expires within 90 days'
         WHEN w.warranty_end IS NOT NULL          THEN '3 Expired'
         WHEN COALESCE(w.serial_no, '') = ''      THEN '7 No record: serial number missing'
         WHEN lower(COALESCE(w.vendor, '')) ~ '(dell|lenovo|ibm|cisco|meraki)' THEN '4 No record: ready for warranty sync'
         WHEN lower(COALESCE(w.vendor, '')) ~ '^(hp|hewlett)' THEN '5 No record: HP, import manually'
         ELSE '6 No record: other manufacturer'
       END AS warranty_status,
       COUNT(*) AS devices
FROM (
  SELECT d.device_pk, d.serial_no, v.name AS vendor, MAX(pli.end_date) AS warranty_end
  FROM view_device_v2 d
  LEFT JOIN view_hardware_v1 h ON h.hardware_pk = d.hardware_fk
  LEFT JOIN view_vendor_v1 v ON v.vendor_pk = h.vendor_fk
  LEFT JOIN view_purchaselineitems_to_devices_v1 ptd ON ptd.device_fk = d.device_pk
  LEFT JOIN view_purchaselineitem_v1 pli ON pli.purchaselineitem_pk = ptd.purchaselineitem_fk
  WHERE d.type = 'physical'
    AND d.in_service = true
  GROUP BY d.device_pk, d.serial_no, v.name
) w
GROUP BY 1
ORDER BY 1



/*
 - Name: C6 Physical Devices by Subtype, Hardware Model and Vendor
 - Purpose: Counts in-service physical devices (end points) by device type, physical subtype,
   hardware model name and vendor. Devices with no subtype show as "not set"; devices with no
   hardware model or vendor show as "(no hardware model)" / "(no vendor)".
 - Chart: Table (or Bar: dimension physical_subtype, metric SUM(endpoints))
 - Device42 version: v19.11
 - Date Created: 11/13/26
 - Changes: 11/13/26 Initial version
            11/13/26 Added device_type column (leftmost); removed total_endpoints column
*/
SELECT COALESCE(NULLIF(d.type, ''), 'unknown')                    AS device_type,
       COALESCE(NULLIF(d.physicalsubtype, ''), 'not set')         AS physical_subtype,
       COALESCE(NULLIF(h.name, ''), '(no hardware model)')        AS hardware_model,
       COALESCE(NULLIF(v.name, ''), '(no vendor)')                AS vendor,
       COUNT(*)                                                   AS endpoints
FROM view_device_v2 d
LEFT JOIN view_hardware_v1 h ON h.hardware_pk = d.hardware_fk
LEFT JOIN view_vendor_v1   v ON v.vendor_pk   = h.vendor_fk
WHERE d.in_service = true
  AND d.type = 'physical'
GROUP BY 1, 2, 3, 4
ORDER BY 2, 5 DESC, 3



###################################################
##              SLM charts (S1–S5)               ##
###################################################



/*
 - Name: S1 Licensed vs Discovered, Top 10 Managed Titles
 - Purpose: Compares purchased (licensed) and installed (discovered) counts for the ten managed
   software titles with the most installs, with vendor and license model.
 - Chart: Bar, horizontal grouped (y-axis title, metrics SUM(licensed) and SUM(discovered))
 - Device42 version: v19.11
 - Date Created: 11/10/26
 - Changes: 11/10/26 Empty licensed_count shown as 0
*/
SELECT s.name AS title,
       v.name AS vendor,
       lm.name AS license_model,
       COALESCE(s.licensed_count, 0) AS licensed,
       s.discovered_count AS discovered
FROM view_software_v1 s
LEFT JOIN view_vendor_v1 v ON v.vendor_pk = s.vendor_fk
LEFT JOIN view_softwarelicensemodel_v1 lm ON lm.softwarelicensemodel_pk = s.softwarelicensemodel_fk
WHERE lower(s.software_type) = 'managed'
ORDER BY s.discovered_count DESC
LIMIT 10








/*
 - Name: S2 Shelfware - Licenses to Stop Paying For
 - Purpose: Lists managed titles where purchased licenses exceed installs, with the annual peso
   value of the unused licenses. Uses Device42 licensed_count when entered, otherwise the
   contract list in the query.
 - Chart: Big Number (SUM(reclaimable_php)) + Table
 - Device42 version: v19.11
 - Date Created: 11/10/26
 - Changes: 11/10/26 Added contract list so the query works before license counts are entered
 - Note: the contract figures below are EXAMPLES; replace them with Insular Life's contracts
*/
SELECT s.name AS title,
       s.discovered_count AS installed,
       COALESCE(NULLIF(s.licensed_count, 0), c.purchased) AS purchased,
       COALESCE(NULLIF(s.licensed_count, 0), c.purchased) - s.discovered_count AS unused_licenses,
       c.unit_cost_php,
       (COALESCE(NULLIF(s.licensed_count, 0), c.purchased) - s.discovered_count) * c.unit_cost_php AS reclaimable_php
FROM view_software_v1 s
JOIN (VALUES
        ('Datto RMM',             800, 1200),
        ('Sophos Endpoint Agent', 800, 1500)
     ) AS c(title, purchased, unit_cost_php)
  ON lower(s.name) = lower(c.title)
WHERE COALESCE(NULLIF(s.licensed_count, 0), c.purchased) > s.discovered_count
ORDER BY reclaimable_php DESC







/*
 - Name: S3 Audit Exposure - Installed Beyond Purchased
 - Purpose: Lists managed titles where installs exceed purchased licenses, with the annual peso
   cost of closing the gap. Uses Device42 licensed_count when entered, otherwise the contract
   list in the query.
 - Chart: Big Number (SUM(exposure_php)) + Table
 - Device42 version: v19.11
 - Date Created: 11/10/26
 - Changes: 11/10/26 Added contract list; empty licensed_count no longer hides rows
 - Note: keep the contract list identical to S2
*/
SELECT s.name AS title,
       s.discovered_count AS installed,
       COALESCE(NULLIF(s.licensed_count, 0), c.purchased) AS purchased,
       s.discovered_count - COALESCE(NULLIF(s.licensed_count, 0), c.purchased) AS shortfall,
       c.unit_cost_php,
       (s.discovered_count - COALESCE(NULLIF(s.licensed_count, 0), c.purchased)) * c.unit_cost_php AS exposure_php
FROM view_software_v1 s
JOIN (VALUES
        ('Datto RMM',             800, 1200),
        ('Sophos Endpoint Agent', 800, 1500)
     ) AS c(title, purchased, unit_cost_php)
  ON lower(s.name) = lower(c.title)
WHERE s.discovered_count > COALESCE(NULLIF(s.licensed_count, 0), c.purchased)
ORDER BY exposure_php DESC








/*
 - Name: S4 Prohibited and Candidate Software by Device
 - Purpose: Lists installs of software marked Prohibited, plus candidates matched by name
   (remote-access, torrent, personal cloud sync, proxy tools) for security review.
 - Chart: Big Number (COUNT(*)) + Table
 - Device42 version: v19.11
 - Date Created: 11/10/26
 - Changes: 11/10/26 Added name-based candidates while no titles are marked Prohibited
*/
SELECT d.name AS device,
       s.name AS software,
       CASE WHEN lower(s.software_type) = 'prohibited' THEN 'Marked prohibited'
            ELSE 'Candidate (by name)' END AS status
FROM view_softwareinuse_v1 siu
JOIN view_software_v1 s ON s.software_pk = siu.software_fk
JOIN view_device_v2 d   ON d.device_pk = siu.device_fk
WHERE lower(s.software_type) = 'prohibited'
   OR s.name ~* '(torrent|anydesk|teamviewer|ultraviewer|rustdesk|tor browser|dropbox|google drive|megasync|cheat engine|psiphon|hotspot shield|ultrasurf)'
ORDER BY 3, 2, 1










/*
 - Name: S5 Software Installs by Days Undetected
 - Purpose: Groups software-in-use records by days since the install was last detected, to find
   stale records that inflate license counts.
 - Chart: Bar (x-axis last_detected, metric SUM(installs))
 - Device42 version: v19.11
 - Date Created: 11/10/26
 - Changes: 11/10/26 Replaced 30-day cut-off with buckets so the chart returns rows early in the project
*/
SELECT CASE
         WHEN siu.days_undetected IS NULL THEN '0 Not tracked'
         WHEN siu.days_undetected <= 1    THEN '1 Seen in latest scan'
         WHEN siu.days_undetected <= 7    THEN '2 2-7 days'
         WHEN siu.days_undetected <= 14   THEN '3 8-14 days'
         WHEN siu.days_undetected <= 30   THEN '4 15-30 days'
         ELSE '5 Over 30 days'
       END AS last_detected,
       COUNT(*) AS installs
FROM view_softwareinuse_v1 siu
GROUP BY 1
ORDER BY 1




































































































































