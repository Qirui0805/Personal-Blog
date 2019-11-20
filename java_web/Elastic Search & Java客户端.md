# 入门
## 简单的认识
ElasticSearch 是基于Lucene的一款开源搜索引擎，向用户提供了RESTful API与之交互。ElasticSearch是面向文档的，关系型数据库相当于把对象的所有字段如姓名，年龄，地址都拆开，但面对复杂结构的对象就很麻烦。而ElasticSearch可以存储整个对象，对象的每一个字段都可以搜索
## 与ElasticSearch的交互
### Java客户端 High Level Rest Client
High Level Rest Client works on Low Level Rest Client. 最初还有一个客户端叫Transport Client,但官方要在8.0的时候将其完全抛弃，因此已经不使用了。

**注意**
- 客户端的版本尽量和Elastic版本一样，也可以低于但不能高于，如客户端7.0可以访问7.x的Elastic，但访问6.X的客户端时有可能有些API就不支持
#### Maven Repository
```xml
<dependency>
    <groupId>org.elasticsearch.client</groupId>
    <artifactId>elasticsearch-rest-high-level-client</artifactId>
    <version>7.2.1</version>
</dependency>
```
版本根据所用的Elastic自行选择
### 基于Http协议以JSON为交互数据格式的RESTful API
端口号为9200，可以使用WEB客户端，如浏览器，postman，或者用curl命令，格式为`curl -X<VERB> '<PROTOCOL>://<HOST>:<PORT>/<PATH>?<QUERY_STRING>' -d '<BODY>'`
- X<VERB>: GET,POST,PUT,DELETE
- QUERY_STRING: 查询的选项，如pretty要求返回美观的JSON字段（即分行和缩进）
- BODY: 请求主体
## 基本概念
索引（indice/index),类型（type），文档（document），字段（fileds），可以分别对应关系型数据库的库，表，行，列
## 搜索
### 检索文档
```html
GET /index/type/document
```
**实例**：（通过postman）
```html
http://kibana.addx.live:9200/log-test-2019.10.30/_doc/aY84Gm4BvhD42qvR5sdX
```
返回JSON格式的实体
```html
{
    "_index": "log-test-2019.10.30",
    "_type": "_doc",
    "_id": "aY84Gm4BvhD42qvR5sdX",
    "_version": 1,
    "_seq_no": 20792,
    "_primary_term": 1,
    "found": true,
    "_source": {
        "@version": "1",
        "host": "ip-172-31-13-245.cn-north-1.compute.internal",
        "level": "INFO",
        "level_value": 20000,
        "port": 37990,
        "@timestamp": "2019-10-30T01:12:46.789Z",
        "type": "test",
        "thread_name": "http-nio-7777-exec-9",
        "message": "Updating hardware info for device with serial number: 0ec78cea3142aa9d9d88c2d672b4a801. Current battery level is 92",
        "logger_name": "com.addx.iotcamera.controller.device.response.MqttResponseController"
    }
}
```
- _index, _type, _id: 索引，类型和文档
- _source: 文档的内容
**无法直接检索类型**
```html
http://kibana.addx.live:9200/log-test-2019.10.30/_doc
    
{
    "error": "Incorrect HTTP method for uri [/log-test-2019.10.30/_doc] and method [GET], allowed: [POST]",
    "status": 405
}
```
### 搜索
在索引或类型后加_search
```html
GET http://kibana.addx.live:9200/log-test-2019.10.30/_search   OR
GET http://kibana.addx.live:9200/log-test-2019.10.30/_doc/_search
```
```html
{
    "took": 0,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 10000,
            "relation": "gte"
        },
        "max_score": 1.0,
        "hits": [
            {
                "_index": "log-test-2019.10.30",
                "_type": "_doc",
                "_id": "SrKgG24BvhD42qvRwRjs",
                "_score": 1.0,
                "_source": {
                    "@version": "1",
                    "host": "ip-172-31-13-245.cn-north-1.compute.internal",
                    "level": "INFO",
                    "level_value": 20000,
                    "port": 42434,
                    "@timestamp": "2019-10-30T07:45:50.471Z",
                    "type": "test",
                    "thread_name": "nioEventLoopGroup-4-1",
                    "message": "UDP messsage content:557d90bcebbca4e6",
                    "logger_name": "com.addx.iotcamera.handle.UdpServerHandler"
                }
            },
            {
                "_index": "log-test-2019.10.30",
                "_type": "_doc",
                "_id": "S7KgG24BvhD42qvRwRjw",
                "_score": 1.0,
                "_source": {
                    "@version": "1",
                    "host": "ip-172-31-13-245.cn-north-1.compute.internal",
                    "level": "INFO",
                    "level_value": 20000,
                    "port": 42434,
                    "@timestamp": "2019-10-30T07:45:50.475Z",
                    "type": "test",
                    (后面还有很多）
```
- took: 时间，Milliseconds
- shards: 分页
- 第一个hits: 内容总体信息，第二个hits: 内容
- source：和检索一样
- score: 相关性评分
## 搜索
### 空搜索
就是上面说的简单搜索
### 多索引多类型
用星号或逗号
如：`/g*,u*/_search`, `/_all/user,tweet/_search` 
_all表示所有索引
### 简易搜索（查询字符串（query string））
```html
GET /_all/tweet/_search?q=tweet:elasticsearch+tweet:mary
```
基本格式是q=fieldname:value+/-fieldname:value, +表示一定满足，-表示一定不满足，数字和日期前还可以加上><
如果不加fieldname，elastic会将所有fieldname的值都合起来组成一个新的field叫_all,然后去搜索_all这个字段，所以任一字段有value就会返回  
**优点**
简单，适合命令行模式，在开发时进行一次性的查询  
**缺点**
编码后很难阅读，格式很容易错  
