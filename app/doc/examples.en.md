# Examples

Each of these is a complete script — swap in your own host and it runs.

## Block a host

Return a `response` from `http_request` and the request stops there.

```lua
--!name=Block Ads
--!hosts=keyword:ads.example.com
--!kind=request

function http_request(request)
	log("blocked " .. request.url)
	return {
		response = {
			status = 403,
			body = "blocked",
			headers = { ["Content-Type"] = "text/plain; charset=utf-8" },
		}
	}
end
```

## Add a header

Modify the existing table and return it. Don't replace `headers` outright — that clears everything else.

```lua
--!name=Add Header
--!hosts=full:api.example.com
--!kind=request

function http_request(request)
	request.headers["X-Debug"] = "1"
	request.headers["User-Agent"] = "CustomClient/1.0"
	return request
end
```

## Rewrite a JSON response

Responses are usually compressed, so check `encodings` first. Write the result straight back to `body`.

```lua
--!name=Patch Profile
--!hosts=full:api.example.com
--!kind=response

function http_response(response)
	local raw = response.body
	if response.encodings.gzip then
		raw = gzip.decode(raw) or raw
	elseif response.encodings.zstd then
		raw = zstd.decode(raw) or raw
	end

	local ok, obj = pcall(json.decode, raw)
	if not ok or type(obj) ~= "table" then
		return nil
	end

	obj.is_vip = true
	obj.expire_at = time.unix() + 86400
	response.body = json.encode(obj)
	return response
end
```

## Only touch certain paths

Usually just a few endpoints on a host matter; let the rest through early.

```lua
--!name=Selective Patch
--!hosts=full:api.example.com
--!kind=response

function http_response(response)
	local path = response.path or ""
	if not path:match("^/v1/user") then
		return nil
	end

	local ok, obj = pcall(json.decode, response.body)
	if not ok then
		return nil
	end
	obj.patched = true
	response.body = json.encode(obj)
	return response
end
```

## Serve an API that doesn't exist

`fake:` points the host at the local device and the script answers directly. Once installed, the app calling `https://local.brook.api/hello?a=1` gets this JSON back.

```lua
--!name=Local API
--!hosts=fake:local.brook.api
--!kind=request

function http_request(request)
	local path = request.path or ""
	local params = query.parse(path:match("%?(.*)$") or "")

	return {
		response = {
			status = 200,
			headers = { ["Content-Type"] = "application/json; charset=utf-8" },
			body = json.encode({
				ok = true,
				echo = params,
				now = time.iso8601(),
			}),
		}
	}
end
```

## Sign a request

The signature has to cover what actually gets sent, so change the body first and sign afterwards.

```lua
--!name=Sign Request
--!hosts=full:api.example.com
--!kind=request

local SECRET = "your-shared-secret"

function http_request(request)
	local ok, obj = pcall(json.decode, request.body)
	if not ok or type(obj) ~= "table" then
		return nil
	end

	obj.ts = time.unix()
	obj.nonce = uuid():lower()
	local body = json.encode(obj)

	request.body = body
	request.headers["X-Signature"] = hex.encode(hmac.sha256(SECRET, body))
	return request
end
```

## Call an external service

`http()` reports failure by returning `error` rather than raising, so check it. Never request a host your own `--!hosts` matches.

```lua
--!name=Enrich
--!hosts=full:api.example.com
--!kind=request

function http_request(request)
	local r = http({
		method = "GET",
		url = "https://worker.example.net/token",
		timeout = 5,
	})

	if r.error or r.status ~= 200 then
		log("token fetch failed", "warning")
		return nil
	end

	request.headers["Authorization"] = "Bearer " .. r.body
	return request
end
```

## Cache a lookup

The script environment is shared across packets, so a plain variable works as a cache. Add an expiry while you're at it.

```lua
--!name=Cached Token
--!hosts=full:api.example.com
--!kind=request

local cached_token = nil
local cached_at = 0

local function get_token()
	local now = time.unix()
	if cached_token and now - cached_at < 300 then
		return cached_token
	end

	local r = http({ url = "https://worker.example.net/token", timeout = 5 })
	if r.error or r.status ~= 200 then
		return cached_token   -- keep using the old one
	end

	cached_token = r.body
	cached_at = now
	return cached_token
end

function http_request(request)
	local token = get_token()
	if token then
		request.headers["Authorization"] = "Bearer " .. token
	end
	return request
end
```

## Patch a protobuf field

You have to work out the field number yourself first. Remember to put the leading header back when reassembling.

```lua
--!name=Patch Protobuf
--!hosts=full:grpc.example.com
--!kind=request

function http_request(request)
	local ok, fields, offset = pcall(protobuf.decode, request.body)
	if not ok or not fields then
		return nil
	end

	local changed = false
	for _, f in ipairs(fields) do
		if f.n == 2 and f.wt == 0 then
			f.v = 1
			changed = true
		end
	end
	if not changed then
		return nil
	end

	request.body = request.body:sub(1, offset - 1) .. protobuf.encode(fields)
	return request
end
```

## Watch without changing anything

To find out what an app is calling, return `nil`. The output shows up on the app's log page, tagged with `--!name`.

```lua
--!name=Inspect
--!hosts=regexp:\.example\.com$
--!kind=request,response

function http_request(request)
	log(request.method .. " " .. request.url)
	return nil
end

function http_response(response)
	log(response.status .. " " .. (response.host or "?") .. " " .. #response.body .. "B")
	return nil
end
```
