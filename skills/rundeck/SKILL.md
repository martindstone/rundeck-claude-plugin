---
name: rundeck
description: Generate, import, and execute Rundeck job YAML using Rundeck CLI.
---

Use this skill when the user wants to create or run Rundeck jobs.

Preparation:
  1. Ensure `rd` exists in the system PATH. If it is not, then ask the user to install the Rundeck CLI, give them the URL for the install instructions - https://docs.rundeck.com/docs/learning/howto/learn-rd-cli.html - and stop.
  2. Check for the presence of the following environment variables: `RD_URL`, `RD_TOKEN`, and `RD_PROJECT`. If any are missing, ask the user to provide them before proceeding.

Rules:
  - Don't output any API tokens (e.g., `RD_TOKEN`) in the conversation. If you need to report that the variable is set, just say (set) without showing the value.
  - Use the `rd` CLI tool to interact with Rundeck. See the reference below for available commands and options.
  - When asked to create a job, generate YAML first and show it to the user.
  - Do not import or execute a job unless the user explicitly approves.
  - Prefer safe/read-only commands unless the user explicitly requests a mutating action.
  - Use environment variables for `rd` CLI configuration: `RD_URL`, `RD_TOKEN`, `RD_PROJECT`.
  - For any Rundeck-related request, determine the appropriate `rd` CLI command to accomplish the task and execute it. If the request is ambiguous, ask the user for clarification before proceeding. If there's no `rd` command available, use the Rundeck REST API directly.
  - IMPORTANT: When building job workflows that interact with external systems (PagerDuty, ServiceNow, Slack, AWS, etc.), ALWAYS check for available Rundeck plugins before falling back to script steps or HTTP workflow steps. Never use a script or HTTP step to call an API when a native Rundeck plugin exists for that purpose.
  - When no native plugin exists for a required API call, prefer HTTP workflow steps over script steps. Script steps introduce runtime dependencies (Python, jq, etc.) that may not be available on the runner.
  - When adding workflow steps for PagerDuty operations, use the native PagerDuty plugin steps documented below. Do NOT use script steps, command steps, or HTTP workflow steps to call the PagerDuty API.
  - IMPORTANT: Before writing new job YAML from scratch, download an existing job from the project as a format reference: `rd jobs list -p $RD_PROJECT -i <id> -f /tmp/ref.yaml -F yaml`. This avoids structural mistakes.
  - When you need to verify a PagerDuty result (e.g., check that a note was posted), use the `bin/pd-api` helper (e.g., `bin/pd-api request GET /incidents/<id>/notes`) rather than calling curl directly.

PagerDuty Key Storage:
  - Before building a job with PagerDuty steps, run `rd keys list` (without `--path`) to see the full key storage tree, then navigate into subdirectories as needed. Using `rd keys list --path keys` may return a 404 on some Rundeck installations.
  - If a key storage path exists (e.g., `keys/pagerduty/api-key`), use that path as the value for `api_token` or `apiKey` in the plugin configuration.
  - If no key is found in storage, ask the user to provide the path to a key in Rundeck's key storage containing a valid PagerDuty API key.
  - If they can't provide a key storage path, offer to add the PagerDuty key from the environment variable `PAGERDUTY_API_KEY` if it is set, or ask them to provide the value of the PagerDuty API key directly (but do not store it in an environment variable or echo it back in the conversation).
  - Also check whether the project has a PagerDuty PluginGroup configured at the project level (via the Rundeck UI or API). If it does, the `api_token`/`apiKey` fields may be optional in individual step configurations.

Rundeck Job YAML Structure:
  Rundeck job YAML uses `sequence.commands` (NOT `workflow.steps`). The correct top-level structure is:

  ```yaml
  - name: My Job
    description: ''
    group: MyGroup
    loglevel: INFO
    executionEnabled: true
    scheduleEnabled: true
    schedules: []
    options:
    - name: my_option
      required: true
    plugins:
      ExecutionLifecycle: null
    sequence:
      keepgoing: false
      strategy: node-first
      commands:
      - type: some-plugin-step
        nodeStep: false
        configuration:
          someKey: someValue
  ```

  Key points:
  - Top-level key is `sequence`, not `workflow`
  - Steps are listed under `sequence.commands`, not `sequence.steps`
  - `keepgoing` and `strategy` belong inside `sequence`
  - Options are a top-level list under `options`, not nested under `sequence`

Plugin Workflow Step YAML Format:
  Plugin-based workflow steps use the `type` field with the plugin's name identifier, `nodeStep: false`, and a `configuration` block:

  ```yaml
  sequence:
    commands:
      - type: pd-note-step
        nodeStep: false
        configuration:
          api_token: "keys/pagerduty/api-key"   # key storage path
          incident_id: "${option.incident_id}"
          note: "Automated note from Rundeck"
  ```

  Key storage paths are referenced as plain strings (e.g., `keys/pagerduty/api-key`). Rundeck resolves them automatically at runtime.

