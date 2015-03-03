---
layout: post
title: "搜索引擎——查询&评分&排序"
tagline: "搜索引擎——查询&评分&排序"
description: "搜索引擎——查询&评分&排序"
tags: [搜索引擎]
---
{% include JB/setup %}

##查询方式

##评分算法
*   TF：文档中词项出现的频率越高，那么文档或者域的得分越高
*   DF：文档频率(df(t))，表示出现t的所有文档的数目

由于DF的值往往比较大所以需要把它映射到一个较小的取值范围内，产生了idf(inverse document frequency,逆文档频率)

idf = log(N/df(t))
N文档数目

公式得出罕见词的idf会很高，而高频词的idf会比较低

那么得到了tf-idf(t,d) = tf(t,d) * idf(t)

针对查询Q得到了文档d的得分
score(Q，d) = （t属于Q求和）tf-idf(t,d)
##搜索质量评估指标