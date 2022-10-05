---
title: 'fx: golang依赖注入框架'
tags:
  - golang
  - 依赖注入
categories:
  - framework
abbrlink: 45063
date: 2022-10-02 17:02:04
---

# uber fx
fx 是 uber 开源的一款依赖注入框架，依赖注入这个名词对我来说一直是个很奇怪的存在（不了解 Java ），小项目引入依赖注入完全没必要啊，凭空提高复杂度（逃
fx 的作用是解决了golang项目中坐落各个包的全局变量，以及数不清的 ```init``` 函数。

这里也作为我学习的记录，分享一下我对 fx 的理解，包括一些源码的分析。

##  函数签名
这里列出主要会用到的函数与方法，并对其作用作出简要解释。

```go
func New(opts ...Option) *App
func Provide(constructors ...interface{}) Option 
func Supply(values ...interface{}) Option
func Populate(targets ...interface{}) Option
func Invoke(funcs ...interface{}) Option
```
其中 ```New``` 函数没什么好说的，它根据传入的 opts 构建 fx.App。
下面的四个函数的返回类型都是 Option，也就是 New 函数的入参。
* Provide 
该函数传入的参数是构造函数 也就是 类似 ```func NewC(A, B ...) C ``` 的函数 ，构造 C 需要依赖于 A，B … 
* Supply
函数传入的参数是已经构造完毕的值（value），也就是说 ```Provide(NewC)``` → ```Supply(C)``` ， 其中 C = NewC(...)
当其他构造函数依赖于类型C时，不通过调用 NewC 生成，而是直接使用提供的 C。
* Populate
在 ```New``` 函数外部，我们先var了一个 C，并且通过 ```Provide``` 注入 C 的构造函数，那么外部通过植入的变量C，在初始化完成后通过 ```Provide``` 的构造函数完成构造。这样就可以在 ```New``` 函数外部使用这个经过构造的变量。
* Invoke
直接贴注释 ```registers functions that are executed eagerly on application start```
它注册一些在app启动时需要执行的函数，被注册的 func 的入参，通过 Provide 注入的构造函数生成。

需要注意的一点是 Invoke 注册的函数的运行是**有顺序**的，而 Provide 注入的构造函数并没有顺序，后面会更详细的分析。

---

```go
type fx.App struct {...}
    func (app *App) Err() error
    func (app *App) Run() 
    func (app *App) Start(context.Context) error
```
其中 ```Run``` 还是调用了 ```Start``` 。

## 使用
### Provide、Populate、Invoke、Supply
在我们写 golang 项目的时候，经常会遇到要使用包内全局变量的，通过 ```import``` 其他包，来使用包内的全局变量。

```go
package modx

var Foo TypeX

func init() {
    Foo = NewTypeX()
    ...
}
```

我们可能会遇到这种情况

```go
func NewA() TypeA 
func NewB(TypeA) TypeB
func NewC(TypeA, TypeB) TypeC
```
当我们需要一个 TypeC 时， 需要按照顺序手动构造 TypeA， TypeB，然后在构造 TypeC。
使用fx后
```go
func main() {
    var x NewC
    fx.New(
        fx.Provide(NewA, NewB, NewC),
        fx.Populate(&x)
    )
    fmt.Println(x)
}
```
我们将各个构造函数通过  ```Provide``` 函数注入到 ```fx.App``` 后，fx就会帮我们管理构造函数的调用，并且这种调用是 lazy的，即当某一个构造函数不存在被依赖时，那么它是不会被调用的。并且，这些构造函数被调用后构造的变量，是会被缓存的，所以当其他函数存在多个对其依赖时，只会被执行一次，之后都将直接返回第一构造的变量。

所以，我们使用 ```Provide``` 向fx注入构造函数时，注入顺序并不重要。

在这个例子里，我们 Populate 了一个TypeC 指针的外部变量，那么fx就会去调用 ```func NewC()```，然后一次调用其依赖。

<small>*需要注意的是 Populate(targets ...interface{})  中传入的targets必须得是目标类型TypeX的指针类型  \*TypeX，哪怕 TypeX 本身就是指针类型*</small>

