---
layout:     post
title:      分布式调用组件Qschedule介绍
author:     "冬至未"
header-style: text
tags:
    -Qunar
    -GC
    -Java
---

# 背景

最近，门票搜索发生了一次故障。在排查故障原因中，我们发现JVM在重启后会覆盖gc.log，导致问题现场丢失，后续排查问题非常困难，最后是通过监控判断宕机原因是JVM Full GC导致。

# 目的

找到一种新的gc日志打印方式，既可以避免在重启时被覆盖，又可以正常的被公司压缩脚本压缩和删除。

# 可选解决方案

GC日志文件路径是在JVM启动参数中设置的，公司内一般是 startenv.sh 中设置。以我们系统为例：

之前GC日志打印参数

```
-Xloggc:$CATALINA_BASE/logs/gc.log
```

这种打印方式会导致jvm重启时会将新的gc日志覆盖输出到 gc.log 中，从而导致旧的日志内容被覆盖。在调研过程中发现有几种可选方案，各有优缺点。

## 1.修改gc日志文件名

因为重启前后JVM的gc日志才导致被覆盖，所以我们自然想到更改gc日志文件名格式。新老JVM打印的gc日志文件名不一样就不会被覆盖了。在调研中，我们发现Java 可以定制gc日志文件名。

> Java在 -Xloggc 中提供了一些可选的命令行参数和扩展的标识符。
>
> 可用的标识符： ％p -Java进程的PID ％t -创建日志文件时的日期戳（格式：YYYY-MM-DD_HH-MM-SS。） ％n -如果启用了日志轮换逻辑，则为日志段ID，否则为“ 0” %% -文件名中的'％'转义字符
>
> 输入字符串中允许多个标识符和同一标识符的多个条目。
>
> 示例： 1）PID $ java -Xloggc：gc.％p.log ...＆ 1234
>
> GC日志：gc.1234.log
>
> 2）日期时间戳 $ date Thu Dec 22 22:07:17 MSK 2011 $ java -Xloggc：gc.％t.log ...
>
> GC日志：gc-2011-12-22_22-07-17.log
>
> 3）启用日志轮转时的日志段ID $ java -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=1M -Xloggc:gc.log.%n ...
>
> GC日志文件：gc.log.0，gc.log.2， gc.log.3，...，gc.log.9
>
> 4）禁用日志轮换时的日志段ID $ java -XX：+ UseGCLogFileRotation -Xloggc：gc.log。％n ...
>
> GC日志：gc.log .0
>
> 5）复杂示例 $ java -Xloggc：gc.％t.％p.log.％n
>
> GC日志：gc-2011-12-22_22-07-17.1234.log.0

因此我们通过在gc日志名中添加 %p 和 %t 解决覆盖问题，如下所示。**但在生产实践中我们发现这种方式和公司的日志自动压缩机制不兼容**。公司gc日志压缩脚本如下所示

```shell
#!/bin/sh
DATE=$(date -d "yesterday" +%F)
DATE10=$(date -d "10 days ago"  +%F)
SECOND=$(echo $RANDOM | cut -c1-3)
if [ -d /home/q/www ] ; then
  APP=$(ls -1 /home/q/www | grep -v default)
  if [ "$APP" != "" ] ; then
    for i in $(ls -1 /home/q/www | grep -v default)
    do
      if [ -d /home/q/www/$i ] ; then
        sleep $SECOND
        ### 硬编码 gc.log
        [ -f /home/q/www/$i/logs/gc.log ] && cp /home/q/www/$i/logs/gc.log /home/q/www/$i/logs/gc.log.$DATE && echo > /home/q/www/$i/logs/gc.log && gzip /home/q/www/$i/logs/gc.log.$DATE
        rm -f /home/q/www/$i/logs/gc.log.${DATE10}.gz
      fi
    done
  fi
fi
```

可以看到日志压缩脚本只能识别并压缩 gc.log ，自定义格式的gc日志无法被压缩和删除。除此之外，如果在gc日志文件名使用 %t 占位符，还会命中公司按天压缩日志的脚本，导致gc日志被压缩，**新的gc日志会丢失**。公司按天压缩日志的脚本如下所示

```shell
#!/bin/bash
ZIPDATE=$(date +%F -d "-1 day");
DELDATE=$(date +%F -d "-7 day");
SECOND=$(echo $RANDOM | cut -c1-3)

sleep $SECOND
for i in `find /home/q/www/ -maxdepth 2 \( -type d -o -type l \) -name logs`;
do
        for file in `find -L $i -maxdepth 1 -type f \( -name "*${ZIPDATE}*" -a ! -name "*.gz" \)`;
        do
            ### gzip 会将原文件压缩并重命名
            gzip -S ".suf" ${file} ;mv ${file}.suf ${file}.gz &
        done
        wait     
        find -L $i -maxdepth 1 -type f \( -name "*${DELDATE}*" -a -name "*.gz" -a ! -name "*${DELDATE}-[0-2][0-9]*" \) -exec rm -f {} \;
        #find -L $i -maxdepth 1 -type f -mtime +1 ! -name "*.gz" -name "*201*" -exec gzip -v {} \;
done
```

因此，通过修改gc日志文件名的方式解决覆盖问题，可以采用两种方案。

1. 在gc日志文件名中加入进程号。形如 -Xloggc:$CATALINA_BASE/logs/gc-%p.log。这种方案修改简单，但会导致gc日志不会被自动压缩删除，但是gc文件一般也不会太大，问题不大。也可以通过自定义日志压缩脚本实现自动压缩删除。
2. 在gc日志文件名中加入时间戳。形如 -Xloggc:$CATALINA_BASE/logs/gc-%t.log。这种方案修改简单，gc日志名更易理解，**但和公司日志压缩脚本严重冲突**，必须自定义日志压缩脚本实现自动压缩删除。



## 2.将gc日志打印到标准输出流(catalina.out)

换一种思路，可以将gc日志打印到标准输出流中。

在tomcat服务中，输出到标准输出流的日志会重定向到 catalina.out 中，在tomcat重启后，新的服务会追加写入catalina.out而不是覆盖写入。除此之外，公司对catalina.out也有专门的日志压缩删除脚本(/home/q/tools/bin/catalina_logrotate.sh)。

该方案实施简单，将 -Xloggc 参数改为 -verbose:gc 即可。

优点

* 实施简单。只需要改java启动参数
* 运维成本低。和公司压缩日志脚本兼容，不需要自己处理日志的压缩和删除，也有利于快速扩容

缺点

*    gc日志和其他日志混杂，会提高分析gc日志的难度。例如使用诸如（[gceasy.io](http://gceasy.io/)，GCViewer ....）之类的GC工具分析GC日志文件时需要预处理提取出gc日志。

# 最终解决方案

最终我们采用第2种方案：将gc日志打印到标准输出流(对于tomcat服务来说是 catalina.out)。因为一般情况下，我们是通过人工查看gc日志即可发现和解决问题，极少有需要使用诸如（[gceasy.io](http://gceasy.io/)，GCViewer ....）之类的GC工具分析GC日志文件的场景。因此这种方案可以有效解决我们的需求。

