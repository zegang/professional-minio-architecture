## 2.3 命令的执行

Minio的服务命令`serverCmd`定义于`minio/cmd/server-main.go`文件中。`serverCmd`命令的`Action`是`serverMain`函数，这个函数是真正的Minio服务命令的入口函数。

### 2.3.1 系统信号
函数开始对系统信号进行了处理。使用了os/signal包的signal.Notify函数。这个函数用于将传入的信号传递给提供的通道。在这种情况下，通道是`globalOSSignalCh`。正在监听的信号是`os.Interrupt`，`syscall.SIGTERM`和`syscall.SIGQUIT`。
```
// serverMain handler called for 'minio server' command.
func serverMain(ctx *cli.Context) {
	var warnings []string

	signal.Notify(globalOSSignalCh, os.Interrupt, syscall.SIGTERM, syscall.SIGQUIT)

	go handleSignals()
...
}
```
os.Interrupt是当有人在终端中按Ctrl+C时发送的信号。syscall.SIGTERM是发送给进程以请求其终止的终止信号，但与SIGKILL信号不同，它可以被进程捕获和解释（或忽略）。syscall.SIGQUIT类似于SIGTERM，但在终止进程时还会产生核心转储。

然后使用go关键字启动了一个goroutine。这个goroutine将与程序的其余部分并发运行`handleSignals()`函数，位于文件`minio/cmd/signals.go`中。
```
package cmd
...
func handleSignals() {
...
for {
		select {
		case err := <-globalHTTPServerErrorCh:
			logger.LogIf(context.Background(), err)
			exit(stopProcess())
		case osSignal := <-globalOSSignalCh:
			logger.Info("Exiting on signal: %s", strings.ToUpper(osSignal.String()))
			daemon.SdNotify(false, daemon.SdNotifyStopping)
			exit(stopProcess())
		case signal := <-globalServiceSignalCh:
			switch signal {
			case serviceRestart:
				logger.Info("Restarting on service signal")
				daemon.SdNotify(false, daemon.SdNotifyReloading)
				stop := stopProcess()
				rerr := restartProcess()
				if rerr == nil {
					daemon.SdNotify(false, daemon.SdNotifyReady)
				}
				logger.LogIf(context.Background(), rerr)
				exit(stop && rerr == nil)
			case serviceStop:
				logger.Info("Stopping on service signal")
				daemon.SdNotify(false, daemon.SdNotifyStopping)
				exit(stopProcess())
			}
		}
	}
}
```
### 2.3.2 默认Profiler速率
函数位于`minio/cmd/utils.go`中。
```
func setDefaultProfilerRates() {
	runtime.MemProfileRate = 128 << 10 // 512KB -> 128K - Must be constant throughout application lifetime.
	runtime.SetMutexProfileFraction(0) // Disable until needed
	runtime.SetBlockProfileRate(0)     // Disable until needed
}
```

### 2.3.3 bootstrapTrace与控制台日志记录器
`bootstrapTrace`函数的目的是记录系统启动过程中的各个步骤。它首先打印出开始执行任务的时间，然后执行任务，最后记录任务执行完成的时间。这对于调试和性能分析非常有用。这个函数接受两个参数：一个字符串和一个函数。
```
func bootstrapTrace(msg string, worker func()) {
	if serverDebugLog {
		fmt.Println(time.Now().Round(time.Millisecond).Format(time.RFC3339), " bootstrap: ", msg)
	}

	now := time.Now()
	worker()
	dur := time.Since(now)

	info := madmin.TraceInfo{
		TraceType: madmin.TraceBootstrap,
		Time:      UTCNow(),
		NodeName:  globalLocalNodeName,
		FuncName:  "BOOTSTRAP",
		Message:   fmt.Sprintf("%s %s (duration: %s)", getSource(2), msg, dur),
	}
	globalBootstrapTracer.Record(info)

	if globalTrace.NumSubscribers(madmin.TraceBootstrap) == 0 {
		return
	}

	globalTrace.Publish(info)
}
```
下面是一个例子，字符串是"newConsoleLogger"，函数是一个匿名函数，也被称为闭包。在这个闭包中，首先创建一个新的控制台日志记录器`NewConsoleLogger(GlobalContext)`，然后将其添加到系统目标中`logger.AddSystemTarget(GlobalContext, globalConsoleSys)`。然后，设置节点名称`globalConsoleSys.SetNodeName(globalLocalNodeName)`。
```
	bootstrapTrace("newConsoleLogger", func() {
		globalConsoleSys = NewConsoleLogger(GlobalContext)
		logger.AddSystemTarget(GlobalContext, globalConsoleSys)

		// Set node name, only set for distributed setup.
		globalConsoleSys.SetNodeName(globalLocalNodeName)
	})
```

