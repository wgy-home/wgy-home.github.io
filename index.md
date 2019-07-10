---
title: logstash grok正则捕获

date:

categories: 运维

tags:
  - 运维
  - elk
  - logstash

---

# 需求

因为之前使用filebeat收集日志直接传输到es中，使用kibana展示，但是由于全部日志内容在message字段中，不方便kibana处理，所以需要将日志格式化后在写入到es内，logstash可以通过grok插件帮我实现这个需求

# 环境介绍

软件名|版本|备注
---|---|---
filebeat|6.6.2|
logstash|6.6.2|
elasticsearch|6.2.4|
kibana|6.2.4



## logstash 插件安装

```shell
# bin/logstash-plugin install xxx
```

# grok使用

### 1. filebeat配置

```shell
# cat filebeat.yml

filebeat.config:
  prospectors:
    path: ${path.config}/prospectors.d/*.yml
    reload.enabled: true
  modules:
    path: ${path.config}/modules.d/*.yml
    reload.enabled: true

output.logstash:
  enabled: true
  # The Logstash hosts 这里写logstash的主机和端口
  hosts: ["10.10.0.199:33971","10.10.0.208:33971","10.10.1.2:33971"]
  loadbalance: true
  #bulk_max_size: 0
logging.level: info
logging.to_files: true
logging.files:
  path: /xdfapp/logs/filebeat
  name: filebeat.log
  keepfiles: 7
  permissions: 0644

```

```shell
# cat prospectors.d/java.yml
- type: log
  paths:
    - /xdfapp/logs/hotfix/tomcat/*info*
    - /xdfapp/logs/hotfix/tomcat2/*info*
# 多行合并，匹配每行开头如果不为: 年-月-日(2019-01-01),则合并到上一行的末尾。
    multiline.pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
    multiline.negate: true  # true 或 false；默认是false，匹配pattern的行合并到上一行；true，不匹配pattern的行合并到上一行
    multiline.match: after  # after 或 before，合并到上一行的末尾或开头
    multiline.timeout: 30s  # 设定一个超时时间，在时间结束后，即使没有匹配到新pattern来启动新事件，Filebeat也会发送多行事件。默认值是5秒
  fields:
    log_type: java-info
    resolution_type: java-info
```

### 2. logstash 配置

先看一下我的日志，两条日志一条正常的info日志，一条多行的错误日志。

可以通过[grokdebug](http://grokdebug.herokuapp.com/)在线调试
```
2019-07-03 00:59:43 INFO com.noriental.cas.filter.AuthenticationCustomFilter:167 [http-bio-8080-exec-219] [011011484958] client url:/, validating client status faild................
2019-07-03 10:16:28 ERROR com.noriental.utils.exception.ExceptionUtils:45 [http-bio-8080-exec-226] [010548972503] [010548972503][/classroom/course_close][61951240190]
com.noriental.xxxweb.utils.xxxWebException: 60038 : 已经下课，不再需要下课
	at com.noriental.xxxweb.utils.ActionBase.processServiceException(ActionBase.java:315)
	at com.noriental.xxxweb.utils.ActionBase.invokeService(ActionBase.java:202)
	at com.noriental.xxxweb.classroominteraction.controller.action.Action_course_close.doExecute(Action_course_close.java:42)
	at com.noriental.xxxweb.utils.ActionBase.execute(ActionBase.java:131)
	at com.noriental.xxxweb.utils.ActionBase.execute(ActionBase.java:32)
	at com.noriental.xxxweb.utils.BaseActionController.executeActionTemplate(BaseActionController.java:150)
	at com.noriental.xxxweb.utils.BaseActionController.executeActionTemplate(BaseActionController.java:143)
.......

```

先上我的logstash配置文件。

```shell
bash-4.2$ cat config/logstash.conf 
input {
  beats {
    port => 5044
  }
}

filter {
    if [fields][resolution_type] == "java-info-hotfix" {
        grok {
            patterns_dir => ["/usr/share/logstash/pipeline/patterns"]
            match => {"message" => "%{TOMCATINFO}"}
        }
    }
}

output {
  elasticsearch {
    hosts => ["http://efk-elasticsearch.hotfix-test.svc:9200"]
    index => "logstash-%{[fields][log_type]}-%{+YYYY.MM.dd}"
  }
}


```

> [logstash官方预定义的表达式](https://github.com/logstash-plugins/logstash-patterns-core/tree/master/patterns)，下面配置文件中是filter调用的正则表达试，

```shell
bash-4.2$ cat pipeline/patterns 
LOG_DATE [0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}:[0-9]{2}:[0-9]{2}
REQID (%{USERNAME}%{NUMBER}|%{NUMBER})
TOMCATCLASS  (%{JAVACLASS}.%{INT})
MESSAGE  .*
TOMCATINFO %{LOG_DATE:@timestamp}\s+%{LOGLEVEL:loglevel}\s+%{TOMCATCLASS:class}\s+\[%{REQID:process}\]\s+\[%{REQID:requestid}\]\s+%{MESSAGE:message}
```
