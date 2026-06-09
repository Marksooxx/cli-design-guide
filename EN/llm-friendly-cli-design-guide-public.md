# CLI Design Guide for LLMs and Agents

Command-line tools are no longer just for human terminal users. Increasingly, CLIs are invoked by LLMs, automation scripts, IDE agents, MCP adapters, and CI pipelines. For these consumers, a CLI is not merely a "runnable" entry point — it is an interface that must be discovered, understood, corrected, and reliably consumed.

A truly agent-friendly CLI is not simply one that "supports JSON output." It must simultaneously deliver the following capabilities:

- **Discoverable**: Models can quickly learn how to invoke the tool through `--help` and examples
- **Predictable**: The same input produces stable, expected behavior across different contexts
- **Recoverable**: Errors are correctable, and dangerous operations have clear safety boundaries
- **Parsable**: Output is clearly structured for reliable programmatic consumption
- **Contract-driven**: Fields, events, errors, and versions have explicit conventions that won't silently break integrations after upgrades

This document provides a set of practical guidelines for public release. The goal is not to make every CLI look the same, but to help you build command-line interfaces that work well for both humans and automated agents.

---

## 1. How to Use This Guide

The recommendations in this document fall into three categories:

- **Baseline Requirements**: Failure to meet these will significantly reduce agent usability
- **Recommended Practices**: Worth adopting for the vast majority of CLIs
- **Advanced Capabilities**: Suitable for mature tools or specific scenarios

If you are designing a new CLI, satisfy the baseline requirements first, then progressively adopt recommended practices. If you are retrofitting an existing tool, prioritize output contracts, error models, and non-interactive execution.

---

## 2. Design the Information Architecture Before the Command Word Order

CLI command word order typically follows one of two mainstream styles:

- **Resource-centric**: `noun verb`
  e.g. `mytool project create`, `mytool config get`
- **Task-centric**: `verb noun` or top-level verbs
  e.g. `docker run`, `git commit`, `kubectl get pods`

There is no universally "correct" word order in the ecosystem. The key to deciding is not to imitate a famous tool, but to answer two questions first:

- Do users and agents think in terms of "what object to act on" or "what action to perform" first?
- Is your tool organized around resources or around tasks?

Recommended practices:

- Maintain consistent style within a single CLI — don't mix resource-centric and task-centric patterns
- Keep command hierarchy to 3 levels or fewer
- High-frequency operations may have top-level shortcut commands, e.g. `init`, `run`

Deep nesting directly increases the cognitive burden on agents. Rather than stacking more intermediate nouns, a better approach is often to push deep concepts down into flags, configuration items, or input parameters.

---

## 3. Flag Design: Seek Convention Consistency First

For agents, whether flag naming follows community conventions often matters more than whether the documentation is elegant. The more a name matches expectations, the less the model needs to repeatedly consult `--help`.

Recommended practices:

- Provide both short and long flags
- In documentation and integration guides, explicitly recommend that agents use long flags
- Provide paired forms for boolean options, e.g. `--cache` / `--no-cache`

Common naming conventions:

| Purpose | Preferred Name |
|---|---|
| Output path | `--output` |
| Input path | `--input` or positional argument |
| Force execution | `--force` |
| Skip confirmation | `--yes` / `-y` |
| Quiet mode | `--quiet` / `-q` |
| Verbose output | `--verbose` / `-v` |
| Dry run | `--dry-run` |
| Config file | `--config PATH` |
| Non-interactive mode | `--non-interactive` or `--no-input` |

Regarding output format:

- If the tool supports multiple output formats, prefer `--format json|yaml|text`
- If the tool only has "default text + JSON" output, a standalone `--json` is also acceptable
- Whichever form is chosen, keep it consistent across the entire tool

Regarding `--verbose` and `--debug`:

- `--verbose` is appropriate for more detailed normal operational information
- `--debug` may exist, but it should represent additional developer diagnostic information, not be conflated with `--verbose`

---

## 4. The Core of Output Design Is Not JSON — It's the Contract

Supporting JSON output is only the first step. What truly determines whether an agent can integrate reliably is whether the output has a clear and long-term stable contract.

