# 第零九章 • S-表达式

## Lisp 列表

Lisp 程序代码与数据的形式完全相同，这使得它非常强大，能完成许多其他语言不能完成的事情。为了拥有这个强大的特性，我们需要将求值过程分为读取并存储输入、对输入进行求值两个过程。

本章结束后，程序的运行结果会和上一章的有轻微的不同。这是因为我们会花时间去更改程序内部的工作方式。在软件开发中，这被叫做**重构**。重构可能对于当前的程序运行结果并没有太大的影响，但因为工作方式的优化，会使我们在后面的开发中更加省心。

为了存储输入，我们需要创建一个内部列表结构，能够递归地表示数字、操作符号以及其他的列表。在 Lisp 中，这个结构被称为 S-表达式(Symbolic Expression)。我们将扩展 `lval` 结构来表示它。S-表达式求值也是典型的 Lisp 式过程：首先取列表第一个元素为操作符，然后遍历所有剩下的元素，将它们作为操作数。

有了 S-表达式，我们才算真正迈进了 Lisp 的大门。

## 指针

在 C 语言中，要表示列表，就必须正确的使用指针。C 语言中的指针一直如洪水猛兽般存在。虽然概念上非常简单，但是用起来却变幻多端，神秘莫测，这使得指针看上去比实际要可怕得多。幸运的是，在本书中我们只会用一些指针在 C 语言中最常规的用法。

我们之所以需要指针，主要是由 C 语言中函数的工作方式决定的。C 语言函数的参数**全部**是通过值传递的。也就是说，传递给函数的实际是实参的拷贝。对于 `int`、`long`、`char`等系统类型以及用户自定义的结构体都是成立的。这种方式适用于绝大多数情况，但也会偶尔出现问题。

一种常见的情况是，如果我们有一个巨大结构体需要作为参数传递，则每次调用函数，就会对实参进行一次拷贝，这无疑是对性能和内存的浪费。

另外一个问题是，结构体的大小终究是有限的，无论多大，也只能是个固定的大小。而如果我们想向函数传递一组数据，而且数据的总数还是不固定的，结构体就明显的无能为力了。

为了解决这个问题，C 语言的开发者们想出了一个聪明的办法。他们把内存想象成一个巨大的字节数组，每个字节都可以拥有一个全局的索引值。这有点像门牌号：第一个字节索引为 0，第二个字节索引为 1，等等。

在这种情况下，计算机中的所有数据，包括当前运行的程序中的结构体、变量都有相应的索引值与其对应(数据的开始字节的索引作为整个数据的索引)。所以，除了将数据本身拷贝到函数参数，我们还可以只拷贝数据的索引值。在函数内部则可以根据索引值找到需要的数据本身(译者注：我们将这个索引值称为*地址*，存储地址的变量称为*指针*)。使用指针，函数可以修改指定位置的内存而无需拷贝。除此之外，指针还可以做其他很多事情。

因为计算机内存的大小是固定的，表示一个地址所需要的字节数也是固定的。但是地址指向的内存的字节数是可以变化的。这就意味着，我们可以创建一个大小可变的数据结构，并将其指针传入函数，对其进行读取及修改。

所以，所谓的指针也仅仅是一个数字而已。是内存中的一块数据的开始字节的索引值。指针的类型用来提示程序员和编译器指针指向的是一块什么样的数据，占多少个字节等。

指针类型是在现有类型的后面加一个星号组成，我们之前已经见过一些指针的示例了，如：`mpc_parser_t*`、`mpc_ast_t*` 以及 `char*`。

要创建指针，我们就需要获取数据的地址。C 语言提供了取地址符(`&`)来获取某个数据的地址。在前面的章节中，我们也曾传给过 `mpc_parse` 函数一个指针，以便其能将输出放到我们声明的 `mpc_result_t` 变量中。

最后，为了获取指针所指向的地址的数据值(称为*解引用*)，我们需要在指针左边使用 `*` 操作符。要获取结构体指针的某个字段，需要使用 `->` 操作符，而不是 `.`，这你在第七章已经见过了。

## 栈(Stack)和堆(Heap)

前面说过，我们可以把内存简单粗暴地想象成一个巨大的字节数组。事实上，它被更加合理地划分成了两部分，即*栈*和*堆*。

