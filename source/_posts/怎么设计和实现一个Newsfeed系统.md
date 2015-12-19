title: 怎么设计和实现一个Newsfeed系统
date: 2015-04-01 09:14:52
categories: Python学习笔记
tags: [Newsfeed, celery, djcelery, redis, stream framework]
---


也是拖了好久，才开始写这篇关于Newsfeed系统设计与实现的介绍。今天已经把新产品需要的第一个版本的Newsfeed系统部署完了，所以对整个过程也有了清晰的了解。我希望能够通过我研究，设计，实现和部署这个系统的过程，来展示Facebook，Twitter和朋友圈的基本实现原理大概是什么样的。  
首先，想要实现类似Facebook的信息流的系统，如果使用最简单的脚本语言加Mysql的实现方式也是未尝不可，只是当一个系统的容量到一定级别之后，简单的技术就会变得难以满足系统的需求了。特别是在实际需求中，feed所在的表可能会经历超大并发的读写，而且由于feed的特殊性——例如Facebook，twitter，这些feed其实都只是一个简单的动态信息，而并非一个完整的对象信息，因此关系型数据库对于Newsfeed系统来说并不是最首选的方案。Feed的实际需求还有：

> * 1.用户需要频繁查询自己和自己好友产生的动态集合。
> * 2.用户需要频繁的查询自己或其他用户产生的动态的子集。
>
第一条需求远大于第二条，即刷朋友圈的次数远大于查看用户相册的次数。  

如果采用关系型数据库，那么第一个需求的实现应该是先从好友表中获取该用户的好友ID，然后通过好友ID在这个表中查询所有的数据，并通过时间排序分页返回之类，这实际上在数据库操作中遍历了大量其他无用的信息，而且当表的容量增大，如果不采用高效的分表或索引，这个操作的代价将会更大。而第二个需求，也是存在同样的问题，当1000个用户分别查看1个用户的timeline时，其中有999次数据库操作是浪费的，因此在Newsfeed的系统中采用Nosql的数据库是更加明智的方案。  
在开始，选择redis还是cassandra着实让我犹豫了一阵，这篇文章彻底打开了我的思路：

