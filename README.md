# Code Review Comments 中文翻译

原文地址 https://github.com/golang/go/wiki/CodeReviewComments#comment-sentences

## Gofmt

运行 gofmt 在您的代码上 可自动修复大多数机械样式问题。 几乎所有的Go代码都在使用 gofmt。 本文档的其余部分介绍了非机械样式点。

一种可选的工具是使用 goimports ，它是 gofmt 的超集，用于额外添加（和删除）导入行。

## 注释

请参阅 https://golang.org/doc/effective_go.html#commentary 。对象声明的注释应该是完整的句子，即使这看起来有点多余。 当提取到godoc文档中时，这种方法可以使它们格式化良好。 注释应以所描述事物的名称开头，并以句点结尾：

```
// Request represents a request to run a command.
type Request struct { ...

// Encode writes the JSON encoding of req to w.
func Encode(w io.Writer, req *Request) { ...
```
等等。

## Context（上下文）
context的值。Context类型承载跨API和流程边界的安全凭证，跟踪信息，期限和取消信号。 Go程序从传入的RPC和HTTP请求到传出的请求，在整个函数调用链中显式传递Context。

大多数使用Context的函数都应将其作为第一个参数：
```
func F(ctx context.Context, /* other arguments */) {}
```
从来没有特定于请求的函数可以使用context.Background()，但是即使您认为不需要传递Context也会出错。 默认情况是传递Context。 仅当您有充分的理由认为替代方法有误时，才直接使用context.Background()。 

不要将Context成员添加到结构类型； 而是将ctx参数添加到该类型需要传递的每个方法上。 一个例外是方法的签名必须与标准库或第三方库中的接口匹配的方法。

不要在功能签名中创建自定义上下文类型或使用除Context之外的接口。 

如果要传递应用程序数据，请将其放在参数中，接收器中，全局变量中，或者如果它确实属于该参数，则在Context值中。 

Context是不可变的，因此可以将相同的ctx传递给共享相同截止时间，取消信号，凭证，父级跟踪等的多个调用。 

## Copying（复制）

为避免意外的别名，从另一个包中复制结构时要小心。 例如，bytes.Buffer类型包含[] byte切片。 如果复制缓冲区，则副本中的切片可能会使原始数组中的数组成为别名，从而导致后续方法调用产生令人惊讶的效果。 

通常，如果其方法与指针类型 *T 相关联，则不要复制类型T的值。 

## Crypto Rand
不要使用软件包 math/rand 生成密钥，即使是一次性密钥也是如此。 在没有种子情况下，生成器是完全可预测的。 将time.Nanoseconds()的作为随机生成器的种子，只有几熵（应该是消耗很少的意思）。 相反，请使用 crypto/rand's Reader，如果需要文本，请打印为十六进制或base64：

```
import (
	"crypto/rand"
	// "encoding/base64"
	// "encoding/hex"
	"fmt"
)

func Key() string {
	buf := make([]byte, 16)
	_, err := rand.Read(buf)
	if err != nil {
		panic(err)  // out of randomness, should never happen
	}
	return fmt.Sprintf("%x", buf)
	// or hex.EncodeToString(buf)
	// or base64.StdEncoding.EncodeToString(buf)
}
```
## 声明空切片

声明空切片时，首选
```
var t []string
```
其次
```
t := []string{}
```
前者声明nil slice值，而后者声明非nil但长度为零。 它们在功能上等同，他们 len和 cap均为零，而零切片是首选的风格。

请注意，在少数情况下，首选非零但长度为零的切片，例如在编码JSON对象时（ nil切片编码为 null，而 []string{}编码为JSON数组 []）。

在设计接口时，请避免区分零切片和零长度的非零切片，因为这会导致细微的编程错误。

