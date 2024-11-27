# build包解析


## 说明

构建工具
- ci.go
  - build用于编译项目构建工具
  - test，跑全部单元测试
  - lint，验证所有代码格式
- update-license.go
  - 用于更新所有代码头部license注释



## 代码分析


### 更新license
update-license.go

整体是变量声明，一些需要扫描的文件后缀、忽略的文件夹、授权前缀、正则匹配正则表达式、替换更新的模版

定义一个info结构体，输入文件名和year，用于生成替换模版的变量

核心代码

获取所有文件
写入作者
根据cpu核数
并发获取文件
替换文件，前缀





```

func main() {
// 获取文件
	var (
		files = getFiles()
		filec = make(chan string)
		infoc = make(chan *info, 20)
		wg    sync.WaitGroup
	)
// git获取作者，写入
	writeAuthors(files)

	go func() {
		for _, f := range files {
			filec <- f
		}
		close(filec)
	}()
	// 并发更新
	for i := runtime.NumCPU(); i >= 0; i-- {
		// getting file info is slow and needs to be parallel.
		// it traverses git history for each file.
		wg.Add(1)
		go getInfo(filec, infoc, &wg)
	}
	go func() {
		wg.Wait()
		close(infoc)
	}()
	writeLicenses(infoc)
}




```


### 编译项目


ci.go 

test 进行单元测试
line 通过golangci跑line
install 遍历cmd下面有main的包，进行 install 到bin目录

核心代码 

```


根据不同命令参数，走不同逻辑，核心是install 逻辑

func main() {
	log.SetFlags(log.Lshortfile)

	if !build.FileExist(filepath.Join("build", "ci.go")) {
		log.Fatal("this script must be run from the root of the repository")
	}
	if len(os.Args) < 2 {
		log.Fatal("need subcommand as first argument")
	}
	switch os.Args[1] {
	case "install":
		doInstall(os.Args[2:])
	case "test":
		doTest(os.Args[2:])
	case "lint":
		doLint(os.Args[2:])
	case "check_tidy":
		doCheckTidy()
	case "check_generate":
		doCheckGenerate()
	case "check_baddeps":
		doCheckBadDeps()
	case "archive":
		doArchive(os.Args[2:])
	case "dockerx":
		doDockerBuildx(os.Args[2:])
	case "debsrc":
		doDebianSource(os.Args[2:])
	case "nsis":
		doWindowsInstaller(os.Args[2:])
	case "purge":
		doPurge(os.Args[2:])
	case "sanitycheck":
		doSanityCheck()
	default:
		log.Fatal("unknown command ", os.Args[1])
	}
}



遍历cmd下，找main package，然后install到bin目录

	// Now we choose what we're even building.
	// Default: collect all 'main' packages in cmd/ and build those.
	packages := flag.Args()
	if len(packages) == 0 {
		packages = build.FindMainPackages("./cmd")
	}

	// Do the build!
	for _, pkg := range packages {
		args := slices.Clone(gobuild.Args)
		args = append(args, "-o", executablePath(path.Base(pkg)))
		args = append(args, pkg)
		build.MustRun(&exec.Cmd{Path: gobuild.Path, Args: args, Env: gobuild.Env})
	}



```

