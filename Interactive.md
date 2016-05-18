# 第零四章 • 交互

## 读取-求值-输出

在编写我们的 Lisp 之前，我们需要寻找一种和它交互的方式。最简单的方法，我们可以修改代码，重新编译，然后再次运行。这个方案虽然理论上可行，但是太为繁琐。如果可以动态地和程序进行交互，我们就可以快速地测试程序在各种条件下的行为。我们称这种模式为交互提示。

这种模式下的程序读取用户的输入，在程序内部进行处理，然后返回一些信息给用户。这种系统也被叫做 *REPL*，是 *read-evaluate-print loop*  (读取-求值-输出循环) 的简写。这种技术被广泛地应用在各种编程语言的解释器中，如果你学过 *Python*，那你一定不会陌生。

在编写一个完整的 *REPL* 之前，我们先实现一个简单的程序：读取用户的输入，简单处理后返回给用户。在后面的章节中，我们会对这个程序不断扩展，最后能够正确地读取并解析一个真正的 Lisp 程序，并将结果返回给用户。

## 交互提示

为了实现这个简单的想法，可以使用一个循环不断打印信息并等待用户输入。为了获取用户输入的内容，我们可以使用 `stdio.h` 中的 `fgets` 函数。这个函数可以一直读取直到遇到换行符为止。我们需要找个地方存储用户的输入。为此可以声明一个固定大小的数组缓冲区。

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

> #### 代码中的 `/*...*/` 是什么？

> 这是 C 语言中的注释，是为了向其它阅读代码的人解释代码作用的。在编译的时候，会被编译器忽略掉。

现在来深入解读一下这个程序。

`static char input[2048];` 这行代码声明了一个拥有 2048 个字符长度的全局数组。这个数组中存储的数据可以在程序的任何地方获取到。我们会把用户在命令中输入的语句保存到这里面来。`static` 关键字标明这个数组仅在本文件中可见。`[2048]` 表明了数组的大小。

我们使用 `while(1)` 来构造一个无限循环，条件语句 `1` 永远都为真，所以这个循环会一直执行下去。

我们使用 `fputs` 打印提示信息。这个函数和前面介绍过的 `puts` 函数区别是 `fputs` 不会在末尾自动加换行符。我们使用 `fgets` 函数来获取用户在命令行中输入的字符串。这两个函数都需要指定写入或读取的文件。在这里，我们使用 `stdin` 和 `stdout` 作为输入和输出。这两个变量都是在 `<stdio.h>` 中定义的，用来表示向命令行进行输入和输出。当我们把 `stdin` 传给 `fgets` 后，它就会等待用户输入一串字符，并按下回车键。如果得到了字串，就会把字串连同换行符存放到 `input` 数组中。为了不让获取到的数据太大数组装不下，我们还要指定一下可以获取的最大长度为 `2048`。

我们使用 `printf` 函数将处理后的信息返回给用户。`printf` 允许我们同时打印多个不同类型的值。它会自动对第一个字符串参数中的模式进行匹配。例如，在上面的例子中，可以在第一个参数中看到 `%s` 字样。`printf` 将自动把 `%s` 替换为后面的参数中的值。`s` 代表字符串(`string`)。

