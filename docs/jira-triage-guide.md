# Jira Triage Agent — Testing Guide

This guide walks through setting up and testing the unified triage agent for Jira issues. The triage agent is source-agnostic — the same agent handles both GitHub issues and Jira issues, selected by the `TRIAGE_SOURCE` environment variable.

## How it works

```
Jira issue created (or /fs-triage comment)
  -> Jira Automation Rule fires HTTP webhook
  -> GitHub repository_dispatch event
  -> jira-dispatch.yml validates enrollment, dispatches jira-triage.yml
  -> jira-triage.yml runs the unified triage agent with TRIAGE_SOURCE=jira
  -> pre-triage.sh routes to pre-triage-jira.sh (fetches Jira data on host)
  -> Agent triages in sandbox (reads pre-fetched JSON, scores clarity, decides action)
  -> post-triage.sh routes to post-triage-jira.sh (posts ADF comment + labels to Jira)
```

**Security model:** Jira credentials (`JIRA_EMAIL`, `JIRA_API_TOKEN`) never enter the agent sandbox. The pre-script fetches issue data on the GitHub Actions host and writes it to a JSON file that gets mounted into the sandbox. The post-script reads the agent's JSON output and posts results back to Jira, also on the host.

**Known limitation:** The current implementation uses a static Jira API token (`JIRA_API_TOKEN`), which means all triage comments appear under the name of the user who generated the token. A future improvement will replace this with a method that generates temporary tokens (e.g., OAuth 2.0 or Jira Forge app) so comments can be attributed to a service account.

## Prerequisites

