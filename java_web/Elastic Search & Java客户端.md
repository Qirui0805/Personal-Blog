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
## 基本概念
索引（indice/index),类型（type），文档（document），字段（fileds），可以分别对应关系型数据库的库，表，行，列
