---
layout: post
title: "python property 属性用法"
date: 2015-04-05 17:26:05 +0800
comments: true
categories: DevOps
tags: Python
---
python提供了一个property类：

	class property([fget[, fset[, fdel[, doc]]]])

property类为新式类（继承object）返回property属性  
fget函数用来获取属性值。fset设置属性值。fdel用来删除属性。doc为属性创建docstring。  
property的典型应用是应用一个被管理的属性x：  
	
	class C(object):
    def __init__(self):
        self._x = None

    def getx(self):
        return self._x

    def setx(self, value):
        self._x = value

    def delx(self):
        del self._x

    x = property(getx, setx, delx, "I'm the 'x' property.")

如果c是类C的实例，c.x将会调用getx，c.x＝value将会调用setx，del c.x会调用用delx  


将property 用作decorrator可以定义只读properties：

	class Parrot(object):
    	def __init__(self):
        	self._voltage = 100000

    	@property
    	def voltage(self):
        	"""Get the current voltage."""
        	return self._voltage
        	
@property修饰符将voltage函数变为voltage函数的同名属性的只读getter函数，同时这只这个属性的docstring与voltage函数相同

一个property对象具有getter，setter和deleter方法可以用当作修饰符将属性同名函数变为属性对应的访问函数，下面是例子：  
	
	class C(object):
    	def __init__(self):
        	self._x = None

	    @property
    	def x(self):
        	"""I'm the 'x' property."""
        	return self._x
		
		@x.setter
    	def x(self, value):
        	self._x = value

	    @x.deleter
    	def x(self):
        	del self._x
        	
上面的用法与第一个列子相同，需要注意的是几个函数的名字都相同

