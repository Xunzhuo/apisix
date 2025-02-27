---
title: kafka-logger
---

<!--
#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
-->

## 目录

- [**简介**](#简介)
- [**属性**](#属性)
- [**工作原理**](#工作原理)
- [**如何启用**](#如何启用)
- [**测试插件**](#测试插件)
- [**禁用插件**](#禁用插件)

## 简介

`kafka-logger` 是一个插件，可用作ngx_lua nginx 模块的 Kafka 客户端驱动程序。

它可以将接口请求日志以 JSON 的形式推送给外部 Kafka 集群。如果在短时间内没有收到日志数据，请放心，它会在我们的批处理处理器中的计时器功能到期后自动发送日志。

有关 Apache APISIX 中 Batch-Processor 的更多信息，请参考。
[Batch-Processor](../batch-processor.md)

## 属性

| 名称             | 类型    | 必选项 | 默认值         | 有效值  | 描述                                             |
| ---------------- | ------- | ------ | -------------- | ------- | ------------------------------------------------ |
| broker_list      | object  | 必须   |                |         | 要推送的 kafka 的 broker 列表。                  |
| kafka_topic      | string  | 必须   |                |         | 要推送的 topic。                                 |
| producer_type    | string  | 可选   | async          | ["async", "sync"]        | 生产者发送消息的模式。          |
| required_acks          | integer | 可选    | 1              | [0, 1, -1] | 生产者在确认一个请求发送完成之前需要收到的反馈信息的数量。这个参数是为了保证发送请求的可靠性。语义同 kafka 生产者的 acks 参数（如果设置 `acks=0`，则 producer 不会等待服务器的反馈。该消息会被立刻添加到 socket buffer 中并认为已经发送完成。如果设置 `acks=1`，leader 节点会将记录写入本地日志，并且在所有 follower 节点反馈之前就先确认成功。如果设置 `acks=-1`，这就意味着 leader 节点会等待所有同步中的副本确认之后再确认这条记录是否发送完成。）。         |
| key              | string  | 可选   |                |         | 用于消息的分区分配。                             |
| timeout          | integer | 可选   | 3              | [1,...] | 发送数据的超时时间。                             |
| name             | string  | 可选   | "kafka logger" |         | batch processor 的唯一标识。                     |
| meta_format      | enum    | 可选   | "default"      | ["default"，"origin"] | `default`：获取请求信息以默认的 JSON 编码方式。`origin`：获取请求信息以 HTTP 原始请求方式。[具体示例](#meta_format-参考示例)|
| batch_max_size   | integer | 可选   | 1000           | [1,...] | 设置每批发送日志的最大条数，当日志条数达到设置的最大值时，会自动推送全部日志到 `Kafka` 服务。|
| inactive_timeout | integer | 可选   | 5              | [1,...] | 刷新缓冲区的最大时间（以秒为单位），当达到最大的刷新时间时，无论缓冲区中的日志数量是否达到设置的最大条数，也会自动将全部日志推送到 `Kafka` 服务。 |
| buffer_duration  | integer | 可选   | 60             | [1,...] | 必须先处理批次中最旧条目的最长期限（以秒为单位）。 |
| max_retry_count  | integer | 可选   | 0              | [0,...] | 从处理管道中移除之前的最大重试次数。             |
| retry_delay      | integer | 可选   | 1              | [0,...] | 如果执行失败，则应延迟执行流程的秒数。           |
| include_req_body | boolean | 可选   | false          | [false, true] | 是否包括请求 body。false： 表示不包含请求的 body ； true： 表示包含请求的 body 。|
| include_req_body_expr | array  | 可选    |           |         | 是否采集请求body, 基于[lua-resty-expr](https://github.com/api7/lua-resty-expr)。 该选项需要开启 `include_req_body`|
| cluster_name     | integer | 可选   | 1              | [0,...] | kafka 集群的名称。当有两个或多个 kafka 集群时，可以指定不同的名称。只适用于 producer_type 是 async 模式。|

### meta_format 参考示例

- **default**:

    ```json
    {
     "upstream": "127.0.0.1:1980",
     "start_time": 1619414294760,
     "client_ip": "127.0.0.1",
     "service_id": "",
     "route_id": "1",
     "request": {
       "querystring": {
         "ab": "cd"
       },
       "size": 90,
       "uri": "/hello?ab=cd",
       "url": "http://localhost:1984/hello?ab=cd",
       "headers": {
         "host": "localhost",
         "content-length": "6",
         "connection": "close"
       },
       "body": "abcdef",
       "method": "GET"
     },
     "response": {
       "headers": {
         "connection": "close",
         "content-type": "text/plain; charset=utf-8",
         "date": "Mon, 26 Apr 2021 05:18:14 GMT",
         "server": "APISIX/2.5",
         "transfer-encoding": "chunked"
       },
       "size": 190,
       "status": 200
     },
     "server": {
       "hostname": "localhost",
       "version": "2.5"
     },
     "latency": 0
    }
    ```

- **origin**:

    ```http
    GET /hello?ab=cd HTTP/1.1
    host: localhost
    content-length: 6
    connection: close

    abcdef
    ```

## 工作原理

消息将首先写入缓冲区。
当缓冲区超过`batch_max_size`时，它将发送到 kafka 服务器，
或每个`buffer_duration`刷新缓冲区。

如果成功，则返回 `true`。
如果出现错误，则返回 `nil`，并带有描述错误的字符串（`buffer overflow`）。

### Broker 列表

插件支持一次推送到多个 Broker，如下配置：

```json
{
    "127.0.0.1":9092,
    "127.0.0.1":9093
}
```

## 如何启用

1. 为特定路由启用 kafka-logger 插件。

```shell
curl http://127.0.0.1:9080/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "plugins": {
       "kafka-logger": {
           "broker_list" :
             {
               "127.0.0.1":9092
             },
           "kafka_topic" : "test2",
           "key" : "key1"
       }
    },
    "upstream": {
       "nodes": {
           "127.0.0.1:1980": 1
       },
       "type": "roundrobin"
    },
    "uri": "/hello"
}'
```

## 测试插件

 成功

```shell
$ curl -i http://127.0.0.1:9080/hello
HTTP/1.1 200 OK
...
hello, world
```

## 插件元数据设置

| 名称             | 类型    | 必选项 | 默认值        | 有效值  | 描述                                             |
| ---------------- | ------- | ------ | ------------- | ------- | ------------------------------------------------ |
| log_format       | object  | 可选   | {"host": "$host", "@timestamp": "$time_iso8601", "client_ip": "$remote_addr"} |         | 以 JSON 格式的键值对来声明日志格式。对于值部分，仅支持字符串。如果是以 `$` 开头，则表明是要获取 __APISIX__ 变量或 [Nginx 内置变量](http://nginx.org/en/docs/varindex.html)。特别的，**该设置是全局生效的**，意味着指定 log_format 后，将对所有绑定 http-logger 的 Route 或 Service 生效。 |

**APISIX 变量**

|       变量名      |           描述          |      使用示例    |
|------------------|-------------------------|----------------|
| route_id         | `route` 的 id           | $route_id      |
| route_name       | `route` 的 name         | $route_name    |
| service_id       | `service` 的 id         | $service_id    |
| service_name     | `service` 的 name       | $service_name  |
| consumer_name    | `consumer` 的 username  | $consumer_name |

### 设置日志格式示例

```shell
curl http://127.0.0.1:9080/apisix/admin/plugin_metadata/kafka-logger -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "log_format": {
        "host": "$host",
        "@timestamp": "$time_iso8601",
        "client_ip": "$remote_addr"
    }
}'
```

在日志收集处，将得到类似下面的日志：

```shell
{"host":"localhost","@timestamp":"2020-09-23T19:05:05-04:00","client_ip":"127.0.0.1","route_id":"1"}
{"host":"localhost","@timestamp":"2020-09-23T19:05:05-04:00","client_ip":"127.0.0.1","route_id":"1"}
```

## 禁用插件

当您要禁用`kafka-logger`插件时，这很简单，您可以在插件配置中删除相应的 json 配置，无需重新启动服务，它将立即生效：

```shell
$ curl http://127.0.0.1:9080/apisix/admin/routes/1  -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "methods": ["GET"],
    "uri": "/hello",
    "plugins": {},
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "127.0.0.1:1980": 1
        }
    }
}'
```
