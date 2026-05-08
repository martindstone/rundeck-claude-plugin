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
  - use the `rd` CLI tool to interact with Rundeck. See the reference at https://docs.rundeck.com/docs/rd-cli/commands.html for available commands and options.
  - When asked to create a job, generate YAML first and show it to the user.
  - Do not import or execute a job unless the user explicitly approves.
  - Prefer safe/read-only commands unless the user explicitly requests a mutating action.
  - Use environment variables for `rd` CLI configuration:
    - RD_URL
    - RD_TOKEN
    - RD_PROJECT
  - For any Rundeck-related request, determine the appropriate `rd` CLI command to accomplish the task and execute it. If the request is ambiguous, ask the user for clarification before proceeding. If there's no `rd` command available, you can use the Rundeck REST API directly.
  - When adding workflow steps to Rundeck jobs for communicating with PagerDuty, use the available PagerDuty plugins within Rundeck. See if the project key storage contains a PagerDuty API Key for use with PagerDuty from Rundeck. If none is found, ask the user to provide the path to a key in Rundeck's key storage containing a valid PagerDuty API key, and use that in the workflow step configuration. If they can't provide a key storage path, you can offer to add the PagerDuty key from the environment variable `PAGERDUTY_API_KEY` if it is set, or ask them to provide the value of the PagerDuty API key directly in the conversation (but do not store it in an environment variable or show it in the conversation). Use the provided PagerDuty API key in the workflow step configuration for communicating with PagerDuty.

Helpers:
  - Rundeck Job YAML Reference:
    See: https://docs.rundeck.com/docs/manual/document-format-reference/job-yaml-v12.html
  - List Projects:
    `rd projects list`
  - List Jobs:
    `rd jobs list --project <project-name>`

Commands:
  - Generate YAML:
    Ask the user what they want the new Rundeck job to do. Generate a sample Rundeck job YAML based on the user's requirements and save it to a file, e.g., `check-disk.yaml`.

  - Import YAML:
    `rd jobs load --file <yaml-file> --project <project-name> --format yaml`

  - Execute job:
    `rd run --id <job-id> --project <project-name> --follow`

  - Other requests: Use the guide at https://docs.rundeck.com/docs/rd-cli/commands.html to find the appropriate `rd` CLI command to accomplish the user's request.

Additional Rundeck CLI Help:
  - For more details on using the `rd` CLI, refer to the official documentation: https://docs.rundeck.com/docs/rd-cli/commands.html
  - To export/download a job definition, use rd jobs list --project <project> --idlist <id> -f <file> --format yaml. There is no rd jobs export command.
  - `rd jobs info` has no --project flag - `rd jobs info -i <id>` only — it does not accept --project.
  - The rd CLI has no runners subcommand. To list runners and their status, query the API directly: `curl -s -H "X-Rundeck-Auth-Token: $RD_TOKEN" -H "Accept: application/json" "$RD_URL/api/50/runnerManagement/runners"`

Rundeck CLI Basic Help:
  ```
    Usage: rd [-hV] [COMMAND]
    -h, --help      Show this help message and exit.
    -V, --version   Print version information and exit.

  Available commands:

    acl         Generate, Test, and Validate ACLPolicy files
    adhoc       Run adhoc command or script on matching nodes.
    cluster     Manage Rundeck Enterprise Cluster
    executions  List running executions, attach and follow their output, or kill them.
    jobs        List and manage Jobs.
    keys        Manage Keys via the Key Storage Facility.
    license     Manage Rundeck Enterprise License
    metrics     View metrics endpoints information.
    nodes       List node resources.
    plugins     Manage Rundeck plugins
    projects    List and manage projects.
    retry       Run a Job based on a specific execution.
    run         Run a Job.
    scheduler   View scheduler information
    system      View system information
    tokens      Create, and manage tokens
    users       Manage user information.
    version     Print version information
  ```
