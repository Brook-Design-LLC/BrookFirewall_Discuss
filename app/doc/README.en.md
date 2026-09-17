# MITM Lua Scripts

Scripts written in Lua 5.4 that can intercept an HTTP request before it goes out, or a response before it reaches the app.

Two things are required: the device must already trust the app's root certificate, and the script must declare which hosts it wants to intercept.

- [API reference](api.en.md)
- [persist](persist.en.md)
- [Examples](examples.en.md)

## A minimal script

```lua
--!name=Hello
--!hosts=full:api.example.com
--!kind=response

function http_response(response)
	response.headers["X-Hello"] = "world"
	return response
end
```

Save it as a `.lua` file and upload it, or paste it into the built-in editor.

## Metadata

The `--!` comments at the top are the configuration. They must come before any code — parsing stops at the first line that isn't a `--!` comment (blank lines and ordinary `--` comments are skipped).

| Field | Purpose |
| --- | --- |
| `name` | Display name, also the tag used when writing logs. Defaults to the filename |
| `hosts` | Which hosts to intercept, comma separated |
| `host-regex` | Regular expression only, same as `regexp:`. Can appear on several lines |
| `kind` | `request`, `response`, or both separated by a comma. Defaults to both |

### The four host forms

| Form | How it matches |
| --- | --- |
| `keyword:gs-loc` | Matches if the host contains this text, case insensitive |
| `full:api.example.com` | The host must match exactly |
| `regexp:\.apple\.com$` | Regular expression, case insensitive |
| `fake:local.brook.api` | Exact match, and points the host at the local device — see below |

Without a prefix the value is treated as a keyword. A misspelled prefix (`ful:` for example) is not an error; the whole string becomes a keyword, so check your spelling.

`regexp:` matches partially — there are no implicit anchors, so `regexp:api` matches every host containing `api`. Add `$` yourself to anchor it.

**A script with neither `hosts` nor `host-regex` will not run.** The settings page marks it accordingly.

### fake: serving an API that doesn't exist

`fake:` points the host at the local device, so the app can connect and your script answers directly. There is no real server involved, but as far as the app is concerned it just called a normal HTTPS API. See the [examples](examples.en.md).

## The two entry points

Define `http_request`, `http_response`, or both. Traffic in a direction you didn't define passes through untouched.

The argument is a table:

| Field | Contents |
| --- | --- |
| `id` | The packet's UUID; a request and its response share the same one |
| `method` | `GET`, `POST`, and so on |
| `url` | The full URL |
| `host` | Hostname only, no port |
| `path` | Path including the query string |
| `status` | Status code, nil on requests |
| `headers` | Table of headers, names keep their original casing |
| `body` | Raw contents, may be binary |
| `encodings` | Format flags derived from the headers, same as `content.flags` |

`body` gives you the raw bytes. When the peer sends compressed content this is the compressed data — decompress it yourself, and use `encodings` to decide which codec to use.

## Return values

**Pass through**: return `nil`. The packet is left completely untouched.

**Modify and forward**: return a table containing only the fields you want to change; anything you leave out stays as it was.

```lua
function http_response(response)
	response.status = 404
	return response
end

function http_request(request)
	return { body = "replaced" }   -- only changes the body
end
```

You can change `headers`, `body`, `host`, `method`, `path`, and `status` on responses. Changing `host` updates the `Host` header too.

`headers` is replaced wholesale rather than merged, so add a header by modifying the existing table (`response.headers["X-Foo"] = "1"`). Passing in a fresh table with a single key wipes out every other header.

After changing the body you don't need to touch `Content-Length` or `Content-Encoding`; both are handled automatically before sending.

**Short circuit**: requests only. Return a table containing `response` and the request stops there — the remote server is never contacted.

```lua
return {
	response = {
		status = 403,
		body = "nope",
		headers = { ["Content-Type"] = "text/plain; charset=utf-8" },
	}
}
```

`status` defaults to 200, `body` defaults to empty, and if you omit `headers` the Content-Type becomes `application/json; charset=utf-8`.

## What's available

The environment is deliberately small. The only standard libraries are `base`, `table`, `string`, `math`, and `utf8`.

There is no `io`, `os`, `package`, `debug`, or `coroutine`, and no `print`, `load`, `dofile`, or `loadfile`. Use `log` to output messages; it writes to the app's log.

Everything else available to you — JSON, compression, hashing, crypto, outbound requests — is in the [API reference](api.en.md).

## Limits

| Item | Limit |
| --- | --- |
| Single execution | 100ms |
| Memory | 1.5MB |
| persist | 128KB |
| Output of compression, crypto, and `http()` | 512KB each |
| `http()` timeout | 10s by default, configurable from 1 to 30 |

Time spent waiting on `http()` doesn't count towards the 100ms, so several sequential requests won't time out — a large loop will.

If a script times out, runs out of memory, or raises an error you didn't catch, the modification is discarded, the packet passes through unchanged, and the message goes to the log. A broken script never blocks traffic.

## How long state lives

Each script's environment is created once when it loads, and the top-level code runs only at that point, so variables you declare are shared across packets:

```lua
local count = 0

function http_request(request)
	count = count + 1
	log("handled " .. count .. " requests")
	return nil
end
```

That memory only lasts for the current connection though. Any change to the script list (adding, deleting, toggling, editing) or a VPN reconnect reloads every script and resets the variables. [persist](persist.en.md) behaves the same way — neither is written to disk.