### 4.1 Structured Output Should Be Explicitly Activated

Baseline requirements:

- Provide structured output capability
- Let consumers explicitly select it, e.g. `--format json`
- Do not require consumers to infer the current output format from TTY state

Example:

```bash
$ mytool list
NAME       STATUS    SIZE
audio.wav  ok        2.3MB
voice.ogg  pending   1.1MB

$ mytool list --format json
{
  "schema_version": "1",
  "status": "ok",
  "data": [
    {"name": "audio.wav", "status": "ok", "size_bytes": 2411724},
    {"name": "voice.ogg", "status": "pending", "size_bytes": 1153434}
  ],
  "errors": [],
  "warnings": [],
  "meta": {}
}
```

The benefit of explicit parameter passing is stable output behavior. Whether in a terminal, CI, pipe, redirect, or agent invocation context, the same command will not silently change its output format due to different runtime contexts.

### 4.2 Default Output Must Not Drift with TTY State

Recommended practices:

- TTY detection should only control presentation-layer features such as color, progress bars, and interactive prompts
- Do not use TTY detection to automatically switch between "table" and "JSON"

If you genuinely want to offer automatic format selection, make it an explicit opt-in capability, e.g. `--auto-format`, rather than the default behavior.

### 4.3 stdout and stderr Must Have Clear Division of Labor

Baseline requirements:

- `stdout` carries the primary result
- `stderr` carries logs, progress, warnings, and auxiliary diagnostic information

In structured mode, it is recommended to treat `stdout` as the sole machine-consumption channel. This means:

- Success results go to `stdout`
- Structured error results also go to `stdout`
- Any logs and progress not part of the result contract go to `stderr`

The benefit is that consumers only need to parse one stable channel, without constantly splitting between logs and results.

### 4.4 Design a Stable Result Envelope for Structured Output

Recommended practices:

- Include `schema_version` at the top level
- Keep top-level fields stable
- Prefer appending new fields — do not directly rename or delete old fields
- Return an empty array when a list is empty, rather than omitting the field
- All key fields that automated workflows depend on must be explicitly declared in documentation

A simple yet stable result envelope is usually sufficient:

```json
{
  "schema_version": "1",
  "status": "ok",
  "data": {},
  "errors": [],
  "warnings": [],
  "meta": {}
}
```

Where:

- `status` should use `ok`, `partial`, `error`
- `data` carries the primary result
- `errors` carries structured error items
- `warnings` carries alerts that do not block execution but are worth upstream handling
- `meta` carries cursors, pagination, timing, request IDs, and other supplementary information

### 4.5 Versioning Should Be Done Early — Don't Wait Until Breaking Changes Need Patching

Recommended practices:

- Include `schema_version` in all structured output
- Agree on a compatibility policy, e.g. "adding fields is allowed; deleting fields requires a deprecation cycle"
- Major breaking changes should be explicitly enabled via a new schema version or new format option

For CLIs that will be called by scripts and agents over the long term, this is more important than whether YAML is supported.

---

## 5. The Error Model Determines Whether an Agent Can Self-Recover

The biggest problem with many CLIs is not that they fail, but that they fail in an inactionable way. For humans, "invalid input" is already frustrating; for agents, it is nearly zero-information.

### 5.1 Text Errors Must Contain at Least Three Parts

Baseline requirements:

- What the problem is
- What the cause is
- What to do next to fix it

Example:

```text
Error: --format value 'mp4' is not supported
Reason: 'audio convert' only accepts audio formats
Hint: try one of: wav, ogg, flac, mp3
See: mytool audio convert --help
```

Such errors are clear for humans and sufficiently friendly for agents, because the model can self-correct directly based on the `Hint`.

### 5.2 Structured Errors Should Be Compatible with the Result Envelope

Baseline requirements:

- In structured mode, provide machine-readable error objects
- Error objects must contain at least `code`, `message`, `hint`

Example:

```json
{
  "schema_version": "1",
  "status": "error",
  "data": null,
  "errors": [
    {
      "code": "INVALID_FORMAT",
      "message": "format 'mp4' not supported",
      "hint": "use one of: wav, ogg, flac, mp3",
      "param": "--format"
    }
  ],
  "warnings": [],
  "meta": {}
}
```

