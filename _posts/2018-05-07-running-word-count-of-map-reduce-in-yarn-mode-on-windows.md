---
layout: post
title: "在 Windows 上运行 Yarn 模式的 MapReduce 的 Word Count"
date: "2018-04-25"
description: "在 Windows 上运行 Yarn 模式的 MapReduce 的 Word Count"
tag: [hadoop, mapreduce]
---


**环境:** CentOS-6.8-x86_64, hadoop-2.7.3, Windows 7

### 需求
使用 Local 模式运行时, MapReduce 作业的资源都是从本地机器上获取, 因此当作业较大时会运行的很慢, 由此提交到远程集群上去以获得较好的性能

####  Word Count 代码
```
public class WordCount {

	// map 内部类
	public static class WCMapper extends Mapper<LongWritable, Text, Text, LongWritable>{
		@Override
		protected void map(LongWritable key, Text value, Context context)
				throws IOException, InterruptedException {
			// 取得每一行，并将其按空格分隔成数组
			String line = value.toString();
			String[] words = line.split(" ");
			// 将每一行按空格切割，并将 word 作为 key, 1 作为 value 传入 context 作为下一步 reduce 的输入
			for (String word : words) {
				context.write(new Text(word), new LongWritable(1));
			}
		}
	}

	// reduce 内部类
	public static class WCReducer extends Reducer<Text, LongWritable, Text, LongWritable>{
		@Override
		protected void reduce(Text key, Iterable<LongWritable> values,
				Context context) throws IOException, InterruptedException {
			// 计数器
			long counter = 0;
			for (LongWritable value : values) {
				counter += value.get();
			}
			context.write(key, new LongWritable(counter));
		}
	}

	public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException{
		// 将集群地址配置到 conf 中
    Configuration conf = new Configuration();
    // hadoop ha 配置
    conf.set("fs.defaultFS", "hdfs://ns1");
    conf.set("dfs.nameservices", "ns1");
    conf.set("dfs.ha.namenodes.ns1","nn1,nn2");
    conf.set("dfs.namenode.rpc-address.ns1.nn1", "test01:8020");
    conf.set("dfs.namenode.rpc-address.ns1.nn2", "test02:8020");
    conf.set("dfs.client.failover.proxy.provider.ns1", "org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider");
    // yarn ha 配置
    conf.set("yarn.resourcemanager.ha.enabled", "true");
    conf.set("yarn.resourcemanager.cluster-id", "test-yarn");
    conf.set("yarn.resourcemanager.ha.rm-ids", "rm1,rm2");
    conf.set("yarn.resourcemanager.hostname.rm1", "test01");
    conf.set("yarn.resourcemanager.hostname.rm2", "test02");
    // mapreduce 配置
    conf.set("mapreduce.framework.name", "yarn");
    // 作业提交是否跨平台
    conf.set("mapreduce.app-submission.cross-platform", "true");
    conf.set("mapreduce.remote.os", "linux");
    conf.set("mapreduce.jobhistory.address", "test01:10020");
    // 设置集群上的 lib 目录: 这里将集群的路径建议不使用环境变量, 使用环境变量在 windows 下提交比较麻烦, 并且集群的安装地址并不会经常变化
    conf.set("mapreduce.application.classpath", "${HADOOP_HOME}/etc/hadoop,"
                +"${HADOOP_HOME}/share/hadoop/common/*,${HADOOP_HOME}/share/hadoop/common/lib/*,"
                +"${HADOOP_HOME}/share/hadoop/hdfs/*,${HADOOP_HOME}/share/hadoop/hdfs/lib/*,"
                +"${HADOOP_HOME}/share/hadoop/mapreduce/*,${HADOOP_HOME}/share/hadoop/mapreduce/lib/*,"
                +"${HADOOP_HOME}/share/hadoop/yarn/*,${HADOOP_HOME}/share/hadoop/yarn/lib/*");
		// 取得一个作业实例
		Job job = Job.getInstance(conf);

    // 设置 MR 处理类的路径: 这里因为 ExportResult 类所在的 jar 包已经存在 classpath 中,　所以可已使用 setJarByClass 的方式代替 setJar
		job.setJarByClass(WordCount.class);

		// 设置mapper，以及mapper的key,value的class
		job.setMapperClass(WCMapper.class);
		job.setMapOutputKeyClass(Text.class);
		job.setMapOutputValueClass(LongWritable.class);
		// 将要计算的数据传入作业
		FileInputFormat.setInputPaths(job, new Path(args[0]));

		// 设置 reducer,以及 reducer 的 key, value 的 class
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
相比于 Local 模式下提交 MapReduce 作业, Yarn 模式下提交只是多配置了 yarn 和 mapreduce 等部分信息

>**参考:**
[从Java代码远程提交YARN MapReduce任务](https://blog.csdn.net/xiao_jun_0820/article/details/43308743)  
[hadoop window 远程提交job到集群并运行](http://www.80iter.com/blog/1450700068151583)  
[hadoop运行mapreduce作业无法连接0.0.0.0/0.0.0.0:10020](https://blog.csdn.net/tangzwgo/article/details/25893989)  
[Win7 submit mapreduce with problem Stack trace: ExitCodeException exitCode=1: /bin/bash: line 0: fg: no job control](https://community.hortonworks.com/questions/66238/win7-submit-mapreduce-with-problem-stack-trace-exi.html)  
[/bin/bash: line 0: fg: no job control一般解决方法](https://blog.csdn.net/fansy1990/article/details/27526167)  
[IDE远程提交mapreduce任务至linux，遇到ClassNotFoundException: Mapper](https://blog.csdn.net/qq_19648191/article/details/56684268)   
