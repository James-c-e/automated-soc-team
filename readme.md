# Security Onion SOC Manager Workflow

This repository defines an objective-driven workflow. The main thread is
the SOC manager; its two delegated workers are SOC Analysts. There is no local
application runner or custom orchestration framework.

## Operating model

For every work cycle, the manager reads OBJECTIVE.md, assigns two focused and
independent tasks, waits for both reports, reconciles them, verifies results
through the supplied Security Onion and Kibana APIs, confirms successful work
to the agents, and assigns the next tasks. All operational searches, rule
changes, and alert tuning are delegated to the agents.

The manager reports to the user every 15 minutes with completed work, agent
findings, API evidence, changes, validation/read-back status, risks, blockers,
rollback status, and next tasks. The manager does not decide that the overall
objective is complete; only the user does.

## Scope

Allowed work is analysis and alert tuning: searches, detection rules,
thresholds, suppressions, and alert queries. Adding devices or sensors and
changing deployment, infrastructure, ingestion, integrations, topology, or
platform services are out of scope.

Every change requires pre-change evidence, a narrow reversible implementation,
post-change read-back, and behavioral validation. Use the supplied API
endpoints, credentials, schemas, and project evidence; record failed or
ambiguous operations as blockers rather than claiming success.

## Example first cycle

1. Analyst: baseline recent alerts and enabled rules through the supplied APIs;
   identify one high-confidence network-threat coverage or false-positive
   tuning opportunity with evidence.
2. Responder: independently inspect the relevant rule/configuration path and
   propose the smallest safe implementation, validation query, and rollback.
3. Manager: reconcile both reports, delegate the approved implementation, then
   verify persistence and behavior via API read-back/search evidence.

Do not add Python, Node.js, or another custom orchestration layer. See
OBJECTIVE.md for the live backlog and `working history.md` for completed
Objective cycles.

## Kibana access

Kibana is served over HTTP on port 5601:

    http://${SO_SECURITY_ONION_HOST}:5601

Use the API key from .env exactly as stored; do not base64-decode or
transform it before sending the header. Never print or commit the key.

Read-only health and dashboard checks:

    curl -H "Authorization: ApiKey $SO_KIBANA_API_KEY" \
      http://${SO_SECURITY_ONION_HOST}:5601/api/status

    curl -H "Authorization: ApiKey $SO_KIBANA_API_KEY" \
      "http://${SO_SECURITY_ONION_HOST}:5601/api/saved_objects/_find?type=dashboard&per_page=1"

    curl -H "Authorization: ApiKey $SO_KIBANA_API_KEY" \
      "http://${SO_SECURITY_ONION_HOST}:5601/api/saved_objects/dashboard/<dashboard-id>"

Port 5601 is not configured for HTTPS. A successful dashboard read must include
HTTP 200 and returned saved-object data such as the dashboard ID, type, and
attributes. Keep saved-object checks read-only; do not import, export, create,
update, or delete objects without an explicitly delegated task.

## Read-only Elasticsearch searches

Elasticsearch is available over HTTPS on port 9200:

    https://${SO_SECURITY_ONION_HOST}:9200

Use the same API key exactly as stored:

    curl -H "Authorization: ApiKey $SO_ES_API_KEY" \
      https://${SO_SECURITY_ONION_HOST}:9200/_cluster/health

Discover indices and mappings before searching:

    curl -H "Authorization: ApiKey $SO_ES_API_KEY" \
      "https://${SO_SECURITY_ONION_HOST}:9200/_cat/indices?format=json"

    curl -H "Authorization: ApiKey $SO_ES_API_KEY" \
      "https://${SO_SECURITY_ONION_HOST}:9200/<index-or-pattern>/_mapping"

Bounded event count:

    curl -H "Authorization: ApiKey $SO_ES_API_KEY" \
      -H "Content-Type: application/json" -X POST \
      "https://${SO_SECURITY_ONION_HOST}:9200/<index-or-pattern>/_count" \
      --data '{"query":{"bool":{"filter":[{"range":{"@timestamp":{"gte":"now-15m","lte":"now"}}}]}}}'

Bounded event search:

    curl -H "Authorization: ApiKey $SO_ES_API_KEY" \
      -H "Content-Type: application/json" -X POST \
      "https://${SO_SECURITY_ONION_HOST}:9200/<index-or-pattern>/_search" \
      --data '{"size":25,"track_total_hits":true,"sort":[{"@timestamp":{"order":"desc"}}],"query":{"bool":{"filter":[{"range":{"@timestamp":{"gte":"now-15m","lte":"now"}}}]}}}'

Use an explicit index, short time range, small result size, and validated
fields. Do not use bulk, delete-by-query, update, or settings APIs. Record the
UTC timestamp, endpoint, index, query, HTTP status, took/hits or count, and
relevant rule or event identifiers without recording credentials.
