# MyNetworker

A lightweight, original Roblox networking library: one `RemoteEvent` + `RemoteFunction`
pair per service, with method-call style messaging instead of hand-rolled remotes.

Built from scratch (own implementation, own `Signal` class — no external deps),
following the same API shape we worked through in chat: `.new()`, `:fire()`,
`:fetch()`, `getServerChangedSignal()`.

## Install

1. Put the `src` folder somewhere shared, like `ReplicatedStorage`, renamed to
   whatever you like (e.g. `ReplicatedStorage.MyNetworker`).
2. Require it from server and client scripts:
   ```lua
   local MyNetworker = require(ReplicatedStorage.MyNetworker)
   ```

> Note: `NetworkerServer` creates a `_links` folder as a sibling of itself
> (`script.Parent`) the first time it's used — that's where each tag's
> `RemoteEvent`/`RemoteFunction` pair lives. No manual remote setup needed.

## Server

```lua
local MyNetworker = require(ReplicatedStorage.MyNetworker)

local PartService = {}

function PartService:init()
	self.networker = MyNetworker.server.new("partService", self, {
		self.requestPart, -- only methods listed here are callable by clients
	})
end

function PartService:requestPart(player, position)
	local part = Instance.new("Part")
	part.Position = position
	part.Anchored = true
	part.Parent = workspace
	return true
end

return PartService
```

## Client

```lua
local MyNetworker = require(ReplicatedStorage.MyNetworker)

local networker = MyNetworker.client.new("partService", {})

networker:fire("requestPart", Vector3.new(0, 10, 0))             -- fire and forget
local ok = networker:fetch("requestPart", Vector3.new(0, 10, 0)) -- wait for a reply
```

## Pushing values to clients (auto-sync)

```lua
-- SERVER
self.networker:set(player, "coins", 100)   -- one player
self.networker:setAll("coins", 100)        -- everyone
```

```lua
-- CLIENT
local coinsChanged = networker:getServerChangedSignal("coins")
coinsChanged:Connect(function(newAmount)
	print("Coins updated:", newAmount)
end)
```

## Server -> client method calls

The server can also call a method on the client's module by name, same as the
client calls the server:

```lua
-- SERVER
self.networker:fire(player, "showPopup", "Welcome!")

-- CLIENT
local clientModule = {}
function clientModule.showPopup(self, text)
	print("popup:", text)
end
local networker = MyNetworker.client.new("partService", clientModule)
```

## Files

| File | Purpose |
|---|---|
| `init.luau` | Entry point, exposes `.server` and `.client` |
| `NetworkerServer.luau` | Server-side wrapper + access control |
| `NetworkerClient.luau` | Client-side wrapper |
| `NetworkerUtils.luau` | Shared types/constants |
| `Signal.luau` | Minimal event/signal class used for changed-signals |

## Testing

This was tested two ways:
1. **Logic-level**: the core algorithm (whitelist matching, fire/fetch routing,
   set/setAll sync signals, server→client calls, destroy/cleanup, instance-tag
   resolution) was ported into a mock Roblox environment and run through 11
   automated checks outside of Roblox — all 11 passed. This also caught and
   fixed a real bug (the instance-tag branch was calling `WaitForChild` on an
   attribute, which doesn't work — attributes need
   `GetAttributeChangedSignal():Wait()` instead, which is what's in this code now).
2. **Engine-level**: things that only exist in the real Roblox engine
   (actual network replication between two separate clients, real
   RemoteEvent/RemoteFunction objects) can't be exercised outside Studio.
   Suggested smoke test:

1. Drop `src` into `ReplicatedStorage` as `MyNetworker`.
2. Add the two example scripts from `/example` (rename to drop `-example`,
   put server one in `ServerScriptService`, client one in `StarterPlayerScripts`).
3. Press Play — you should see a blue part spawn at `(0, 10, 0)` and a print
   in the server output confirming the player who requested it.
