---
layout: post
title: "MapReduce WordCount实例"
date: 2016-03-06 15:08:34 +0800
comments: true
categories: BigData
tags: mapreduce
---
搭建开发环境，开发mapreduce job程序，部署mapreduce程序。
#开发环境搭建
开发mapreduce程序的环境有可能与运行环境不同，我是Windows环境上开发mapreduce job程序，编译没有错误后，打包放到hadoop集群中调试和运行。这样就要求开发环境依赖的jar包，在hadoop集群环境中存在并且版本一致。  
我的开发环境是Windows＋eclipse＋maven，hadoop集群环境上部署的版本号为：

	jintao@node0:~/Project$ hadoop  version
	Hadoop 2.7.1
	Subversion https://git-wip-us.apache.org/repos/asf/hadoop.git -r 15ecc87ccf4a0228f35af08fc56de536e6ce657a
	Compiled by jenkins on 2015-06-29T06:04Z
	Compiled with protoc 2.5.0
	From source with checksum fc0a1a23fc1868e4d5ee7fa2b28a58a
	This command was run using /usr/local/hadoop-2.7.1/share/hadoop/common/hadoop-common-2.7.1.jar

maven配置文件pom.xml的配置如下：
	
	<dependencies>
	<dependency>
  		<groupId>org.apache.hadoop</groupId>
  		<artifactId>hadoop-client</artifactId>
  		<version>2.7.1</version>
  	</dependency>
  	</dependencies>
 
maven工程会自动将依赖的jar下载到build path中。  
你也可以建立一个普通java工程，手动将依赖的jar添加到build path中，这样需要找依赖的jar，比较麻烦，但是可以让新手更快的了解mapreduce相关类分布情况。  

<!-- more -->

#开发mapreduce job程序
一般情况下，一个job包含一个map类和一个reduce类，但是也可以只有一个类，或者都不包含。本文例子中的WordCount是mapreduce程序开发的helloworld程序，包含map和reduce类，通过WordCount我们可以了解mapreduce程序开发的基本框架。
##Mapper
map程序的基类为：

	@InterfaceAudience.Public
	@InterfaceStability.Stable
	public class Mapper<KEYIN,VALUEIN,KEYOUT,VALUEOUT> extends Object

Maps将输入的key/value对处理并输出一个key/value形式的中间结果集。  

Maps是一个个独立的任务线程分别处理不同的记录集合，输出中间结果集。输出结果集的类型不一定与初始的输入记录相同。一个输入key/value对，有可能输出零个或多个key/value结果对。  

