---
layout: post
title: "logstash"
tagline: "logstash"
description: "logstash"
tags: [logstash, 分布式]
---
{% include JB/setup %}

##logstash使用Redis并自定义Grok匹配

然后修改agent.conf：

input {
  file {
    type => "nginx"
    path => ["/var/log/nginx/access.log" ]
  }
}
output {
  redis {
    host => "MyHome-1.domain.com"
    data_type => "channel"
    key => "nginx"
    type => "nginx"
  }
}
启动方式还是一样。

接着修改server.conf:

    input {
      redis {
        host => "MyHome-1.domain.com"
        data_type => "channel"
        type => "nginx"
        key => "nginx"
      }
    }
    filter {
      grok {
        type => "nginx"
        pattern => "%{NGINXACCESS}"
        patterns_dir => ["/usr/local/logstash/etc/patterns"]
      }
    }
    output {
      elasticsearch { }
    }
然后创建Grok的patterns目录，主要就是github上clone下来的那个咯~在目录下新建一个叫nginx的文件，内容如下：


    NGINXURI %{URIPATH}(?:%{URIPARAM})*
    NGINXACCESS \[%{HTTPDATE}\] %{NUMBER:code} %{IP:client} %{HOSTNAME} %{WORD:method} %{NGINXURI:req} %{URIPROTO}/%{NUMBER:version} %{IP:upstream}(:%{POSINT:port})? %{NUMBER:upstime} %{NUMBER:reqtime} %{NUMBER:size} "(%{URIPROTO}://%{HOST:referer}%{NGINXURI:referer}|-)" %{QS:useragent} "(%{IP:x_forwarder_for}|-)"

Grok正则的编写，可以参考wiki进行测试。

也可以不写配置文件，直接用–grok-patterns-path参数启动即可。
