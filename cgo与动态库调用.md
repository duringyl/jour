# CGO调用编译好的动态库

## 一、理论

1. Go代码中使用`import "C"`启用CGO, import语句上方的注释为C代码, 可直接写C代码或者include别的代码库
2. 在使用动态链接库的情况下，Go程序编译过一次之后不用再编译, 更换so文件可直接替换代码运行逻辑

## 二、步骤概述

1. .h头文件中定义函数签名，Go会使用里面的签名所以.h文件定义好后不能改动, 例如`modEntryImpl.h`
2. 写C代码, 其中引用头文件并**实现**签名的函数(不同的代码运行逻辑)
3. 将C代码编译成动态库，**动态库名字固定**, 例如`libmodEntryImpl.so`(linux系统必须加lib前缀,和.so后缀)
4. 编写go代码, 在`import "C"`的注释中引用动态库，**动态库名字固定**, 在Go代码中调用C函数，并且最后编译成功
5. 运行Go编译出来的程序, 初次运行可能会错误因为会load不到动态库so文件, 需运行`export LD_LIBRARY_PATH=${动态库路径}`
6. 运行一般会成功
7. 转步骤2, 实现第二种运行逻辑, 将之前的动态库备份一下然后转步骤3逻辑编译成动态库(同名同路径)
8. 直接运行上次Go编译出来的程序, 看运行情况

## 三、文件树

```
.
├── go.mod
├── libc
│   ├── libmodEntryImpl.so	# 编译出的固定路径固定文件名动态库文件
│   ├── modEntryImpl.c		# 第一种运行逻辑
│   ├── modEntryImpl2.c		# 第二种运行逻辑
│   └── modEntryImpl.h		# 定义好的头文件
├── module
│   ├── mod_wrapper.go		# go代码, 其中import "C"
│   └── packet.go
├── main.go
└── readme.md
```

## 四、命令过程

```shell
cd ${go_src}/libc

# 写c代码实现h文件中签名的函数
vim modEntryImpl.c

# 将第一种实现编译成动态库
gcc -shared -fPIC -o libmodEntryImpl.so -I. modEntryImpl.c

# 编译go主程
cd ${go_src}
go build -o go_so.exe main.go

# 在环境变量设置动态链接库索引路径,非永久生效
export LD_LIBRARY_PATH=${go_src}/libc

# 运行程序, 观察结果
./go_so.exe


# 写第二种c代码实现、备份第一种的动态库、编译成动态库
cd libc && vim modEntryImpl2.c
mv libmodEntryImpl.so libmodEntryImpl_bk1.so
gcc -shared -fPIC -o libmodEntryImpl.so -I. modEntryImpl2.c

# 运行程序, 观察结果
cd .. && ./go_so.exe

```

## 五、代码

1. mod_wrapper.go + packet.go

```go
package module

/*
//gcc编译头文件索引路径-I
#cgo CFLAGS: -I../libc

// 从工作路径搜索动态库，不需环境路径: -Wl,-rpath,./
// gcc编译链接库路径-L, 链接库-l: modEntryImpl(实际名称libmodEntryImpl.so)
#cgo LDFLAGS: -L../libc -lmodEntryImpl  -Wl,-rpath,./

// 包含固定头文件
#include<modEntryImpl.h>

typedef int(createRunCtx)(void *pathPtr, int modPIdx);
*/
import "C"
import (
	"context"
	"unsafe"
)

type ModWrapper struct { }

func (w *ModWrapper) Entry(pk *Packet) {
	runCtx := pk.GetContext()
	buf := pk.Get()

	if pk.GetDir() == PkUL {
		C.EntryULImplementedInSo((*C.char)(unsafe.Pointer(&buf[0])), C.int(len(buf)), (*C.char)(runCtx))

		return
	}
	C.EntryDLImplementedInSo((*C.char)(unsafe.Pointer(&buf[0])), C.int(len(buf)), (*C.char)(runCtx))

}
```

```go
package module

import "unsafe"

const (
	PkUL = 0xF0
	PkDL = 0x0F
)

type Packet struct {
	Curr   uint32
	RunCtx []unsafe.Pointer
	Dir    uint8
	Buff   []byte
}

func (p *Packet) Get() []byte {
	return p.Buff
}

func (p *Packet) GetDir() uint8 {
	return p.Dir
}

func (p *Packet) GetContext() unsafe.Pointer {
	return p.RunCtx[p.Curr]
}
```

2. main.go

```go
package main

import (
	"fmt"
	"satellite/module"
	"unsafe"
)

func main() {
	mod := new(module.ModWrapper)

	ulStr := "ul bytes flow 0987654321 nmlkjihgfedcba"
	dlStr := "dl bytes flow 1234567890 abcdefghijklmn"

	pkD := &module.Packet{
		Curr:   0,
		Dir:    module.PkDL,
		Buff:   []byte(dlStr),
		RunCtx: make([]unsafe.Pointer, 1),
	}

	pkU := &module.Packet{
		Curr: 0,
		Dir:  module.PkUL,
		Buff: []byte(ulStr),
		RunCtx: make([]unsafe.Pointer, 1),
	}

	fmt.Println("go entry ul pk now:")
	mod.Entry(pkU)

	fmt.Println("go entry dl pk now:")
	mod.Entry(pkD)

}
```

3. modEntryImpl.h

```c
#ifndef __SCEPS_MOD_IMPLEMENT_INTERFACE__
#define __SCEPS_MOD_IMPLEMENT_INTERFACE__

int EntryULImplementedInSo(char *buf, int len, char *runCtx);
int EntryDLImplementedInSo(char *buf, int len, char *runCtx);

#endif
```

4. modEntryImpl.c

```c
#include <modEntryImpl.h>
#include<stdio.h>
#include<stdlib.h>
#include<string.h>

int EntryULImplementedInSo(char *buf, int len, char *runCtx){
    char *str = (char *)malloc(len + 1);
    memcpy(str, buf, len);
    str[len] = '\0';
    printf("this is mod entry UL implement 11111, the content is:\n");
    printf("\t%s\n", str);
}

int EntryDLImplementedInSo(char *buf, int len, char *runCtx){
    char *str = (char *)malloc(len + 1);
    memcpy(str, buf, len);
    str[len] = '\0';
    printf("this is mod entry DL implement 11111, the content is:\n");
    printf("\t%s\n", str);
}
```

5. modEntryImpl2.c

```c
#include <modEntryImpl.h>
#include<stdio.h>
#include<stdlib.h>
#include<string.h>

int EntryULImplementedInSo(char *buf, int len, char *runCtx){
    char *str = (char *)malloc(len + 1);
    memcpy(str, buf, len);
    str[len] = '\0';
    printf("mod entry UL 22222:\n");
    printf("\t%s\n", str);
}

int EntryDLImplementedInSo(char *buf, int len, char *runCtx){
    char *str = (char *)malloc(len + 1);
    memcpy(str, buf, len);
    str[len] = '\0';
    printf("mod entry DL 22222:\n");
    printf("\t%s\n", str);
}
```

6. go.mod

```
module satellite

go 1.20

```