有些人可能已经听说过一些关于堆和栈的神秘传说，例如“栈从上往下增长，而堆则是从下往上”，或是“栈的数量很多，但堆只有一个”云云。其实这些事情都是无关紧要的。在 C 语言中，处理好栈和堆确实是件麻烦的事情，但这并不代表它们很神秘。实际上，它们只是内存中的两块不同的区域，分别用来完成不同的任务而已。

### 栈(The Stack)

栈是程序赖以生存的地方，所有的临时变量和数据结构都保存于其中，供你读取及编辑。每次调用一个新的函数，就会有一块新的栈区压入，并在其中存放函数内的临时变量、传入的实参的拷贝以及其它的一些信息。当函数运行完毕，这块栈区就会被弹出并回收，供其他函数使用。

我喜欢把栈想象成一个建筑工地。每次需要干点新事情的时候，我们就圈出一块地方来，放工具、原料，并在这里工作。如果需要的话，我们也可以到工地的其他地方，甚至是离开工地。但是我们所有的工作都是在自己的地方完成的。一旦工作完成，我们就把工作成果转移到新的地方，并把现在工作的地方清理干净。

### 堆(The Heap)

堆占据另一部分内存，主要用来存放长生命周期期的数据。堆中的数据必须手动申请和释放。申请内存使用 `malloc` 函数。这个函数接受一个数字作为要申请的字节数，返回申请好的内存块的指针。

当使用完毕申请的内存，我们还需要将其释放，只要将 `malloc`  函数返回的指针传给 `free` 函数即可。

堆比栈的使用难度要大一些，因为它要求程序员手动调用 `free` 函数释放内存，而且还要正确调用。如果不释放，程序就有可能不断申请新的内存，而不释放旧的，导致内存越用越多。这也被称为*内存泄漏*。避免这种情况发生的一个简单有效的办法就是，针对每一个 `malloc` 函数调用，都有且只有一个 `free` 函数与之对应。这某种程度上就能保证程序能正确处理堆内存的使用。

我把堆想象成一个自助存储仓库，我们使用 `malloc` 函数申请存储空间。我们可以在自主存储仓库和建筑工地之间自由存取。它非常适合用来存放大件的偶尔才用一次的物件。唯一的问题就是在用完之后要记得使用 `free` 函数将空间归还。

## 解析表达式

因为现在我们考虑的是 S-表达式，而不是之前的波兰表达式了，我们需要更新一下语法分析器。S-表达式的语法非常简单。只是小括号之间包含一组表达式而已。而这些表达式可以是数字、操作符或是其他的 S-表达式。只需修改一下之前写的就可以了。另外，我们还需把 `operator` 规则重命名为 `symbol`。为之后添加更多的操作符以及变量、函数等做准备。

```c
mpc_parser_t* Number = mpc_new("number");
mpc_parser_t* Symbol = mpc_new("symbol");
mpc_parser_t* Sexpr  = mpc_new("sexpr");
mpc_parser_t* Expr   = mpc_new("expr");
mpc_parser_t* Lispy  = mpc_new("lispy");

mpca_lang(MPCA_LANG_DEFAULT,
  "                                          \
    number : /-?[0-9]+/ ;                    \
    symbol : '+' | '-' | '*' | '/' ;         \
    sexpr  : '(' <expr>* ')' ;               \
    expr   : <number> | <symbol> | <sexpr> ; \
    lispy  : /^/ <expr>* /$/ ;               \
  ",
  Number, Symbol, Sexpr, Expr, Lispy);
```

同时，还要记得在退出之前做好清理工作：

```c
mpc_cleanup(5, Number, Symbol, Sexpr, Expr, Lispy);
```

## 表达式结构

首先，我们需要让 `lval` 能够存储 S-表达式。这意味着我们还要能存储符号(Symbols)和数字。我们向枚举中添加两个新的类型。`LVAL_SYM` 表示操作符类型，例如 `+` 等，`LVAL_SEXPR` 表示 S-表达式。

```c
enum { LVAL_ERR, LVAL_NUM, LVAL_SYM, LVAL_SEXPR };
```

S-表达式是一个可变长度的列表。在本章的开头已经提到，我们不能创建可变长度的结构体，所以只能使用指针来表示它。我们为  `lval` 结构体创建一个 `cell` 字段，指向一个存放 `lval*` 列表的区域。所以 `cell` 的类型就应该是 `lval**`。指向 `lval*` 的指针。我们还需要知道 `cell` 列表中的元素个数，所以我创建了 `count` 字段。

