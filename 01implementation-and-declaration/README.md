# 01实现和声明

```
package pkg

// 命令行执行: go tool compile -S main.go
var ID = 9527 // int

/*
go.cuinfo.packagename. SDWARFINFO dupok size=0
        0x0000 70 6b 67                                         pkg
"".ID SNOPTRDATA size=8
        0x0000 37 25 00 00 00 00 00 00                          7%......
 */

/*
size=8: 表示默认是int型变量, int型变量在amd64上是8个字节
NOPTRDATA: NO PTR DATA, 表示该数据是没有指针的, Go的自动内存管理器就不会去管它

0x0000 37 25 00 00 00 00 00 00 对应16进制格式 -> 0x2537 -> 十进制9527
amd64位的CPU是小端序的, 也就是从左到右的顺序是从低位到高位

上面的内容只是目标文件的汇编, 和Go语言的汇编还是不一样, 上面的汇编是引导到了本机的环境了
 */
```

Go汇编语言提供了DATA命令用于初始化包变量, DATA命令的语法如下: 

`DATA symbol+offset(SB)/width, value`

- symbol: 表示符号名称, 例如ID
- offset(SB)/width: offset表示变量在哪里定义, 在内存哪个地方, width就是size=8
- value: 初始值3725

使用以下命令来给ID变量初始化为十六进制的0x2537: 

小端序是这样定义的, 由低到高: 
```
// 将变量ID和内存0(SB)/1绑到一起
DATA ·ID+0(SB)/1, $0x37
DATA ·ID+1(SB)/1, $0x25
```

变量定义好之后需要导出以供其他代码引用, Go语言里有私有的名称和公有的名称, 汇编语言没有这个概念, 但是需要强制通过`GLOBAL`把符号导出来: 

```
// 语法规则: GLOBL symbol(SB), width(8字节)
GLOBL ·ID, $8
```

注意: 汇编里所有的数字需要以美元$符号开头, 类似于Go语言的常量的东西

有了上面的知识点, 开始编写实际的代码案例: 

编写变量的声明文件pkg.go: 

```
package main

var ID int

func main() {
	// 这里用println是为了尽可能的少引入内容
	println(ID)
}
``` 

编写变量的赋值文件pkg.s: 

```
GLOBL ·ID(SB),$8

DATA ·ID+0(SB)/1,$0x37
DATA ·ID+1(SB)/1,$0x25
DATA ·ID+2(SB)/1,$0x00
DATA ·ID+3(SB)/1,$0x00
DATA ·ID+4(SB)/1,$0x00
DATA ·ID+5(SB)/1,$0x00
DATA ·ID+6(SB)/1,$0x00
DATA ·ID+7(SB)/1,$0x00
```

然后执行`go run .`, 注意: 不要执行`go run pkg.go pkg.s`, 一定要把包当做一个整体来运行, 前期测试的时候, 不需要去写Hello World的程序, 这个程序对于汇编语言来说是一个很复杂的程序

package main

var Name = "gopher"

//func main() {
//	println(Name)
//}


