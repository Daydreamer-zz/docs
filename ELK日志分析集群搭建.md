# ELK日志分析集群搭建

## 1.简介

对日志的收集，存储，查询，展示的开源项目

logstash(日志收集)，elasticsearch(存储+搜索)，kibana(展示)

ELKstack就是以上集合

## 2.安装配置

- Elasticsearh部分

  安装：JDK环境

  ```
  yum install -y java-1.8.0-openjdk java-1.8.0-openjdk-devel
  ```

  安装Elasticsearch的rpm包

  ```
  rpm -ivh elasticsearch-5.4.0.rpm
  ```

  ```
  systemctl damon-reload
  ```

  配置

  ```
  vim /etc/elasticsearch/elasticsearch.yml
  ```

  ```
  cluster.name: elk-cluster
  node.name: node1
  path.data: /data/elkdata
  path.logs: /data/logs
  bootstrap.memory_lock: true    #锁住内存，不使用交换分区
  network.host: 192.168.2.4
  descovery.zen.ping.unicast.host: ["192.168.2.4","192.168.2.10"]
  ```

  启动

  ```
  systemctl start elasticsearch
  ```

- elasticsearch安装head插件

  注意5.0以上的版本无法使用.../bin/plugin install head  方法安装，只能手动安装

  ```
  git clone https://github.com/mobz/elasticsearch-head.git
  ```

  ```
  cd elasticsearch-head/
  ```

  ```
  yum install -y npm
  ```

  ```
  npm install grunt -save
  ```

  ```
  rpm install
  ```

  安装过程很长，由于某些不可描述的原因可能会失败

  ```
  rpm run start &    #后台启动，端口9100
  ```

  修改elasticsearch的配置文件

  ```
  vim /etc/elasticsearch/elasticsearch.yml
  ```

  添加

  ```
  http.cors.enabled: true
  http.cors.allow-origin: "*"
  ```

  重启elasticsearch

  ```
  systemctl restart elasticsearch
  ```

  浏览器访问

  http://192.168.2.4:9100连接http://192.168.2.4:9200

  直接用curl命令获取集群健康状态 

  ```
  curl -XGET 'http://192.168.2.4:9200/_cluster/health?pretty=true'
  ```

  返回的"status"："green" 为正常

  ​                              "yellow" 为不正常

  ​                               "red"   为主分片丢失

- Logstash部分(也需要jdk)

  可以安装在任意需要收集日志的节点

  安装rpm包

  ```
  rpm -ivh logstash-5.4.0.rpm
  ```

  测试标准输入输出

  ```
  /usr/share/logstash/bin/logstash -e 'input{ stdin{} } output{ stdout { codec => rubydebug } }'
  ```

  键入什么输出什么说明成功

  测试输出到文件

  ```
  /usr/share/logstash/bin/logstash -e 'input{ stdin{} } output{ file { path => "/tmp/log-%{+YYYY.MM.dd}.log" gzip => true }}'
  ```

  测试输出到elasticsearch

  ```
  /usr/share/logstash/bin/logstash -e 'input{ stdin{} } output{ elasticsearch{ hosts => ["192.168.2.4:9200"] index => "logstash-test-%{+YYYY.MM.dd}" }}'
  ```

  elasticsearch的数据具体目录： /data/elkdata/nodes/0/indices

  输出到elasticsarch，访问http://192.168.2.4:9100连接上9200即可查看测试输出

  logstash可以从许多服务中输入(input)日志(文件)，并输出(output)到各种服务中

  通过写配置文件，可以达到收集日志文件的目的

  ```
  vim /etc/logstash/conf.d/demo.conf
  ```

  ```
  input{
      file{
          path => "/var/log/messages"
          type => "system-log"
          start_position => "beginning"
      }
  }
  filter{
      
  }
  output{
      elasticsearch{
          hosts => ["192.168.2.4:9200"]
          index => "system-log-%{+YYYY.MM}"
      }
  }
  ```

  启动

  ```
  /usr/share/logstash/bin/logstash -f /etc/logstash/conf.d/demo.conf
  ```

- kibana部分(也需要jdk)

  rpm包安装

  ```
  rpm -ivh kibana-5.4.0-x86_64.rpm
  ```

  配置

  ```
  vim /etc/kibana/kibana.yml
  ```

  ```
  server.port: 5601
  server.host: "192.168.2.5"
  elasticsearch.url: "http://192.168.2.4:9200"
  ```

  启动

  ```
  systemctl start kibana
  ```

  浏览器访问 http://192.168.2.5:5601

## 3.logstash的高级应用

### if判断输出

logstash的if判断写入不同的标签(从而实现一个配置文件收集不同的位置，并写入不同标签)

通过type定义多个，然后在用if判断输出到不同的index

如下实例

