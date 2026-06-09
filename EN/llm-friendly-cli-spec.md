# CLI Specification for LLMs / Agents

Status: Draft 1.0
Scope: New CLI development, existing CLI retrofitting, unified specification for AI code generation and team code review.

---

## 1. Purpose

This specification defines a set of CLI behavioral requirements applicable to LLMs, IDE agents, automation scripts, MCP adapters, and CI pipelines.

A CLI conforming to this specification must simultaneously satisfy the following goals:

- **Discoverable**: Consumers can independently learn usage through `--help` and examples
- **Predictable**: The same input produces stable output across different contexts
- **Recoverable**: Errors can be corrected; dangerous operations have safety boundaries
- **Parsable**: Output can be stably machine-read
- **Contract-driven**: Structured output has versioning and compatibility rules

---

## 2. Normative Terminology

This specification uses the following terms:

- `MUST`: Mandatory requirement; non-conformance means the implementation does not comply with this specification
- `MUST NOT`: Mandatory prohibition; any occurrence means the implementation does not comply
- `SHOULD`: Strong recommendation; if not satisfied, a clear, documented reason must exist
- `SHOULD NOT`: Strong discouragement; if adopted, a clear, documented reason must exist
- `MAY`: Optional capability

Unless otherwise stated, all requirements apply to:

- Top-level commands
- Subcommands
- Structured output mode
- Non-interactive execution mode

---

## 3. Conformance Requirements

A CLI claiming "conformance to this specification":

- `MUST` satisfy all `MUST` and `MUST NOT` clauses in this document
- `SHOULD` satisfy all `SHOULD` and `SHOULD NOT` clauses; where not satisfied, the deviation rationale `MUST` be documented in the project documentation
- `MAY` independently implement optional capabilities

When teams use AI to generate CLIs, the generated output is only considered "spec-qualified" after satisfying all mandatory requirements in Sections 4 through 12.

---

## 4. Command Structure

### 4.1 Command Style Consistency

- The CLI `MUST` maintain a consistent command organization style within the same tool.
- The CLI `MAY` adopt a resource-centric structure, e.g. `mytool project create`.
- The CLI `MAY` adopt a task-centric structure, e.g. `mytool build` or `mytool get project`.
- The CLI `MUST NOT` mix multiple conflicting command mental models without a clear design rationale.

### 4.2 Hierarchy Depth

- Command hierarchy `SHOULD` not exceed 3 levels.
- If it exceeds 3 levels, the design document `MUST` clearly explain the necessity of additional levels.
- High-frequency commands `MAY` expose top-level shortcut entries, e.g. `init`, `run`.

### 4.3 Subcommand Discoverability

- Every subcommand `MUST` support `--help`.
- The top-level command `SHOULD` provide an enumerable command listing capability, e.g. `commands`.

---

## 5. Arguments and Flags

### 5.1 Short and Long Flags

- The CLI `SHOULD` provide both short flags and long flags.
- Documentation and integration guides `SHOULD` explicitly require agents to prefer long flags.
- If an option only has a long flag, it is not a violation; however, its naming `MUST` remain clear and stable.

### 5.2 Naming Conventions

Unless a sufficient and documented domain-specific reason exists, the following names `MUST` be used preferentially:

| Purpose | Canonical Name |
|---|---|
| Output path | `--output` |
| Input path | `--input` or positional argument |
| Force execution | `--force` |
| Skip confirmation | `--yes` / `-y` |
| Quiet mode | `--quiet` / `-q` |
| Detailed logging | `--verbose` / `-v` |
| Dry run | `--dry-run` |
| Config file | `--config PATH` |
| Non-interactive mode | `--non-interactive` or `--no-input` |

### 5.3 Output Format Flag

- The CLI `MUST` provide an explicit structured output switch.
- Multi-format tools `SHOULD` use `--format <value>`, e.g. `--format json`.
- Tools supporting only "default text + JSON" `MAY` use `--json`.
- The format switch within the same tool `MUST` be consistent.

### 5.4 Boolean Flags

