# sleep

这是一个比较常见的功能，你会怎么做呢？ Google 一下，你会找到[lua 的官方指南](http://lua-users.org/wiki/SleepFunction)，

里面介绍了 10 种 sleep 不同的方法（操作系统不一样，方法还有区别），选择一个用，然后你就杯具了:( 你会发现 nginx 高并发的特性不见了！

在 OpenResty 里面选择使用库的时候，有一个基本的原则：***尽量使用 ngx lua 的库函数，尽量不用 lua 的库函数，因为 lua 的库都是同步阻塞的。***

```
# you do not need the following line if you are using
# the ngx_openresty bundle:
lua_package_path "/path/to/lua-resty-redis/lib/?.lua;;";

server {
    location /non_block {
        content_by_lua '
            ngx.sleep(0.1)
        ';
    }
}
```

本章节内容好少，只是想通过一个真实的例子，来提醒大家，做 OpenResty 开发，[ngx lua 的文档](http://wiki.nginx.org/HttpLuaModule)是你的首选， lua 语言的库都是同步阻塞的，用的时候要三思。
