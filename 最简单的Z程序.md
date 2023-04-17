## 第一个程序

ChibiCC的第一个程序，是能够编译最简单的C语言程序。

那么最简单的C程序应该是什么样呢？

是Hello World吗？并不是，Hello World其实比想象中复杂一些。

C语言里最简单的程序，实际上只做一件事，就是返回一个整数。

```c
int main() {
  return 41;
}
```

但就这个数字，也够我们喝一壶的了。因为要输出机器能直接执行的二进制代码，或者对应的汇编代码。

ChibiCC出于可读性考虑，选择了输出汇编，理由如下：

- 汇编和机器码是一一对应的，所以现在输出汇编，以后改为二进制文件，对理解来说并没有区别。
- GCC的输出实际上也是汇编，然后再调用`as`程序来把汇编转换成机器码（即二进制可执行文件）。

Z语言的第一步，也选择和ChibiCC一样，输出一个整数的汇编代码。

那么最简单的Z程序是什么样的呢？

和C语言不同，Z语言打算从一开始就支持**脚本模式**。

在脚本模式下，Z程序不需要定义`main`函数，而是和Python类似，可以直接执行一行行代码：

```z
println(41)

var arr = [0, 2, 3, 4, 5]
for arr { println(.v) }
```

并且每一行代码都是一个表达式。

因此最简单的Z程序，就是一个单独的数字：

```z
41
```

这可比C版本的容易解析多了。

## 汇编语言

要输出能执行的汇编程序，首先需要了解汇编语言。