Compared to outputting only a string of text, stable error codes significantly reduce the fragility of upstream integration logic.

### 5.3 Spelling Correction Is a High-Return Small Feature

Recommended practices:

- For unknown subcommands and unknown flags, provide "Did you mean"
- Keep suggested candidates few, typically 1 to 3 is enough

Example:

```text
Error: unknown subcommand 'conver'
Did you mean: convert?
```

### 5.4 Exit Codes Should Be Simple, Stable, and Documented

Recommended practices:

- `0`: Complete success
- `1`: Execution failure or partial failure; see structured results for details
- `2`: Invocation error, e.g. wrong argument, unknown subcommand, missing required parameter
- `130`: Interrupted

If the tool needs finer-grained exit codes, they can be extended, but the high-level semantics of `1` and `2` should be kept stable. For agents, "called it wrong" vs. "it errored during execution" is the most critical distinction.

---

## 6. Safety Boundaries Must Be Real, Not Just Apparent

For agents, the most dangerous CLI is not one that is "powerful," but one that is "destructive by default with unreliable preview."

### 6.1 Destructive Commands Should Be Protected by Default

Baseline requirements:

- Operations such as deletion, overwriting, sending data out, deployment, and remote writes must have protective measures by default
- Protective measures may be confirmation prompts, explicit `--yes`, two-phase execution, or safer semantic design

For example:

```text
mytool delete file.wav
```

Should not directly perform an irreversible deletion. Better designs include:

- Default to moving to trash/recycle bin; `--hard` for real deletion
- Default to failing with a hint that `--yes` is required
- Generate a plan first, then explicitly execute it

### 6.2 Provide --dry-run Only When Semantics Hold

Recommended practices:

- Provide `--dry-run` for local file writes, bulk renames, config rewriting, and similar operations
- Prefer `--plan` for deployment-type tasks
- For remote side-effect operations where dry-run semantics cannot genuinely hold, clearly state that it is unsupported

The wrong thing to do is not "no `--dry-run`" — it is providing a fake dry-run that appears safe but is not semantically equivalent. For agents, this is more dangerous than having no preview capability at all.

### 6.3 For Retryable Side-Effect Operations, Explicitly Support Idempotency

Recommended practices:

- Clearly document which commands are naturally idempotent, e.g. `get`, `list`, `describe`
- Support `idempotency key` for side-effect commands such as sending, creating, charging, and notifying

For example:

```bash
mytool send-email --idempotency-key 6d8b7c4a-...
```

If an agent retries due to timeout or network jitter, the absence of idempotency guarantees turns "safe retry" into "duplicate execution" very easily.

### 6.4 Preserve Audit Capability for Real-World Problems

Recommended practices:

- Provide operation history or operation detail queries
- Return operation IDs, request IDs, or task IDs in the result envelope

`undo` can be a nice advanced capability, but it only applies to operations that are semantically truly reversible — it should not be packaged as a universal promise.

---

## 7. Non-Interactive Mode Must Be a First-Class Citizen

Agents have no reliable stdin, and many automation environments do not allow interactive input. A CLI that only works when "a human presses Enter in a terminal" cannot become a qualified automation interface.

Baseline requirements:

- Provide `--yes`, `--non-interactive`, or equivalent capability
- In non-interactive environments, when encountering a step that would normally require confirmation, the default must be to fail with a clear error
- Do not silently select defaults in non-interactive mode

Error examples:

- Entering a non-TTY environment and automatically selecting "yes"
- Silently skipping warnings without any indication
- Suddenly popping up a password prompt, causing the process to hang

Recommended practices:

- Credentials should preferentially support environment variables
- Support reading credentials from files or secure channels other than stdin
- In error messages, clearly indicate what parameter is needed to proceed in non-interactive mode

---

## 8. Long Tasks, Streaming Output, and Batch Semantics Require Separate Design

Many CLI output designs only consider "print a chunk of results after the command finishes," but real-world scenarios also include:

- Long-running processing tasks
- Batch operations
- Watch mode
- Partially successful, partially failed execution results

If these scenarios are forced into one-shot text output, the agent experience rapidly deteriorates.

### 8.1 Prefer JSON Lines or NDJSON for Long Tasks

Recommended practices:

- Use plain JSON for snapshot queries
- Use JSON Lines or NDJSON for streaming tasks
- Each event must contain a stable `event` field

Example:

```text
{"event":"start","total":100}
{"event":"progress","done":10,"current":"file_010.wav"}
{"event":"item","id":"file_010.wav","status":"ok"}
{"event":"item","id":"file_011.wav","status":"error","error":{"code":"TIMEOUT","message":"..."}}
{"event":"done","summary":{"total":100,"ok":97,"failed":3}}
```

This way agents can incrementally consume results, rather than waiting for a massive array to be returned all at once at the end.

### 8.2 Batch Operations Must Explicitly Express Partial Failures

Baseline requirements:

- Batch tasks must not return only successful items
- Must be able to distinguish total success, total failure, and partial success
- Both the exit code and the result envelope must reflect the fact of failure

Recommended result envelope shape:

```json
{
  "schema_version": "1",
  "status": "partial",
  "data": {
    "summary": {"total": 100, "ok": 97, "failed": 3},
    "results": [
      {"id": "file_001", "status": "ok"},
      {
        "id": "file_042",
        "status": "error",
        "error": {
          "code": "TIMEOUT",
          "message": "upload timed out"
        }
      }
    ]
  },
  "errors": [],
  "warnings": [],
  "meta": {}
}
```

If a batch task encounters partial failure, the exit code must at minimum not present as "complete success."

### 8.3 Long Tasks Should Support Interruption and Resumption

Recommended practices:

- On interruption, exit gracefully and clean up temporary state
- Resumable tasks should support `--resume`
- Expose checkpoints, task IDs, or context needed for resumption in the output

---

## 9. `--help` Is Part of the Interface, Not a Bonus Manual

For agents, `--help` is often the first contact with your CLI. How well the help text is written directly affects whether the first invocation succeeds.

Baseline requirements:

- Every subcommand has help text
- Help text includes at minimum: purpose, argument descriptions, examples, exit code descriptions
- Examples cover the 2 to 3 most common usage patterns

A usable help page should generally contain:

```text
USAGE:
    mytool audio convert <INPUT> [OPTIONS]

ARGUMENTS:
    <INPUT>    Input file or directory

OPTIONS:
    -o, --output <PATH>     Output path
    -f, --format <FORMAT>   Target format
        --dry-run           Preview without writing
        --yes               Skip confirmation prompts

EXAMPLES:
    mytool audio convert in.wav -f ogg
    mytool audio convert ./input -f ogg --dry-run

EXIT CODES:
    0  success
    1  execution failed or partially failed
    2  invalid invocation
```

Recommended practices:

- Provide a `commands` or equivalent enumerable interface for the command tree
- Provide `config show`
- Provide `doctor` for complex tools
- For mature tools, provide `schema <cmd>` to export command structure definitions

These are not necessarily baseline requirements, but they significantly improve agent exploration efficiency.

---

## 10. Configuration and State Management Must Be Automation-Friendly

A CLI that relies on implicit local state can hardly be a reliable agent tool.

Recommended practices:

- Clearly define configuration priority: `CLI flag > env var > project config > user config > default`
- Use a unified prefix for environment variables, e.g. `MYTOOL_`
- Support project-level configuration overriding user-level configuration

Configuration locations should follow platform conventions and be clearly stated in documentation:

| Platform | Recommended Location |
|---|---|
| Linux | `~/.config/mytool/config.toml` |
| macOS | `~/Library/Application Support/mytool/` or consistent XDG-style directory |
| Windows | `%APPDATA%\mytool\config.toml` |
| Project-level | `./mytool.toml` |

More important than insisting on a single correct path is maintaining consistency.

Beyond baseline requirements, an important principle is: **minimize reliance on hidden state**. Context that can be expressed through explicit parameters should not be hidden away in a local cache, the result of the last run, or an implicit session.

