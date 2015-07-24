# 正确的记录日志

看过本章第一节的同学应该还记得，log_by_lua是一个会话阶段最后发生的，文件操作是阻塞的（FreeBSD直接无视），nginx为了实时高效的给请求方应答后，日志记录是在应答后异步记录完成的。由此可见如果我们有日志输出的情况，最好统一到log_by_lua阶段。如果我们自定义放在content_by_lua阶段，那么将线性的增加请求处理时间。

在公司某个定制化项目中，nginx上的日志内容都要输送到syslog日志服务器。我们使用了[lua-resty-logger-socket](https://github.com/cloudflare/lua-resty-logger-socket)这个库。

> 调用示例代码如下（有问题的）：


```lua
-- lua_package_path "/path/to/lua-resty-logger-socket/lib/?.lua;;";
--
--    server {
--        location / {
--            content_by_lua lua/log.lua;
--        }
--    }

-- lua/log.lua
local logger = require "resty.logger.socket"
if not logger.initted() then
    local ok, err = logger.init{
        host = 'xxx',
        port = 1234,
        flush_limit = 1,   --日志长度大于flush_limit的时候会将msg信息推送一次
        drop_limit = 99999,
    }
    if not ok then
        ngx.log(ngx.ERR, "failed to initialize the logger: ",err)
        return
    end
end

local msg = string.format(.....)
local bytes, err = logger.log(msg)
if err then
    ngx.log(ngx.ERR, "failed to log message: ", err)
    return
end
```


在实测过程中我们发现了些问题：

* 缓存无效：如果flush_limit的值稍大一些（例如 2000），会导致某些体积比较小的日志出现莫名其妙的丢失，所以我们只能把flush_limit调整的很小
* 自己拼写msg所有内容，比较辛苦

那么我们来看[lua-resty-logger-socket](https://github.com/cloudflare/lua-resty-logger-socket)这个库的log函数是如何实现的呢，代码如下：
```lua
function _M.log(msg)  
   ...

    if (debug) then
        ngx.update_time()
        ngx_log(DEBUG, ngx.now(), ":log message length: " .. #msg)
    end

    local msg_len = #msg

    if (is_exiting()) then
        exiting = true
        _write_buffer(msg)
        _flush_buffer()
        if (debug) then
            ngx_log(DEBUG, "Nginx worker is exiting")
        end
        bytes = 0
    elseif (msg_len + buffer_size < flush_limit) then  -- 历史日志大小+本地日志大小小于推送上限
        _write_buffer(msg)
        bytes = msg_len
    elseif (msg_len + buffer_size <= drop_limit) then
        _write_buffer(msg)
        _flush_buffer()
        bytes = msg_len
    else
        _flush_buffer()
        if (debug) then
            ngx_log(DEBUG, "logger buffer is full, this log message will be "
                    .. "dropped")
        end
        bytes = 0
        --- this log message doesn't fit in buffer, drop it  

        ...
```

由于在content_by_lua阶段变量的生命周期会随着会话的终结而终结，所以当日志量小于flush_limit的情况下这些日志就不能被累积，也不会触发_flush_buffer函数，所以小日志会丢失。

这些坑回头看来这么明显，所有的问题都是因为我们把lua/log.lua用错阶段了，应该放到log_by_lua阶段，所有的问题都不复存在。

> 修正后：

```
 lua_package_path "/path/to/lua-resty-logger-socket/lib/?.lua;;";

    server {
        location / {
            content_by_lua lua/content.lua;
            log_by_lua lua/log.lua;
        }
    }
```

这里有个新问题，如果我的log里面需要输出一些content的临时变量，两阶段之间如何传递参数呢？

> 方法肯定有，推荐下面这个：

```
    location /test {
        rewrite_by_lua '
            ngx.say("foo = ", ngx.ctx.foo)
            ngx.ctx.foo = 76
        ';
        access_by_lua '
            ngx.ctx.foo = ngx.ctx.foo + 3
        ';
        content_by_lua '
            ngx.say(ngx.ctx.foo)
        ';
    }
```

更多有关ngx.ctx信息，请看[这里](http://wiki.nginx.org/HttpLuaModuleZh#ngx.ctx)。
