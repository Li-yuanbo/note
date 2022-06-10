## 1. 上下文Context

上下文 `context.Context`Go 语言中用来设置超时时间、同步信号，传递请求相关值的结构体。上下文与 Goroutine 有比较密切的关系，是 Go 语言中独特的设计。

 `context.Context`接口定义了四个需要实现的方法，其中包括：

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key interface{}) interface{}
}
```

1. `Deadline` ： 返回 `context.Context`被取消的时间，也就是完成工作的截止日期；

2. `Done` ： 返回一个 Channel，这个 Channel 会在当前工作完成或者上下文被取消后关闭，多次调用 `Done` 方法会返回同一个 Channel；

3. `Err`：返回`context.Context`结束的原因，它只会在`Done`方法对应的 Channel 关闭时返回非空的值；

- 如果 `context.Context`被取消，会返回 `Canceled` 错误；

- 如果 `context.Context`超时，会返回 `DeadlineExceeded` 错误；

4. `Value` ：从 `context.Context`中获取键对应的值，对于同一个上下文来说，多次调用 `Value` 并传入相同的 `Key` 会返回相同的结果，该方法可以用来传递请求特定的数据；

### 1.1 设计原理

在 Goroutine 构成的树形结构中对信号进行同步以减少计算资源的浪费是 `context.Context`的最大作用。Go 服务的每一个请求都是通过单独的 Goroutine 处理的，HTTP/RPC 请求的处理器会启动新的 Goroutine 访问数据库和其他服务。

如下图所示，我们可能会创建多个 Goroutine 来处理一次请求，而 `context.Context`的作用是在不同 Goroutine 之间同步请求特定数据、取消信号以及处理请求的截止日期。

<img src="/Users/yuanbli/yuanbli/markdown/Golang/pic/context_1.png" alt="4.1" style="zoom:75%;" />

每一个 `context.Context`都会从最顶层的 Goroutine 一层一层传递到最下层。`context.Context`可以在上层 Goroutine 执行出现错误时，将信号及时同步给下层。

### 1.2 设计原理

`context`包中最常用的方法还是 `context.Background`、`context.TODO`，这两个方法都会返回预先初始化好的私有变量 `background` 和 `todo`，它们会在同一个 Go 程序中被复用：

```go
func Background() Context {
	return background
}

func TODO() Context {
	return todo
}
```

从源代码来看，`context.Background`和 `context.TODO` 也只是互为别名，没有太大的差别，只是在使用和语义上稍有不同：

- `context.Background`是上下文的默认值，所有其他的上下文都应该从它衍生出来
- `context.TODO`应该仅在不确定应该使用哪种上下文时使用

在多数情况下，如果当前函数没有上下文作为入参，我们都会使用 `context.Background`作为起始的上下文向下传递。

### 1.3 取消信号

`context.WithCancel`函数能够从 `context.Context`中衍生出一个新的子上下文并返回用于取消该上下文的函数。一旦我们执行返回的取消函数，当前上下文以及它的子上下文都会被取消，所有的 Goroutine 都会同步收到这一取消信号。

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
```

### 1.4 传值方法

`context` 包中的 `context.WithValue`能从父上下文中创建一个子上下文，传值的子上下文使用 `context.valueCtx`类型：

```go
func WithValue(parent Context, key, val interface{}) Context {
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
	return &valueCtx{parent, key, val}
}
```

`context.valueCtx` 结构体会将除了 `Value` 之外的 `Err`、`Deadline` 等方法代理到父上下文中，它只会响应 `context.valueCtx.Value`方法。

如果 `context.valueCtx`中存储的键值对与 `context.valueCtx.Value`方法中传入的参数不匹配，就会从父上下文中查找该键对应的值直到某个父上下文中返回 `nil` 或者查找到对应的值。