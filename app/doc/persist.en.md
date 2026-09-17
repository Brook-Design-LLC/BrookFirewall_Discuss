# persist

Keeps data around for the next packet.

```lua
persist.get(key)           -- nil if absent
persist.set(key, value)
persist.delete(key)
```

The colon form `persist:get(key)` works too; the two are equivalent.

`key` must be a string. `value` can be nil, a boolean, a number, a string, or a table.

## When to use it

A script's environment is shared across packets, so an ordinary variable already keeps data around:

```lua
local hits = 0

function http_request(request)
	hits = hits + 1
	return nil
end
```

For counters and flags that's all you need.

What `persist` adds is that it doesn't count against the script's 1.5MB memory budget, which makes it the right place for larger data — a cached response, a growing list of IDs.

Neither is **written to disk**. Any change to the script list, or a VPN reconnect, reloads the environment and clears both. This holds state for the current session, not settings.

## The 128KB limit

Every `persist.set` measures everything already stored plus the value you're writing. Above 128KB:

- `persist: out of memory` is raised
- the new value is **not stored**, and everything previously stored **is kept**
- as long as you catch the error, the script carries on

If you don't catch it the handler counts as failed and the packet passes through unchanged.

So when the size isn't fully under your control, wrap it in `pcall`:

```lua
local ok, err = pcall(persist.set, persist, "cache", payload)
if not ok then
	log(err, "warning")
	persist.delete("cache")
end
```

Note the arguments: `persist` itself goes in as the first one.

## Example

Recording the most recent response status per host. Because the old data survives a failed write, "reset when full" is a safe strategy:

```lua
function http_response(response)
	local host = response.host
	if not host then
		return nil
	end

	local seen = persist.get("seen") or {}
	local entry = { status = response.status, at = time.unix() }
	seen[host] = entry

	local ok = pcall(persist.set, persist, "seen", seen)
	if not ok then
		pcall(persist.set, persist, "seen", { [host] = entry })
	end
	return nil
end
```
