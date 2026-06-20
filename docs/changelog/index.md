# Changelog

This page records user-visible Vanguard changes, upgrade impact, compatibility
boundaries, and migration guidance. Dates use `YYYY-MM-DD`; package versions
follow semantic versioning.

!!! abstract "Current release"
    **Vanguard 0.1.14** was released on **2026-06-19**. It uses network
    protocol **1** and is the current documented package release.

## Release Index

| Version | Date | Status | Network protocol | Focus |
| --- | --- | --- | ---: | --- |
| [`0.1.15`](#0115-unreleased) | Unreleased | Next | 1 planned | Plugin Developer API and extension lifecycle |
| [`0.1.14`](#0114-2026-06-19) | 2026-06-19 | Current | 1 | Linked errors, Math, Switch, and class access groups |
| [`0.1.13`](#0113-2026-06-19) | 2026-06-19 | Previous | 1 | Resilient startup, classes, utilities, and server-authoritative networking |
| [`0.1.10`](#0110-date-not-recorded) | Not recorded | Historical boundary | 1 | Bound `RemoteProperty` method compatibility |

## Unreleased

The next planned package is `0.1.15`, centered on the Plugin Developer API.
Planned work is not available until it moves into a released section.

See the [Roadmap](../roadmap/index.md) for scope, sequencing, and completion
criteria.

## 0.1.15 - Unreleased

### Planned Focus

`0.1.15` is planned to introduce a stable Plugin Developer API with:

- named, versioned plugin manifests;
- explicit server, client, or shared runtime scope;
- deterministic plugin dependency ordering;
- initialization, startup, failure, and cleanup contracts;
- narrow extension capabilities instead of mutable internal access;
- typed registration and lookup APIs;
- linked plugin diagnostics and test coverage;
- extension-author documentation and migration guidance.

The target remains network protocol `1` unless the final API changes replicated
hierarchy or transport semantics. No Plugin Developer API is part of `0.1.14`.

Detailed acceptance criteria live in the
[0.1.15 roadmap milestone](../roadmap/index.md#0115-plugin-developer-api).

## 0.1.14 - 2026-06-19

### Release Summary

`0.1.14` focuses on diagnostics and expressive local application code. It adds
stable linked errors, gameplay-oriented Math helpers, explicit Switch dispatch,
and access-controlled member groups for Vanguard classes.

The network protocol remains `1`. These features change local framework
behavior and error text without changing replicated hierarchy, argument order,
Player identity, guard timing, or remote-property transport.

### Upgrade Impact

| Area | Compatibility | Action |
| --- | --- | --- |
| Existing classes | Legacy top-level constructors, fields, methods, inheritance, and `IsA` behavior remain supported. | Adopt `Public`, `Private`, and `Static` only where they clarify ownership. |
| Error handling | Framework errors retain their specific message and now add a code and docs URL. | Update log parsing that assumes a single-line message. |
| Network rejections | Names such as `INVALID_PAYLOAD` remain stable; function/property errors add a catalog URL. | Continue using rejection names for metrics and policy. |
| Network transport | Protocol remains `1`; `0.1.13` and `0.1.14` are protocol-compatible. | Prefer the same package on both runtimes despite protocol compatibility. |
| Utilities | New helpers are additive and dependency-free. | Require through `Vanguard.Util` or use `Vanguard.Math`, `Vanguard.Switch`, and `Vanguard.Error`. |

### Added

#### Linked Error Catalog

Framework-owned failures now use stable identifiers such as:

- `VG-LIFE-001` for lifecycle state and hook failures;
- `VG-MODULE-001` for isolated module discovery failures;
- `VG-REG-001` and `VG-REG-002` for registry conflicts and misses;
- `VG-CLASS-001` and `VG-CLASS-002` for class definitions and member conflicts;
- `VG-NET-001` through `VG-NET-105` for protocol, configuration, and request rejections;
- `VG-MATH-001`, `VG-MATH-002`, `VG-SWITCH-001`, and `VG-UTIL-001` for helpers.

Every formatted failure includes a direct URL to its entry in the
[Error Reference](../errors/index.md). `Vanguard.Error` exposes constructors,
formatting, throwing, assertion, catalog lookup, and network-rejection mapping.

```text
[Vanguard VG-NET-001] Vanguard network protocol mismatch (client 1, server 2)
Docs: https://twrblxdevs.github.io/vanguard-docs/errors/#vg-net-001
```

#### Math Utility

Added `Vanguard.Math` and `Vanguard.Util.Math` with:

- decimal `round` and increment-based `snap`;
- `lerp`, `inverseLerp`, and optional-clamp `map`;
- relative `approximatelyEqual` comparison;
- `wrap` and `pingPong` repetition;
- `smoothstep` and `smootherstep` easing;
- `moveTowards` and variadic `average`;
- `EPSILON` and `TAU` constants.

Invalid arguments and ranges use dedicated linked error codes. See
[Math](../utilities/math/index.md).

#### Switch Utility

Added builder-based JavaScript-style case dispatch without fall-through:

```lua
local result = Vanguard.Switch.new(state)
	:Case("Paused", "Playing")
	:Cases({ "Playing", "Resuming" }, handler)
	:Default("Idle")
	:Run(...)
```

`Switch.match(value, cases, defaultResult, ...)` provides concise direct-map
dispatch. Handlers receive the selected value plus arguments passed to `Run` or
`match`. See [Switch](../utilities/switch/index.md).

#### Public, Private, And Static Class Members

Class definitions may now group members by access and ownership:

- `Public` defines instance-visible members. Functions receive
  `(self, private, ...arguments)`.
- `Private` defines per-instance hidden defaults and private methods.
- `Private.Constructor` initializes hidden state from construction arguments.
- `Static` defines class-only members excluded from instance lookup.
- `Class:SetStatic(name, value)` adds or replaces a class-only member later.

Private defaults are deep-cloned per instance. Base and child private states are
separate; inherited public wrappers receive the private state of the class that
declared them. Vanguard rejects cross-category and inherited public/static name
conflicts with `VG-CLASS-002`.

Legacy root-level members remain public and use their original signatures. See
[Classes](../classes/index.md).

#### Protocol 1 Specification

Added a dedicated [Network Protocol 1](../network-protocol/index.md) reference
covering:

- `_VanguardRemotes` hierarchy and attributes;
- remote publication timing and client discovery;
- function, signal, and property call semantics;
- the six-stage guard order and Player identity;
- rejection transport and server-only details;
- serialization and delivery boundaries;
- package compatibility and protocol-bump criteria.

### Changed

- Server and client framework assertions now produce linked Vanguard errors.
- Utility argument and lifecycle assertions use `VG-UTIL-001` or a dedicated
  utility code.
- Module discovery logs preserve the original cause under `VG-MODULE-001`.
- `VanguardInit` failures are wrapped with the object name and `VG-LIFE-001`.
- Asynchronous `VanguardStart` failures are caught and logged with a docs link
  instead of surfacing as unclassified task errors.
- Function and property network rejections append the matching error-catalog
  URL while preserving their existing `[VanguardNetwork/CODE]` prefix.
- `Vanguard.Error`, `Vanguard.Math`, and `Vanguard.Switch` are direct aliases of
  their utility modules for concise access.
- Main-module Luau exports now include error, Math, Switch, and expanded class
  definition types.

### Fixed

- Private nested-table defaults no longer risk being shared between instances;
  each construction receives a deep-cloned state graph.
- Static members no longer leak through instance `__index` lookup.
- Inherited public methods retain access to their owning base class's private
  state rather than a child's unrelated hidden representation.

### Upgrade Checklist

- [ ] Update to `twrblxdevs/vanguard@0.1.14`.
- [ ] Run `wally install` in the game project and restart the Studio session.
- [ ] Confirm both runtimes report network protocol `1`.
- [ ] Check log ingestion for multi-line framework errors containing `Docs:`.
- [ ] Keep metrics keyed by network rejection names, not full error strings.
- [ ] Add access groups incrementally; existing classes require no rewrite.
- [ ] Review new Math and Switch helpers before maintaining duplicate local
      implementations.

## 0.1.13 - 2026-06-19

### Release Summary

`0.1.13` expands Vanguard from a service-and-controller bootstrapper into a
more complete application framework. The release focuses on four practical
problems in production Roblox games:

1. one broken ModuleScript should not prevent unrelated systems from loading;
2. service dependencies need deterministic initialization order;
3. reusable domain objects and asynchronous helpers should not require a
   second framework;
4. every inbound remote needs a consistent, server-owned trust boundary.

The existing service, controller, component, and remote APIs remain available.
Most projects can upgrade without rewriting definitions, then adopt priority,
classes, utilities, and network rules incrementally.

### Upgrade Impact

| Area | Existing behavior after upgrade | Recommended action |
| --- | --- | --- |
| Package installation | Games continue using the Wally alias already declared by the project. | Update the version and run `wally install` in the **game project**. |
| Module discovery | A failed discovered module is logged and skipped while other modules continue loading. | Treat loader warnings as real missing dependencies even when startup continues. |
| Service initialization | Services default to priority `0`; equal-priority init hooks run concurrently. | Assign distinct priorities only when one init phase must finish before another. |
| Start hooks | Start hooks remain asynchronous and are not awaited by `Start` or `OnStart`. | Keep required readiness work in `VanguardInit`. |
| Network rules | Remotes without configured rules preserve their existing application behavior. | Add validation and rate limits first to mutable or expensive remotes. |
| Client/server compatibility | Protocol mismatches stop proxy construction; package-version mismatches warn. | Ensure both runtimes require the same installed package. |
| Remote properties | Bound methods support both dot and colon calls. | No code change is required for `Property.Set(value)` or `Property:Set(value)`. |

### Added

#### Resilient Module Discovery

Folder loaders now isolate each ModuleScript require used by:

- `AddServices` and `AddServicesDeep`;
- `AddControllers` and `AddControllersDeep`;
- `AddComponents` and `AddComponentsDeep`;
- `AddClasses` and `AddClassesDeep`.

If a module throws, returns no value, or returns multiple values, Vanguard logs
its full path and continues scanning the folder. Valid sibling modules still
register and can start normally.

This is failure isolation, not silent recovery. A skipped service, controller,
component, or class is absent from its registry. Any later code that requires
that object can still fail and should surface the missing dependency directly.

See [Lifecycle: Registration Phase](../lifecycle/index.md#registration-phase)
and [Troubleshooting: module loading](../troubleshooting/index.md#module-code-did-not-return-exactly-one-value).

#### Deterministic Service Priority

Server services accept a numeric `Priority`. Higher values initialize first.
Services with equal priority initialize concurrently, with alphabetical names
providing deterministic scheduling inside each group.

```lua
local DatabaseService = Vanguard.CreateService({
	Name = "DatabaseService",
	Priority = 100,
	Client = {},
})

local ProfileService = Vanguard.CreateService({
	Name = "ProfileService",
	Priority = 50,
	Client = {},
})
```

The server lifecycle for these services is:

```text
DatabaseService.VanguardInit completes
  -> ProfileService.VanguardInit completes
  -> all VanguardStart hooks are scheduled
  -> components start
  -> Vanguard reports ready
```

Priority controls init completion order. It does not delay registration: every
service is already in the registry before the first init hook runs. Start hooks
are scheduled in priority order but are not awaited.

Read the complete [lifecycle ordering contract](../lifecycle/index.md#server-initialization).

#### Registered Classes

Vanguard now includes independent server and client class registries plus a
standalone class utility. Classes support:

- `Constructor` functions and callable class tables;
- base-first constructor execution across inheritance chains;
- inherited methods, static values, and supported metamethods;
- `IsA` checks by class object or registered name;
- `CreateClass`, `RegisterClass`, `GetClass`, `HasClass`, `GetClasses`, and
  `UnregisterClass`;
- folder discovery through `AddClasses` and `AddClassesDeep`;
- automatic class loading before services or controllers during `Bootstrap`.

```lua
local Entity = Vanguard.CreateClass({
	Name = "Entity",
	Constructor = function(self, id)
		self.Id = id
	end,
})

function Entity:GetId()
	return self.Id
end

local entity = Entity("entity-1")
assert(entity:IsA(Entity))
```

Class registries are runtime-local. Shared server/client code must register the
class on both runtimes when both sides need name-based lookup. Existing class
instances continue working after their class is unregistered.

See the [Classes guide](../classes/index.md).

#### Server-Authoritative Network Guard Pipeline

Every inbound remote function, client-fired signal, and remote-property read
can use the same guard pipeline. Rules can be declared globally, as service
defaults, or for one named remote.

Requests pass through these stages in order:

| Stage | Purpose | Failure code |
| ---: | --- | --- |
| 1. Rate limit | Consume the server-supplied Player's rolling-window budget. | `RATE_LIMITED` |
| 2. Validate | Check payload types, shapes, lengths, ranges, and finite numbers. | `INVALID_PAYLOAD` |
| 3. Global authenticate | Enforce server-wide session, profile, ban, or access policy. | `UNAUTHENTICATED` |
| 4. Remote authenticate | Enforce service or remote-specific identity requirements. | `UNAUTHENTICATED` |
| 5. Global verify | Apply server-wide contextual checks. | `UNVERIFIED` |
| 6. Remote verify | Authorize the requested action against authoritative game state. | `UNVERIFIED` |

Callbacks must explicitly return `true` to pass. Returning `false` or `nil`, or
throwing, rejects the request. Guard and limiter exceptions use `GUARD_ERROR`.

```lua
local Validator = require(Vanguard.Util.Validator)

local TradeService = Vanguard.CreateService({
	Name = "TradeService",

	Client = {
		Offer = function(self, player, targetUserId, itemIds)
			return self.Server:Offer(player, targetUserId, itemIds)
		end,
	},

	Network = {
		Offer = {
			RateLimit = { Limit = 3, Window = 2 },
			Validate = Validator.tuple(
				Validator.integer({ Min = 1 }),
				Validator.array(Validator.string(), { MaxLength = 20 })
			),
			Authenticate = function(player)
				return ProfileService:IsLoaded(player), "Profile unavailable"
			end,
			Verify = function(player, _context, targetUserId, itemIds)
				return TradeService:CanOffer(player, targetUserId, itemIds)
			end,
		},
	},
})
```

Additional network observability includes `GetNetworkStats`,
`ResetNetworkStats`, structured rejection callbacks, optional rejection logs,
and five-second duplicate-log throttling. Statistics still count every
rejection when duplicate output is suppressed.

See [Network Security](../network-security/index.md) for the threat model,
configuration precedence, callback contracts, and secure patterns.

#### Network Protocol Verification

The server stamps its remote container with `VanguardProtocol` and
`VanguardVersion`. Before creating service proxies, the client:

- rejects an incompatible protocol;
- records the server package version;
- warns when compatible client and server package versions differ.

Protocol verification catches stale or mixed framework installations. It is
not player authentication and does not prove that client code is unmodified.
The current protocol is `1`.

#### Utility Toolkit

The package now ships a dependency-free utility layer under `Vanguard.Util`.

| Utility | Capability added |
| --- | --- |
| `Cache` | TTL expiration, LRU capacity, lazy values, cached `nil`, clock injection, and cleanup aliases |
| `Class` | Constructors, inheritance, runtime checks, and class creation without global registration |
| `Cleaner` | Deterministic cleanup for functions, connections, Instances, threads, promises, and cleanup objects |
| `Logger` | Scoped, level-based output with framework configuration integration |
| `NetworkGuard` | Standalone rate-limit, validation, authentication, and verification composition |
| `Promise` | Chaining, rejection recovery, aggregation, events, delays, and asynchronous composition |
| `RateLimiter` | Per-key rolling-window budgets, retry timing, weak keys, and injectable clocks |
| `Signal` | Local event dispatch with connect, once, wait, fire, and destroy behavior |
| `Validator` | Composable primitive, tuple, array, shape, union, optional, and custom validators |

Main-module shortcuts create common utilities without manually requiring their
module:

```lua
local cache = Vanguard.CreateCache({ Capacity = 100, TTL = 30 })
local limiter = Vanguard.CreateRateLimiter({ Limit = 10, Window = 1 })
local logger = Vanguard.CreateLogger("Inventory")
```

Start with the [Utilities index](../utilities/index.md) for individual guides
and API contracts.

#### Runtime Diagnostics and Update Checks

Startup logs now report the package version, object counts, and elapsed startup
time. `debug` logging adds per-object lifecycle details.

The server can also check the Wally index after startup and warn when a newer
package is available. The check is non-blocking: unavailable HTTP, Studio
settings, or registry errors do not affect readiness.

```lua
Vanguard.Start({
	LogLevel = "debug",
	CheckForUpdates = true,
})
```

Set `CheckForUpdates = false` to disable it or provide `UpdateCheckUrl` for a
custom package index.

#### Expanded Documentation and Type Exports

The release adds end-to-end documentation for setup, configuration, lifecycle,
services, controllers, components, classes, networking, network security,
utilities, migration, and troubleshooting. Public framework and utility types
are exported for Luau autocomplete and static analysis.

### Changed

- Bootstrap class folders load before service or controller folders, allowing
  modules to resolve registered classes while they are required.
- Service initialization runs in descending priority groups instead of one
  undifferentiated batch.
- Service start hooks are scheduled in priority order, with alphabetical
  ordering for ties.
- Framework readiness now clearly distinguishes awaited init work from spawned
  start work.
- Global, service-default, and named-remote network rules merge predictably;
  named fields can disable inherited service defaults with `false`.
- Startup and rejection output uses scoped logging with configurable levels.
- Public methods consistently support the documented dot or colon invocation
  forms where methods are bound by the framework.

### Fixed

#### Module Failure Isolation

A require error in one discovered ModuleScript no longer aborts the complete
folder load. Modules returning zero or multiple values are reported with their
path and skipped using the same isolation path.

#### RemoteProperty Primitive Values

`RemoteProperty` methods now retain the property object as `self` when called
with dot syntax. A property initialized with `false`, a number, a string, or
another primitive no longer attempts to index that value as the property
instance.

Both invocation forms are supported:

```lua
StoreInitialized:Set(true)
StoreInitialized.Set(true)
```

#### Client Compatibility Gate

The client validates the network protocol before building proxies, preventing
hard-to-diagnose calls through an incompatible server remote layout.

### Security

- Added payload validation before application callbacks execute.
- Added per-player rolling-window rate limits; malformed traffic consumes
  capacity before validation.
- Added authentication callbacks for server-owned session and identity state.
- Added verification callbacks for ownership, permissions, cooldowns,
  distance, currency, inventory, and other authoritative state.
- Added structured rejection codes, server-only internal error details, and
  optional monitoring through `OnRejected`.
- Added inbound guards for function calls, client-fired signals, and property
  reads.

!!! warning "Security boundary"
    Client-provided booleans, UserIds, replicated values, and tokens embedded
    in LocalScripts are not authentication. Use the `Player` supplied by
    Roblox and verify requests against server-owned state.

Guards do not create transactions. Mutation code should re-check critical
state when it can change between verification and the final write.

### Upgrade Guide

#### 1. Update the Package

Change the game project's `wally.toml`:

```toml
[dependencies]
Vanguard = "twrblxdevs/vanguard@0.1.13"
```

Then reinstall packages from that game project:

```shell
wally install
```

Restart the Rojo sync and Studio play session so both server and client execute
the newly installed package.

#### 2. Verify the Runtime Version

```lua
print(Vanguard.Version) -- 0.1.13
print(Vanguard.NetworkProtocol) -- 1
```

On the client, inspect the server handshake when diagnosing mixed installs:

```lua
local info = Vanguard.GetNetworkInfo()
print(info.Protocol, info.ServerVersion)
```

#### 3. Add Priority Only Where Ordering Is Required

Leave unrelated services at the default priority. Assign different values when
one service's init work must complete before another priority group begins.
Services in the same group run concurrently and should not wait on each other
without an explicit shared promise.

#### 4. Protect High-Risk Remotes First

Start with remotes that mutate data, spend currency, grant items, perform
expensive queries, or affect another player. Add controls in this order:

1. payload validation and collection-size limits;
2. a conservative per-player rate limit;
3. authentication against loaded server session state;
4. action-specific authorization against current game state;
5. rejection monitoring and operational tuning.

#### 5. Review Loader Warnings

Startup can continue after a discovered module fails. Do not mistake framework
readiness for proof that every requested definition loaded. Resolve each module
warning and verify required objects with `HasClass`, `GetService`, component
lookup, or controller lookup as appropriate.

### Upgrade Checklist

- [ ] Update the game project's dependency to `0.1.13`.
- [ ] Run `wally install` and confirm Studio no longer shows an older `_Index`
      package path.
- [ ] Confirm server and client report protocol `1` and compatible versions.
- [ ] Run one startup with `LogLevel = "debug"` and resolve loader warnings.
- [ ] Move required setup into `VanguardInit`; keep long-running loops in
      `VanguardStart`.
- [ ] Assign service priorities only for real init dependencies.
- [ ] Validate and rate-limit sensitive inbound remotes.
- [ ] Authenticate against server-owned profile or session state.
- [ ] Inspect `GetNetworkStats()` during play testing.
- [ ] Test remote-property reads and updates from both server and client.

### Behavioral Boundaries

| Behavior | Contract in `0.1.13` |
| --- | --- |
| Init hooks | Awaited; errors reject startup. |
| Start hooks | Spawned and not awaited; errors do not reject the completed startup chain. |
| Controller priority | Controllers initialize concurrently and do not currently expose `Priority`. |
| Class registry | Separate on server and client; classes have no framework lifecycle. |
| Failed discovered modules | Skipped after logging; dependent registry lookups may still fail later. |
| Remote signals | Rejected inbound events are dropped; one-way transport provides no client response. |
| Guard callbacks | Must explicitly return `true`; errors reject with `GUARD_ERROR`. |
| Protocol verification | Detects framework compatibility only; it is not authentication. |
| Update checks | Server-only and non-blocking; HTTP failure never blocks readiness. |
| Promise utility | Does not include built-in cancellation or timeout policy. |

### Related Documentation

- [Getting Started](../getting-started/index.md)
- [Lifecycle](../lifecycle/index.md)
- [Services](../services/index.md)
- [Classes](../classes/index.md)
- [Networking](../networking/index.md)
- [Network Security](../network-security/index.md)
- [Utilities](../utilities/index.md)
- [Type System](../type-system/index.md)
- [Troubleshooting](../troubleshooting/index.md)

## 0.1.10 - Date Not Recorded

The repository preserves one specific compatibility boundary from this release:

### Fixed

- Bound `RemoteProperty` methods began supporting both dot and colon calls.
- Calling `Property.Set(true)` no longer treated the primitive value as `self`
  and attempted to index it with `_value`.

Projects still showing a Wally `_Index` path below `0.1.10` should update before
diagnosing remote-property invocation failures.

## Earlier Releases

Complete notes for releases before `0.1.13` were not preserved in repository
history. This page documents only compatibility details supported by the
current source and guides rather than inventing release dates or version
assignments.

## Release Notes Policy

Future release entries should include:

- a release summary and upgrade-impact matrix;
- **Added**, **Changed**, **Deprecated**, **Removed**, **Fixed**, and
  **Security** sections when applicable;
- explicit protocol, configuration, and lifecycle compatibility notes;
- migration steps for behavior changes;
- known behavioral boundaries and links to updated guides.

Breaking changes require a dedicated migration section. Package compatibility
and network protocol compatibility are tracked separately because a package
version change does not always require a protocol change.
