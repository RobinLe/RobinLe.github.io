---
layout: post
title: Docker日志解决方案（rsyslog+logstash）
category: docker
tags: logs
---

### 测试环境

|   Host    |     OS      |      IP      |     Services     |
| :-------: | :---------: | :----------: | :--------------: |
| syslog服务器 | Ubuntu16.04 | 192.168.3.48 | rsyslog，logstash |
| docker服务器 | Ubuntu16.04 | 192.168.3.45 |      nginx       |

docker服务器启动nginx容器，nginx容器日志通过syslog传到syslog服务器，syslog服务器把日志输出到logstash中。





---

### Syslog日志服务器配置

修改**rsyslog**配置文件`/etc/rsyslog.conf`，取消**TCP**或者**UDP**日志接收模块注释，下图开启**TCP514端口**接收日志

```
root@192.168.3.48:~# vi /etc/rsyslog.conf
```

![](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/18.png)

创建日志转发模板文件`/etc/rsyslog.d/01-template-json.conf`，使用**json**格式进行日志转发

```
root@192.168.3.48:~# touch /etc/rsyslog.d/01-template-json.conf
```
输入以下文件内容吗，定义**syslog**的**json**模板及各个字段，[更多字段内容](http://www.rsyslog.com/doc/v8-stable/configuration/properties.html)

```
template(name="json-template"
  type="list") {

    constant(value="{")
      constant(value="\"@timestamp\":\"")     property(name="timereported" dateFormat="rfc3339")
      constant(value="\",\"@version\":\"1")
      constant(value="\",\"message\":\"")     property(name="msg" format="json")
      constant(value="\",\"sysloghost\":\"")  property(name="hostname")
      constant(value="\",\"severity\":\"")    property(name="syslogseverity-text")
      constant(value="\",\"facility\":\"")    property(name="syslogfacility-text")
      constant(value="\",\"programname\":\"") property(name="programname")
      constant(value="\",\"procid\":\"")      property(name="procid")
    constant(value="\"}\n")
}
```

创建日志转发配置文件`/etc/rsyslog.d/60-output.conf`

```
root@192.168.3.48:~# touch /etc/rsyslog.d/60-output.conf
```
输入以下内容，定义所有日志使用`json-template`模板转发到`192.168.3.48:10514`，**2个@@表示TCP端口，1个@表示UDP端口**
```
# This line sends all lines to defined IP address at port 10514,
# using the "json-template" format template

*.*                         @@192.168.3.48:10514;json-template
```

重启**rsyslog**服务

```
root@192.168.3.48:~# /etc/init.d/rsyslog restart
```

![](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/22.png)





---

### Docker服务器配置syslog驱动

为docker daemon配置syslog驱动，并设置远程syslog日志服务器地址和端口接收日志

```
root@192.168.3.45:~# dockerd start --log-driver=syslog \
--log-opt syslog-address=tcp://192.168.3.48:514 &
```

> 或者启动容器时配置syslog中心服务器记录日志（不建议使用该方式）
> ```
> root@192.168.3.45:~# docker run --log-driver=syslog \
> --log-opt syslog-address=tcp://192.168.3.48:514
> ```


启动nginx容器，访问nginx容器
```
root@192.168.3.45:~# docker run -it --rm -p 80:80 nginx
root@192.168.3.45:~# curl 192.168.3.45
```

![](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/19.png)

查看容器，输出了nginx访问日志

![](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/20.png)

查看**syslog服务器**日志，确定接收到**nginx容器**日志

```
root@192.168.3.48:~# cat /var/log/syslog
```

![](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/21.png)





---

### logstash配置（容器方式）

logstash容器运行在syslog服务器上，下载logstash镜像，创建logstash配置文件

```
root@192.168.3.48:~# docker pull logstash
root@192.168.3.48:~# cd ~ 
root@192.168.3.48:~# mkdir logstash_pipeline
root@192.168.3.48:~# vi logstash_pipeline/logstash.conf
```

定义`logstash`输入输出与过滤条件，输入从**tcp10514**端口接受日志数据（也可以直接读取syslog采集，但需要对syslog文件有读权限），直接输出到标准输出，[更多配置参考](https://www.elastic.co/guide/en/logstash/current/config-examples.html)

```
input {
  tcp {
    port => 10514
    codec => "json"
    type => "rsyslog"
  }
#    file { 
#       path => "/logs/syslog" 
#       start_position => "beginning"
#    }
# stdin { }
}

filter { }

output {
    stdout { }
}
```

启动**logstash**容器，接收**syslog**日志

```
root@192.168.3.48:~# docker run -it --rm -p 192.168.3.48:10514:10514 -v ~/logstash_pipeline/:/logstash_pipeline logstash -f /logstash_pipeline/logstash.conf
```

查看**logstash**容器输出，成功接收到**syslog**日志。数据流向`nginx容器-->syslog服务器-->logstash容器`

![](https://raw.githubusercontent.com/RobinLe/RobinLe.github.io/master/_posts/images/23.png)

