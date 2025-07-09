# Go语言基础之处理并发错误

我们可以在Go语言中十分便捷地开启goroutine去并发地执行任务，但是如何有效的处理并发过程中的错误则是一个很棘手的问题，本文介绍了一些处理并发错误的方法。

### recover goroutine中的panic <a href="#recovergoroutine-zhong-de-panic" id="recovergoroutine-zhong-de-panic"></a>

我们知道可以在代码中使用 recover 来会恢复程序中意想不到的 panic，而 panic 只会触发当前 goroutine 中的 defer 操作。

例如在下面的示例代码中，无法在 main 函数中 recover 另一个goroutine中引发的 panic。

```
func f1() {
	defer func() {
		if e := recover(); e != nil {
			fmt.Printf("recover panic:%v\n", e)
		}
	}()
	// 开启一个goroutine执行任务
	go func() {
		fmt.Println("in goroutine....")
		// 只能触发当前goroutine中的defer
		panic("panic in goroutine")
	}()

	time.Sleep(time.Second)
	fmt.Println("exit")
}
```

执行上面的 f1 函数会得到如下结果：

```
in goroutine....
panic: panic in goroutine

goroutine 6 [running]:
main.f1.func2()
        /Users/liwenzhou/workspace/github/the-road-to-learn-golang/ch12/goroutine_recover.go:20 +0x65
created by main.f1
        /Users/liwenzhou/workspace/github/the-road-to-learn-golang/ch12/goroutine_recover.go:17 +0x48

Process finished with exit code 2
```

从输出结果可以看到程序并没有正常退出，而是由于 panic 异常退出了（exit code 2）。

正如上面示例演示的那样，在启用 goroutine 去执行任务的场景下，如果想要 recover goroutine中可能出现的 panic 就需要在 goroutine 中使用 recover。就像下面的 f2 函数那样。

```
func f2() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Printf("recover outer panic:%v\n", r)
		}
	}()
	// 开启一个goroutine执行任务
	go func() {
		defer func() {
			if r := recover(); r != nil {
				fmt.Printf("recover inner panic:%v\n", r)
			}
		}()
		fmt.Println("in goroutine....")
		// 只能触发当前goroutine中的defer
		panic("panic in goroutine")
	}()

	time.Sleep(time.Second)
	fmt.Println("exit")
}
```

执行 f2 函数会得到如下输出结果。

```
in goroutine....
recover inner panic:panic in goroutine
exit
```

程序中的 panic 被 recover 成功捕获，程序最终正常退出。

### errgroup <a href="#errgroup" id="errgroup"></a>

在以往演示的并发示例中，我们通常像下面的示例代码那样在 go 关键字后，调用一个函数或匿名函数。

```
go func(){
  // ...
}

go foo()
```

在之前讲解并发的代码示例中我们默认被并发的那些函数都不会返回错误，但真实的情况往往是事与愿违。

当我们想要将一个任务拆分成多个子任务交给多个 goroutine 去运行，这时我们该如何获取到子任务可能返回的错误呢？

假设我们有多个网址需要并发去获取它们的内容，这时候我们会写出类似下面的代码。

```
// fetchUrlDemo 并发获取url内容
func fetchUrlDemo() {
	wg := sync.WaitGroup{}
	var urls = []string{
		"http://pkg.go.dev",
		"http://www.liwenzhou.com",
		"http://www.yixieqitawangzhi.com",
	}

	for _, url := range urls {
		wg.Add(1)
		go func(url string) {
			defer wg.Done()
			resp, err := http.Get(url)
			if err == nil {
				fmt.Printf("获取%s成功\n", url)
				resp.Body.Close()
			}
			return // 如何将错误返回呢？
		}(url)
	}
	wg.Wait()
	// 如何获取goroutine中可能出现的错误呢？
}
```

