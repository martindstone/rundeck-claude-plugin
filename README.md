# Rundeck Claude Plugin

This repository contains a local Claude plugin for working with Rundeck through the Rundeck CLI.

The plugin defines one skill, `rundeck-job`, which is intended to help Claude:

- generate Rundeck job YAML
- import jobs with the `rd` CLI
- list or inspect Rundeck resources
- execute Rundeck jobs when explicitly asked

## Repository Layout

- `plugin.json`: Claude plugin manifest
- `skills/rundeck-job/SKILL.md`: instructions for Rundeck-related requests

## Prerequisites

You need the Rundeck CLI installed and available on your `PATH` as `rd`.

Official install documentation:

- https://docs.rundeck.com/docs/learning/howto/learn-rd-cli.html

### Install on macOS

If you use Homebrew:

```bash
brew tap rundeck/rundeck-cli
brew install rundeck-cli
```

Then verify it:

```bash
rd version
```

### Install on Linux

Follow the official Rundeck CLI installation guide for your distribution and preferred package method:

- https://docs.rundeck.com/docs/learning/howto/learn-rd-cli.html

After installation, verify it:

```bash
rd version
```

## Required Environment Variables

The skill expects these environment variables to be set before Claude uses the plugin:

- `RD_URL`
- `RD_TOKEN`
- `RD_PROJECT`

### `RD_URL`

This is the base URL of your Rundeck server.

Example:

```bash
export RD_URL="https://rundeck.example.com"
```

How to get it:

- use the URL you already open in the Rundeck web UI
- ask your Rundeck administrator for the correct server URL

### `RD_TOKEN`

This is a Rundeck API token with permission to read or manage the resources you want Claude to use.

Example:

```bash
export RD_TOKEN="your-api-token"
```

How to get it:

- in Rundeck, create or copy an API token from your user profile or token management area
- if token creation is restricted, ask your Rundeck administrator to provide one with the needed permissions

Do not commit this value to the repository.

### `RD_PROJECT`

This is the Rundeck project Claude should use by default.

Example:

```bash
export RD_PROJECT="my-project"
```

How to get it:

- copy the project name from the Rundeck web UI
- or list available projects with:

```bash
rd projects list
```

## Example Shell Setup

You can export the required variables in your current shell session:

```bash
export RD_URL="https://rundeck.example.com"
export RD_TOKEN="your-rundeck-api-token"
export RD_PROJECT="my-project"
export PD_API_TOKEN="your-pagerduty-api-token"
```

If you want them to persist, add the exports to your shell profile such as `~/.zshrc` or `~/.bashrc`.

## Testing the Plugin Locally

You can test this plugin by pointing Claude at the repository directory with `--plugin-dir`.

```bash
claude --plugin-dir /path/where/you/cloned/rundeck-claude-plugin
```

Before running Claude, make sure:

- `rd` is installed and works
- `RD_URL`, `RD_TOKEN`, and `RD_PROJECT` are set in the shell where you launch Claude

Once Claude starts with the plugin loaded, you can try prompts like:

- "Create a Rundeck job YAML that checks disk usage on all nodes"
- "List jobs in my Rundeck project"
- "Show me a sample Rundeck job that echoes hello world"

## Notes

The skill is written to prefer safe behavior:

- it checks that `rd` is available
- it requires the Rundeck environment variables to be set
- it should generate YAML before importing or running jobs
- it should not import or execute jobs unless explicitly asked
- it should not allow any auth tokens to be output to the terminal
