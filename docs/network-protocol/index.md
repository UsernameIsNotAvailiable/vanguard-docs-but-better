# Network Protocol 1

Protocol `1` is Vanguard's client/server compatibility contract. It defines the
replicated remote hierarchy, discovery handshake, call shapes, Player identity,
guard timing, error transport, and readiness rules used to construct client
service proxies.

Package version and protocol version are intentionally separate. Vanguard
`0.1.14` uses protocol `1` because linked errors, Math, Switch, and class member
groups do not change replicated transport. The planned `0.1.15` Plugin
Developer API will also remain on protocol `1` unless its final design changes
replicated hierarchy or call semantics.

## Compatibility Contract

The server publishes two attributes on `_VanguardRemotes`:

| Attribute | Protocol 1 value | Purpose |
| --- | --- | --- |
| `VanguardProtocol` | `1` | Hard compatibility gate for remote layout and semantics |
| `VanguardVersion` | Server package version | Diagnostic signal for mixed package installations |

The client requires an exact protocol match before it builds any service proxy.
Package versions may differ when the protocol matches; Vanguard warns because
local behavior or bug fixes may still differ.

| Client protocol | Server protocol | Package versions | Result |
| ---: | ---: | --- | --- |
| 1 | 1 | Same | Proxy construction continues |
| 1 | 1 | Different | Warning, then proxy construction continues |
| 1 | Missing | Any | `VG-NET-001`; proxy construction stops |
| 1 | Any value other than 1 | Any | `VG-NET-001`; proxy construction stops |

## Replicated Hierarchy

Protocol 1 publishes remotes beneath the Vanguard package ModuleScript:

```text
Vanguard [ModuleScript]
└── _VanguardRemotes [Folder]
    ├── VanguardProtocol = 1 [Attribute]
    ├── VanguardVersion = "0.1.14" [Attribute]
    └── <ServiceName> [Folder]
        ├── <FunctionName> [RemoteFunction]
        ├── <SignalName> [RemoteEvent or UnreliableRemoteEvent]
        └── <PropertyName> [Folder]
            ├── _Get [RemoteFunction]
            └── _Changed [RemoteEvent]
```

Only services with at least one supported `Client` member receive a service
folder. A purely server-side service is intentionally absent from the client
protocol surface.

Service and remote names come directly from registration names and `Client`
table keys. Renaming one is a game API change even when Vanguard's protocol
number remains `1`.

## Publication Timeline

The server builds the complete remote tree before service init, but keeps the
root folder unparented while required initialization runs. It publishes
`_VanguardRemotes` after:

1. every service definition is registered;
2. network options and named rules validate;
3. service folders, guards, functions, signals, and properties are compiled;
4. all `VanguardInit` priority groups complete;
5. `VanguardStart` hooks are scheduled.

The folder is parented before components start and before Vanguard marks server
readiness complete. A failed init therefore cannot expose a partially-ready
remote API to clients.

## Client Discovery Handshake

The first client `GetService(name)` performs this sequence:

1. return the cached proxy when one already exists;
2. wait for `_VanguardRemotes` for at most `RemoteTimeout` seconds;
3. read `VanguardProtocol` and `VanguardVersion`;
4. reject a protocol other than `1` with `VG-NET-001`;
5. warn when the server package version differs;
6. wait for the named service folder for at most `RemoteTimeout` seconds;
7. map each recognized child to a client proxy member;
8. cache the completed service proxy.

`RemoteTimeout` defaults to 15 seconds. A timeout uses `VG-NET-002`. Increasing
it can accommodate intentionally long server init, but it does not repair a
failed server bootstrap or a service with no client remotes.

```lua
Vanguard.Start({
	RemoteTimeout = 30,
})
```

## Remote Function Contract

### Client To Server

Generated client methods support dot and colon calls:

```lua
InventoryService.GetItems()
InventoryService:GetItems()
```

For colon calls, the generated proxy detects and removes its own service table
before invoking the `RemoteFunction`. Only explicit application arguments are
sent over Roblox transport.

### Server Invocation Shape

Roblox supplies the sending `Player`; the client cannot choose that argument.
Vanguard invokes service client methods as:

```lua
clientMethod(callContext, player, ...payload)
```

The temporary `callContext` provides:

- `self.Server`: the owning server service;
- `self.<RemoteName>`: another entry from the service's `Client` table;
- writes to normal keys: updates the `Client` table;
- writes to `self.Server`: rejected as read-only.

All return values are forwarded through the `RemoteFunction`, subject to Roblox
serialization. An application error thrown by the service method propagates to
the caller. With `ServicePromises = true`, the generated client method catches
that transport error and rejects a Vanguard Promise. In yielding mode, callers
should use `pcall` when they need recovery.

## Signal Contract

`CreateSignal()` maps to `RemoteEvent`. `CreateUnreliableSignal()` attempts to
use `UnreliableRemoteEvent` and falls back to `RemoteEvent` when unavailable.

### Client To Server

```lua
Signal:Fire(...payload)
```

Server listeners receive:

```lua
function(player, ...payload)
```

The Player is Roblox-supplied. Vanguard runs the inbound guard once before any
listener. A rejected event is dropped; protocol 1 has no rejection response for
one-way signal transport.

