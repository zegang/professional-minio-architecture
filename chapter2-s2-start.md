
# 第二章 Minio的启动

## 2.2 一切的开端
我们可以通过如下命令启动Minio服务。`minio`是可执行程序，可由Go源码编译生成。
```
./minio server /data
```
Go执行程序的开端是`main`包的`main`函数。Minio文件`minio/main.go`中响应的`main`函数源码如下：
```
// Copyright (c) 2015-2021 MinIO, Inc.
//
// This file is part of MinIO Object Storage stack
//
// This program is free software: you can redistribute it and/or modify
// it under the terms of the GNU Affero General Public License as published by
// the Free Software Foundation, either version 3 of the License, or
// (at your option) any later version.
//
// This program is distributed in the hope that it will be useful
// but WITHOUT ANY WARRANTY; without even the implied warranty of
// MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
// GNU Affero General Public License for more details.
//
// You should have received a copy of the GNU Affero General Public License
// along with this program.  If not, see <http://www.gnu.org/licenses/>.

package main // import "github.com/minio/minio"

import (
	"os"

	// MUST be first import.
	_ "github.com/minio/minio/internal/init"

	minio "github.com/minio/minio/cmd"
)

func main() {
	minio.Main(os.Args)
}
```
其主要有3步：
1. 引入了`github.com/minio/minio/internal/init`包，这个包的作用是初始化一些全局变量，比如日志系统，配置系统等。
2. 以别名`minio`引入了`github.com/minio/minio/cmd`包，这个包是真正的命令行处理包，它包含了所有的命令行处理函数。
3. 调用了`minio.Main`函数，这个函数是真正的入口函数，它接受一个参数，是命令行参数。

### 2.2.1 Internal包的初始化
`minio/internal/init`包中有2个文件: `init_darwin_adm64.go`, `init.go`。主要是时间的配置。
- `init.go`文件中定义了`init`函数: 配置使用UTC时间。
```
package init

import "os"

func init() {
	// All MinIO operations must be under UTC.
	os.Setenv("TZ", "UTC")
}
```
- `init_darwin_adm64.go`文件中定义了`init`函数: 配置使用UTC时间, 零实行的Bug修复(https://github.com/golang/go/issues/49233)。

### 2.2.2 `minio.Main`函数
`minio/cmd/main.go`文件中定义了`Main`函数，它是真正的入口函数，它接受一个参数，是命令行参数。首先，它获取了命令行参数的第一个参数，作为app的名字。然后，调用了`newApp`函数，创建了一个app对象。最后，调用了app对象的`Run`方法，传入了命令行参数。如果`Run`方法返回了错误，那么就退出程序。
```
// Main main for minio server.
func Main(args []string) {
	// Set the minio app name.
	appName := filepath.Base(args[0])
...
	// Run the app - exit on error.
	if err := newApp(appName).Run(args); err != nil {
		os.Exit(1) //nolint:gocritic
	}
}
```

### 2.2.3 `newApp`函数与命令集
`newApp`函数创建了一个app对象，这个对象包含了所有的命令集。
```
func newApp(name string) *cli.App {
	// Collection of minio commands currently supported are.
	commands := []cli.Command{}

	// Collection of minio commands currently supported in a trie tree.
	commandsTree := trie.NewTrie()
...
	// Register all commands.
	registerCommand(serverCmd)
	registerCommand(gatewayCmd) // hidden kept for guiding users.
...
	app := cli.NewApp()
	app.Name = name
	app.Author = "MinIO, Inc."
...
	app.Commands = commands
	app.CustomAppHelpTemplate = minioHelpTemplate
...
	return app
}
```
其中最重要的是注册了:
- `serverCmd`命令，这个命令是真正的Minio服务命令。
- `gatewayCmd`命令，这个命令是Minio网关命令。

### 2.2.4 `serverCmd`服务命令
Minio的服务命令`serverCmd`定义于`minio/cmd/server-main.go`文件中。
 ```
 var serverCmd = cli.Command{
	Name:   "server",
	Usage:  "start object storage server",
	Flags:  append(ServerFlags, GlobalFlags...),
	Action: serverMain,
	CustomHelpTemplate: `NAME:
  {{.HelpName}} - {{.Usage}}
...
}
```
`serverCmd`命令的`Action`是`serverMain`函数，这个函数是真正的Minio服务命令的入口函数。
```
// serverMain handler called for 'minio server' command.
func serverMain(ctx *cli.Context) {
...
	<-globalOSSignalCh
}
```