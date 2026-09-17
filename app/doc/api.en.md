# API

All of the following are globals — just call them.

Lua strings can hold arbitrary bytes, so bodies, keys, and hash results are all passed around as plain strings.

There are two failure styles. Codec functions (compression, hex, base64) return `nil`. Bad arguments, crypto failures, and JSON conversion errors raise instead, so they need `pcall` to avoid aborting your handler. Each section says which applies.

## Compression

```lua
gzip.encode(s)   gzip.decode(s)   gunzip(s)
zlib.encode(s)   zlib.decode(s)
zstd.encode(s)   zstd.decode(s)
```

Returns `nil` on failure. `gunzip` is an alias for `gzip.decode`. `zlib` corresponds to `Content-Encoding: deflate`.

`zstd.decode` needs the original length embedded in the data; content produced by streaming compression can't be decoded and returns `nil`.

When handling a response, check `encodings` first:

```lua
local raw = response.body
if response.encodings.gzip then
	raw = gzip.decode(raw) or raw
elseif response.encodings.zstd then
	raw = zstd.decode(raw) or raw
end
```

## Hashes and HMAC

```lua
sha256(s)   sha1(s)   md5(s)
hmac.sha256(key, data)   hmac.sha1(key, data)
```

The three hashes return a **lowercase hex string**; the two HMACs return **raw bytes**. Run the latter through `hex.encode` if you need hex.

```lua
local mac = hmac.sha256(secret, request.body)
request.headers["X-Sign"] = hex.encode(mac)
```

## hex and base64

```lua
hex.encode(s)      hex.decode(s)
base64.encode(s)   base64.decode(s)
```

Encoding never fails. Both `decode` functions return `nil` on malformed input. Surrounding whitespace and newlines are stripped automatically.

## JSON

```lua
json.decode(s)
json.encode(t)
json.encode_utf8(t)
```

These three **raise** on failure, so wrap them in `pcall` when the input isn't trusted.

`json.decode` accepts a complete JSON document or a bare value. Objects become tables; arrays become tables keyed `1..n`.

`json.encode` goes the other way: a table with consecutive integer keys starting at 1 and no string keys becomes an array, anything else becomes an object, and an empty table becomes `[]`.

Strings must be valid UTF-8 or the call raises — which happens as soon as one binary field ends up in the body. Use `json.encode_utf8` in that case: strictly valid UTF-8 strings are kept as-is; common Apple text framing (`0x0A` + length byte) is stripped before decoding UTF-8; stray bytes appear as `\u00xx` in the JSON. The trade-off is that it's lossy: `json.decode` cannot restore the original bytes. Use `base64.encode` when you need the data preserved.

```lua
local ok, obj = pcall(json.decode, response.body)
if not ok then
	return nil
end
obj.patched = true
response.body = json.encode(obj)
```

## Cookies and query strings

```lua
cookie.parse(s)   cookie.build(t)
query.parse(s)    query.build(t)
```

`cookie` handles the request-side Cookie header (`a=1; b=2`), not `Set-Cookie`.

`query` handles `application/x-www-form-urlencoded`, which covers both query strings and form bodies. Parsing performs percent-decoding and treats `+` as a space. Full URLs (`https://host/path?a=1`), `?a=1` forms, and plain query strings (`a=1&b=2`) are all accepted; the query portion is extracted automatically when needed.

Neither `build` **preserves the original order**. If what you're touching is order-sensitive — some signature schemes are — assemble the string yourself.

```lua
local c = cookie.parse(request.headers["Cookie"] or "")
c.session = "new-token"
request.headers["Cookie"] = cookie.build(c)
```

## Time and randomness

```lua
time.unix()        -- integer seconds
time.iso8601()     -- "2026-08-24T06:01:00Z"
uuid()             -- uppercase UUID string
random_bytes(n)    -- n random bytes, n from 1 to 4096
```

`uuid()` is uppercase; call `:lower()` yourself if you need it lowercase.

