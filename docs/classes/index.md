# Classes

Vanguard provides a lightweight class utility plus independent server and client class registries. Classes support constructors, inheritance, callable class tables, inherited methods and metamethods, runtime type checks, and grouped public, private, and static members.

## Create and Register a Class

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
```

`Vanguard.CreateClass` creates and immediately registers the class in the current runtime.

Instantiate with `.new` or by calling the class:

```lua
local first = Entity.new("entity-1")
local second = Entity("entity-2")
```

Both forms are equivalent.

## Public, Private, And Static Members

Vanguard `0.1.14` adds optional access groups while preserving legacy top-level
class definitions.

```lua
local Wallet
Wallet = Vanguard.CreateClass({
	Name = "Wallet",

	Private = {
		Balance = 0,
		History = {},

		Constructor = function(_self, private, openingBalance)
			private.Balance = openingBalance
		end,

		Record = function(_self, private, amount)
			table.insert(private.History, amount)
		end,
	},

	Public = {
		Deposit = function(_self, private, amount)
			private.Balance += amount
			private:Record(amount)
		end,

		GetBalance = function(_self, private)
			return private.Balance
		end,
	},

	Static = {
		Currency = "Credits",

		Open = function(openingBalance)
			return Wallet(openingBalance)
		end,
	},
})
```

```lua
local wallet = Wallet.Open(100)
wallet:Deposit(25)

wallet:GetBalance() -- 125
wallet.Balance -- nil
wallet.Record -- nil
Wallet.Currency -- "Credits"
wallet.Currency -- nil
```

### Public Members

`Public` contains fields and methods visible on instances. Grouped public
functions use this signature:

```lua
function(self, private, ...arguments)
```

`private` is the hidden state owned by the class that declared the method.
Calling a grouped public method on anything other than a constructed instance
raises [`VG-CLASS-001`](../errors/index.md#vg-class-001).

Top-level methods remain public and retain the legacy `(self, ...arguments)`
signature. This keeps existing classes source-compatible:

```lua
function Wallet:GetDisplayName()
	return self.Name
end
```

### Private Members

`Private` values are stored in a side table keyed by instance and owning class.
They are not copied onto the public instance or class table.

- Non-function defaults are deep-cloned for each instance.
- Nested tables are not shared between instances.
- Private methods are bound into the private state.
- `private:Method(...)` and `private.Method(...)` are both accepted.
- External reads of private names return nil unless a separate public field uses
  that name.

`Private.Constructor` receives `(self, private, ...arguments)` and is the place
to derive hidden state from constructor arguments.

Privacy is runtime encapsulation, not a security boundary. A public method can
still intentionally return a private value, and all client code remains visible
to the client.

### Static Members

`Static` members live on the class table and are excluded from instance lookup.
Static members inherit through class inheritance:

```lua
Wallet.Currency -- "Credits"
PremiumWallet.Currency -- "Credits"
wallet.Currency -- nil
```

Add or replace a static member after class creation with:

```lua
Wallet:SetStatic("MaximumBalance", 1_000_000)
Class.setStatic(Wallet, "MaximumBalance", 1_000_000)
```

Direct assignments such as `Wallet.NewMethod = function() end` remain public by
default for compatibility. Use `SetStatic` when the member must remain
class-only.

### Access Conflicts

A member name may have only one access category in a class definition. Vanguard
raises [`VG-CLASS-002`](../errors/index.md#vg-class-002) when:

- a key appears in more than one of `Public`, `Private`, or `Static`;
- a grouped member duplicates a top-level member;
- a group uses a reserved Vanguard key;
- a child changes an inherited public member into static or vice versa;
- a static declaration uses a metamethod name.

Base and child private states are separate. An inherited base public method
receives base-private state; a child public method receives child-private state.
This prevents a subclass from accidentally depending on its parent's hidden
representation.

## Definition Fields

| Field | Required | Description |
| --- | --- | --- |
| `Name` | Yes | Non-empty class and registry name |
| `Extends` | No | Parent Vanguard class; main API also accepts a registered class name |
| `Constructor` | No | Called for every new instance |
| `Public` | No | Instance-visible members; functions receive `(self, private, ...)` |
| `Private` | No | Per-instance hidden defaults and methods; may include `Constructor` |
| `Static` | No | Class-only members excluded from instance lookup |

All other fields are copied onto the class. Define instance methods in the definition or assign them afterward.

Class definitions may not define `new`; use `Constructor` instead. Vanguard reserves `Name`, `Extends`, `Extend`, `IsA`, `Public`, `Private`, `Static`, `SetStatic`, `Super`, `new`, `__index`, `__metatable`, and `__vanguardClass`.

## Inheritance

```lua
local PlayerEntity = Vanguard.CreateClass({
	Name = "PlayerEntity",
	Extends = Entity,

	Constructor = function(self, _id, player)
		self.Player = player
	end,
})
```

Or extend from the parent:

```lua
local PlayerEntity = Entity:Extend({
	Name = "PlayerEntity",
	Constructor = function(self, _id, player)
		self.Player = player
	end,
})