我们使用字符串来表示符号(Symbols)，另外我们还增加了另一个字符串用来存储错误信息。也就是说现在 `lval` 可以存储更加具体的错误信息了，而不只是一个错误代码，这使得我们的错误报告系统更加灵活好用。我们也可以删除掉之前写的错误枚举了。升级过后的 `lval` 结构体如下所示：

```c
typedef struct lval {
  int type;
  long num;
  /* Error and Symbol types have some string data */
  char* err;
  char* sym;
  /* Count and Pointer to a list of "lval*" */
  int count;
  struct lval** cell;
} lval;
```

> #### 有指向指向指针的指针的指针吗？

> 这里有个古老的笑话，说是可以根据 C 程序员的程序中指针后面的星星数(`*`)作为其水平的评分。:P

> 初级水平的人写的程序可能只会用到像 `char*` 或是奇怪的 `int*` 等一级指针，所以他们被称为一星程序员。而大多数中级的程序员则会用到诸如 `lval**` 这类的二级指针，所以他们被称为二星程序员。但据说能用三级指针的就真的很少见了，你可能会在一些伟大的作品中见到，这些代码的妙处凡夫俗子自然也是体会不到的。果真如此，三星程序员这个称号真是极大的赞誉了。
> 
> 但据我所知，还没有人用到过四级指针。

## 构造函数和析构函数

我们可以重写 `lval` 的构造函数，使其返回 `lval` 的指针，而不是其本身。这样做会使得对 `lval` 变量进行跟踪更加简单。为此，我们需要用到 `malloc` 库函数以及 `sizeof` 操作符为 `lval` 结构体在堆上申请足够大的内存区域，然后使用 `->` 操作符填充结构体中的相关字段。

当我们构造一个 `lval` 时，它的某些指针字段可能会包含其他的在堆上申请的内存，所以我们应该小心行事。当某个 `lval` 完成使命之后，我们不仅需要删除它本身所指向的堆内存，还要删除它的字段所指向的堆内存。

```c
/* Construct a pointer to a new Number lval */ 
lval* lval_num(long x) {
  lval* v = malloc(sizeof(lval));
  v->type = LVAL_NUM;
  v->num = x;
  return v;
}
```

```c
/* Construct a pointer to a new Error lval */ 
lval* lval_err(char* m) {
  lval* v = malloc(sizeof(lval));
  v->type = LVAL_ERR;
  v->err = malloc(strlen(m) + 1);
  strcpy(v->err, m);
  return v;
}
```

```c
/* Construct a pointer to a new Symbol lval */ 
lval* lval_sym(char* s) {
  lval* v = malloc(sizeof(lval));
  v->type = LVAL_SYM;
  v->sym = malloc(strlen(s) + 1);
  strcpy(v->sym, s);
  return v;
}
```

```c
/* A pointer to a new empty Sexpr lval */
lval* lval_sexpr(void) {
  lval* v = malloc(sizeof(lval));
  v->type = LVAL_SEXPR;
  v->count = 0;
  v->cell = NULL;
  return v;
}
```

> #### `NULL` 是什么？

> `NULL` 是一个指向内存地址 0 的特殊常量。按照惯例，它通常被用来表示空值或无数据。在上面的代码中，我们使用 `NULL` 来表示虽然我们有一个数据指针，但它目前还没有指向任何内容。在本书的后续章节中你讲经常性地遇到这个特殊的常量，所以，请眼熟它。

---

> #### 为什么要使用 `strlen(s) + 1`？

> 在 C 语言中，字符串是以空字符做为终止标记的。所以，C 语言字符串的最后一个字符一定是 `\0`。请确保所有的字符串都是按照这个约定来存储的，不然程序就会因为莫名其妙的错误退出。`strlen` 函数返回的是字符串的实际长度(所以不包括结尾的 `\0` 终止符)。所以为了保证有足够的空间存储所有字符，我们需要在额外 +1。

现在，我们需要一个定制的函数来删除 `lval*`。这个函数应该调用 `free` 函数来释放本身所指向的由 `malloc` 函数所申请的内存。但更重要的是，它应该根据自身的类型，释放所有它的字段指向的内存。所以我们只需按照上方介绍的那些构造函数对应释放相应的字段，就不会有内存的泄露了。

