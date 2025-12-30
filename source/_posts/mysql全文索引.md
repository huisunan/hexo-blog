---
title: mysql全文索引
date: 2022-03-12 10:44
tags: mysql
categories: 
---

<!--more-->

# 全文索引

## 添加全文索引

在5.7.6版本,MySQL内置了ngram全文解析器,用来支持亚洲语种的分词.  
ALTER TABLE \[your\_table\] ADD FULLTEXT INDEX \[index\_name\] \(\[column\_name\]\) WITH PARSER ngram;

## 搜索

全文索引的搜索使用关键词 MATCH和AGAINST  
SELECT \* FROM \[your\_table\] WHERE MATCH \(\[column\_name\]\) AGAINST \(\[key\_word\]\);