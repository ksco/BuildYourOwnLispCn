# 第零七章 • 计算

## 树型结构

现在，我们可以读取输入，解析并得到表达式的内部结构。但是并不能对它进行计算。本章我们将编写代码，对表达式的内部结构进行计算求值。

所谓的内部结构就是上一章中我们打印出来的内容。它被称为*抽象语法树*(Abstract Syntax Tree，简称 AST)。它用来表示用户输入的表达式的结构。操作数和操作符等需要被处理的实际数据都位于叶子节点上。而非叶子节点上则包含了遍历和求值的信息。

在做解析和求值之前，我们先来看一下数据结构具体的内部表示。在 `mpc.h` 中，可以找到 `mpc_ast_t` 类型的定义，这里面就是我们解析表达式得到的数据结构。

```c
typedef struct mpc_ast_t {
  char* tag;
  char* contents;
  mpc_state_t state;
  int children_num;
  struct mpc_ast_t** children;
} mpc_ast_t;
```

下面来逐一的看看结构体各个字段的含义。

第一个为 `tag` 字段。当我们打印这个树形结构时，`tag` 就是在节点内容之前的信息，它表示了解析这个节点时所用到的所有规则。例如：`expr|number|regex`。

`tag` 字段非常重要，因为它可以让我们知道创建节点时需要用到的规则。

第二个是 `contents` 字段，它包含了节点中具体的内容，例如 `'*'`、`'('`、`'5'`。你会发现，对于表示分支的非叶子节点，这个字段为空。而对于叶子节点，则包含了操作数和操作符。

下一个字段是 `state`。这里面包含了解析器发现这个节点时所处的状态，例如在代码中的行数和列数等。在我们的程序中不会用到这个字段。

最后的两个字段 `children_num` 和 `children` 帮助我们来遍历抽象语法树。前一个字段告诉我们有多少个孩子节点，后一个字段是包含这些节点的数组。

其中，`children` 字段的类型是 `mpc_ast_t**`。这是一个二重指针类型。实际上，它并不像看起来那么可怕，我们会在后面的章节中详细解释它。现在你只需要知道它是孩子节点的列表即可。

我们可以对 `children` 使用数组的语法，在其后使用 `[x]` 来获取某个下标的值。比如，可以用 `children[0]` 来获取第一个孩子节点。注意，在 C 语言中数组是从 `0` 开始计数的。

因为 `mpc_ast_t*` 是指向结构体的指针类型，所以获取其字段的语法有些许不同。我们需要使用 `->` 符号，而不是 `.` 符号。

```c
/* Load AST from output */
mpc_ast_t* a = r.output;
printf("Tag: %s\n", a->tag);
printf("Contents: %s\n", a->contents);
printf("Number of children: %i\n", a->children_num);

/* Get First Child */
mpc_ast_t* c0 = a->children[0];
printf("First Child Tag: %s\n", c0->tag);
printf("First Child Contents: %s\n", c0->contents);
printf("First Child Number of children: %i\n",
  c0->children_num);
```

## 递归(Recursion)

树形结构是自身重复的。树的每个孩子节点都是树，每个孩子节点的孩子节点也是树，以此类推。正如编程语言一样，树形结构也是递归和重复的。显然，如果我们想编写函数处理所有可能的情况，就必须要保证函数可以处理任意深度。幸运的是，我们可以使用递归的天生优势来轻松地处理这种重复自身的结构。

简而言之，递归函数就是在执行的过程中调用自身的函数。这听起来可能有些奇怪，因为这会导致函数无穷尽地执行下去。但是函数对于不同的输入会产生不同的输出，如果我们每次递归都改变或使用不同的输入，并设置递归终止的条件，我们就可以使用递归做一些有用的事情。

举个例子，我们可以使用递归来计算树形结构中节点个数。

首先考虑最简单的情况，如果输入的树没有子节点，我们只需简单的返回 `1` 就行了。如果输入的树有一个或多个子节点，这时返回的结果就是自身节点的 `1`，加上所有子节点的值。

但我们怎样得到所有子节点的值呢？这正是我们要着手解决的问题。请看下面的 C 语言代码。

```c
int number_of_nodes(mpc_ast_t* t) {
  if (t->children_num == 0) { return 1; }
  if (t->children_num >= 1) {
    int total = 1;
    for (int i = 0; i < t->children_num; i++) {
      total = total + number_of_nodes(t->children[i]);
    }
    return total;
  }
}
```

递归函数的定义可能看起来有些奇怪。我们首先假定存在某个函数能够正确的工作，然后使用它来编写刚刚假定存在的函数。

正如世间万物，递归函数也有规律可循。首先需要定义的是最基本的情况，用来终止递归的执行。例如上例中的 `t->children_num == 0`。之后定义的是需要递归的情况，例如上例中的 `t->children_num >= 1`。这部分将计算过程分为几个相似的部分，然后调用自身来递归处理这些部分，最后将结果整合起来。

