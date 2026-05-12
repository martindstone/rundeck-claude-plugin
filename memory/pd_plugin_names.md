---
name: PagerDuty Plugin Names
description: Exact type name identifiers for all PagerDuty WorkflowStep plugins in this Rundeck instance
type: reference
---

Queried from `$RD_URL/api/50/plugin/list` — PagerDuty Plugins bundle v5.17.0-20251103.

WorkflowStep plugins (use `type: <name>` + `nodeStep: false` in YAML):
- `pd-note-step` — PagerDuty / Incident / Note (api_token, email, incident_id, note)
- `pd-update-incident-step` — PagerDuty / Incident / Update (api_token, email, incident_id, status, resolution, escalation_level, assignees)
- `pd-status-update-step` — PagerDuty / Status / Update (api_token, email, incident_id, message*)
- `pd-escalate-incident-step` — PagerDuty / Incident / Escalate (api_token, email, incident_id*, escalation_level)
- `pagerduty-get-incident` — PagerDuty / Incident / Get (apiKey, pd-id*)
- `pd-sent-event-step` — PagerDuty / Event / Send (service_key, event_action*, payload_summary*, payload_source*)
- `pagerduty-send-change-event` — PagerDuty / Change Event / Send (routingKey*, summary*, source, apiKey)
- `pagerduty-add-additional-responders` — PagerDuty / Incident / Add Responders (apiKey, pd-id*, requester*, message*)
- `pagerduty-update-escalation` — PagerDuty / Incident / Update Escalation (apiKey, pd-id*, escalation-policy*)
- `pd-run-response-play-step` — PagerDuty / Incident / Run Response Play (api_token, email, incident_id*, response_play_id*)
- `pd-start-incident-workflow-step` — PagerDuty / Incident / Start Incident Workflow (api_token, incident_id*, incident_workflow_id*)
- `pagerduty-create-user` — PagerDuty / User / Create (apiKey, name*, email, role)
- `pagerduty-get-user` — PagerDuty / User / Get (apiKey, pd-id*)
- `pagerduty-update-user` — PagerDuty / User / Update (apiKey, pd-id*, name*)
- `pagerduty-delete-user` — PagerDuty / User / Delete (apiKey, pd-id*)
- `pagerduty-list-users` — PagerDuty / Users / List (apiKey, teams, query)

* = required field. Older pd-* plugins use `api_token`; newer pagerduty-* plugins use `apiKey`. Both accept a key storage path string.
