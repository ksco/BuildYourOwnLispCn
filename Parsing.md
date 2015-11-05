# 第零六章 • 语法分析

## 波兰表达式

为了验证 `mpc` 的威力，本章我们尝试实现一个简单的语法解析器－－[波兰表达式](en.wikipedia.org/wiki/Polish_notation)，它是我们将要实现的 Lisp 的数学运算部分。波兰表达式也是一种数学标记语言，它的运算符在操作数的前面。

举例来说：

| 普通表达式 | 波兰表达式 |
|-----------------|------------------------|
| `1 + 2 + 6` | `+ 1 2 6` |
| `6 + (2 * 9)` | `+ 6 (* 2 9)` |
| `(10 * 2) / (4 + 2)` | `/ (* 10 2) (+ 4 2)` |

现在，我们需要编写这种标记语言的语法规则。我们可以先用白话文来尝试描述它，而后再将其公式化。

我们观察到，波兰表达式总是以操作符开头，后面跟着操作数或其他的包裹在圆括号中的表达式。也就是说，“程序(`Program`)是由一个操作符(`Operator`)加上一个或多个表达式(`Expression`)组成的”，而 “表达式(`Expression`)可以是一个数字，或者是包裹在圆括号中的一个操作符(`Operator`)加上一个或多个表达式(`Expression`)”。

下面是一个更加规整的描述：

| 名称 | 定义 |
|-----------------|------------------------|
| 程序(`Program`) | `开始输入` --> `操作符` --> `一个或多个表达式(Expression)` --> `结束输入` |
| 表达式(Expression) | `数字、左括号 (、操作符` --> `一个或多个表达式` --> `右括号 )` |
| 操作符(Operator) | `'+'、'-'、'*' 、 '/'` |
| 数字(`Number`) | `可选的负号 -` --> `一个或多个 0 到 9 之间的字符` |

## 正则表达式

我们可以使用上一章学过的符号来表示大多数的规则，但是在表示*数字*和*程序*规则时可能会遇到一些问题。这些规则需要用到一些我们没有讲解过的符号。我们还不知道如何表达开始和结束输入、可选字符、字符范围等。

这些规则由正则表达式(Regular Expression)定义。正则表达式适合定义一些小型的语法规则，例如单词或是数字等。正则表达式不支持复杂的规则，但它清晰且精确地界定了输入是否符合规则。下面是正则表达式的基本规则：

| 语法表示 | 作用 |
|------------|-------------------------|
| `.` | 要求任意字符 |
| `a` | 要求字符 `a` |
| `[abcdef]` | 要求 `abcdef` 中的任意一个 |
| `[a-f]` | 要求按照字母顺序，`a` 到 `f` 中的任意一个 |
| `a?` | 要求 `a` 字符或什么都没有，即 `a` 为可选的 |
| `a*` | 要求有 0 个或多个字符 `a` |
| `a+` | 要求有 1 个或多个字符 `a` |
| `^` | 开始输入 |
| `$` | 结束输入 |

