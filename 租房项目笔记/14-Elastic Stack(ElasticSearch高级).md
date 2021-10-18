# 14-Elastic Stack(ElasticSearch高级)

# 1.全文检索

## 1.1.倒排索引

倒排索引源于实际应用中需要根据属性的值来查找记录.这种索引表中的每一项都包括一个属性值和具有该属性值的各记录的地址.由于不是由记录来确定属性值.而是由属性值来确定记录的位置,因而称为倒排索引(inverted index).带有倒排索引的文件我们称为倒排索引文件,简称倒排文件(inverted file).

正排索引:

![image-20200312190707167](14-Elastic Stack(ElasticSearch高级).assets/image-20200312190707167.png)

转化成倒排索引:

![image-20200312190743406](14-Elastic Stack(ElasticSearch高级).assets/image-20200312190743406.png)

说明:

* "单词ID"一栏记录了每个单词的单词编号;
* 第二栏是对应的单词;
* 第三栏即每个单词对应的倒排列表;
* 比如单词"谷歌",其单词编号为1,倒排列表为(1,2,3,4,5),说明文档集中每个文档都包含了这个单词.

而事实上,索引系统还可以记录除此之外的更多信息,在单词对应的倒排列表中不仅记录了文档编号,还记载了单词频率信息(TF),即这个单词在某个文档中的出现次数,之所以要记录这个信息,是因为词频信息在搜索结果排序时,计算查询和文档相似度是很重要的一个计算因子,所以将其记录在倒排列表中,以方便后续排序时进行分值计算.

![image-20200312193153744](14-Elastic Stack(ElasticSearch高级).assets/image-20200312193153744.png)

倒排索引还可以记载更多的信息,除了记录文档编号和单词频率信息外,额外记载了两类信息,即每个单词对应的"文档频率信息",以及在倒排列表中记录单词某个文档出现的位置信息.

![image-20200312193634724](14-Elastic Stack(ElasticSearch高级).assets/image-20200312193634724.png)

## 1.2.全文搜索

全文搜索两个最重要的方面是:

* **相关性(Relevance)**它是评价查询与其结果见的相关程度,并根据这种相关程度对结果排名的一种能力,这种计算方式可以是TF/DF方法,地理位置邻近,模糊相似,或其他的某些算法.
* **分析(Analysis)**它是将文本块转换为有区别的,规范化的token的一个过程,目的是为了创建倒排索引以及查询倒排索引.

### 1.2.1.构造数据

~~~shell
PUT http://172.16.124.131:9200/fechin01
#请求体
{
	"settings":{
		"index":{
			"number_of_shards":"1",
			"number_of_replicas":"0"
		}
	},
	"mappings":{
		"person":{
			"properties":{
				"name":{
					"type":"text"
				},
				"age":{
					"type":"integer"
				},
				"mail":{
					"type":"keyword"
				},
				"hobby":{
					"type":"text",
					"analyzer":"ik_max_word"
				}
			}
		}
	}
}
~~~

之后插入数据:

~~~shell
POST http://172.16.124.131:9200/fechin01/_bulk
#请求体
{"index":{"_index":"fechin","_type":"person"}}
{"name":"张三","age":20,"mail":"111@qq.com","hobby":"羽毛球,乒乓球,足球"}
{"index":{"_index":"fechin","_type":"person"}}
{"name":"李四","age":21,"mail":"222@qq.com","hobby":"羽毛球,乒乓球,足球,篮球"}
{"index":{"_index":"fechin","_type":"person"}}
{"name":"王五","age":22,"mail":"333@qq.com","hobby":"羽毛球,篮球,游泳,听音乐"}
{"index":{"_index":"fechin","_type":"person"}}
{"name":"赵六","age":23,"mail":"444@qq.com","hobby":"跑步,游泳,篮球"}
{"index":{"_index":"fechin","_type":"person"}}
{"name":"孙七","age":24,"mail":"555@qq.com","hobby":"听音乐,看电影,羽毛球"}
~~~

### 1.2.2.单词搜索

~~~shell
POST http://172.16.124.131:9200/fechin01/_search
#请求体
{
	"query":{
		"match":{
			"hobby":"音乐"
		}
	},
	"highlight":{
		"fields":{
			"hobby":{}
		}
	}
}
~~~

过程说明:

1. **检查字段类型**:爱好hobby字段是一个text类型(指定了IK分词器),这意味着查询字符串本身也应该被分词;
2. **分析查询字符串**:将查询的字符串"音乐"传入IK分词器中,输出的结果是单个项音乐,因为只有一个单词项,所以match查询执行的是单个底层term查询;
3. **查找匹配文档**:用term查询在倒排索引中查找"音乐",然后获取一组包含该项的文档,本例的结果是文档:3,5;
4. **为每个文档评分**:用term查询计算每个文档相关度评分_score,这是种将词频(term frequency,即词"音乐"在相关文档的hobby字段中出现的频率)和反向文档频率(inverse document frequency,即词"音乐"在所有文档的hobby字段中出现的频率),以及字段的长度(即字段越短相关度越高)相结合的计算方式.

### 1.2.3.多词搜索

