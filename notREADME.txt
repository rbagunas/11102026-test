# 11102026-test
11102026-test


# C1 · Devices by type and subtype
SELECT COALESCE(d.type, 'unknown') AS device_type,
       COALESCE(NULLIF(d.physicalsubtype, ''), NULLIF(d.virtualsubtype, ''), 'not set') AS device_subtype,
       COUNT(*) AS devices
FROM view_device_v2 d
WHERE d.in_service = true
GROUP BY 1, 2
ORDER BY 1, 3 DESC
Chart it as a sunburst (type in the inner ring, subtype outside) so a switch reads as physical › switch, not just physical. Rows showing not set are hardware models still waiting for a subtype (see "Classifying laptops and desktops automatically").


# C2 · Idle hardware: no update for 30+ days
SELECT d.name,
       COALESCE(NULLIF(d.physicalsubtype, ''), 'not set') AS subtype,
       d.last_edited::date AS last_updated,
       CURRENT_DATE - d.last_edited::date AS days_unchanged
FROM view_device_v2 d
WHERE d.type = 'physical'
  AND d.in_service = true
  AND d.last_edited < CURRENT_DATE - 30
ORDER BY days_unchanged DESC
Device42's own examples use last_edited as the last-discovered date. It also moves when someone edits a record by hand, so treat the list as candidates to check. For the CFO tile, count the rows and multiply by the replacement cost (₱45,000 per laptop in the payback model).


# C3 · Devices on end-of-support operating systems
SELECT d.os_name, d.os_version, COUNT(*) AS devices
FROM view_device_v1 d
WHERE d.os_name ~* '(windows (xp|vista|7|8)|windows 10|server 2003|server 2008|server 2012)'
GROUP BY 1, 2
ORDER BY 3 DESC
Once OS end-of-life dates are entered, view_oseoleos_v1 (columns os_fk, version, eol, eos) can replace the name pattern with real dates.


# C4 · Certificates by expiry window
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


# C5 · Hardware by warranty status
SELECT CASE
         WHEN w.warranty_end IS NULL              THEN 'No warranty record'
         WHEN w.warranty_end < CURRENT_DATE       THEN 'Expired'
         WHEN w.warranty_end <= CURRENT_DATE + 90 THEN 'Expires within 90 days'
         ELSE 'Active'
       END AS warranty_status,
       COUNT(*) AS devices
FROM (
  SELECT d.device_pk, MAX(pli.end_date) AS warranty_end
  FROM view_device_v2 d
  LEFT JOIN view_purchaselineitems_to_devices_v1 ptd ON ptd.device_fk = d.device_pk
  LEFT JOIN view_purchaselineitem_v1 pli ON pli.purchaselineitem_pk = ptd.purchaselineitem_fk
  WHERE d.type = 'physical'
    AND d.in_service = true
  GROUP BY d.device_pk
) w
GROUP BY 1
ORDER BY 2 DESC
It takes the latest end date per device, so a device with several contracts counts once. To count warranty lines only, add AND pli.contract_type_name ILIKE '%warrant%' to the last join, after checking the contract type names warranty autodiscovery writes.


# S1 · Licensed vs. discovered, top 10 managed titles
SELECT s.name AS title,
       v.name AS vendor,
       lm.name AS license_model,
       s.licensed_count,
       s.discovered_count
FROM view_software_v1 s
LEFT JOIN view_vendor_v1 v ON v.vendor_pk = s.vendor_fk
LEFT JOIN view_softwarelicensemodel_v1 lm ON lm.softwarelicensemodel_pk = s.softwarelicensemodel_fk
WHERE lower(s.software_type) = 'managed'
ORDER BY s.discovered_count DESC
LIMIT 10
licensed_count stays at 0 until purchased quantities or license keys are entered on the software component (SLM step 3). Chart it as grouped bars, licensed beside discovered.


# S2 · Shelfware: licenses Insular Life can stop paying for
SELECT s.name AS title,
       s.licensed_count - s.discovered_count AS unused_licenses,
       p.unit_cost_php,
       (s.licensed_count - s.discovered_count) * p.unit_cost_php AS reclaimable_php
FROM view_software_v1 s
JOIN (VALUES
        ('Adobe Acrobat', 14000),
        ('Microsoft Visio', 10500),
        ('Microsoft Project', 10500)
     ) AS p(title_prefix, unit_cost_php)
  ON s.name ILIKE p.title_prefix || '%'
WHERE lower(s.software_type) = 'managed'
  AND s.licensed_count > s.discovered_count
