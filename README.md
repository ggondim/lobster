# Lobster

An OpenClaw-native workflow shell: typed (JSON-first) pipelines, jobs, and approval gates.


## Example of lobster at work
OpenClaw or any other AI agent can use `lobster` as a workflow engine and not construct a query every time - thus saving tokens, providing room for determinism, and resumability.

### Watching a PR that hasn't had changes
```
node bin/lobster.js "workflows.run --name github.pr.monitor --args-json '{\"repo\":\"moltbot/moltbot\",\"pr\":1152}'"
[
  {
    "kind": "github.pr.monitor",
    "repo": "moltbot/moltbot",
    "prNumber": 1152,
    "key": "github.pr:moltbot/moltbot#1152",
    "changed": false,
    "summary": {
      "changedFields": [],
      "changes": {}
    },
    "prSnapshot": {
      "author": {
        "id": "MDQ6VXNlcjE0MzY4NTM=",
        "is_bot": false,
        "login": "vignesh07",
        "name": "Vignesh"
      },
      "baseRefName": "main",
      "headRefName": "feat/lobster-plugin",
      "isDraft": false,
      "mergeable": "MERGEABLE",
      "number": 1152,
      "reviewDecision": "",
      "state": "OPEN",
      "title": "feat: Add optional lobster plugin tool (typed workflows, approvals/resume)",
      "updatedAt": "2026-01-18T20:16:56Z",
      "url": "https://github.com/moltbot/moltbot/pull/1152"
    }
  }
]
```
### And a PR that has a state change (in this case an approved PR)

```
 node bin/lobster.js "workflows.run --name github.pr.monitor --args-json '{\"repo\":\"moltbot/moltbot\",\"pr\":1200}'"
[
  {
    "kind": "github.pr.monitor",
    "repo": "moltbot/moltbot",
    "prNumber": 1200,
    "key": "github.pr:moltbot/moltbot#1200",
    "changed": true,
    "summary": {
      "changedFields": [
        "number",
        "title",
        "url",
        "state",
        "isDraft",
        "mergeable",
        "reviewDecision",
        "updatedAt",
        "baseRefName",
        "headRefName"
      ],
      "changes": {
        "number": {
          "from": null,
          "to": 1200
        },
        "title": {
          "from": null,
          "to": "feat(tui): add syntax highlighting for code blocks"
        },
        "url": {
          "from": null,
          "to": "https://github.com/moltbot/moltbot/pull/1200"
        },
        "state": {
          "from": null,
          "to": "MERGED"
        },
        "isDraft": {
          "from": null,
          "to": false
        },
        "mergeable": {
          "from": null,
          "to": "UNKNOWN"
        },
        "reviewDecision": {
          "from": null,
          "to": ""
        },
        "updatedAt": {
          "from": null,
          "to": "2026-01-19T05:06:09Z"
        },
        "baseRefName": {
          "from": null,
          "to": "main"
        },
        "headRefName": {
          "from": null,
          "to": "feat/tui-syntax-highlighting"
        }
      }
    },
    "prSnapshot": {
      "author": {
        "id": "MDQ6VXNlcjE0MzY4NTM=",
        "is_bot": false,
        "login": "vignesh07",
        "name": "Vignesh"
      },
      "baseRefName": "main",
      "headRefName": "feat/tui-syntax-highlighting",
      "isDraft": false,
      "mergeable": "UNKNOWN",
      "number": 1200,
      "reviewDecision": "",
      "state": "MERGED",
      "title": "feat(tui): add syntax highlighting for code blocks",
      "updatedAt": "2026-01-19T05:06:09Z",
      "url": "https://github.com/moltbot/moltbot/pull/1200"
    }
  }
]
```

## Goals


- Typed pipelines (objects/arrays), not text pipes.
- Local-first execution.
- No new auth surface: Lobster must not own OAuth/tokens.
- Composable macros that OpenClaw can invoke in one step to save tokens.

## Quick start

From this folder:

- `pnpm install`
- `pnpm test`
- `pnpm lint`
- `node ./bin/lobster.js --help`
- `node ./bin/lobster.js doctor`
- `node ./bin/lobster.js "exec --json --shell 'echo [1,2,3]' | where '0>=0' | json"`

### Notes

- `pnpm test` runs `tsc` and then executes tests against `dist/`.
- `bin/lobster.js` prefers the compiled entrypoint in `dist/` when present.