有关Go中的nil的更多讨论，请参见Francesc Campoy的演讲 [Understanding Nil](https://www.youtube.com/watch?v=ynoY2xz-F8s)。

## Doc文档
所有顶级的导出名称都应具有文档注释，特殊的未导出类型或函数声明也应具有文档注释。请参见 https://golang.org/doc/effective_go.html#commentary 有关注释约定的更多信息。

## 不要使用panic

请参阅 https://golang.org/doc/effective_go.html#errors 。不要对正常的错误处理使用panic。使用错误和多个返回值。

## 错误字符串

错误字符串不应大写（除非以专有名词或缩写开头）或标点符号结束，因为它们通常是在其他上下文之后打印的。 也就是说，请使用 fmt.Errorf("something bad")，不要使用 fmt.Errorf("Something bad")，这样 log.Printf("Reading %s: %v", filename, err)格式就不会出现虚假的大写字母中间消息。 这不适用于日志记录，后者是隐式面向行的，并且未在其他消息中合并。

## 示例
添加新程序包时，请包括预期用法的示例：可运行的示例， 或演示完整调用顺序的简单测试。

阅读有关[testable Example() functions](https://blog.golang.org/examples)的更多信息。

## Goroutine生命周期

当您生成goroutine时，请清楚何时或是否退出。

Goroutine可以通过阻塞通道的发送或接收来泄漏：即使Gogortine的阻塞通道不可访问，垃圾收集器也不会终止goroutine。 

即使goroutine不会泄漏，在不再需要它们时仍在飞行中也会导致其他细微且难以诊断的问题。 在关闭的通道上发送紧急消息。 “在不需要结果之后”修改仍在使用的输入仍然会导致数据争用。 将goroutine进行任意长时间的飞行可能会导致不可预测的内存使用情况。 

尝试使并发代码足够简单，以使goroutine生存期显而易见。 如果那不可行，请记录goroutines何时以及为何退出。

## 错误处理

请参阅 https://golang.org/doc/effective_go.html#errors 。不要使用_变量丢弃错误。 如果函数返回错误，请检查以确保函数成功。 处理错误，将其退回，或者在真正特殊的情况下panic。

## Imports
除避免名称冲突外，避免重命名导入； 好的软件包名称不需要重命名。 发生名称冲突时，最好重命名本地的或特定于项目的导入。

导入包是按组组织排序的，组之间有空白行。标准库软件包始终在第一组中。
```
package main

import (
	"fmt"
	"hash/adler32"
	"os"

	"appengine/foo"
	"appengine/user"

	"github.com/foo/bar"
	"rsc.io/goversion/version"
)
```
goimports 将为您做到这一点。

## Import Blank
仅出于辅助作用而导入的软件包（使用语法import _"pkg"）应仅在程序的主软件包或需要它们的测试中导入。

## Import .

由于循环依赖关系，导入 . 形式不能用于要测试的程序包，因此在测试中很有用：
```
package foo_test

import (
	"bar/testutil" // also imports "foo"
	. "foo"
)
```
在这种情况下，测试文件不能放在foo包中，因为它使用了导入foo的bar / testutil。 因此，我们使用“import .” 使文件假装为foo软件包的一部分的格式，即使它不是。 除这种情况外，请勿使用import . 在您的程序中。 由于不清楚Quux之类的名称是否是当前包或导入包中的顶级标识符，因此使程序更难阅读。

## In-Band Errors(带内错误？我猜是内部错误)

在C和类似语言中，函数通常返回-1这样的值或null来表示错误或没有结果：
```
// Lookup returns the value for key or "" if there is no mapping for key.
func Lookup(key string) string

// Failing to check a for an in-band error value can lead to bugs:
Parse(Lookup(key))  // returns "parse failure for value" instead of "no value for key"
```
Go对多个返回值的支持提供了更好的解决方案。 函数应该返回一个附加值以指示其其他返回值是否有效，而不是要求客户端检查带内错误值。 该返回值可以是错误，也可以是布尔值（不需要说明时）。 它应该是最终的返回值。 
```
// Lookup returns the value for key or ok=false if there is no mapping for key.
func Lookup(key string) (value string, ok bool)
```
这样可以防止调用者错误地使用结果：
```
Parse(Lookup(key))  // compile-time error
```
并鼓励使用更健壮和易读的代码：
```
value, ok := Lookup(key)
if !ok {
	return fmt.Errorf("no value for %q", key)
}
return Parse(value)
```
此规则适用于导出的函数，但用于未导出的功能也很有用。

当nil，""，0和-1这样的值是函数的有效结果时作为返回值是很好的，即调用者不需要与其他值做出不同的处理。

某些标准库函数（如程序包“strings”中的函数）返回内部错误。 这大大简化了字符串处理代码，但需要程序员加倍努力。 通常，Go代码应返回错误的其他值。 

## 缩进错误流
尝试使普通代码路径保持最小缩进，并缩进错误处理，并首先对其进行处理。 通过允许在视觉上快速扫描正常路径，可以提高代码的可读性。 例如，不要写：
```
if err != nil {
	// error handling
} else {
	// normal code
}
```
应该写：
```
if err != nil {
	// error handling
	return // or continue, etc.
}
// normal code
```
如果该 if语句具有初始化语句，例如：
```
if x, err := f(); err != nil {
	// error handling
	return
} else {
	// use x
}
```
那么这可能需要将short变量声明移至其自己的行：
```
x, err := f()
if err != nil {
	// error handling
	return
}
// use x
```

## 首字母缩写
名称中的缩写词或首字母缩写词（例如“ URL”或“ NATO”）具有一致的大小写。 例如，“ URL”应显示为“ URL”或“ url”（如在“ urlPony”或“ URLPony”中一样），而从不显示为“ Url”。 例如：ServeHTTP而不是ServeHttp。 对于具有多个已初始化“单词”的标识符，请使用“ xmlHTTPRequest”或“ XMLHTTPRequest”。

当“ ID”是“ identifier”的缩写时，该规则也适用于“ ID”（这在几乎所有情况下都不是“ ego”，“ superego”中的“ id”），因此请写上“ appID”而不是“ appId”。 

协议缓冲区编译器生成的代码不受此规则约束。 人工编写的代码要比机器编写的代码具有更高的标准。

## 接口（Interfaces）

Go接口通常属于使用接口类型的值的包中，而不是实现这些值的包中。 实现包应返回具体的（通常是指针或结构）类型：这样，可以将新方法添加到实现中，而无需进行大量重构。

不要在“用于模拟”的API的实现方定义接口； 而是设计API，以便可以使用实际实现的公共API对其进行测试。 

在使用接口之前，不要先定义它们：如果没有实际的用法示例，很难知道接口是否是必需的，更不用说它应该包含什么方法了。
```
package consumer  // consumer.go

type Thinger interface { Thing() bool }

func Foo(t Thinger) string { … }
```
```
package consumer // consumer_test.go

type fakeThinger struct{ … }
func (t fakeThinger) Thing() bool { … }
…
if Foo(fakeThinger{…}) == "x" { … }
```
```
// DO NOT DO IT!!!
package producer

type Thinger interface { Thing() bool }

type defaultThinger struct{ … }
func (t defaultThinger) Thing() bool { … }

func NewThinger() Thinger { return defaultThinger{ … } }
```
返回一个具体的类型，并让消费者模拟生产者实现。
```
package producer

type Thinger struct{ … }
func (t Thinger) Thing() bool { … }

func NewThinger() Thinger { return Thinger{ … } }
```
## 行长度
Go代码中没有严格的行长度限制，但要避免行太长。 同样，请勿添加换行符以使行在可读性更强的情况下保持较短（例如，如果它们是重复的）。

在大多数情况下，人们“不自然地”换行（在函数调用或函数声明的中间，或多或少，例如，尽管周围有一些例外情况），如果参数数量合理且合理，则不需要换行简短的变量名。 长行似乎包含长名，而摆脱长名会很有帮助。 

换句话说，换行是因为您所写内容的语义（作为一般规则），而不是因为行长。 如果发现行太长，则更改名称或语义，可能会得到不错的结果。 

实际上，对于功能应该持续多长时间，这是完全相同的建议。 没有规则“一个函数的长度不能超过N行”，但是肯定存在这样一个问题，即函数太长，微小函数过于紧凑，解决方案是更改函数边界的位置，而不是改变开始数行。

## Mixed Caps

请参阅https://golang.org/doc/effective_go.html#mixed-caps。 即使违反其他语言的约定也是如此。 例如，未导出的常量是maxLength而不是MaxLength或MAX_LENGTH。

另请参阅 [Initialisms](https://github.com/golang/go/wiki/CodeReviewComments#initialisms)。

## 命名返回值参数

考虑一下godoc中的表现。命名结果参数如下：
```
func (n *Node) Parent1() (node *Node) {}
func (n *Node) Parent2() (node *Node, err error) {}
```
会在godoc中繁杂；更好的表现：
```
func (n *Node) Parent1() *Node {}
func (n *Node) Parent2() (*Node, error) {}
```
另一方面，在某些情况下,如果函数返回两个或三个相同类型的参数， 或者如果从上下文中不清楚结果的含义，则添加名称可能很有用。不要仅仅为了避免在内部声明var而命名结果参数; 牺牲了一个小的实现简洁性，却以不必要的API详细程度为代价。
```
func (f *Foo) Location() (float64, float64, error)
```
还不如：
```
// Location returns f's latitude and longitude.
// Negative values mean south and west, respectively.
func (f *Foo) Location() (lat, long float64, err error)
```
如果函数只有几行，则可以使用裸返回。 一旦成为通用的函数，请明确说明您的返回值。 结论：不值得 命名结果参数的原因仅在于它使您可以使用裸返回。 文档的清晰性始终比在函数中保存一两行更为重要。

最后，在某些情况下，您需要命名结果参数才能进行更改 延迟关闭。 总是可以的。

## Naked Returns（裸返回）

请参阅 命名返回值参数 。

## 包注释

像godoc提出的所有注释一样，包注释必须出现在package子句的旁边，且不能有空行。
```
// Package math provides basic constants and mathematical functions.
package math

/*
Package template implements data-driven templates for generating textual
output such as HTML.
....
*/
package template
```

对于“ package main”注释，在二进制名称之后可以使用其他样式的注释（如果使用二进制格式，则可以大写），例如，对于 a package main目录中的，seedgen您可以编写：
```
// Binary seedgen ...
package main
```
或者
```
// Command seedgen ...
package main
```
或者
```
// Program seedgen ...
package main
```
或者
```
// The seedgen command ...
package main
```
或者
```
// The seedgen program ...
package main
```
或者
```
// Seedgen ..
package main
```
这些示例，以及这些的明智的变体是可以接受的。

请注意，包注释以小写单词开头的句子是不可可接受的，因为它们是公开可见的 应该用适当的英语写成，包括首字母大写的句子。当二进制名称是第一个单词时，将其大写即使它与拼写不完全匹配命令行调用也是必需的。

请参见 https://golang.org/doc/effective_go.html#commentary 有关注释约定的更多信息。

## 包名

您对包中所有名称的引用都将使用包名完成，因此您可以从标识符中省略该名称。例如，如果您在庞大的包中，您不需要输入ChubbyFile，客户端将其输入为 chubby.ChubbyFile。 而是命名类型 File，客户端将其写为 chubby.File。 避免使用无意义的程序包名称，例如util，common，misc，api，types 和interfaces。 请参阅 http://golang.org/doc/effective_go.html#package-names 和 http://blog.golang.org/package-names 以获得更多信息。

## 传递值

不要为了节省一些字节，将指针作为函数参数传递。 如果函数整个过程中都将 x 仅作为 *x 使用，则自变量不应是指针。常见的实例包括传递指向字符串（ *string的指针 ）或指向接口值的指针（ *io.Reader）。在这两种情况下，值本身都是固定大小，可以直接传递。这个建议不适用于大型结构，甚至不适用于可能增长的小型结构。

## 接收返回值的变量名称

方法的接收者的名称应反映其身份。 通常，其类型的一个或两个字母缩写就足够了（例如，“客户”是“ c”或“ cl”）。 不要使用通用名称，例如“ me”，“ this”或“ self”，这是面向对象语言的典型标识符，这些标识符赋予该方法特殊的含义。 在Go中，方法的接收者只是另一个参数，因此应相应地命名。 该名称不必像方法参数那样具有描述性，因为它的作用是显而易见的，没有任何文档目的。 它可能很短，因为它将出现在该类型的每种方法的几乎每一行上；熟悉承认简洁。也要保持一致：如果在一种方法中将接收器称为“ c”，则在另一种方法中请勿将其称为“ cl”。

## 接收返回值的变量类型

选择在方法上使用值接收器还是指针接收器可能很困难，特别是对于新的Go程序员而言。 如果有疑问，请使用指针，但是有时出于效率的原因（例如，小的不变结构或基本类型的值），值接收器才有意义。 一些有用的准则：

- 如果接收方是地图，函数或频道，请不要使用指向它们的指针。 如果接收方是切片，并且该方法未重新切片或重新分配切片，请不要使用指向该切片的指针。
- 如果该方法需要更改接收方，则接收方必须是指针。
- 如果接收方是包含sync.Mutex或类似同步字段的结构，则接收方必须是一个指针以避免复制。
- 如果接收器是大型结构或数组，则指针接收器效率更高。 多大？ 假设这等效于将其所有元素作为参数传递给方法。 如果感觉太大，则对于接收器来说也太大。
- 函数或方法（同时执行或从该方法调用时）是否会使接收方发生变化？ 值类型在调用方法时创建接收方的副本，因此外部更新将不会应用于此接收方。 如果更改必须在原始接收器中可见，则接收器必须是指针。
- 如果接收方是结构，数组或切片，并且其任何元素都是指向可能正在变异的对象的指针，则最好使用指针接收方，因为它会使读者更加清楚意图。
- 如果接收方是一个很小的数组或结构，自然是一个值类型（例如，诸如time.Time类型），没有可变字段且没有指针，或者仅仅是一个简单的基本类型（如int或string），价值接收者是有道理的。 值接收器可以减少可以生成的垃圾数量； 如果将值传递给value方法，则可以使用堆栈上的副本而不是在堆上分配。 （编译器会尽量避免这种分配，但是它不可能总是成功。）由于这个原因，请勿在没有进行概要分析的情况下选择值接收器类型。
- 最后，如有疑问，请使用指针接收器。

## 同步功能

首选同步函数-在异步函数上直接返回结果或在返回之前完成所有回调或通道操作的函数。

同步函数使goroutine在调用中保持本地状态，从而更容易推断其寿命，并避免泄漏和数据争用。 它们也更容易测试：调用方可以传递输入并检查输出，而无需轮询或同步。

如果调用者需要更多的并发性，则可以通过从单独的goroutine中调用该函数来轻松地添加它。 但是，在调用者端消除不必要的并发是非常困难的-有时是不可能的。

## 有用的测试失败

测试应该失败，并显示有用的消息，指出错误，输入了什么，实际得到了什么以及期望了什么。 编写一堆assertFoo帮助程序可能很诱人，但是请确保您的帮助程序会产生有用的错误消息。 试着假设调试您失败的测试的人不是您，也不是您的团队。 典型的Go测试失败，例如：
```
if got != tt.want {
	t.Errorf("Foo(%q) = %d; want %d", tt.in, got, tt.want) // or Fatalf, if test can't test anything more past this point
}
```
请注意，这里的顺序是实际的 != 预期的，并且消息也使用该顺序。 一些测试框架鼓励向后编写这些代码：0！= x，“预期为0，得到x”，依此类推。 Go不行。

如果这看起来很麻烦，那么您可能需要编写一个 [table-driven test](https://github.com/golang/go/wiki/TableDrivenTests) 。

当使用具有不同输入的测试助手时，消除失败测试歧义的另一种常用技术是用不同的TestFoo函数包装每个调用者，因此测试将以该名称失败：
```
func TestSingleValue(t *testing.T) { testHelper(t, []int{80}) }
func TestNoValues(t *testing.T)    { testHelper(t, []int{}) }
```
无论如何，您都有责任失败，并会向所有人发送有用的消息，以供将来调试您的代码的人使用。

## 变量名

Go中的变量名应该短而不是长。 对于范围有限的局部变量尤其如此。 比起 lineCount 更倾向于 c。比起 sliceIndex 更倾向于 i。

基本规则：名称声明越远，使用的名称就越具有描述性。 对于方法接收者，一个或两个字母就足够了。 如循环指数和读者共同的变量可以是单个字符（ i， r）。 更多不寻常的事物和全局变量需要更多描述性名称。 