~~~shell
POST http://172.16.124.131:9200/fechin01/person01/_search
#请求体
{
	"query":{
		"match":{
			"hobby":"音乐 篮球"
		}
	},
	"highlight":{
		"fields":{
			"hobby":{}
		}
	}
}
#响应体
{
    "took": 22,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 4,
        "max_score": 1.3192271,
        "hits": [
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "D6nHznAByqluhSOWWW9g",
                "_score": 1.3192271,
                "_source": {
                    "name": "王五",
                    "age": 22,
                    "mail": "333@qq.com",
                    "hobby": "羽毛球,篮球,游泳,听音乐"
                },
                "highlight": {
                    "hobby": [
                        "羽毛球,<em>篮球</em>,游泳,听<em>音乐</em>"
                    ]
                }
            },
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "EanHznAByqluhSOWWW9g",
                "_score": 0.81652206,
                "_source": {
                    "name": "孙七",
                    "age": 24,
                    "mail": "555@qq.com",
                    "hobby": "听音乐,看电影,羽毛球"
                },
                "highlight": {
                    "hobby": [
                        "听<em>音乐</em>,看电影,羽毛球"
                    ]
                }
            },
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "EKnHznAByqluhSOWWW9g",
                "_score": 0.6987338,
                "_source": {
                    "name": "赵六",
                    "age": 23,
                    "mail": "444@qq.com",
                    "hobby": "跑步,游泳,篮球"
                },
                "highlight": {
                    "hobby": [
                        "跑步,游泳,<em>篮球</em>"
                    ]
                }
            },
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "DqnHznAByqluhSOWWW9g",
                "_score": 0.50270504,
                "_source": {
                    "name": "李四",
                    "age": 21,
                    "mail": "222@qq.com",
                    "hobby": "羽毛球,乒乓球,足球,篮球"
                },
                "highlight": {
                    "hobby": [
                        "羽毛球,乒乓球,足球,<em>篮球</em>"
                    ]
                }
            }
        ]
    }
}
~~~

由上述结果可以得到查出来的是既包含"音乐"也包含"篮球"的用户,结果返回的是"或"的关系.在ElasticSearch中,可以指定词之间的逻辑关系.

~~~shell
POST http://172.16.124.131:9200/fechin01/person01/_search
#请求体
{
	"query":{
		"match":{
			"hobby":{
				"query":"音乐 篮球",
				"operator":"and"
			}
		}
	},
	"highlight":{
		"fields":{
			"hobby":{}
		}
	}
}
#响应体
{
    "took": 40,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 1,
        "max_score": 1.3192271,
        "hits": [
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "D6nHznAByqluhSOWWW9g",
                "_score": 1.3192271,
                "_source": {
                    "name": "王五",
                    "age": 22,
                    "mail": "333@qq.com",
                    "hobby": "羽毛球,篮球,游泳,听音乐"
                },
                "highlight": {
                    "hobby": [
                        "羽毛球,<em>篮球</em>,游泳,听<em>音乐</em>"
                    ]
                }
            }
        ]
    }
}
~~~

我们可以看到结果是符合预期的.前面我们测试了"OR"和"AND"搜索,这是两个极端,在实际场景中,并不会选取这两个极端,我们只需要符合一定的相似度就可以查询到数据,在ElasticSearch中也支持这样的查询,通过minimum_should_match来指定匹配度,如70%.

~~~shell
POST http://172.16.124.131:9200/fechin01/person01/_search
#请求体
{
	"query":{
		"match":{
			"hobby":{
				"query":"音乐 篮球",
				"minimum_should_match":"80%"
			}
		}
	},
	"highlight":{
		"fields":{
			"hobby":{}
		}
	}
}
~~~

相似度多少合适,需要在实际的需求中进行反复测试,才可得到合理的值

### 1.2.4.组合搜索

~~~shell
POST http://172.16.124.131:9200/fechin01/person01/_search
#请求体
{
	"query":{
		"bool":{
			"must":{
				"match":{
					"hobby":"篮球"
				}
			},
			"must_not":{
				"match":{
					"hobby":"音乐"
				}
			},
			"should":{
				"match":{
					"hobby":"游泳"
				}
			}
		}
	},
	"highlight":{
		"fields":{
			"hobby":{}
		}
	}
}
#响应体
{
    "took": 22,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 2,
        "max_score": 1.8336569,
        "hits": [
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "EKnHznAByqluhSOWWW9g",
                "_score": 1.8336569,
                "_source": {
                    "name": "赵六",
                    "age": 23,
                    "mail": "444@qq.com",
                    "hobby": "跑步,游泳,篮球"
                },
                "highlight": {
                    "hobby": [
                        "跑步,<em>游泳</em>,<em>篮球</em>"
                    ]
                }
            },
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "DqnHznAByqluhSOWWW9g",
                "_score": 0.50270504,
                "_source": {
                    "name": "李四",
                    "age": 21,
                    "mail": "222@qq.com",
                    "hobby": "羽毛球,乒乓球,足球,篮球"
                },
                "highlight": {
                    "hobby": [
                        "羽毛球,乒乓球,足球,<em>篮球</em>"
                    ]
                }
            }
        ]
    }
}
~~~

上面搜索的结果的意思是:搜索结果中必须包含篮球,不能包含音乐,如果包含游泳,那么相似度会更高.

> 评分的计算规则:
>
> * bool查询会为每个文档计算相关度评分\_score,再讲所有匹配的must和should语句的分数\_score求和,最后除以must和should语句的总数;
> * must_not语句不会影响评分,它的作用只是将不相关的文档排除;

默认情况下,should中的内容不是必须匹配的,如果查询语句中没有must,那么就会至少匹配其中一个,当然,也可以通过`minimum_should_match`参数进行控制,该值可以是数字也可以是百分比.