更多关于 `printf` 的模式种类及其用法，可以参考[文档](http://en.cppreference.com/w/c/io/printf)。

> #### 我怎么才能知道一些类似于 `fgets` 或 `printf` 的函数的用法？

> 很明显你不可能一开始就知道这些标准库函数的作用和用法，这些都需要经验。幸运的是，C 语言的标准库非常精炼。绝大多数的库函数都可以在平时的练习中了解并学会使用。如果你想要解决的是底层的、基本的问题，关注一下[参考文档](http://en.cppreference.com/w/c)会大有裨益，因为很可能标准库中的某个函数所做的事情正是你想要的！

## 编译

你可以使用在第二章介绍过的命令来编译上面的程序：

`cc -std=c99 -Wall prompt.c -o prompt`

编译通过之后，你应该试着运行并测试一下这个程序。测试完成后，可以使用 `Ctrl+c` 快捷键来退出程序。如果一切正常，你会得到类似于下面的结果：


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

在 Mac 和 Linux 上，我们需要用到 `editline` 库来解决这个问题。并把 `fputs` 和 `fgets` 替换为这个库提供的相同功能的函数。

如果你用的是 Windows 系统，则可以直接跳到本章的最后。因为接下来的几个小节都是和安装与配置 `editline` 相关的内容。

## 使用 editline 库

我们会用到 `editline` 库提供的两个函数：`readline` 和 `add_history`。`readline` 和 `fgets` 一样，从命令行读取一行输入，并且允许用户使用左右箭头进行编辑。`add_history` 可以纪录下我们之前输入过的命令，并使用上下箭头来获取。新的程序如下所示：

```c
#include <stdio.h>
#include <stdlib.h>

#include <editline/readline.h>
#include <editline/history.h>

int main(int argc, char** argv) {
   
  /* Print Version and Exit Information */
  puts("Lispy Version 0.0.0.0.1");
  puts("Press Ctrl+c to Exit\n");
   
  /* In a never ending loop */
  while (1) {
    
    /* Output our prompt and get input */
    char* input = readline("lispy> ");
    
    /* Add input to history */
    add_history(input);
    
    /* Echo input back to user */    
    printf("No you're a %s\n", input);

    /* Free retrieved input */
    free(input);
    
  }
  
  return 0;
}
```

我们增加了一些新的头文件。`<stdlib.h>` 提供了 `free` 函数。`<editline/readline.h>` 和 `<editline/history.h>` 提供了 `editline` 库中的 `readline` 和 `add_history` 函数。

在上面的程序中，我们使用 `readline` 读取用户输入，使用 `add_history` 将该输入添加到历史纪录当中，最后使用 `printf` 将其加工并打印出来。

与 `fgets` 不同的是，`readline` 并不在结尾添加换行符。所以我们在 `printf` 函数中添加了一个换行符。另外，我们还需要使用 `free` 函数手动释放 `readline` 函数返回给我们的缓冲区 `input`。这是因为 `readline` 不同于 `fgets` 函数，后者使用已经存在的空间，而前者会申请一块新的内存，所以需要手动释放。内存的申请与释放问题我们会在后面的章节中深入讨论。

## 链接 editline 并编译

如果你使用前面我们提供的命令行来编译这个程序，你会得到类似于下面的错误，因为在使用之前，你必须先在电脑上安装 `editline` 库。

`fatal error: editline/readline.h: No such file or directory #include <editline/readline.h>`

在 Mac 上，`editline` 包含在 *Command Line Tools* 中，安装方法我们在第二章有说明。安装完后，可能还是会出现头文件不存在的编译错误。这时，可以移除 `#include <editline/history.h>` 这行代码，再试一次。

在 Linux 上，可以使用 `sudo apt-get install libedit-dev` 来安装 `editline`。在 Fedora 上，使用 `su -c "yum install libedit-dev*"` 命令安装。

一旦你安装好了 `editline`，你可以再次编译试一下。然后将会得到如下的错误：

```
undefined reference to `readline'
undefined reference to `add_history'
```

这是因为没有将 `editline` 链接到程序中。我们需要使用 `-ledit` 标记来完成链接，用法如下：

`cc -std=c99 -Wall prompt.c -ledit -o prompt`

再次运行程序，就可以自由地编辑输入的文字了！

> #### 为什么我的程序还是不能编译？

>在有些系统上，`editline` 的安装、包含、链接的方式可能会有些许差别，请善用搜索引擎哦。

## 预处理器

对于这样的一个小程序而言，我们针对不同的系统编写不同的代码是可以的。但是如果我把我的代码发给一个使用不同的操作系统的朋友，让他帮我完善一下代码，可能就会出问题了。理想情况下，我希望我的代码可以在任何操作系统上编译并运行。这在 C 语言中是个很普遍的问题，叫做可移植性(portability)。这通常都是个很棘手的问题。

但是 C 提供了一个机制来帮助我们，叫做预处理器(preprocessor)。

预处理器也是一个程序，它在编译之前运行。它有很多作用。而我们之前就已经悄悄地用过预处理器了。任何以井号 `#` 开头的语句都是一个预处理命令。为了使用标准库中的函数，我们已经用它来包含(include)过头文件了。

预处理的另一个作用是检测当前的代码在哪个操作系统中运行，从而来产生平台相关的代码。而这也正是我们做可移植性工作时所需要的。

在 Windows 上，我们可以伪造一个 `readline` 和 `add_history` 函数，而在其他系统上就使用 `editline` 库提供给我们的真正有作用的函数。

为了达到这个目的，我们需要把平台相关的代码包在`#ifdef`、`#else` 和 `#endif` 预处理命令中。如果条件为真，包裹在 `#ifdef` 和 `#else` 之间的代码就会被执行，否则，`#else` 和 `endif` 之间的代码被执行。通过这个特性，我们就能写出在 Windows、Linux 和 Mac 三大平台上都能正确编译的代码了：

```c
#include <stdio.h>
#include <stdlib.h>

/* If we are compiling on Windows compile these functions */
#ifdef _WIN32
#include <string.h>

static char buffer[2048];

/* Fake readline function */
char* readline(char* prompt) {
  fputs(prompt, stdout);
  fgets(buffer, 2048, stdin);
  char* cpy = malloc(strlen(buffer)+1);
  strcpy(cpy, buffer);
  cpy[strlen(cpy)-1] = '\0';
  return cpy;
}

/* Fake add_history function */
void add_history(char* unused) {}

/* Otherwise include the editline headers */
#else
#include <editline/readline.h>
#include <editline/history.h>
#endif

int main(int argc, char** argv) {
   
  puts("Lispy Version 0.0.0.0.1");
  puts("Press Ctrl+c to Exit\n");
   
  while (1) {
    
    /* Now in either case readline will be correctly defined */
    char* input = readline("lispy> ");
    add_history(input);

    printf("No you're a %s\n", input);
    free(input);
    
  }
  
  return 0;
}
```

## 参考

`prompt_unix.c`

```c
#include <stdio.h>
#include <stdlib.h>

#include <editline/readline.h>
#include <editline/history.h>

int main(int argc, char** argv) {
   
  /* Print Version and Exit Information */
  puts("Lispy Version 0.0.0.0.1");
  puts("Press Ctrl+c to Exit\n");
   
  /* In a never ending loop */
  while (1) {
    
    /* Output our prompt and get input */
    char* input = readline("lispy> ");
    
    /* Add input to history */
    add_history(input);
    
    /* Echo input back to user */    
    printf("No you're a %s\n", input);

    /* Free retrived input */
    free(input);
    
  }
  
  return 0;
}
```

`prompt_windows.c`
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

`prompt.c`
```c
#include <stdio.h>
#include <stdlib.h>

/* If we are compiling on Windows compile these functions */
#ifdef _WIN32
#include <string.h>

static char buffer[2048];

/* Fake readline function */
char* readline(char* prompt) {
  fputs(prompt, stdout);
  fgets(buffer, 2048, stdin);
  char* cpy = malloc(strlen(buffer)+1);
  strcpy(cpy, buffer);
  cpy[strlen(cpy)-1] = '\0';
  return cpy;
}

/* Fake add_history function */
void add_history(char* unused) {}

/* Otherwise include the editline headers */
#else
#include <editline/readline.h>
#include <editline/history.h>
#endif

int main(int argc, char** argv) {
   
  puts("Lispy Version 0.0.0.0.1");
  puts("Press Ctrl+c to Exit\n");
   
  while (1) {
    
    /* Now in either case readline will be correctly defined */
    char* input = readline("lispy> ");
    add_history(input);

    printf("No you're a %s\n", input);
    free(input);
    
  }
  
  return 0;
}
```

## 彩蛋

- 将提示信息 `lispy>` 换成其他你喜欢的。
- 修改打印的信息。
- 在程序开头的提示信息中添加一些其他的信息。
- 在字符串中，`\n` 表示什么？
- `printf` 还有哪些输出模式？
- 如果你向 `printf` 传递一个与模式不匹配的值会怎样？
- 预处理器 `#ifndef` 有什么用？
- 预处理器 `#define` 有什么用？
- `_WIN32` 在 Windows 中有定义，那在 Linux 和 Mac 中定义了什么呢？