/*
不要包含main函数, 否则编译出来的汇编代码会很长

go tool compile -S string.go

go.cuinfo.packagename. SDWARFINFO dupok size=0
        0x0000 6d 61 69 6e                                      main

// 这部分是初始化gopher的字符串
// 0x0000这个可以猜测是gopher字符串的地址
// 67 6f 70 68 65 72是gopher字符串的值 fmt.Printf("%x", []byte("gopher"))
// 因为Go语言的字符串并不是值类型，Go字符串其实是一种只读的引用类型。如果多个代码中出现了相同的"gopher"只读\字符串时，程序链接后可以引用的同一个符号go.string."gopher"
// S RO read only Data 该符号有一个SRODATA标志表示这个数据在只读内存段，dupok表示出现多个相同标识符的数据时只保留一个就可以了
// duplicates
go.string."gopher" SRODATA dupok size=6
        0x0000 67 6f 70 68 65 72                                gopher
""..inittask SNOPTRDATA size=24
        0x0000 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
        0x0010 00 00 00 00 00 00 00 00                          ........

"".Name SDATA size=16
        0x0000 00 00 00 00 00 00 00 00 06 00 00 00 00 00 00 00  ................
        rel 0+8 t=1 go.string."gopher"+0

要读懂上面的代码, 需要知道go里面的反射包reflect, 一个是数据的指针, 一部分是数据的长度
type StringHeader struct {
    Data uintptr // 在amd64的环境下 uintptr是8个字节，int也是8个字节, 正好16个字节
    Len  int
}

所以这里的16个字节和Name SDATA size=16是对应的
0x0000 00 00 00 00 00 00 00 00 这8个字节是空对应uintptr 对应的是0x0000 67 6f 70 68 65 72 这前4个零的地址
06 00 00 00 00 00 00 00 这8个字节对应的是字符串的长度, 6个字符
 */

## 定义字符串初始化的汇编代码

```pkg.go
GLOBL ·NameData(SB),$8
DATA ·NameData(SB)/8,$"gopher"

GLOBL ·Name(SB),$16
DATA ·Name+0(SB)/8,$·NameData(SB)
DATA ·Name+8(SB)/8,$6
```

```main.go
var Name string

func main() {
	println(Name)
}
```

执行`go run .`会报下面的错误: 

```
# implementation-and-declaration
main.NameData: missing Go type information for global symbol: size 8
```

意思是`pkg.s`中定义的NameData, Go语言需要知道它是否包含指针信息, 有两种办法, 一种是在pkg.go中声明一下: 

```golang
package main

import "fmt"

var NameData [8]byte
var Name string

func main() {
	fmt.Println(string(NameData[0:]))
	println(Name)
}
```

为什么上面声明一下就可以了, Go语言只关心一点, 就是里面有没有指针, 如果有指针的话, 进行垃圾回收的时候需要进行扫描变量里面的数据
如果没有指针, 它才不关心数据类型, 为什么会报错, 是和Go语言GC的工作机制有关系

或者在`pkg.s`中指明一下NameData的状态NOPTR: 

```pkg.s
// https://github.com/golang/go/blob/master/src/runtime/textflag.h
// 该文件就是一些宏定义
#include "textflag.h"

GLOBL ·NameData(SB),NOPTR,$8
DATA ·NameData(SB)/8,$"gopher"

GLOBL ·Name(SB),$16
DATA ·Name+0(SB)/8,$·NameData(SB)
DATA ·Name+8(SB)/8,$6
```

上面代码里, 是把NameData绑定到Name里了, 如果要修改字符串的内容, 在Go语言里是实现不了的, 或者只能使用unsafe包来改变, 但是在汇编语言里是可以的:

```package main
   
   var NameData [8]byte
   var Name string
   
   func main() {
   	println(Name)
   	NameData[0] = '?'
   	println(Name)
   }
``` 

所以说字符串的只读性, Go语言来保证, 汇编语言保证不了, 用汇编语言的人就是要灵活, 要实现这个功能

用汇编语言是为了性能和底层CPU特殊的功能, 所以不要跑偏了

```
GLOBL ·Name(SB),$24 // 24个字节, amd64位CPU寄存器的长度是8个字节, 再分配8个字节, 好对齐？
DATA ·Name+0(SB)/8,$·Name+16(SB) // $·Name+16(SB)是字符串值的内存地址  
DATA ·Name+8(SB)/8,$6 // 这里设置了字符串的长度, 多少长度就会显示几个字符串
DATA ·Name+16(SB)/8,$"gopher" // 这里填充了字符串的内容
```
这种把头部和数据打包到一起, 性能会很高, 因为它是紧密相关的, 如果分开的话, 相当于是有两端内存, 
通过头部, 再找到它的数据, 虽然都是两次, 但是离的很近, 只有16个偏移量, 这时性能会比较好, 当然不是真的好, 有可能会好