理解递归函数需要动一番脑筋，在继续之前，请确保自己理解了上面的内容。在后面的章节中，我们会大量地用到递归。如果你还是不清楚递归的原理，请参考本章福利部分的某些问题。

## 求值

为了解析求值前面生成的语法树，我们需要写一个递归函数。但是在开始之前，我们先观察一下树的结构。使用上一章的程序打印一些表达式的解析结果，你能观察到什么？

```
lispy> * 10 (+ 1 51)
>
  regex
  operator|char:1:1 '*'
  expr|number|regex:1:3 '10'
  expr|>
    char:1:6 '('
    operator|char:1:7 '+'
    expr|number|regex:1:9 '1'
    expr|number|regex:1:11 '51'
    char:1:13 ')'
  regex
```

首先我们注意到，有 `number` 标签的节点一定是一个数字，并且没有孩子节点。我们可以直接将其转换为一个数字。这将是递归函数中的基本情况。

如果一个节点有 `expr` 标签，但没有 `number` 标签，我们需要看他的第二个孩子节点是什么操作符(第一个孩子节点永远是 `(` 字符)。然后我们需要使用这个操作符来对后面的孩子节点进行求值。当然，除了最后的 `)` 节点。这就是所谓的递归的情况啦。

在对语法树进行求值的时候，正如前面编写的 `number_of_nodes` 函数，需要保存计算的结果。在这里，我们使用 C 语言中 `long` 类型(长整形)。

另外，为了检测节点的类型，或是获得节点中保存的数值，我们会用到节点中的 `tag` 和 `contents` 字段。这些字段都是字符串类型的，所以需要用到一些辅助性的库函数：

| 函数名 | 作用 |
|-----------------|------------------------|
| `atoi` | 将 `char*` 转化为 `long` 型 |
| `strcmp` | 接受两个 `char*` 参数，比较他们是否相等，如果相等就返回 0 |
| `strstr` | 接受两个 `char*`，如果第一个字符串包含第二个，返回其在第一个中首次出现的位置的指针，否则返回 0 |

我们可以使用 `strcmp` 来检查应该使用什么操作符，并使用 `strstr` 来检测 `tag` 中是否含有某个字段。有了这些基础，我们的递归求值函数就可以写出来啦：

```c
long eval(mpc_ast_t* t) {
  
  /* If tagged as number return it directly. */ 
  if (strstr(t->tag, "number")) {
    return atoi(t->contents);
  }
  
  /* The operator is always second child. */
  char* op = t->children[1]->contents;
  
  /* We store the third child in `x` */
  long x = eval(t->children[2]);
  
  /* Iterate the remaining children and combining. */
  int i = 3;
  while (strstr(t->children[i]->tag, "expr")) {
    x = eval_op(x, op, eval(t->children[i]));
    i++;
  }
  
  return x;  
}
```

其中，`eval_op` 函数的定义如下。它接受一个数字，一个操作符，和另一个数字。它检测操作符的类型，对其进行相应的计算，并将结果返回。

```c
/* Use operator string to see which operation to perform */
long eval_op(long x, char* op, long y) {
  if (strcmp(op, "+") == 0) { return x + y; }
  if (strcmp(op, "-") == 0) { return x - y; }
  if (strcmp(op, "*") == 0) { return x * y; }
  if (strcmp(op, "/") == 0) { return x / y; }
  return 0;
}
```

##打印

有了求值函数，就不能满足于打印语法树了，现在我们可以打印语法树求值后的结果啦。

```c
long result = eval(r.output);
printf("%li\n", result);
mpc_ast_delete(r.output);
```

所有的工作完成无误后，就能看到我们的新语言执行一些基本的数学运算啦！

```
Lispy Version 0.0.0.0.3
Press Ctrl+c to Exit

lispy> + 5 6
11
lispy> - (* 10 10) (+ 1 1 1)
97
```

## 参考

`evaluation.c`

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

/* Use operator string to see which operation to perform */
long eval_op(long x, char* op, long y) {
  if (strcmp(op, "+") == 0) { return x + y; }
  if (strcmp(op, "-") == 0) { return x - y; }
  if (strcmp(op, "*") == 0) { return x * y; }
  if (strcmp(op, "/") == 0) { return x / y; }
  return 0;
}

long eval(mpc_ast_t* t) {
  
  /* If tagged as number return it directly. */ 
  if (strstr(t->tag, "number")) {
    return atoi(t->contents);
  }
  
  /* The operator is always second child. */
  char* op = t->children[1]->contents;
  
  /* We store the third child in `x` */
  long x = eval(t->children[2]);
  
  /* Iterate the remaining children and combining. */
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
  
  puts("Lispy Version 0.0.0.0.3");
  puts("Press Ctrl+c to Exit\n");
  
  while (1) {
  
    char* input = readline("lispy> ");
    add_history(input);
    
    mpc_result_t r;
    if (mpc_parse("<stdin>", input, Lispy, &r)) {
      
      long result = eval(r.output);
      printf("%li\n", result);
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

