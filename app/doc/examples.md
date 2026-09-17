# 範例

每一份都是完整的腳本，改掉網域就能用。

## 擋掉一個網域

在 `http_request` 裡回傳一個 `response`，請求就到此為止。

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

## 加一個表頭

在原本的 table 上改再回傳。不要整個換掉 `headers`，那會清光其他表頭。

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

## 改寫回應的 JSON

回應多半是壓縮過的，先看 `encodings` 決定要不要解。改完直接寫回 `body` 即可。

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

## 只處理特定路徑

同一個網域下通常只有幾支 API 要動，其他早點放行。

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

## 做一支不存在的 API

`fake:` 會讓這個網域指向本機，再由腳本直接回應。裝好之後 App 打 `https://local.brook.api/hello?a=1` 就會拿到這段 JSON。

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

## 幫請求簽名

簽章要對最終送出去的內容計算，所以先改 body、再算簽章。

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

## 向外部服務查詢

`http()` 失敗是回傳 `error` 而不是拋錯，要自己檢查。不要對自己 `--!hosts` 命中的網域發請求。

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

## 快取查詢結果

腳本環境跨封包共用，用一般變數就能當快取，順便設個過期時間。

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
		return cached_token   -- 拿不到就沿用舊的
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

## 改 protobuf 的欄位

要先自己觀察出目標欄位的編號。重組時記得把前面那段標頭接回去。

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

## 只看不改

想知道某個 App 在打什麼 API 的時候，回傳 `nil` 就好。日誌可以在 App 的日誌頁看到，tag 是 `--!name`。

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
