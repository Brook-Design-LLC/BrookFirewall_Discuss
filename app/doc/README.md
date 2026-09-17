# MITM Lua 腳本

用 Lua 5.4 寫的腳本，可以在 HTTP 請求送出前、或回應交回 App 前把它攔下來修改。

使用前有兩個前提：裝置必須已經信任 App 的根憑證，腳本也必須在 metadata 裡宣告要攔哪些網域。

- [API 一覽](api.md)
- [persist](persist.md)
- [範例](examples.md)

## 最小的腳本

```lua
--!name=Hello
--!hosts=full:api.example.com
--!kind=response

function http_response(response)
	response.headers["X-Hello"] = "world"
	return response
end
```

存成 `.lua` 上傳，或直接用內建編輯器貼上。

## Metadata

開頭的 `--!` 註解就是設定。它必須放在所有程式碼之前，一旦出現第一行不是 `--!` 的內容就不再往下讀（空行和一般的 `--` 註解不影響）。

| 欄位 | 作用 |
| --- | --- |
| `name` | 顯示名稱，也是寫 log 時的 tag。不填就用檔名 |
| `hosts` | 要攔哪些網域，逗號分隔 |
| `host-regex` | 只接受正規表示式，等同 `regexp:`，可以寫很多行 |
| `kind` | `request`、`response`，或用逗號寫兩個。不填就兩種都跑 |

### hosts 的四種寫法

| 寫法 | 怎麼比對 |
| --- | --- |
| `keyword:gs-loc` | 網域只要包含這段字就命中，不分大小寫 |
| `full:api.example.com` | 網域必須完全相等 |
| `regexp:\.apple\.com$` | 正規表示式，不分大小寫 |
| `fake:local.brook.api` | 完全相等，另外還會讓這個網域指向本機，見下 |

沒寫前綴時當成 `keyword`。前綴打錯（例如 `ful:`）不會報錯，整串會被當成 keyword 拿去比對，所以要看清楚。

`regexp:` 是部分匹配，沒有自動加錨點——`regexp:api` 會命中所有含 `api` 的網域。要限定結尾請自己寫 `$`。

**沒有寫 `hosts` 也沒寫 `host-regex` 的腳本不會執行**，設定頁上會標示出來。

### fake：做一支不存在的 API

`fake:` 會讓這個網域指向本機，App 因此連得出去，接著由你的腳本直接回應。沒有任何真實伺服器，但對 App 來說就是一支正常的 HTTPS API。做法見[範例](examples.md)。

## 兩個進入點

定義 `http_request`、`http_response`，或兩個都定義。沒定義的方向，封包會原樣通過。

傳進來的參數是一個 table：

| 欄位 | 內容 |
| --- | --- |
| `id` | 封包的 UUID，request 和它的 response 共用同一個 |
| `method` | `GET`、`POST` 之類 |
| `url` | 完整網址 |
| `host` | 純網域，不含 port |
| `path` | 路徑加 query string |
| `status` | 狀態碼，request 上是 nil |
| `headers` | 表頭的 table，名稱維持原本的大小寫 |
| `body` | 原始內容，可以是二進位 |
| `encodings` | 從表頭判斷出的格式，內容同 `content.flags` |

`body` 拿到的是原始位元組。對方回壓縮內容時這裡就是壓縮後的資料，要自己解，`encodings` 會告訴你該用哪個。

## 回傳值

**放行**：回傳 `nil`。封包完全不會被動到。

**修改後轉發**：回傳一個 table，只放你要改的欄位，沒提到的維持原狀。

```lua
function http_response(response)
	response.status = 404
	return response
end

function http_request(request)
	return { body = "replaced" }   -- 只改 body
end
```

可以改 `headers`、`body`、`host`、`method`、`path`，以及回應的 `status`。改 `host` 時 `Host` 表頭會跟著更新。

`headers` 是整個換掉而不是合併，所以要加表頭請在原本的 table 上改（`response.headers["X-Foo"] = "1"`），不要丟一個只有一組 key 的新 table 進去，那會把其他表頭全部清光。

改完 body 之後不必管 `Content-Length` 和 `Content-Encoding`，送出前會自動處理。

**短路**：只有 `http_request` 可以。回傳一個帶 `response` 的 table，請求就到此為止，遠端不會被連上。

```lua
return {
	response = {
		status = 403,
		body = "nope",
		headers = { ["Content-Type"] = "text/plain; charset=utf-8" },
	}
}
```

`status` 預設 200，`body` 預設空的，沒給 `headers` 時 Content-Type 會補上 `application/json; charset=utf-8`。

## 可以用什麼

環境是精簡過的，標準庫只有 `base`、`table`、`string`、`math`、`utf8`。

沒有 `io`、`os`、`package`、`debug`、`coroutine`，也沒有 `print`、`load`、`dofile`、`loadfile`。要輸出訊息請用 `log`，它會寫進 App 的日誌。

其餘可用的函式（JSON、壓縮、雜湊、加解密、發出站請求等）見 [API 一覽](api.md)。

## 限制

| 項目 | 上限 |
| --- | --- |
| 單次執行 | 100ms |
| 記憶體 | 1.5MB |
| persist | 128KB |
| 壓縮、加解密、`http()` 的輸出 | 各 512KB |
| `http()` 逾時 | 預設 10 秒，可設 1 到 30 秒 |

`http()` 等待的時間不算進 100ms，所以連續發幾個請求不會超時，跑一個大迴圈才會。

腳本超時、記憶體不足、或有沒接住的錯誤時，這次的修改會放棄、封包原樣放行，訊息寫進日誌。寫壞的腳本不會擋住流量。

## 狀態會活多久

每支腳本的環境在載入時建立一次，最外層的程式碼也只跑那一次，所以你宣告的變數是跨封包共用的：

```lua
local count = 0

function http_request(request)
	count = count + 1
	log("已處理 " .. count .. " 個請求")
	return nil
end
```

但這份記憶只在當前連線期間有效。腳本清單只要有變動（新增、刪除、開關、改內容），或 VPN 重連，所有腳本都會重新載入，變數歸零。[persist](persist.md) 也是一樣，兩者都不會寫進磁碟。
