# 编码为 array 还是 object

首先大家请看这段源码：
```lua
-- http://www.kyne.com.au/~mark/software/lua-cjson.php
-- version: 2.1 devel

local json = require("cjson")
ngx.say("value --> ", json.encode({dogs={}}))
```

输出结果
> value --> {"dogs":{}}

注意看下 encode 后 key 的值类型，"{}" 代表 key 的值是个 object ，"[]" 则代表 key 的值是个数组。对于强类型语言(c/c++, java 等)，这时候就有点不爽。因为类型不是他期望的要做容错。对于 lua 本身，是把数组和字典融合到一起了，所以他是无法区分空数组和空字典的。

参考 openresty-cjson 中额外贴出测试案例，我们就很容易找到思路了。

```lua
-- 内容节选 lua-cjson-2.1.0.2/tests/agentzh.t
=== TEST 1: empty tables as objects
--- lua
local cjson = require "cjson"
print(cjson.encode({}))
print(cjson.encode({dogs = {}}))
--- out
{}
{"dogs":{}}


=== TEST 2: empty tables as arrays
--- lua
local cjson = require "cjson"
cjson.encode_empty_table_as_object(false)
print(cjson.encode({}))
print(cjson.encode({dogs = {}}))
--- out
[]
{"dogs":[]}
```

综合本章节提到的各种问题，我们可以封装一个 json encode 的示例函数：

```lua
function json_encode( data, empty_table_as_object )
  --lua 的数据类型里面， array 和 dict 是同一个东西。对应到 json encode 的时候，就会有不同的判断
  --对于 linux ，我们用的是 cjson 库： A Lua table with only positive integer keys of type number will be encoded as a JSON array. All other tables will be encoded as a JSON object.
  --cjson 对于空的 table ，就会被处理为 object ，也就是{}
  --dkjson 默认对空 table 会处理为 array ，也就是[]
  --处理方法：对于 cjson ，使用 encode_empty_table_as_object 这个方法。文档里面没有，看源码
  --对于 dkjson ，需要设置 meta 信息。 local a= {}； a.s = {};a.b='中文';setmetatable(a.s,  { __jsontype = 'object' });ngx.say(comm.json_encode(a))

    local json_value = nil
    if json.encode_empty_table_as_object then
        json.encode_empty_table_as_object(empty_table_as_object or false) -- 空的 table 默认为 array
    end
    if require("ffi").os ~= "Windows" then
        json.encode_sparse_array(true)
    end
    pcall(function (data) json_value = json.encode(data) end, data)
    return json_value
end
```