~~~shell
POST http://172.16.124.131:9200/fechin01/person01/_search
#请求体
{
	"query":{
		"bool":{
			"should":[
				{
					"match":{
						"hobby":"游泳"
					}
				},
				{
					"match":{
						"hobby":"篮球"
					}
				},
				{
					"match":{
						"hobby":"音乐"
					}
				}
			],
			"minimum_should_match":2
		}
	},
	"highlight":{
		"fields":{
			"hobby":{}
		}
	}
}
#响应体
{
    "took": 25,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 4,
        "max_score": 2.135749,
        "hits": [
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "D6nHznAByqluhSOWWW9g",
                "_score": 2.135749,
                "_source": {
                    "name": "王五",
                    "age": 22,
                    "mail": "333@qq.com",
                    "hobby": "羽毛球,篮球,游泳,听音乐"
                },
                "highlight": {
                    "hobby": [
                        "羽毛球,<em>篮球</em>,<em>游泳</em>,听<em>音乐</em>"
                    ]
                }
            },
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "EKnHznAByqluhSOWWW9g",
                "_score": 1.8336569,
                "_source": {
                    "name": "赵六",
                    "age": 23,
                    "mail": "444@qq.com",
                    "hobby": "跑步,游泳,篮球"
                },
                "highlight": {
                    "hobby": [
                        "跑步,<em>游泳</em>,<em>篮球</em>"
                    ]
                }
            },
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "EanHznAByqluhSOWWW9g",
                "_score": 0.81652206,
                "_source": {
                    "name": "孙七",
                    "age": 24,
                    "mail": "555@qq.com",
                    "hobby": "听音乐,看电影,羽毛球"
                },
                "highlight": {
                    "hobby": [
                        "听<em>音乐</em>,看电影,羽毛球"
                    ]
                }
            },
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "DqnHznAByqluhSOWWW9g",
                "_score": 0.50270504,
                "_source": {
                    "name": "李四",
                    "age": 21,
                    "mail": "222@qq.com",
                    "hobby": "羽毛球,乒乓球,足球,篮球"
                },
                "highlight": {
                    "hobby": [
                        "羽毛球,乒乓球,足球,<em>篮球</em>"
                    ]
                }
            }
        ]
    }
}
~~~

`minimum_should_match`值为2,意思是should中的三个词,至少要满足2个.

### 1.2.5.权重

有些时候,我们可能需要对某些词增加权重来影响该条数数据的得分,如下:

~~~shell
POST http://172.16.124.131:9200/fechin01/person01/_search
#请求体
{
	"query":{
		"bool":{
			"must":{
				"match":{
					"hobby":{
						"query":"游泳篮球",
						"operator":"and"
					}
				}
			},
			"should":[
				{
					"match":{
						"hobby":{
							"query":"音乐",
							"boost":10
						}
					}
				},
				{
					"match":{
						"hobby":{
							"query":"跑步",
							"boost":2
						}
					}
				}
				
			]
		}
	},
	"highlight":{
		"fields":{
			"hobby":{}
		}
	}
}
~~~

在添加完权重之后,我们发现`_socre`会和不加权重完全不一样.

### 1.2.6.短语匹配

在ElasticSearch中,短语匹配意味着不仅仅是词要匹配,并且词的顺序也要一致,如下:

~~~shell
POST http://172.16.124.131:9200/fechin01/person01/_search
#请求体
{
	"query":{
		"match_phrase":{
			"hobby":{
				"query":"羽毛球篮球"
			}
		}
	},
	"highlight":{
		"fields":{
			"hobby":{}
		}
	}
}
#响应体
{
    "took": 61,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 1,
        "max_score": 1.307641,
        "hits": [
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "D6nHznAByqluhSOWWW9g",
                "_score": 1.307641,
                "_source": {
                    "name": "王五",
                    "age": 22,
                    "mail": "333@qq.com",
                    "hobby": "羽毛球,篮球,游泳,听音乐"
                },
                "highlight": {
                    "hobby": [
                        "<em>羽毛</em><em>球</em>,<em>篮球</em>,游泳,听音乐"
                    ]
                }
            }
        ]
    }
}
~~~

如果觉得这样太过于苛刻,可以增加`slop`参数,允许跳过N个词进行匹配

~~~shell
POST http://172.16.124.131:9200/fechin01/person01/_search
#请求体
{
	"query":{
		"match_phrase":{
			"hobby":{
				"query":"羽毛球足球",
				"slop":3
			}
		}
	},
	"highlight":{
		"fields":{
			"hobby":{}
		}
	}
}
#响应体
{
    "took": 9,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": 2,
        "max_score": 0.6476142,
        "hits": [
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "DanHznAByqluhSOWWW9g",
                "_score": 0.6476142,
                "_source": {
                    "name": "张三",
                    "age": 20,
                    "mail": "111@qq.com",
                    "hobby": "羽毛球,乒乓球,足球"
                },
                "highlight": {
                    "hobby": [
                        "<em>羽毛</em><em>球</em>,乒乓球,<em>足球</em>"
                    ]
                }
            },
            {
                "_index": "fechin01",
                "_type": "person01",
                "_id": "DqnHznAByqluhSOWWW9g",
                "_score": 0.594337,
                "_source": {
                    "name": "李四",
                    "age": 21,
                    "mail": "222@qq.com",
                    "hobby": "羽毛球,乒乓球,足球,篮球"
                },
                "highlight": {
                    "hobby": [
                        "<em>羽毛</em><em>球</em>,乒乓球,<em>足球</em>,篮球"
                    ]
                }
            }
        ]
    }
}
~~~