### 2.3.4 服务命令处理函数

```
	// Handle all server command args and build the disks layout
	bootstrapTrace("serverHandleCmdArgs", func() {
		err := buildServerCtxt(ctx, &globalServerCtxt)
		logger.FatalIf(err, "Unable to prepare the list of endpoints")

		serverHandleCmdArgs(globalServerCtxt)
	})
```
`buildServerCtxt`位于文件`minio/cmd/common-main.go`中，其目的是从`cli.Context`类型对象构造`serverCtxt`类型对象。基于`github.com/urfave/cli`的`minio/cli`包用于构建命令行应用。
```
func buildServerCtxt(ctx *cli.Context, ctxt *serverCtxt) (err error) {
	// Get "json" flag from command line argument and
	ctxt.JSON = ctx.IsSet("json") || ctx.GlobalIsSet("json")
	// Get quiet flag from command line argument.
	ctxt.Quiet = ctx.IsSet("quiet") || ctx.GlobalIsSet("quiet")
	// Get anonymous flag from command line argument.
	ctxt.Anonymous = ctx.IsSet("anonymous") || ctx.GlobalIsSet("anonymous")
	// Fetch address option
	ctxt.Addr = ctx.GlobalString("address")
	if ctxt.Addr == "" || ctxt.Addr == ":"+GlobalMinioDefaultPort {
		ctxt.Addr = ctx.String("address")
	}

	// Fetch console address option
	ctxt.ConsoleAddr = ctx.GlobalString("console-address")
	if ctxt.ConsoleAddr == "" {
		ctxt.ConsoleAddr = ctx.String("console-address")
	}

	// Check "no-compat" flag from command line argument.
	ctxt.StrictS3Compat = !(ctx.IsSet("no-compat") || ctx.GlobalIsSet("no-compat"))

	switch {
	case ctx.IsSet("config-dir"):
		ctxt.ConfigDir = ctx.String("config-dir")
		ctxt.configDirSet = true
	case ctx.GlobalIsSet("config-dir"):
		ctxt.ConfigDir = ctx.GlobalString("config-dir")
		ctxt.configDirSet = true
	}

	switch {
	case ctx.IsSet("certs-dir"):
		ctxt.CertsDir = ctx.String("certs-dir")
		ctxt.certsDirSet = true
	case ctx.GlobalIsSet("certs-dir"):
		ctxt.CertsDir = ctx.GlobalString("certs-dir")
		ctxt.certsDirSet = true
	}

	ctxt.FTP = ctx.StringSlice("ftp")
	ctxt.SFTP = ctx.StringSlice("sftp")

	ctxt.Interface = ctx.String("interface")
	ctxt.UserTimeout = ctx.Duration("conn-user-timeout")
	ctxt.ConnReadDeadline = ctx.Duration("conn-read-deadline")
	ctxt.ConnWriteDeadline = ctx.Duration("conn-write-deadline")
	ctxt.ConnClientReadDeadline = ctx.Duration("conn-client-read-deadline")
	ctxt.ConnClientWriteDeadline = ctx.Duration("conn-client-write-deadline")

	ctxt.ShutdownTimeout = ctx.Duration("shutdown-timeout")
	ctxt.IdleTimeout = ctx.Duration("idle-timeout")
	ctxt.ReadHeaderTimeout = ctx.Duration("read-header-timeout")
	ctxt.MaxIdleConnsPerHost = ctx.Int("max-idle-conns-per-host")

	if conf := ctx.String("config"); len(conf) > 0 {
		err = mergeServerCtxtFromConfigFile(conf, ctxt)
	} else {
		err = mergeDisksLayoutFromArgs(serverCmdArgs(ctx), ctxt)
	}

	return err
}
```
`func serverHandleCmdArgs(ctxt serverCtxt)`位于文件`minio/cmd/server-main.go`中，其主要：
1. 处理了通用的参数，`handleCommonArgs(ctxt)`;
2. 检查和加载TLS认证，`globalPublicCerts, globalTLSCerts, globalIsTLS, err = getTLSConfig()`;
3. 检查和加载根证书，`globalRootCAs, err = certs.GetRootCAs(globalCertsCADir.Get())`;
4. 把全局公共证书加入全局根证书;
5. 把全局根证书存入环境变量；