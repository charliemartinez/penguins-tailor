# GUI integration work

## Status

This document lists the changes needed for `penguins-tailor` to integrate with
the independent `penguins-gui` application.

The JSON commands and flags below are proposed and are not current public
features. Existing commands remain the source of truth until the structured
interface is implemented and tested.

The common request and event contract is defined by
`penguins-gui/PROTOCOL.md` and its JSON Schemas.

## Existing foundation

Tailor already provides the public operations needed for a first integration:

```text
tailor version
tailor get
tailor list
tailor show <costume>
tailor wear <costume>
tailor wear <costume> --dry-run
tailor wear <costume> --linear
```

It also already has internal costume types, a dry-run path, linear output,
structured reports and technical logging. The integration should expose these
existing facts through a stable machine interface rather than make the GUI
read atelier YAML or parse formatted terminal output.

## Boundary

Tailor remains the sole owner of:

- locating and updating ateliers;
- reading and validating costume/accessory definitions;
- distribution compatibility decisions;
- package and repository changes;
- sequences, branding, preseed and finalization;
- dry-run planning and application reports.

The GUI sends stable identifiers and options, then presents what Tailor
returns. It never opens Tailor's private directories or interprets wardrobe
files directly.

## First useful delivery: read-only JSON

Before executing `wear`, expose the data needed to build the GUI safely.

### Capability discovery

A proposed read-only command is:

```text
tailor capabilities --json
```

It should report:

- Tailor and supported protocol versions;
- available operations;
- support for dry-run, NDJSON events and cancellation;
- privilege requirements per operation;
- supported atelier/catalogue features.

### Atelier status

Return a finite JSON document containing:

- local atelier path;
- configured remote URL and branch;
- current revision when available;
- whether the atelier is present and readable;
- whether an update is available only when this has actually been checked;
- warnings or errors.

Do not perform network access merely to render the main window. Synchronization
must remain an explicit user action.

### Costume catalogue

Add a JSON form of `tailor list` that returns stable costume identifiers plus
summary data:

- identifier and display name;
- version and description;
- supported or rejected status for the current distribution;
- reason when incompatible;
- source atelier and branch.

Add a JSON form of `tailor show <costume>` for details already understood by
Tailor, such as declared packages, accessories, repositories, branding and
sequence summaries. Sensitive values and arbitrary script bodies should not be
sent merely to populate the GUI.

The exact CLI spelling can be `--json` on the existing commands; no new command
is required if that keeps the interface simpler.

## Dry-run plan

`tailor wear <costume> --dry-run` already exists. Its machine form should
return a schema-valid plan instead of formatted text.

A proposed request flow is:

```text
tailor wear --check --request - --json
```

or an equivalent `--json` extension to the current dry-run command. The plan
should include:

- resolved costume and atelier identity;
- detected distribution and compatibility;
- packages to install, remove, keep or skip;
- repositories to add or change;
- accessories in execution order;
- sequence/finalization step summaries;
- branding/preseed changes;
- warnings and blocking errors;
- normalized options.

The plan must not change repositories, package state, files or services.

## Structured execution

### Proposed invocation

One possible form is:

```text
tailor wear --request - --events ndjson
```

It reads a JSON request from stdin and emits protocol documents only on stdout.
The same internal `Wear` implementation used by the human CLI performs the
operation.

### Event sources

The current execution path can expose events at natural boundaries:

- request accepted and compatibility checked;
- repositories being configured;
- package phase started and package outcomes;
- each accessory started/completed;
- each sequence step started/completed;
- branding, preseed and finalization phases;
- terminal report or error.

Use stable identifiers and explicit indexes. Terminal formatting, spinner
frames, split-screen rendering and ANSI colours must not enter protocol stdout.

An event retains a concise human `message`, for example:

```json
{
  "tool": "tailor",
  "operation": "wear",
  "state": "running",
  "step": {
    "id": "install-packages",
    "index": 3,
    "total": 8
  },
  "message": "Installing packages"
}
```

The real event also contains the common protocol, request, run, sequence,
timestamp and tool-version fields.

## Final report

On success Tailor emits one terminal result. It normally has no ISO artifact,
so domain information belongs in `data`, for example:

- costume and version applied;
- package counts and outcomes;
- accessories completed or skipped;
- sequence steps completed or skipped;
- warnings;
- whether a reboot or logout is recommended;
- path to a generated report, if one exists.

The GUI should be able to offer Eggs remastering as a separate next operation,
but Tailor must not invoke Eggs as part of this contract.

## Request fields

The minimum request maps directly to current behaviour:

| Request field | Existing Tailor concept |
| --- | --- |
| `costume` | positional argument of `tailor wear` |
| `branch` | `--branch` |
| `dry_run` | `--dry-run` |

The earlier design also contains optional `add` and `remove` package arrays.
These remain future fields until Tailor deliberately supports them as public
semantics. The GUI must not emulate them by calling the system package manager.

Unknown request fields and invalid combinations must fail validation rather
than being ignored.

## Errors

Begin with codes derived from real Tailor failures:

- `INVALID_REQUEST`;
- `UNSUPPORTED_PROTOCOL`;
- `ATELIER_NOT_FOUND`;
- `ATELIER_INVALID`;
- `COSTUME_NOT_FOUND`;
- `COSTUME_INVALID`;
- `INCOMPATIBLE_DISTRIBUTION`;
- `PRIVILEGES_REQUIRED`;
- `REPOSITORY_FAILED`;
- `PACKAGE_FAILED`;
- `ACCESSORY_FAILED`;
- `SEQUENCE_FAILED`;
- `CANCELLED`.

Include structured details such as the affected package or step when safe.
Preserve a non-zero process exit for terminal failures.

## Logging rules

Human mode keeps the existing split-screen/linear presentation and technical
log. Protocol mode follows stricter stream rules:

- stdout contains JSON/NDJSON only;
- each event is flushed immediately;
- protocol errors are emitted as terminal error documents when possible;
- unexpected diagnostics go to stderr;
- raw subprocess output is not copied to protocol stdout;
- secrets and repository credentials are always redacted.

The event emitter should receive semantic events from the wear/report logic,
not scrape Tailor's own log.

## Cancellation

Cancellation must respect package-manager and script safety. Capabilities may
initially report `supports_cancellation: false`. It should become true only
after Tailor can stop at known boundaries, finish necessary cleanup and emit a
truthful final state.

## Suggested implementation steps

Each item is intended as a small Codex task:

1. add shared protocol Go types and fixture tests;
2. add `--json` to `version`, `list` and `show` without changing defaults;
3. expose atelier status as a read-only JSON document;
4. convert the existing dry-run report into a serializable plan type;
5. add an NDJSON writer with sequence numbers and flushing;
6. introduce semantic event callbacks in `Wear` phases;
7. emit package/accessory/sequence outcomes in a terminal result;
8. map current errors to stable codes;
9. add cancellation only at verified safe boundaries;
10. integrate the Tailor adapter in `penguins-gui`.

## Acceptance test

The initial Tailor integration is complete when:

1. the GUI detects Tailor independently from Eggs;
2. it obtains a schema-valid costume catalogue without reading atelier files;
3. it shows a dry-run plan that makes no system changes;
4. a confirmed `wear` emits schema-valid NDJSON with no terminal formatting;
5. the final report distinguishes applied, skipped and failed work;
6. ordinary `tailor list`, `show` and `wear` retain their current human output;
7. Tailor remains independently usable and never needs the GUI installed.