Option Variable Syntax:
  The correct syntax depends on context:
  - In plugin `configuration` fields: use `${option.option_name}` — this works for both regular and secure (valueExposed: true) options
  - In `script` step body content: use `@option.option_name@` for secure options, `${option.option_name}` for regular options
  - In log filter config, notification bodies, and HTTP step fields: use `${option.option_name}`
  - Data context variables from log filters: `${data.variable_name}` — NOTE: older-style `pd-*` plugins do NOT expand `${data.xxx}` in their configuration fields. Use an HTTP step for the action instead if you need to pass captured data.

Passing Data Between Steps (Log Filters):
  Use the `json-mapper` log filter to capture values from JSON step output into context variables:

  ```yaml
  - type: edu.ohio.ais.rundeck.HttpWorkflowStepPlugin
    nodeStep: false
    configuration:
      remoteUrl: "https://api.example.com/data"
      method: GET
      printResponse: 'true'       # REQUIRED — the HTTP step does NOT log the response body by default
      printResponseCode: 'false'  # Keep this false; mixing non-JSON lines breaks the json-mapper
    plugins:
      LogFilter:
      - type: json-mapper
        config:
          filter: '.some.field'
          prefix: my_var
          logData: 'true'
          extraQuotes: 'false'    # REQUIRED when the captured value will be embedded in a JSON string body
  ```

  Then reference the captured value in a subsequent step as `${data.my_var}`.

  Important caveats:
  - `printResponse: 'true'` is required on the HTTP step — without it, the response body is not logged and the json-mapper has nothing to capture.
  - `printResponseCode: 'false'` is required when using json-mapper — the "Response Code: HTTP/1.1 200 OK" line is not valid JSON and will cause the mapper to fail.
  - `extraQuotes: 'false'` is required when embedding the captured value inside a JSON string (e.g., in an HTTP step `body` field). Without it, the value is wrapped in extra quotes that break the JSON.
  - Older-style `pd-*` plugins (e.g., `pd-note-step`) do NOT expand `${data.xxx}` variables. Use an HTTP workflow step for the action if you need to use captured data.

HTTP Request Step (`edu.ohio.ais.rundeck.HttpWorkflowStepPlugin`):
  Key configuration properties:
  - `remoteUrl` (required): the URL
  - `method`: GET, POST, PUT, PATCH, DELETE (default: GET)
  - `headers`: JSON object string, e.g. `'{"Authorization": "Token token=${option.api_key}", "Content-Type": "application/json"}'`
  - `body`: request body string
  - `printResponse`: set to `'true'` to log the response body (required for json-mapper to work)
  - `printResponseCode`: set to `'true'` to log the HTTP status line (avoid when using json-mapper)
  - `printResponseToFile`: set to `'true'` to write response body to a file
  - `file`: file path when using `printResponseToFile`
  - `authentication`: supports OAuth and basic auth
  - `sslVerify`: set to `'false'` to skip SSL validation

Discovering Available Plugins:
  To find all available plugin type names and their services, query the Rundeck API:
  ```
  curl -s -H "X-Rundeck-Auth-Token: $RD_TOKEN" -H "Accept: application/json" "$RD_URL/api/50/plugin/list"
  ```
  Each plugin entry includes `name` (the type identifier to use in YAML), `title` (human-readable), and `service` (e.g., WorkflowStep, Notification, LogFilter).

  To get configuration properties for a specific plugin:
  ```
  curl -s -H "X-Rundeck-Auth-Token: $RD_TOKEN" -H "Accept: application/json" "$RD_URL/api/50/plugin/detail/<service>/<plugin-name>"
  ```

PagerDuty Workflow Step Plugins (service: WorkflowStep):
  Use these `type` values in YAML. All use `nodeStep: false`.

  | Plugin Title                                      | type (name)                          | Key config fields                                                           |
  |---------------------------------------------------|--------------------------------------|-----------------------------------------------------------------------------|
  | PagerDuty / Incident / Note                       | pd-note-step                         | api_token, email, incident_id, note                                         |
  | PagerDuty / Incident / Update                     | pd-update-incident-step              | api_token, email, incident_id, status, resolution, escalation_level, assignees |
  | PagerDuty / Status / Update                       | pd-status-update-step                | api_token, email, incident_id, message (required)                           |
  | PagerDuty / Incident / Escalate                   | pd-escalate-incident-step            | api_token, email, incident_id (required), escalation_level                  |
  | PagerDuty / Incident / Get                        | pagerduty-get-incident               | apiKey, pd-id (required), host, port, includeFirstTriggerLogEntry           |
  | PagerDuty / Event / Send                          | pd-sent-event-step                   | service_key, event_action (required), payload_summary (required), payload_source (required), payload_severity, dedupe_key |
  | PagerDuty / Change Event / Send Change Event      | pagerduty-send-change-event          | routingKey (required), summary (required), source, custom, apiKey           |
  | PagerDuty / Incident / Add Responders             | pagerduty-add-additional-responders  | apiKey, pd-id (required), requester (required), message (required), escalation-policy, user |
  | PagerDuty / Incident / Update Escalation          | pagerduty-update-escalation          | apiKey, pd-id (required), escalation-policy (required), requester           |
  | PagerDuty / Incident / Run Response Play          | pd-run-response-play-step            | api_token, email, incident_id (required), response_play_id (required)       |
  | PagerDuty / Incident / Start Incident Workflow    | pd-start-incident-workflow-step      | api_token, incident_id (required), incident_workflow_id (required)          |
  | PagerDuty / User / Create                         | pagerduty-create-user                | apiKey, name (required), email, role, color, title, description             |
  | PagerDuty / User / Get                            | pagerduty-get-user                   | apiKey, pd-id (required), host, port                                        |
  | PagerDuty / User / Update                         | pagerduty-update-user                | apiKey, pd-id (required), name (required), email, role, color, title        |
  | PagerDuty / User / Delete                         | pagerduty-delete-user                | apiKey, pd-id (required), host, port                                        |
  | PagerDuty / Users / List                          | pagerduty-list-users                 | apiKey, teams, query, host, port                                            |

  Note: Newer-style plugins (pagerduty-*) use `apiKey`; older-style plugins (pd-*) use `api_token`. Both accept a key storage path as the value.
  Note: Older-style `pd-*` plugins do NOT expand `${data.xxx}` context variables in their configuration fields. If you need to pass data captured from a previous step (e.g., via json-mapper), use an HTTP workflow step instead.

