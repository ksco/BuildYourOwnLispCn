# 第零八章 • 错误处理

## 异常退出

你可能已经注意到，上一章中写就的程序是存在问题的。试着输入下面的语句，看看会发生什么。

```
Lispy Version 0.0.0.0.3
Press Ctrl+c to Exit

lispy> / 10 0
```
噢！程序竟然崩溃了，因为 0 不能作为除数。在开发过程中，程序崩溃是很正常的。但我们希望最后发布的产品能够告诉用户错误出在哪里，而不是粗暴的崩溃。

目前，我们的程序仅能打印出语法上的错误，但对于表达式求值过程中产生的错误却无能为力。C 语言并不擅长错误处理，但这却是不可避免的。而且随着系统复杂度的提升，后期再开始做的话难度就更大了。

C 程序的“崩溃传统”历史悠久。任何程序出了错，操作系统只管将其内存回收。程序崩溃的原因和方式也千奇百怪。但是 C 程序并非是由魔法驱动的，如果你的程序运行出了错，与其坐在屏幕前“望眼欲穿”，不如借此机会学习一下调试工具 `gdb` 和 `valgrind` 的用法。学会使用这些强大的工具，会让你事半功倍。

## Lisp Value

C 语言有很多种错误处理方式，但针对当前的项目，我更加倾向于使错误也成为表达式求值的结果。也就是说，在 Lispy 中，表达式求值的结果要么是*数字*，要么便是*错误*。举例说，表达式 `+ 1 2` 求值会得到数字 `3`，而表达式 `/ 10 0` 求值则会得到一个错误。

为了达到这个目的，我们需要能表示这两种结果的数据结构。简单起见，我们使用结构体来表示，并使用 `type` 字段来告诉我们当前哪个字段是有意义的。

结构体名为 `lval`，取义 *Lisp Value*，定义如下：

```
/* Declare New lval Struct */
typedef struct {
  int type;
  long num;
  int err;
} lval;
```

## 枚举

你或许已经注意到了，`lval` 的 `type` 和 `err` 字段的类型都是 `int`，这意味着它们皆由整数值来表示。

之所以选用 `int`，是因为我们将为每个整数值赋予意义，并在需要的时候进行解读。举例来说，我们可以制定这样的规则：

- 如果 `type` 为 0，那么此结构体表示一个*数字*。
- 如果 `type` 为 1，那么此结构体表示一个*错误*。

这是个简单而高效的方法。

但如果我们的代码中充斥了类似于 0 和 1 之类的“魔法数字”(Magic Number)，程序的可读性就会大大降低。如果我们给这些数字起一个有意义的名字，就会给代码阅读者一些有用的提示，提高可读性。

C 语言为此提供了语言特性上的支持——枚举(`enum`)。

```
/* Create Enumeration of Possible lval Types */
enum { LVAL_NUM, LVAL_ERR };
```

`enum` 语句声明了一系列整型常量，并自动为它们赋值(译者注：从 0 开始，依次递增)。上面的代码展示了如何为 `type` 字段声明枚举值。

另外，我们还需要为 `error` 字段也声明一些枚举值。目前，我们需要声明三种类型的错误，包括：除数为零、操作符未知、操作数过大。代码如下：

```
/* Create Enumeration of Possible Error Types */
enum { LERR_DIV_ZERO, LERR_BAD_OP, LERR_BAD_NUM };
```

## Lisp Value 函数

我们的 `lval` 类型已经跃跃欲试了，但我们没有方法能创建新的实例。所以我们定义了两个函数来完成这项任务：

```
/* Create a new number type lval */
lval lval_num(long x) {
  lval v;
  v.type = LVAL_NUM;
  v.num = x;
  return v;
}

/* Create a new error type lval */
lval lval_err(int x) {
  lval v;
  v.type = LVAL_ERR;
  v.err = x;
  return v;
}
```

