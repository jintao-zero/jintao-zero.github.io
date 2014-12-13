---
layout: post
title: "CentOS 6.5 安装ip netns"
date: 2014-12-11 16:42:57 +0800
comments: true
categories: ['CentOS'] 
---
项目需要用到网络空间这一个技术，但是在CentOS 6.x版本中，ip命令并不能够使用netns这个参数：    

	[root@gaichao ~]# ip netns
	Object "netns" is unknown, try "ip help".
	[root@gaichao ~]#
下面分别介绍以下安装失败与成功的经验：
<!-- more -->  
##安装ip netns失败
根据google指引，找到了这个方法[OpenVNet test tools](https://github.com/amotoki/openvnet-test-tools/blob/master/README.md) ,也许是由于时间的推移，有的依赖做了改变，导致我在用文中方法安装netns时没有成功，下面把错误信息做一下记录：  

CentOS 6.5内核本身是支持网络空间的，只不过系统自带的iproute不支持网络空间(没有netns子命令)

根据指导，需要运行以下命令安装一个支持网络空间的新版本iproute：  

	# yum install http://rdo.fedorapeople.org/rdo-release.rpm
	# yum install iproute
运行第一个命令安装OpenStack RDO发布仓库到/etc/yum.repos.d目录，这个目录里面是yum命令安装、更新软件时需要用到的信息，第一步可以执行成功。    

运行第二个命令升级iproute，报下面的错误：  

	[root@gaichao ~]# yum install iproute
	Loaded plugins: fastestmirror, refresh-packagekit, security
	Loading mirror speeds from cached hostfile
 	* base: mirrors.pubyun.com
 	* epel: mirror01.idc.hinet.net
 	* extras: mirrors.btte.net
 	* updates: mirrors.btte.net
	http://repos.fedorapeople.org/repos/openstack/	openstack-juno/epel-6/repodata/repomd.xml: [Errno 14] PYCURL ERROR 22 - "The requested URL 	returned error: 404 Not Found"
	Trying other mirror.
	Error: Cannot retrieve repository metadata (repomd.xml) for repository: openstack-juno. 	Please verify its path and try again
	[root@gaichao ~]#  
		
根据错误提示，发现报错的文件不存在，路径已经更新为https://repos.fedorapeople.org/repos/openstack/openstack-juno/epel-7/repodata/ 所以尝试修改/etc/yum.repos.d/rdo-release.repo文件，将baseurl修改为当前可以找到repomd.xml文件的路径：  
	
	[openstack-juno]
	name=OpenStack Juno Repository
	baseurl=http://repos.fedorapeople.org/repos/openstack/openstack-juno/epel-7/
	enabled=1
	skip_if_unavailable=0
	gpgcheck=1
	gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-RDO-Juno
	
运行安装命令成功  

	[root@gaichao ~]# yum install iproute

但是查看iproute安装包，并没有按照设想安装带netns功能的iproute版本  

	[root@gaichao ~]# rpm -qa | grep iproute
	iproute-2.6.32-33.el6_6.x86_64
	[root@gaichao ~]#
	
实际应该安装到这个版本iproute-2.6.32-130.el6ost.netns.2.x86_64  
后来又尝试把iproute-2.6.32-130.el6ost.netns.2.x86_64 rpm包下载到本地进行安装，但是也没有成功
##安装ip netns 成功
继续网络中搜索，发现[德哥@Digoal的博客](http://digoal126.wap.blog.163.com/w2/blogDetail.do;jsessionid=C9F7702BA05F4C979FFF72779D38F7D9.blog84-8010?blogId=fks_087070086086089066083081080068072087087064092081081067080086&showRest=true&p=5&hostID=digoal@126)的方法是可行的  
根据文中的提示juno下已经没有epel6目录了，所以可以选择havana，路径如下：https://repos.fedorapeople.org/repos/openstack/openstack-havana/epel-6/ 需要修改/etc/yum.repos.d/rdo-release.repo中的baseurl:

	[openstack-juno]
	name=OpenStack Juno Repository
	baseurl=http://repos.fedorapeople.org/repos/openstack/openstack-havana/epel-6/
	enabled=1
	skip_if_unavailable=0
	gpgcheck=1
	gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-RDO-Juno

因为系统已经按照了iproute包，所以可以执行升级命令，进行升级：
	
	yum upgrade iproute	

执行成功后，运行以下命令查看iproute是否已经升级成为带netns命令

	# rpm -qa | grep iproute
	iproute-2.6.32-130.el6ost.netns.2.x86_64
	# ip netns
	# ip netns add test
	# ip netns
	test
	# ip netns delete test
	
笔者安装成功，主要参考上面两个链接的内容，感谢