---
layout: post
title: "golang cpu性能优化"
date: 2017-01-01 17:18:20 +0800
comments: true
categories: program
tags: golang
---
本文根据[Profiling Go Programs](https://blog.golang.org/profiling-go-programs)文章，进行演示如何利用Golang性能工具进行cpu性能统计和优化。  
##准备工作
1、Golang编译运行环境。

    go version go1.7 darwin/amd64

2、下载[测试源码](https://storage.googleapis.com/google-code-archive-source/v2/code.google.com/benchgraffiti/source-archive.zip)

    -rw-r--r-- 1 jintao staff 16594  1  1 16:58 havlak1.go
    -rw-r--r-- 1 jintao staff 16597  1  1 16:58 havlak2.go
    -rw-r--r-- 1 jintao staff 16832  1  1 16:58 havlak3.go
    -rw-r--r-- 1 jintao staff 16905  1  1 16:58 havlak4.go
    -rw-r--r-- 1 jintao staff 17501  1  1 16:58 havlak5.go
    -rw-r--r-- 1 jintao staff  9467  1  1 16:58 havlak6.go

havlak1-6是源文件，以及优化后的源文件

##采集CPU性能数据
优化程序之前，需要采集性能数据。多种方法可以使用，本文中采用的方法是引入`runtime/pprof`包，在代码文件main函数中添加如下代码片段:  

    var cpuprofile = flag.String("cpuprofile", "", "write cpu profile to file")

    func main() {
        flag.Parse()
        if *cpuprofile != "" {
            f, err := os.Create(*cpuprofile)
            if err != nil {
              log.Fatal(err)
            }
            pprof.StartCPUProfile(f)
            defer pprof.StopCPUProfile()
        }
        ...	
利用flag包设置并解析cpuprofile参数，传入文件名，创建文件，调用`ppfof.StartCPUProfile`方法开始进行采样，并将采样结果保存在文件中，在程序返回之前调用`pprof.StopCPUProfile`方法确保所有采样数据刷新到结果文件中。  

    MacBook-Pro-2:havlak_new jintao$ time ./havlak1 -cpuprofile=havlak1.prof
	# of loops: 76000 (including 1 artificial root node)

	real	0m21.768s
	user	0m31.026s
	sys	0m0.284s
产生性能文件havlak1.prof
<!-- more -->
##分析性能文件
`go tool pprof`是[Google's pprof C++ profiler](https://code.google.com/p/gperftools/wiki/GooglePerformanceTools)一个变种，利用go tool pprof命令读取分析性能文件:

    MacBook-Pro-2:havlak_new jintao$ go tool pprof havlak1 havlak1.prof
    Entering interactive mode (type "help" for commands)
    (pprof)

输入`help`可以查看哪些可用命令，最常用是`top n`命令，查看前n个样本：  

    (pprof) top
    18750ms of 27430ms total (68.36%)
    Dropped 95 nodes (cum <= 137.15ms)
    Showing top 10 nodes out of 80 (cum >= 790ms)
          flat  flat%   sum%        cum   cum%
        3360ms 12.25% 12.25%     6790ms 24.75%  runtime.scanobject
        2890ms 10.54% 22.79%    14430ms 52.61%  main.FindLoops
        2470ms  9.00% 31.79%     2760ms 10.06%  runtime.mapaccess1_fast64
        1950ms  7.11% 38.90%     5140ms 18.74%  runtime.mapassign1
        1690ms  6.16% 45.06%     4080ms 14.87%  runtime.mallocgc
        1630ms  5.94% 51.00%     1630ms  5.94%  runtime.heapBitsForObject
        1580ms  5.76% 56.76%     3440ms 12.54%  main.DFS
        1250ms  4.56% 61.32%     1250ms  4.56%  runtime.memmove
        1140ms  4.16% 65.48%     1890ms  6.89%  runtime.greyobject
         790ms  2.88% 68.36%      790ms  2.88%  runtime/internal/atomic.Or8
    (pprof)

启动性能分析时，Go程序每秒钟停止100次并对当前正在执行的goroutine调用栈进行采样。从上面数据可以看到，程序总执行时间为27430ms，采样top10函数一共占用18750ms（68.36%）。每行是一个函数的统计数据，前两列数据分别为采样时goroutine正在当前函数中的时间和占比，后两列为采样时此函数出现（正在执行或正在等待调用函数返回）的时间和占比。sum%列为前n行消耗时间之和对于总时间的占比。`main.FindLoops`函数正在执行的时间是2890ms占10.54%，在调用栈中出现的时间为14430ms占比为52.61%。`runtime.mapaccess1_fast64`执行的时间为2470ms占9.00%，在调用栈中出现的时间为2760ms占比为10.06%。使用-cum参数，按照第四第五列排序  

    (pprof) top5 -cum
    2.89s of 27.43s total (10.54%)
    Dropped 95 nodes (cum <= 0.14s)
    Showing top 5 nodes out of 80 (cum >= 14.43s)
      flat  flat%   sum%        cum   cum%
         0     0%     0%     22.79s 83.08%  runtime.goexit
         0     0%     0%     14.52s 52.93%  main.main
         0     0%     0%     14.52s 52.93%  runtime.main
         0     0%     0%     14.43s 52.61%  main.FindHavlakLoops
     2.89s 10.54% 10.54%     14.43s 52.61%  main.FindLoops
    (pprof)

调用栈采样数据中关于函数间的调用关系可以有其他的有趣方式进行展现。比如`web`命令输出一个图片并用浏览器打开。`gv`命令写PostScript并在Ghostview中打开。  

    (pprof) web
    (pprof)
以下为图片部分截图：   
![web](/images/havlak1.png)
    图中每个方块对应一个单独函数，方块的大小与函数消耗的时间相对应。从X到Y的边显示X调用Y；边上的数字代表在被调用函数中消耗的时间。从图中我们可以发现在runtime.mapaccess1_fast64和runtime.mapassign1函数上消耗了较多时间。  
    可以只显示包含某个函数的调用关系图：  
        
    (pprof) web mapaccess
    (pprof)

![mapaccess](/images/havlak1-mapaccess.png)  
从上图我们可以发现主要是main.DFS和main.FindLoops函数调用了runtime.mapaccess  
接下来重点分析`main.DFS`和`main.FindLoops`两个函数的时间消耗情况：  

    (pprof) list DFS
    Total: 27.43s
    ROUTINE ======================== main.DFS in /Users/jintao/Project/opensource/benchgraffiti/havlak_new/havlak1.go
         1.58s      6.84s (flat, cum) 24.94% of Total
             .          .    235:	return false
             .          .    236:}
             .          .    237:
             .          .    238:// DFS - Depth-First-Search and node numbering.
             .          .    239://
          20ms       20ms    240:func DFS(currentNode *BasicBlock, nodes []*UnionFindNode, number map[*BasicBlock]int, last []int, current int) int {
         140ms      140ms    241:	nodes[current].Init(currentNode, current)
             .      310ms    242:	number[currentNode] = current
             .          .    243:
             .          .    244:	lastid := current
            1s         1s    245:	for _, target := range currentNode.OutEdges {
         160ms      1.31s    246:		if number[target] == unvisited {
          40ms      3.44s    247:			lastid = DFS(target, nodes, number, last, lastid+1)
             .          .    248:		}
             .          .    249:	}
         200ms      600ms    250:	last[number[currentNode]] = lastid
          20ms       20ms    251:	return lastid
             .          .    252:}
             .          .    253:
             .          .    254:// FindLoops
             .          .    255://
             .          .    256:// Find loops and build loop forest using Havlak's algorithm, which
    (pprof)
`list DFS` 会列出所有匹配DFS函数名的函数。从上面的代码我们发现耗时的语句分别在`242 246 247 250`行其中 247行是与DFS行数调用有关，其他三行都是与number变量有关，number是一个map数据结构，可以考虑改为采用slice，使用block number作为索引。  

对文件进行修改，diff修改如下：  

    240c240
    < func DFS(currentNode *BasicBlock, nodes []*UnionFindNode, number map[*BasicBlock]int, last []int, current int) int {
    ---
    > func DFS(currentNode *BasicBlock, nodes []*UnionFindNode, number []int, last []int, current int) int {
    242c242
    < 	number[currentNode] = current
    ---
    > 	number[currentNode.Name] = current
    246c246
    < 		if number[target] == unvisited {
    ---
    > 		if number[target.Name] == unvisited {
    250c250
    < 	last[number[currentNode]] = lastid
    ---
    > 	last[number[currentNode.Name]] = lastid
    271c271
    < 	number := make(map[*BasicBlock]int)
    ---
    > 	number := make([]int, size)
    287c287
    < 		number[bb] = unvisited
    ---
    > 		number[bb.Name] = unvisited
    315c315
    < 				v := number[nodeV]
    ---
    > 				v := number[nodeV.Name]

测试修改后的cpu性能：

	MacBook-Pro-2:havlak_new jintao$ go build havlak2.go
	MacBook-Pro-2:havlak_new jintao$ time ./havlak2 -	cpuprofile=havlak2.prof
	# of loops: 76000 (including 1 artificial root node)

	real	0m11.685s
	user	0m19.647s
	sys	0m0.268s

再次使用go tool profile工具查看topn数据：  

	MacBook-Pro-2:havlak_new jintao$ go tool pprof havlak2 havlak2.prof
	Entering interactive mode (type "help" for commands)
	(pprof) top
	11460ms of 16610ms total (68.99%)
	Dropped 92 nodes (cum <= 83.05ms)
	Showing top 10 nodes out of 87 (cum >= 540ms)
   	   flat  flat%   sum%        cum   cum%
    2950ms 17.76% 17.76%     5630ms 33.90%  runtime.scanobject
    1500ms  9.03% 26.79%     4100ms 24.68%  runtime.mallocgc
    1260ms  7.59% 34.38%     1260ms  7.59%  runtime.heapBitsForObject
    1250ms  7.53% 41.90%     8700ms 52.38%  main.FindLoops
     920ms  5.54% 47.44%     1450ms  8.73%  runtime.greyobject
     920ms  5.54% 52.98%     2120ms 12.76%  runtime.mapassign1
     770ms  4.64% 57.62%      780ms  4.70%  runtime.heapBitsSetType
     760ms  4.58% 62.19%      760ms  4.58%  runtime.memmove
     590ms  3.55% 65.74%     1500ms  9.03%  runtime.makemap
     540ms  3.25% 68.99%      540ms  3.25%  runtime/internal/atomic.Or8
	(pprof)

从上面可以看到`main.DFS`已经不在topn列表上，其他函数的时间也在显著下降。现在累计有24.68%的时间用在分配内存和垃圾回收（`runtime.mallocgc`）。下面需要使用进行内存性能优化。  

##参考
1、[Golang官方博客](https://blog.golang.org/profiling-go-programs)  
2、[C++ pprof](https://code.google.com/p/gperftools/wiki/GooglePerformanceTools)
