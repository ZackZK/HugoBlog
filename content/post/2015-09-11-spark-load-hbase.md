---
date: 2015-09-11T00:00:00Z
description: 介绍spark的读取hbase及在集群环境下的配置
modified: 2015-09-12
tags:
- spark
- hbase
- spark cluster
- spark configuration
title: spark读取hbase内容及在集群环境下的配置
---

## spark读取hbase ##
对于读取hbase，spark提供了newAPIHadoopRDD接口可以很方便的读取hbase内容。下面是一个具体的例子：

首先，加入下面依赖：

{{< highlight xml >}}
    <dependencies>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.10</artifactId>
            <version>1.4.1</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-common</artifactId>
            <version>1.1.2</version>
        </dependency>

        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-client</artifactId>
            <version>1.1.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-server</artifactId>
            <version>1.1.2</version>
        </dependency>
    </dependencies>
{{< / highlight >}}

下面的例子从hbase数据库中读取数据，并进行RDD操作，生成两两组合。

{{< highlight java >}}

import org.apache.hadoop.hbase.HBaseConfiguration;
import org.apache.hadoop.hbase.util.Base64;
import org.apache.hadoop.hbase.util.Bytes;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.function.Function;
import org.apache.spark.api.java.function.Function2;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.client.Scan;
import org.apache.hadoop.hbase.client.Result;
import org.apache.hadoop.hbase.io.ImmutableBytesWritable;
import org.apache.hadoop.hbase.mapreduce.TableInputFormat;
import org.apache.spark.api.java.JavaPairRDD;

import org.apache.hadoop.hbase.protobuf.ProtobufUtil;
import org.apache.hadoop.hbase.protobuf.generated.ClientProtos;
import org.apache.spark.api.java.function.PairFunction;
import scala.Tuple10;
import scala.Tuple2;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

/*
* hbase content:
* hbase(main):003:0> scan 'scores'
ROW                             COLUMN+CELL
 Jim                            column=course:art, timestamp=1441549968185, value=80
 Jim                            column=course:math, timestamp=1441549968167, value=89
 Jim                            column=grade:, timestamp=1441549968142, value=4
 Tom                            column=course:art, timestamp=1441549912222, value=88
 Tom                            column=course:math, timestamp=1441549842348, value=97
 Tom                            column=grade:, timestamp=1441549820516, value=5
2 row(s) in 0.1330 seconds
*/
public class spark_hbase_main {
    private static String appName = "Hello";