同理，当我们不是Populate变量，而是 ```Invoke ``` 一些函数，比如初始化函数，这些函数同样依赖于其余类型，那么fx就会去寻找对应的依赖的构造函数。
```go
func main() {
    printC := func (x TypeC) {
        fmt.Println(x)
    }
    fx.New(
        fx.Provide(NewA, NewB, NewC),
        fx.Invoke(printC)
    )
}
```

---
#### ！
**所以在这里要再重点提一件事，所有的注入的构造函数都需要 ```Invoke or Populate``` 来 “激活链路”**

在这里举一个我遇到的问题。

```go
type Server Struct {
    HTTPS  *HttpServer
    ...
}

func AddRouter(srv *HttpServer) {...}
```
我有这么一个结构体，它依赖于 ```*HttpServer``` ，我在 ```NewServer``` 中添加了 hook（后面会讲），通过 ```Server``` 启动 http 服务。

然后我只使用 ```Invoke``` 添加了一个 ```AddRouter``` 来给 ```*HttpServer``` 添加路由。

当我 使用 ```fx.Run()``` 的时候，并没有看到终端打印 http 服务启动的信息。

其实就是作为入口的 ```AddRouter``` 只依赖了 ```*HttpServer```，那么它只会去调用 ```NewHttpServer``` ，并没有执行 ```NewServer```， 而我需要通过 ```Server``` 来注册 hook，启动 http 服务。

---
`Supply`  就略过不讲了，上一节已经足够了。
### Run、Start、Lifecycle
这部分是比较复杂的部分，一般来说，像跑一个http服务，你可以只使用 ```Provide & Invoke & Populate``` 来完成依赖注入，然后使用 ```Populate``` 植入在 ```fx.New``` 里的外部变量，来启动http服务，比如我将一个 ```http.Server``` 植入，那么在 ```fx.New``` 结束后，我就可以拿着完成初始化的 ```http.Server``` 来启动 http 服务。

而 ```Run、Start、LifeCycle```  主要涉及长期运行的协程，这块直接结合**源码**分析吧

---
我们看一下 `fx.Lifecycle`，它是 `fx.App` 的一个字段，构造函数的依赖用到它时，使用的就是 `fx.New()` 构造的 `fx.App` 的该字段。

下面这个是官方文档给出的例子，它在构造 `ServeMux` 的时注册了服务启动函数。

```go
func NewMux(lc fx.Lifecycle, logger *log.Logger) *http.ServeMux {
	logger.Print("Executing NewMux.")
	mux := http.NewServeMux()
	server := &http.Server{
		Addr:    ":8080",
		Handler: mux,
	}

	lc.Append(fx.Hook{
		OnStart: func(context.Context) error {
			logger.Print("Starting HTTP server.")
			go server.ListenAndServe()
			return nil
		},
		OnStop: func(ctx context.Context) error {
			logger.Print("Stopping HTTP server.")
			return server.Shutdown(ctx)
		},
	})

	return mux
}
```

我们需要向 `Lifecyle` 注册勾子，主要是添加一些需要在 `fx.App` 启动和关闭时需要执行的操作。

接下来我们再深入看看 `Lifecycle.Start`  （主要看# 后的注释）
```go
func (l *Lifecycle) Start(ctx context.Context) error {
	if ctx == nil {
		return errors.New("called OnStart with nil context")
	}

	l.mu.Lock()
	l.startRecords = make(HookRecords, 0, len(l.hooks))
	l.mu.Unlock()

	for _, hook := range l.hooks { # 遍历我们先前注册的hook
		// if ctx has cancelled, bail out of the loop.
		if err := ctx.Err(); err != nil {
			return err
		}

		if hook.OnStart != nil {
			l.mu.Lock()
			l.runningHook = hook
			l.mu.Unlock()

			runtime, err := l.runStartHook(ctx, hook) # 这里对hook进行调用
			if err != nil {
				return err
			}

			l.mu.Lock()
			l.startRecords = append(l.startRecords, HookRecord{
				CallerFrame: hook.callerFrame,
				Func:        hook.OnStart,
				Runtime:     runtime,
			})
			l.mu.Unlock()
		}
		l.numStarted++
	}

	return nil
}
```