```c
void lval_del(lval* v) {

  switch (v->type) {
    /* Do nothing special for number type */
    case LVAL_NUM: break;

    /* For Err or Sym free the string data */
    case LVAL_ERR: free(v->err); break;
    case LVAL_SYM: free(v->sym); break;

    /* If Sexpr then delete all elements inside */
    case LVAL_SEXPR:
      for (int i = 0; i < v->count; i++) {
        lval_del(v->cell[i]);
      }
      /* Also free the memory allocated to contain the pointers */
      free(v->cell);
    break;
  }

  /* Free the memory allocated for the "lval" struct itself */
  free(v);
}
```

## 读取表达式

首先我们会读取整个程序，并构造一个 `lval*` 来表示它，然后我们对这个 `lval*` 进行遍历求值来得到程序的运行结果。第一阶段负责把抽象语法树(abstract syntax tree)转换为一个 S-表达式，第二阶段则根据我们已由的 Lisp 规则对 S-表达式进行遍历求值。

为了完成第一步，我们可以递归的查看语法分析树中的每个节点，并根据节点的 `tag` 和 `contents` 字段构造出不同类型的 `lval*`。

如果给定节点的被标记为 `number` 或 `symbol`，则我们可以调用对应的构造函数直接返回一个 `lval*`。如果给定的节点被标记为 `root` 或 `sexpr`，则我们应该构造一个空的 S-表达式类型的 `lval*`，并逐一将它的子节点加入。

为了更加方便的像一个 S-表达式中添加元素，我们可以创建一个函数 `lval_add`，这个函数将表达式的子表达式计数加一，然后使用 `realloc` 函数为 `v->cell` 字段重新扩大申请内存，用于存储刚刚加入的子表达式 `lval* x`。

<!--
Don't Lisps use Cons cells?
Other Lisps have a slightly different definition of what an S-Expression is. In most other Lisps S-Expressions are defined inductively as either an atom such as a symbol of number, or two other S-Expressions joined, or cons, together.
This naturally leads to an implementation using linked lists, a different data structure to the one we are using. I choose to represent S-Expressions as a variable sized array in this book for the purposes of simplicity, but it is important to be aware that the official definition, and typical implementation are both subtly different.
-->

```c
lval* lval_read_num(mpc_ast_t* t) {
  errno = 0;
  long x = strtol(t->contents, NULL, 10);
  return errno != ERANGE ?
    lval_num(x) : lval_err("invalid number");
}
```

```c
lval* lval_read(mpc_ast_t* t) {

  /* If Symbol or Number return conversion to that type */
  if (strstr(t->tag, "number")) { return lval_read_num(t); }
  if (strstr(t->tag, "symbol")) { return lval_sym(t->contents); }

  /* If root (>) or sexpr then create empty list */
  lval* x = NULL;
  if (strcmp(t->tag, ">") == 0) { x = lval_sexpr(); } 
  if (strstr(t->tag, "sexpr"))  { x = lval_sexpr(); }

  /* Fill this list with any valid expression contained within */
  for (int i = 0; i < t->children_num; i++) {
    if (strcmp(t->children[i]->contents, "(") == 0) { continue; }
    if (strcmp(t->children[i]->contents, ")") == 0) { continue; }
    if (strcmp(t->children[i]->contents, "}") == 0) { continue; }
    if (strcmp(t->children[i]->contents, "{") == 0) { continue; }
    if (strcmp(t->children[i]->tag,  "regex") == 0) { continue; }
    x = lval_add(x, lval_read(t->children[i]));
  }

  return x;
}
```

```c
lval* lval_add(lval* v, lval* x) {
  v->count++;
  v->cell = realloc(v->cell, sizeof(lval*) * v->count);
  v->cell[v->count-1] = x;
  return v;
}
```

## 打印表达式

距离验证本章做的所有改变只有一步之遥了！我们还需要修改打印函数使其能够打印出 S-表达式。通过将 S-表达式打印出来和输入对比可以进一步检查读入模块是否正确工作。

为了打印 S-表达式，我们创建一个新函数，遍历所有的子表达式，并将它们单独打印出来，以空格隔开，正如输入的时候一样。

```c
void lval_expr_print(lval* v, char open, char close) {
  putchar(open);
  for (int i = 0; i < v->count; i++) {

    /* Print Value contained within */
    lval_print(v->cell[i]);

    /* Don't print trailing space if last element */
    if (i != (v->count-1)) {
      putchar(' ');
    }
  }
  putchar(close);
}
```

```c
void lval_print(lval* v) {
  switch (v->type) {
    case LVAL_NUM:   printf("%li", v->num); break;
    case LVAL_ERR:   printf("Error: %s", v->err); break;
    case LVAL_SYM:   printf("%s", v->sym); break;
    case LVAL_SEXPR: lval_expr_print(v, '(', ')'); break;
  }
}

void lval_println(lval* v) { lval_print(v); putchar('\n'); }
```