    public static void main(String[] args) {
        SparkConf sparkConf = new SparkConf().setAppName(appName);
        JavaSparkContext jsc = new JavaSparkContext(sparkConf);

        Configuration conf = HBaseConfiguration.create();

        Scan scan = new Scan();
        scan.addFamily(Bytes.toBytes("course"));
        scan.addColumn(Bytes.toBytes("course"), Bytes.toBytes("art"));
        scan.addColumn(Bytes.toBytes("course"), Bytes.toBytes("math"));

        String scanToString = "";
        try {
            ClientProtos.Scan proto = ProtobufUtil.toScan(scan);
            scanToString = Base64.encodeBytes(proto.toByteArray());
        } catch (IOException io) {
            System.out.println(io);
        }

        for (int i = 0; i < 2; i++) {
            try {
                String tableName = "scores";
                conf.set(TableInputFormat.INPUT_TABLE, tableName);
                conf.set(TableInputFormat.SCAN, scanToString);

                //获得hbase查询结果Result
                JavaPairRDD<ImmutableBytesWritable, Result> hBaseRDD = jsc.newAPIHadoopRDD(conf,
                        TableInputFormat.class, ImmutableBytesWritable.class,
                        Result.class);

                /* 生成类似 [(Jim, 80, 89), (Tom, 88, 97)] 的RDD */
                JavaPairRDD<String, List<Integer>> art_scores = hBaseRDD.mapToPair(
                        new PairFunction<Tuple2<ImmutableBytesWritable, Result>, String, List<Integer>>() {
                            @Override
                            public Tuple2<String, List<Integer>> call(Tuple2<ImmutableBytesWritable, Result> results) {

                                List<Integer> list = new ArrayList<Integer>();

                                byte[] art_score = results._2().getValue(Bytes.toBytes("course"), Bytes.toBytes("art"));
                                byte[] math_score = results._2().getValue(Bytes.toBytes("course"), Bytes.toBytes("math"));

                                /* 注意： Hbase里存的数据以Byte Array形式存储， 需要使用Integer.parseInt(Bytes.toString(art_score))将数据内容转化为整型
                                * Integer.parseInt(price.toString()) 会得到错误答案 */
                                list.add(Integer.parseInt(Bytes.toString(art_score)));
                                list.add(Integer.parseInt(Bytes.toString(math_score)));

                                return new Tuple2<String, List<Integer>>(Bytes.toString(results._1().get()), list);
                            }
                        }
                );

                /* 如果是使用Java8， 可以简化成下面的形式: 
                JavaPairRDD<Integer, Double> stock_price_pair = hBaseRDD.mapToPair(
                    (results) -> {
                          List<Integer> list = new ArrayList<Integer>();

                          byte[] art_score = results._2().getValue(Bytes.toBytes("course"), Bytes.toBytes("art"));
                          byte[] math_score = results._2().getValue(Bytes.toBytes("course"), Bytes.toBytes("math"));

                          list.add(Integer.parseInt(Bytes.toString(art_score)));
                          list.add(Integer.parseInt(Bytes.toString(math_score)));

                          return new Tuple2<String, List<Integer>>(Bytes.toString(results._1().get()), list);
                    }
                )
                */

                /* 笛卡尔乘积，生成 [((Jim, 80, 89), (Tom, 88, 97)), ((Tom, 88, 97), (Jim, 80, 89)), ((Jim, 80, 89), (Jim, 80, 89)),
                ((Tom, 88, 97), (Tom, 88, 97))] 的RDD */] */
                JavaPairRDD<Tuple2<String, List<Integer>>, Tuple2<String, List<Integer>>> cart = art_scores.cartesian(art_scores);

                /* 利用row key的大小关系去除重复的组合关系， 生成 [((Jim, 80, 89), (Tom, 88, 97))] */
                JavaPairRDD<Tuple2<String, List<Integer>>, Tuple2<String, List<Integer>>> cart2 = cart.filter(
                        new Function<Tuple2<Tuple2<String, List<Integer>>, Tuple2<String, List<Integer>>>, Boolean>() {
                            public Boolean call(Tuple2<Tuple2<String, List<Integer>>, Tuple2<String, List<Integer>>> tuple2Tuple2Tuple2) throws Exception {

                                return tuple2Tuple2Tuple2._1()._1().compareTo(tuple2Tuple2Tuple2._2()._1()) < 0;
                            }
                        }
                );

                /* 得到最终结果 [((Jim, 80, 89), (Tom, 88, 97))] */
                cart_all = cart2.collect();

            } catch (Exception e) {
                System.out.println(e);
            }
        }
    }
}
{{< / highlight >}}

编译运行时，Hbase的一些库可能会找不到，一种办法是在命令行中加入Hbase相应的jar包，比如：

{{< highlight bash >}}
spark-submit --master spark://127.0.0.0:7078 --class spark_hbase_main --jars ${HBASE_HOME}/lib/hbase-common-1.0.1.1.jar,${HBASE_HOME}/lib/hbase-client-1.0.1.1.jar,${HBASE_HOME}/lib/guava-12.0.1.jar,${HBASE_HOME}/lib/hbase-protocol-1.0.1.1.jar,${HBASE_HOME}/lib/hbase-server-1.0.1.1.jar,${HBASE_HOME}/lib/htrace-core-3.1.0-incubating.jar spark-hbase-1.0.jar
{{< / highlight >}}

如果缺少相应的Hbase的jar包，比如缺少hbase-common-1.0.1.1.jar会出现下面的错误：

{{< highlight bash >}}
Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/hadoop/hbase/protobuf/generated/ClientProtos$Scan
        at java.lang.Class.forName0(Native Method)
        at java.lang.Class.forName(Class.java:278)
        at org.apache.spark.deploy.SparkSubmit$.org$apache$spark$deploy$SparkSubmit$$runMain(SparkSubmit.scala:634)
        at org.apache.spark.deploy.SparkSubmit$.doRunMain$1(SparkSubmit.scala:170)
        at org.apache.spark.deploy.SparkSubmit$.submit(SparkSubmit.scala:193)
        at org.apache.spark.deploy.SparkSubmit$.main(SparkSubmit.scala:112)
        at org.apache.spark.deploy.SparkSubmit.main(SparkSubmit.scala)
