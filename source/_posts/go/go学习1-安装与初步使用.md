---
title: go学习1-安装与初步使用
date: 2017-09-25 15:44:32
categories:
reward: false
tags:
     - blog
     - go
     - 高并发
---

## 一、下载安装
从[go主页](https://golang.org/dl/)，下载对应平台的安装包；下面以windows环境安装1.9版本示例。

+ 打开下载的MSI文件，并按照提示进行安装go工具
+ 如果第一步安装使用的是默认安装目录 c:\Go\，那么安装程序就已经将 GOROOT 和 Path 两个环境变量设置好了，无须再对其进行手工设置。
+ 如果修改默认安装路径，则需要修改环境变量GOROOT和PATH内go的路径。

## 二、一些环境变量

### GOROOT
GOROOT就是go的安装路径，默认为c:\Go\。

### GOBIN
go install编译存放路径。不允许设置多个路径；一般设置为%GOPATH%\bin或%GOROOT%\bin。

### PATH
系统环境变量，设置后可以直接执行go命令，而不用带上完整路径；一般将GOBIN和GOROOT\bin目录添加进去；

### GOPATH
+ GOPATH是用来设置包加载路径的重要变量。可以设置多个路径，windows下用分号(;)分隔
+ GOPATH是作为编译后二进制的存放目的地和import包时的搜索路径 (其实也是你的工作目录, 你可以在src下创建你自己的go源文件, 然后开始工作)
+ GOPATH之下主要包含三个目录: bin、pkg、src
+ bin目录主要存放可执行文件; pkg目录存放编译好的库文件, 主要是*.a文件; src目录下主要存放go的源文件
+ 不要把GOPATH设置成go的安装路径
+ GOPATH可以是一个目录列表, go get下载的第三方库, 一般都会下载到列表的第一个目录里面
+ 需要把GOPATH中的可执行目录也配置到环境变量中, 否则你自行下载的第三方go工具就无法使用了

<!--more-->

## 三、编写第一个Go程序
创建hello.go 文件并编辑其内容如下：
``` go
package main  
  
import "fmt"  
  
func main() {  
    fmt.Printf("hello, world\n")  
}  
```

保存后进入该目录，执行 go run hello.go

看到 "hello, world" 证明我们的 go 安装成功了。

## 四、go命令解释

### go build
这个命令主要用于测试编译。在包的编译过程中，若有必要，会同时编译与之相关联的包。
+ 如果是普通包，当你执行go build之后，它不会产生任何文件。如果你需要在$GOPATH/pkg下生成相应的文件，那就得执行go install了。
+ 如果是main包，当你执行go build之后，它就会在当前目录下生成一个可执行文件。如果你需要在$GOPATH/bin下生成相应的文件，需要执行go install，或者使用go build -o 路径/a.exe。
+ 如果某个项目文件夹下有多个文件，而你只想编译某个文件，就可在go build之后加上文件名，例如go build a.go；go build命令默认会编译当前目录下的所有go文件。
+ 你也可以指定编译输出的文件名。我们可以指定go build -o astaxie.exe，默认情况是你的package名(非main包)，或者是第一个源文件的文件名(main包)。
+ go build会忽略目录下以“_”或“.”开头的go文件。
+ 如果你的源代码针对不同的操作系统需要不同的处理，那么你可以根据不同的操作系统后缀来命名文件。例如有一个读取数组的程序，它对于不同的操作系统可能有如下几个源文件（array_linux.go array_darwin.go array_windows.go array_freebsd.go），go build的时候会选择性地编译以系统名结尾的文件（Linux、Darwin、Windows、Freebsd）。例如Linux系统下面编译只会选择array_linux.go文件，其它系统命名后缀文件全部忽略。

### go clean
这个命令是用来移除当前源码包里面编译生成的文件。这些文件包括：
+ **_obj/**             旧的object目录，由Makefiles遗留
+ **_test/**            旧的test目录，由Makefiles遗留
+ **_testmain.go**      旧的gotest文件，由Makefiles遗留
+ **test.out**          旧的test记录，由Makefiles遗留
+ **build.out**         旧的test记录，由Makefiles遗留
+ **\*.[568ao]**        object文件，由Makefiles遗留

+ **DIR(.exe)**         由go build产生
+ **DIR.test(.exe)**    由go test -c产生
+ **MAINFILE(.exe)**    由go build MAINFILE.go产生

利用这个命令清除编译文件，然后github提交源码，在本机测试的时候这些编译文件都是和系统相关的，但是对于源码管理来说没必要。

### go fmt
有过C/C++经验的读者会知道,一些人经常为代码采取K&R风格还是ANSI风格而争论不休。在go中，代码则有标准的风格。由于之前已经有的一些习惯或其它的原因我们常将代码写成ANSI风格或者其它更合适自己的格式，这将为人们在阅读别人的代码时添加不必要的负担，所以go强制了代码格式（比如左大括号必须放在行尾），不按照此格式的代码将不能编译通过，为了减少浪费在排版上的时间，go工具集中提供了一个go fmt命令 它可以帮你格式化你写好的代码文件，使你写代码的时候不需要关心格式，你只需要在写完之后执行go fmt <文件名>.go，你的代码就被修改成了标准格式，但是我平常很少用到这个命令，因为开发工具里面一般都带了保存时候自动格式化功能，这个功能其实在底层就是调用了go fmt。

使用go fmt命令，更多时候是用go fmt，而且需要参数-w，否则格式化结果不会写入文件。go fmt -w src，可以格式化整个项目。

### go get
这个命令是用来动态获取远程代码包的，目前支持的有BitBucket、GitHub、Google Code和Launchpad。这个命令在内部实际上分成了两步操作：第一步是下载源码包，第二步是执行go install。下载源码包的go工具会自动根据不同的域名调用不同的源码工具，对应关系如下：
+ BitBucket (Mercurial Git)
+ GitHub (Git)
+ Google Code Project Hosting (Git, Mercurial, Subversion)
+ Launchpad (Bazaar)

所以为了go get 能正常工作，你必须确保安装了合适的源码管理工具，并同时把这些命令加入你的PATH中。其实go get支持自定义域名的功能，具体参见go help remote。

### go install
这个命令在内部实际上分成了两步操作：第一步是生成结果文件(可执行文件或者.a包)，第二步会把编译好的结果移到$GOPATH/pkg或者$GOPATH/bin。

### go test
执行这个命令，会自动读取源码目录下面名为*_test.go的文件，生成并运行测试用的可执行文件。输出的信息类似：
``` go
ok   archive/tar   0.011s
FAIL archive/zip   0.022s
ok   compress/gzip 0.033s
...
```
默认的情况下，不需要任何的参数，它会自动把你源码包下面所有test文件测试完毕，当然你也可以带上参数，详情请参考go help testflag

### go doc
(1.2rc1 以後沒有 go doc 指令, 只留下 godoc 指令) 很多人说go不需要任何的第三方文档，例如chm手册之类的（其实我已经做了一个了，chm手册），因为它内部就有一个很强大的文档工具。

如何查看相应package的文档呢？ 例如builtin包，那么执行go doc builtin 如果是http包，那么执行go doc net/http 查看某一个包里面的函数，那么执行godoc fmt Printf 也可以查看相应的代码，执行godoc -src fmt Printf

通过命令在命令行执行 godoc -http=:端口号 比如godoc -http=:8080。然后在浏览器中打开127.0.0.1:8080，你将会看到一个golang.org的本地copy版本，通过它你可以查询pkg文档等其它内容。如果你设置了GOPATH，在pkg分类下，不但会列出标准包的文档，还会列出你本地GOPATH中所有项目的相关文档，这对于经常被墙的用户来说是一个不错的选择。


### 其他
go还提供了其它很多的工具，例如下面的这些工具：
+ go fix 用来修复以前老版本的代码到新版本，例如go1之前老版本的代码转化到go1
+ go version 查看go当前的版本
+ go env 查看当前go的环境变量
+ go list 列出当前全部安装的package
+ go run 编译并运行Go程序
以上这些工具还有很多参数没有一一介绍，用户可以使用go help 命令获取更详细的帮助信息。

## 五、代码结构

### 基本原则
+ 将所有Go代码放在一个工作空间(workspace)下。
+ 一个工作空间包含多个代码仓库。
+ 每个仓库有一个或多个包（packages）。
+ 每个包又包含一个或多个Go代码源文件。
+ 包的路径在import中体现。

### 工作空间(workspace)
每个工作空间根目录下包括： 
- src Go源代码 
- src文件夹下放置多个版本控制的仓库（项目）。 
- pkg 包对象 
- bin 生成的可执行文件 
go命令编译源文件，并生产对应的结果文件到pkg和bin目录。

### 包路径
包路径用来唯一标记一个包，包路径反映了包在本地工作空间或远程代码库（接下来会详细说明）中的位置。

标准库中的包直接使用短包路径，比如“fmp”或“net/http”。但是对于自己编写的包，基础路径的名字不能与标准库和其它一些外部库冲突。

如果你的代码直接保存在一些版本控制的代码库里，应该直接使用代码库的根目录作为基本路径。比如，如果你的代码都存放在github的user用户下（github.com/user），那github.com/user就应该是基本路径。

即使你现在还没有打算把代码发布带远程代码仓库，最好还是按照以后准备提交代码仓库的方式组织代码结构。实际中可以使用任何组合来标记路径名称，只要对应的路径名称不与基本库冲突。

### 包命名
所有Go源代码都以下边的语句开始：package name

其中name就是包引用是默认的名称。一个包中的所有文件必须使用同一个包名。

可执行命令的包名必须是main。

一个二进制文件下所有的包名不需要唯一，但是引用路径必须唯一。

### 测试
Go自带了一个轻量级的测试框架，由go test命令和testing包组成。

你可以通过新建xx_test.go写一个测试，其中包含若干个TestXXX函数。测试框架会自动执行这些函数；如果函数中包含tError或t.Fail, 对应的测试会被判为失败。

如：
``` go
package stringutil

import "testing"

func TestReverse(t *testing.T) {
    cases := []struct {
        in, want string
    }{
        {"Hello, world", "dlrow ,olleH"},
        {"Hello, 世界", "界世 ,olleH"},
        {"", ""},
    }
    for _, c := range cases {
        got := Reverse(c.in)
        if got != c.want {
            t.Errorf("Reverse(%q) == %q, want %q", c.in, got, c.want)
        }
    }
}
```

然后通过go test执行测试：
``` go
go test github.com/user/stringutil
ok      github.com/user/stringutil 0.165s
```

同样，在包文件夹下可以忽略包路径而直接执行go test命令：
``` go
$ go test
ok      github.com/user/stringutil 0.165s
```

### 远程包
包的引用路径用来描述如何通过版本控制系统获取包的源代码。go工具通过引用路径自动从远程代码仓库获取包文件。比如本文中用的例子也对应的保存在github.com/golang/example下。go可以通过包的代码仓库的url直接获取、生成、安装对应的包。
``` go
$ go get github.com/golang/example/hello
$ $GOPATH/bin/hello
Hello, Go examples!
```
如果工作空间中不存在对应的包，go会将对应的包放到GOPATH环境变量指明的工作空间下。（如果包已经存在，go跳过代码拉去而直接执行go install）。