---

## 11. Startup Performance Gets Amplified in Agent Loops

A single slow startup might be tolerable for humans; but in an agent loop, after dozens of invocations stack up, the experience rapidly degrades.

Recommended practices:

- Make `--help` and meta-information commands as fast as possible
- Avoid loading heavy dependencies before parsing arguments
- Use lazy imports or layered initialization

Empirically:

- `--help` and `--version` should be noticeably faster than business commands
- If certain types of tasks require lengthy initialization, consider `warmup` or a background daemon mode

Don't let a call that only wants to check help text end up launching a model, connecting to a database, or scanning the entire project directory on the way.

---

## 12. When Writing Integration Notes for Agents, Keep It Lean and Precise

When the CLI itself is sufficiently self-describing, the accompanying agent integration notes don't need to be thick. Whether they end up in a `SKILL.md`, tool metadata, system prompt, or MCP description, the key points are usually just a few:

- Start exploring with `--help`
- Prefer long flags
- Always explicitly request structured format for output that needs parsing
- Non-interactive environments must pass `--yes` or `--non-interactive`
- Use `--dry-run` first for local destructive operations; focus on idempotency and staging for remote side-effect operations

A good CLI should make the integration notes read more like "calling conventions" than "external documentation patching up the CLI's own shortcomings."

---

## 13. Design Checklist

### Baseline Requirements

- [ ] Command naming style is consistent within the tool
- [ ] Supports explicit structured output
- [ ] Default output format does not drift with TTY state
- [ ] Clear division of labor between `stdout` and `stderr`
- [ ] Error messages include problem, cause, and suggestion
- [ ] Structured errors include at least `code`, `message`, `hint`
- [ ] Exit codes at least stably distinguish invocation errors from execution errors
- [ ] Destructive commands are protected by default
- [ ] Supports non-interactive execution
- [ ] `--help` includes examples
- [ ] Batch operations can express partial failure

### Recommended Practices

- [ ] Dual short/long flag system
- [ ] Long flags have clear semantics and follow community conventions
- [ ] Structured output includes `schema_version`
- [ ] Long tasks support JSON Lines or NDJSON
- [ ] Provides spelling correction
- [ ] Command hierarchy does not exceed 3 levels
- [ ] Configuration priority is clear
- [ ] Credentials support environment variables
- [ ] Side-effect commands support idempotent retry mechanism
- [ ] Local destructive operations support genuinely reliable `--dry-run`
- [ ] Long tasks support graceful interruption
- [ ] `--help` and meta-information commands start fast
- [ ] Provides operation history, request IDs, or task IDs

### Advanced Capabilities

- [ ] `doctor` environment self-check
- [ ] `schema <cmd>` export structure definitions
- [ ] Offline documentation query
- [ ] `--resume` for long tasks
- [ ] `warmup` or daemon mode
- [ ] `undo` for reversible operations
- [ ] `--plan` for deployment-type tasks

---

## 14. Common Anti-Patterns

The following practices make a CLI fragile, dangerous, or difficult to integrate in agent scenarios:

- Output format automatically changes based on TTY state
- Logs, progress, and results are mixed into the same channel
- Error messages are just one line of `Error`
- Silently selecting defaults in non-interactive environments
- Default behavior is destructive
- Providing a fake `--dry-run` with semantically incorrect behavior
- Batch tasks only output successful items, omitting failures
- Structured output has no version field
- Output fields frequently drift or get renamed without a compatibility policy
- Startup loads many dependencies unrelated to the current command
- Heavy reliance on implicit local state, causing the same command to behave differently on different machines

---

## 15. Conclusion

The best LLM-friendly CLI is not one that "looks like an API" — it is one that "possesses interface quality in itself."

It should do three things:

- One glance at `--help` and you can start calling it correctly
- One glance at an error message and you know what to change next
- One glance at structured output and you can reliably integrate into automation systems

When these capabilities genuinely land in the CLI itself, agent-facing integration documentation will naturally become thinner, and automation pipelines will become noticeably more stable. At that point, the CLI is no longer just a terminal tool — it is an interface that is friendly to both humans and machines.
