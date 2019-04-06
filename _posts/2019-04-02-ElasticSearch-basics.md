---
layout: post
title:  "ElasticSearch 基础"
categories: tools
tags:  es
author: 刚子
---

* content
{:toc}


## 前言


总结ElasticSearch的基础知识点

##  课程目录











## 一、简易教程

### 1. 下载安装

下载[安装文件](https://www.elastic.co/downloads/past-releases/elasticsearch-5-5-3)到`/opt/elasticsearch`目录下面并解压

```
>cd /opt/elasticsearch
>tar -zxvf elasticsearch-5.5.3.tar.gz
```

### 2. 启动并访问

```
>cd /opt/elasticsearch/elasticsearch-5.5.3
>./bin/elasticsearch
##./bin/elasticsearch -d 后台启动
```

root账户启动会报错：can not run elasticsearch as root，创建独立的用户来启动

```
>groupadd esgroup
>useradd esuser -g esgroup
>passwd esuser
>chown -R esuser:esgroup elasticsearch/
```

root用户关闭防火墙

```
>vi /etc/selinux/config
SELINUX=disabled
>systemctl stop firewalld.service
>systemctl disable firewalld.service
>setenforce 0
>getenforce 0
```

es配置可以使用ip访问

```
>vi config/elasticsearch.yml
network.host: 192.168.237.129   #或者0.0.0.0允许所有人访问
>su root
>vim /etc/security/limits.conf
esuser hard nofile 65536
esuser soft nofile 65536
>vi /etc/sysctl.conf
vm.max_map_count=655360
>sysctl -p
```

### 3. 创建、删除索引

```
curl -X PUT 'http://localhost:9200/weather'
curl -X DELETE 'http://localhost:9200/weather'
```

### 4. 添加文档

```
curl -XPUT "http://localhost:9200/movies/movie/1" -d'
{
    "title": "The Godfather",
    "director": "Francis Ford Coppola",
    "year": 1972,
    "genres": ["Crime", "Drama"]
}'

```

### 5. 搜索所有文档

```
http://localhost:9200/_search # 搜索所有索引和所有类型
http://localhost:9200/movies/_search # 在电影索引中搜索所有类型
http://localhost:9200/movies/movie/_search # 在电影索引中显式搜索电影类型的文档
```

### 6. 安装中文分词插件

```
>./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v5.5.3/elasticsearch-analysis-ik-5.5.3.zip
```

### 7. 创建索引，并对文档字段进行中文分词

```
curl -X PUT 'http://localhost:9200/accounts' -d '
{
  "mappings": {
    "person": {
      "properties": {
        "user": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        },
        "title": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        },
        "desc": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        }
      }
    }
  }
}'
```

### 8. 搜索指定字段

```
curl 'http://localhost:9200/accounts/person/_search'  -d '
{
  "query" : { "match" : { "desc" : "系统" }},
  "from": 1,
  "size": 1
}'

# from指定分页起始位置
# size表示每页几条数据
```

### 9. 配置文件

```
>vim config/elasticsearch.yml
cluster.name=myesclustername

node.name=node_001
```

[重要配置的修改](https://www.elastic.co/guide/cn/elasticsearch/guide/current/important-configuration-changes.html#_%E6%8C%87%E5%AE%9A%E5%90%8D%E5%AD%97)

## 二、查询DSL

* 添加雇员索引文档

```
curl -X PUT "localhost:9200/megacorp/employee/1" -H 'Content-Type: application/json' -d'
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}
'
```

* 创建文档，非更新

```
PUT /website/blog/123?op_type=create
# or
PUT /website/blog/123/_create
```

* 检查文档是否存在

```
curl -i -XHEAD http://localhost:9200/website/blog/123
```

* QueryString搜索

```
curl -X GET "localhost:9200/megacorp/employee/_search?q=last_name:Smith"
```

* 查询表达式(DSL:domain-specific language)搜索指定字段

```
curl -X GET "localhost:9200/megacorp/employee/_search" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match" : {
            "last_name" : "Smith"
        }
    }
}
'
```

* 过滤器--对查询结果进行进一步过滤

```
# 搜索姓氏为 Smith 的雇员，但这次我们只需要年龄大于 30 的
curl -X GET "localhost:9200/megacorp/employee/_search" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "bool": {
            "must": {
                "match" : {
                    "last_name" : "smith" 
                }
            },
            "filter": {
                "range" : {
                    "age" : { "gt" : 30 } 
                }
            }
        }
    }
}
'
```

* 全文搜索

```
curl -X GET "localhost:9200/megacorp/employee/_search" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match" : {
            "about" : "rock climbing"
        }
    }
}
'
```

* 短语搜索--精确匹配内容中出现短语的文档

```
curl -X GET "localhost:9200/megacorp/employee/_search" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    }
}
'
```

* 高亮搜索

```
curl -X GET "localhost:9200/megacorp/employee/_search" -H 'Content-Type: application/json' -d'
{
    "query" : {
        "match_phrase" : {
            "about" : "rock climbing"
        }
    },
    "highlight": {
        "fields" : {
            "about" : {}
        }
    }
}
'
```

更多高亮设置参考[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-highlighting.html#highlighting-settings)

* 数据分析--聚合、汇总

[分析](https://www.elastic.co/guide/cn/elasticsearch/guide/current/_analytics.html#_analytics)

* 通过版本号更新数据

```
curl -X PUT "localhost:9200/website/blog/1?version=1" -H 'Content-Type: application/json' -d'
{
  "title": "My first blog entry",
  "text":  "Starting to get the hang of this..."
}
'
```

更多请参考[乐观并发控制](https://www.elastic.co/guide/cn/elasticsearch/guide/current/optimistic-concurrency-control.html#optimistic-concurrency-control)

* 查看索引、文档信息

```
# 查看索引的文档数量
GET _cat/count/freshsharepro?v

# 查看文档字段信息
GET freshsharepro/commodity/_mapping
```

## 三、Kibana

### 下载安装

从[下载地址](https://www.elastic.co/cn/downloads/past-releases/kibana-5-5-3)下载到`/opt/elasticsearch`目录下，安装方法参考[安装 Kibana](https://www.elastic.co/guide/cn/kibana/current/install.html)

设置任何人都可以访问

```
>vim confg/kibana.yml
server.host: "0.0.0.0"
```

### 启动并访问

```
>./bin/kibana
>./bin/kibana &  #后面添加&代表后台启动，shell窗口执行exit命令后kibana会一直后台启动
```

### 查询语法

[query string syntax](https://www.elastic.co/guide/en/elasticsearch/reference/5.5/query-dsl-query-string-query.html#query-string-syntax)

## 四、中文分词

### 英文分词示例

```
PUT test/doc/1
{
  "msg":"Eating an apple a day keeps doctor away"
}

# 使用单词eat无法搜索到包含eating的内容
POST test/_search
{
  "query":{
    "match":{
      "msg":"eat"
    }
  }
}

# 分析一下字段的分词规则，发现默认的standard分词器没有把eating切分为eat
POST test/_analyze
{
  "field": "msg",
  "text": "Eating an apple a day keeps doctor away"
}

# 写分词器默认standard，也可以指定，写分词器一经指定就不能修改，如果修改的话只能重建索引
PUT test/_mapping/doc
{
  "properties": {
    "msg_english":{
      "type":"text",
      "analyzer": "english"
    }
  }
}

POST test/doc/2
{
  "msg":"Eating an apple a day keeps doctor away",
  "msg_english":"Eating an apple a day keeps doctor away"
}

# 读分词器不指定的话默认与写分词器一致，一般来讲不需要特别指定读时分词器
POST test/_search
{
  "query":{
    "match":{
      "msg_english":{
        "query": "eat"
      }
    }
  }
}

POST test/_analyze
{
  "field": "msg_english",
  "text": "Eating an apple a day keeps doctor away"
}

# standard分词器添加三个过滤器之后效果与english分词器一样，stemmer指的是词干提取
POST _analyze
{
  "char_filter": [], 
  "tokenizer": "standard",
  "filter": [
    "stop",
    "lowercase",
    "stemmer"
  ],
  "text": "Eating an apple a day keeps doctor away"
}
```

### 安装使用中文分词插件ik

```
>./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v5.5.3/elasticsearch-analysis-ik-5.5.3.zip
```

* 指定索引中文档的分词器的类型为ik

```
# 试一下IK的分词器分词效果
POST _analyze
{
  "analyzer": "ik_max_word",
  "text": "我爱北京天安门"
}
# 或者
POST _analyze
{
  "char_filter": [], 
  "tokenizer": "ik_max_word",
  "filter": [],
  "text": "我爱北京天安门"
}

# 设置索引文档字段的分词器，注意只能设置一次，否则只能重新创建索引
PUT /test1
{
  "mappings": {
    "person": {
      "properties": {
        "user": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        },
        "title": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        },
        "desc": {
          "type": "text",
          "analyzer": "ik_max_word",
          "search_analyzer": "ik_max_word"
        }
      }
    }
  }
}
PUT test1/person/1
{
  "user":"用户1",
  "title":"标题",
  "desc":"注意只能设置一次，否则只能重新创建索引"
}
GET test1/person/_search
{
  "query": {
    "match": {
      "desc": "注意"
    }
  }
}
# 查看字段分词器用的是哪个
GET test1/person/_mapping
```

### 自定义静态词库文件

```
# 创建自定义词库文件my.dic
>/opt/elasticsearch/elasticsearch-5.5.3/config/analysis-ik
>vim my.dic
我爱北京天安门

#修改ik配置文件，指定词库文件
>vim IKAnalyzer.cfg.xml

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
        <comment>IK Analyzer 扩展配置</comment>
        <!--用户可以在这里配置自己的扩展字典 -->
        <entry key="ext_dict">my.dic</entry>
         <!--用户可以在这里配置自己的扩展停止词字典-->
        <entry key="ext_stopwords"></entry>
        <!--用户可以在这里配置远程扩展字典 -->
        <!-- <entry key="remote_ext_dict">words_location</entry> -->
        <!--用户可以在这里配置远程扩展停止词字典-->
        <!-- <entry key="remote_ext_stopwords">words_location</entry> -->
</properties>

# 重启ES试一下分词效果
POST _analyze
{
  "char_filter": [], 
  "tokenizer": "ik_max_word",
  "filter": [],
  "text": "我爱北京天安门"
}
POST _analyze
{
  "analyzer": "ik_max_word",
  "text": "我爱北京天安门"
}
```

### 自定义动态词库文件

```
# 设置动态词库url地址
>cd /opt/tomcat/apache-tomcat-8.5.37/webapps/ROOT
>vim my.dic
我爱

#修改ik配置文件，指定词库文件
>vim IKAnalyzer.cfg.xml

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
        <comment>IK Analyzer 扩展配置</comment>
        <!--用户可以在这里配置自己的扩展字典 -->
        <entry key="ext_dict">my.dic</entry>
         <!--用户可以在这里配置自己的扩展停止词字典-->
        <entry key="ext_stopwords"></entry>
        <!--用户可以在这里配置远程扩展字典 -->
        <entry key="remote_ext_dict">http://192.168.255.131:8080/my.dic</entry>
        <!--用户可以在这里配置远程扩展停止词字典-->
        <!-- <entry key="remote_ext_stopwords">words_location</entry> -->
</properties>

# 重启ES试一下分词效果
POST _analyze
{
  "char_filter": [], 
  "tokenizer": "ik_max_word",
  "filter": [],
  "text": "我爱北京天安门"
}
# 修改自定义远程词库之后会有最多1分钟生效时间
```

### 维护ik自定义词库到数据库(未经实验)

```java
//提供head请求服务，让ES隔一分钟调用一次，判断词库是否发生变化
@RequestMapping(value="/es/getCustomDic",method=RequestMethod.HEAD)
public void getCustomDic(HttpServletRequest request,HttpServletResponse response) throws Exception{
    String latest_ETags=getLatest_ETags();
    String old_ETags=request.getHeader("If-None-Match");
    if(latest_ETags.equals("")||!latest_ETags.equals(old_ETags)){
        refreshETags();
        response.setHeader("Etag", getLatest_ETags());
    }
}

//相同的服务地址，get请求获取自定义词库字符串
@RequestMapping(value="/es/getCustomDic",method=RequestMethod.GET,produces = {"text/html;charset=utf-8"})
public String getCustomDic(HttpServletRequest request,HttpServletResponse response) throws Exception{
    String old_ETags=request.getHeader("If-None-Match");
    logger.info("get请求，old_ETags="+old_ETags);
    StringBuilder hotwordStr=new StringBuilder();
    //先让热词状态改为生效状态
    HWIceServiceClient.getServicePrx(HotWordIPrx.class).updateHotWordIsEffect("family");
    //说明第一次请求或者最新标示已经更新
    List<String> hotWord=HWIceServiceClient.getServicePrx(HotWordIPrx.class).getAllHotWords("family");
    logger.info("新的热词加入,个数为: "+hotWord.size());
    hotWord.forEach(str->{
        hotwordStr.append(str+"\r\n");
    });
    refreshETags();
    return hotwordStr.toString();
}
```

## 五、同义词

### 维护静态同义词词库

* 创建词库文件

```
>cd /opt/elasticsearch/elasticsearch-5.5.3/config
>mkdir analysis
>cd analysis
>vim synonyms.txt
西红柿,番茄,土豆,马铃薯
社保,公积金
```

* 创建索引，添加自定义分词器，指定同义词过滤器、同义词库文件地址

```
PUT /synonymtest
{
    "settings": {
    "index": {
      "analysis": {
        "analyzer": {
          "jt_cn": {
            "type": "custom",
            "use_smart": "true",
            "tokenizer": "ik_smart",
            "filter": ["jt_tfr","jt_sfr"],
            "char_filter": ["jt_cfr"]
          },
          "ik_smart": {
            "type": "ik_smart",
            "use_smart": "true"
          },
          "ik_max_word": {
            "type": "ik_max_word",
            "use_smart": "false"
          }
        },
        "filter": {
          "jt_tfr": {
            "type": "stop",
            "stopwords": [" "]
          },
          "jt_sfr": {
            "type": "synonym",
            "synonyms_path": "analysis/synonyms.txt" //这个是相对于${es_home}/config目录而言的地址
          }
        },
        "char_filter": {
            "jt_cfr": {
                "type": "mapping",
                "mappings": [
                    "| => \\|"
                ]
            }
        }
      }
    }
  }
}
```

* 给文档字段创建映射，指定自定义分词器

```
PUT /synonymtest/mytype/_mapping
{
    "mytype":{
        "properties":{
            "title":{
                "analyzer":"jt_cn",
                "term_vector":"with_positions_offsets",
                "boost":8,
                "store":true,
                "type":"text"
            }
        }
    }
}
```

* 添加数据

```
PUT /synonymtest/mytype/1
{
"title": "番茄"
}
PUT /synonymtest/mytype/2
{
"title": "西红柿"
}
PUT /synonymtest/mytype/3
{
"title": "我是西红柿"
}
PUT /synonymtest/mytype/4
{
"title": "我是番茄"
}
PUT /synonymtest/mytype/5
{
"title": "土豆"
}
PUT /synonymtest/mytype/6
{
"title": "aa"
}
```

* 搜索同义词

```
POST /synonymtest/mytype/_search?pretty
{
  "query": {
    "match_phrase": {
      "title": {
        "query": "西红柿",
        "analyzer": "jt_cn"
      }
    }
  },
"highlight": {
    "pre_tags": [
      "<tag1>",
      "<tag2>"
    ],
    "post_tags": [
      "</tag1>",
      "</tag2>"
    ],
    "fields": {
      "title": {}
    }
  }
}
```

### 维护动态同义词词库

* mysql创建同义词维护表

```
DROP TABLE IF EXISTS `synonym_config`;
CREATE TABLE `synonym_config` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `synonyms` varchar(128) DEFAULT NULL,
  `last_update_time` datetime DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=7 DEFAULT CHARSET=utf8;

INSERT INTO `synonym_config` VALUES ('1', '西红柿,番茄,圣女果', '2019-04-01 16:10:16');
INSERT INTO `synonym_config` VALUES ('2', '馄饨,抄手', '2019-04-02 16:10:40');
INSERT INTO `synonym_config` VALUES ('5', '小说,笑说,晓说', '2019-03-31 17:23:36');
INSERT INTO `synonym_config` VALUES ('6', '你好,利好', '2019-04-02 17:27:06');
```

* 创建一个SpringBoot应用，用来提供同义词数据接口供ElasticSearch定期检查导入

[dynamic-synonym-website](https://github.com/QigangZhong/dynamic-synonym-website)

* 编译打包动态同义词ES插件

[elasticsearch-analysis-dynamic-synonym](https://github.com/QigangZhong/elasticsearch-analysis-dynamic-synonym)

```
mvn clean package
```

在`target>release`目录下找到xxx.zip，放到`${es_home}/plugins/dynamic-synonym/`下解压，重启ES

```
DELETE synonymtest

PUT synonymtest
{
    "settings":{
        "index":{
            "analysis":{
                "filter":{
                    "local_synonym":{
                      "type":"synonym",
                        "synonyms_path":"synonym.txt",
                        "interval":60
                    },
                    "http_synonym":{
                        "type":"dynamic_synonym",
                        "synonyms_path":"http://192.168.237.129:8080/synonym",
                        "interval":60
                    }
                },
                "analyzer":{
                    "ik_max_word_syno":{
                        "type":"custom",
                        "tokenizer":"ik_max_word",
                        "filter":[
                            "http_synonym"
                        ]
                    },
                    "ik_smart_syno":{
                        "type":"custom",
                        "tokenizer":"ik_smart",
                        "filter":[
                            "http_synonym"
                        ]
                    }
                }
            }
        }
    }
}

POST synonymtest/product/_mapping
{
  "product":{
      "properties":{
          "id":{
              "type":"long"
          },
          "name":{
              "type":"text",
              "analyzer":"ik_max_word_syno",
              "search_analyzer":"ik_max_word_syno"
          }
      }
  }
}

GET synonymtest/product/_mapping

PUT /synonymtest/product/1
{
  "id":111,
  "name":"番茄炒蛋"
}

PUT /synonymtest/product/2
{
  "id":222,
  "name":"西红柿炒蛋"
}

GET /synonymtest/product/_search

GET synonymtest/product/_search
{
  "query": {
    "match_phrase": {
      "name": {
        "query": "西红柿", 
        "analyzer": "ik_max_word_syno"
      }
    }
  }
}

POST synonymtest/_analyze
{
  "analyzer": "ik_max_word_syno", 
  "text": "西红柿"
}
```

## 六、Java SDK

[Java API](https://www.elastic.co/guide/en/elasticsearch/client/java-api/current/index.html)

## 参考
[ElasticSearch权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)

[ElasticSearch-IK拓展自定义词库（2）：HTTP请求动态热词内容方式](https://my.oschina.net/jsonyang/blog/1782832)