```ruby
input{
    file{
        path => "/var/log/message"
        type => "system-log"
        start_position => "beginning"
    }
    file{
        path => "/data/logs/elk-cluster.log"
        type => "es-logs"
    }
}
output{
    if [type] == "system-log" {
        elasticsearch {
            hosts => "[192.168.2.4:9200]"
            index => "system-log-%{+YYYY.MM}"
        }
    }
    if [type] == "es-logs" {
        elasticsearch {
            hosts => "[192.168.2.4:9200]"
            index => "es-logs-%{+YYYY.MM}"
        }
    }
}
```

注意：如果日志文件没有更新的话，logstash可能是无法收集，原因是/var/lib/logstash/plugins/inputs/file/.sincedb.xxxx 文件(是隐藏的)记录了日志文件的大和inode，也记录了logstash收集日志文件的位置

所以在head中你删除了此项目，凡是`.sincedb.xxxx`没删除，导致无法继续收集日志，也无法重新从文件开头收集 

### 如何收集java的日志

logstash默认将日志文件的一行当做一个事件，并分开发送，但是，在遇到如java日志多行为一个日志事件时：就需要用到codec插件

logstash的路径

一个事件--->input--->codec--->filter--->codec--->output

如果将识别多行日志的插件放到filter中，则所有的input日志都会生效，这显然不合理；所以需要将插件放到需要识别多行的`input codec`中  `codec => multiline`

编写方案：由于日志文件有时间戳，所以用"["区分，即匹配到一个"["就为一个事件

```
vim /etc/logstash/conf.d/codec.conf
```

```
input{
    stdin{
        codec => multiline {
            pattern => "^\["   #正则匹配，开头为[的
            negate => true     #规定匹配到为真
            what => "previous"
        }
    }
}
output{
    stdout{
        codec => rubydebug
    }
}
```

经过测试，只要遇到中括号就把"["以上的行合并为一个事件，满足要求

所有把以上整合进file模块

```
file{
    path => "/var/log/elasticsearch/mues.log"
    type => "es-log"
    start_position => "beginning"
    codec => multiline {
        pattern => "^\["   #正则匹配，开头为[的
        negate => true     #规定匹配到为真
        what => "previous
    }
}
```

### 收集nginx json格式日志

修改nginx配置文件，使日志输出为json格式

```
vim /etc/nginx/nginx.conf
```

```
http{
    .....
    log_format access_log_json '{"user_ip":"$http_x_forward_for","lan_ip":"$remote_addr","log_time":"time_iso8601","user_req":"$request","http_code":"$status","body_bytes_sent":"$body_bytes_sent","req_time":"$request_time","user_ua":"$http_user_agent"}';
    access_log /var/log/nginx/access_json_log access_log_json;
}
```

在logstash中配置测试

```
vim /etc/logstash/conf.d/nginx_log.conf
```

```
input{
    file{
        path => "/var/log/nginx/access_json.log"
        codec => "json"
    }
}
output{
    elasticsearch {
        hosts => "[192.168.2.4:9200]"
        index => "nginx_access_json-%{+YYYY.MM}"
    }
}
```

### lohstashi input插件syslog

修改rsysog配置文件，rsyslog可以将日志输出到文件，数据库或者其他应用

```
vim /etc/rsyslog.conf
```

```
$ModLoad imudp
$UDPServerRun 514

$ModLoad imtcp
$InputTCPServerRun 514

........
*.* @@192.168.2.4:514
```

修改logstash配置文件

```
vim /etc/logstash/conf.d/syslog.conf
```

```
input{
    syslog{
        type => "system-syslog"
        port => 514
    }
}
output{
    stdout{
        codec => rubydebug
    }
}
```

重启rsyslog服务

```
systemctl restart rsyslog
```

### logstash写入redis再从redis读取

安装redis

```
mkdir -p /application
```

```
tar xf redis-5.0.0.tar.gz
```

```
cd redis-5.0.0
```

```
make
```

```
cd src/
```

```
make install PREFIX=/appliction/redis
```

复制默认配置文件到安装目录

修改配置文件

```
bind 192.168.2.4
......
daemonize yes
......
requirepass 199747
```

启动

```
/application/resis/bin/redis-server /application/redis/redis.conf
```

连接测试

```
/application/resis/bin/redis-cli -h 192.168.2.4
```

```
> AUTH 199747
```

配置logstash写入redis

```
vim /etc/logstash/conf.d/nginx.conf
```

```
input{
    file{
        path => "/var/log/nginx/access_json.log"
        codec => "json"
    }
}
output{
    redis{
        host => "192.168.2.4"
        port => 6379
        db => "1"
        password => "199747"
        data_type => "list"
        key => "nginx_access_json_log"
    }
}
```

logstash从redis读取后再写入elasticsearch

```
input{
    redis{
        data_type => "list"
        key => "nginx_access_json_log"
        db => "1"
        host => "192.168.2.4"
        port => 6379
        password => "199747"
    }
}
output{
    elasticsearch{
        hosts => "[192.168.2.4:9200]"
        index => "redis-nginx-%{+YYYY.MM.dd}"
    }
}
```

这样就实现了消息队列扩展：数据---->logstash---->MQ/Redis---->logstash---->Elasticsearch