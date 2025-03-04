---
layout: post
title: "java利用 redis sorted set 对数据进行排序"
date: 2018-09-10 10:33:30
categories: 日常工作
---

> 最近项目中遇到了需要对数据排序的需求，之前都是在持久化层或者是利用 java 代码进行的排序。而最近在研究 redis ，之前看到过有人利用 redis sorted set 对数据进行排序，就顺便进行了尝试，发现效果不错，所以记录下 。

<!-- more -->

## Redis 简介

- redis 是开源的 key-value 内存数据库，同时支持可选的持久化方案。通俗点说，就是十分适合用来做缓存的 nosql（因为跑在内存里）。
- 相比较 memocached ，支持更多的数据类型（字符串(String), 哈希(Map), 列表(list), 集合(sets) 和 有序集合(sorted sets)），使用上更加方便。

## Springboot 项目引入redis

- 首先就是需要在程序运行的机器上安装 redis，安装教程可以参照[这里](http://www.runoob.com/redis/redis-install.html)。

- 利用 idea 创建项目，选择 spring initializer，然后再引入依赖的步骤里选择 `NoSQL->Redis` ，打上勾即可，项目初始化完成后便引入了 redis 所需的依赖。当然，也可以使用 spring-boot-starter 引入 spring-data-redis  ，同样可以引入所需的依赖，我这里没做尝试。

## 实战代码

- Sorted set 中，每一项元素都又两部分组成：score 与 string 。其中，score 可以通俗的理解为权值或者评分，用于排序；string用来存储对应的数据，更详细的说明可以参照[官网文档](https://redis.io/topics/data-types-intro)。

- 所以接下来的工作就比较简单了，获取对象，转换为 string，再获取排序时需要的数值作为 score，然后先存储到 sorted set 中，再依次取出即可。

- jedis 是 java 下一个易用的 redis client，官方文档在[这里](https://github.com/xetorthio/jedis)。

- 核心伪代码如下：

  ```
  for(int i = 0;i < array.length;i++){
  	Double score = array[i].getScore();//获取score
  	String member = array[i].serialize();//对对象进行序列化，方便存储
   	jedis.zadd("sort_set", score, member);//放入sorted set   
  }
  Set<Tuple> set = jedis.zrevrangeWithScores(key, 0, -1);//获取排序后set
  Map result = new LinkedHashMap<Double, Object>();
  int rank = 0;
  for (Tuple tuple : set) {
  //遍历存储
      Map tmp = new HashMap<String,Object>(2);
      tmp.put("score", tuple.getScore());
      tmp.put("member",tuple.getElement());
      result.put(++rank, tmp);
  }
  //清除sorted set 方便下次排序
  jedis.zremrangeByRank(BizConstants.COMPANY_SORT_SETS, 0, -1);
  jedis.close();
  ```

- 代码中 result 即为排序后的结果，当然，根据需求也可以使用别的集合类型。