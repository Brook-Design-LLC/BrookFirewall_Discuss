# API

以下函式都是全域的，直接呼叫即可。

Lua 的 string 可以裝任意位元組，所以 body、金鑰、雜湊結果都直接用 string 傳遞。

失敗的方式有兩種：編解碼類（壓縮、hex、base64）回傳 `nil`；參數錯誤、加解密失敗、JSON 轉換失敗則是拋出錯誤，需要 `pcall` 才不會中斷。每一節都會註明。

## 壓縮

```lua
gzip.encode(s)   gzip.decode(s)   gunzip(s)
zlib.encode(s)   zlib.decode(s)
zstd.encode(s)   zstd.decode(s)
```

失敗回傳 `nil`。`gunzip` 是 `gzip.decode` 的別名。`zlib` 對應 `Content-Encoding: deflate`。

`zstd.decode` 需要資料本身帶有原始長度，串流方式壓出來的會解不開，回傳 `nil`。

處理回應時先看 `encodings` 再決定要不要解：

```lua
local raw = response.body
if response.encodings.gzip then
	raw = gzip.decode(raw) or raw
elseif response.encodings.zstd then
	raw = zstd.decode(raw) or raw
end
```

## 雜湊與 HMAC

```lua
sha256(s)   sha1(s)   md5(s)
hmac.sha256(key, data)   hmac.sha1(key, data)
```

三個雜湊回傳**小寫 hex 字串**，兩個 HMAC 回傳**原始 bytes**。需要 hex 就再過一次 `hex.encode`。

```lua
local mac = hmac.sha256(secret, request.body)
request.headers["X-Sign"] = hex.encode(mac)
```

## hex 與 base64

```lua
hex.encode(s)      hex.decode(s)
base64.encode(s)   base64.decode(s)
```

編碼方向不會失敗，兩個 `decode` 在輸入不合法時回傳 `nil`。前後的空白和換行會自動去掉。

## JSON

```lua
json.decode(s)
json.encode(t)
json.encode_utf8(t)
```

這三個失敗時是**拋錯**，處理來路不明的資料請用 `pcall`。

`json.decode` 接受完整 JSON，也接受單獨的值。物件變成 table，陣列變成鍵為 `1..n` 的 table。

`json.encode` 反過來：整數鍵、從 1 開始連續、又沒有字串鍵的 table 編成陣列，其餘編成物件，空 table 編成 `[]`。

字串必須是合法 UTF-8，否則拋錯——body 裡混一個二進位欄位就會遇到。這時改用 `json.encode_utf8`：已是合法 UTF-8 的字串原樣輸出；常見的 Apple 文字 framing（`0x0A` + 長度 byte）會先剝掉再解 UTF-8；其餘無效 byte 以 `\u00xx` 出現在 JSON 裡。代價是不可逆，`json.decode` 無法還原成原本的位元組；需要保留原始資料請用 `base64.encode`。

```lua
local ok, obj = pcall(json.decode, response.body)
if not ok then
	return nil
end
obj.patched = true
response.body = json.encode(obj)
```

## Cookie 與 query string

```lua
cookie.parse(s)   cookie.build(t)
query.parse(s)    query.build(t)
```

`cookie` 處理的是請求端的 Cookie 表頭（`a=1; b=2`），不處理 `Set-Cookie`。

`query` 處理 `application/x-www-form-urlencoded`，query string 和表單 body 都適用，parse 會做百分號解碼並把 `+` 當空白。也接受完整 URL（`https://host/path?a=1`）或 `?a=1` 形式，會自動取出 `?` 後的 query；純 query string（`a=1&b=2`）同樣支援。

兩個 build 出來的**順序不保證**和原本一樣。如果你要動的東西對參數順序敏感（例如某些簽章），請自己拼字串。

```lua
local c = cookie.parse(request.headers["Cookie"] or "")
c.session = "new-token"
request.headers["Cookie"] = cookie.build(c)
```

## 時間與隨機

```lua
time.unix()        -- 整數秒
time.iso8601()     -- "2026-08-24T06:01:00Z"
uuid()             -- 大寫 UUID 字串
random_bytes(n)    -- n 個隨機 bytes，n 為 1..4096
```

`uuid()` 是大寫，需要小寫請自己 `:lower()`。

## AES

```lua
aes.encrypt(mode, key, iv, data)
aes.decrypt(mode, key, iv, data)
```

`mode` 是 `"cbc"` 或 `"gcm"`。key 長度必須是 16、24 或 32 bytes。

