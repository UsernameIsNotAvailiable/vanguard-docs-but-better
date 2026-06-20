# Roadmap

This roadmap describes Vanguard's intended direction after `0.1.14`. It exists
to make priorities, dependencies, and completion criteria visible without
turning exploratory work into a release-date promise.

!!! info "Release line"
    **Current:** Vanguard `0.1.14`, network protocol `1`  
    **Next:** Vanguard `0.1.15`, centered on the Plugin Developer API

## Status Legend

| Status | Meaning |
| --- | --- |
| **Current** | Available in the latest Vanguard package |
| **Next** | Planned for the next package release |
| **Planned** | Accepted direction, sequenced after the next release |
| **Exploring** | Requirements and design are still being evaluated |
| **Ongoing** | Work continues across multiple releases |

Roadmap status is not a publication date. Security fixes, Roblox platform
changes, and compatibility work may change sequencing.

## Release Overview

| Milestone | Status | Primary outcome |
| --- | --- | --- |
| `0.1.14` | Current | Linked errors, Math, Switch, access-controlled classes, and protocol 1 documentation |
| `0.1.15` | Next | Stable Plugin Developer API and extension lifecycle |
| Toolbox distribution | Planned | Verified non-Wally installation path |
| Additional utilities | Exploring | More production-focused, dependency-free helpers |
| Studio tooling | Exploring | Generation, validation, inspection, and diagnostics |
| Documentation expansion | Ongoing | Recipes, examples, architecture, and versioned guidance |
| Community contributions | Ongoing | Clear contribution, review, and RFC process |

## 0.1.15: Plugin Developer API

The Plugin Developer API is the next release focus. Its purpose is to let
extensions integrate with Vanguard through public contracts instead of
requiring internal registries, private tables, or bootstrap implementation
details.

### Goals

- Register named, versioned extensions with explicit runtime scope.
- Give plugins deterministic initialization and startup hooks.
- Resolve declared plugin dependencies before lifecycle execution.
- Expose narrow capabilities for framework integration.
- Keep service, controller, component, class, and network ownership explicit.
- Produce linked errors and diagnostics for plugin failures.
- Preserve Vanguard startup guarantees when one optional plugin fails.
- Make plugin state inspectable by future Studio tooling.

### Proposed Manifest Shape

The final API may change during implementation. This illustrates the intended
contract, not a currently available function:

```lua
Vanguard.RegisterPlugin({
	Name = "ExamplePlugin",
	Version = "1.0.0",
	Runtime = "Shared",
	Dependencies = {
		MetricsPlugin = ">=1.0.0",
	},

	Init = function(context)
		-- Register capabilities and perform required setup.
	end,

	Start = function(context)
		-- Begin runtime work after required plugin init completes.
	end,
})
```

Expected context capabilities include scoped logging, runtime metadata,
read-only framework version and protocol information, plugin dependency lookup,
cleanup ownership, and deliberately limited registration helpers.

### Lifecycle Direction

The planned lifecycle is:

```text
discover plugin manifests
  -> validate names, versions, runtime, and dependencies
  -> build dependency order
  -> initialize required plugins
  -> initialize Vanguard services/controllers
  -> schedule plugin and framework start hooks
  -> expose readiness and diagnostics
```

Exact ordering between plugin and framework hooks will be finalized with tests
and documented before release. Circular dependencies and missing required
plugins must fail with actionable linked errors.

### Extension Boundaries

The API is intended to support:

- reusable framework integrations;
- custom diagnostics and metrics;
- domain-specific registries and factories;
- opt-in service, controller, component, or class discovery helpers;
- future Studio tools that inspect public plugin metadata;
- protocol-aware network extensions that declare compatibility explicitly.

The API is not intended to expose every internal Vanguard table. Plugins should
receive capabilities they need rather than mutable access to framework state.

### Non-Goals For 0.1.15

- A hosted plugin marketplace.
- Downloading or executing remote code at runtime.
- Automatic trust of third-party plugins.
- A breaking network protocol revision solely for local plugin registration.
- A complete visual Studio tool suite in the same release.

### Definition Of Done

`0.1.15` is ready when:

- [ ] plugin manifests have validated names, versions, and runtime scope;
- [ ] registration and lookup APIs are public and typed;
- [ ] dependencies have deterministic ordering and cycle diagnostics;
- [ ] plugin init failures follow documented required/optional behavior;
- [ ] plugin start failures are isolated and logged with stable error codes;
- [ ] plugin-owned cleanup has a defined lifecycle;
- [ ] server and client registries remain separate where required;
- [ ] protocol compatibility rules are explicit;
- [ ] unit tests cover ordering, missing dependencies, cycles, and failures;
- [ ] API reference, migration guidance, examples, and troubleshooting ship
      with the release.

