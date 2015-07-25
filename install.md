#如何安装火焰图生成工具

####安装 SystemTap
> 环境 CentOS 6.5 2.6.32-504.23.4.el6.x86_64 

SystemTap 是一个诊断 Linux 系统性能或功能问题的开源软件，为了诊断系统问题或性能，开发者或调试人员只需要写一些脚本，然后通过 SystemTap 提供的命令行接口就可以对正在运行的内核进行诊断调试。

首先需要安装内核开发包和调试包（这一步非常重要并且最为繁琐）：

```
# #Installaion:
# rpm -ivh kernel-debuginfo-($version).rpm
# rpm -ivh kernel-debuginfo-common-($version).rpm
# rpm -ivh kernel-devel-($version).rpm    
```

其中$version 使用 linux 命令 uname -r 查看，需要保证内核版本和上述开发包版本一致才能使用 systemtap 。([下载](http://debuginfo.centos.org))

安装 systemtap ：

```
# yum install systemtap
# ...
# 测试 systemtap 安装成功否：
# stap -v -e 'probe vfs.read {printf("read performed\n"); exit()}'

Pass 1: parsed user script and 103 library script(s) using 201628 virt/29508 res/3144 shr/26860 data kb, in 10 usr/190 sys/219 real ms.
Pass 2: analyzed script: 1 probe(s), 1 function(s), 3 embed(s), 0 global(s) using 296120 virt/124876 res/4120 shr/121352 data kb, in 660 usr/1020 sys/1889 real ms.
Pass 3: translated to C into "/tmp/stapffFP7 E/stap_82 c0 f95 e47 d351 a956 e1587 c4 dd4 cee1_1459_src.c" using 296120 virt/125204 res/4448 shr/121352 data kb, in 10 usr/50 sys/56 real ms.
Pass 4: compiled C into "stap_82 c0 f95 e47 d351 a956 e1587 c4 dd4 cee1_1459.ko" in 620 usr/620 sys/1379 real ms.
Pass 5: starting run.
read performed
Pass 5: run completed in 20 usr/30 sys/354 real ms.
```
如果出现如上输出表示安装成功。

####火焰图绘制

首先，需要下载 ngx 工具包：[Github 地址](https://github.com/openresty/nginx-systemtap-toolkit)，该工具包即是用 perl 生成 stap 探测脚本并运行的脚本，如果是要抓 lua 级别的情况，请使用工具 ngx-sample-lua-bt

```
# ps -ef | grep nginx  （ ps ：得到类似这样的输出，其中 15010 即使 worker 进程的 pid ，后面需要用到）
hippo    14857     1  0 Jul01 ?        00:00:00 nginx: master process /opt/openresty/nginx/sbin/nginx -p /home/hippo/skylar_server_code/nginx/main_server/ -c conf/nginx.conf
hippo    15010 14857  0 Jul01 ?        00:00:12 nginx: worker process
# ./ngx-sample-lua-bt -p 15010 --luajit20 -t 5 > tmp.bt （-p 是要抓的进程的 pid --luajit20|--luajit51 是 luajit 的版本 -t 是探测的时间，单位是秒， 探测结果输出到 tmp.bt ）
# ./fix-lua-bt tmp.bt > flame.bt  (处理 ngx-sample-lua-bt 的输出，使其可读性更佳)
```

其次，下载 Flame-Graphic 生成包：[Github 地址](https://github.com/brendangregg/FlameGraph),该工具包中包含多个火焰图生成工具，其中， stackcollapse-stap.pl 才是为 SystemTap 抓取的栈信息的生成工具

```
# stackcollapse-stap.pl flame.bt > flame.cbt
# flamegraph.pl flame.cbt > flame.svg
```
如果一切正常，那么会生成 flame.svg ，这便是火焰图，用浏览器打开即可。

####问题回顾

在整个安装部署过程中，遇到的最大问题便是内核开发包和调试信息包的安装，找不到和内核版本对应的，好不容易找到了又不能下载，@！￥#@……%@#，于是升级了内核，在后面的过程便没遇到什么问题。
ps ：如果在执行 ngx-sample-lua-bt 的时间周期内（上面的命令是 5 秒）,抓取的 worker 没有任何业务在跑，那么生成的火焰图便没有业务内容，不要惊讶哦~~~~~




