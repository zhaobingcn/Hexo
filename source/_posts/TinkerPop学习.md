---
title: TinkerPop学习
date: 2017-07-17 15:53:22
tags: 
    - TinkerPop
    - Graph
    - Janusgraph
---

# TinkerPop简介
   TinkerPop由一系列的共同操作组件组成。他最基本的API定义了如何实现一个图，节点，边等等操作。最简单的实现图操作的方法是实现核心api。用户可以使用Gremlin语言进行图形操作。而且还提供了更高级的图形操作，实现实时查询，优化表现性能（索引机制等）。如果是一个图处理系统（OLAP），可以实现GraphComputer API。这些API定义了消息是如何在不同的工作节点之上进行通信的。同样的Gremlin遍历可以实现在图数据库(OLTP)和图处理(OLAP)系统中。Gremlin是一个基于图的DSL语言。
   ![](/images/tinkerpop.png)
    
## 一些图的指标
   **Graph Features**  
   graph.features(),表示当前图支持的性质。
   **Vertex Properties**
   1.一个属性支持多个值，多个值用数组表示
   2.属性上面还支持key/value对表示的属性
    ``` java
        gremlin> g.V().as('a').
                       properties('location').as('b').
                       hasNot('endTime').as('c').
                       select('a','b','c').by('name').by(value).by('startTime') // determine the current location of each person
        ==>[a:marko,b:santa fe,c:2005]
        ==>[a:stephen,b:purcellville,c:2006]
        ==>[a:matthias,b:seattle,c:2014]
        ==>[a:daniel,b:aachen,c:2009]
        gremlin> g.V().has('name','gremlin').inE('uses').
                       order().by('skill',incr).as('a').
                       outV().as('b').
                       select('a','b').by('skill').by('name') // rank the users of gremlin by their skill level
        ==>[a:3,b:matthias]
        ==>[a:4,b:marko]
        ==>[a:5,b:stephen]
        ==>[a:5,b:daniel]
    ```
   注意as， select， by的使用方法。
   **Graph Variables**
   存储了关于数据库的元数据信息
   **Graph Transactions**
   
   
    
    
    