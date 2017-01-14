---
layout: post
title: "golang go命令详解"
date: 2017-01-13 19:17:12 +0800
comments: true
categories: program
tags: Golang
---
#go命令
[Compile packages and dependencies](#compile)      
[Remove object files](#clean)  
[Description of testing flags](#testing)

`Go`工具用来管理Go语音源代码。
使用：  

	go command [arguments]

命令为：

	build       compile packages and dependencies
	clean       remove object files
	doc         show documentation for package or symbol
	env         print Go environment information
	fix         run go tool fix on packages
	fmt         run gofmt on package sources
	generate    generate Go files by processing source
	get         download and install packages and dependencies
	install     compile and install packages and dependencies
	list        list packages
	run         compile and run Go program
	test        test packages
	tool        run specified go tool
	version     print Go version
	vet         run go tool vet on packages
使用`go help [command]`查看命令相信信息。  
更多topic：  
	
	c           calling between Go and C
	buildmode   description of build modes
	filetype    file types
	gopath      GOPATH environment variable
	environment environment variables
	importpath  import path syntax
	packages    description of package lists
	testflag    description of testing flags
	testfunc    description of testing functions
使用`go help [topic]`查看topic帮助信息  

<!-- more -->

##<a name="compile"></a>编译包及其依赖  
使用：  
	
	go build [-o output] [-i] [build flags] [packages]

`Build`编译包，及其依赖，但是不进行安装。    

如果输入的是一些.go文件，build认为是编译由这些文件组成的单独包。 
当单独编译main包时，build将可执行程序写入输出文件，输出文件以列出的第一个.go源文件名称命名，或者以main包源文件所在的文件夹名称命名，Windows环境下，可执行成后后缀为.exe。  

当编译多个包或者一个非main包，build命令只是编译这些包，丢弃结果文件，只是查看包是否能够编译成功。

编译包时，build忽略以`_test.go`为后缀的文件。  

`-o`参数，可以用来指定编译结果存放路径。  
`-i`参数，设置将目标程序依赖的包进行安装。  

build 标志是`build,clean,get,install,list,run,test`共享的：  

    -a
        force rebuilding of packages that are already up-to-date.
    -n
        print the commands but do not run them.
    -p n
        the number of programs, such as build commands or
        test binaries, that can be run in parallel.
        The default is the number of CPUs available.
    -race
        enable data race detection.
        Supported only on linux/amd64, freebsd/amd64, darwin/amd64 and windows/amd64.
    -msan
        enable interoperation with memory sanitizer.
        Supported only on linux/amd64,
        and only with Clang/LLVM as the host C compiler.
    -v
        print the names of packages as they are compiled.
    -work
        print the name of the temporary work directory and
        do not delete it when exiting.
    -x
        print the commands.

    -asmflags 'flag list'
        arguments to pass on each go tool asm invocation.
    -buildmode mode
        build mode to use. See 'go help buildmode' for more.
    -compiler name
        name of compiler to use, as in runtime.Compiler (gccgo or gc).
    -gccgoflags 'arg list'
        arguments to pass on each gccgo compiler/linker invocation.
    -gcflags 'arg list'
        arguments to pass on each go tool compile invocation.
    -installsuffix suffix
        a suffix to use in the name of the package installation directory,
        in order to keep output separate from default builds.
        If using the -race flag, the install suffix is automatically set to race
        or, if set explicitly, has _race appended to it.  Likewise for the -msan
        flag.  Using a -buildmode option that requires non-default compile flags
        has a similar effect.
    -ldflags 'flag list'
        arguments to pass on each go tool link invocation.
    -linkshared
        link against shared libraries previously created with
        -buildmode=shared.
    -pkgdir dir
        install and load all packages from dir instead of the usual locations.
        For example, when building with a non-standard configuration,
        use -pkgdir to keep generated packages in a separate location.
    -tags 'tag list'
        a list of build tags to consider satisfied during the build.
        For more information about build tags, see the description of
        build constraints in the documentation for the go/build package.
    -toolexec 'cmd args'
        a program to use to invoke toolchain programs like vet and asm.
        For example, instead of running asm, the go command will run
        'cmd args /path/to/asm <arguments for asm>'.
上面列的参数标志接收以空格为分隔符的字符串列表。如果参数中包含空格，那么需要将参数用单引号或者双引号包裹。  

使用`go help packages`查看更多关于包的信息，使用`go help gopath`查看更多关于编译和安装路径的信息，使用`go help c`查看更多关于Go和C/C++调用的帮助信息。  

`Build`依附于`go help gopath`中描述的惯例。并不是所有项目遵从惯例。使用不同构建惯例或者构建系统可能会选择使用更低层的构建工具如`go tool compile`和`go tool link`。 

##<a name="clean"></a>删除目标文件  
使用：  

    go clean [-i] [-r] [-n] [-x] [build flags] [packages]

将包源代码路径中的目标文件删除。`go`命令通常在临时目录构建目标对象，所以`go clean`通常用来删除其他工具或者手动执行`go build`生成的结果文件。  

特别的，clena命令将包源码路径下生成的如下文件删除：  

    _obj/            old object directory, left from Makefiles
    _test/           old test directory, left from Makefiles
    _testmain.go     old gotest file, left from Makefiles
    test.out         old test log, left from Makefiles
    build.out        old test log, left from Makefiles
    *.[568ao]        object files, left from Makefiles

    DIR(.exe)        from go build
    DIR.test(.exe)   from go test -c
    MAINFILE(.exe)   from go build MAINFILE.go
    *.so             from SWIG

`-i`参数指定go clean 命令将go install命令安装的包文件或者二进制文件删除。   
`-n`参数指定go clean把要执行的删除命令打印，但是不实际执行。  
`-r`参数指定go clean对于指定包的所有依赖进行go clean命令。  
`-x`参数指定go clean在执行删除操作时将命令打印。  



##<a name="testing"></a>Description of testing flags

    -bench regexp
        Run (sub)benchmarks matching a regular expression.
        The given regular expression is split into smaller ones by top-level '/', where each must match the corresponding part of a benchmark's identifier.
        By default, no benchmarks run. To run all benchmarks,
        use '-bench .' or '-bench=.'.

    -benchmem
        Print memory allocation statistics for benchmarks.

    -benchtime t
        Run enough iterations of each benchmark to take t, specified as a time.Duration (for example, -benchtime 1h30s).
        The default is 1 second (1s).

    -blockprofile block.out
        Write a goroutine blocking profile to the specified file when all tests are complete.
        Writes test binary as -c would.

    -blockprofilerate n
        Control the detail provided in goroutine blocking profiles by calling runtime.SetBlockProfileRate with n.
        See 'go doc runtime.SetBlockProfileRate'.
        The profiler aims to sample, on average, one blocking event every n nanoseconds the program spends blocked.  By default, if -test.blockprofile is set without this flag, all blocking events are recorded, equivalent to -test.blockprofilerate=1.

    -count n
        Run each test and benchmark n times (default 1).
        If -cpu is set, run n times for each GOMAXPROCS value.
        Examples are always run once.

    -cover
        Enable coverage analysis.

    -covermode set,count,atomic
        Set the mode for coverage analysis for the package[s] being tested. The default is "set" unless -race is enabled, in which case it is "atomic".
        The values:
            set: bool: does this statement run?
            count: int: how many times does this statement run?
            atomic: int: count, but correct in multithreaded tests; significantly more expensive.
        Sets -cover.
    
    -coverpkg pkg1,pkg2,pkg3
        Apply coverage analysis in each test to the given list of packages.
        The default is for each test to analyze only the package being tested.
        Packages are specified as import paths.
        Sets -cover.

    -coverprofile cover.out
        Write a coverage profile to the file after all tests have passed.
        Sets -cover.

    -cpu 1,2,4
        Specify a list of GOMAXPROCS values for which the tests or benchmarks should be executed.  The default is the current value of GOMAXPROCS.

    -cpuprofile cpu.out
        Write a CPU profile to the specified file before exiting.
        Writes test binary as -c would.

    -memprofile mem.out
        Write a memory profile to the file after all tests have passed.
        Writes test binary as -c would.

    -memprofilerate n
        Enable more precise (and expensive) memory profiles by setting runtime.MemProfileRate.  See 'go doc runtime.MemProfileRate'.
        To profile all memory allocations, use -test.memprofilerate=1 and pass --alloc_space flag to the pprof tool.

    -outputdir directory
        Place output files from profiling in the specified directory, by default the directory in which "go test" is running.

    -parallel n
        Allow parallel execution of test functions that call t.Parallel. The value of this flag is the maximum number of tests to run simultaneously; by default, it is set to the value of GOMAXPROCS.
        Note that -parallel only applies within a single test binary.
        The 'go test' command may run tests for different packages in parallel as well, according to the setting of the -p flag
        (see 'go help build').

    -run regexp
        Run only those tests and examples matching the regular expression.
        For tests the regular expression is split into smaller ones by top-level '/', where each must match the corresponding part of a test's identifier.

    -short
        Tell long-running tests to shorten their run time.
        It is off by default but set during all.bash so that installing the Go tree can run a sanity check but not spend time running exhaustive tests.

    -timeout t
        If a test runs longer than t, panic. The default is 10 minutes (10m).

    -trace trace.out
        Write an execution trace to the specified file before exiting.
    
    -v
        Verbose output: log all tests as they are run. Also print all text from Log and Logf calls even if the test succeeds.