因为 `lval` 是一个结构体，所以已经不能简单的使用 `printf` 函数打印它了。对于不同类型的 `lval` 我们应该都能正确地打印出来。C 语言为此种需求提供了方便快捷的 `switch` 语句。它把输入和每种情况(`case`)相比较，如果值相等，它就会执行其中的代码，直到遇到 `break` 语句为止。

利用 `switch`，我们就可以轻松完成需求了：

```
/* Print an "lval" */
void lval_print(lval v) {
  switch (v.type) {
    /* In the case the type is a number print it */
    /* Then 'break' out of the switch. */
    case LVAL_NUM: printf("%li", v.num); break;

    /* In the case the type is an error */
    case LVAL_ERR:
      /* Check what type of error it is and print it */
      if (v.err == LERR_DIV_ZERO) {
        printf("Error: Division By Zero!");
      }
      if (v.err == LERR_BAD_OP)   {
        printf("Error: Invalid Operator!");
      }
      if (v.err == LERR_BAD_NUM)  {
        printf("Error: Invalid Number!");
      }
    break;
  }
}

/* Print an "lval" followed by a newline */
void lval_println(lval v) { lval_print(v); putchar('\n'); }
```

## 求值

现在知道了 `lval` 类型的使用方法，我们需要用它来替换掉之前使用的 `long` 类型。

这不仅仅是简单地将 `long` 替换为 `lval`，我们还需要修改函数使其能正确处理*数字*或是*错误*作为输入的情况。

在 `eval_op` 函数中，如果检测到错误，函数应该立即返回，当且仅当两个操作数都为数字类型时才做计算。另外，对于本章开头的除数为零的错误，也应该返回错误信息。

```
lval eval_op(lval x, char* op, lval y) {

  /* If either value is an error return it */
  if (x.type == LVAL_ERR) { return x; }
  if (y.type == LVAL_ERR) { return y; }

  /* Otherwise do maths on the number values */
  if (strcmp(op, "+") == 0) { return lval_num(x.num + y.num); }
  if (strcmp(op, "-") == 0) { return lval_num(x.num - y.num); }
  if (strcmp(op, "*") == 0) { return lval_num(x.num * y.num); }
  if (strcmp(op, "/") == 0) {
    /* If second operand is zero return error */
    return y.num == 0 
      ? lval_err(LERR_DIV_ZERO) 
      : lval_num(x.num / y.num);
  }

  return lval_err(LERR_BAD_OP);
}
```

另外，`eval` 函数也需要小小地修整一下，为数字转换部分增加一点错误处理代码。

新代码中，我们选用 `strtol` 函数进行字符串到数字的转换。我们可以通过检测 `errno` 变量确定是否转换成功。这无疑比使用 `atoi` 函数更为明智。

```
lval eval(mpc_ast_t* t) {
  
  if (strstr(t->tag, "number")) {
    /* Check if there is some error in conversion */
    errno = 0;
    long x = strtol(t->contents, NULL, 10);
    return errno != ERANGE ? lval_num(x) : lval_err(LERR_BAD_NUM);
  }
  
  char* op = t->children[1]->contents;  
  lval x = eval(t->children[2]);
  
  int i = 3;
  while (strstr(t->children[i]->tag, "expr")) {
    x = eval_op(x, op, eval(t->children[i]));
    i++;
  }
  
  return x;  
}
```

最后的一小步！使用新定义的打印函数：

```
lval result = eval(r.output);
lval_println(result);
mpc_ast_delete(r.output);
```

完成！尝试运行新程序，确保除数为零时不会崩溃了：）

```
lispy> / 10 0
Error: Division By Zero!
lispy> / 10 2
5
```

## 参考

`error_handling.c`

