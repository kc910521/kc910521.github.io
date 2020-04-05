---
layout: default
title: "es 分享"
tags: document
---
#  期末串讲-ES v1     

## 蹭蹭：

## 几个重要网站

仅供参考的中文文档：
https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html

正统英文文档：
https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html

java API：
https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/index.html

HTTP API：
https://www.elastic.co/guide/en/elasticsearch/reference/current/docs.html

知乎ES专栏：
https://zhuanlan.zhihu.com/Elasticsearch?utm_source=com.alibaba.android.rimet&utm_medium=social




## es版本

 现在6.x，以5.x为例，5.1以前文档仅存在参考意义，中文文档2.x

Es插件和常用命令
pinyin分词，IK，head

所有索引
http://192.168.2.1:9200/_cat/indices

内容
http://192.168.2.1:9200/5af3e411cff47e1151f2bf8c_module_data_5afe8e5e4cedfd28c5429a76/_search

字段mapping
http://192.168.2.1:9200/5b480e754cedfd06b005c433_module_data_5b480e8b4cedfd06b005c4ea/_mapping


请求的基本形式：
http://{ip}:9200/{index}/{type}/{docId}/{operation}

============== es的大门 ===================================

# 大门：

## Es常用字段类型  

Core datatypes
string 过时
text and keyword
Numeric datatypes
long, integer, short, byte, double, float, half_float, scaled_float
Date datatype
date
JSON中没有日期类型，所以在ELasticsearch中，日期类型可以是以下几种：
1. 日期格式的字符串：e.g. “2015-01-01” or “2015/01/01 12:10:30”.
2. long类型的毫秒数( milliseconds-since-the-epoch)
3. integer的秒数(seconds-since-the-epoch)
Boolean datatype
boolean
Binary datatype
binary base64 不存储不搜索
Range datatypes
integer_range, float_range, long_range, double_range, date_range
Complex datatypes
edit
Array datatype
Array support does not require a dedicated type
Object datatype
object for single JSON objects
Nested datatype
nested for arrays of JSON objects
https://blog.csdn.net/napoay/article/details/73100110?utm_source=copy

## 分词
http://192.168.2.1:9200/5af3e411cff47e1151f2bf8c_module_data_5afe8e5e4cedfd28c5429a76/_analyze?text=抢的&analyzer=pinyin


term、match query
https://www.elastic.co/guide/en/elasticsearch/reference/6.4/query-dsl-term-query.html

term不分词≈match_phase

使用db思路想

filter：

## 多字段


                "key2555": {
                                "properties": {
                                    "selected": {
                                        "properties": {
                                            "label": {
                                                "type": "text",
                                                "fields": {
                                                    "pinyin": {
                                                        "type": "text",
                                                        "analyzer": "pinyin"
                                                    },
                                                    "pinyin_raw": {
                                                        "type": "text",
                                                        "analyzer": "pinyin_raw",
                                                        "fielddata": true
                                                    },
                                                    "raw": {
                                                        "type": "keyword"
                                                    },
                                                    "raw_lower": {
                                                        "type": "text",
                                                        "analyzer": "raw_lower"
                                                    },
                                                    "text": {
                                                        "type": "text",
                                                        "analyzer": "standard"
                                                    }
                                                }
                                            },
                                            "value": {
                                                "type": "keyword"
                                            }
                                        }
                                    }
                                }
                            }


## reindex和alias
建立A_INST index和别名A，系统使用别名操作；
建立A_INST_COPY,更新类型，填入数据from A；
删除A_INST
设置A_INST_COPY别名A

同义词
自己看
================== 你可以独立使用ES了 ============

# 二楼雅座：

## Es 分析器（analyzer）、分词器（tokenizer）和过滤器（filter）,以及ngram
Analyzer包含两个核心组件，Tokenizer以及TokenFilter。两者的区别在于，前者在字符级别处理流，而后者则在词语级别处理流。
Tokenizer是Analyzer的第一步，其构造函数接收一个Reader作为参数，而TokenFilter则是一个类似的拦截器，其参数可以是TokenStream、Tokenizer

Tokenizer:截取
filter：过滤结果

ngram：
再_settings tokenizer中 …
"min_gram": "1",
"max_gram": "16"

Eg：
* 长度 1（unigram）： [ q, u, i, c, k ]
* 长度 2（bigram）： [ qu, ui, ic, ck ]
* 长度 3（trigram）： [ qui, uic, ick ]
* 长度 4（four-gram）： [ quic, uick ]
* 长度 5（five-gram）： [ quick ]


## 事务、锁机制及多版本并发控制
不支持事务；

