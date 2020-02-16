---
title: "GCC 内联汇编"
date: 2020-02-03T19:55:21+08:00
draft: false
---
GCC 支持内联汇编，
格式如下：

```asm
asm ( assembler template
    : output operands               (optional)
    : input operands                (optional)
    : list of clobbered registers       (optional)
    );
```
`asm` 又可以写作 `__asm__`, 
`__asm__`主要用来避免命名冲突。
`assembler template` 就是内联的汇编代码，
`output operands`, `input operands`, `list of clobbered registers`分别指代
输入操作数，输入操作数，修饰寄存器列表。
参数的顺序从左到右使用数字序号来引用，
例如`%0`表示第一个参数。
以下是几个 GCC 内联汇编的例子：

## 1. 内存操作数(Memory operand constraint(m))

```C
__asm__("sidt %0\n" : :"m"(loc));
```

以上代码的作用等同于`*loc = idt`（idt 表示中断向量表）,
`"m"`表示操作数位于内存中，
其中`%0`表示第一个参数也就是`loc`。

## 2. 参数序号引用(Matching(Digit) constraints)

```C
__asm__("incl %0" :"=a"(var):"0"(var));
```

在这个例子中`var`既用作输入参数由用作输出参数，
`=a`表示使用`eax`寄存器来存放变量`var`，
`=`是修饰符，表示输出变量，
它告诉 GCC 这个变量的值会被覆盖。
"0"表示使用和第一个参数一样的存储来作为输出来源，
在这里就是`eax`寄存器。
这段代码展开后等价于：

```asm
mov var %eax
incl %eax
mov %eax, var
```

## 3. 常用寄存器的代号

我们知道`a`代表`eax`寄存器，x86 还有多种可用的寄存器，代号如下：

```txt
a %eax
b     %ebx
c     %ecx
d     %edx
S %esi
D %edi
```
还有一个特殊的代号`r`表示由 GCC 来选择可用的寄存器。

## 4. 修饰寄存器(clobbered register)

修饰寄存器可以理解为保留的不可用的寄存器。

```C
asm ("movl $count, %%ecx;
      up: lodsl;
      stosl;
      loop up;"
    :           /* no output */
    :"S"(src), "D"(dst) /* input */
    :"%ecx", "%eax" );  /* clobbered list */
```

`lodsl`和`stosl`会隐式的使用`eax`和`ecx`寄存器。
但是 GCC 并不知道这一点，
所以它可能会在展开的代码中使用这两个寄存器，
为了避免这种情况我们需要把`eax`和`ecx`添加到修饰寄存器列表中。

## 5. 参数修饰符

上面的例子中我们已经见过`=`修饰符，除了`=`以外还有三种修饰符：

1. `+`表示操作数在一条指令中既被读又背写
2. `&`表示表示输出寄存器不能和输入寄存器重叠，只能用来做输出
3. `%`告诉 GCC 该操作数可能被累加

## 6. 内联汇编的综合例子

```C
int main(void)
{
    int x = 10, y;

    asm ("movl %1, %%eax;
         "movl %%eax, %0;"
        :"=r"(y)    /* y is output operand */
        :"r"(x)     /* x is input operand */
        :"%eax");   /* %eax is clobbered register */
}
```

以上代码展开后就是：

```asm
    main:
        pushl %ebp
        movl %esp,%ebp
        subl $8,%esp
        movl $10,-4(%ebp)
        movl -4(%ebp),%edx  /* x=10 is stored in %edx */
#APP    /* asm starts here */
        movl %edx, %eax     /* x is moved to %eax */
        movl %eax, %edx     /* y is allocated in edx and updated */

#NO_APP /* asm ends here */
        movl %edx,-8(%ebp)  /* value of y in stack is updated with
                   the value in %edx */
```

`eax`在修饰集群器列表中，
所以我们看到内联汇编展开的代码中没有再引用`eax`寄存器。
可以看到变量`x`和`y`都使用`edx`寄存器存储，
有时候需要保证输入、输出变量使用不同的寄存器，
这个是可以使用`&`修饰符：

```C
    int main(void)
{
    int x = 10, y;

    asm ("movl %1, %%eax;
         "movl %%eax, %0;"
        :"=&r"(y)   /* y is output operand, note the
                   & constraint modifier. */
        :"r"(x)     /* x is input operand */
        :"%eax");   /* %eax is clobbered register */
}
```

展开后的代码是：

```asm
    main:
        pushl %ebp
        movl %esp,%ebp
        subl $8,%esp
        movl $10,-4(%ebp)
        movl -4(%ebp),%ecx  /* x, the input is in %ecx */
#APP
    movl %ecx, %eax
    movl %eax, %edx     /* y, the output is in %edx */

#NO_APP
        movl %edx,-8(%ebp)
```
可以看到现在`x`存在于`ecx`寄存器，
而`y`存放于`edx`寄存器。