- Boolean options requiring explicit toggling `SHOULD` support paired forms, e.g. `--cache` / `--no-cache`.
- The CLI `SHOULD NOT` use `--enable-*` / `--disable-*` extensively as the default pattern.

### 5.5 `--verbose` and `--debug`

- `--verbose` `SHOULD` represent more detailed normal operational information.
- `--debug` `MAY` exist, but its semantics `MUST` be distinct from `--verbose`.
- The CLI `MUST NOT` make `--debug` and `--verbose` express the same level of meaning.

---

## 6. Output and Structured Contract

### 6.1 Explicit Structured Output

- The CLI `MUST` support structured output.
- Structured output `MUST` be enabled via an explicit parameter.
- The CLI `MUST NOT` require consumers to infer the output format from TTY state.

### 6.2 TTY Behavior

- The default output format `MUST NOT` change based on TTY vs. non-TTY environment.
- TTY detection `MAY` be used for presentation-layer capabilities such as color, progress bars, and interactive prompts.
- TTY detection `MUST NOT` be used to automatically switch between text results and JSON results.

### 6.3 stdout / stderr Division of Labor

- `stdout` `MUST` carry the primary result.
- `stderr` `MUST` carry logs, progress, warning text, and auxiliary diagnostic information.
- In structured output mode, all machine-required data `MUST` appear on `stdout`.
- In structured output mode, `stdout` `MUST NOT` be polluted with non-contractual log text.

### 6.4 Structured Result Envelope

Snapshot-style structured output `MUST` use a stable top-level envelope. The top level must contain at minimum the following fields:

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

Field requirements:

- `schema_version` `MUST` exist, type string
- `status` `MUST` exist, allowed values: `ok`, `partial`, `error`
- `data` `MUST` exist; may be `null`, empty object, or empty array when there is no data, as defined by the command documentation
- `errors` `MUST` exist, type array
- `warnings` `MUST` exist, type array
- `meta` `MUST` exist, type object

### 6.5 Compatibility Rules

- Within the same `schema_version` major version, field semantics `MUST NOT` drift.
- New fields `MAY` be appended.
- Existing fields `MUST NOT` be silently deleted or renamed.
- Incompatible changes `MUST` be exposed via a new major version schema.
- If multiple version output formats are provided, e.g. `json-v2`, switching `MUST` be via explicit opt-in.

### 6.6 Key Field Documentation

- All key fields that upstream automation depends on `MUST` be listed in the documentation.
- When list-type results are empty, the field `MUST` return an empty array and `MUST NOT` be absent due to emptiness.

---

## 7. Error Model and Exit Codes

### 7.1 Text Errors

In non-structured mode, error messages `MUST` contain at minimum:

- The error problem
- The error cause
- A fix suggestion

Conformant example:

```text
Error: --format value 'mp4' is not supported
Reason: 'audio convert' only accepts audio formats
Hint: try one of: wav, ogg, flac, mp3
See: mytool audio convert --help
```

### 7.2 Structured Errors

In structured mode, error results `MUST` reuse the top-level envelope from Section 6.4.

Error items `MUST` contain at minimum the following fields:

```json
{
  "code": "INVALID_FORMAT",
  "message": "format 'mp4' not supported",
  "hint": "use one of: wav, ogg, flac, mp3"
}
```

Error items `MAY` append the following fields:

- `param`
- `path`
- `retryable`
- `details`

### 7.3 Spelling Correction

- For unknown subcommands and unknown flags, the CLI `SHOULD` provide a `Did you mean` hint.
- The number of candidates `SHOULD` be limited to 1–3.

### 7.4 Exit Codes

The CLI `MUST` at minimum stably support the following exit code semantics:

| Exit Code | Meaning |
|---|---|
| `0` | Complete success |
| `1` | Execution failure or partial failure |
| `2` | Invocation error |
| `130` | User or host interrupt |

Supplementary requirements:

- Invocation problems such as argument errors, unknown commands, and missing required parameters `MUST` return `2`
- Runtime failures and partial failures `MUST` return `1`
- When partial failure occurs, the exit code `MUST NOT` be `0`
- If finer-grained exit codes are implemented, they `MUST` be listed in the documentation and `MUST NOT` alter the semantics of the above 4 codes

---

## 8. Non-Interactive Execution and Dangerous Operations

### 8.1 Non-Interactive Mode

- All commands that may trigger interactive prompts `MUST` support non-interactive execution.
- The CLI `MUST` provide `--yes`, `--non-interactive`, or equivalent capability.
- In non-interactive environments, if a command requires user confirmation and the consumer has not explicitly authorized it, the CLI `MUST` fail with a clear error.
- The above error `SHOULD` return exit code `2`.
- The CLI `MUST NOT` silently select defaults and continue executing dangerous operations in non-interactive environments.

### 8.2 Destructive Operations

Deletion, overwriting, deployment, remote writes, sending data out, payments, and similar operations are considered destructive or high-risk.

Such commands:

- `MUST` be protected by default
- `MUST NOT` directly execute irreversible side effects without confirmation or explicit authorization parameters
- `MAY` meet the requirement through safer default semantics, e.g. default to recycle bin, default to plan-only

### 8.3 `--dry-run`

- For operations that can be reliably previewed — such as local file writes, bulk renames, and configuration changes — the CLI `SHOULD` support `--dry-run`
- The `--dry-run` result `MUST` present a change view semantically equivalent to the real execution
- The CLI `MUST NOT` provide a fake `--dry-run` whose semantics do not hold

### 8.4 `--plan`

- For deployment, infrastructure, and configuration-application tasks, the CLI `SHOULD` preferentially provide `--plan`
- The `--plan` output `SHOULD` reflect the diff of impending changes

### 8.5 Idempotency

- For remote side-effect commands, the CLI `SHOULD` support an idempotent retry mechanism
- If idempotency is supported, the CLI `SHOULD` provide an explicit parameter, e.g. `--idempotency-key`
- For side-effect commands that do not support idempotency, the documentation `MUST` explicitly state the retry risks

---

## 9. Batch Tasks, Streaming Output, and Partial Failure

### 9.1 Batch Results

For batch operations, the CLI `MUST` be able to express:

- Total count
- Success count
- Failure count
- Locatable per-item results or failure items

Batch results `MUST NOT` output only successful items while omitting failures.

### 9.2 Partial Failure

If a batch task has a mix of successes and failures:

- The top-level `status` `MUST` be `partial`
- The exit code `MUST` be `1`
- The result envelope `MUST` contain failure item details or locatable failure references

### 9.3 Streaming Structured Output

Long tasks, watch mode, and per-item processing flows `SHOULD` use JSON Lines or NDJSON in structured mode.

Each event object:

- `MUST` contain `event`
- `SHOULD` contain identifying information related to the event
- `MAY` contain timestamps, progress, and error information

Recommended event types:

- `start`
- `progress`
- `item`
- `warning`
- `done`

### 9.4 Streaming Output Channel Rules

- Streaming structured events `MUST` be written to `stdout`
- Progress text and logs `MUST NOT` be mixed into the structured event stream

---

## 10. Help Information and Introspection Capabilities

### 10.1 `--help`

The help page for each command `MUST` contain at minimum:

- Purpose description
- `USAGE`
- Argument descriptions
- Option descriptions
- Examples
- Exit code descriptions

Examples `MUST` cover the 2 to 3 most common invocation scenarios.

### 10.2 `--version`

- The top-level command `MUST` support `--version`
- Version output `SHOULD` be concise, stable, and easy for scripts to read

### 10.3 Supplementary Introspection Capabilities

The following capabilities are not mandatory, but `SHOULD` or `MAY` be implemented depending on tool complexity:

- `commands`: enumerate available subcommands
- `config show`: view currently effective configuration
- `doctor`: environment self-check
- `schema <cmd>`: export the structural definition of a command

---

## 11. Configuration, Environment Variables, and State

### 11.1 Configuration Priority

The CLI `SHOULD` use the following priority:

```text
CLI flag > env var > project config > user config > default
```

### 11.2 Environment Variables

- Environment variables `SHOULD` use a unified prefix, e.g. `MYTOOL_`
- Key configurations such as credentials, keys, endpoints, and log levels `SHOULD` support environment variable injection

### 11.3 Configuration Locations

The CLI `MUST` clearly state configuration file locations and override rules in the documentation. Recommended locations:

| Platform | Recommended Location |
|---|---|
| Linux | `~/.config/mytool/config.toml` |
| macOS | `~/Library/Application Support/mytool/` or consistent XDG-style directory |
| Windows | `%APPDATA%\mytool\config.toml` |
| Project-level | `./mytool.toml` |

### 11.4 Implicit State

- The CLI `MUST NOT` rely on undocumented hidden state to alter the primary semantics of a command
- If implicit state such as caches, sessions, or recently-used objects exists, its scope of influence `MUST` be documented

---

## 12. Performance Requirements

### 12.1 Startup Behavior

- `--help` and `--version` `SHOULD` avoid loading heavy dependencies
- Before argument parsing, the CLI `SHOULD NOT` perform expensive initialization unrelated to the current command

### 12.2 Performance Targets

The following are recommended targets, not mandatory upper bounds:

- `--help` and `--version` `SHOULD` be noticeably faster than business commands
- Common meta-information command cold starts `SHOULD` be kept in the ~300ms range as much as possible

### 12.3 Hot Path Optimization

If certain command classes require re-initialization, the CLI `MAY` provide:

- `warmup`
- Daemon mode
- Warm cache

---

## 13. Prohibited Patterns

The following patterns are specification-prohibited or strongly discouraged:

- Output format automatically switching based on TTY state
- Mixing log text into `stdout` in structured mode
- Error messages outputting only context-free `Error`
- Silently selecting defaults and continuing execution in non-interactive environments
- Batch tasks outputting only successful items
- Returning `0` when partial failure occurs
- Providing a fake `--dry-run` with semantically incorrect behavior
- Structured output without a version field
- Silently deleting, renaming, or changing field semantics within the same major schema version
- Relying on undocumented hidden state to alter primary behavior

---

## 14. Minimum Compliance Checklist

Implementers `MUST` confirm the following items item by item before delivery:

- [ ] Command structure style is consistent
- [ ] Every command supports `--help`
- [ ] Top level supports `--version`
- [ ] Supports explicit structured output
- [ ] Default output format does not change with TTY
- [ ] Clear division of labor between `stdout` and `stderr`
- [ ] Structured output includes `schema_version`, `status`, `data`, `errors`, `warnings`, `meta`
- [ ] Error objects contain at least `code`, `message`, `hint`
- [ ] Exit codes implement stable semantics for `0`, `1`, `2`, `130`
- [ ] Non-interactive environment is executable; will not hang on interactive prompts
- [ ] Destructive operations are protected by default
- [ ] Batch tasks can express partial failure
- [ ] Documentation clearly states configuration locations, configuration priority, and key fields

---

## 15. Execution Rules for AI Code Generation

When this specification serves as input to an AI, the caller should treat the following rules as default execution constraints:

- The AI `MUST` treat this specification as the highest-priority constraint for CLI behavior
- The AI `MUST` generate `--help`, structured output, error model, and non-interactive mode simultaneously when generating code
- The AI `MUST NOT` only implement a "happy-path" text-output CLI
- The AI `MUST` implement partial failure semantics when generating batch commands
- The AI `MUST` implement confirmation protection and non-interactive failure strategy when generating destructive commands
- The AI `SHOULD` directly adopt the Section 6.4 top-level envelope when generating result structures
- The AI `SHOULD` use Section 14 as acceptance criteria when generating review checklists

---

## 16. Conclusion

The core requirement of this specification can be summarized in one sentence:

> A CLI is not just a terminal tool; it is an interface that must be reliably consumed by both humans and agents simultaneously.

Any design that confuses humans, makes scripts fragile, or prevents agents from self-correcting has no place in a CLI built for the automation era.