Caused by: java.lang.ClassNotFoundException: org.apache.hadoop.hbase.protobuf.generated.ClientProtos$Scan
        at java.net.URLClassLoader$1.run(URLClassLoader.java:366)
        at java.net.URLClassLoader$1.run(URLClassLoader.java:355)
        at java.security.AccessController.doPrivileged(Native Method)
        at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:425)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:358)
        ... 7 more
{{< / highlight >}}

缺少hbase-protocol-1.0.1.1.jar会报如下错误：

{{< highlight bash >}}
Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/hadoop/hbase/protobuf/generated/ClientProtos$Scan
        at java.lang.Class.forName0(Native Method)
        at java.lang.Class.forName(Class.java:278)
        at org.apache.spark.deploy.SparkSubmit$.org$apache$spark$deploy$SparkSubmit$$runMain(SparkSubmit.scala:634)
        at org.apache.spark.deploy.SparkSubmit$.doRunMain$1(SparkSubmit.scala:170)
        at org.apache.spark.deploy.SparkSubmit$.submit(SparkSubmit.scala:193)
        at org.apache.spark.deploy.SparkSubmit$.main(SparkSubmit.scala:112)
        at org.apache.spark.deploy.SparkSubmit.main(SparkSubmit.scala)
Caused by: java.lang.ClassNotFoundException: org.apache.hadoop.hbase.protobuf.generated.ClientProtos$Scan
        at java.net.URLClassLoader$1.ru
        n(URLClassLoader.java:366)
        at java.net.URLClassLoader$1.run(URLClassLoader.java:355)
        at java.security.AccessController.doPrivileged(Native Method)
        at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:425)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:358)
        ... 7 more
{{< / highlight >}}

缺少hbase-server-1.0.1.1.jar会报如下错误：
{{< highlight bash >}}
Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/hadoop/hbase/mapreduce/TableInputFormat
        at spark_hbase_main.main(spark_hbase_main.java:129)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:606)
        at org.apache.spark.deploy.SparkSubmit$.org$apache$spark$deploy$SparkSubmit$$runMain(SparkSubmit.scala:665)
        at org.apache.spark.deploy.SparkSubmit$.doRunMain$1(SparkSubmit.scala:170)
        at org.apache.spark.deploy.SparkSubmit$.submit(SparkSubmit.scala:193)
        at org.apache.spark.deploy.SparkSubmit$.main(SparkSubmit.scala:112)
        at org.apache.spark.deploy.SparkSubmit.main(SparkSubmit.scala)
Caused by: java.lang.ClassNotFoundException: org.apache.hadoop.hbase.mapreduce.TableInputFormat
        at java.net.URLClassLoader$1.run(URLClassLoader.java:366)
        at java.net.URLClassLoader$1.run(URLClassLoader.java:355)
        at java.security.AccessController.doPrivileged(Native Method)
        at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:425)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:358)
        ... 10 more
{{< / highlight >}}