1.全局锁(利用文档)
PUT /lockindex/locktype/global/_create
同时如果有另一个线程要进行相关更新操作，那么同样执行上述代码是会报错。在上述线程执行完DELETE对应doc之后，该线程就可以重新获取到doc的锁从而执行自己的一些列操作。
delete /fs/lock/global 解锁

2.document锁，粒度更细的锁  需要通过脚本来实现：
POST /fs/lock/1/_update
{
  "upsert": { "process_id": 123 },
  "script": "if ( ctx._source.process_id != process_id ) { assert false }; ctx.op = 'noop';"
  "params": {
    "process_id": 123
  }
}
3.共享锁排它锁

4.乐观多版本并发控制：

_version默认自增，小于该数报409
可通过PUT /website/blog/2?version=5&version_type=external
外部控制版本号需要大于当前版本号

对于商品stock_count，我们需要重新检索最新文档然后申请新的更改操作。


在es后台，有很多类似于replica同步的请求，这些请求都是异步多线程的，对于多个修改请求是乱序的，因此会使用_version乐观锁来控制这种并发的请求处理。当后续的修改请求先到达，对应修改成功之后_version会加1，然后检测到之前的修改到达会直接丢弃掉该请求；而当后续的修改请求按照正常顺序到达则会正常修改然后_version在前一次修改后的基础上加1（此时_version可能就是3，会基于之前修改后的状态）。 


## Es节点类型
 Master主要管理集群信息、primary分片和replica分片信息、维护index信息。
DataNode用来存储数据，维护倒排索引，提供数据检索等。
Client:是作为任务分发用的，它里面也会存元数据，但是它不会对元数据做任何修改。client node存在的好处是可以分担下data node的一部分压力；为什么client node能分担data node的一部分压力？因为es的查询是两层汇聚的结果，第一层是在data node上做查询结果汇聚，然后把结果发给client node，client node接收到data node发来的结果后再做第二次的汇聚，然后把最终的查询结果返回给用户


## 主从分片的动态调整


## Es，内存、缓存和硬盘

translog日志提供了一个所有还未被flush到磁盘的操作的持久化记录。当ES启动的时候，它会使用最新的commit point从磁盘恢复所有已有的segments，然后将重现所有在translog里面的操作来添加更新，这些更新发生在最新的一次commit的记录之后还未被fsync
索引之segment memory：
 Shard（分片）        一个Shard就是一个Lucene实例，是一个完整的搜索引擎。一个索引可以只包含一个Shard，只是一般情况下会用多个分片，可以拆分索引到不同的节点上，分担索引压力。
segment       elasticsearch中的每个分片包含多个segment，每一个segment都是一个倒排索引；在查询的时，会把所有的segment查询结果汇总归并后最为最终的分片查询结果返回；       在创建索引的时候，elasticsearch会把文档信息写到内存bugffer中（为了安全，也一起写到translog），定时（可配置）把数据写到segment缓存小文件中，然后刷新查询，使刚写入的segment可查。  虽然写入的segment可查询，但是还没有持久化到磁盘上。因此，还是会存在丢失的可能性的。        所以，elasticsearch会执行flush操作，把segment持久化到磁盘上并清除translog的数据（因为这个时候，数据已经写到磁盘上，不在需要了）。  当索引数据不断增长时，对应的segment也会不断的增多，查询性能可能就会下降。因此，Elasticsearch会触发segment合并的线程，把很多小的segment合并成更大的segment，然后删除小的segment。       segment是不可变的，当我们更新一个文档时，会把老的数据打上已删除的标记，然后写一条新的文档。在执行flush操作的时候，才会把已删除的记录物理删除掉。
https://blog.csdn.net/liyantianmin/article/details/72973281?utm_source=copy

       一个segment是一个完备的lucene倒排索引，而倒排索引是通过词典(Term Dictionary)到文档列表(Postings List)的映射关系，快速做查询的。所以每个segment都有会一些索引数据驻留在heap里。
       因此segment越多，瓜分掉的heap也越多，并且这部分heap是无法被GC掉的！
Segment合并
通过每隔一秒的自动刷新机制会创建一个新的segment，用不了多久就会有很多的segment。segment会消耗系统的文件句柄，内存，CPU时钟。最重要的是，每一次请求都会依次检查所有的segment。segment越多，检索就会越慢。
ES通过在后台merge这些segment的方式解决这个问题。小的segment merge到大的，大的merge到更大的。。。
这个过程也是那些被”删除”的文档真正被清除出文件系统的过程，因为被标记为删除的文档不会被拷贝到大的segment中。




￼






## 课后习题（家长签字）：

1.aggregation，buckets，metrics

## 孩子们的问题

1.



