---
layout: post
title: "在 Windows 上运行 MapReduce 的 Word Count"
date: "2018-04-25"
description: "在 Windows 上运行 MapReduce 的 Word Count"
tag: [hadoop, mapreduce]
---

**环境:** CentOS-6.8-x86_64, hadoop-2.7.3, Windows 7

### 需求
在 Linux 机器上安装了 Hadoop 集群, 但由于 Linux 机器没有装图形界面, IDEA 等图形界面的开发工具无法使用, 在 Windows 下开发完再扔到 Linux 机器上运行感觉十分麻烦; 于是想在 Windows 下开发, 作为客户端提交作业到 Linux 上部署的 Hadoop 集群, Google 一番果然已经有人这样做了, 照猫画虎记录如下

####  Word Count 代码
```
public class WordCount {

	//map内部类
	public static class WCMapper extends Mapper<LongWritable, Text, Text, LongWritable>{
		@Override
		protected void map(LongWritable key, Text value, Context context)
				throws IOException, InterruptedException {
			//取得每一行，并将其按空格分隔成数组
			String line = value.toString();
			String[] words = line.split(" ");
			//将每一行按空格切割，并将word作为key,1作为value传入context作为下一步reduce的输入
			for (String word : words) {
				context.write(new Text(word), new LongWritable(1));
			}
		}
	}

	//reduce内部类
	public static class WCReducer extends Reducer<Text, LongWritable, Text, LongWritable>{
		@Override
		protected void reduce(Text key, Iterable<LongWritable> values,
				Context context) throws IOException, InterruptedException {
			//计数器
			long counter = 0;
			for (LongWritable value : values) {
				counter += value.get();
			}
			context.write(key, new LongWritable(counter));
		}
	}

	public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException{
		Configuration conf = new Configuration();
		// 将集群地址配置到 conf 中, 因为是 hadoop ha 所以配置了 nameservices 等信息  
    conf.set("fs.defaultFS", "hdfs://ns1");
    conf.set("dfs.nameservices", "ns1");
    conf.set("dfs.ha.namenodes.ns1","nn1,nn2");
    conf.set("dfs.namenode.rpc-address.ns1.nn1", "test01:8020");
    conf.set("dfs.namenode.rpc-address.ns1.nn2", "test02:8020");
    conf.set("dfs.client.failover.proxy.provider.ns1", "org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider");
		// 取得一个作业实例
		Job job = Job.getInstance(conf);

		// 设置WordCount.class的路径
		job.setJarByClass(WordCount.class);

		// 设置mapper，以及mapper的key,value的class
		job.setMapperClass(WCMapper.class);
		job.setMapOutputKeyClass(Text.class);
		job.setMapOutputValueClass(LongWritable.class);
		// 将要计算的数据传入作业
		FileInputFormat.setInputPaths(job, new Path(args[0]));

		// 设置reducer,以及reducer的key，value的class
		job.setReducerClass(WCReducer.class);
		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(LongWritable.class);
		// 将作业计算的结果写出
		FileOutputFormat.setOutputPath(job, new Path(args[1]));

		// 等待作业完成
		job.waitForCompletion(true);
	}

}
```
在 Windows 机器上并没有安装 Hadoop 集群, 即系统中没有 HADOOP_HOME 等环境变量, 作业提交时也不知道到哪里, 所以需要配置集群地址信息等

#### 报错解决
- java.io.IOException: Could not locate executable null\bin\winutils.exe in the Hadoop binaries.  
- Exception in thread "main" java.lang.UnsatisfiedLinkError: org.apache.hadoop.io.nativeio.NativeIO$Windows.access0(Ljava/lang/String;I)Z

在 Github 找到了编译好的包和文件, 克隆到本地, 将文件夹在环境变量中配置为 HADOOP_HOME, 并把 %HADOOP_HOME%/bin 添加到 PATH 路径中, 确保代码运行环境中可以找到相关路径即可

>**参考:**
[第一个MapReduce程序——WordCount](https://songlee24.github.io/2015/07/29/mapreduce-word-count/)
[Win7 Eclipse调试Centos Hadoop2.2-Mapreduce](http://zy19982004.iteye.com/blog/2024467)
[hadoop.dll-and-winutils.exe-for-hadoop2.7.3-on-windows_X64](https://github.com/rucyang/hadoop.dll-and-winutils.exe-for-hadoop2.7.3-on-windows_X64)