## AES

```lua
aes.encrypt(mode, key, iv, data)
aes.decrypt(mode, key, iv, data)
```

`mode` is either `"cbc"` or `"gcm"`. The key must be 16, 24, or 32 bytes.

CBC uses PKCS7 padding with a 16-byte IV. GCM takes a 12-byte IV, and the encrypted result is the ciphertext followed directly by a 16-byte tag — pass the whole thing back in to decrypt.

All errors are raised, including GCM authentication failures.

```lua
local iv = random_bytes(12)
local sealed = aes.encrypt("gcm", key, iv, plaintext)
```

## protobuf

There is no schema, so you get field numbers rather than names.

```lua
local fields, offset = protobuf.decode(data)
local fields, offset = protobuf.decode(start, data)
local bytes = protobuf.encode(fields)
```

`decode` returns the fields plus the offset where the message actually starts. That second value matters because plenty of APIs prepend a header of their own.

With a single argument it searches for the start; with `start` it begins there, falling back to searching if `start` is invalid. It raises if no valid message is found.

Fields look like this:

```lua
{
	{ n = 1, wt = 0, v = 123 },
	{ n = 2, wt = 2, v = "hello" },
	{ n = 3, wt = 2, v = { { n = 1, wt = 0, v = 1 } } },
	{ n = 4, wt = 1, v = 4627729705071802319, d = 24.998625 },
}
```

`n` is the field number and `wt` the wire type (0, 1, 2, 5). A `wt` of 2 whose contents also look like protobuf is decoded into a nested table, otherwise it stays as bytes — this guess is occasionally wrong, so check the type when it matters.

Without a schema there is no way to be sure whether a payload is a nested message or opaque bytes, so anything that fails to parse falls back to bytes. Hitting a parser limit (nesting past 32 levels, more than 2048 fields in one message, more than 512KB of data) raises instead of falling back — the data is fine and the parser gave up, so degrading silently would just hand you the wrong structure.

Field numbers are validated more strictly than in `protoc --decode_raw`: when a tag varint exceeds 32 bits, protoc truncates it to a wrong field number and accepts the message anyway, while this decoder raises.

Wire types 1 (fixed64) and 5 (fixed32) are ambiguous between integers and floats without a schema, so both readings are provided: `v` is the exact integer bit pattern and `d` is the IEEE 754 reading. Read `d` for coordinates, distances, and similar values.

For `encode`, `wt` has to agree with the type of `v` or the call raises. When `wt` is 1 or 5 and `v` is omitted, `d` is encoded as a float instead.

## content.flags

```lua
content.flags(headers)
```

Inspects `Content-Encoding` and `Content-Type` only; it decodes nothing. Returns seven booleans:

```lua
{ gzip, zlib, zstd, protobuf, messagepack, cbor, bson }
```

The `encodings` field on a packet is the same table already computed for you, so most of the time you can just read `response.encodings`. This function is for when you hold a different set of headers, such as a response from `http()`.

## http

```lua
http({ method = "GET", url = "...", headers = {}, body = "", timeout = 10 })
```

Only `url` is required. `method` defaults to GET; `timeout` is in seconds, defaults to 10, and is clamped between 1 and 30.

**Only callable inside `http_request` / `http_response`.** At the top level of a script it fails and the whole script won't load.

Success returns `{ status, headers, body }`, failure returns `{ error = "..." }`. Failures don't raise, so always check:

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

Don't request a host your own `--!hosts` matches — the request re-enters this same script.

## log

```lua
log(msg)
log(msg, level)
```

Writes a line to the app's log, tagged with the script's `--!name`.

`level` can be `debug`, `info`, `success`, `warning`, or `error`, defaulting to `info`. The log page colours these differently.

`msg` doesn't have to be a string; any value is converted to text. Tables print their address rather than their contents, so run them through `json.encode` first.

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

Keeps data across packets — see [persist](persist.en.md).