> #### 我没法声明这些函数，因为它们互相调用了对方。

> `lval_expr_print` 函数内部调用了 `lval_print` 函数，`lval_print` 内部又调用了 `lval_expr_print`。似乎是没有办法解决依赖性的。C 语言提供了*前置声明*来解决这个问题。前置声明只定义了函数的形式，而没有函数体(译者注：前置声明就是告诉编译器：“我保证有这个函数，你放心调用就是了”)。它允许其他函数调用它，而具体的函数定义则在后面。函数声明只要将函数定义的函数体换成 `;` 即可。在我们的程序中，应该将 `void lval_print(lval* v);` 语句放在一个比 `lval_expr_print`  函数靠前的地方。在以后的编程过程中，你一定会再次遇到这个问题，所以请记住这个解决方案！

在主循环中，我们可以将求值部分移除了，替换为新写就的读取和打印函数。

```c
lval* x = lval_read(r.output);
lval_println(x);
lval_del(x);
```

正常情况下，你应该可以看到下面这样的结果。

```c
lispy> + 2 2
(+ 2 2)
lispy> + 2 (* 7 6) (* 2 5)
(+ 2 (* 7 6) (* 2 5))
lispy> *     55     101  (+ 0 0 0)
(* 55 101 (+ 0 0 0))
lispy>
```

## 表达式求值

求值函数的具体行为和之前的并无太大变化。我们需要将其适配本章定义的 `lval*` 以及更加灵活的表达式定义。其实可以把求值函数想象成某种转换器－－它读取 `lval*` 作为输入，通过某种方式将其转化为新的 `lval*` 并输出。在有些时候，求值函数不对输入做任何修改，原封不动的将其返回；有些时候，它会对输入的做一些改动；而在大多数情况下，它会将输入的 `lval*` 删除，返回完全不同的东西。如果要返回新的东西，一定要记得将原有的作为输入的 `lval*` 删除。

对于 S-表达式，我们首先遍历它所有的子节点，如果子节点有任何错误，我们就使用稍后定义的函数 `lval_take` 将遇到的第一个错误返回。

对于没有子节点的 S-表达式直接将其返回就可以了，这是为了处理空表达式 `{}` 的情况。另外，我们还需要检查只有一个子节点的表达式，例如 `{5}`，这种情况我们应该将其包含的表达式返回。

如果以上情况都不成立，那我们就知道这是一个合法的表达式，有个多于一个的子节点。对于此种情况，我们使用稍后定义的函数 `lval_pop` 将第一个元素从表达式中分离开来，然后检查确保它是一个 `symbol`。然后根据它的具体类型，将它和参数一起传入 `builtin_op` 函数计算求值。如果它不是 `symbol`，我们就将它以及传进来的其它参数删除，然后返回一个错误。

对于其它的非 S-表达式类型，我们就直接将其返回。

```c
lval* lval_eval_sexpr(lval* v) {

  /* Evaluate Children */
  for (int i = 0; i < v->count; i++) {
    v->cell[i] = lval_eval(v->cell[i]);
  }

  /* Error Checking */
  for (int i = 0; i < v->count; i++) {
    if (v->cell[i]->type == LVAL_ERR) { return lval_take(v, i); }
  }

  /* Empty Expression */
  if (v->count == 0) { return v; }

  /* Single Expression */
  if (v->count == 1) { return lval_take(v, 0); }

  /* Ensure First Element is Symbol */
  lval* f = lval_pop(v, 0);
  if (f->type != LVAL_SYM) {
    lval_del(f); lval_del(v);
    return lval_err("S-expression Does not start with symbol!");
  }

  /* Call builtin with operator */
  lval* result = builtin_op(v, f->sym);
  lval_del(f);
  return result;
}
```

```c
lval* lval_eval(lval* v) {
  /* Evaluate Sexpressions */
  if (v->type == LVAL_SEXPR) { return lval_eval_sexpr(v); }
  /* All other lval types remain the same */
  return v;
}
```

上面的代码中用到了两个我们还没有定义的函数：`lval_pop` 和 `lval_take`。这两个都是用于操作 `lval` 类型的通用型函数，我们在本章和之后的章节中都会用到。