- A GitHub repository with fullsend installed (per-repo mode)
- GCP Workload Identity Federation configured for Claude authentication
- A Jira Cloud project you have admin access to
- A Jira API token ([create one here](https://id.atlassian.com/manage-profile/security/api-tokens))
- `gh` CLI authenticated with push access to the repo

## Setup

### Step 1: Set GitHub secrets

The triage agent needs Jira credentials to fetch issue data and post results:

```bash
REPO="<your-org>/<your-repo>"

gh secret set JIRA_HOST --repo "$REPO"
# Enter your Jira host (e.g., myorg.atlassian.net)

gh secret set JIRA_EMAIL --repo "$REPO"
# Enter the email associated with the Jira API token

gh secret set JIRA_API_TOKEN --repo "$REPO"
# Enter the Jira API token
```

Verify the existing fullsend secrets are also set:
- `FULLSEND_GCP_WIF_PROVIDER` - GCP Workload Identity Federation provider
- `FULLSEND_GCP_PROJECT_ID` - GCP project ID for Vertex AI

### Step 2: Create `.jira.yml`

Create a `.jira.yml` file at the root of your repository to enroll Jira projects:

```yaml
version: 1
jira_projects:
  - project_key: MYPROJ
    host: myorg.atlassian.net
    linked_github_repos: []
```

| Field | Description |
|-------|-------------|
| `project_key` | The Jira project key (e.g., `MYPROJ`, `KFLUXINFRA`) |
| `host` | The Jira Cloud hostname (without `https://`) |
| `linked_github_repos` | Optional list of `org/repo` values for cross-repo GitHub search |

You can enroll multiple projects:

```yaml
version: 1
jira_projects:
  - project_key: MYPROJ
    host: myorg.atlassian.net
    linked_github_repos:
      - myorg/backend
      - myorg/frontend
  - project_key: INFRA
    host: myorg.atlassian.net
    linked_github_repos:
      - myorg/infra-deployments
```

### Step 3: Add customized triage files

The unified triage agent files must be placed in `.fullsend/customized/` to overlay the upstream defaults. Copy these from the fullsend scaffold:

```
.fullsend/customized/
  agents/triage.md                        # Unified prompt (GitHub + Jira)
  env/triage.env                          # TRIAGE_SOURCE + source-specific vars
  harness/triage.yaml                     # Unified harness config
  policies/triage.yaml                    # Sandbox policy
  schemas/triage-result.schema.json       # Unified output schema
  scripts/
    pre-triage.sh                         # Router: dispatches to pre-triage-${SOURCE}.sh
    post-triage.sh                        # Router: dispatches to post-triage-${SOURCE}.sh
    pre-triage-github.sh                  # GitHub pre-script (label reset)
    post-triage-github.sh                 # GitHub post-script (gh API)
    pre-triage-jira.sh                    # Jira pre-script (fetch issue + strip labels)
    post-triage-jira.sh                   # Jira post-script (REST API + ADF comment)
    markdown-to-adf.py                    # Markdown to Atlassian Document Format converter
```

Ensure all `.sh` files are executable (`chmod +x`).

### Step 4: Add workflows

Add two workflow files to `.github/workflows/`:

**`jira-dispatch.yml`** catches `repository_dispatch` events from Jira Automation, validates project enrollment against `.jira.yml`, and dispatches `jira-triage.yml`.

**`jira-triage.yml`** is the triage agent workflow. It checks out upstream defaults from `fullsend-ai/fullsend@v0`, overlays your customizations, authenticates to GCP, and runs `fullsend run triage` with `TRIAGE_SOURCE=jira`.

See the existing workflow files in this repository for the complete implementations.

### Step 5: Set up Jira Automation rules

Two Jira Automation rules connect your Jira project to the triage agent. These must be created manually in the Jira UI for now. Automating this setup via the `fullsend` CLI is planned for a future release.

#### Obtain a GitHub dispatch token

Create a fine-grained Personal Access Token:

1. Go to GitHub -> Settings -> Developer settings -> Fine-grained tokens -> New token
2. **Token name**: `fullsend-jira-dispatch`
3. **Repository access**: Select your repo only
4. **Permissions**: Contents -> Read and write
5. Copy the token value

#### Rule 1: Auto-triage on issue creation

1. Open your Jira space -> **Space Settings** -> **Automation**
2. Click **Create flow**
3. **Trigger**: Select **Work item created**
4. **Action**: **Send web request**
   - **URL**: `https://api.github.com/repos/<ORG>/<REPO>/dispatches`
   - **HTTP method**: `POST`
   - **Headers**:
     - `Authorization`: `Bearer <YOUR_GITHUB_PAT>`
     - `Accept`: `application/vnd.github.v3+json`
     - `Content-Type`: `application/json`
   - **Body** (Custom data / JSON):
     ```json
     {
       "event_type": "jira-issue-created",
       "client_payload": {
         "issue_key": "{{issue.key}}",
         "project_key": "{{issue.fields.project.key}}"
       }
     }
     ```
5. Name: `fullsend: auto-triage on issue creation`
6. Click **Turn it on**

#### Rule 2: On-demand `/fs-triage` command

1. **Space Settings** -> **Automation** -> **Create flow**
2. **Trigger**: **Comment added**
3. **Condition**: **Comment body contains text** -> `/fs-triage`
4. **Action**: **Send web request** (same URL and headers as Rule 1)
   - **Body**:
     ```json
     {
       "event_type": "jira-command",
       "client_payload": {
         "issue_key": "{{issue.key}}",
         "project_key": "{{issue.fields.project.key}}",
         "command": "/fs-triage"
       }
     }
     ```
5. Name: `fullsend: /fs-triage command`
6. Click **Turn it on**

## Testing

### Test 1: On-demand triage (quickest)

Trigger the triage workflow manually without Jira Automation:

```bash
gh workflow run jira-triage.yml \
  --repo <ORG>/<REPO> \
  -f issue_key=MYPROJ-123 \
  -f project_key=MYPROJ
```

Watch the run:

```bash
gh run list --repo <ORG>/<REPO> --limit 3
gh run watch <RUN_ID> --repo <ORG>/<REPO>
```

After completion, check the Jira issue for:
- A triage comment (ADF-formatted)
- A `fullsend:needs-info` or `fullsend:ready-to-code` label

### Test 2: On-demand via `/fs-triage` comment

Comment `/fs-triage` on a Jira issue. Wait 30-60 seconds for Jira Automation to fire, then:

```bash
gh run list --repo <ORG>/<REPO> --event repository_dispatch --limit 5
```

You should see a `Jira Dispatch` run followed by a `Jira Triage` run.

### Test 3: Auto-triage on issue creation

Create a new Jira issue in the enrolled project:

```bash
acli jira --action createIssue \
  --project MYPROJ \
  --type Task \
  --summary "Test: fullsend auto-triage" \
  --description "This is a test issue with an intentionally vague description."
```

Wait 30-60 seconds, then verify the workflow fired and the issue was triaged.

### Test 4: Re-triage after clarification

1. The first triage should result in `fullsend:needs-info` (vague description)
2. Add a detailed comment to the Jira issue explaining the problem
3. Comment `/fs-triage` to re-trigger
4. The agent should accumulate the prior analysis and may upgrade to `fullsend:ready-to-code`

## Triage actions

The agent produces one of five actions:

| Action | Label applied | When |
|--------|--------------|------|
| `insufficient` | `fullsend:needs-info` | Missing information. Agent asks a clarifying question. |
| `sufficient` | `fullsend:ready-to-code` or `fullsend:triaged` | Enough info for a developer to start work. |
| `duplicate` | `fullsend:duplicate` | Matches an existing issue. |
| `blocked` | `fullsend:blocked` | Depends on unresolved work. |
| `question` | `fullsend:question` | Issue is a support question, not a bug/feature. |

For `sufficient`, the label depends on category:
- `bug`, `documentation`, `performance` -> `fullsend:ready-to-code`
- `feature` -> `fullsend:feature`
- `security`, `other` -> `fullsend:triaged`

## Control labels

These labels are managed by the triage agent. Do not add or remove them manually.

| Label | Meaning |
|-------|---------|
| `fullsend:needs-info` | Missing information |
| `fullsend:ready-to-code` | Ready for implementation |
| `fullsend:triaged` | Triaged, needs human prioritization |
| `fullsend:feature` | Feature request awaiting prioritization |
| `fullsend:duplicate` | Duplicate of another issue |
| `fullsend:blocked` | Blocked on other work |
| `fullsend:question` | Support question |

## Troubleshooting

### No GitHub Actions run after Jira event

- **Check Jira Automation audit log**: Space Settings -> Automation -> click rule -> Audit log
- **Test the webhook manually**:
  ```bash
  curl -s -o /dev/null -w "%{http_code}" \
    -X POST \
    -H "Authorization: Bearer <YOUR_GITHUB_PAT>" \
    -H "Accept: application/vnd.github.v3+json" \
    -H "Content-Type: application/json" \
    -d '{"event_type":"jira-issue-created","client_payload":{"issue_key":"MYPROJ-1","project_key":"MYPROJ"}}' \
    https://api.github.com/repos/<ORG>/<REPO>/dispatches
  ```
  `204` = success. `401` = invalid token. `404` = wrong repo or insufficient scope.
- **Check token scope**: The PAT needs `Contents: Read and write` on the specific repo.

### Jira Dispatch succeeds but Jira Triage fails

```bash
gh run view <RUN_ID> --repo <ORG>/<REPO> --log-failed
```

Common issues:
- **`.jira.yml` not found**: Ensure the file exists at the repo root
- **Project not enrolled**: Check `project_key` in `.jira.yml` matches exactly
- **Missing secrets**: Verify `JIRA_HOST`, `JIRA_EMAIL`, `JIRA_API_TOKEN` are set
- **GCP auth failure**: Check `FULLSEND_GCP_WIF_PROVIDER` and `FULLSEND_GCP_PROJECT_ID`

### Agent runs but no Jira comment appears

- Check the post-script output in the workflow logs for HTTP response codes
- Verify `JIRA_HOST` matches your Jira instance exactly (no `https://` prefix)
- Verify `JIRA_API_TOKEN` is valid and the associated user has comment permissions

### The `/fs-triage` comment condition doesn't match

- The condition must match the literal string `/fs-triage` (case-sensitive)
- Check for leading/trailing spaces in the Jira Automation condition value
