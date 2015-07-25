#ngx.shared.DICT

ngx.shared.DICT 是基于黑红树结构和 LRU 算法的 kv 存储，不支持数据持久化，提供有限的数据类型的支持。

适用范围： cache ，会话间同步数据；  
不适用：数据持久化，日志记录，大容量数据，队列性质的暂存；

OpenResty 中[lua-resty-lock](https://github.com/openresty/lua-resty-lock)组件也是基于 shared.DICT 实现的

*如果你使用的是由奇虎公司提供的 windows 平台下的[mul-worker](https://github.com/LomoX-Offical/nginx-openresty-windows)版本，那么在使用 ngx.shared.DICT 的时候需要注意的是每一个 Key 占用的最小内存是 4 K ，即是 1 M 内存只能存储 256 个 key-value 对*。

主要函数和执行阶段请参考维基百科[HttpLuaModule#ngx.shared.DICT](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT)
