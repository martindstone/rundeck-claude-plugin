---
name: pagerduty
description: interact with the PagerDuty API to manage incidents, users, and other resources.
---

Use this skill when the user wants to interact with PagerDuty to manage incidents, users, and other resources.

Preparation:
  1. Check for the presence of the following environment variables: `PAGERDUTY_API_TOKEN`. If it is missing, ask the user to provide it before proceeding.
  2. Check that `/usr/bin/env node -v` returns a valid Node.js version. If it does not, ask the user to install Node.js and ensure that the `node` executable is in the PATH.
  3. Check that `pd-api` can successfully make a test API call (e.g., `pd-api request GET /abilities`) to verify that the PagerDuty API token is valid and has the necessary permissions. If the call fails, ask the user to verify their API token and permissions.

Rules:
  - Don't output any API tokens (e.g., `PAGERDUTY_API_TOKEN`) in the conversation. If you need to report that the variable is set, just say (set) without showing the value.
  - Always ask the user before performing any mutating actions (e.g., creating or updating incidents). For read-only actions (e.g., listing incidents), you can proceed without explicit approval.
  - Use the `pd-api` CLI tool to interact with the PagerDuty API. See the reference at https://developer.pagerduty.com/api-reference/e65c5833eeb07-pager-duty-api for available endpoints and parameters.
  - When asked to create or update an incident, generate a summary of the changes and show it to the user before asking for approval to proceed.

Helpers:
  - this claude code plugin includes the `bin/pd-api` CLI tool for making API requests to PagerDuty.
  - PagerDuty API Reference: https://developer.pagerduty.com/api-reference/e65c5833eeb07-pager-duty-api
  - also see `skills/pagerduty/REST-openapiv3-spec.json` for a machine-readable version of the API spec that you can use to find endpoints and parameters.

pd-api helper usage:
  pd-api request METHOD ENDPOINT [PARAMS_JSON] [BODY_JSON] [HEADERS_JSON]
  pd-api fetch   METHOD ENDPOINT [PARAMS_JSON] [BODY_JSON] [HEADERS_JSON] [FETCH_LIMIT]

pd-api helper examples:
  pd-api request GET /users '{"limit":10}'
  pd-api fetch GET /users '{"query":"alice"}'
  pd-api request POST /incidents '{}' '{"incident":{"type":"incident","title":"Test"}}'