PagerDuty Notification Plugins (service: Notification):
  Used in job notification blocks (on-start, on-success, on-failure), not as workflow steps.

  | Plugin Title                                          | name                                     |
  |-------------------------------------------------------|------------------------------------------|
  | Send a PagerDuty v2 event                             | PagerDutyEventNotification               |
  | PagerDuty / Notification / Run Response Play          | pagerduty-response-notification          |
  | PagerDuty / Notification / Start Incident Workflow    | pagerduty-startincidentworkflow-notification |
  | PagerDuty / Notification / Escalate Incident          | pd-escalate-incident-notification        |
  | PagerDuty / Notification / Incident Note              | pd-note-incident-notification            |

PagerDuty Log Filter Plugin (service: LogFilter):
  Applied to individual steps via `plugins.LogFilter` in the step definition.

  | Plugin Title                   | name                              |
  |--------------------------------|-----------------------------------|
  | PagerDuty Incident Output      | pagerduty-incident-output-capture |

rd CLI Command Reference:

  List Projects:
    rd projects list

  List Jobs:
    rd jobs list -p <project>
    rd jobs list -p <project> -j <name-filter>
    rd jobs list -p <project> -g <group-filter>
    rd jobs list -p <project> -i <id1>,<id2>

  Download Job Definition:
    rd jobs list -p <project> -i <id> -f <output-file> -F yaml
    (Note: there is no "rd jobs export" command — use "rd jobs list -f" to download)

  Get Job Info:
    rd jobs info -i <id>
    (Note: rd jobs info does NOT accept -p/--project — use the job ID only)

  Import Job YAML:
    rd jobs load -f <yaml-file> -p <project> -F yaml
    rd jobs load -f <yaml-file> -p <project> -F yaml -d update   # update if exists

  Run a Job:
    rd run -i <job-id> -p <project> -f                          # follow output
    rd run -j "group/name" -p <project> -f -- -opt1 val1        # run by name, pass options
    rd run -i <job-id> -p <project> -f -- -opt1 val1 -opt2 val2 # pass job options with --

  Follow Execution Output:
    rd executions follow -e <execution-id> -f

  List Executions:
    rd executions list -p <project>

  List Nodes:
    rd nodes list -p <project>
    rd nodes list -p <project> -F <node-filter>

  Key Storage:
    rd keys list                                                  # list top-level tree first
    rd keys list --path keys/pagerduty                           # then navigate into subdirs
    rd keys create --path keys/pagerduty/api-key --type password --prompt   # interactive
    rd keys upload --path keys/pagerduty/api-key --type password --file <file>

  Runners (no rd subcommand — use API directly):
    curl -s -H "X-Rundeck-Auth-Token: $RD_TOKEN" -H "Accept: application/json" "$RD_URL/api/50/runnerManagement/runners"

  Cluster:
    rd cluster members

rd CLI Gotchas:
  - `rd jobs info` has no -p/--project flag — use `-i <id>` only.
  - `rd plugins list` shows installable repository plugins, NOT the type names needed in YAML. Use the REST API (`/api/50/plugin/list`) to get type names.
  - `rd run` requires either `-i <id>` or `-j "group/name"`. When using `-j`, `-p` is also required.
  - Job options are passed after `--` as `-optname value` pairs (e.g., `-- -incident_id P123456`).
  - `rd` subcommand help: run the subcommand with no arguments (not `--help`) to see usage, e.g., `rd jobs load` with no args prints usage.
  - The `rd` CLI has no runners subcommand.

Rundeck Job YAML Reference:
  See: https://docs.rundeck.com/docs/manual/document-format-reference/job-yaml-v12.html