# 2.搭建ElasticSearch集群

## 2.1.集群节点

ElasticSearch的集群是由多个节点组成的,通过cluster.name设置集群名称,并且用于区分其它的集群,每个节点通过node.name指定节点名称.

在ElasticSearch中,节点的类型主要有4种:

* master节点
  * 配置文件中`node.master`属性为true(默认为true),就有资格被选为master节点;
  * master节点用于控制整个集群的操作,比如创建或删除索引,管理其它非master节点.
* data节点
  * 配置文件中`node.data`属性为true(默认为true),就有资格被设置成data节点;
  * data节点主要用于执行数据相关的操作,比如文档的CRUD.
* 客户端节点
  * 配置文件中`node.master`属性和`node.data`属性均为false;
  * 该节点不能作为master节点,也不能作为data节点;
  * 可以作为客户端节点,用于响应客户的请求,把请求转发到其他节点
* 部落节点
  * 当一个节点配置`tribe.*`的时候,它是一个特殊的客户端,它可以连接多个集群,在所有连接的集群上执行搜索和其他操作.

## 2.2.使用docker搭建集群

~~~shell
#在/opt目录下创建elasticsearch的挂载目录/elasticsearch/node01 /elasticsearch/node02
#复制安装目录下的elasticsearch.yml、jvm.options文件，做如下修改:

#容器一的elasticsearch.yml配置
cluster.name: es-fechin-cluster
node.name: node01
network.host: 172.16.124.131 
http.port: 9200
discovery.zen.ping.unicast.hosts: ["172.16.124.131"]
discovery.zen.minimum_master_nodes: 1
http.cors.enabled: true
http.cors.allow-origin: "*"
node.master: true
node.data: true
#容器二的elasticsearch.yml配置
cluster.name: es-fechin-cluster
node.name: node02
network.host: 172.16.124.131 
http.port: 9201
discovery.zen.ping.unicast.hosts: ["172.16.124.131"]
discovery.zen.minimum_master_nodes: 1
http.cors.enabled: true
http.cors.allow-origin: "*"
node.master: false
node.data: true

#创建容器
docker create --name es-node01 --net host -v /opt/elasticsearch/node01/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml -v /opt/elasticsearch/node01/jvm.options:/usr/share/elasticsearch/config/jvm.options -v /opt/elasticsearch/node01/data:/usr/share/elasticsearch/data elasticsearch:6.5.4

docker create --name es-node02 --net host -v /opt/elasticsearch/node02/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml -v /opt/elasticsearch/node02/jvm.options:/usr/share/elasticsearch/config/jvm.options -v /opt/elasticsearch/node02/data:/usr/share/elasticsearch/data elasticsearch:6.5.4
#修改文件夹权限,在修改权限之前需要创建文件夹
chmod 777 /opt/elasticsearch/node01/data
chmod 777 /opt/elasticsearch/node02/data
#启动容器
docker start es-node01 && docker logs -f es-node01
docker start es-node02 && docker logs -f es-node02
~~~

在启动的过程中会遇到memory areas is too low的问题

![image-20200314134424764](14-Elastic Stack(ElasticSearch高级).assets/image-20200314134424764.png)

我们需要:

~~~shell
vi /etc/sysctl.conf 
#添加如下配置
vm.max_map_count=655360
#执行命令
sysctl -p
~~~

再次重启elasticsearch容器,完成.

下面我们进行测试:创建索引

![image-20200314135457240](14-Elastic Stack(ElasticSearch高级).assets/image-20200314135457240.png)

![image-20200314135711554](14-Elastic Stack(ElasticSearch高级).assets/image-20200314135711554.png)

查看集群状态:http://172.16.124.131:9200/_cluster/health

~~~json
{
    "cluster_name": "es-fechin-cluster",
    "status": "green",
    "timed_out": false,
    "number_of_nodes": 2,
    "number_of_data_nodes": 2,
    "active_primary_shards": 5,
    "active_shards": 10,
    "relocating_shards": 0,
    "initializing_shards": 0,
    "unassigned_shards": 0,
    "delayed_unassigned_shards": 0,
    "number_of_pending_tasks": 0,
    "number_of_in_flight_fetch": 0,
    "task_max_waiting_in_queue_millis": 0,
    "active_shards_percent_as_number": 100
}
~~~

集群状态的三种颜色:

| 颜色   | 意义                                        |
| ------ | ------------------------------------------- |
| green  | 所有主要分片和复制分片都可用                |
| yellow | 所有主要的分片可用,但不是所有复制分片都可用 |
| red    | 不是所有的主要分片都可用                    |

## 2.3.分片和副本

为了将数据添加到ElasticSearch中,我们需要**索引(index)**----一个存储关联数据的地方.实际上,索引只是一个用来指向一个或多个**分片(shards)**的"逻辑命名空间(logical namespace)".

