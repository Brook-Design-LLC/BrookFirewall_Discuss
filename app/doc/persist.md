# persist

把資料留到下一個封包。

```lua
persist.get(key)           -- 沒有就回傳 nil
persist.set(key, value)
persist.delete(key)
```

`persist:get(key)` 這種冒號寫法也支援，兩者等價。

`key` 必須是字串。`value` 可以是 nil、boolean、number、string 或 table。

## 什麼時候該用

腳本的環境跨封包共用，所以一般的變數就能保存資料：

```lua
local hits = 0

function http_request(request)
	hits = hits + 1
	return nil
end
```

計數器、旗標這類小東西這樣寫就夠了。

`persist` 的差別是它不佔腳本那 1.5MB 的記憶體額度，所以適合放比較大的資料，例如快取一份回應或累積一批 ID。

兩者都**不會寫進磁碟**。腳本清單一有變動或 VPN 重連，環境重新載入，變數和 persist 一起歸零。這裡存的是這次連線期間的狀態，不是設定。

## 128KB 上限

每次 `persist.set` 會把已存的全部內容加上這次要寫的資料一起計算，超過 128KB 時：

- 拋出 `persist: out of memory`
- 這次的值**不寫入**，先前存的**全部保留**
- 只要你接住這個錯誤，腳本會繼續跑下去

沒接住的話這次的處理算失敗，封包會原樣放行。

所以資料大小不完全可控時，用 `pcall` 包起來：

```lua
local ok, err = pcall(persist.set, persist, "cache", payload)
if not ok then
	log(err, "warning")
	persist.delete("cache")
end
```

注意 `pcall` 的參數：`persist` 本身要當第一個參數傳進去。

## 例子

記下每個網域最近一次的回應狀態。因為超限時舊資料還在，「滿了就重置」是安全的做法：

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