Vanguard.RegisterClass(PlayerEntity)
```

`Class:Extend` creates a class but does not automatically register it. Register explicitly when registry lookup is needed.

## Constructor Order

Every class in the inheritance chain initializes base-first. For each class,
`Private.Constructor` runs before the legacy top-level `Constructor`; both
receive the same construction arguments.

```lua
local playerEntity = PlayerEntity("entity-1", player)
```

Call order:

```text
Entity.Private.Constructor(instance, entityPrivate, "entity-1", player)
Entity.Constructor(instance, "entity-1", player)
PlayerEntity.Private.Constructor(instance, playerPrivate, "entity-1", player)
PlayerEntity.Constructor(instance, "entity-1", player)
```

Do not manually call the base constructor; Vanguard already does it.

If a constructor throws, instance creation throws and no instance is returned. Constructors do not have automatic cleanup, so avoid acquiring resources before validation that may fail.

## Method and Static Inheritance

Instances resolve methods through the child class, then parent classes.

```lua
print(playerEntity:GetId())
```

Class table reads also inherit static values through the parent class metatable.

Metamethods directly defined on a parent are copied into child classes when the child does not define its own. `__index`, `__metatable`, and Vanguard's internal marker are not copied.

## Runtime Checks

Instance method:

```lua
playerEntity:IsA(PlayerEntity) -- true
playerEntity:IsA(Entity) -- true
playerEntity:IsA("Entity") -- true
```

Utility checks:

```lua
local Class = require(Vanguard.Util.Class)

Class.isClass(Entity) -- true
Class.isInstance(playerEntity) -- true
Class.isInstance(playerEntity, Entity) -- true
Class.isA(PlayerEntity, Entity) -- true
Class.getClass(playerEntity) == PlayerEntity -- true
```

String checks compare class names across the inheritance chain. Class-object checks are stronger when you already hold the expected class.

## Registry API

### CreateClass

```lua
local class = Vanguard.CreateClass(definition)
```

Creates and registers. If `Extends` is a string, the named parent must already be registered.

### RegisterClass

```lua
local Class = require(Vanguard.Util.Class)
local Item = Class.create({ Name = "Item" })
Vanguard.RegisterClass(Item)
```

Registering the same class object again is idempotent. Registering a different class with the same name errors.

### GetClass and HasClass

```lua
local Item = Vanguard.GetClass("Item")
if Vanguard.HasClass("Item") then
	-- Registered.
end
```

`GetClass` errors when missing. `HasClass` returns a boolean.

### GetClasses

```lua
local classes = Vanguard.GetClasses()
```

Returns a shallow clone of the registry.

### UnregisterClass

```lua
local removed = Vanguard.UnregisterClass("Item")
```

Returns the removed class or nil. Existing instances continue working; unregistering only removes discovery by name.

## Loading Class Modules

```lua
Vanguard.AddClasses(folder)
Vanguard.AddClassesDeep(folder)
```

`LoadClasses` and `LoadClassesDeep` are aliases.

Modules may return:

- a class already made with `Vanguard.CreateClass`;
- a class made with `Vanguard.Util.Class.create`; or
- a plain class definition.

Plain definitions are passed to `Vanguard.CreateClass` automatically.

## Bootstrap

```lua
Vanguard.Bootstrap({
	Classes = script.Parent.Classes,
	Services = script.Parent.Services,
})
```

Class folders load before service or controller folders. This allows those modules to call `GetClass` while they are being required.

## Runtime Scope

Server and client registries are separate. Register a shared class on both runtimes when both need name-based discovery. Requiring the same shared class module separately on server and client produces runtime-local class objects.

Classes have no Vanguard lifecycle and may be registered or unregistered after startup.

## Class Utility Without the Registry

```lua
local Class = require(Vanguard.Util.Class)

local Temporary = Class.create({
	Name = "Temporary",
})
```

Use the utility directly for private classes that do not need global discovery. Use the Vanguard registry for shared framework-level concepts, factories, or classes referenced by name.
