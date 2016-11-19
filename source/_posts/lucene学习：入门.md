---
title: Lucene学习：入门
date: 2016-11-19 22:16:35
tags:
	- Lucene
categories: lucene
---

## 全文检索简介

全文检索是将存储文件的全部内容都进行检索，也许是一本书，也许是一篇文章。他可以根据需要获得全文中的有关文章、节、段、句、词等信息，形象的表示就是给整本书的每个字词添加一个标签，以进行各种统计分析。Lucene全文检索就是基于这种应用的，Lucene采用“倒排索引”建立关键词与文件的相关映射，如下如所示：

![](/images/lucene.png)

## Lucene的核心jar包

* **lucene-core-5.5.0.jar**: 其中包括了常用的文档，索引，搜索，存储等相关核心代码；
* **lucene-analyzers-common-5.5.0.jar**: 包括了各种词法分析器，用于对文件内容关键词进行切分，提取（但是只包含英文的）；
* **lucene-queryparser-5.5.0.jar**: 提取了搜索相关的代码，用于各种搜索，比如模糊搜索，范围搜索，等等

## 主要开发包说明
* **org.apache.lucene.analysis**：语言分析器，主要用于分词；
* **org.apache.lucene.document**：索引文档的管理；
* **org.apache.lucene.index**：索引管理，如增、删、改；
* **org.apache.lucene.queryparser**：查询分析；
* **org.apache.lucene.search**：检索管理；
* **org.apache.lucene.store**：数据存储管理；
* **org.apache.lucene.util**：工具包

## 索引操作的核心类

* **Directory**: 代表索引文档的存储位置，这是一个抽象类，有两个子类FSDirectory和RAMDriectory两个主要子类。前者将索引写入文件系统，后者将索引写入内存；
* **Analyzer**: 建立索引时所使用的分析器，在一个文档被索引之前对文档内容进行分词处理，主要子类有StandardAnalyzer(对汉字的分词是一个汉字一个词)。要想使用针对汉字的分析器，需要额外引入IKAnalyzer，PaodingAnalyzer等；
* **IndexWriterConfig**: 操作索引库配置信息；
* **IndexWriter**: 建立索引的核心类，用来操作索引(增，删，改)；
* **Document**: 代表一个索引文档，这里的文档可以指一篇文章，一个HTML页面等。一个Document可以由多个Field对象组成。可以把一个Document想成数据库中的一个记录，而每个Field就是记录的一个字段；
* **Field**: 代表索引文档中存储的数据，用来描述文档的某个属性，比如一封电子邮件的内容和标题可以用两个Field对象分别描述

## 写入索引

### 用maven导入依赖的jar包

```java
	<dependency>
	  <groupId>org.apache.lucene</groupId>
	  <artifactId>lucene-core</artifactId>
	  <version>5.5.0</version>
	</dependency>
	<dependency>
	  <groupId>org.apache.lucene</groupId>
	  <artifactId>lucene-analyzers-common</artifactId>
	  <version>5.5.0</version>
	</dependency>
	<dependency>
	  <groupId>org.apache.lucene</groupId>
	  <artifactId>lucene-queryparser</artifactId>
	  <version>5.5.0</version>
	</dependency>
```

### 写入索引

```java
	//选择语言分析器
	Analyzer analyzer = new StandardAnalyzer();
	//索引文档的存储位置
	Directory directory = FSDirectory.open(Paths.get("./index"));
	//配置索引库信息
	IndexWriterConfig indexWriterConfig = new IndexWriterConfig(analyzer);
	//创建IndexWriter，用来进行索引文件的写入
	IndexWriter indexWriter = new IndexWriter(directory, indexWriterConfig);
    //准备写入Document的文档
	String[] texts = new String[]{
	        "Apache Lucene is a high-performance",
	        "full-featured text search engine library written entirely in Java",
	        "It is a technology suitable for nearly any application that requires full-text search.",
	        "Apache Lucene is an open source project available for free download. ",
	        "Please use the links on the right to access Lucene."
	};
	//建立索引文档
	for(String text: texts){
	    Document document = new Document();
	    document.add(new TextField("info", text, Field.Store.YES));
	    indexWriter.addDocument(document);
	}
	indexWriter.close();
	directory.close();
```

## 查询索引的核心类

* **IndexReader**: 读取索引的工具类，常见的子类有DirectoryReader；
* **IndexSeracher**: 查询索引的核心类；
* **QueryPaerser**: 查询分析器，表示从哪里查用哪个分析器查；
* **Query**: 代表一次查询；
* **TopDocs**: 封装了匹配情况,旧版本用的是Hits，新版本已经弃用；
* **ScoreDocs**: 匹配情况的数据，里面封装了索引文档得分和索引ID

## 查询索引

```java
	Analyzer analyzer = new StandardAnalyzer();
	Directory directory = FSDirectory.open(Paths.get("./index"));
	//建立IndexReader
	IndexReader reader = DirectoryReader.open(directory);
	IndexSearcher indexSearcher = new IndexSearcher(reader);
	QueryParser queryParser = new QueryParser("info", analyzer);
	Query query = queryParser.parse("lucene");
	TopDocs topDocs = indexSearcher.search(query, 1000);
	System.out.println("总共匹配多少个：" + topDocs.totalHits);
	ScoreDoc[] hits = topDocs.scoreDocs;
	System.out.println("总共多少条数据：" + hits.length);
	for(ScoreDoc hit: hits){
	    System.out.println("匹配的分：" + hit.score);
	    System.out.println("文档索引id：" + hit.doc);
	    Document document = indexSearcher.doc(hit.doc);
	    System.out.println(document.get("info"));
	}
	reader.close();
	directory.close();
```
以上就是两个简单的全文索引建立和查找的例子，但是已经涵盖了主要的思想。





