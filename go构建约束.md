## Go源文件中的 "//go:build"

### 构建约束

在看别人源码的时候有时会看到类似这样打头的注释:

    //go:build xxx

比如gin的gin/internal/json包里面就有。

这到底什么意思呢?

这其实是go里面的构建约束，也叫构建标签，build的时候给编译器的标签，可以决定一个文件是否需要包含在包之中。

比如:

    //go:build (linux && 386) || (darwin && !cgo)

表示执行go build的时候, 目标系统是386的Linux或者没有启用cgo的darwin时, 当前文件会被编译进来。

这个就是构建(编译)约束。

### 特性

- 可以出现在任何源文件里面, 不仅仅是go源文件
- 必须写在文件顶部, 前面只可以有空白行或者其他注释。
- 为了区分构建约束和包文件, 在构建约束后面应该有一空白行
- 由||、&&、!运算符(与、或、非)和括号组成的表达式

### 可用的标签

- 目标操作系统, 拼写和runtime.GOOS一致, 可以用命令go tool dist list查看可能的值
- 目标系统架构, 拼写和runtime.GOARCH一致
- unix, 标识GOOS是Unix或者Unix-like系统
- 正在使用的编译器, gc或者gccgo
- cgo, 如果支持cgo命令
- 每个go主要发布版本的术语, 比如: `go1.1`, 满足从go1.1版本开始, `go1.12`从go1.12版本开始, 其他类推
- 通过`-tags`标记给出的其他标签, `-tags`多个标签用逗号分隔列出
    
    比如gin的gin/internal/json就是通过标签来确定使用哪个json库的:

        go build -tags=jsoniter
        go build -tags=go_json

- `go:build ignore`: 忽略编译

### 其他方式

如果一个文件名称, 满足如下条件:

    *_GOOS
    *_GOARCH
    *_GOOS_GOARCH

比如一个文件名称`source_windows_amd64.go`当目标系统是Windows且CPU架构为amd64时会被包含编译, GOOS和GOARCH分别代表目标系统的类别和(处理器)架构类型, 满足上面规则的文件(名)被认为需要满足这些条件的隐式约束。

例如:

    dns_windows.go: 编译Windows包时才会编译本文件
    math_386.s: 编译包目标是32-bit x86时包含

这样的好处就是, 当同一个方法在不同操作系统下实现是不一样的时候, 可以用相同的函数名称分别实现, 编译的时候根据目标系统类型编译对应系统的源文件。

### 注意事项

go 1.16和之前的版本的语法是使用`// +build`, gofmt命令在遇到旧语法时会自动添加等效的新语法

[参考](https://segmentfault.com/a/1190000042956051?utm_source=sf-similar-article)
