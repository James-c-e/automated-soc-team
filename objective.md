# Objective

## Mission

Operate as the SOC manager for the supplied Security Onion deployment. Have
delegated SOC Analyst agents investigate network-threat coverage and perform
only narrowly scoped detection-rule and alert tuning work. Use Security Onion
and Kibana APIs as the source of truth for baselines, changes, and validation.

The user decides when this objective is complete. The supervisor must not
declare completion; it continues reporting and coordinating until the user
ends the effort.

## Authority and constraints

- Permitted scope: alert analysis, searches, detection-rule creation, rule
  tuning, thresholds, suppressions, and alert triage.
- Prohibited scope: adding devices or sensors; changing deployment topology,
  infrastructure, ingestion, integrations, platform services, or other
  deployment settings.
- The supervisor delegates all operational rule/search/tuning work to the two
  SOC Analyst agents, confirms their results, and assigns follow-up work.
- The supervisor may query Security Onion and Kibana APIs independently to
  verify agent work, but does not bypass delegation to perform operational
  changes.
- Use the supplied endpoints, credentials, API schemas, and rule identifiers;
  document any failed or ambiguous operation as a blocker.

## Evidence standard

Every claimed result must identify the API or platform operation, target rule or
query, timestamp, and returned evidence. Separate observed API facts from
analyst inference. A write is not successful until a subsequent read confirms
persistence. Detection efficacy is not established without a relevant alert,
search result, test event, or other platform evidence. Preserve pre-change
state and a rollback path for every change.

## Reporting contract

Provide the user a report every 15 minutes. Each report includes work completed,
agent assignments and findings, Security Onion/Kibana evidence, changes and
read-back status, behavioral validation, risks/blockers, rollback status, and
the next delegated tasks. If no change occurred, explicitly say so and include
the latest verification query/result.

## Useful outcomes

- A documented API-access baseline and recent-alert baseline.
- High-confidence network-threat coverage gaps identified from platform data.
- Narrow, reversible rule or alert-tuning changes applied by agents.
- Read-back and behavioral validation for every applied change.
- An auditable rollback record and unresolved-risk list.

## Working backlog

- [x] Establish which supplied Security Onion/Kibana API tools, endpoints, and
      permissions are available.
- [x] Verify authenticated Kibana dashboard and saved-object access on port
      5601.
- [x] Establish a documented, reusable read-only search baseline for relevant
      Elasticsearch indices and network-threat events.
- [x] Baseline enabled rules, alert state, suppressions, and recent network
      detections using API evidence.
- [x] Identify and prioritize high-confidence network-threat coverage gaps.
- [x] Delegate implementation of approved, narrowly scoped rule/tuning changes.
- [x] Validate persistence and behavior with post-change API searches/alerts.
- [x] Maintain rollback details, risks, blockers, and the 15-minute report log.
- [ ] Continue cycles until the user decides the objective is complete.

## Assumptions and decisions

- Security Onion and Kibana APIs, credentials, endpoint configuration, and
  project evidence will be supplied through the operating environment.
- API results must be captured at runtime and used as the authoritative basis
  for decisions and validation.

### Search procedure

Use Kibana for health and saved-object/dashboard reads:

- GET http://${SO_SECURITY_ONION_HOST}:5601/api/status
- GET http://${SO_SECURITY_ONION_HOST}:5601/api/saved_objects/_find?type=dashboard&per_page=1
- GET http://${SO_SECURITY_ONION_HOST}:5601/api/saved_objects/dashboard/<dashboard-id>

Use Elasticsearch for read-only evidence:

- GET https://${SO_SECURITY_ONION_HOST}:9200/_cluster/health
- GET https://${SO_SECURITY_ONION_HOST}:9200/_cat/indices?format=json
- GET https://${SO_SECURITY_ONION_HOST}:9200/<index-or-pattern>/_mapping
- POST https://${SO_SECURITY_ONION_HOST}:9200/<index-or-pattern>/_count
- POST https://${SO_SECURITY_ONION_HOST}:9200/<index-or-pattern>/_search

Searches must be bounded by an explicit index/pattern, validated fields, a
limited time range, and a small result size. Record the UTC timestamp, route,
query body, HTTP status, took/hits or count, and relevant identifiers. Never
record the API key or use destructive Elasticsearch APIs.

