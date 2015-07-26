# 调试
调试是一个程序猿非常重要的能力，人写的程序总会有 bug ，所以需要 debug 。***如何方便和快速的定位 bug***，是我们讨论的重点，只要 bug 能定位，解决就不是问题。

对于熟悉用 Visual Studio 和 Eclipse 这些强大的集成开发环境的来做 C++和 Java 的同学来说， OpenResty 的 debug 要原始很多，但是对于习惯 Python 开发的同学来说，又是那么的熟悉。张银奎有本[《软件调试》](http://www.amazon.cn/%E8%BD%AF%E4%BB%B6%E8%B0%83%E8%AF%95-Software-Debugging-%E5%BC%A0%E9%93%B6%E5%A5%8 E/dp/B001 AUKASG)的书， windows 客户端程序猿应该都看过，大家可以去试读下，看看里面有多复杂:(

对于 OpenResty ，坏消息是，没有单步调试这些玩意儿（我们尝试搞出来过 ngx lua 的单步调试，但是没人用...）;好消息是，它像 Python 一样，非常简单，不用复杂的技术，只靠 print 和 log 就能定位绝大部分问题，难题有[火焰图](images/flame-graphic.svg)这个神器。

* ####关闭 code cache
这个选项在调试的时候最好关闭。
```lua
lua_code_cache off;
```
这样，你修改完代码后，不用 reload nginx 就可以生效了。在生产环境下记得打开这个选项。

* ####记录日志

这个看上去谁都会的东西，要想做好也不容易。

你有遇到这样的情况吗？ QA 发现了一个 bug ，开发说我修改代码加个日志看看，然后 QA 重现这个问题，发现日志不够详细，需要再加，反复几次，然后再给 QA 一个没有日志的版本，继续测试其他功能。

如果产品已经发布到用户那里了呢？如果用户那里是隔离网，不能远程怎么办？

***你在写代码的时候，就需要考虑到调试日志。*** 比如这个代码：
```lua
local response, err = redis_op.finish_client_task(client_mid, task_id)
if response then
    put_job(client_mid, result)
    ngx.log(ngx.WARN, "put job:", common.json_encode({channel="task_status", mid=client_mid, data=result}))
end
```
我们在做一个操作后，就把结果记录到 nginx 的 error.log 里面，等级是 warn 。在生产环境下，日志等级默认为 error ，在我们需要详细日志的时候，把等级调整为 warn 即可。在我们的实际使用中，我们会把一些很少发生的重要事件，做为 error 级别记录下来，即使它并不是 nginx 的错误。

与日志配套的，你需要[logrotate](http://linuxcommand.org/man_pages/logrotate8.html)来做日志的切分和备份。
