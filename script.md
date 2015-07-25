# script 压缩复杂请求
从[pipeline 那一章节](https://github.com/moonbingbing/openresty-best-practices/blob/master/redis/pipeline.md)，我们知道对于多个简单的 redis 命令可以汇聚到一个请求中，提升服务端的并发能力。然而，在有些场景下，我们每次命令的输入需要引用上个命令的输出，甚至可能还要对第一个命令的输出做一些加工，再把加工结果当成第二个命令的输入。 pipeline 难以处理这样的场景。庆幸的是，我们可以用 redis 里的 script 来压缩这些复杂命令。

script 的核心思想是在 redis 命令里嵌入 Lua 脚本，来实现一些复杂操作。 Redis 中和脚本相关的命令有：
- EVAL
- EVALSHA
- SCRIPT EXISTS
- SCRIPT FLUSH
- SCRIPT KILL
- SCRIPT LOAD

官网上给出了这些命令的基本语法，感兴趣的同学可以到[这里](http://redis.io/commands/eval)查阅。其中 EVAL 的基本语法如下：
>EVAL script numkeys key [key ...] arg [arg ...]

EVAL 的第一个参数*script*是一段 Lua 脚本程序。 这段 Lua 脚本不需要（也不应该）定义函数。它运行在 Redis 服务器中。
EVAL 的第二个参数*numkeys*是参数的个数，后面的参数*key*（从第三个参数），表示在脚本中所用到的那些 Redis 键(key)，这些键名参数可以在 Lua 中通过全局变量 KEYS 数组，用 1 为基址的形式访问( KEYS[1] ， KEYS[2] ，以此类推)。
在命令的最后，那些不是键名参数的附加参数*arg [arg ...]* ，可以在 Lua 中通过全局变量 ARGV 数组访问，访问的形式和 KEYS 变量类似( ARGV[1] 、 ARGV[2] ，诸如此类)。下面是执行 eval 命令的简单例子：
```
eval "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}" 2 key1 key2 first second
1) "key1"
2) "key2"
3) "first"
4) "second"
```

openresty 中已经对 redis 的所有原语操作进行了封装。下面我们以 EVAL 为例，来看一下 openresty 中如何利用 script 来压缩请求：
```
# you do not need the following line if you are using
    # the ngx_openresty bundle:
    lua_package_path "/path/to/lua-resty-redis/lib/?.lua;;";

    server {
        location /usescript {
            content_by_lua '
            local redis = require "resty.redis"
            local red = redis:new()

            red:set_timeout(1000) -- 1 sec

            -- or connect to a unix domain socket file listened
            -- by a redis server:
            --     local ok, err = red:connect("unix:/path/to/redis.sock")

            local ok, err = red:connect("127.0.0.1", 6379)
            if not ok then
                ngx.say("failed to connect: ", err)
                return
            end

            --- use scripts in eval cmd
            local id = "1"
            ok, err = red:eval([[
                local info = redis.call('get', KEYS[1])
                info = json.decode(info)
                local g_id = info.gid

                local g_info = redis.call('get', g_id)
                return g_info
                ]], 1, id)

            if not ok then
               ngx.say("failed to get the group info: ", err)
               return
            end

            -- put it into the connection pool of size 100,
            -- with 10 seconds max idle time
            local ok, err = red:set_keepalive(10000, 100)
            if not ok then
                ngx.say("failed to set keepalive: ", err)
                return
            end

            -- or just close the connection right away:
            -- local ok, err = red:close()
            -- if not ok then
            --     ngx.say("failed to close: ", err)
            --     return
            -- end
          ';
        }
      }
```

从上面的例子可以看到，我们要根据一个对象的 id 来查询该 id 所属 gourp 的信息时，我们的第一个命令是从 redis 中读取 id 为 1 （ id 的值可以通过参数的方式传递到 script 中）的对象的信息（由于这些信息一般 json 格式存在 redis 中，因此我们要做一个解码操作，将 info 转换成 lua 对象）。然后提取信息中的 groupid 字段，以 groupid 作为 key 查询 groupinfo 。这样我们就可以把两个 get 放到一个 TCP 请求中，做到减少 TCP 请求数量，减少网络延时的效果啦。