CBC 用 PKCS7，iv 為 16 bytes。GCM 的 iv 為 12 bytes，加密結果是密文後面直接接 16 bytes 的 tag，解密時整包傳回去即可。

錯誤都是拋出來的，包含 GCM 驗證失敗。

```lua
local iv = random_bytes(12)
local sealed = aes.encrypt("gcm", key, iv, plaintext)
```

## protobuf

沒有 schema，所以你拿到的是欄位編號而不是名稱。

```lua
local fields, offset = protobuf.decode(data)
local fields, offset = protobuf.decode(start, data)
local bytes = protobuf.encode(fields)
```

`decode` 回傳欄位陣列，以及資料實際開始的位置。第二個回傳值有用，是因為不少 API 會在 protobuf 前面加一段自己的標頭。

只給一個參數時會自動尋找開頭；給了 `start` 就從那裡開始，`start` 無效時退回自動尋找。找不到合法訊息會拋錯。

欄位長這樣：

```lua
{
	{ n = 1, wt = 0, v = 123 },
	{ n = 2, wt = 2, v = "hello" },
	{ n = 3, wt = 2, v = { { n = 1, wt = 0, v = 1 } } },
	{ n = 4, wt = 1, v = 4627729705071802319, d = 24.998625 },
}
```

`n` 是欄位編號，`wt` 是 wire type（0、1、2、5）。`wt` 為 2 的欄位如果內容看起來也像 protobuf，會自動解成巢狀 table，否則當成 bytes——偶爾會判斷錯，需要時自己檢查型別。

沒有 schema 就無法確定一段內容是巢狀訊息還是不透明 bytes，所以解不開時退回 bytes。但如果是撞到解析上限（巢狀超過 32 層、單一訊息超過 2048 個欄位、資料超過 512KB）就會拋錯，不會默默退回 bytes——那代表資料是好的而解析器放棄了，靜默退化只會讓你拿到錯的結構。

解析器在欄位編號上比 `protoc --decode_raw` 嚴格：tag varint 超過 32 bit 時 protoc 會截斷成一個錯的欄位編號並照樣接受，這裡直接拋錯。

`wt` 為 1（fixed64）和 5（fixed32）沒有 schema 就分不出整數還是浮點，所以兩種解讀都給：`v` 是精確的位元組整數，`d` 是 IEEE 754 解讀。經緯度、距離這類欄位讀 `d`。

`encode` 的 `wt` 必須和 `v` 的型別對得上，否則拋錯。`wt` 為 1 或 5 時若省略 `v`，會改用 `d` 編碼成浮點。

## content.flags

```lua
content.flags(headers)
```

只看 `Content-Encoding` 和 `Content-Type` 判斷格式，不做解碼。回傳七個 boolean：

```lua
{ gzip, zlib, zstd, protobuf, messagepack, cbor, bson }
```

封包上的 `encodings` 就是同一張表，所以多數時候直接讀 `response.encodings` 即可。這個函式用在你手上有另一組表頭時，例如 `http()` 拿回來的回應。

## http

```lua
http({ method = "GET", url = "...", headers = {}, body = "", timeout = 10 })
```

只有 `url` 必填。`method` 預設 GET，`timeout` 單位是秒、預設 10、範圍 1 到 30。

**只能在 `http_request` / `http_response` 裡呼叫**，寫在腳本最外層會讓整支腳本載入失敗。

成功回傳 `{ status, headers, body }`，失敗回傳 `{ error = "..." }`。失敗不會拋錯，一定要自己檢查：

```lua
local r = http({
	method = "POST",
	url = "https://example.com/v1/verify",
	headers = { ["Content-Type"] = "application/json" },
	body = json.encode({ token = token }),
	timeout = 5,
})
if r.error then
	log(r.error, "warning")
	return nil
end
```

不要對自己 `--!hosts` 命中的網域發請求，那會重新進入這支腳本。

## log

```lua
log(msg)
log(msg, level)
```

寫一行到 App 日誌，tag 是腳本的 `--!name`。

`level` 可以是 `debug`、`info`、`success`、`warning`、`error`，不填當 `info`。日誌頁會用不同顏色標示這幾種等級。

`msg` 不必是字串，任何值都會轉成文字，但 table 會印出位址而不是內容，想看內容請自己 `json.encode`。

```lua
log("hit " .. request.host)
log("patched " .. request.path, "success")
log(json.encode(request.headers), "debug")
```

## persist

```lua
persist.get(key)
persist.set(key, value)
persist.delete(key)
```

跨封包保存資料，見 [persist](persist.md)。