### Server To Client

Protocol 1 supports `Fire`, `FireFor`, `FireAll`, `FireExcept`, and `FireWhere`.
Client listeners receive only the application payload. Server-to-client signal
delivery does not run the inbound guard pipeline.

Reliable events preserve Roblox `RemoteEvent` behavior. Unreliable events may
be dropped or reordered and are suitable only for replaceable state such as aim
or cosmetic motion.

## Remote Property Contract

A property is a folder containing `_Get` and `_Changed`.

### Read Path

`ClientProperty:Get()` invokes `_Get`. The request passes through rate limiting,
validation, authentication, and verification with remote type `Property`. On
success, the server returns the Player override when one exists, otherwise the
global value.

### Update Path

- `Set(value)` updates the global value and fires `_Changed` to all clients.
- `SetFor(player, value)` stores an override and fires only to that Player.
- `ClearFor(player)` removes an override and sends the current global value.
- Player overrides are removed when the Player leaves.

Property values are internally packed so `nil` remains a valid stored value.
The update event is server-to-client traffic and does not pass through inbound
guards.

`Observe(callback)` subscribes before asynchronously fetching the current value.
A fast update and initial read can therefore occur close together; consumers
should treat callbacks as state replacement rather than event deltas.

## Guard Pipeline On The Wire

Every inbound function, client-fired signal, and property read uses one compiled
guard. Protocol 1 enforces this exact order:

1. **Rate limit** consumes one unit for the Roblox-supplied Player.
2. **Validate** checks client payload only.
3. **Global authenticate** checks server-wide identity/session policy.
4. **Remote authenticate** checks merged service and named-remote policy.
5. **Global verify** checks server-wide contextual policy.
6. **Remote verify** authorizes the specific action and payload.

Malformed requests consume rate-limit capacity because rate limiting happens
first. Every callback must explicitly return `true`. Returning `false`, `nil`,
or throwing rejects the request.

The callback context contains:

```lua
type NetworkContext = {
	Player: Player,
	Service: any,
	ServiceName: string,
	RemoteName: string,
	RemoteType: "Function" | "Signal" | "Property",
	Arguments: { any } & { n: number },
}
```

`Arguments.n` preserves trailing nil payload values.

## Rejection And Error Transport

Protocol 1 uses stable rejection names:

| Rejection | Catalog code | Client-visible behavior |
| --- | --- | --- |
| `RATE_LIMITED` | `VG-NET-101` | Function/property error; signal dropped |
| `INVALID_PAYLOAD` | `VG-NET-102` | Function/property error; signal dropped |
| `UNAUTHENTICATED` | `VG-NET-103` | Function/property error; signal dropped |
| `UNVERIFIED` | `VG-NET-104` | Function/property error; signal dropped |
| `GUARD_ERROR` | `VG-NET-105` | Generic function/property error; signal dropped |

Function and property errors retain the rejection name and now include a direct
error-catalog URL. Internal guard exceptions remain server-only in rejection
detail, logs, and `OnRejected`; they are not disclosed to clients.

Rejection logs are throttled for five seconds per Player, service, remote, and
code. Statistics and `OnRejected` still process every rejected request.

## Serialization And Delivery Boundaries

Protocol 1 relies on Roblox remote serialization. Vanguard does not add a
custom serializer, schema negotiation, compression, retry layer, ordering
layer, or transaction system.

Consequences:

- only Roblox-serializable payloads can cross the boundary;
- validators run after Roblox has delivered a decoded payload to the server;
- unreliable delivery guarantees come from `UnreliableRemoteEvent`;
- a successful guard does not lock game state before mutation;
- critical mutation code should re-check state that can change concurrently;
- server-to-client data is visible to the receiving client and is not secret.

## What Requires A Protocol Bump

A future Vanguard release should change the protocol number when clients and
servers can no longer safely interpret the same replicated contract. Examples:

- renaming `_VanguardRemotes`, `_Get`, or `_Changed`;
- changing service or property folder structure;
- changing function argument placement or Player identity semantics;
- changing marker-to-remote mappings;
- requiring new handshake attributes;
- changing rejection transport in a way old clients cannot interpret.

These changes do **not** require a protocol bump by themselves:

- local-only utilities such as Math or Switch;
- class registry or member-access features;
- richer logs, error codes, and documentation links;
- type-export improvements;
- bug fixes that preserve the hierarchy and call contract.

## Diagnostics

```lua
print(Vanguard.Version)
print(Vanguard.NetworkProtocol)

local info = Vanguard.GetNetworkInfo()
print(info.Protocol, info.ServerVersion)
```

On the server, `GetNetworkInfo()` is available immediately. On the client,
fields populate during first remote-container discovery.

When Studio reports a protocol mismatch:

1. inspect the Wally `_Index` version in both server and client stacks;
2. confirm both bootstraps require the same package alias;
3. run `wally install` in the game project;
4. remove stale or duplicate package copies;
5. restart Rojo sync and the play session.

Continue with [Networking](../networking/index.md) for API usage,
[Network Security](../network-security/index.md) for policy, and
[Errors](../errors/index.md) for diagnostics.
