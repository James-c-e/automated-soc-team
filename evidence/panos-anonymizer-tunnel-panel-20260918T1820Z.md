# PAN-OS anonymizer / unusual tunnel dashboard panel

- Change type: Kibana saved-object dashboard panel only
- Dashboard: `panos-threat-monitoring-readonly`
- Added panel ID: `panos-anonymizer-tunnel-6`
- Added title: `PAN-OS anonymizer / unusual tunnel activity`
- Evidence timestamp: 2026-09-18 18:20 UTC

## Pre-change state

- Read-back endpoint: `GET /api/saved_objects/dashboard/panos-threat-monitoring-readonly`
- HTTP status: 200
- `updated_at`: `2026-09-18T18:15:02.263Z`
- Panel count: 5
- Preserved object: `evidence/panos-dns-malware-panel-post-20260918T181449Z.json`
- Preserved object SHA-256: `e46cc9a2b2df054e7c879560d5b0d66d57dee0155191774fe662069787151acd`

## Platform evidence

- Read-only Elasticsearch search against `/.internal.alerts-security.alerts-default-*/_search`
- Time range: preceding 24 hours
- Rule: `PAN-OS Anonymizer or Unusual Tunnel Application`
- Matching alerts: 2,113
- Search response: HTTP 200, `took: 16 ms`

## Change and validation

- Operation: `PUT /api/saved_objects/dashboard/panos-threat-monitoring-readonly`
- Header: `kbn-xsrf: true`
- PUT HTTP status: 200
- Post-change read-back: HTTP 200
- Post-change `updated_at`: `2026-09-18T18:20:07.431Z`
- Post-change panel count: 6
- Read-back confirmed panel ID, title, grid position, rule name, KQL, and investigation fields.
- No detection rules were modified.

## Rollback

Restore the preserved five-panel `attributes` and `references` from the pre-change object with `PUT /api/saved_objects/dashboard/panos-threat-monitoring-readonly`, include `kbn-xsrf: true`, then read the object back and confirm panel count 5 and the pre-change `updated_at` lineage. No rule rollback is required.
