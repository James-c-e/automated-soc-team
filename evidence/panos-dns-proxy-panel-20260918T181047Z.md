# PAN-OS DNS-proxy panel change

- Dashboard: `panos-threat-monitoring-readonly`
- Panel ID: `panos-dns-proxy-4`
- Panel title: `PAN-OS blocked DNS-proxy / spyware activity`
- Pre-change dashboard JSON capture: `/tmp/panos_dashboard_pre.json`
- Pre-change SHA-256: `572c6c3de53e0c5a7d1b91b35ff8ef2ec42585608aeb23ca62630ee4549b1148`
- Elasticsearch operation: `POST /.ds-logs-panw.panos-palo-*/_search`
- Search result: HTTP 200, `took: 15 ms`, `hits.total: 1,825` for the preceding 24 hours
- Evidence: `spyware_detected` + `THREAT` + `dns-proxy`, with `drop`/`drop-packet` actions
- Dashboard write: `PUT /api/saved_objects/dashboard/panos-threat-monitoring-readonly` with `kbn-xsrf: true`, HTTP 200
- Final read-back: `GET /api/saved_objects/dashboard/panos-threat-monitoring-readonly`, HTTP 200
- Final updated time: `2026-09-18T18:10:47.062Z`
- Final panel count: 4

## Rollback

Restore the captured pre-change dashboard attributes and references from
`/tmp/panos_dashboard_pre.json` with the same PUT endpoint and `kbn-xsrf: true`,
then GET the dashboard and confirm the panel count and JSON match the recorded
pre-change SHA-256.
