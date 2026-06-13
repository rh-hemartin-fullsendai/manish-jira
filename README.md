# manish-jira

This repository is set up to test a Jira agent integrated with
[fullsend](https://github.com/fullsend-ai/fullsend). It was created by
[@manish-jangra](https://github.com/manish-jangra).

## Usage

You can trigger the Jira agent by posting a slash command as a comment on
any issue in this repository:

```
/jira-triage
```

This dispatches the `jira-agent` workflow, which runs the fullsend Jira
agent against the issue.

## Workflows

- **[`jira-dispatch.yml`](.github/workflows/jira-dispatch.yml)** — Listens
  for `/jira-triage` comments on issues and dispatches the Jira agent
  workflow.
- **[`jira-agent.yml`](.github/workflows/jira-agent.yml)** — Runs the
  fullsend Jira agent. Accepts an issue key and source (`github` or
  `jira`) as inputs.
