---
title: spark学习笔记一
date: 2016-05-10 12:23:57
tags: Spark
categories: Spark
---
本例子是我初尝 Spark 的sparkStreaming官方小例子修改的。

我的思路是使用jdbc 链接数据库，然后查询数据库，将查询结果生成一个RDD ，放入RDD queue，然后每次取出rdd 进行计算和过滤处理。

## 1.sparkStreamingDemo
由于这个demo需要spark 和jdbc 的依赖包。在pom.xml文件中如下（关于新建maven 的spark工程请参考idea 构建maven 管理的spark项目）

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.us.demo</groupId>
    <artifactId>mySpark</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <spark.version>2.0.2</spark.version>
        <scala.version>2.11</scala.version>
    </properties>


    <dependencies>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_${scala.version}</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-streaming_${scala.version}</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_${scala.version}</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-hive_${scala.version}</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-mllib_${scala.version}</artifactId>
            <version>${spark.version}</version>
        </dependency>

        <!-- JDBC-->

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.12</version>
        </dependency>

    </dependencies>

    <build>
        <plugins>

            <plugin>
                <groupId>org.scala-tools</groupId>
                <artifactId>maven-scala-plugin</artifactId>
                <version>2.15.2</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>testCompile</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>

            <plugin>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.6.0</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>2.19</version>
                <configuration>
                    <skip>true</skip>
                </configuration>
            </plugin>

        </plugins>
    </build>

</project>
```
SparkStreamingDemo demo 的代码如下，我会尽量逐行增加注释：
```
import java.sql.{DriverManager, ResultSet}

import scala.collection.mutable.Queue
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.rdd.RDD
import org.apache.spark.streaming.{Seconds, StreamingContext}

/**
  * Created by yangyibo on 16/11/23.
  */
object SparkStreamingDemo {
  def main(args: Array[String]) {

    //创建spark实例
    val sparkConf = new SparkConf().setAppName("QueueStream")
    sparkConf.setMaster("local")
    // 创建sparkStreamingContext ，Seconds是多久去Rdd中取一次数据。
    val ssc = new StreamingContext(sparkConf, Seconds(3))

    // Create the queue through which RDDs can be pushed to a QueueInputDStream
    var rddQueue = new Queue[RDD[String]]()
    // 从rdd队列中读取输入流
    val inputStream = ssc.queueStream(rddQueue)
    //将输入流中的每个元素（每个元素都是一个String）后面添加一个“a“字符，并返回一个新的rdd。
    val mappedStream = inputStream.map(x => (x + "a", 1))
    //reduceByKey(_ + _)对每个元素统计次数。map(x => (x._2,x._1))是将map的key和value 交换位置。后边是过滤次数超过1次的且String 相等于“testa“
    val reducedStream = mappedStream.reduceByKey(_ + _)
        .map(x => (x._2,x._1)).filter((x)=>x._1>1).filter((x)=>x._2.equals("testa"))
    reducedStream.print()
    //将每次计算的结果存储在./out/resulted处。
    reducedStream.saveAsTextFiles("./out/resulted")
    ssc.start()

    //从数据库中查出每个用户的姓名，返回的是一个String有序队列seq，因为生成RDD的对象必须是seq。
    val seq = conn()
    println(Seq)
     //将seq生成RDD然后放入Spark的Streaming的RDD队列，作为输入流。
    for (i <- 1 to 3) {

      rddQueue.synchronized {
        rddQueue += ssc.sparkContext.makeRDD(seq,10)
      }
      Thread.sleep(3000)
    }
    ssc.stop()
  }


//从数据库中取出每个用户的名字，是个String有序队列
  def conn(): Seq[String] = {
    val user = "root"
    val password = "admin"
    val host = "localhost"
    val database = "msm"
    val conn_str = "jdbc:mysql://" + host + ":3306/" + database + "?user=" + user + "&password=" + password
    //classOf[com.mysql.jdbc.Driver]
    Class.forName("com.mysql.jdbc.Driver").newInstance();
    val conn = DriverManager.getConnection(conn_str)
    var setName = Seq("")
    try {
      // Configure to be Read Only
      val statement = conn.createStatement(ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY)

      // Execute Query，查询用户表 sec_user 是我的用户表，有name属性。
      val rs = statement.executeQuery("select * from sec_user")
      // Iterate Over ResultSet

      while (rs.next) {
        // 返回行号
        // println(rs.getRow)
        val name = rs.getString("name")
        setName = setName :+ name
      }
      return setName
    }
    finally {
      conn.close
    }
  }
}
```

## 2.scala 链接mysql

```
import java.sql.{Connection, DriverManager, ResultSet}

/**
  * Created by yangyibo on 16/11/23.
  */
object DB {

  def main(args: Array[String]) {
    val user = "root"
    val password = "admin"
    val host = "localhost"
    val database = "msm"
    val conn_str = "jdbc:mysql://" + host + ":3306/" + database + "?user=" + user + "&password=" + password
    println(conn_str)
    val conn = connect(conn_str)
    val statement =conn.createStatement(ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY);
    // Execute Query
    val rs = statement.executeQuery("select * from sec_user")
    // Iterate Over ResultSet
    while (rs.next) {
      // 返回行号
      // println(rs.getRow)
      val name = rs.getString("name")
      println(name)
    }
    closeConn(conn)
  }

  def connect(conn_str: String): Connection = {
    //classOf[com.mysql.jdbc.Driver]
    Class.forName("com.mysql.jdbc.Driver").newInstance();
    return  DriverManager.getConnection(conn_str)
  }

  def closeConn(conn:Connection): Unit ={
    conn.close()
  }

}
```

### scala 链接MySQL所需依赖

```
        <!-- JDBC-->

        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.12</version>
        </dependency>
```