`lval_pop` 函数将所操作的 S-表达式的第 `i` 个元素取出，并将在其后面的元素向前移动填补空缺，使得这个 S-表达式不再包含这个元素。然后将取出的元素返回。需要注意的是，这个函数并不会将这个 S- 表达式删除。它只是从中取出某个元素，剩下的元素都保持原样。这意味着这两部分最终都需要在某个地方使用 `lval_del` 函数删除。

`lval_take` 和 `lval_pop` 函数类似，不过它将取出元素之后剩下的列表删除了。它利用了 `lval_pop` 函数并做了一点小小的改变，却使得我们的代码可读性更高了一些。所以，不同于 `lval_pop`，你只需负责使用 `lval_del` 删除取出的元素即可。

```c
lval* lval_pop(lval* v, int i) {
  /* Find the item at "i" */
  lval* x = v->cell[i];

  /* Shift memory after the item at "i" over the top */
  memmove(&v->cell[i], &v->cell[i+1],
    sizeof(lval*) * (v->count-i-1));

  /* Decrease the count of items in the list */
  v->count--;

  /* Reallocate the memory used */
  v->cell = realloc(v->cell, sizeof(lval*) * v->count);
  return x;
}
```

```c
lval* lval_take(lval* v, int i) {
  lval* x = lval_pop(v, i);
  lval_del(v);
  return x;
}
```

We also need to define the evaluation function builtin_op. This is like the eval_op function used in our previous chapter but modified to take a single lval* representing a list of all the arguments to operate on. It needs to do some more rigorous error checking. If any of the inputs are a non-number lval* we need to return an error.

First it checks that all the arguments input are numbers. It then pops the first argument to start. If there are no more sub-expressions and the operator is subtraction it performs unary negation on this first number. This makes expressions such as (- 5) evaluate correctly.

If there are more arguments it constantly pops the next one from the list and performs arithmetic depending on which operator we're meant to be using. If a zero is encountered on division it deletes the temporary x, y, and the argument list a, and returns an error.

If there have been no errors the input arguments are deleted and the new expression returned.

```c
lval* builtin_op(lval* a, char* op) {
  
  /* Ensure all arguments are numbers */
  for (int i = 0; i < a->count; i++) {
    if (a->cell[i]->type != LVAL_NUM) {
      lval_del(a);
      return lval_err("Cannot operate on non-number!");
    }
  }
  
  /* Pop the first element */
  lval* x = lval_pop(a, 0);

  /* If no arguments and sub then perform unary negation */
  if ((strcmp(op, "-") == 0) && a->count == 0) {
    x->num = -x->num;
  }

  /* While there are still elements remaining */
  while (a->count > 0) {

    /* Pop the next element */
    lval* y = lval_pop(a, 0);

    if (strcmp(op, "+") == 0) { x->num += y->num; }
    if (strcmp(op, "-") == 0) { x->num -= y->num; }
    if (strcmp(op, "*") == 0) { x->num *= y->num; }
    if (strcmp(op, "/") == 0) {
      if (y->num == 0) {
        lval_del(x); lval_del(y);
        x = lval_err("Division By Zero!"); break;
      }
      x->num /= y->num;
    }

    lval_del(y);
  }

  lval_del(a); return x;
}
```

This completes our evaluation functions. We just need to change main again so it passes the input through this evaluation before printing it.

```c
lval* x = lval_eval(lval_read(r.output));
lval_println(x);
lval_del(x);
```

Now you should now be able to evaluate expressions correctly in the same way as in the previous chapter. Take a little breather and have a play around with the new evaluation. Make sure everything is working correctly, and the behaviour is as expected. In the next chapter we're going to make great use of these changes to implement some cool new features.

```c
lispy> + 1 (* 7 5) 3
39
lispy> (- 100)
-100
lispy>
()
lispy> /
/
lispy> (/ ())
Error: Cannot operate on non-number!
lispy>
```

## 参考

**s_expressions.c**

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

/* Add SYM and SEXPR as possible lval types */
enum { LVAL_ERR, LVAL_NUM, LVAL_SYM, LVAL_SEXPR };

typedef struct lval {
  int type;
  long num;
  /* Error and Symbol types have some string data */
  char* err;
  char* sym;
  /* Count and Pointer to a list of "lval*"; */
  int count;
  struct lval** cell;
} lval;

/* Construct a pointer to a new Number lval */ 
lval* lval_num(long x) {
  lval* v = malloc(sizeof(lval));
  v->type = LVAL_NUM;
  v->num = x;
  return v;
}