执行上述`fetchUrlDemo`函数得到如下输出结果，由于 [http://www.yixieqitawangzhi.com](http://www.yixieqitawangzhi.com/) 是我随意编造的一个并不真实存在的 url，所以对它的 HTTP 请求会返回错误。

```
获取http://pkg.go.dev成功
获取http://www.liwenzhou.com成功
```

在上面的示例代码中，我们开启了 3 个 goroutine 分别去获取3个 url 的内容。类似这种将任务分为若干个子任务的场景会有很多，那么我们如何获取子任务中可能出现的错误呢？

errgroup 包就是为了解决这类问题而开发的，它能为处理公共任务的子任务而开启的一组 goroutine 提供同步、error 传播和基于context 的取消功能。

errgroup 包中定义了一个 Group 类型，它包含了若干个不可导出的字段。

```
type Group struct {
	cancel func()

	wg sync.WaitGroup

	errOnce sync.Once
	err     error
}
```

errgroup.Group 提供了`Go`和`Wait`两个方法。

```
func (g *Group) Go(f func() error)
```

* Go 函数会在新的 goroutine 中调用传入的函数f。
* 第一个返回非零错误的调用将取消该Group；下面的Wait方法会返回该错误

```
func (g *Group) Wait() error
```

* Wait 会阻塞直至由上述 Go 方法调用的所有函数都返回，然后从它们返回第一个非nil的错误（如果有）。

下面的示例代码演示了如何使用 errgroup 包来处理多个子任务 goroutine 中可能返回的 error。

```
// fetchUrlDemo2 使用errgroup并发获取url内容
func fetchUrlDemo2() error {
	g := new(errgroup.Group) // 创建等待组（类似sync.WaitGroup）
	var urls = []string{
		"http://pkg.go.dev",
		"http://www.liwenzhou.com",
		"http://www.yixieqitawangzhi.com",
	}
	for _, url := range urls {
		url := url // 注意此处声明新的变量
		// 启动一个goroutine去获取url内容
		g.Go(func() error {
			resp, err := http.Get(url)
			if err == nil {
				fmt.Printf("获取%s成功\n", url)
				resp.Body.Close()
			}
			return err // 返回错误
		})
	}
	if err := g.Wait(); err != nil {
		// 处理可能出现的错误
		fmt.Println(err)
		return err
	}
	fmt.Println("所有goroutine均成功")
	return nil
}
```

执行上面的`fetchUrlDemo2`函数会得到如下输出结果。

```
获取http://pkg.go.dev成功
获取http://www.liwenzhou.com成功
Get "http://www.yixieqitawangzhi.com": dial tcp: lookup www.yixieqitawangzhi.com: no such host
```

当子任务的 goroutine 中对`http://www.yixieqitawangzhi.com` 发起 HTTP 请求时会返回一个错误，这个错误会由 errgroup.Group 的 Wait 方法返回。

通过阅读下方 errgroup.Group 的 Go 方法源码，我们可以看到当任意一个函数 f 返回错误时，会通过`g.errOnce.Do`只将第一个返回的错误记录，并且如果存在 cancel 方法则会调用cancel。

```
func (g *Group) Go(f func() error) {
	g.wg.Add(1)

	go func() {
		defer g.wg.Done()

		if err := f(); err != nil {
			g.errOnce.Do(func() {
				g.err = err
				if g.cancel != nil {
					g.cancel()
				}
			})
		}
	}()
}
```

那么如何创建带有 cancel 方法的 errgroup.Group 呢？

答案是通过 errorgroup 包提供的 WithContext 函数。

```
func WithContext(ctx context.Context) (*Group, context.Context)
```

WithContext 函数接收一个父 context，返回一个新的 Group 对象和一个关联的子 context 对象。下面的代码片段是一个官方文档给出的示例。

```
package main

import (
	"context"
	"crypto/md5"
	"fmt"
	"io/ioutil"
	"log"
	"os"
	"path/filepath"

	"golang.org/x/sync/errgroup"
)

// Pipeline demonstrates the use of a Group to implement a multi-stage
// pipeline: a version of the MD5All function with bounded parallelism from
// https://blog.golang.org/pipelines.
func main() {
	m, err := MD5All(context.Background(), ".")
	if err != nil {
		log.Fatal(err)
	}

	for k, sum := range m {
		fmt.Printf("%s:\t%x\n", k, sum)
	}
}

type result struct {
	path string
	sum  [md5.Size]byte
}

// MD5All reads all the files in the file tree rooted at root and returns a map
// from file path to the MD5 sum of the file's contents. If the directory walk
// fails or any read operation fails, MD5All returns an error.
func MD5All(ctx context.Context, root string) (map[string][md5.Size]byte, error) {
	// ctx is canceled when g.Wait() returns. When this version of MD5All returns
	// - even in case of error! - we know that all of the goroutines have finished
	// and the memory they were using can be garbage-collected.
	g, ctx := errgroup.WithContext(ctx)
	paths := make(chan string)

	g.Go(func() error {

		return filepath.Walk(root, func(path string, info os.FileInfo, err error) error {
			if err != nil {
				return err
			}
			if !info.Mode().IsRegular() {
				return nil
			}
			select {
			case paths <- path:
			case <-ctx.Done():
				return ctx.Err()
			}
			return nil
		})
	})

	// Start a fixed number of goroutines to read and digest files.
	c := make(chan result)
	const numDigesters = 20
	for i := 0; i < numDigesters; i++ {
		g.Go(func() error {
			for path := range paths {
				data, err := ioutil.ReadFile(path)
				if err != nil {
					return err
				}
				select {
				case c <- result{path, md5.Sum(data)}:
				case <-ctx.Done():
					return ctx.Err()
				}
			}
			return nil
		})
	}
	go func() {
		g.Wait()
		close(c)
	}()

	m := make(map[string][md5.Size]byte)
	for r := range c {
		m[r.path] = r.sum
	}
	// Check whether any of the goroutines failed. Since g is accumulating the
	// errors, we don't need to send them (or check for them) in the individual
	// results sent on the channel.
	if err := g.Wait(); err != nil {
		return nil, err
	}
	return m, nil
}
```

或者这里有另外一个示例。

```
func GetFriends(ctx context.Context, user int64) (map[string]*User, error) {
  g, ctx := errgroup.WithContext(ctx)
  friendIds := make(chan int64)

  // Produce
  g.Go(func() error {
     defer close(friendIds)
     for it := GetFriendIds(user); ; {
        if id, err := it.Next(ctx); err != nil {
           if err == io.EOF {
              return nil
           }
           return fmt.Errorf("GetFriendIds %d: %s", user, err)
        } else {
           select {
           case <-ctx.Done():
              return ctx.Err()
           case friendIds <- id:
           }
        }
     }
  })

  friends := make(chan *User)

  // Map
  workers := int32(nWorkers)
  for i := 0; i < nWorkers; i++ {
     g.Go(func() error {
        defer func() {
           // Last one out closes shop
           if atomic.AddInt32(&workers, -1) == 0 {
              close(friends)
           }
        }()

        for id := range friendIds {
           if friend, err := GetUserProfile(ctx, id); err != nil {
              return fmt.Errorf("GetUserProfile %d: %s", user, err)
           } else {
              select {
              case <-ctx.Done():
                 return ctx.Err()
              case friends <- friend:
              }
           }
        }
        return nil
     })
  }

  // Reduce
  ret := map[string]*User{}
  g.Go(func() error {
     for friend := range friends {
        ret[friend.Name] = friend
     }
     return nil
  })

  return ret, g.Wait()
}
```

可惜这两个示例不太好理解。。。

***
