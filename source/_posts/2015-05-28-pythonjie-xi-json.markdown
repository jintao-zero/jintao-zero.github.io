---
layout: post
title: "python解析json"
date: 2015-05-28 15:21:55 +0800
comments: true
categories:
tags: [Python]
---
#JSON简介
JSON(JavaScript Object Notation)是一个轻量级的数据交换格式。方便人类读写。方便机器解析和产生。它基于[JavaScript Programming Language](http://javascript.crockford.com/)的子集，[Standard ECMA-262 3rd Edition - December 1999.](http://www.ecma-international.org/publications/files/ECMA-ST/Ecma-262.pdf)。JSON使用完全独立于各种语言的文本格式，但是也使用了类似C语言家族的习惯，包括C,C++,C#,Java,JavaScript,Perl,Python以及其他语言。这些特性使得JSON成为理想的数据交换语言。  

JSON基于两种结构：  

*  一个name/value对集合。在各种语言中，这种集合被认为是一个对象，记录，结构体，字典，哈希表，有键列表，或者关联数组
*  一个有序列表。在大多数语言中，这种列表被认为是一个数组，向量，列表，或者序列。

这些都是常见的数据结构。事实上大部分现代计算机语言都以某种形式支持它们。这使得一种数据格式在同样基于这些结构的编程语言之间交换成为可能。
关于JSON格式中各个基本类型和数据结构的详细描述，可以参考[json](http://www.json.org/)

  <!-- more -->

#Python JSON模块
##类
类JSONDecoder

	class JSONDecoder(__builtin__.object)   
将JSON格式解码为Python对应类型，下面是对应关系：  
	
     	| JSON          | Python            |
        +===============+===================+
        | object        | dict              |
        +---------------+-------------------+
       	| array         | list              |
       	+---------------+-------------------+
       	| string        | unicode           |
       	+---------------+-------------------+
       	| number (int)  | int, long         |
       	+---------------+-------------------+
       	| number (real) | float             |
       	+---------------+-------------------+
       	| true          | True              |
       	+---------------+-------------------+
       	| false         | False             |
       	+---------------+-------------------+
       	| null          | None              |
       	+---------------+-------------------+
       	
类JSONEncoder	

	class JSONEncoder(__builtin__.object)
将Python中的类型转换为JSON格式，下面是对应关系：

        | Python            | JSON          |
        +===================+===============+
        | dict              | object        |
        +-------------------+---------------+
        | list, tuple       | array         |
        +-------------------+---------------+
        | str, unicode      | string        |
        +-------------------+---------------+
        | int, long, float  | number        |
        +-------------------+---------------+
        | True              | true          |
        +-------------------+---------------+
        | False             | false         |
        +-------------------+---------------+
        | None              | null          |
        +-------------------+---------------+
	  
##函数
json模块对外提供了4个函数来进行json格式的编码、解码工作：  
1、dump函数 
 
	dump(obj, fp, skipkeys=False, ensure_ascii=True, check_circular=True, allow_nan=True, cls=None, indent=None, separators=None, encoding='utf-8', default=None, sort_keys=False, **kw)

序列化``obj``为json格式流保存到fp中（fp是一个支持.write()函数的类文件对象）  

* skipkeys  
  如果``skipkeys``为true，字典（``dict``）中键不为基本类型（``str``,``unicode``,``int``,``long``,``float``,``bool``,``None``）时，键值对将会被过滤掉，否则产生``TypeError``异常

* ensure_ascii    
  如果``ensure_ascii``为true(默认为true)，输出中的所有非ASCII字符都转义为``\uXXXX``序列，并且返回值序列只包含ASCII序列。

* check_circular  
  如果check_circular为false，将会忽略对容器类型的循环引用检测，进而导致``OverflowError``
  
* allow_nan  
  如果``allow_nan``为false，序列化float类型无限值（``nan``,``inf``,``-inf``）时，将会导致``ValueError``错误
  
* indent  
  如果indent指定为非负数值，JSON数组和对象成员将会以indent缩进整齐打印。

* separators  
  如果separators是一个``(item_separator, dict_separator)``元组，它就会用来替代缺省的``(', ', ': ')``分隔符。``(',', ':')``是最长用的压缩分隔符
* encoding
  json文本字符编码，缺省为UTF-8
* default(obj)  
  default(obj)是一个函数返回obj的序列化版本或者产生TypeError。
* sort_keys  
  如果为True(sort_keys默认为False)，字典将会按照key排序输出。
   
    
2、dumps函数
	
	dumps(obj, skipkeys=False, ensure_ascii=True, check_circular=True, allow_nan=True, cls=None, indent=None, separators=None, encoding='utf-8', default=None, sort_keys=False, **kw)
  	  	 
序列化``obj``对象为JSON格式的字符串。
各个参数的意思于上面dump函数相同

3、load函数

	load(fp, encoding=None, cls=None, object_hook=None, parse_float=None, parse_int=None, parse_constant=None, object_pairs_hook=None, **kw)

load函数从fp引用的对象中读取JSON格式字符串，反序列化为Python对象。  

* encoding  
  如果fp的内容不是基于ASCII字符集，那么需要指定encoding为相应编码集。
* object_hook  
  object_hook可选参数被用来代替Python中的``dict``类型解析json序列串中的对象。这一特性被用来定制解码器。
* object_pairs_hook  
  object_pairs_hook作为可选函数，被用来解析有序对，如果同时设置了object_pairs_hook和object_hook参数，优先使用object_pairs_hook函数。
  
4、loads

	loads(s, encoding=None, cls=None, object_hook=None, parse_float=None, parse_int=None, parse_constant=None, object_pairs_hook=None, **kw)

loads函数将s字符串反序列化为Python对象。  

* parse_float  
  如果指定parse_float参数，它被用来反序列化float字符串，缺省为float(num_str)。

* parse_int  
  如果指定parse_int参数，它被用来反序列化int字符串，缺省为int(num_str)
  
* parse_constant  
  如果指定parse_constant参数，它将被用来返序列化-Infinity, Infinity, NaN, null, true, false.
  
##示例
1、编码Python基本类型
	
	>>> import json
    >>> json.dumps(['foo', {'bar': ('baz', None, 1.0, 2)}])
    '["foo", {"bar": ["baz", null, 1.0, 2]}]'
    >>> print json.dumps("\"foo\bar")
    "\"foo\bar"
    >>> print json.dumps(u'\u1234')
    "\u1234"
    >>> print json.dumps('\\')
    "\\"
    >>> print json.dumps({"c": 0, "b": 0, "a": 0}, sort_keys=True)
    {"a": 0, "b": 0, "c": 0}
    >>> from StringIO import StringIO
    >>> io = StringIO()
    >>> json.dump(['streaming API'], io)
    >>> io.getvalue()
    '["streaming API"]'   
    
2、压缩编码

	>>> import json
	>>> json.dumps([1,2,3,{'4': 5, '6': 7}], sort_keys=True, separators=(',',':'))
    '[1,2,3,{"4":5,"6":7}]'
   
3、优雅打印
	
	>>> import json
    >>> print json.dumps({'4': 5, '6': 7}, sort_keys=True,
    ...                  indent=4, separators=(',', ': '))
    {
        "4": 5,
        "6": 7
    }

4、反序列化JSON
	
	>>> import json
    >>> obj = [u'foo', {u'bar': [u'baz', None, 1.0, 2]}]
    >>> json.loads('["foo", {"bar":["baz", null, 1.0, 2]}]') == obj
    True
    >>> json.loads('"\\"foo\\bar"') == u'"foo\x08ar'
    True
    >>> from StringIO import StringIO
    >>> io = StringIO('["streaming API"]')
    >>> json.load(io)[0] == 'streaming API'
    True
    
5、定制JSON对象反序列化
	
	>>> import json
    >>> def as_complex(dct):
    ...     if '__complex__' in dct:
    ...         return complex(dct['real'], dct['imag'])
    ...     return dct
    ...
    >>> json.loads('{"__complex__": true, "real": 1, "imag": 2}',
    ...     object_hook=as_complex)
    (1+2j)
    >>> from decimal import Decimal
    >>> json.loads('1.1', parse_float=Decimal) == Decimal('1.1')
    True
    
6、定制JSON对象序列化
	
	>>> import json
    >>> def encode_complex(obj):
    ...     if isinstance(obj, complex):
    ...         return [obj.real, obj.imag]
    ...     raise TypeError(repr(o) + " is not JSON serializable")
    ...
    >>> json.dumps(2 + 1j, default=encode_complex)
    '[2.0, 1.0]'
    >>> json.JSONEncoder(default=encode_complex).encode(2 + 1j)
    '[2.0, 1.0]'
    >>> ''.join(json.JSONEncoder(default=encode_complex).iterencode(2 + 1j))
    '[2.0, 1.0]'
    
7、使用json.tool校验和打印json
	
	$ echo '{"json":"obj"}' | python -m json.tool
    {
        "json": "obj"
    }
    $ echo '{ 1.2:3.4}' | python -m json.tool
    Expecting property name enclosed in double quotes: line 1 column 3 (char 2)
    
    
