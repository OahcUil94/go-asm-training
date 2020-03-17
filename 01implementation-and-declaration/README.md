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
