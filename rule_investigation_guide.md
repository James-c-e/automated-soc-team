# Security Onion Rule Investigation Guide

This guide is for SOC Analysts investigating and tuning network-threat rules in
Security Onion through Kibana and Elasticsearch. It covers evidence collection,
safe rule changes, validation, and rollback. It does not cover deployment,
devices, sensors, ingestion, integrations, topology, or platform services.

## 1. Access and handling requirements

Use the API key from `.env` exactly as stored. Do not decode, print, log,
commit, or place the key in evidence files. Kibana uses HTTP on port 5601 and
Elasticsearch uses HTTPS on port 9200.

Set credentials in the shell without writing them to the repository:

```bash
export SO_KIBANA_API_KEY='(value copied exactly from .env)'
export SO_ES_API_KEY="$SO_KIBANA_API_KEY"
```

Confirm connectivity before investigating:

```bash
curl -sS -o /tmp/kibana-status.json -w '%{http_code}\n' \
  -H "Authorization: ApiKey $SO_KIBANA_API_KEY" \
  http://${SO_SECURITY_ONION_HOST}:5601/api/status

curl -sS -o /tmp/es-health.json -w '%{http_code}\n' \
  -H "Authorization: ApiKey $SO_ES_API_KEY" \
  https://${SO_SECURITY_ONION_HOST}:9200/_cluster/health
```

Record the UTC timestamp, endpoint, HTTP status, result count, and relevant
identifiers. Never record credentials or unrestricted event payloads.

## 2. Rule inventory

Retrieve the complete rule inventory from Kibana:

```bash
curl -sS \
  -H "Authorization: ApiKey $SO_KIBANA_API_KEY" \
  "http://${SO_SECURITY_ONION_HOST}:5601/api/detection_engine/rules/_find?per_page=1000"
```

For each rule, record its ID, name, enabled state, query, index patterns,
interval, severity, risk score, suppression or threshold, actions, exceptions,
MITRE metadata, and revision. Separate enabled rules from disabled rules and
note duplicate or overlapping coverage.

Before searching events, discover indices and mappings:

```bash
curl -sS -H "Authorization: ApiKey $SO_ES_API_KEY" \
  "https://${SO_SECURITY_ONION_HOST}:9200/_cat/indices?format=json"

curl -sS -H "Authorization: ApiKey $SO_ES_API_KEY" \
  "https://${SO_SECURITY_ONION_HOST}:9200/logs-*/_mapping"
```

Use only fields confirmed by the mapping. Prefer an explicit index or narrow
pattern rather than searching every index.

## 3. Alert-rate review

Measure both recent activity and the longer baseline. Use the alert index that
is present in the deployment, commonly `.internal.alerts-security.alerts-*`:

```json
{
  "size": 0,
  "track_total_hits": true,
  "query": {
    "range": { "@timestamp": { "gte": "now-1h", "lte": "now" } }
  },
  "aggs": {
    "by_rule": {
      "terms": { "field": "kibana.alert.rule.name", "size": 100 }
    }
  }
}
```

Repeat with `now-24h` and record total alerts, per-rule counts, the first and
last alert times, and the proportion of alerts generated before a recent rule
revision. A rolling-hour budget of 10 alerts is the operational maximum. Treat
the budget as a total across rules, not as 10 alerts per rule.

When historical alerts remain inside the rolling window after a change, report
that explicitly. Do not claim the budget is met until a complete post-change
window has elapsed.

## 4. Representative alert and source-event searches

For each active rule, retrieve a small sample of alerts and the corresponding
source events. Keep the time range bounded and return only useful fields:

```json
{
  "size": 25,
  "_source": [
    "@timestamp", "event.dataset", "event.kind", "event.action",
    "event.outcome", "source.ip", "destination.ip", "destination.port",
    "network.transport", "panw.panos.type", "panw.panos.action",
    "panw.panos.ruleset", "panw.panos.application",
    "panw.panos.threat_category", "panw.panos.threat_name"
  ],
  "sort": [{ "@timestamp": { "order": "desc" } }],
  "query": {
    "bool": {
      "filter": [
        { "range": { "@timestamp": { "gte": "now-1h", "lte": "now" } } },
        { "term": { "kibana.alert.rule.uuid": "<rule-id>" } }
      ]
    }
  }
}
```

Then search `logs-*` with the rule query and the same bounded time range. If a
rule is based on PAN-OS data, confirm the dataset and fields before using KQL,
for example:

```text
event.dataset:"panw.panos" and panw.panos.type:"THREAT"
```

Compare alert documents with source events to identify duplicate firing,
blocked repetition, successful activity, allowed benign services, and genuine
threat indicators. Check source/destination pairs, rulesets, applications,
actions, outcomes, risk levels, threat categories, and recurrence.

## 5. Threat classification

Classify each alert using evidence rather than its rule name:

- **Confirmed threat:** malicious verdict, exploit/spyware evidence, high-risk
  blocked behavior, or corroborating indicators.
- **Suspicious:** allowed or incomplete activity with meaningful risk, unusual
  destination, tunneling, remote access, or repeated behavior.
- **Benign or expected:** sanctioned service, known administration, normal
  resolver/VPN behavior, or a blocked repetitive probe with no escalation value.
- **Insufficient evidence:** fields or behavioral context are missing.

For each classification, document the query, time window, sample size, key
fields, alternative explanation, confidence, and false-positive impact.

## 6. MITRE ATT&CK mapping review

Every rule should have a nonempty, structurally valid `threat` metadata array.
Review that the technique describes the behavior actually detected, not merely
the severity or data source. Check tactic and technique IDs, names, references,
and sub-techniques where applicable. Do not use a mapping as proof that a rule
provides behavioral coverage.

Useful network-focused examples include T1046 for network service discovery,
T1021 for remote services, T1090 for proxy behavior, T1572 for protocol
tunneling, T1071.004 for DNS, T1568.002 for dynamic resolution, and T1219 for
remote access tools. Remove or correct mappings that cannot be supported by the
rule query and observed fields.

## 7. Safe tuning and rule creation

Only tune a rule when the evidence identifies a concrete source of noise or a
defensible missing behavior. Preserve confirmed threat coverage. Prefer, in
order:

1. Narrow the query to the relevant event type, action, outcome, risk level,
   application, port, or network direction.
2. Add a narrowly scoped exception for a proven benign tuple, service, or
   destination.
3. Add thresholding or suppression for repeated equivalent events, using a
   grouping key that retains distinct sources and destinations.
4. Create a new rule only when the fields, baseline, and expected alert rate
   are sufficient to support it.

Before a write, save the complete rule JSON and state the intended change,
expected alert-rate effect, false-negative risk, and rollback payload. Change
only the necessary fields. Do not alter indices, ingestion, integrations,
deployment, or platform services.

For a Kibana rule update, use the detection-engine rule endpoint with the rule
ID in the JSON body and the required CSRF header. Preserve all unrelated
fields:

```bash
curl -sS -X PUT \
  -H "Authorization: ApiKey $SO_KIBANA_API_KEY" \
  -H 'Content-Type: application/json' \
  -H 'kbn-xsrf: true' \
  "http://${SO_SECURITY_ONION_HOST}:5601/api/detection_engine/rules" \
  --data-binary @rule-update.json
```

Do not treat HTTP 200 alone as proof of success. A failed or ambiguous write
is a blocker until resolved.

## 8. Pre/post evidence and read-back

For every change, retain separate pre-change and post-change evidence outside
the secret material. Include:

- UTC timestamps and API endpoints.
- HTTP status and response error, if any.
- Rule ID, revision, enabled state, exact query, schedule, threshold,
  suppression, severity, risk score, actions, exceptions, and MITRE metadata.
- The before/after alert counts for 15 minutes, 1 hour, and 24 hours when
  available.
- The source-event validation query and hit count.

Immediately GET the rule after writing it and compare the returned object with
the intended payload. Confirm that query, schedule, threshold, suppression,
actions, severity, risk score, indices, and enabled state were not changed
unless explicitly authorized.

## 9. Behavioral validation

Run the rule or wait for its scheduled execution, then verify execution status
through the rule API. Search source events and alert documents after the write.
Use a post-change interval that excludes pre-change history when assessing new
alerts. Confirm both sides of the change:

- The noisy or benign pattern no longer generates unwanted alerts.
- Confirmed threat examples still match and alert.

Record zero matches as a validation result, not as proof that the rule is
effective; distinguish “no matching data observed” from “tested and detected.”
Recheck the total rolling-hour alert count only after old alerts age out.

## 10. Rollback

Keep the complete pre-change rule object and the exact write timestamp. Roll
back by restoring only the previous rule object through the same scoped Kibana
endpoint, then perform GET read-back and behavioral validation again. Confirm
that the prior query, schedule, threshold, suppression, actions, severity, risk
score, indices, enabled state, and MITRE metadata are restored.

If a rollback cannot be read back or validated, report the rule as unresolved
and escalate it to the SOC manager. Never delete alert history to make a rate
appear lower.

## 11. Investigation report template

Use this structure for each rule:

```text
Rule and ID:
Investigation UTC window:
API endpoints and HTTP statuses:
Alert counts: 15m / 1h / 24h:
Representative alert identifiers:
Representative source-event identifiers:
Threat classification and confidence:
MITRE mapping review:
Change proposed or applied:
Expected rate and false-positive effect:
Pre-change evidence:
Post-change read-back:
Behavioral validation:
Rollback artifact and procedure:
Open questions or insufficient data:
```

The final cycle report must identify completed work, evidence, validation
status, remaining risk, rollback location, and the next recommended action.