/* Construct a pointer to a new Error lval */ 
lval* lval_err(char* m) {
  lval* v = malloc(sizeof(lval));
  v->type = LVAL_ERR;
  v->err = malloc(strlen(m) + 1);
  strcpy(v->err, m);
  return v;
}

/* Construct a pointer to a new Symbol lval */ 
lval* lval_sym(char* s) {
  lval* v = malloc(sizeof(lval));
  v->type = LVAL_SYM;
  v->sym = malloc(strlen(s) + 1);
  strcpy(v->sym, s);
  return v;
}

/* A pointer to a new empty Sexpr lval */
lval* lval_sexpr(void) {
  lval* v = malloc(sizeof(lval));
  v->type = LVAL_SEXPR;
  v->count = 0;
  v->cell = NULL;
  return v;
}

void lval_del(lval* v) {

  switch (v->type) {
    /* Do nothing special for number type */
    case LVAL_NUM: break;
    
    /* For Err or Sym free the string data */
    case LVAL_ERR: free(v->err); break;
    case LVAL_SYM: free(v->sym); break;
    
    /* If Sexpr then delete all elements inside */
    case LVAL_SEXPR:
      for (int i = 0; i < v->count; i++) {
        lval_del(v->cell[i]);
      }
      /* Also free the memory allocated to contain the pointers */
      free(v->cell);
    break;
  }
  
  /* Free the memory allocated for the "lval" struct itself */
  free(v);
}

lval* lval_add(lval* v, lval* x) {
  v->count++;
  v->cell = realloc(v->cell, sizeof(lval*) * v->count);
  v->cell[v->count-1] = x;
  return v;
}

lval* lval_pop(lval* v, int i) {
  /* Find the item at "i" */
  lval* x = v->cell[i];
  
  /* Shift memory after the item at "i" over the top */
  memmove(&v->cell[i], &v->cell[i+1],
    sizeof(lval*) * (v->count-i-1));
  
  /* Decrease the count of items in the list */
  v->count--;
  
  /* Reallocate the memory used */
  v->cell = realloc(v->cell, sizeof(lval*) * v->count);
  return x;
}

lval* lval_take(lval* v, int i) {
  lval* x = lval_pop(v, i);
  lval_del(v);
  return x;
}

void lval_print(lval* v);

void lval_expr_print(lval* v, char open, char close) {
  putchar(open);
  for (int i = 0; i < v->count; i++) {
    
    /* Print Value contained within */
    lval_print(v->cell[i]);
    
    /* Don't print trailing space if last element */
    if (i != (v->count-1)) {
      putchar(' ');
    }
  }
  putchar(close);
}

void lval_print(lval* v) {
  switch (v->type) {
    case LVAL_NUM:   printf("%li", v->num); break;
    case LVAL_ERR:   printf("Error: %s", v->err); break;
    case LVAL_SYM:   printf("%s", v->sym); break;
    case LVAL_SEXPR: lval_expr_print(v, '(', ')'); break;
  }
}

void lval_println(lval* v) { lval_print(v); putchar('\n'); }

lval* builtin_op(lval* a, char* op) {
  
  /* Ensure all arguments are numbers */
  for (int i = 0; i < a->count; i++) {
    if (a->cell[i]->type != LVAL_NUM) {
      lval_del(a);
      return lval_err("Cannot operate on non-number!");
    }
  }
  
  /* Pop the first element */
  lval* x = lval_pop(a, 0);
  
  /* If no arguments and sub then perform unary negation */
  if ((strcmp(op, "-") == 0) && a->count == 0) {
    x->num = -x->num;
  }
  
  /* While there are still elements remaining */
  while (a->count > 0) {
  
    /* Pop the next element */
    lval* y = lval_pop(a, 0);
    
    /* Perform operation */
    if (strcmp(op, "+") == 0) { x->num += y->num; }
    if (strcmp(op, "-") == 0) { x->num -= y->num; }
    if (strcmp(op, "*") == 0) { x->num *= y->num; }
    if (strcmp(op, "/") == 0) {
      if (y->num == 0) {
        lval_del(x); lval_del(y);
        x = lval_err("Division By Zero.");
        break;
      }
      x->num /= y->num;
    }
    
    /* Delete element now finished with */
    lval_del(y);
  }
  
  /* Delete input expression and return result */
  lval_del(a);
  return x;
}

lval* lval_eval(lval* v);