* 一个分片(shard)是一个最小级别的工作单元,它只是保存了索引中所有数据的一部分;
* 我们需要知道是分片就是一个Lucene实例,并且它本身就是一个完整的搜索引擎,应用程序不会和它直接通信;
* 分片可以是主分片(primary shard)或者是复制分片(replica shard);
* 索引中的每个文档都属于一个单独的主分片,所以主分片的数量决定了索引最多存储多少数据;
* 复制分片只是主分片的一个副本,它可以防止硬件故障导致的数据丢失,同时可以提供读请求,比如搜索或者从别的shard取回文档;
* 当索引创建完成的时候,主分片的数量就固定了,但是复制分片的数量可以随时调整;

## 2.4.故障转移

为了测试故障转移,需要再向集群中添加一个节点,并且将所有节点的node.master设置为true;

我们再创建索引,分片数为5,副本数为1,如下图所示

![image-20200314145052664](14-Elastic Stack(ElasticSearch高级).assets/image-20200314145052664.png)

### 2.4.1.测试

![image-20200314145425918](14-Elastic Stack(ElasticSearch高级).assets/image-20200314145425918.png)

当前集群状态为黄色,表示主界定啊可用,副本节点不完全可用;

![image-20200314145553586](14-Elastic Stack(ElasticSearch高级).assets/image-20200314145553586.png)

过一段时间,发现节点列表中看不到node01,副本节点分配到了node02和node03,集群状态恢复到了绿色.

