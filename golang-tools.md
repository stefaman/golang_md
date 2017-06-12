## doc comment
comment precede before export identifiers
only one package comment, 如果很长可以独立一个注释文件，如 "doc.go", like
`/*
comment
*/
pacakge fmt
`
// BUG(rsc): 在godoc 上显示Bugs条目
### go doc
不区分大小写

### godoc
godoc 可以作为web server, 包含了GOPATH AND GOROOT
GOPATH= godoc -http=:6060  忽略GOPATH的影响
Its -analysis=type and -analysis=pointer flags augment the documentation and the
source code wit h the results of advance d st atic analysis.

## path
import "import path"
go <tool> <import path>
import "path" 的解读不是语言规范，而是工具的规定。下面讨论的都是标准工具。
path是现对于GoPath 路径的，表示一个目录，导入目录下面的包， 包的名字不一定与目录同名，虽然不推荐这样做，会引起混淆。

import path的解读体现了Go 依赖convention的一面。

go get gopl.io/...
go get -u github.com/golang/lint/golint
上述命令首先fetch http://gopl.io?go-get=1;
返回的html包含“<meta name="go-import" content="gopl.io git https://github.com/adonovan/gopl.io">”；里面有代码仓库真正所在。
而且会递归下载源代码的依赖包；自动build and install;
`-u` 选项下载所有最新的依赖包
`...`下载所有subtree

go help importpaht
go help gopath

以上所说的是import path, 由go tool 解读。也可以直接指定目录，使用`./???` or `../???`的相对路径形式；如果没有指定目录，默认当前目录。
once experiment or for test, a package can be a list of go files.

## 依赖、版本、更新

### internal package
如果的包的import path has a segment called "internal", then this package can only imported by package located in the same root path before "internal".
for example, the package "net/http/internal/chunked" can be imported by "net/http", "net/http/httputil", but can not by "net/url".
## 包的别名
1. 避免命名冲突
2. `_` 只是为了运行导入包的init函数， 如image/... 下面的包是典型，导入包是为了注册某些资源。

## go list
显示包的import path, importing packages(直接import的包), dependencies packages(递归的完整的依赖),包含的文件等信息
go list ...xml...  实现了搜索
go list ... 显示所有包
go list net/... 显示所有subtree
go list -json
go list -f 可以格式化显示信息

## compile
都是静态编译
- go build
会检查编译错误和依赖；最终只会保留执行文件，中间文件均丢弃。
执行文件存放在当前目录。
go build -i installs the packages that are dependencies of the build target.

- go install
$GOPATH/pkg 存放包编译文件
$GOPATH/bin 存放执行文件
后续的go build and go install 不会重新编译未更新的package or command. 此次更新的检查？？？

- cross-compile
set GOOS and GOARCH，指定需要的编译的目标操作系统和CPU，经测试linux/windows可以互相指定编译。

条件编译：
通过文件命名指定针对的操作系统，如 main_linux.go， main_windows.go
通过文件注释指定针对的目标系统，如
.go file 一开始的`//+build linux darwin`表示只针对类此的目标系统；`//+build ignore`表示不用编译该文件
see “build constraints” go doc go/build