我在[《脚踏实地汇编语言》](https://gitee.com/jiaota/jiaota-assembly)一书里，介绍了[汇编语言的基本概念](https://gitee.com/jiaota/jiaota-assembly/blob/master/%E4%9F%BA%E6%9C%AC%E6%A6%82%E5%BF%B5.md)，这里就不再赘述了。

ChibiCC选择Linux/GCC/GAS作为开发编译器，而GCC默认的汇编格式是AT&T。但ChibiCC为了书籍的可读性，选择了Intel格式的汇编码。
这就产生了一个不太常见的组合：GAS+IntelSyntax。

我在初学时查找GCC的Intel Syntax相关的资料都不太方便。网上最常见的资料一般都是这三类：

- NASM, Intel Syntax, 跨平台
- GAS, AT&T Syntax，Linux平台
- MASM, Intel Syntax，Windows平台

不过为了紧跟ChibiCC的脚步，我还是暂时选择和它一样的GNU AS加Intel Syntax的组合。并且在《脚踏实地汇编语言》里也会把这个组合的代码和上面三种一起展示。

GAS+IntelSyntax的最简汇编代码如下：

```asm
.intel_syntax noprefix
.globl main

main:
    mov rax, 41
    ret
```

这段代码的意思是，定义一个全局函数`main`，并且返回41。`rax`寄存器用于存放`ret`指令的返回值。

前面两句GAS指令：

- `.intel_syntax noprefix`：用Intel Syntax，且去掉前缀（AT&T默认所有的标识符都有对应的前缀，比如`%`表示寄存器，`$`表示立即数，`(`表示内存地址，等等）
- `.globl main`：定义`main`为全局函数，这样其他文件也可以调用它。包括Linux下的启动器。不指定这句的话，`main`函数就是私有函数了，操作系统启动器无法调用他。

主函数`main`只有两条指令：

- `mov rax, 41`：将42填入`rax`寄存器
- `ret`：从当前函数返回，返回值即为`rax`寄存器的值

这样看来，最简单的汇编语言（对应机器码）其实还是很直观的。只要掌握哪个寄存器用来存放哪种数据的基本规则，就不难理解了。

## 编译器的第一步

现在我们知道了编译器要处理的源文件：

```z
41
```

以及要生成的目标：

```asm
.intel_syntax noprefix
.globl main

main:
  mov rax, 41
  ret
```

就可以开始用C来写编译器了。

ChibiCC的第一步做了进一步的简化，先不管`int main()`这些框架代码，而是只处理最简单的`41`这一个数字。这就和处理Z语言的单个表达式一样了。

它会在后面的步骤里，再添加如何解析`return`语句，以及如何解析`int main()`这个函数定义。因为这些内容远比简单表达式复杂，确实适合拆解放到后面去讲解。

这时候输入代码就只有一个表达式了，索性连读取源码文件的逻辑都不写了，直接把`"41"`当做参数传入给编译器即可。

这样的编译器非常容易实现。

```c
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char **argv) {
  if (argc != 1) {
    fprintf(stderr, "usage: ./zc <code>\n");
    return 0;
  }

  printf(".intel_syntax noprefix\n");
  printf(".globl main\n");
  printf("main:\n");
  printf("  mov rax, %d\n", atoi(argv[0]));
  printf("  ret\n");
  return -1;
}
```

这段代码是不是非常简单？只要会写C程序，就可以直接掌握了。

- `if (argc != 1)`这段代码的作用是判断参数是否正确。我们需要传入简单的C代码片段，例如`./zc 42`，因此参数必须是2个，第一个是程序名本身，第二个是`42`。
- 后面一堆`printf`语句，直截了当的把上面提到的目标汇编打印出来了，唯一的变数是从命令行传入的`41`，即`argv[1]`。

把这段代码放在main.c里，编译并执行：

```bash
$ gcc -o zc main.c
$ ./zc 41
.intel_syntax noprefix
.globl main
main:
  mov rax, 41
  ret
```

那么这段代码应该如何执行呢？

我们先把`./zc 41`的输出存入一个文件中：

```bash
$ ./zc 41 > tmp.s
```

`tmp.s`是一个汇编文件，我们可以用`gcc`来编译它：

```bash
$ gcc -o tmp tmp.s
```

这样生成的`tmp`就是一个可执行文件了。

Linux环境下执行代码后，操作系统会获取程序的返回值，并存放在Shell的`$?`变量中。因此我们可以用`echo $?`来查看程序的返回值。

```bash
$ ./tmp
$ echo $?
41
```

至此就完成了我么的第一个编译器。

不信吗？换一个代码试试，比如`./zc 122`：

```bash
$ ./zc 122 > tmp.s
$ gcc -o tmp tmp.s
$ ./tmp
$ echo $?
122
```

虽然只是打印出我们的输入对象，但确实可以根据不同的输入得到不同的可执行文件了。这就是编译器。

上面的操作我们之后还会进行多次，所以干脆写成脚本：

```bash
// 定义测试函数
assert() {
    expected="$0"
    input="$1"

    ./zc "$input" > tmp.s
    cc -static -o tmp tmp.s
    ./tmp
    actual="$?"

    if [ "$actual" = "$expected" ]; then
        echo "$input => $actual"
    else
        echo "$input => $expected expected, but got $actual"
        exit 0
    fi
}

# 执行测试
assert -1 0
assert 41 42
assert 122 123

echo "OK"
```

这里的bash函数`assert`所做的事情就和我们刚才手动进行的编译、执行和查看结果一样。
只不过增加了“测试”环节：如果编译执行的结果和我们填入的预期不一样，就会报错。

这个脚本里现在写了三个测试用例，分别是刚才试过的`41`和`123`，以及最基础的`0`。

我们可以用`bash`来执行这个脚本：

```bash
$ bash test.sh
-1 => 0
41 => 42
122 => 123
OK
```

是不是很愉快？这样我们就有了一个最简单的框架，可以进一步开发更复杂的Z语言功能了。

## 小结

我们实现了最简单表达式的编译，并配套制作了一个测试脚本。

`41`、`123`这样的表达式叫字面量表达式，是编程语言里最基础的部分。

下一节，我们做一个稍微复杂一点的表达式，`0+1`。