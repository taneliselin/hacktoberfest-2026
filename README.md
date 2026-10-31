# delegate

> Hand a coding task to a worker model. Get back a patch that passes your tests.

One file. Zero dependencies. Any OpenAI-compatible model. Your repo stays untouched until you say so.

`delegate` is a small command-line tool that lets an AI agent (Claude Code, or a plain shell script) outsource a bounded coding task to a cheaper or faster worker model. The worker edits a temporary copy of your repo, and the tool, not the worker, decides whether the job succeeded by running your acceptance command before and after. You get a patch file, a full transcript and a metrics row. Nothing lands in your working tree until you run `apply`.

## Why you will like it

- **It judges with your tests, not with the model's opinion.** The acceptance command runs before the worker starts and again after it finishes. Status `accepted` means exit 0 plus a non-empty patch. No "trust me, it works".
- **Your repo is never touched during a run.** The worker lives in a temp copy. The patch waits in `.delegate/patches/` until you read it and apply it.
- **Zero dependencies.** Python standard library only. One file, about 550 lines, easy to read in one sitting and easy to audit.
- **Model-agnostic.** Anything that speaks OpenAI-style `chat/completions` with tool calling works: hosted APIs, self-hosted vLLM, cloud gateways.
- **Works where API keys are banned.** Auth is just headers. Use a Bearer token, custom headers, or a short-lived OAuth token (for example from `gcloud auth print-access-token` for Vertex AI's OpenAI-compatible endpoint).
- **The worker cannot see your secrets.** Commands run with a scrubbed environment (only `PATH`, `LANG`, `TERM`, a temp `HOME`). `.env*`, keys, databases and files over 1 MB are never copied.
- **Contained by design.** Path traversal is blocked, steps and wall-clock time are capped, tool output is truncated, malformed tool calls are fed back to the model instead of crashing the loop.
- **Fully inspectable.** Every run writes a JSONL transcript and a line in `metrics.csv` (status, steps, tokens, seconds).
- **Self-testing.** `--selftest` spins up an in-process fake model server and checks 15 behaviours, including header handling, secret scrubbing and patch validity. No API key needed.

## See it work

A trimmed result from a real smoke test (a deliberately trivial task):

```json
{
  "status": "accepted",
  "stop_reason": "finished",
  "acceptance_before": { "exit_code": 1 },
  "acceptance_after":  { "exit_code": 0 },
  "files_changed": ["README.md"],
  "steps": 5,
  "usage": { "prompt": 20542, "completion": 540 },
  "seconds": 35.1
}
```

Acceptance went from failing to passing in five steps, and the patch is waiting for your review.

## Quick start

Requirements: Python 3, `bash`, `diff`, and either `git` or `patch` (for `apply`).

```bash
# 1. Drop the file into your repo
mkdir -p tools/delegate && cp delegate.py tools/delegate/
echo ".delegate/" >> .gitignore

# 2. Check your setup (no network, no API key)
python3 tools/delegate/delegate.py --selftest

# 3. Point it at a model
export DELEGATE_BASE_URL="https://<host>/v1"          # chat/completions is appended
export DELEGATE_MODEL="<model-name>"
export DELEGATE_API_KEY="<token>"                      # optional, sent as Bearer

# 4. Delegate something
python3 tools/delegate/delegate.py run \
  --task-file TASK.md \
  --accept "npm test"

# 5. Read the patch, then apply it
cat .delegate/patches/<run_id>.patch
python3 tools/delegate/delegate.py apply <run_id>
```

### Example: Vertex AI (no API key required)

```bash
export DELEGATE_BASE_URL="https://aiplatform.googleapis.com/v1beta1/projects/<PROJECT_ID>/locations/global/endpoints/openapi"
export DELEGATE_MODEL="google/<model-id>"
export DELEGATE_API_KEY="$(gcloud auth print-access-token)"   # lasts about one hour
```

## Configuration

| Variable | Meaning |
|---|---|
| `DELEGATE_BASE_URL` | Base URL of the OpenAI-compatible API. Required. |
| `DELEGATE_MODEL` | Model name sent in the request. |
| `DELEGATE_API_KEY` | Optional. Sent as `Authorization: Bearer ...`. |
| `DELEGATE_HEADERS` | Optional extra headers as `Name: value`, separated by `;`. |
| `DELEGATE_TEMPERATURE` | Optional. Default `0.2`. |

| `run` option | Default | Meaning |
|---|---|---|
| `--task-file` / `--task` | none | The task, written to be self-contained. |
| `--accept` | required | Command that decides success (exit 0 = pass). |
| `--max-steps` | 25 | Maximum worker steps. |
| `--timeout` | 600 | Worker wall-clock budget in seconds. |
| `--accept-timeout` | 300 | Time limit for each acceptance run. |
| `--repo` | `.` | Repository root. |

## How it works

1. Copies your repo to a temp directory (twice: an untouched original and a working copy).
2. Runs your acceptance command on the working copy and records the result.
3. Runs the worker loop. The worker has seven tools: `list_dir`, `read_file`, `write_file`, `replace_in_file`, `grep`, `run`, `finish`.
4. Runs your acceptance command again.
5. Diffs the original against the working copy and saves the patch.
6. Prints one JSON object and writes the transcript and metrics. Statuses: `accepted`, `no_change`, `failed`, `error`.

## Use it from Claude Code

Add this to your project's `CLAUDE.md` and let the main agent delegate:

```markdown
## Delegation
Bounded, verifiable coding tasks can be delegated to a worker:
  python3 tools/delegate/delegate.py run --task-file TASK.md --accept "<test command>"
Write the task into TASK.md so it stands alone (goal, relevant files,
constraints). The worker cannot see this conversation. Always read the patch
(.delegate/patches/<run_id>.patch) before running
"python3 tools/delegate/delegate.py apply <run_id>". If status is failed, read
the end of the transcript before retrying. Never delegate secrets or
architecture decisions.
```

## Safety model

What it does:
- Never modifies your repo during a run.
- Hides secrets from the worker (scrubbed environment, excluded files).
- Keeps the worker inside the temp directory (paths are resolved and checked).
- Caps steps, time and output size.

What it does not do:
- It is not a container sandbox. The worker's shell commands run on the machine that runs `delegate`, with that user's permissions. Run it inside an environment that is already disposable (a cloud dev container, a VM, a CI job), not on a machine you cannot afford to lose.
- It does not stop a worker from making a bad change that your tests do not catch. Read the patch.

## Status

Tested: the 15-check selftest, and one real end-to-end run against a Gemini model through Vertex AI's OpenAI-compatible endpoint on a trivial task. Larger and messier tasks are the next thing to learn from, and bug reports from real use are very welcome.

Known rough edges: OAuth tokens expire after about an hour and are not refreshed automatically; there is no per-step progress output yet; the worker has no network access to anything except the model API, by design, but the tool itself cannot enforce that on your machine.

## Contributing (Hacktoberfest friendly)

Small, well-scoped pull requests are the best pull requests. A few ideas, roughly from easy to adventurous:

- [ ] Per-step progress logging to stderr (step number, tools used, durations).
- [ ] Automatic token refresh for OAuth-based endpoints.
- [ ] A `--dry-run` that prints the task prompt without calling a model.
- [ ] Cost estimate per run from token usage and a user-supplied price.
- [ ] A Docker backend so worker commands run in a real container.
- [ ] Run several workers in parallel on the same task and keep the best patch.
- [ ] Windows support.
- [ ] More selftest cases (timeouts, empty patches, 429 retries).

Ground rules:
1. Keep it standard-library only.
2. Run `python3 delegate.py --selftest` before opening a PR, and add a check for what you changed.
3. One change per PR, with a short description of what you tested.

## License

See [LICENSE](LICENSE).
<!-- TODO: add a LICENSE file before publishing -->