缺少htrace-core-3.1.0-incubating.jar会报如下错误：
{{< highlight bash >}}
15/09/12 16:56:59 ERROR TableInputFormat: java.io.IOException: java.lang.reflect.InvocationTargetException
        at org.apache.hadoop.hbase.client.ConnectionFactory.createConnection(ConnectionFactory.java:240)
        at org.apache.hadoop.hbase.client.ConnectionFactory.createConnection(ConnectionFactory.java:218)
        at org.apache.hadoop.hbase.client.ConnectionFactory.createConnection(ConnectionFactory.java:119)
        at org.apache.hadoop.hbase.mapreduce.TableInputFormat.initialize(TableInputFormat.java:183)
        at org.apache.hadoop.hbase.mapreduce.TableInputFormatBase.getSplits(TableInputFormatBase.java:241)
        at org.apache.hadoop.hbase.mapreduce.TableInputFormat.getSplits(TableInputFormat.java:237)
        at org.apache.spark.rdd.NewHadoopRDD.getPartitions(NewHadoopRDD.scala:95)
        at org.apache.spark.rdd.RDD$$anonfun$partitions$2.apply(RDD.scala:219)
        at org.apache.spark.rdd.RDD$$anonfun$partitions$2.apply(RDD.scala:217)
        at scala.Option.getOrElse(Option.scala:120)
        at org.apache.spark.rdd.RDD.partitions(RDD.scala:217)
        at org.apache.spark.rdd.MapPartitionsRDD.getPartitions(MapPartitionsRDD.scala:32)
        at org.apache.spark.rdd.RDD$$anonfun$partitions$2.apply(RDD.scala:219)
        at org.apache.spark.rdd.RDD$$anonfun$partitions$2.apply(RDD.scala:217)
        at scala.Option.getOrElse(Option.scala:120)
        at org.apache.spark.rdd.RDD.partitions(RDD.scala:217)
        at org.apache.spark.Partitioner$.defaultPartitioner(Partitioner.scala:65)
        at org.apache.spark.rdd.PairRDDFunctions$$anonfun$groupByKey$3.apply(PairRDDFunctions.scala:587)
        at org.apache.spark.rdd.PairRDDFunctions$$anonfun$groupByKey$3.apply(PairRDDFunctions.scala:587)
        at org.apache.spark.rdd.RDDOperationScope$.withScope(RDDOperationScope.scala:147)
        at org.apache.spark.rdd.RDDOperationScope$.withScope(RDDOperationScope.scala:108)
        at org.apache.spark.rdd.RDD.withScope(RDD.scala:286)
        at org.apache.spark.rdd.PairRDDFunctions.groupByKey(PairRDDFunctions.scala:586)
        at org.apache.spark.api.java.JavaPairRDD.groupByKey(JavaPairRDD.scala:547)
        at spark_hbase_main.main(spark_hbase_main.java:167)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:606)
        at org.apache.spark.deploy.SparkSubmit$.org$apache$spark$deploy$SparkSubmit$$runMain(SparkSubmit.scala:665)
        at org.apache.spark.deploy.SparkSubmit$.doRunMain$1(SparkSubmit.scala:170)
        at org.apache.spark.deploy.SparkSubmit$.submit(SparkSubmit.scala:193)
        at org.apache.spark.deploy.SparkSubmit$.main(SparkSubmit.scala:112)
        at org.apache.spark.deploy.SparkSubmit.main(SparkSubmit.scala)
Caused by: java.lang.reflect.InvocationTargetException
        at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
        at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:57)
        at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
        at java.lang.reflect.Constructor.newInstance(Constructor.java:526)
        at org.apache.hadoop.hbase.client.ConnectionFactory.createConnection(ConnectionFactory.java:238)
        ... 33 more
Caused by: java.lang.NoClassDefFoundError: org/apache/htrace/Trace
        at org.apache.hadoop.hbase.zookeeper.RecoverableZooKeeper.exists(RecoverableZooKeeper.java:218)
        at org.apache.hadoop.hbase.zookeeper.ZKUtil.checkExists(ZKUtil.java:481)
        at org.apache.hadoop.hbase.zookeeper.ZKClusterId.readClusterIdZNode(ZKClusterId.java:65)
        at org.apache.hadoop.hbase.client.ZooKeeperRegistry.getClusterId(ZooKeeperRegistry.java:86)
        at org.apache.hadoop.hbase.client.ConnectionManager$HConnectionImplementation.retrieveClusterId(ConnectionManager.java:833)
        at org.apache.hadoop.hbase.client.ConnectionManager$HConnectionImplementation.<init>(ConnectionManager.java:623)
        ... 38 more
Caused by: java.lang.ClassNotFoundException: org.apache.htrace.Trace
        at java.net.URLClassLoader$1.run(URLClassLoader.java:366)
        at java.net.URLClassLoader$1.run(URLClassLoader.java:355)
        at java.security.AccessController.doPrivileged(Native Method)
        at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:425)
        at java.lang.ClassLoader.loadClass(ClassLoader.java:358)
        ... 44 more
{{< / highlight >}}

## spark集群环境下面读取hbase ##
如果要在集群上运行，会发现worker node访问不了hbase内容。原因我们之前的例子里没有指定hbase的地址信息。
加入下面代码：

{{< highlight java >}}
  conf.set("hbase.zookeeper.quorum", "18.18.18.18"); //指定ip地址
  conf.set("hbase.zookeeper.property.clientPort", "2182"); // zookeeper的端口号
{{< / highlight >}}

注意，在调试过程中，还是发现如果spark的其他worker运行在其他node上，hbase还是无法正常访问。打开spark的debug级别的log，
在spark目录conf/log4j.properties，修改：

{{< highlight java >}}
log4j.rootCategory=DEBUG, console
{{< / highlight >}}

发现其他主机通过主机名来访问hbase服务器，但是由于DNS或者/etc/hosts中并没有该主机信息，导致连接不上。修改worker主机上/etc/hosts文件
加入“18.18.18.18 hbase主机名”解决问题。

