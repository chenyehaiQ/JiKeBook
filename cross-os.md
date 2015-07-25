# 跨平台的库选择

大家看过上面三个 json 的例子就发现，都是围绕 cjson 库的。原因也比较简单，就是 cjson 是默认绑定到 openresty 上的。所以在 linux 环境下我们也默认的使用了他。在 360 天擎项目中， linux 用户只是很少量的一部分。大部分用户更多的是 windows 操作系统，但 cjson 目前还没有 windows 版本。所以对于 windows 用户，我们只能选择 dkjson （编解码效率没有 cjson 快，优势是纯 lua ，完美跨任何平台）。

并且我们的代码肯定不会因为 win 、 linux 的并存而写两套程序。那么我们就必须要把 json 处理部分封装一下，隐藏系统差异造成的差异化处理。

```lua
local _M = { _VERSION = '1.0' }
-- require("ffi").os 获取系统类型
local json = require(require("ffi").os == "Windows" and "dkjson" or "cjson")

function _M.json_decode( str )
    return json.decode(str)
end
function _M.json_encode( data )
    return json.encode(data)
end

return _M

```

在我们的应用中，对于操作系统版本差异、操作系统位数差异、同时支持不通数据库使用等，几乎都是使用这个方法完成的，十分值得推荐。

额外说个点， github 上有个项目[cloudflare/lua-resty-json](https://github.com/cloudflare/lua-resty-json)，从官方资料上介绍 decode 的速度更快，我们也做了小范围应用。所以上面的 decode json 对象来源，就可以改成这个库。世界总是有新鲜玩意，多了解多发发现，然后再充实自己吧。