[http://www.csdn.net/article/2013-11-07/2817430-design-decisions-for-scaling-your-high-traffic-feeds](http://www.csdn.net/article/2013-11-07/2817430-design-decisions-for-scaling-your-high-traffic-feeds)

这是一个国外时尚网站和instagram的发展经验。

Fashiolista和Instagram都经历了从Redis开始，然后转战Cassandra的过程。笔者之所以会推荐从Redis开始是因为Redis更容易启动和维持。  
然而Redis存在一定的限制，所有的数据需要被存储在RAM中，成本很高。另外，Redis不支持分片，这意味着你必须在结点间分片（Twemproxy 是一个不错的选择），这种分片很容易，但是添加和删除节点时的数据处理很复杂。当然你可以将Redis作为缓存，然后重新访问数据库，来克服这个限制。但 是随着访问数据库的成本越来越高，笔者建议还是用Cassandra代替Redis。  
Redis是一种部署比较容易的Nosql数据库，它基于Key-value存储，数据全部存在内存中，访问速度和并发量着实很让人吃惊。对于各种软件中的Notification系统和这种没到Facebook,Twitter那种量级的Newsfeed系统，redis应该是性价比最高的一种数据库解决办法了：它的集群配置比较方便，配置文件比较简单（即使这样还是费了很大的劲，配置文件的详解还是有的，所以配置起来还算方便：[http://blog.csdn.net/neubuffer/article/details/17003909](http://blog.csdn.net/neubuffer/article/details/17003909)
），并发量不用怎么处理就可以轻松过10w+（[http://my.oschina.net/pblack/blog/102394](http://my.oschina.net/pblack/blog/102394)）

```
$ numactl -C 6 ./redis-benchmark -q -n 100000 -d 256 
PING (inline): 145137.88 requests per second 
PING: 144717.80 requests per second MSET (10 keys): 65487.89 requests per second 
SET: 142653.36 requests per second 
GET: 142450.14 requests per second 
INCR: 143061.52 requests per second 
LPUSH: 144092.22 requests per second 
LPOP: 142247.52 requests per second 
SADD: 144717.80 requests per second 
SPOP: 143678.17 requests per second 
LPUSH (again, in order to bench LRANGE): 143061.52 requests per second 
LRANGE (first 100 elements): 29577.05 requests per second 
LRANGE (first 300 elements): 10431.88 requests per second 
LRANGE (first 450 elements): 7010.66 requests per second 
LRANGE (first 600 elements): 5296.61 requests per second
```

存储的问题解决了，接下来就是主体的服务了。目前最主流的开源feed系统应该就是上面引用的文章所说的feedly，也就是现在改名为stream framework的一个开源项目，其源代码在github上面可以被搜到。目前开源的版本只有python的，其他语言的如果要实现必须去官网购买付费的服务。

* 非开源的：http://getstream.io/get_started/
* 开源的wiki地址：https://stream-framework.readthedocs.org/en/latest/installation.html

直接用pip安装方式就可以安装好对python的支持。

根据框架要求安装好几个python的库的支持之后，就可以参照提供的这个示例来设计整个系统了  
[https://github.com/tbarbugli/stream_framework_example](https://github.com/tbarbugli/stream_framework_example)  
这个框架的原理是使用框架本身提供的Activity类和它的结构，用户根据需求自定义verb，即动态的动作（不知道这么翻译行不行），然后根据使用的数据库形式派生自定义的feed类，确定自定义feed的格式和单个feed的数据上限。这个框架的核心在于Manager类，用户需要继承出一个子类，来实现基本的feed操作。例如添加，删除，分发。（修改feed貌似还不支持）  
每个feed在数据库中的存在形式即一个key，对应的数据格式是sorted set，这是一个有序集合，每一个元素都有一个排序用的score，而这个score在这就是时间戳。我在自己设计实现相册的时候，也使用了这种设计思想。每一个有序集合里，只存着该feed所拥有的动态的activityId，而动态的基本信息会经过哈希后存在全局的10个hashes中。  
根据上面提出的两个需求，每个用户就需要有两个feed，一个是每个用户自己的feed，也称为每个用户的timeline，一个是每个用户关注的用户的动态集合，即每个用户的feed，这个feed的维护是通过每次该用户关注的用户发布动态时的分发来进行的。当然，后来加上的相册不在这个框架提供的范围内，我给每个用户又添加了一个存储所有image的key，也是存储每个image唯一的id，再将具体的image信息存储在全局的hashes中。
按照以上设计实现出来的系统中，当一个用户想要查看自己的feed，他就只需要直接查询自己的feed的这个key中的信息，当一个用户想要查看自己或某人的timeline的时候，他也只需要查询这个timeline中的信息即可，这些操作都是简单地GET，而不会像关系型数据库需要广大的遍历。当然，这个设计的关键之处就在于如何维护这些key中的内容。  
Redis的官网有详细的博客和操作介绍：[http://redis.io](http://redis.io)

说完了存储，就是最关键的分发。Stream framework的关键就在于利用celery异步队列处理机制实现了动态的分发。它默认必须安装django-celery来支持异步处理分发。  
Django-celery是celery在django的一个支持版本。它是独立于django的一个异步队列处理框架，简单地说就是他有一个队列，有N个worker（由用户自定义）等待执行队列中的task和一个或多个django app，也就是task的发送方贯穿而成。因为是django的版本，所以它的task实际上都是在django中预先编译好的。在stream framework中，用来实现分发的task就包括了几个具体的task，如果要自己写一些定向分发动态或notification，djcelery是必须要熟悉和掌握的。  
我们可能想要实现类似微信朋友圈推广那种下沉式feed，就要通过定向分发将这个动态分发到指定用户的feed中去，而这个分发的工作量是极其大的，如果要给前端的管理平台等提供一个发布的接口，我们必须要用djcelery在后台运行的一个worker来不断的执行这个task，然后先将http请求返回给用户，而这个task的执行结果可以通过设置celery的backend存储目标来输出。  
具体的celery设置可以参考：
[http://docs.celeryproject.org/en/latest/django/](http://docs.celeryproject.org/en/latest/django/)