lval* lval_eval_sexpr(lval* v) {
  
  /* Evaluate Children */
  for (int i = 0; i < v->count; i++) {
    v->cell[i] = lval_eval(v->cell[i]);
  }
  
  /* Error Checking */
  for (int i = 0; i < v->count; i++) {
    if (v->cell[i]->type == LVAL_ERR) { return lval_take(v, i); }
  }
  
  /* Empty Expression */
  if (v->count == 0) { return v; }
  
  /* Single Expression */
  if (v->count == 1) { return lval_take(v, 0); }
  
  /* Ensure First Element is Symbol */
  lval* f = lval_pop(v, 0);
  if (f->type != LVAL_SYM) {
    lval_del(f); lval_del(v);
    return lval_err("S-expression Does not start with symbol.");
  }
  
  /* Call builtin with operator */
  lval* result = builtin_op(v, f->sym);
  lval_del(f);
  return result;
}

lval* lval_eval(lval* v) {
  /* Evaluate Sexpressions */
  if (v->type == LVAL_SEXPR) { return lval_eval_sexpr(v); }
  /* All other lval types remain the same */
  return v;
}

lval* lval_read_num(mpc_ast_t* t) {
  errno = 0;
  long x = strtol(t->contents, NULL, 10);
  return errno != ERANGE ?
    lval_num(x) : lval_err("invalid number");
}

lval* lval_read(mpc_ast_t* t) {
  
  /* If Symbol or Number return conversion to that type */
  if (strstr(t->tag, "number")) { return lval_read_num(t); }
  if (strstr(t->tag, "symbol")) { return lval_sym(t->contents); }
  
  /* If root (>) or sexpr then create empty list */
  lval* x = NULL;
  if (strcmp(t->tag, ">") == 0) { x = lval_sexpr(); } 
  if (strstr(t->tag, "sexpr"))  { x = lval_sexpr(); }
  
  /* Fill this list with any valid expression contained within */
  for (int i = 0; i < t->children_num; i++) {
    if (strcmp(t->children[i]->contents, "(") == 0) { continue; }
    if (strcmp(t->children[i]->contents, ")") == 0) { continue; }
    if (strcmp(t->children[i]->contents, "}") == 0) { continue; }
    if (strcmp(t->children[i]->contents, "{") == 0) { continue; }
    if (strcmp(t->children[i]->tag,  "regex") == 0) { continue; }
    x = lval_add(x, lval_read(t->children[i]));
  }
  
  return x;
}

int main(int argc, char** argv) {
  
  mpc_parser_t* Number = mpc_new("number");
  mpc_parser_t* Symbol = mpc_new("symbol");
  mpc_parser_t* Sexpr  = mpc_new("sexpr");
  mpc_parser_t* Expr   = mpc_new("expr");
  mpc_parser_t* Lispy  = mpc_new("lispy");
  
  mpca_lang(MPCA_LANG_DEFAULT,
    "                                          \
      number : /-?[0-9]+/ ;                    \
      symbol : '+' | '-' | '*' | '/' ;         \
      sexpr  : '(' <expr>* ')' ;               \
      expr   : <number> | <symbol> | <sexpr> ; \
      lispy  : /^/ <expr>* /$/ ;               \
    ",
    Number, Symbol, Sexpr, Expr, Lispy);
  
  puts("Lispy Version 0.0.0.0.5");
  puts("Press Ctrl+c to Exit\n");
  
  while (1) {
  
    char* input = readline("lispy> ");
    add_history(input);
    
    mpc_result_t r;
    if (mpc_parse("<stdin>", input, Lispy, &r)) {
      lval* x = lval_eval(lval_read(r.output));
      lval_println(x);
      lval_del(x);
      mpc_ast_delete(r.output);
    } else {    
      mpc_err_print(r.error);
      mpc_err_delete(r.error);
    }
    
    free(input);
    
  }
  
  mpc_cleanup(5, Number, Symbol, Sexpr, Expr, Lispy);
  
  return 0;
}
```

## 福利

- 举例说明在我们的程序中哪个变量是在**栈**上的。
- 举例说明在我们的程序中哪个变量是在**堆**上的。
- `strcpy` 函数的作用是什么？
- `realloc` 函数的作用是什么？
- `memmove` 函数的作用是什么？
- `memmove` 于 `memcpy` 有何不同？
- 扩展解析和求值部分使其支持求余操作符 `%`。
- 扩展解析和求值部分使其支持浮点数(`double`)。