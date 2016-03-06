---
layout: post
title: "hdfs文件系统java api介绍"
date: 2015-12-21 17:29:19 +0800
comments: true
categories: BigData
tags: Hadoop Hdfs
---
# Hadoop Filesystems
Hadoop对于文件系统有一个抽象设计，HDFS只是一个实现。客户端使用Java 抽象类org.apache.hadoop.fs.FileSystem来访问Hadoop中的文件系统，有多个具体实现类:  
	
	FileSystem		URI scheme 		Java implementaion  
	Local			file            fs.LocalFileSystem  
	
	HDFS			hdfs			hdfs.DistributedFileSystem
	
	WebHDFS			webhdfs			hdfs.web.WebHdfsFileSystem
	
	Secure
	WebHDFs      	swebhdfs		hdfs.web.SWebHdfsFileSystem
	
	HAR				har				fs.HarFileSystem
	
	View			viewfs			viewfs.ViewFileSystem
	
	FTP				ftp				fs.ftp.FTPFileSystem
	
	S3				s3a				fs.s3a.S3AFileSystem
	
	Azure			wasb			fs.azure.NativeAzureFileSystem
	
	Swift			swift			fs.swift.snative.SwiftNative
	
Hadoop提供多个接口访问文件系统，接口利用URI的scheme来创建对应的实例与文件系统进行通讯。hdfs命令可以用来操作Hadoop各种文件系统，比如列出本地文件系统根目录内容：
	
	[root@yun1 ~]# hdfs dfs -ls file:///
	Found 25 items
	-rw-r--r--   1 root root          0 2015-10-19 11:06 file:///.autofsck
	-rw-r--r--   1 root root          0 2014-07-04 14:16 file:///.autorelabel
	dr-xr-xr-x   - root root       4096 2014-07-04 14:36 file:///bin
	dr-xr-xr-x   - root root       1024 2014-09-17 15:54 file:///boot
	drwxr-xr-x   - root root       4096 2014-07-04 09:17 file:///data
	-rw-r--r--   1 root root       1001 2014-07-29 16:28 file:///dataplatform.log
	drwxr-xr-x   - root root       3580 2015-10-19 11:08 file:///dev

# Java Interface  

<!-- more -->

访问HDFS文件系统需要先获取一个文件系统实例，FileSystem是一个文件系统统一访问接口。通过以下接口可以获取访问hdfs文件系统的实例：    
	
	public static FileSystem get(Configuration conf) throws IOException
	public static FileSystem get(URI uri, Configuration conf) throws IOException
	public static FileSystem get(URI uri, Configuration conf, String user) throws IOException
	
Configuration对象封装客户端或者服务端的配置信息，这些信息由classpath目录下配置文件初始化，比如etc/hadoop/core-site.xml。第一个接口根据conf中的信息，返回文件系统。第二个接口使用URI的scheme和授权来觉得使用哪个文件系统，如果没有设置scheme，则使用conf中配置的文件系统。第三个接口是以user用户的身份获取文件系统。 使用FileSystem实例，调用open方法可以获取一个输入文件流：  
	
	public FSDataInputStream open(Path f) throws IOException
	public abstract FSDataInputStream open(Path f, int bufferSize) throws IOException 
	
下面的代码实例用于读取hdfs文件输出到标准输出：

	public class FileSystemCat {
		
		public static void main(String[] args) throws Exception {
			String uri = args[0];
			Configuration conf = new Configuration();
			FileSystem fs = FileSystem.get(URI.create(uri), conf);
			InputStream in = null;
			try {
				in = fs.open(new Path(uri));
				IOUtils.copyBytes(in, System.out, 4096, false);
			} finally {
				IOUtils.closeStream(in);
			}
		}
	}

编译文件后，使用hadoop命令运行class文件：
	
	# hadoop FileSystemCat hdfs://localhost/user/tom/quangle.txt
	On the top of the Crumpetty Tree
	The Quangle Wangle sat, 
	But his face you could not see,
	On account of his Beaver Hat.

## Writing Data
FileSystem提供多个接口来创建文件，最简单的是输入一个路径作为参数，返回一个输出流：

	public FSDataOutputStream create(Path f) throws IOException	存在多个重载接口，允许指定是否重写已存在文件，文件的副本个数，写文件的缓冲区大小，文件的块大小和文件权限等等。存在append接口对已存在文件进行追加：
	public FSDataOutputStream append(Path f) throws IOException
下面的代码实例是拷贝一个本地文件到Hadoop文件系统中：

	public class FileCopyWithProgress {
		public static void main(String []args) throws Exception {
			String localSrc = args[0];
			String dst = args[1];
			
			InputStream in = new BufferedInputStream(new FileInputStream(localSrc));
			
			Configuration conf = new Configuration();
			FileSystem fs = FileSystem.get(URI.create(dst), conf);
			OutputStream out = fs.create(new Path(dst), new Progressable() {
				public void progress() {
					System.out.print(".");
				}
			});
			
			IOUtils.copyBytes(in, out, 4096, true);
		}
	}
运行程序结果如下：
	#hadoop FileCopyWithProgress input/docs/1400-8.txt hdfs://localhost/user/tom/1400-8.txt

创建目录使用下面的接口：
	
	public boolean mkdirs(Path f) throws IOException
	
## Quering the FileSystem
FilsSystem提供了接口来获取文件或者文件夹信息，信息包括文件长度，块大小，复制，修改时间，所有权，访问权限属性。  

	abstract FileStatus	getFileStatus(Path f)
	
FileSystem提供了接口来列出一个目录中包含的内容：  
	
	abstract FileStatus[]	listStatus(Path f)
	FileStatus[]	listStatus(Path[] files)
	FileStatus[]	listStatus(Path[] files, PathFilter filter)
	FileStatus[]	listStatus(Path f, PathFilter filter)