![image-20200314150209598](14-Elastic Stack(ElasticSearch高级).assets/image-20200314150209598.png

![image-20200314150224061](14-Elastic Stack(ElasticSearch高级).assets/image-20200314150224061.png)

我们再将node01恢复,如上图:我们发现node01无法加入到集群中了,这其实就是集群中的的脑裂问题.

### 2.4.2.脑裂问题

![image-20200314151223268](14-Elastic Stack(ElasticSearch高级).assets/image-20200314151223268.png)

解决方案:

* 不能让节点很容易变成master,必须有多个节点确认后才可以;
* 设置`minimum_master_nodes`的大小为(N/2)+1,N为集群中节点数;

## 2.5.分布式文档

### 2.5.1.路由

![image-20200314154125483](14-Elastic Stack(ElasticSearch高级).assets/image-20200314154125483.png)

如图所示:当我们想一个集群保存文档时,文档该存储到哪个节点呢?是随机吗?是轮询吗?

实际上,在ElasticSearch中,会采用计算的方式来确定存储到哪个节点,计算公式如下:

> `shard = hash(routing)% number_of_primary_shards`

* routing值是一个任意的字符串,它默认是_id,但也可以自定义;
* 这个routing字符串通过哈希函数生成一个数字,然后除以主切片的数量得到一个余数(remainder),余数的范围永远是0到number_of_primary_shards - 1,这个数字就是特定文档所在的分片;
* 这也就是为什么创建了主分片后,不能修改的原因;

### 2.5.2.文档的写操作

![image-20200314155105352](14-Elastic Stack(ElasticSearch高级).assets/image-20200314155105352.png)

新建,索引,删除请求都是写(write)操作,它们必须在主分片上成功完成才能复制到相关的复制分片上.步骤如下:

1. 客户端给Node01发送新建,索引或删除请求;
2. 节点使用文档的`_id`确定文档属于主分片0,她转发请求到Node3,主分片0位于这个节点上.
3. Node3在主分片上执行请求,如果成功,它转发请求到响应的位于Node1和Node2的复制节点上.当所有的复制节点都报告成功,Node3报告成功到请求的节点,请求节点再报告给客户端.
4. 客户端接收到成功响应的时候,文档的修改已经被应用于主分片和所有复制分片,修改生效.

### 2.5.3.搜索单个文档

![image-20200314155835478](14-Elastic Stack(ElasticSearch高级).assets/image-20200314155835478.png)

1. 客户端给Node01发送get请求.
2. 节点使用文档的_id确定文档属于分片0,分片0对应的复制分片三个节点上都有,此时,她转发请求到了Node02.
3. Node02返回文档给Node01然后返回给客户端

对于读请求,为了平衡负载,请求节点会为每个请求选择不同的分片,它会循环所有分片副本;

可能的情况是:一个索引的文档已经存在于主分片上确还没有来得及同步到复制分片上.这时复制分片会报告文档未找到,主分片会成功返回文档,一旦索引请求成功返回给用户,文档则在主分片和复制分片都是可用的.

### 2.5.4.全文搜索

全文搜索分为两个阶段:搜索(query)+取回(fetch)

![image-20200314160521412](14-Elastic Stack(ElasticSearch高级).assets/image-20200314160521412.png)

#### 2.5.4.1.搜索

1. 客户端发送一个search请求给Node3,Node3创建了一个长度为from+size的空优先级队;
2. Node3转发这个搜索请求到索引中每个分片的原本或副本,每个分片在本地执行这个查询并且将结果到一个大小为from+size的有序本地优先队列里去;
3. 每个分片返回document的ID和它优先队列里的所有documet的排序值给协调节点Node3.Node3把这些值合并到自己的优先队列里产生全局排列结果;

#### 2.5.4.2.取回

1. 协调节点辨别出哪个document需要取回,并且向相关分片发出GET请求;
2. 每个分片加载document并且根据需要丰富(enrich)它们,然后再将document返回协调节点.
3. 一旦所有的document都被取回,协调节点会将结果返回给客户端.

# 3.Java客户端

在ElasticSearch中,为Java提供了2种客户端,一种是Rest风格的客户端.另一种是Java API的客户端.https://www.elastic.co/guide/en/elasticsearch/client/index.html

![image-20200314173131710](14-Elastic Stack(ElasticSearch高级).assets/image-20200314173131710.png)

## 3.1.REST客户端

ElasticSearch提供了2种REST客户端,一种是低级客户端,一种是高级客户端

* Java Low Level REST Client :官方提供的低级客户端.该客户端通过http来连接ElasticSearch集群.用户在使用该客户端需要将请求数据手动拼接成ElasticSearch所需JSON格式进行发送,收到响应同时也需要将返回的JSON数据手动封装成对象.虽然麻烦,不过客户端兼容所有的ElasticSearch版本;
* Java High Level Rest Client:官方提供的高级客户端.该客户端基于低级客户端实现,它提供了很多边界的API来解决低级客户端需要手动转换数据格式的问题.

## 3.2.构造函数

~~~shell
POST http://172.16.124.131:9200/haoke/house/_bulk
#请求体
{"index":{"_index":"haoke","_type":"house"}}
{"id":"1001","title":"整租 · 南丹大楼 1居室 7500","price":"7500"}
{"index":{"_index":"haoke","_type":"house"}}
{"id":"1002","title":"陆家嘴板块，精装设计一室一厅，可拎包入住诚意租。","price":"8500"}
{"index":{"_index":"haoke","_type":"house"}}
{"id":"1003","title":"整租 · 健安坊 1居室 4050","price":"7500"}
{"index":{"_index":"haoke","_type":"house"}}
{"id":"1004","title":"整租 · 中凯城市之光+视野开阔+景色秀丽+拎包入住","price":"6500"}
{"index":{"_index":"haoke","_type":"house"}}
{"id":"1005","title":"整租 · 南京西路品质小区 21213三轨交汇 配套齐* 拎包入 住","price":"6000"}
{"index":{"_index":"haoke","_type":"house"}}
{"id":"1006","title":"祥康里 简约风格 *南户型 拎包入住 看房随时","price":"7000"}
~~~

## 3.3.REST低级客户端

### 3.3.1.POM文件

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.0.RELEASE</version>
    </parent>

    <groupId>org.example</groupId>
    <artifactId>demo-elasticsearch-low-level</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.4</version>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-client</artifactId>
            <version>6.5.4</version>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-high-level-client</artifactId>
            <version>6.5.4</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.9.4</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- java编译插件 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.2</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
~~~

### 3.3.2.编写测试用例

~~~java
package org.fechin.elasticsearch.test;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.http.HttpHost;
import org.apache.http.util.EntityUtils;
import org.elasticsearch.client.*;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;

/**
 * @Author:朱国庆
 * @Date：2020/3/14 19:00
 * @Desription: haoke-manage
 * @Version: 1.0
 */
public class TestElasticSearchRest {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private RestClient restClient;

    @Before
    public void init(){
        RestClientBuilder restClientBuilder = RestClient.builder(
                new HttpHost("172.16.124.131", 9200, "http"),
                new HttpHost("172.16.124.131", 9201, "http"),
                new HttpHost("172.16.124.131", 9202, "http"));

        restClientBuilder.setFailureListener(new RestClient.FailureListener() {
            @Override
            public void onFailure(Node node){
                System.out.println("出错了 -> " + node);
            }
        });

        this.restClient = restClientBuilder.build();
    }

    @After
    public void after() throws IOException {
        restClient.close();
    }

    /**
     * 查询集群的状态
     * @throws IOException
     */
    @Test
    public void testInfo() throws IOException {
        Request request = new Request("GET", "/_cluster/state");
        request.addParameter("pretty", "true");
        Response response = this.restClient.performRequest(request);

        System.out.println("请求完成 -> "+response.getStatusLine());
        System.out.println(EntityUtils.toString(response.getEntity()));

    }

    /**
     * 新增数据
     * @throws Exception
     */
    @Test
    public void testSave() throws Exception{
        Request request = new Request("POST", "/haoke/house");
        request.addParameter("pretty", "true");

        // 构造数据
        Map<String, Object> data = new HashMap<>();
        data.put("id", 2001);
        data.put("title", "南京西路 一室一厅");
        data.put("price", 3500);

        String json = MAPPER.writeValueAsString(data);
        request.setJsonEntity(json);

        Response response = this.restClient.performRequest(request);
        System.out.println("请求完成 -> "+response.getStatusLine());
        System.out.println(EntityUtils.toString(response.getEntity()));

    }

    // 根据id查询数据
    @Test
    public void testQueryData() throws IOException {
        Request request = new Request("GET", "/haoke/house/E4vC2HABnqcVXysGZA_M");
        request.addParameter("pretty", "true");

        Response response = this.restClient.performRequest(request);

        System.out.println(response.getStatusLine());
        System.out.println(EntityUtils.toString(response.getEntity()));
    }

    // 搜索数据
    @Test
    public void testSearchData() throws IOException {
        Request request = new Request("POST", "/haoke/house/_search");
        String searchJson = "{\"query\": {\"match\": {\"title\": \"拎包入住\"}}}";
        request.setJsonEntity(searchJson);
        request.addParameter("pretty","true");
        Response response = this.restClient.performRequest(request);
        System.out.println(response.getStatusLine());
        System.out.println(EntityUtils.toString(response.getEntity()));
    }
}
~~~

## 3.4.REST高级客户端

### 3.4.1.POM文件

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>


    <groupId>org.example</groupId>
    <artifactId>demo-elasticsearch-high-level</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-high-level-client</artifactId>
            <version>6.5.4</version>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-client</artifactId>
            <version>6.5.4</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.9.4</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- java编译插件 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.2</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
~~~

### 3.4.2.编写测试用例

~~~java
package org.fechin.elasticsearch.test;

import org.apache.http.HttpHost;
import org.elasticsearch.action.ActionListener;
import org.elasticsearch.action.delete.DeleteRequest;
import org.elasticsearch.action.delete.DeleteResponse;
import org.elasticsearch.action.get.GetRequest;
import org.elasticsearch.action.get.GetResponse;
import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.action.index.IndexResponse;
import org.elasticsearch.action.search.SearchRequest;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.action.update.UpdateRequest;
import org.elasticsearch.action.update.UpdateResponse;
import org.elasticsearch.client.RequestOptions;
import org.elasticsearch.client.RestClient;
import org.elasticsearch.client.RestClientBuilder;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.common.Strings;
import org.elasticsearch.common.unit.TimeValue;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.search.SearchHit;
import org.elasticsearch.search.SearchHits;
import org.elasticsearch.search.builder.SearchSourceBuilder;
import org.elasticsearch.search.fetch.subphase.FetchSourceContext;
import org.junit.After;
import org.junit.Before;
import org.junit.Test;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;

/**
 * @Author:朱国庆
 * @Date：2020/3/14 19:37
 * @Desription: haoke-manage
 * @Version: 1.0
 */
public class TestElasticSearchRest {

    private RestHighLevelClient restHighLevelClient;

    @Before
    public void init() {
        RestClientBuilder restClientBuilder = RestClient.builder(
                new HttpHost("172.16.124.131", 9200, "http"),
                new HttpHost("172.16.124.131", 9201, "http"),
                new HttpHost("172.16.124.131", 9202, "http"));
        this.restHighLevelClient = new RestHighLevelClient(restClientBuilder);
    }

    @After
    public void close() throws Exception {
        this.restHighLevelClient.close();
    }

    @Test
    public void testSave() throws Exception {

        Map<String, Object> data = new HashMap<>();
        data.put("id", 2002);
        data.put("title", "南京东路 二室一厅");
        data.put("price", 4000);

        IndexRequest indexRequest = new IndexRequest("haoke", "house").source(data);

        IndexResponse response = this.restHighLevelClient.index(indexRequest, RequestOptions.DEFAULT);

        System.out.println("id -> " + response.getId());
        System.out.println("version -> " + response.getVersion());
        System.out.println("result -> " + response.getResult());
    }

    @Test
    public void testCreateAsync() throws Exception {

        Map<String, Object> data = new HashMap<>();
        data.put("id", "2004");
        data.put("title", "南京东路2 最新房源 二室一厅");
        data.put("price", "5600");

        IndexRequest indexRequest = new IndexRequest("haoke", "house").source(data);

        this.restHighLevelClient.indexAsync(indexRequest, RequestOptions.DEFAULT, new
                ActionListener<IndexResponse>() {
                    @Override
                    public void onResponse(IndexResponse indexResponse) {
                        System.out.println("id->" + indexResponse.getId());
                        System.out.println("index->" + indexResponse.getIndex());
                        System.out.println("type->" + indexResponse.getType());
                        System.out.println("version->" + indexResponse.getVersion());
                        System.out.println("result->" + indexResponse.getResult());
                        System.out.println("shardInfo->" + indexResponse.getShardInfo());
                    }

                    @Override
                    public void onFailure(Exception e) {
                        System.out.println(e);
                    }
                });

        System.out.println("ok");
        Thread.sleep(20000);
    }

    @Test
    public void testQuery() throws Exception {
        GetRequest getRequest = new GetRequest("haoke", "house",
                "Fosz2XABnqcVXysGIQ8l");

        // 指定返回的字段
        String[] includes = new String[]{"title", "id"};
        String[] excludes = Strings.EMPTY_ARRAY;
        FetchSourceContext fetchSourceContext =
                new FetchSourceContext(true, includes, excludes);

        getRequest.fetchSourceContext(fetchSourceContext);

        GetResponse response = this.restHighLevelClient.get(getRequest,
                RequestOptions.DEFAULT);

        System.out.println("数据 -> " + response.getSource());
    }

    /**
     * 判断是否存在
     *
     * @throws Exception
     */
    @Test
    public void testExists() throws Exception {
        GetRequest getRequest = new GetRequest("haoke", "house",
                "Fosz2XABnqcVXysGIQ8l");

        // 不返回的字段
        getRequest.fetchSourceContext(new FetchSourceContext(false));
        boolean exists = this.restHighLevelClient.exists(getRequest, RequestOptions.DEFAULT);
        System.out.println("exists -> " + exists);
    }

    /**
     * 删除数据
     *
     * @throws Exception
     */
    @Test
    public void testDelete() throws Exception {
        DeleteRequest deleteRequest = new DeleteRequest("haoke", "house",
                "Fosz2XABnqcVXysGIQ8l");

        DeleteResponse response = this.restHighLevelClient.delete(deleteRequest,
                RequestOptions.DEFAULT);

        System.out.println(response.status());// OK or NOT_FOUND
    }

    /**
     * 更新数据
     *
     * @throws Exception
     */
    @Test
    public void testUpdate() throws Exception {
        UpdateRequest updateRequest = new UpdateRequest("haoke", "house",
                "EYuH2HABnqcVXysGrQ9D");

        Map<String, Object> data = new HashMap<>();
        data.put("title", "南京西路2 一室一厅2");
        data.put("price", "4000");
        updateRequest.doc(data);

        UpdateResponse response = this.restHighLevelClient.update(updateRequest,
                RequestOptions.DEFAULT);
        System.out.println("version -> " + response.getVersion());
    }

    /**
     * 测试搜索
     *
     * @throws Exception
     */
    @Test
    public void testSearch() throws Exception {
        SearchRequest searchRequest = new SearchRequest("haoke");
        searchRequest.types("house");

        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        sourceBuilder.query(QueryBuilders.matchQuery("title", "拎包入住"));
        sourceBuilder.from(0);
        sourceBuilder.size(5);
        sourceBuilder.timeout(new TimeValue(60, TimeUnit.SECONDS));
        searchRequest.source(sourceBuilder);

        SearchResponse search = this.restHighLevelClient.search(searchRequest,
                RequestOptions.DEFAULT);

        System.out.println("搜索到 " + search.getHits().totalHits + " 条数据.");

        SearchHits hits = search.getHits();
        for (SearchHit hit : hits) {
            System.out.println(hit.getSourceAsString());
        }
    }
}
~~~

# 4.Spring Data ElasticSearch

https://spring.io/projects/spring-data-elasticsearch

## 4.1.POM文件

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.0.RELEASE</version>
    </parent>

    <groupId>org.example</groupId>
    <artifactId>demo-elasticsearch-springboot</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.4</version>
        </dependency>

        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-client</artifactId>
            <version>6.5.4</version>
        </dependency>
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-high-level-client</artifactId>
            <version>6.5.4</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.9.4</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- java编译插件 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.2</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
~~~

## 4.2.application.properties

~~~properties
spring.application.name = demo-elasticsearch-springboot

spring.data.elasticsearch.cluster-name=es-fechin-cluster
spring.data.elasticsearch.cluster-nodes=172.16.124.131:9300,172.16.124.131:9301,172.16.124.131:9302
~~~

## 4.3.启动类(略)

## 4.4.实体类

~~~java
package org.fechin.elasticsearch.pojo;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.Document;
import org.springframework.data.elasticsearch.annotations.Field;

@Data
@AllArgsConstructor
@NoArgsConstructor
@Document(indexName = "fechin", type = "user", createIndex = false)
public class User {

    @Id
    private Long id;
    @Field(store = true)
    private String name;
    @Field
    private Integer age;
    @Field(store = true)
    private String hobby;
}
~~~

## 4.5.测试类

~~~java
package org.fechin.elasticsearch.test;

import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.index.query.QueryBuilders;
import org.fechin.elasticsearch.pojo.User;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.elasticsearch.core.ElasticsearchTemplate;
import org.springframework.data.elasticsearch.core.aggregation.AggregatedPage;
import org.springframework.data.elasticsearch.core.query.*;
import org.springframework.test.context.junit4.SpringRunner;

import java.util.ArrayList;
import java.util.List;

/**
 * @Author:朱国庆
 * @Date：2020/3/14 21:37
 * @Desription: haoke-manage
 * @Version: 1.0
 */

@RunWith(SpringRunner.class)
@SpringBootTest
public class TestSpringBootElasticSearch {
    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;

    @Test
    public void save() {
        User user = new User();
        user.setId(1001L);
        user.setName("张三");
        user.setAge(20);
        user.setHobby("足球、篮球、听音乐");

        IndexQuery indexQuery = new IndexQueryBuilder().withObject(user).build();

        String index = this.elasticsearchTemplate.index(indexQuery);
        System.out.println(index);
    }

    @Test
    public void testBulk() {
        List list = new ArrayList();
        for (int i = 0; i < 5000; i++) {
            User user = new User();
            user.setId(1001L + i);
            user.setAge(i % 50 + 10);
            user.setName("张三" + i);
            user.setHobby("足球、篮球、听音乐");
            IndexQuery indexQuery = new IndexQueryBuilder().withObject(user).build();
            list.add(indexQuery);
        }
        Long start = System.currentTimeMillis();
        this.elasticsearchTemplate.bulkIndex(list);
        System.out.println("用时：" + (System.currentTimeMillis() - start));
    }

    /**
     * 局部更新，全部更新使用index覆盖即可
     */
    @Test
    public void testUpdate() {
        IndexRequest indexRequest = new IndexRequest();
        indexRequest.source("age", "30");

        UpdateQuery updateQuery = new UpdateQueryBuilder()
                .withId("1001")
                .withClass(User.class)
                .withIndexRequest(indexRequest).build();
        this.elasticsearchTemplate.update(updateQuery);
    }

    @Test
    public void testDelete() {
        this.elasticsearchTemplate.delete(User.class, "1001");
    }

    @Test
    public void testSearch() {
        PageRequest pageRequest = PageRequest.of(1, 10); //设置分页参数

        SearchQuery searchQuery = new NativeSearchQueryBuilder()
                .withQuery(QueryBuilders.matchQuery("name", "张三")) // match查询
                .withPageable(pageRequest)
                .build();

        AggregatedPage<User> users =
                this.elasticsearchTemplate.queryForPage(searchQuery, User.class);

        System.out.println("总页数：" + users.getTotalPages()); //获取总页数

        for (User user : users.getContent()) { // 获取搜索到的数据
            System.out.println(user);
        }
    }
}

~~~