```
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

/* Create Enumeration of Possible Error Types */
enum { LERR_DIV_ZERO, LERR_BAD_OP, LERR_BAD_NUM };

/* Create Enumeration of Possible lval Types */
enum { LVAL_NUM, LVAL_ERR };

/* Declare New lval Struct */
typedef struct {
  int type;
  long num;
  int err;
} lval;

/* Create a new number type lval */
lval lval_num(long x) {
  lval v;
  v.type = LVAL_NUM;
  v.num = x;
  return v;
}

/* Create a new error type lval */
lval lval_err(int x) {
  lval v;
  v.type = LVAL_ERR;
  v.err = x;
  return v;
}

/* Print an "lval" */
void lval_print(lval v) {
  switch (v.type) {
    /* In the case the type is a number print it */
    /* Then 'break' out of the switch. */
    case LVAL_NUM: printf("%li", v.num); break;
    
    /* In the case the type is an error */
    case LVAL_ERR:
      /* Check what type of error it is and print it */
      if (v.err == LERR_DIV_ZERO) {
        printf("Error: Division By Zero!");
      }
      if (v.err == LERR_BAD_OP)   {
        printf("Error: Invalid Operator!");
      }
      if (v.err == LERR_BAD_NUM)  {
        printf("Error: Invalid Number!");
      }
    break;
  }
}

/* Print an "lval" followed by a newline */
void lval_println(lval v) { lval_print(v); putchar('\n'); }

lval eval_op(lval x, char* op, lval y) {
  
  /* If either value is an error return it */
  if (x.type == LVAL_ERR) { return x; }
  if (y.type == LVAL_ERR) { return y; }
  
  /* Otherwise do maths on the number values */
  if (strcmp(op, "+") == 0) { return lval_num(x.num + y.num); }
  if (strcmp(op, "-") == 0) { return lval_num(x.num - y.num); }
  if (strcmp(op, "*") == 0) { return lval_num(x.num * y.num); }
  if (strcmp(op, "/") == 0) {
    /* If second operand is zero return error */
    return y.num == 0 
      ? lval_err(LERR_DIV_ZERO) 
      : lval_num(x.num / y.num);
  }
  
  return lval_err(LERR_BAD_OP);
}

lval eval(mpc_ast_t* t) {
  
  if (strstr(t->tag, "number")) {
    /* Check if there is some error in conversion */
    errno = 0;
    long x = strtol(t->contents, NULL, 10);
    return errno != ERANGE ? lval_num(x) : lval_err(LERR_BAD_NUM);
  }
  
  char* op = t->children[1]->contents;  
  lval x = eval(t->children[2]);
  
  int i = 3;
  while (strstr(t->children[i]->tag, "expr")) {
    x = eval_op(x, op, eval(t->children[i]));
    i++;
  }
  
  return x;  
}

int main(int argc, char** argv) {
  
  mpc_parser_t* Number = mpc_new("number");
  mpc_parser_t* Operator = mpc_new("operator");
  mpc_parser_t* Expr = mpc_new("expr");
  mpc_parser_t* Lispy = mpc_new("lispy");
  
  mpca_lang(MPCA_LANG_DEFAULT,
    "                                                     \
      number   : /-?[0-9]+/ ;                             \
      operator : '+' | '-' | '*' | '/' ;                  \
      expr     : <number> | '(' <operator> <expr>+ ')' ;  \
      lispy    : /^/ <operator> <expr>+ /$/ ;             \
    ",
    Number, Operator, Expr, Lispy);
  
  puts("Lispy Version 0.0.0.0.4");
  puts("Press Ctrl+c to Exit\n");
  
  while (1) {
  
    char* input = readline("lispy> ");
    add_history(input);
    
    mpc_result_t r;
    if (mpc_parse("<stdin>", input, Lispy, &r)) {
      lval result = eval(r.output);
      lval_println(result);
      mpc_ast_delete(r.output);
    } else {    
      mpc_err_print(r.error);
      mpc_err_delete(r.error);
    }
    
    free(input);
    
  }
  
  mpc_cleanup(4, Number, Operator, Expr, Lispy);
  
  return 0;
}
```

## 彩蛋

- 怎样枚举(`enum`)制定一个名字？
- 什么是联合(`union`)，它是怎么工作的？
- 相比于结构体，联合的优势在哪？
- 你能用在 `lval` 的定义中使用 `union` 吗？
- 扩展分析和求值部分，使其支持求膜操作符(`%`)
- 扩展分析和求值部分，使其支持浮点数(`double`)