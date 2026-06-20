# Switch

`Vanguard.Switch` and `Vanguard.Util.Switch` provide explicit case dispatch
similar to JavaScript `switch`, adapted to Luau method chaining and tables.

Vanguard Switch has no implicit fall-through. The first matching case runs,
which makes state-machine and command behavior easier to inspect.

## Builder API

```lua
local result = Vanguard.Switch.new(state)
	:Case("Paused", "Playing")
	:Cases({ "Playing", "Resuming" }, function(current, suffix)
		return current .. suffix
	end)
	:Default("Idle")
	:Run("State")
```

For `state == "Playing"`, this returns `"PlayingState"`.

## Case

```lua
switch:Case(expectedValue, result)
```

Adds one case. `result` may be a literal value or function. Function results
receive the switched value first, followed by arguments passed to `Run`.

```lua
local label = Vanguard.Switch.new(status)
	:Case("Ready", function(value)
		return string.upper(value)
	end)
	:Run()
```

## Cases

```lua
switch:Cases({ valueA, valueB, valueC }, result)
```

Groups multiple values under one result without fall-through syntax.

```lua
local isActive = Vanguard.Switch.new(state)
	:Cases({ "Playing", "Loading", "Resuming" }, true)
	:Default(false)
	:Run()
```

The array must contain at least one value. Invalid collections use
[`VG-SWITCH-001`](../../errors/index.md#vg-switch-001).

## Default

```lua
switch:Default(result)
```

Sets the fallback result. Without a matching case or default, `Run` returns
nil. Calling `Default` again replaces the previous fallback.

## HasMatch

```lua
if switch:HasMatch() then
	-- At least one declared case matches.
end
```

`HasMatch` checks declared cases only; a default does not count as a case match.

## Run

```lua
switch:Run(...arguments)
```

Scans cases in declaration order and returns the selected literal or every
value returned by the selected handler. `Run` does not mutate the builder, so a
configured switch may be evaluated again.

## Direct Match

Use `Switch.match` when table-key equality is sufficient:

```lua
local permission = Vanguard.Switch.match(role, {
	Owner = "all",
	Moderator = "moderation",
	Player = "gameplay",
}, "none")
```

Functions are invoked as handlers:

```lua
local response = Vanguard.Switch.match(command, {
	Ping = function(_value, sentAt)
		return os.clock() - sentAt
	end,
}, function(value)
	return `Unknown command: {tostring(value)}`
end, sentAt)
```

Signature:

```lua
Switch.match(value, cases, defaultResult?, ...arguments)
```

## Equality And Ordering

- Cases use Luau `==` equality.
- Strings, numbers, booleans, enums, Instances, and table identity are valid.
- Builder cases run in declaration order.
- `Switch.match` uses direct table lookup.
- No case falls through into the next case.
- Duplicate builder cases select the first declaration.

## State Machine Pattern

```lua
local handlers = Vanguard.Switch.new(machine.State)
	:Case("Idle", function()
		machine:LookForMatch()
	end)
	:Case("Starting", function()
		machine:PreparePlayers()
	end)
	:Case("Running", function()
		machine:TickRound()
	end)
	:Default(function(value)
		warn(`Unhandled state {tostring(value)}`)
	end)

handlers:Run()
```

Switch dispatch organizes selection logic; it does not replace validation or a
domain-specific state-transition policy.