上面是我们目前需要的一些基本规则。如果你对正则表达式感兴趣，[这里](http://regex.learncodethehardway.org/)是关于它的一个完整的教程。

在 `mpc` 中，我们需要将正则表达式包裹在一对 `/` 中。例如，Number 可以用 `/-?[0-9]+/` 来表示。

## 安装 mpc

在我们正式编写这个语法解析器之前，正如之前在 Linux 和 Mac 上使用 `editline` 库一样，首先需要包含 `mpc` 的头文件，然后链接 `mpc` 库。

你可以直接使用第四章的代码，并将源文件重命名为 `parsing.c`，然后从 `mpc` 的[项目主页](http://github.com/orangeduck/mpc) 下载 `mpc.h` 和  `mpc.c`，放到和你的 `parsing.c` 同目录下。

在 `parsing.c` 的顶部添加 `#include "mpc.h"` 将 `mpc` 包含进来。将 `mpc.c` 放到命令行中来链接它。另外，在 Linux 上，还要加一个 -lm  参数来链接数学库。

Mac 和 Linux：

`cc -std=c99 -Wall parsing.c mpc.c -ledit -lm -o parsing`

Windows：

`cc -std=c99 -Wall parsing.c mpc.c -o parsing`

> 等一下，包含头文件难道不是用 `#include <mpc.h>`？

*事实上，在 C 语言中有两种包含头文件的方式，一种是用尖括号 `<>`，还有一种是用 `""` 双引号。通常，尖括号用来包含系统头文件如 `stdio.h`，双引号用来包含其他的头文件如 `mpc.h`。*

## 波兰表达式语法解析

本节把白话文叙述的规则用正式的描述语言编写，并在必要的地方使用正则表达式。下面就是波兰表达式最终的语法规则。认真阅读下方代码，验证其是否与之前叙述的规则相符。

```c
/* Create Some Parsers */
mpc_parser_t* Number   = mpc_new("number");
mpc_parser_t* Operator = mpc_new("operator");
mpc_parser_t* Expr     = mpc_new("expr");
mpc_parser_t* Lispy    = mpc_new("lispy");

/* Define them with the following Language */
mpca_lang(MPCA_LANG_DEFAULT,
  "                                                     \
    number   : /-?[0-9]+/ ;                             \
    operator : '+' | '-' | '*' | '/' ;                  \
    expr     : <number> | '(' <operator> <expr>+ ')' ;  \
    lispy    : /^/ <operator> <expr>+ /$/ ;             \
  ",
  Number, Operator, Expr, Lispy);
```

我们还需要将代码放入第四章编写的交互式程序。将上面的代码放在 `main` 函数的开头处，打印版本信息的代码之前。另外，在 `main` 函数的最后，还应该将使用完毕的解析器删除。只需要将下面的代码放在 `return` 语句之前即可。

/* Undefine and Delete our Parsers */
mpc_cleanup(4, Number, Operator, Expr, Lispy);

> 编译的时候得到一个错误：`undefined reference to 'mpc_lang'`

*注意函数的名字为 `mpca_lang`，`mpc` 后面有个 `a` 字母。*

## 解析用户输入

上面的代码为波兰表达式创建了一个 `mpc` 的解析器，本节我们就使用它来解析用户的每一条输入。我们需要更改之前的 `while` 循环，使它不再只是简单的将用户输入的内容打印回去，而是传进我们的解析器进行解析。我们可以把之前的 `printf` 语句替换成下面的代码：

```c
/* Attempt to Parse the user Input */
mpc_result_t r;
if (mpc_parse("<stdin>", input, Lispy, &r)) {
  /* On Success Print the AST */
  mpc_ast_print(r.output);
  mpc_ast_delete(r.output);
} else {
  /* Otherwise Print the Error */
  mpc_err_print(r.error);
  mpc_err_delete(r.error);
}
```

我们调用了 `mpc_parse` 函数，并将 `Lispy` 解析器和用户输入 `input` 作为参数。它将解析的结果保存到 `&r` 中，如果解析成功，返回值为 `1`，失败为 `0`。对 `r`，我们使用了取地址符 `&`，关于这个符号我们会在后面的章节中讨论。

- 解析成功时会产生一个内部结构，并保存到 `r` 的 `output` 字段中。我们可以使用 `mpc_ast_print` 将这个结构打印出来，使用 `mpc_ast_delete` 将其删除。
- 解析失败时则会将错误信息保存在 `r` 的 `error` 字段中。我们可以使用 `mpc_err_print` 将这个结构打印出来，使用 `mpc_err_delete` 将其删除。

重新编译程序，尝试不同的输入，看看程序的返回信息是什么。正常情况下返回值应该如下所示：

```
Lispy Version 0.0.0.0.2
Press Ctrl+c to Exit

lispy> + 5 (* 2 2)
>
  regex
  operator|char:1:1 '+'
  expr|number|regex:1:3 '5'
  expr|>
    char:1:5 '('
    operator|char:1:6 '*'
    expr|number|regex:1:8 '2'
    expr|number|regex:1:10 '2'
    char:1:11 ')'
  regex
lispy> hello
<stdin>:1:1: error: expected whitespace, '+', '-', '*' or '/' at 'h'
lispy> / 1dog
<stdin>:1:4: error: expected one of '0123456789', whitespace, '-', one or more of one of '0123456789', '(' or end of input at 'd'
lispy>
```

> 编译的时候得到一个错误：`<stdin>:1:1: error: Parser Undefined!`

*出现这个错误说明传给 `mpca_lang` 函数的语法规则存在错误，请仔细检查一下出错的地方。*

## 参考

`parsing.c`

```c
#include "mpc.h"

#ifdef _WIN32

static char buffer[2048];

char* readline(char* prompt) {
  fputs(prompt, stdout);
  fgets(buffer, 2048, stdin);
  char* cpy = malloc(strlen(buffer)+1);
  strcpy(cpy, buffer);
  cpy[strlen(cpy)-1] = '\0';
  return cpy;
}

void add_history(char* unused) {}

#else
#include <editline/readline.h>
#include <editline/history.h>
#endif

int main(int argc, char** argv) {
  
  /* Create Some Parsers */
  mpc_parser_t* Number   = mpc_new("number");
  mpc_parser_t* Operator = mpc_new("operator");
  mpc_parser_t* Expr     = mpc_new("expr");
  mpc_parser_t* Lispy    = mpc_new("lispy");
  
  /* Define them with the following Language */
  mpca_lang(MPCA_LANG_DEFAULT,
    "                                                     \
      number   : /-?[0-9]+/ ;                             \
      operator : '+' | '-' | '*' | '/' ;                  \
      expr     : <number> | '(' <operator> <expr>+ ')' ;  \
      lispy    : /^/ <operator> <expr>+ /$/ ;             \
    ",
    Number, Operator, Expr, Lispy);
  
  puts("Lispy Version 0.0.0.0.2");
  puts("Press Ctrl+c to Exit\n");
  
  while (1) {
  
    char* input = readline("lispy> ");
    add_history(input);
    
    /* Attempt to parse the user input */
    mpc_result_t r;
    if (mpc_parse("<stdin>", input, Lispy, &r)) {
      /* On success print and delete the AST */
      mpc_ast_print(r.output);
      mpc_ast_delete(r.output);
    } else {
      /* Otherwise print and delete the Error */
      mpc_err_print(r.error);
      mpc_err_delete(r.error);
    }
    
    free(input);
  }
  
  /* Undefine and delete our parsers */
  mpc_cleanup(4, Number, Operator, Expr, Lispy);
  
  return 0;
}
```

## 彩蛋

- 使用正则表达式编写规则，使其可以解析任意多个 `a`、`b` 组成的字符串，例如: `aababa`、`bbaa`。
- 使用正则表达式编写规则，使其可以解析任意多个交替出现的 `a`、`b` 组成的字符串，例如: `ababab`、`aba`。
- 修改语法规则，添加新的操作符，例如取余运算符 `%`。
- 修改语法规则，使其可以解析用文本形式的操作符，例如：`add`、`sub`、`mul`、`div`等。
- 修改语法规则，使其能够处理浮点数，例如：`0.01`、`5.21`、`10.2`。