## Roblox Toolbox Release

**Status:** Planned after the Plugin Developer API contract stabilizes.

The Toolbox release will provide a supported installation route for projects
that do not use Wally and Rojo.

### Planned Work

- Publish a clearly owned Vanguard model under the correct creator identity.
- Keep Toolbox and Wally package versions visibly aligned.
- Document manual installation and update procedures.
- Preserve the same package-root require API where Roblox permits it.
- Include a version and protocol diagnostic users can verify in Studio.
- Document how to avoid duplicate Wally and Toolbox copies in one game.
- Add release verification steps for model contents and unexpected scripts.

### Acceptance Criteria

- The model is reproducibly built from tagged source.
- A clean baseplate can install and bootstrap without Wally.
- Server and client use the same package copy.
- Update and removal instructions are tested.
- Toolbox-specific troubleshooting is published before announcement.

## Additional Utility Modules

**Status:** Exploring.

New utilities must solve recurring production problems, remain dependency-free,
have deterministic tests, and avoid duplicating simple Roblox APIs without a
clear ergonomic or correctness gain.

Candidate areas include:

| Candidate | Problem it may solve |
| --- | --- |
| `StateMachine` | Explicit states, guarded transitions, transition events, and history |
| `Queue` / `PriorityQueue` | Ordered jobs, matchmaking, task scheduling, and bounded work |
| `Observable` / `Computed` | Derived local state with deterministic subscriptions and cleanup |
| `Retry` | Backoff, attempt limits, jitter, and Promise integration |
| `Serializer` | Versioned application data transforms, not a replacement for Roblox transport |
| `Pool` | Reuse of expensive local objects with explicit reset and capacity behavior |

Candidate names and APIs are not final. Each accepted utility needs:

- a focused use case and non-goals;
- typed public API;
- cleanup and error behavior;
- custom clock or scheduler injection when timing is involved;
- unit tests for edge cases;
- a complete guide and API table.

## Expanded Documentation

**Status:** Ongoing.

Documentation work continues alongside every release rather than waiting for a
single documentation milestone.

Planned improvements include:

- task-oriented recipes for profiles, inventory, matchmaking, UI, and security;
- complete sample server and client projects;
- extension-author guidance for the `0.1.15` Plugin Developer API;
- architecture diagrams for startup, registries, networking, and cleanup;
- versioned migration notes and compatibility tables;
- more copy-paste examples with expected output and failure cases;
- release checklists that keep source, Wally, Toolbox, and docs aligned;
- searchable error entries for every new public failure code.

Documentation is considered part of feature completion, not follow-up work.

## Studio Tooling

**Status:** Exploring; expected to build on public plugin metadata.

Potential tools include:

- service, controller, component, and class generators;
- bootstrap and folder-layout validation;
- duplicate-name and missing-module diagnostics;
- service-priority and plugin-dependency visualization;
- network rule and remote hierarchy inspection;
- protocol and package-version mismatch diagnostics;
- error-code lookup inside Studio;
- safe project health checks that do not mutate game code automatically.

Studio tooling should consume public Vanguard and Plugin Developer APIs. It
should not depend on unstable internals that force lockstep releases.

Before a tool changes project files, it should preview the affected paths and
make ownership clear. Read-only inspection is the preferred first milestone.

## Community Contributions

**Status:** Ongoing.

Vanguard will make contribution boundaries easier to understand so outside work
can be reviewed consistently.

Planned repository support includes:

- a contribution guide with local test and build commands;
- issue templates for bugs, features, documentation, and security reports;
- a lightweight RFC process for public API and protocol changes;
- labeled starter issues with narrow ownership;
- pull-request checklists for tests, types, docs, and changelog updates;
- a code of conduct and maintainer response expectations;
- explicit security-reporting instructions outside public issue threads.

Protocol, authentication, and serialization changes require deeper review than
isolated utilities or documentation fixes.

## Prioritization Rules

Roadmap items are ordered using these principles:

1. protect compatibility and server authority;
2. fix correctness and diagnostics before adding surface area;
3. stabilize public extension contracts before building tools on them;
4. require tests and documentation for every public API;
5. prefer narrow utilities over overlapping mini-frameworks;
6. keep Wally and future Toolbox distributions behaviorally aligned.

## Suggesting Roadmap Work

Feature proposals should identify:

- the production problem and affected users;
- why existing Roblox or Vanguard APIs are insufficient;
- the smallest useful public contract;
- server/client and protocol implications;
- cleanup, failure, and security behavior;
- expected tests and documentation.

Community interest informs priority, but it does not override compatibility,
security, or maintenance cost.

See the [Changelog](../changelog/index.md) for shipped behavior and the
[API Reference](../api-reference/index.md) for APIs available today.