```go
func (l *Lifecycle) runStartHook(ctx context.Context, hook Hook) (runtime time.Duration, err error) {
	funcName := fxreflect.FuncName(hook.OnStart)
	l.logger.LogEvent(&fxevent.OnStartExecuting{
		CallerName:   hook.callerFrame.Function,
		FunctionName: funcName,
	})
	defer func() {
		l.logger.LogEvent(&fxevent.OnStartExecuted{
			CallerName:   hook.callerFrame.Function,
			FunctionName: funcName,
			Runtime:      runtime,
			Err:          err,
		})
	}()

	begin := l.clock.Now()
	err = hook.OnStart(ctx) # 调用注册的函数
	return l.clock.Since(begin), err
}
```

我们可以看到，我们对于 hook 的调用是同步的，所以 hook 的执行不能消耗太多时间，可以通过在 hook 里开启新协程的方法来异步执行，就像官方文档给的例子一样。

---
搞明白了 `Lifecycle` 的作用，那么就可以深入了解一下 `app.Start` 。
```go
func (app *App) Start(ctx context.Context) (err error) {
	defer func() {
		app.log.LogEvent(&fxevent.Started{Err: err})
	}()

	if app.err != nil {
		// Some provides failed, short-circuit immediately.
		return app.err
	}

	return withTimeout(ctx, &withTimeoutParams{
		hook:      _onStartHook,
		callback:  app.start,
		lifecycle: app.lifecycle,
		log:       app.log,
	})
}
```
这里的callback是下面的 `start` 方法，它调用了 `Lifecycle` 的 `Start` 方法，作用上面讲了。
```go
func (app *App) start(ctx context.Context) error {
	if err := app.lifecycle.Start(ctx); err != nil {
		// Start failed, rolling back.
		app.log.LogEvent(&fxevent.RollingBack{StartErr: err})

		stopErr := app.lifecycle.Stop(ctx)
		app.log.LogEvent(&fxevent.RolledBack{Err: stopErr})

		if stopErr != nil {
			return multierr.Append(err, stopErr)
		}

		return err
	}
	return nil
}
```
---
来看 `Run` 和 `Start` 的关系
```go
func (app *App) Run() {
	if code := app.run(app.Done()); code != 0 {
		app.exit(code)
	}
}

func (app *App) run(done <-chan os.Signal) (exitCode int) {
	startCtx, cancel := app.clock.WithTimeout(context.Background(), app.StartTimeout())
	defer cancel()

	if err := app.Start(startCtx); err != nil {
		return 1
	}

	sig := <-done
	app.log.LogEvent(&fxevent.Stopping{Signal: sig})

	stopCtx, cancel := app.clock.WithTimeout(context.Background(), app.StopTimeout())
	defer cancel()

	if err := app.Stop(stopCtx); err != nil {
		return 1
	}

	return 0
}
```
我们可以看到，```Run``` 方法调用了 ```Start(context.Context)```， 并监听退出信号（这里的done = app.Done() ,which 返回一个接收系统退出信号的双向通道。）

`sig := <-done` 主线程会被阻塞在这一行代码，直到收到系统的退出信号。

那么我们可以看到 ```Run``` 方法的意义其实就是为 ```Start & Stop``` 提供了一个包含默认超时时间的 `context`，用于 `Start & Stop` 的控制。

我们完全可以跳过 `Run` ，直接调用 `Start` ，这样可以自定义超时时间，这样就需要我们手动阻塞主线程，然后最后再调用 `Stop` 来执行 `OnStop` 的 `hook`。


# demo 实例
[使用fx，基于fiber框架的HTTP服务demo](https://github.com/KarlvenK/fx-diy)


# 参考文献
1. https://zhuanlan.zhihu.com/p/418299054
2. https://pkg.go.dev/go.uber.org/fx@v1.18.2