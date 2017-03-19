---
layout: post
title: "protocol-buffers proto3 翻译"
date: 2017-03-12 18:09:36 +0800
comments: true
categories: program
tags: protocol-buffer
---
* [定义Message类型](#message)   
* [标量值类型](#type)
* [默认值](#default)
* [枚举](#enum)
* [使用其他消息类型](#other_message)
* [嵌套类型](#nested)
* [更新消息类型](#update)
* [未知字段](#unknown)
* [Any](#any)
* [Oneof](#oneof)
* [Maps](#maps)
* [包](#packages)
* [服务](#services)
* [json](#json)
* [选项](#options)
* [生成代码](#protoc)
  
这个指导文档如何使用protocol buffer语言结构化你的protocol buffer数据，包括`.proto`文件语法和如何根据`.proto`文件生成代码访问数据。本文主要描述`proto3`版本。    
本文是参考文档，如果需要例子的话可以访问，你选择语言相关的[入门指导](https://developers.google.com/protocol-buffers/docs/tutorials) 

## <a name="message"></a>Message定义  
一个简单例子。下面定义一个搜索请求消息格式，这个消息包含一个查询字符串，包含一个搜索结果页码，和每页包含搜索结果数。下面是该消息`.proto`文件：    

	syntax = "proto3";

	message SearchRequest {
  		string query = 1;
  		int32 page_number = 2;
  		int32 result_per_page = 3;
	}
* 第一行说明正在使用`proto3`语法：如果不指定proto版本，protocol buffer编译器默认认为正在使用`proto2`。这行语句必须是第一条非空、非注释语句。  
* `SearchRequest`消息包含三个字段。每个字段有都有一个名字和类型。  

<!-- more -->

###字段类型  
在上面的例子中，所有字段都是标量类型：两个整型（`page_number`和`result_per_page`）和字符串（`query`）。你也可以指定字段类型为组合类型，包含枚举或者其他类型。  

### 添加Tag标签  
如定义所见，每一个字段都有一个唯一的数字标签。这些标签用来在[消息二进制格式](https://developers.google.com/protocol-buffers/docs/encoding)中标示相应字段，一旦消息类型得到使用，就不应该在修改字段标签。值在1到15之间的tag编码时只占用一个字节，一个字节包含了tag值和字段类型，更多信息可以参考[Protocol Buffer Encoding](https://developers.google.com/protocol-buffers/docs/encoding.html#structure)。16到2047之间的标签占用两个字节。所以应该用1到15之间的标签标示经常使用的字段，并且给未来可能添加进来的频繁使用字段预留一些标签。   
 
自小标签值为1，最大值为2的29次方-1，即536,870,911。tag值区间19000到19999（FieldDescriptor::kFirstReservedNumber 到 FieldDescriptor::kLastReservedNumber）是`Protocol Buffers`的保留值，如果在`.proto`文件中使用了这个范围内的值，编译器将会报警。  

### 指定字段规则  
消息类型可以为如下类型：  

* 单数形式： 一个正确格式的消息可以包含0个或者一个这样的字段（不能大于一个） 
* 重复类型： 这种类型的字段可以在一个消息中重复多次（包括0次）。  

`proto3`中，标量类型的`repeated`字段类型默认使用`packed`编码。  

可以在[Protocol Buffer Encoding](Protocol Buffer Encoding)查看关于编码的详细内容。  

###添加更多Message类型  
一个`.proto`文件中可以定义多个消息类型。因此可以将相关消息类型定义在一个`.proto`文件中，比如可以在上面的proto文件添加一个`SearchResponse`消息类型：  

	message SearchRequest {
  		string query = 1;
 		int32 page_number = 2;
  		int32 result_per_page = 3;
	}

	message SearchResponse {
 		...
	}

### 添加注释  
`.proto`文件中，使用C/C++风格`//`注释

	message SearchRequest {
		string query = 1;
  		int32 page_number = 2;  // Which page number do we want?
	  	int32 result_per_page = 3;  // Number of results to return per page.
	}

###保留字段 
当完整删除一个字段，或者注释掉字段，这种方式修改一个消息类型时，未来用户可能会复用之前的标签值。如果加载就版本`.proto`文件时，这种情况可能会引起数据冲突，程序问题或者其他异常情况。可以通过将已经删除字段的tag值或者字段名设置为`reserved`。如果将来有人用到这些保留值，编译器将会报错。  

	message Foo {
	  	reserved 2, 15, 9 to 11;
  		reserved "foo", "bar";
	}

注意不可以在同一个`reserved`语句中同时标示tag和字段  


###从.proto文件生成什么
当用编译器处理已经编写好的`.proto`文件时，编译器会根据选择的编程语言类型生成代码，使用这些代码可以操作`.proto`文件中定义的消息类型，包括获取、设置字段值，序列化消息到输出流，从输入流中反序列化消息。  
	
* C++，编译器为每个`.proto`文件生成一个`.h`和`.cc`文件，对于文件中定义的每个消息类型生成一个类 
* Java,编译器为每个消息类型生成一个`.java`文件，同时生成一个`Builder`类用于创建消息类实例
* Python，编译器为每个消息类型生成一个包含静态描述符的模块，在python运行时，可以与原类型一起生成一个Python数据存取类  
* Go，编译器为`.proto`文件中的每个消息类型生成一个`.pb.go`文件
* Ruby，编译器生成一个`.rb`文件包含一个ruby模块包括所有文件中国年定义的消息类型  
* JavaNano，编译器生成的内容与Java类似，但是没有`Builder`类型。  
* Objective-C，编译器为每一个`.proto`文件生成一个`pbobjc.h`和`pbobjc.m`文件，为每个消息类型生成一个类。  
* C#，编译器生成一个`.cs`文件，每个消息类型生成一个类。  

可以从[API reference](https://developers.google.com/protocol-buffers/docs/reference/overview)中了解更多API相关信息  

## <a name="type"></a>标量值类型  
一个标量消息字段可以用如下消息类型，这些类型可以使用在`.proto`文件中，表中列出了相应语言的数据类型：  
	
![](/images/proto-1.png)
![](/images/proto-2.png)
![](/images/proto-3.png)
![](/images/proto-4.png)
![](/images/proto-5.png)

可以在[Protocol Buffer Encoding](https://developers.google.com/protocol-buffers/docs/encoding)中查看上面类型如何编码。

## <a name="default"></a>默认值
解析消息时，如果编码消息没有包含某个元素值，那么消息对象中对应字段的值设置为字段类型的默认值。不同类型具有不同默认值：  

* 字符串类型，默认为空字符串
* 字节类型，默认为空字节
* 布尔型，默认为false
* 数字类型，默认为0
* 枚举类型，默认为枚举类型第一个值，这个值必须为0
* message类型，字段不进行设置。具体值依赖不同语言的初始化操作。

重复型字段默认值为空。    

注意，对于标量类型字段，一个消息解析时不会标示某个字段是否是显示设置为默认值，比如一个布尔型字段值为`false`时，不会告诉消息使用者这个布尔字段值为`false`还是被设置的默认值`false`。定义消息类型时，要注意这点。如果一个布尔字段值为`false`时，程序表现为某种行为，如果你不希望默认情况下程序表现为这种行为的话，就需要注意。对于标量类型字段，如果设置为默认值，那么将不会对这个字段进行编码传输。   
[generated code guide](https://developers.google.com/protocol-buffers/docs/reference/overview)查看关于不同语言默认值如何工作的详细情况。  

## <a name="enum"></a>枚举  
定义消息类型时，可能需要设置字段值为一些预定义值，我们可以在消息定义文件中定义`enum`枚举类型，在`SearchRequest`中定义一个枚举类型`Corpus`：

	message SearchRequest {
  		string query = 1;
	  	int32 page_number = 2;
	  	int32 result_per_page = 3;
	  	enum Corpus {
    		UNIVERSAL = 0;
	    	WEB = 1;
    		IMAGES = 2;
    		LOCAL = 3;
    		NEWS = 4;
    		PRODUCTS = 5;
    		VIDEO = 6;
  		}
  		Corpus corpus = 4;
	}

`Corpus`枚举类型的第一个常量值为0：protobuf中每一个枚举类型`必须`将第一个值定义为0，原因如下：  
  
* 必须有一个0值，这样可以使用0值为枚举类型默认值  
* 0值必须为第一个值，这样是为了与`proto2`语法相兼容，`proto2`中以第一个枚举值为默认值  

可以为枚举值定义一个别名，需要在定义别名的枚举类型中设置`allow_alias`选项为`true`，否则编译器将会报错。
	
	enum EnumAllowingAlias {
  		option allow_alias = true;
  		UNKNOWN = 0;
  		STARTED = 1;
  		RUNNING = 1;
	}	
	enum EnumNotAllowingAlias {
  		UNKNOWN = 0;
  		STARTED = 1;
  		// RUNNING = 1;  // Uncommenting this line will cause a 		compile error inside Google and a warning message outside.
	}

枚举类型常量值必须在32位整型之间。因为`enum`类型使用[varint encoding](https://developers.google.com/protocol-buffers/docs/encoding)进行编码，值为负数的话，编码效率低，所以不推荐为负数。可以在消息定义体之中，或者之外定义`enum`枚举类型，可以在`.proto`文件任何为值使用这些枚举类型。 你可以通过`MessageType.EnumType`的方式使用在其他消息中定义的枚举类型。    

当编译带有`enum`的`.proto`文件时，目标语言是`C++`或者`Java`时，产生的代码中包含`enum`定义，目标代码是`Python`时，产生一个`EnumDescriptor `类。  


## <a name="other_message"></a>使用其他消息类型  
可以使用其他消息类型定义字段。比如在`SearchResponse`消息中包含`Result`消息，`Result`消息可以定义在同一个`.proto`文件中。  

	message SearchResponse {
		repeated Result results = 1;
	}

	message Result {
  		string url = 1;
  		string title = 2;
  		repeated string snippets = 3;
	}

###引入定义 
上面的例子中，`Result`定义在`SearchResponse`同样的`.proto`文件中，如果需要使用的消息类型定义在其他文件中消息类型。  

通过import`.proto`文件使用`.proto`文件中的定义。在自己`.proto`文件顶部添加一行引入语句：  

	import "myproject/other_protos.proto";
默认情况下，只能使用被直接`import`的文件中的消息定义。如果需要将`.proto`文件挪到一个新的位置去，可以放一个替代`.proto`文件到老的位置，在这个替代文件中使用`import public`将定义转移到新位置。例子：  

	// new.proto
	// All definitions are moved here


	// old.proto
	// This is the proto that all clients are importing.
	import public "new.proto";
	import "other.proto";
	
	// client.proto
	import "old.proto";
	// You use definitions from old.proto and new.proto, but not other.proto


编译器在`-I/--proto_path`参数指定的目录中搜索被引入的`.proto`文件。如果没有设置这个参数，编译器在当前目录下查找，通常情况下，应该设置`--proto_path`为工程的根目录。  

###使用proto2消息类型  
可以在`proto3`消息中引入`proto2`消息类型。但是proto2枚举类型不能在proto3语法中直接使用。  

## <a name="nested"></a>嵌套类型  
可以在消息类型中定义其他类型，下面的例子中，`Result`消息定义在`SearchResponse`消息中：  
	
	message SearchResponse {
  		message Result {
    	string url = 1;
    	string title = 2;
    	repeated string snippets = 3;
  		}
  		repeated Result results = 1;
	}  

如果想在`SearchResponse`消息以外使用`Result`消息，以`Parent.Type`格式使用：  

	message SomeOtherMessage {
  		SearchResponse.Result result = 1;
	}

可以任意潜逃消息定义：  
	
	message Outer {                  // Level 0
  		message MiddleAA {  // Level 1
    		message Inner {   // Level 2
      			int64 ival = 1;
      			bool  booly = 2;
    		}
  		}
  		message MiddleBB {  // Level 1
    		message Inner {   // Level 2
      			int32 ival = 1;
      			bool  booly = 2;
    		}
  		}
	}

## <a name="update"></a>更新消息类型  
如果已有消息不能满足新的需求，比如，需要增加一个新的字段，但是仍然使用根据老的消息格式产生的代码，对于protocol buffer来说这很容易完成，请记住如下的规则：  

* 不要修改当前字段的标签值   
* 如果新增字段，任何根据老格式序列化的消息，都能够被根据新格式产生的代码进行解析。这种情况下，新增字段的值都会被设置为[默认值](https://developers.google.com/protocol-buffers/docs/proto3#default)。新格式消息同样可以为老格式代码解析，新增字段将会被忽略。  
* 可以删除字段，只要新消息格式中该字段标签值没有被使用。对于删除的字段，最好是对字段名称进行修改，添加`OBSOLETE_ `字段或者设置tag为[reserved](https://developers.google.com/protocol-buffers/docs/proto3#reserved)，这样未来不会错误使用该tag值。  
* `int32`,`uint32`,`int64`,`uint64`,`bool`都是兼容的，意味着可以将字段从一种类型修改为另一种。如果解析时，从序列化数据中解析出的数据与响应字段类型不匹配，那么行为表现为从类型转换。  
* `sint32`和`sint64`互相兼容，但是与其他整型类型不兼容。  
* `string`和`bytes`互相兼容，只要bytes是UTF-8格式。  
* 嵌套消息与`bytes`互相兼容，只要bytes包含该消息的编码 
* `fixed32`与`sfixed32`，`fixed64`，`sfixed64`相兼容。  
* `enum`与`int32`，`uint32`，`int64`，`uint64`相兼容

## <a name="unknown"></a>未知字段  
未知字段是正确格式数据，解析程序不能识别的字段。当旧格式解析程序解析新格式消息时，解析程序无法识别新格式消息中新增字段，认为为未知字段。  

`Proto3`实现可以解析这些未知字段，但是实现可能不支持保留这些字段。不应依赖这些未知字段。对于大部分实现，未知字段都是不可访问的，在序列化时，未知字段将会被忽略。这与`proto2`不同，未知字段也会被保留和序列化。  

## <a name="any"></a>Any
`Any`消息类型允许你使用某消息类型，而不需要引入该类型消息`.proto`文件。`Any`类型包含任意`bytes`类型序列化数据，`Any`类型包含一个URL，作为消息类型的唯一标识。引入`google/protobuf/any.proto`来使用`Any`类型。
 
 	import "google/protobuf/any.proto";

	message ErrorStatus {
  		string message = 1;
  		repeated google.protobuf.Any details = 2;
	}

一个消息类型的默认URL为`type.googleapis.com/packagename.messagename`。    

不同语言都会有运行时库去打包和解包Any类型值，Java中，Any类型字段会有`pack()`和`unpack()`方法访问，C++语言中有`PackFrom()`和`UnpackTo()`方法：  

	// Storing an arbitrary message type in Any.
	NetworkErrorDetails details = ...;
	ErrorStatus status;
	status.add_details()->PackFrom(details);

	// Reading an arbitrary message from Any.
	ErrorStatus status = ...;
	for (const Any& detail : status.details()) {
  		if (detail.Is<NetworkErrorDetails>()) {
    		NetworkErrorDetails network_error;
    		detail.UnpackTo(&network_error);
    		... processing network_error ...
  		}
	}
	
## <a name="oneof"></a>Oneof
如果消息中包含多个字段，同一时间最多设置一个字段，可以使用`oneof`特性来强制这种行为并且节省内存。  

设置`oneof`中的任一字段将会清楚其他字段值。可以使用`case`或者`WhichOneof`方法查看当前哪个字段被设置。  

### 使用Oneof
使用`oneof`关键词在`.proto`文件中定义：  
	
	message SampleMessage {
  		oneof test_oneof {
    		string name = 4;
    		SubMessage sub_message = 9;
  		}
	}

可以在oneof定义中添加除了`repeated`以外任意类型的字段。  

产生的代码中，对于oneof中的字段拥有跟普通字段一样的获取、设置函数。还会获取一个特殊方法检查哪个值被设置。关于oneof更多API请参考[API reference](https://developers.google.com/protocol-buffers/docs/reference/overview)。  

### Oneof 特性  
* 设置oneof字段会清楚其他字段值。如果多次设置oneof字段值，只保留最后一次设置值。

		SampleMessage message;
		message.set_name("name");
		CHECK(message.has_name());
		message.mutable_sub_message();   // Will clear name field.
		CHECK(!message.has_name());  
* 如果解析程序遇到oneof定义中的多个字段值，只有最后一个是有效的。  
* 不能将一个`oneof`类型用于`repeated`中
* 反射对于oneof字段有效
* 使用C++时，防止非法访问内存。下面代码示例中，执行`set_name`时，`sub_message`已经被清除：  
		
		SampleMessage message;
		SubMessage* sub_message = message.mutable_sub_message();
		message.set_name("name");      // Will delete sub_message
		sub_message->set_...            // Crashes here
*使用C++ oneof `Swap`方法时，互换两个变量中设置的字段值：  

		SampleMessage msg1;
		msg1.set_name("name");	
		SampleMessage msg2;
		msg2.mutable_sub_message();
		msg1.swap(&msg2);
		CHECK(msg1.has_sub_message());
		CHECK(msg2.has_name());

###向后兼容问题  
添加或者删除oneof字段时要小心。如果检查oneof值时返回`None`或者`NOT_SET`时,意味着oneof没有被设置或者设置了oneof的一个其他版本。 因为没有办法区分一个未知字段是新加字段。  
  
复用标签问题  

* 添加或者移除字段： 可能会丢失某些字段信息。  
* 删除字段后又添加： 
* 分拆或者合并：  

## <a name="maps"></a>Maps  
protocol buffers提供一个映射类型定义字段：  
	
	map<key_type, value_type> map_field = N;
`key_type `可以为整型或者字符串类型（除了`floating point`和`bytes`以外的标量类型）。`value_type`可以为任何类型。  

下面定义个map类型字段，每个`project`对应一个字符串：  

	map<string, Project> projects = 3;

* Map字段不可以是`repeated`
* 不依赖map顺序  
* 生成`.proteo`文本格式时，maps以key排序。数字按照数字排序  

###向后兼容  
map语法等同于下面的消息定义，所以不支持map的protocol buffers实现仍然可以解析数据：  

	message MapFieldEntry {
  		key_type key = 1;
  		value_type value = 2;
	}

	repeated MapFieldEntry map_field = N;
 
 
## <a name="packages"></a>包
可以在`.proto`文件中添加一个`package`可选项，防止消息类型命名冲突。  
	
	package foo.bar;
	message Open { ... }
	
可以在定义字段时使用包名：  
	
	message Foo {
  		...
  		foo.bar.Open open = 1;
	  	...
	}

根据不同语言产生不同的代码：  

* C++中，产生的类包含在C++命名空间中，例如，`Open`将会出现在`foo::bar`命名空间  
* Java，`package`名被用来作为Java包名，除非在`.proto`文件中使用`option java_package`指定包名  
* Python，`package`指令被忽略，因为Python中包是按照在文件系统中的路径组织的
* Go，`package`包名被用做Go代码包名，除非使用`option java_package`指定包名 
* Ruby，产生的代码包裹在Ruby命名空间，命名空间名称根据`package`参数包名生成
* JavaNano，`package`参数名被用做产生代码包名  
* C#，`package`参数名被用来作为产生代码包名  

###包和命名查找 
protocol buffer中名字查找规则与C++类似：从内到外，逐层查找。`.foo.bar.Baz`意思是从最外层开始查找。  
## <a name="services"></a>定义服务 
如果在`RPC`中使用定义的消息类型，可以在`.proto`文件中定义一个RPC服务接口，protocol buffer编译器将会产生服务接口代码。下面定义一个服务函数，输入`SearchRequest`返回一个`SearchResponse `：

	service SearchService {
  		rpc Search (SearchRequest) returns (SearchResponse);
	}

最简单的使用protocol buffers的RPC系统是[gRPC](https://github.com/grpc/grpc-common)：Google开发的一个语言和平台无关的开源RPC系统。使用一个protocol buffer编译器插件可以直接从`.proto`文件中直接生成相应的RPC代码。  

## <a name="json"></a>JSON映射  
Proto3支持JSON编码，方便系统间共享数据。下面的表格列入了Proto3消息类型与JSON对应关系。 

![](/images/proto-6.png)
![](/images/proto-7.png)

## <a name="options"></a>Options选项  
单独的声明可以用一些options修饰。选项不会改变一个声明的总体意义，但是会影响在特殊上下文中的意义。完整options选项定义在`google/protobuf/descriptor.proto`。  
一些options选项是文件级别的。一些选项是消息级别的。一些选项是字段级别的。  
下面是一些经常是用的选项：  

* `java_package`(文件级别选项)：使用这个选项指定产生的java代码包名。如果没有使用这个参数显示指定包名，编译器将会使用`.proto`文件中package指定的包名作为java代码包名。但是proto包名生成的java包名并不符合Java包名倒序的惯例。如果不是生成Java代码，这个选项无效：  
		
		option java_package = "com.example.foo";
* `java_multiple_files`（文件级别选项）：指定产生的最外层Java类名。如果没有使用`java_outer_classname `参数，则使用`.proto`文件名作为最外层类名，文件名被转换成驼峰模式（`foo_bar.proto`生成`FooBar.java`）。如果不是生成java代码，下面选项无效：  
		
		option java_outer_classname = "Ponycopter";
* `optimize_for `（文件级别选项）：可以设置为`SPEED `，`CODE_SIZE`，`LITE_RUNTIME`。这个参数按照下面方式影响C++和Java代码生成：  

	* `SPEED`(默认值)：protocol buffer编译器为消息类型生成序列化、反序列化和其他操作的相关代码。代码是高度优化的。  
	* `CODE_SIZE`：protocol buffer编译器生成最少类，依赖共享的，基于反射的代码去实现序列化、反序列化和其他操作。生成的代码与`SPEED`相比很小，但是更慢。生成的类的接口与`SPEED`模式一样。这个选项在拥有大量`.proto`文件不需要它们都很快。  
	* `LITE_RUNTIME`：编译器将生成类只依赖轻运行时库（`libprotobuf-lite`而不是`libprotobuf`）。轻运行时库比全库更小。这对于运行在限制平台（比如智能手机）的应用很有用。编译器将会生成跟`SPEED`选项一样的高效代码。生成的类只实现`MessageLite`接口，只提供了全`Message`接口方法的子集。  
		
			option optimize_for = CODE_SIZE;
			
* `cc_enable_arenas`（文件级别选项）：为产生的C++代码使能[arena allocation](https://developers.google.com/protocol-buffers/docs/reference/arenas)
* `objc_class_prefix`（文件级别选项）：为产生的Objective-C代码添加前缀。没有默认值，应该按照Apple推荐的惯例设置此选项。 
* `deprecated`（文件级别选项）：如果设置为`true`，指示这个字段已弃用。在大多数语言中，这个选项无效。Java代码中，这个选项变成`@Deprecated`注解。  
		
		int32 old_field = 6 [deprecated=true];
## 自定义选项 
Protocol Buffers also allows you to define and use your own options. This is an advanced feature which most people don't need. If you do think you need to create your own options, see the Proto2 Language Guide for details. Note that creating custom options uses extensions, which are permitted only for custom options in proto3.
## <a name="protoc"></a>生成代码
使用`protoc`编译器根据`.proto`文件生成Java，Python，C++，Go，Ruby，JavaNano，Objective-C，或者C#代码。如果格式调用程序：  

		protoc --proto_path=IMPORT_PATH --cpp_out=DST_DIR --java_out=DST_DIR --python_out=DST_DIR --go_out=DST_DIR --ruby_out=DST_DIR --javanano_out=DST_DIR --objc_out=DST_DIR --csharp_out=DST_DIR path/to/file.proto

* `IMPORT_PATH` 指定目录，在目录中查找`import`指令引入的文件。如果没有制定，使用当前目录。可以多次使用`--proto_path`参数指定多个路径；按照顺序查找。`-I=IMPORT_PATH`可以作为`--proto_path`的短形式  
* 可以设置多个输出指令：  
	* `--cpp_out`在`DST_DIR`中生成C++代码。  
	* `--java_out`在`DST_DIR`中生成Java代码  
	* `--python_out`在`DST_DIR`中生成Python代码  
	* `--go_out`在`DST_DIR`中生成Go代码  
	* `--ruby_out`在`DST_DIR`中生成Ruby代码
	* `--javanano_out`在`DST_DIR`中生成JavaNano代码
	* `--objc_out`在`DST_DIR`中生成Objective-C代码 
	* `--csharp_out`在`DST_DIR`中生成C#代码 
	* `--php_out`在`DST_DIR`中生成PHP代码
* 必须指定一个或者多个`.proto`文件作为参数。多个`.proto`文件可以同时指定。每个文件必须在`IMPORT_PATH`指定的路径中能够查找到