ORDER BY reclaimable_php DESC
The VALUES list is the price table: one line per title prefix with its annual cost per license in pesos. Replace the placeholder prices with Insular Life's contract prices and add a line per priced title. The CFO tile is a SUM metric on reclaimable_php.


# S3 · Audit exposure: installed beyond what was purchased
SELECT s.name AS title,
       s.discovered_count - s.licensed_count AS shortfall,
       p.unit_cost_php,
       (s.discovered_count - s.licensed_count) * p.unit_cost_php AS exposure_php
FROM view_software_v1 s
LEFT JOIN (VALUES
        ('Adobe Acrobat', 14000),
        ('Microsoft Visio', 10500),
        ('Microsoft Project', 10500)
     ) AS p(title_prefix, unit_cost_php)
  ON s.name ILIKE p.title_prefix || '%'
WHERE lower(s.software_type) = 'managed'
  AND s.discovered_count > s.licensed_count
ORDER BY shortfall DESC
Titles with no price still appear, with an empty peso column, so no shortfall is hidden. Use the same price list as S2.


# S4 · Prohibited software by device
SELECT d.name AS device,
       s.name AS software,
       v.name AS vendor
FROM view_softwareinuse_v1 siu
JOIN view_software_v1 s ON s.software_pk = siu.software_fk
JOIN view_device_v2 d   ON d.device_pk = siu.device_fk
LEFT JOIN view_vendor_v1 v ON v.vendor_pk = s.vendor_fk
WHERE lower(s.software_type) = 'prohibited'
ORDER BY 2, 1


# S5 · Stale software records
SELECT s.name AS title,
       COUNT(*) AS stale_installs
FROM view_softwareinuse_v1 siu
JOIN view_software_v1 s ON s.software_pk = siu.software_fk
WHERE siu.days_undetected > 30
GROUP BY s.name
ORDER BY stale_installs DESC
LIMIT 15






############################################################################


C1 · Devices by type and subtype
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




C2 · Devices by days since last update
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







C3 · Every operating system with its support status
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




C4 · Certificates by expiry window
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






C5 · Hardware by warranty status
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







S1 · Licensed vs. discovered, top 10 managed titles
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







S2 · Shelfware: licenses Insular Life can stop paying for
SELECT s.name AS title,
       s.discovered_count AS installed,
       COALESCE(NULLIF(s.licensed_count, 0), c.purchased) AS purchased,
       COALESCE(NULLIF(s.licensed_count, 0), c.purchased) - s.discovered_count AS unused_licenses,
       c.unit_cost_php,
       (COALESCE(NULLIF(s.licensed_count, 0), c.purchased) - s.discovered_count) * c.unit_cost_php AS reclaimable_php
FROM view_software_v1 s
JOIN (VALUES
        ('Datto RMM',             800, 1200),   -- example figures: replace with the contract
        ('Sophos Endpoint Agent', 800, 1500)    -- example figures: replace with the contract
     ) AS c(title, purchased, unit_cost_php)
  ON lower(s.name) = lower(c.title)
WHERE COALESCE(NULLIF(s.licensed_count, 0), c.purchased) > s.discovered_count
ORDER BY reclaimable_php DESC






S3 · Audit exposure: installed beyond what was purchased
SELECT s.name AS title,
       s.discovered_count AS installed,
       COALESCE(NULLIF(s.licensed_count, 0), c.purchased) AS purchased,
       s.discovered_count - COALESCE(NULLIF(s.licensed_count, 0), c.purchased) AS shortfall,
       c.unit_cost_php,
       (s.discovered_count - COALESCE(NULLIF(s.licensed_count, 0), c.purchased)) * c.unit_cost_php AS exposure_php
FROM view_software_v1 s
JOIN (VALUES
        ('Datto RMM',             800, 1200),   -- same list as S2
        ('Sophos Endpoint Agent', 800, 1500)
     ) AS c(title, purchased, unit_cost_php)
  ON lower(s.name) = lower(c.title)
WHERE s.discovered_count > COALESCE(NULLIF(s.licensed_count, 0), c.purchased)
ORDER BY exposure_php DESC







S4 · Prohibited and candidate software by device
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







S5 · Software installs by days undetected
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








D42-SuperAdmin
D42-Discovery-Admin
D42-Network-Eng
D42-Server-Admin
D42-Cloud-Ops
D42-Storage-DBA
D42-App-Owner
D42-Security-PKI
D42-EUC-Support
D42-SAM-Manager
D42-Auditor-RO
D42-API-Integration
























































































































