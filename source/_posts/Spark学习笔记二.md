---
title: Spark学习笔记二
date: 2016-05-12 14:35:05
tags: Spark
categories: Spark
---

本例子通过Spark sql 链接其他数据库。对数据库的信息进行操作。过滤。


### 首先是main 方法，创建SparkSession实例
```
def main(args: Array[String]) {

    val sparkConf = new SparkConf().setAppName("SparkSQLDemo")
    sparkConf.setMaster("local")
    val spark = SparkSession.builder().appName("SparkSQLDemo").config(sparkConf).getOrCreate()
    //创建数据库连接，从数据库查询数据，并存储本地。
    runJDBCDataSource(spark)
    //从json文件中加载数据，进行搜索
    loadDataSourceFromeJson(spark)
    //从parquet文件中加载数据，进行搜索
    loadDataSourceFromeParquet(spark)
    从RDD中加载数据
    runFromRDD(spark)
    spark.stop()
  }
```

### runJDBCDataSource

链接数据库，操作数据，首先配置数据库连接信息。连接数据库，进行搜索，然后将数据输出到本地文件（此处我输出的文件路径相同。所以每次输出以后记得将文件改名，并将文件夹删除，要不然会报错文件路径已存在呢）
```
private def runJDBCDataSource(spark: SparkSession): Unit = {
    val jdbcDF = spark.read
      .format("jdbc")
      .option("url", "jdbc:mysql://localhost:3306/msm?user=root&password=admin")
      //必须写表名
      .option("dbtable", "sec_user")
      .load()
    //查询数据库中的id, name, telephone三个列并以parquet（列存储）的方式存储在src/main/resources/sec_users路径下（存储后记得将名字改为user.parquet）
    //jdbcDF.select("id", "name", "telephone").write.format("parquet").save("src/main/resources/sec_users")
    //查询数据库中的username, name, telephone三个列并以parquet（列存储）的方式存储在src/main/resources/sec_users路径下存储后记得将名字改为user.json）
    //jdbcDF.select("username", "name", "telephone").write.format("json").save("src/main/resources/sec_users")

    //存储成为一张虚表user_abel
    jdbcDF.select("username", "name", "telephone").write.mode("overwrite").saveAsTable("user_abel")
    val jdbcSQl = spark.sql("select * from user_abel where name like '王%' ")
    jdbcSQl.show()

    jdbcSQl.write.format("json").save("./out/resulted")
  }
```

### loadDataSourceFromeJson

从runJDBCDataSource产生的json文件中读取数据进行处理并将结果存储。
```
private def loadDataSourceFromeJson(spark: SparkSession): Unit = {
   //从runJDBCDataSource产生的user.json中读取数据
    val jsonDF = spark.read.json("src/main/resources/user.json")
    //输出结构
    jsonDF.printSchema()
    //创建临时视图
    jsonDF.createOrReplaceTempView("user")
    //从临时视图进行查询
    val namesDF = spark.sql("SELECT name FROM user WHERE name like '王%'")
    import spark.implicits._
    //操作查询结果，在每个查询结果前加"Name: " 但使用该方法必须导入spark.implicits._
    namesDF.map(attributes => "Name: " + attributes(0)).show()
//将结果以json的形式写入到./out/resultedJSON 路径下 jsonDF.select("name").write.format("json").save("./out/resultedJSON")
  }
```

### loadDataSourceFromeParquet
从runJDBCDataSource产生的parquet(列式存储)文件中读取数据进行处理并将结果存储。
```
private def loadDataSourceFromeParquet(spark: SparkSession): Unit = {
    //从runJDBCDataSource产生的user.json中读取数据
    val parquetDF = spark.read.load("src/main/resources/user.parquet")
     //创建临时视图
    parquetDF.createOrReplaceTempView("user")
    val namesDF = spark.sql("SELECT name FROM user WHERE id > 1 ")
    namesDF.show()
//将结果以parquet的形式写入到./out/resultedParquet 路径下
 parquetDF.select("name").write.format("parquet").save("./out/resultedParquet")
  }
```

### runFromRDD
从RDD中读取数据进行搜索处理。
```
private def runFromRDD(spark: SparkSession): Unit = {
     //创建一个json形式的RDD
    val otherPeopleRDD = spark.sparkContext.makeRDD(
      """{"name":"Yin","address":{"city":"Columbus","state":"Ohio"}}""" :: Nil)
     //从RDD中读取数据
    val otherPeople = spark.read.json(otherPeopleRDD)
    otherPeople.show()
  }

 ```
### 完整代码如下：

 此处我输出的文件路径相同。所以每次输出以后记得将文件改名(gai为user.json和user.parquet)，并将文件夹删除，要不然会报错文件路径已存在呢.
 ```
 import org.apache.spark.SparkConf
 import org.apache.spark.sql.SparkSession

 /**
   * Created by yangyibo on 16/11/24.
   */
 object SparkSQLDemo {


   def main(args: Array[String]) {

     val sparkConf = new SparkConf().setAppName("SparkSQLDemo")
     sparkConf.setMaster("local")
     val spark = SparkSession.builder().appName("SparkSQLDemo").config(sparkConf).getOrCreate()
     runJDBCDataSource(spark)
     loadDataSourceFromeJson(spark)
     loadDataSourceFromeParquet(spark)
     runFromRDD(spark)
     spark.stop()
   }

   private def runJDBCDataSource(spark: SparkSession): Unit = {
     val jdbcDF = spark.read
       .format("jdbc")
       .option("url", "jdbc:mysql://localhost:3306/msm?user=root&password=admin")
       .option("dbtable", "sec_user") //必须写表名
       .load()
     //jdbcDF.select("id", "name", "telephone").write.format("parquet").save("src/main/resources/sec_users")
     //jdbcDF.select("username", "name", "telephone").write.format("json").save("src/main/resources/sec_users")

     //存储成为一张虚表user_abel
     jdbcDF.select("username", "name", "telephone").write.mode("overwrite").saveAsTable("user_abel")
     val jdbcSQl = spark.sql("select * from user_abel where name like '王%' ")
     jdbcSQl.show()
     jdbcSQl.write.format("json").save("./out/resulted")
   }

   private def loadDataSourceFromeJson(spark: SparkSession): Unit = {
     //load 方法是加载parquet 列式存储的数据
     // val jsonDF=spark.read.load("src/main/resources/sec_users/user.json")
     val jsonDF = spark.read.json("src/main/resources/user.json")

     jsonDF.printSchema()
     //创建临时视图
     jsonDF.createOrReplaceTempView("user")
     val namesDF = spark.sql("SELECT name FROM user WHERE name like '王%'")
     import spark.implicits._
     namesDF.map(attributes => "Name: " + attributes(0)).show()
     jsonDF.select("name").write.format("json").save("./out/resultedJSON")
   }

   private def loadDataSourceFromeParquet(spark: SparkSession): Unit = {

     val parquetDF = spark.read.load("src/main/resources/user.parquet")
     parquetDF.createOrReplaceTempView("user")
     val namesDF = spark.sql("SELECT name FROM user WHERE id > 1 ")
     namesDF.show()

     parquetDF.select("name").write.format("parquet").save("./out/resultedParquet")
   }

   private def runFromRDD(spark: SparkSession): Unit = {
     val otherPeopleRDD = spark.sparkContext.makeRDD(
       """{"name":"Yin","address":{"city":"Columbus","state":"Ohio"}}""" :: Nil)
     val otherPeople = spark.read.json(otherPeopleRDD)
     otherPeople.show()
   }

 }
 ```
