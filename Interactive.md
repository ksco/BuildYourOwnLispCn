# 第零四章 • 交互

## 读取-求值-输出

在编写我们的 Lisp 之前，我们需要寻找一种和它交互的方式。最简单的方法，我们可以修改代码，重新编译，然后再次运行。这个方案虽然理论上可行，但是太为繁琐。如果可以动态的和程序进行交互，我们就可以快速的测试程序在各种条件下的行为。我们称这种模式为交互提示。

这种模式下的程序读取用户的输入，在程序内部进行处理，然后返回一些信息给用户。这种系统也被叫做 *REPL*，是 *read-evaluate-print loop*  (读取-求值-输出循环) 的简写。这种技术被广泛的应用在和编程语言的交互中，如果你学过 *Python*，那你一定不会陌生。

在编写一个完整的 *REPL* 之前，我们先实现一个简单的程序：读取用户的输入，简单处理后返回给用户。在后面的章节中，我们会对这个程序不断扩展，最后能够正确的读取并解析一个真正的 Lisp 程序，并将结果返回给用户。

## 交互提示

为了实现这个简单的想法，可以使用一个循环不断打印信息并等待用户输入。为了获取用户输入的内容，我们可以使用 `stdio.h` 中的 `fgets` 函数。这个函数可以一直读取直到遇到换行符为止。我们需要找个地方存储用户的输入。为此我们可以声明一个固定大小的缓冲区。

一旦获取到用户输入的字符串，就可以使用 `printf` 将它打印到命令行中。

```c

#include <stdio.h>

/* Declare a buffer for user input of size 2048 */
static char input[2048];

int main(int argc, char** argv) {

  /* Print Version and Exit Information */
  puts("Lispy Version 0.0.0.0.1");
  puts("Press Ctrl+c to Exit\n");

  /* In a never ending loop */
  while (1) {

    /* Output our prompt */
    fputs("lispy> ", stdout);

    /* Read a line of user input of maximum size 2048 */
    fgets(input, 2048, stdin);

    /* Echo input back to user */
    printf("No you're a %s", input);
  }

  return 0;
}

```

> 代码中的 `/*...*/` 是什么？

*这是 C 语言中的注释，是为了向其它阅读代码的人解释代码作用的。在编译的时候，会被编译器忽略掉。*

现在，我们来深入解读一下这个程序。

`static char input[2048];` 这行代码声明了一个拥有 2048 个字符长度的全局数组。这个数组中存储的数据我们可以在程序的任何地方获取到。我们会把用户在命令中输入的语句保存到这里面来。`static` 关键字标明这个数组仅在本文件中可见。`[2048]` 表明了数组的大小。

我们使用 `while(1)` 来构造一个无限循环，条件语句 `1` 永远都为真，所以这个循环会一直执行下去。

我们使用 `fputs` 打印提示信息。这个函数和前面介绍过的 `puts` 函数区别是 `fputs` 不会在末尾自动加换行符。我们使用 `fgets` 函数来获取用户在命令行中输入的字符串。这两个函数都需要指定写入或读取的文件。在这里，我们使用 `stdin` 和 `stdout` 作为输入和输出。这两个变量都是在 `<stdio.h>` 中定义的，用来表示向命令行进行输入和输出。当我们把 `stdin` 传给 `fgets` 后，它就会等待用户输入一串字符，并按下回车键。如果得到了字串，就会把字串连同换行符存放到 `input` 数组中。为了不让获取到的数据太大数组装不下，我们还要指定一下可以获取的最大长度为 `2048`。

我们使用 `printf` 函数将处理后的信息返回给用户。`printf` 允许我们同时打印多个不同类型的值。它会自动对第一个字符串参数中的模式进行匹配。例如，在上面的例子中，我们可以在第一个参数中看到you `%s` 字样。`printf` 将自动把 `%s` 替换为后面的参数中的值。`s` 代表字符串(`string`)。

更多关于 `printf` 的模式种类及其用法，可以参考[文档](http://en.cppreference.com/w/c/io/printf)。

> 我怎么才能知道一些类似于 `fgets` 或 `printf` 的函数？

*很明显你不可能一开始就知道这些标准库函数的作用和用法，这些都需要经验。幸运的是，C 语言的标准库非常精炼。绝大多数的库函数都可以在平时的练习中了解并学会使用。如果你想要解决的是底层的、基本的问题，关注一下[参考文档](http://en.cppreference.com/w/c)大有裨益，因为很可能标准库中的某个函数所做的事情正是你想要的！*

## 编译

你可以使用我们在第二章介绍过的命令行来编译刚刚这个程序：

`cc -std=c99 -Wall prompt.c -o prompt`

编译通过之后，你应该试着运行并测试一下这个程序。测试完成后，你可以使用 `Ctrl+c` 快捷键来退出程序。如果一切正常，你会得到类似于下面的结果：


```
Lispy Version 0.0.0.0.1
Press Ctrl+c to Exit

lispy> hello
No You're a hello
lispy> my name is Dan
No You're a my name is Dan
lispy> Stop being so rude!
No You're a Stop being so rude!
lispy>
```

## 编辑输入

如果你用的是 Mac 或 Linux，当你用左右箭头键编辑在程序中的输入时，你会遇到一个奇怪的问题：

```
Lispy Version 0.0.0.0.3
Press Ctrl+c to Exit
  
lispy> hel^[[D^[[C 
```

使用箭头键不会前后移动输入的光标，而是会产生像 `^[[D` 或 `^[[C` 这种奇怪的字符。很明显这不是我们想要的结果。

而在 Windows 上则不会有这个现象。

所以在 Mac 和 Linux 上，我们需要用到额外的一个库：`editline`。并把 `fputs` 和 `fgets` 替换为这个库提供的相同功能的函数。

如果你用的是 Windows 系统，你可以直接跳到本章的最后。因为接下来的几个小节都是和 `editline` 相关的内容。

## 使用 Editline 库

## 链接 Editline 并编译

## 预处理器

