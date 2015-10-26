# 第零四章 • 交互

## 读取-求值-输出

在编写我们的 Lisp 之前，我们需要寻找一种和它交互的方式。最简单的方法，我们可以修改代码，重新编译，然后再次运行。这个方案虽然理论上可行，但是太为繁琐。如果可以动态的和程序进行交互，我们就可以快速的测试程序在各种条件下的行为。我们称之为交互提示。

这种模式下的程序读取用户的输入，在程序内部进行处理，然后返回一些信息给用户。这种系统也被叫做 *REPL*，是 *read-evaluate-print loop*  (读取-求值-输出循环) 的简写。这种技术被广泛的应用在和编程语言的交互中，如果你学过 *Python*，那你一定不会陌生。

在编写一个完整的 *REPL* 之前，我们先实现一个简单的程序：读取用户的输入，然后原封不动的返回给用户。在后面的章节中，我们会对这个程序不断扩展，最后能够正确的读取并解析一个真正的 Lisp 程序，并将结果返回给用户。

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

**代码中的 `/*...*/` 是什么？**

*这是 C 语言中的注释，是为了向其它阅读代码的人解释代码作用的。在编译的时候，会被编译器忽略掉。*

