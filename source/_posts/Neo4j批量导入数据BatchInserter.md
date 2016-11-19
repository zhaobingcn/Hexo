---
title: Neo4j批量导入数据BatchInserter
date: 2016-11-12 17:05:03
tags:
	- Neoej
	- BatchInserter
	- Neo4j批量导入
categories: Neo4j
---
## BatcherInserter的使用

当需要向Neo4j大批量导入数据的时候，使用Neo4j核心API，Cypher和LOAD CSV等导入方式速度会很慢，因为这些导入方式会产生很多IO操作。 [BatchInserter](http://neo4j.com/docs/java-reference/current/javadocs/org/neo4j/unsafe/batchinsert/BatchInserter.html)采用批量导入的方式，减少IO操作以提升导入速度，但是牺牲了实时性和事物(Transcations)支持，需要关闭数据库才可以。

``` java
        File dbPath = new File("D:/Neo4jDB/importTest");
        BatchInserter inserter = BatchInserters.inserter(new File(dbPath));

```
