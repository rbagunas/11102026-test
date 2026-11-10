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