## CLI reference

### Running a pipeline

A pipeline is a sequence of commands separated by `|`. Each command receives the previous command's output items and emits new items:

```
lobster '<pipeline>'
lobster run '<pipeline>'
lobster run --mode tool '<pipeline>'
```

Example:

```
lobster 'exec --json --shell "echo [1,2,3]" | where "0>=0" | json'
```

### Modes

Lobster has two output modes:

- **human** (default): renderers like `table` and `json` write directly to stdout.
- **tool** (`--mode tool`): output is always a single JSON envelope on stdout. Use this when calling from an AI agent like OpenClaw.

```
lobster run --mode tool 'exec --json --shell "echo [1,2,3]" | where "0>=0"'
```

Tool mode output:

```json
{
  "protocolVersion": 1,
  "ok": true,
  "status": "ok",
  "output": [1, 2, 3],
  "requiresApproval": null
}
```

### Approval flow (tool mode)

When a pipeline halts at an `approve` gate, the envelope has `status: "needs_approval"` and includes a `resumeToken`:

```json
{
  "protocolVersion": 1,
  "ok": true,
  "status": "needs_approval",
  "output": [],
  "requiresApproval": {
    "type": "approval_request",
    "prompt": "Send 3 emails?",
    "items": [...],
    "resumeToken": "<opaque string>"
  }
}
```

The agent presents the prompt and items to the user, then resumes or cancels:

```
# Approve — continue the pipeline after the gate
lobster resume --token <resumeToken> --approve yes

# Cancel — abort the pipeline
lobster resume --token <resumeToken> --approve no
```

The `resume` command also returns a tool envelope with `status: "ok"` or `"cancelled"`.

### Running a workflow file

```
lobster run path/to/workflow.lobster
lobster run --file path/to/workflow.lobster --args-json '{"tag":"family"}'
```

In tool mode:

```
lobster run --mode tool --file path/to/workflow.lobster --args-json '{"repo":"owner/repo","pr":123}'
```

### Running a named workflow

```
lobster 'workflows.run --name github.pr.monitor --args-json "{\"repo\":\"owner/repo\",\"pr\":123}"'
```

### Doctor

Verify that Lobster is installed and operational (useful for OpenClaw health checks):

```
lobster doctor
```

Output:

```json
{
  "protocolVersion": 1,
  "ok": true,
  "status": "ok",
  "output": [{ "toolMode": true, "protocolVersion": 1, "version": "2026.1.21-1" }],
  "requiresApproval": null
}
```

### Help

```
lobster help
lobster help exec
lobster help approve
lobster help clawd.invoke
```

## Commands

| Command | Description |
|---|---|
| `exec` | Run an OS command; `--json` parses stdout as JSON; `--stdin raw&#124;json&#124;jsonl` feeds pipeline input to subprocess stdin |
| `where` | Filter items by expression (e.g. `where "state=='OPEN'"`) |
| `pick` | Select fields from items (e.g. `pick title url`) |
| `head` | Keep only the first N items |
| `sort` | Sort items by a field |
| `dedupe` | Remove duplicate items by key |
| `map` | Transform items using a shell command or template |
| `group_by` | Group items by field value |
| `template` | Render items with a string template |
| `json` | Pretty-print items as JSON to stdout (human mode renderer) |
| `table` | Render items as a table to stdout (human mode renderer) |
| `approve` | Approval gate — prompts on TTY or emits `approval_request` in tool/non-interactive mode |
| `state.get` | Read a value from Lobster's key/value state store |
| `state.set` | Write a value to Lobster's key/value state store |
| `diff.last` | Compare current items with the last saved snapshot; emits a diff result |
| `clawd.invoke` | Call an OpenClaw tool endpoint (see below) |
| `llm_task.invoke` | Invoke an LLM task via a tool endpoint |
| `gog.gmail.search` | Fetch Gmail messages via the `gog` CLI |
| `gog.gmail.send` | Send Gmail messages via the `gog` CLI |
| `email.triage` | Classify email messages into buckets |
| `workflows.list` | List available named workflows |
| `workflows.run` | Run a named workflow by name with optional JSON args |
| `commands.list` | List all available commands |

## Using Lobster with OpenClaw

OpenClaw calls Lobster as a subprocess tool. The integration points are:

1. **Health check**: call `lobster doctor` and verify `ok: true` in the envelope.
2. **Run a pipeline or workflow**: call `lobster run --mode tool '<pipeline>'`.
3. **Handle approval gates**: when `status` is `"needs_approval"`, show `requiresApproval.prompt` and `requiresApproval.items` to the user, then call `lobster resume`.
4. **Call back into OpenClaw from a pipeline**: use `clawd.invoke` inside a pipeline.

### Calling OpenClaw tools from within a Lobster pipeline

`clawd.invoke` bridges from a Lobster pipeline back to an OpenClaw tool endpoint. Configure the target with environment variables or flags:

- `CLAWD_URL` — OpenClaw base URL (e.g. `http://localhost:3456`)
- `CLAWD_TOKEN` — optional Bearer auth token

```
# Send a message via an OpenClaw tool
exec --json 'gh pr list --json number,title,url' \
  | where "reviewDecision=='APPROVED'" \
  | clawd.invoke --tool message --action send \
      --args-json '{"provider":"telegram","to":"me","message":"PR approved!"}'
```

Using `--each` to call the tool once per input item (merges the item into `--args-json`):

```
exec --json 'gh pr list --json number,title,url' \
  | clawd.invoke --tool trello --action card.create --each \
      --args-json '{"list":"Review"}'
```

**`clawd.invoke` options**

| Flag | Description |
|---|---|
| `--url` | Override `CLAWD_URL` |
| `--token` | Override `CLAWD_TOKEN` |
| `--tool` | Tool name (required) |
| `--action` | Tool action (required) |
| `--args-json` | JSON object of tool arguments |
| `--each` | Call the tool once per input item, merging the item under `--item-key` (default: `item`) |
| `--item-key` | Key name used when merging the item into `--args-json` with `--each` |
| `--session-key` | Optional session attribution |
| `--dry-run` | Log but do not execute the tool call |

## Workflow files

Lobster can run YAML/JSON workflow files with `steps`, `env`, `condition`, and approval gates.

```
lobster run path/to/workflow.lobster
lobster run --file path/to/workflow.lobster --args-json '{"tag":"family"}'
```

Example file:

```yaml
name: inbox-triage
steps:
  - id: collect
    command: inbox list --json
  - id: categorize
    command: inbox categorize --json
    stdin: $collect.stdout
  - id: approve
    command: inbox apply --approve
    stdin: $categorize.stdout
    approval: required
  - id: execute
    command: inbox apply --execute
    stdin: $categorize.stdout
    condition: $approve.approved
```

### Step fields

| Field | Required | Description |
|---|---|---|
| `id` | yes | Unique step identifier; used to reference the step's output in subsequent steps via `$id.stdout` or `$id.json`. |
| `command` | yes (for regular steps) | Shell command to run. Mutually exclusive with `lobster`. |
| `lobster` | yes (for sub-workflow steps) | Path to a `.lobster` file to run as a sub-workflow (resolved relative to the parent workflow). Mutually exclusive with `command`. |
| `args` | no | Key/value map of input arguments passed to the sub-workflow. Values support `${arg}` and `$stepId.stdout`/`$stepId.json` template syntax. Only valid when `lobster` is set. |
| `loop` | no | Repeat the sub-workflow step in a loop. Only valid when `lobster` is set. |
| `loop.maxIterations` | yes (when `loop` is set) | Maximum number of iterations. |
| `loop.condition` | no | Shell command evaluated after each iteration. Exit code 0 continues the loop; non-zero stops it early. Receives `LOBSTER_LOOP_STDOUT`, `LOBSTER_LOOP_JSON`, and `LOBSTER_LOOP_ITERATION` as environment variables. |
| `stdin` | no | Pass a previous step's output as stdin. |
| `approval` | no | Set to `required` to insert an approval gate before the step runs. |
| `condition` | no | Expression that must be truthy for the step to run. |

### Sub-workflow step example

Use the `lobster` field to embed another `.lobster` file as a step in your workflow, optionally passing arguments and looping until a condition is met:

```yaml
steps:
  - id: prepare
    command: echo "hello"

  - id: process
    lobster: ./sub_workflow.lobster
    args:
      input: $prepare.stdout
    loop:
      maxIterations: 10
      condition: '! echo "$LOBSTER_LOOP_STDOUT" | grep -q "^done"'
```

The sub-workflow's last step result (stdout/json) is stored as the step result and is accessible via `$process.stdout` / `$process.json` in subsequent steps.