Hadoop Map-Reduce框架对于job的[InputFormat](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/InputFormat.html)输入数据产生的每个[InputSplit](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/InputSplit.html)派生一个map任务。Mapper的具体实现可以通过[JobContext.getConfiguration()](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/JobContext.html#getConfiguration())获取job的[Configuration配置](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/conf/Configuration.html)。  

框架首先调用Mapper实现类的[setup(org.apache.hadoop.mapreduce.Mapper.Context)](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Mapper.html#setup(org.apache.hadoop.mapreduce.Mapper.Context))，然后对于对于获取的每一个输入记录，调用map方法进行处理。最后调用[cleanup(org.apache.hadoop.mapreduce.Mapper.Context)](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Mapper.html#cleanup(org.apache.hadoop.mapreduce.Mapper.Context))。  

框架将与某个key相关的所有value值形成一组，传给Reducer来产生最后结果。通过指定key比较类，用户可以控制key/value的排序和分组。  

Mapper的输出结果对于每个Reducer进行分片。通过实现一个特定的[Partitioner](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Partitioner.html)，用户可以控制keys分片到Reducer。  

视情况，通过[Job.setCombinerClass(Class)](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Job.html#setCombinerClass(java.lang.Class))用户可以指定一个combiner，combiner可以在Mapper本地对输出结果进行聚合，这样可以减少从Mapper传输到Reducer的数据量。  

通过对Configuration程序可以指定是否并且如何对Mapper结果进行压缩。  

如果reducer个数为0，那么不会对Mapper的结果不对结果进行排序，并且直接输出到[OutputFormat](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/OutputFormat.html)。  
WordCount对应Mapper实现类如下： 
	
	public class WordCountMapper extends Mapper<LongWritable, Text, Text, IntWritable> {
	
		private final static IntWritable one = new IntWritable(1);
		private Text word = new Text();
		private static Logger logger = Logger.getLogger(WordCountMapper.class);
	
		public void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException{
			StringTokenizer iTokenizer = new StringTokenizer(value.toString());
			while (iTokenizer.hasMoreTokens()) {
				word.set(iTokenizer.nextToken());
				context.write(word, one);
			}
		}	
	}

应用也可以重写[ run(org.apache.hadoop.mapreduce.Mapper.Context)](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Mapper.html#run(org.apache.hadoop.mapreduce.Mapper.Context))来更多的控制map处理，比如multi-threaded Mapper。  

##Reducer
Reducer的基类定义如下：  

	@Checkpointable
	@InterfaceAudience.Public
	@InterfaceStability.Stable
	public class Reducer<KEYIN,VALUEIN,KEYOUT,VALUEOUT> extends Object
	
Reducer将Mapper输出的中间结果集处理输出最终结果集。  
Reducer实现类可以通过[ JobContext.getConfiguration()](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/JobContext.html#getConfiguration())获取job的配置信息[Configuration](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/conf/Configuration.html)。  
Reducer主要有三个步骤：  

1. Shuffle  
	Reducer使用Http从个Mapper拷贝传输中间结果集到本地。
	
2. Sort  
	框架将从不同Mapper传输过来的结果集按照key进行合并排序。  
3. Reduce
	在这一阶段，对于每个<key, (collection of values)>调用[reduce(Object, Iterable, org.apache.hadoop.mapreduce.Reducer.Context)](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/Reducer.html#reduce(KEYIN, java.lang.Iterable, org.apache.hadoop.mapreduce.Reducer.Context))方法进行处理。  
	reduce任务调用[TaskInputOutputContext.write(Object, Object)](TaskInputOutputContext.write(Object, Object))将最终结果写到[RecordWriter](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/RecordWriter.html)。  
	Reducer的输出不是有序的。  

WordCount的Reduce实现类为：  

	public class WordCountReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
		private IntWritable result = new IntWritable();
	
		public void reduce(Text key, Iterable<IntWritable> values,
            	Context context) throws IOException, InterruptedException {
			int total = 0;
			for (IntWritable count : values) {
				total += count.get();
			}
			result.set(total);
			context.write(key, result);
		}
	}
## Driver
已经完成mapper和reducer的编写。接下来写一个job driver用来提交程序到hadoop中运行。  
可以有两种方式编写job driver：  
	
 1. 实现一个普通Java类，在类中设置job属性信息，提交job
 2. 实现[Tool](http://hadoop.apache.org/docs/current/api/index.html)接口

实际应用中，我们采用实现Tool接口的方式，这种方式能够更好的控制程序运行。  
WordCount Job的属性设置如下：  

	public class WordCountDriver extends Configured implements Tool {

		public int run(String[] arg0) throws Exception {
			// TODO Auto-generated method stub
			if (arg0.length != 2) {
				System.err.printf("Usage: %s [generic options] <input> <output>\n", 
						getClass().getSimpleName());
				ToolRunner.printGenericCommandUsage(System.err);
				return -1;
			}
		
			//Create new job
			Job job = Job.getInstance();
			job.setJarByClass(getClass());
		
			job.setJobName("WordCount");
		
			job.setMapperClass(WordCountMapper.class);
			job.setReducerClass(WordCountReducer.class);
		
			job.setOutputKeyClass(Text.class);
			job.setOutputValueClass(IntWritable.class);
		
			job.setNumReduceTasks(1);
		
			FileInputFormat.addInputPath(job, new Path(arg0[0]));
			FileOutputFormat.setOutputPath(job, new Path(arg0[1]));
		
			return job.waitForCompletion(true) ? 0 : 1;
		}	
	
		public static void main(String[] args) {
			// TODO Auto-generated method stub
			try {
				int exitCode = ToolRunner.run(new WordCountDriver(), args);
				System.exit(exitCode);
			} catch (Exception e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
		}

	}	

WordCountDriver实现了Tool接口，这样我们运行程序时可以使用GenericOptionParser支持的参数项。  
调用Job.getInstance()获取一个Job实例。设置Job名称。设置输入路径和输出路径。设置mapper和reducer类名。设置输出key和value类型（输入类型由数据源格式确定，数据源缺省为TextInputFormat文本格式，默认key类型为LongWritable，value类型为Text）。
## 本地运行WordCount
将程序打成jar包，放到hadoop集群一台机器上去。在jar包文件所在目录，新建一个input目录，input目录中放置一个文本文件words.txt，运行以下命令： 

	$ hadoop jar hadoopjob.jar com.hugedata.jobs.WordCountDriver -fs file:/// -jt local input/ output/
	
-fs file:///指定使用本地文件系统，-jt local 指定mapreduce任务运行在本地。  
当前目录会产生一个output目录，里面有两个文件（part-r-00000、_SUCCESS），其中part-r-00000即为

	jintao@node0:~/Project$ cat output/part-r-00000
	is	2
	jintao	1
	jintao.	1
	mx	1
	my	2
	mz	1
	name	2
	name.	1
	too.	1
	whit's	1
	your	1
	
##集群运行WordCount
前提是集群环境已经运行正常。将本地目录下的input拷贝到hdfs上：  

	jintao@node0:~/Project$ hdfs dfs -put input
	jintao@node0:~/Project$ hdfs dfs -ls
	Found 2 items
	drwxr-xr-x   - jintao supergroup          0 2016-03-07 16:51 input
	drwxr-xr-x   - jintao supergroup          0 2016-03-05 14:04 tmp
屏幕上打印的过程信息如下： 

	16/03/07 16:55:18 INFO client.RMProxy: Connecting to ResourceManager at node0/192.168.88.120:8032
	16/03/07 16:55:20 INFO input.FileInputFormat: Total input paths to process : 1
	16/03/07 16:55:20 INFO mapreduce.JobSubmitter: number of splits:1
	16/03/07 16:55:20 INFO mapreduce.JobSubmitter: Submitting tokens for job: job_1457076326520_0001
	16/03/07 16:55:21 INFO impl.YarnClientImpl: Submitted application application_1457076326520_0001
	16/03/07 16:55:21 INFO mapreduce.Job: The url to track the job: http://node0:8088/proxy/application_1457076326520_0001/
	16/03/07 16:55:21 INFO mapreduce.Job: Running job: job_1457076326520_0001
	16/03/07 16:55:32 INFO mapreduce.Job: Job job_1457076326520_0001 running in uber mode : false
	16/03/07 16:55:32 INFO mapreduce.Job:  map 0% reduce 0%
	16/03/07 16:55:43 INFO mapreduce.Job:  map 100% reduce 0%
	16/03/07 16:55:51 INFO mapreduce.Job:  map 100% reduce 100%
	16/03/07 16:55:52 INFO mapreduce.Job: Job job_1457076326520_0001 completed successfully
	16/03/07 16:55:52 INFO mapreduce.Job: Counters: 49
	File System Counters
		FILE: Number of bytes read=156
		FILE: Number of bytes written=230351
		FILE: Number of read operations=0
		FILE: Number of large read operations=0
		FILE: Number of write operations=0
		HDFS: Number of bytes read=177
		HDFS: Number of bytes written=77
		HDFS: Number of read operations=6
		HDFS: Number of large read operations=0
		HDFS: Number of write operations=2
	Job Counters
		Launched map tasks=1
		Launched reduce tasks=1
		Data-local map tasks=1
		Total time spent by all maps in occupied slots (ms)=8022
		Total time spent by all reduces in occupied slots (ms)=4886
		Total time spent by all map tasks (ms)=8022
		Total time spent by all reduce tasks (ms)=4886
		Total vcore-seconds taken by all map tasks=8022
		Total vcore-seconds taken by all reduce tasks=4886
		Total megabyte-seconds taken by all map tasks=8214528
		Total megabyte-seconds taken by all reduce tasks=5003264
	Map-Reduce Framework
		Map input records=3
		Map output records=14
		Map output bytes=122
		Map output materialized bytes=156
		Input split bytes=110
		Combine input records=0
		Combine output records=0
		Reduce input groups=11
		Reduce shuffle bytes=156
		Reduce input records=14
		Reduce output records=11
		Spilled Records=28
		Shuffled Maps =1
		Failed Shuffles=0
		Merged Map outputs=1
		GC time elapsed (ms)=243
		CPU time spent (ms)=2860
		Physical memory (bytes) snapshot=444420096
		Virtual memory (bytes) snapshot=3891036160
		Total committed heap usage (bytes)=348651520
	Shuffle Errors
		BAD_ID=0
		CONNECTION=0
		IO_ERROR=0
		WRONG_LENGTH=0
		WRONG_MAP=0
		WRONG_REDUCE=0
	File Input Format Counters
		Bytes Read=67
	File Output Format Counters
		Bytes Written=77
	jintao@node0:~/Project$	
程序执行成功后，会产生一个output目录，执行一下命令，可以产看结果： 
	
	jintao@node0:~/Project$ hdfs dfs -cat output/part-r-00000
	is	2
	jintao	1
	jintao.	1
	mx	1
	my	2
	mz	1
	name	2
	name.	1
	too.	1
	whit's	1
	your	1
	
	
至此，讲述了最基本的mapreduce程序开发、运行过程，后续需要学习的有如果查看日志，如何调试优化程序等方面。