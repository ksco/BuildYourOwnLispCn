# 第零十章 • Q-表达式

## 添加特性

你可能会注意到包括本章在内的之后章节都遵循一个模式，这个模式也是给一个编程语言添加新特性的典型方式。它包含一系列的步骤来从无到有的实现某个特性。下表详细地说明了本章所要引入的 Q-表达式的具体实现步骤。

|   名称   | 描述                      |
| ------- | ------------------------- |
| **语法** | 为新特性添加新的语法规则 |
| **表示** | 为新特性添加新的数据类型 |
| **解析** | 为新特性添加新的函数，正确处理 AST |
| **语义** | 为新特性添加新的函数，用于求值和操作 |

## Q-表达式

本章我们将实现一个新的 Lisp 值类型，叫做 Q-表达式。

它的英文全称为 *quoted expression*，跟 S-表达式一样，也是 Lisp 表达式的一种，但它不受标准 Lisp 求值机制的作用。也就是说，当受到函数的作用时，Q-表达式不会被求值，而是保持原样。这个特性让 Q-表达式有着广泛的应用。我们可以用它来存储和管理其他的 Lisp 值类型，例如数字、符号或 S-表达式等。

在添加 Q-表达式之后，我们还需要定义一系列的操作来管理它。类似于数学操作，这些操作定义了 Q-表达式具体的行为。

Q- 表达式的语法和 S-表达式非常相似，唯一的不同是 Q-表达式包裹在大括号 `{}` 中，而非 S-表达式的小括号 `()`，Q-表达式的语法规则如下所示。

> #### 我从来没听说过 Q-表达式
> 好吧，其实 Q-表达式不存在于其它的 Lisp 方言中，它们通常使用宏来禁止表达式求值。宏看起来类似于普通的函数，但不会对参数进行求值。有一个特殊叫做引用(`'`)的宏，可以用来禁止几乎所有表达式的求值，这个宏也是本书中 Q-表达式的灵感来源。所以 Q-表达式是 Lispy 独有的，我们用它来替代宏完成相应的任务。
> 
>  本书中的 S-表达式和 Q-表达式有滥用概念的嫌疑，但我希望这些“不恰当的行为”能够使我们的 Lispy 的行为更加清晰简洁。

```c
mpc_parser_t* Number = mpc_new("number");
mpc_parser_t* Symbol = mpc_new("symbol");
mpc_parser_t* Sexpr  = mpc_new("sexpr");
mpc_parser_t* Qexpr  = mpc_new("qexpr");
mpc_parser_t* Expr   = mpc_new("expr");
mpc_parser_t* Lispy  = mpc_new("lispy");

mpca_lang(MPCA_LANG_DEFAULT,
  "                                                    \
    number : /-?[0-9]+/ ;                              \
    symbol : '+' | '-' | '*' | '/' ;                   \
    sexpr  : '(' <expr>* ')' ;                         \
    qexpr  : '{' <expr>* '}' ;                         \
    expr   : <number> | <symbol> | <sexpr> | <qexpr> ; \
    lispy  : /^/ <expr>* /$/ ;                         \
  ",
  Number, Symbol, Sexpr, Qexpr, Expr, Lispy);
```

另外，不要忘记同步更新清理函数 `mpc_cleanup` 来处理我们新添加的规则。

```c
mpc_cleanup(6, Number, Symbol, Sexpr, Qexpr, Expr, Lispy);
```

## 读取 Q-表达式

由于 Q-表达式和 S-表达式的形式基本一致，所以它们内部实现也大致是相同的。我们考虑重用 S-表达式的数据结构来表示 Q-表达式，在此之前需要向枚举中添加一个单独的类型。

```c
enum { LVAL_ERR, LVAL_NUM, LVAL_SYM, LVAL_SEXPR, LVAL_QEXPR };
```
另外，还需为其编写一个构造函数。

```c
/* A pointer to a new empty Qexpr lval */
lval* lval_qexpr(void) {
  lval* v = malloc(sizeof(lval));
  v->type = LVAL_QEXPR;
  v->count = 0;
  v->cell = NULL;
  return v;
}
```

Q-表达式的打印和删除逻辑也和 S-表达式别无二致，我们只需照葫芦画瓢，在相应的函数中添加对应的逻辑即可，具体如下所示。

```c
void lval_print(lval* v) {
  switch (v->type) {
    case LVAL_NUM:   printf("%li", v->num); break;
    case LVAL_ERR:   printf("Error: %s", v->err); break;
    case LVAL_SYM:   printf("%s", v->sym); break;
    case LVAL_SEXPR: lval_expr_print(v, '(', ')'); break;
    case LVAL_QEXPR: lval_expr_print(v, '{', '}'); break;
  }
}
```

```c
void lval_del(lval* v) {

  switch (v->type) {
    case LVAL_NUM: break;
    case LVAL_ERR: free(v->err); break;
    case LVAL_SYM: free(v->sym); break;

    /* If Qexpr or Sexpr then delete all elements inside */
    case LVAL_QEXPR:
    case LVAL_SEXPR:
      for (int i = 0; i < v->count; i++) {
        lval_del(v->cell[i]);
      }
      /* Also free the memory allocated to contain the pointers */
      free(v->cell);
    break;
  }

  free(v);
}
```

经过这些简单的变化之后，我们就可以更新读取函数 `lval_read`，使其可以正确读取 Q-表达式了。因为 Q-表达式重用了所有 S-表达式的数据类型，所以我们也自然可以重用所有 S-表达式的函数，例如 `lval_add`。

因此，为了能够读取 Q-表达式，我们只需在抽象语法树中检测并创建空的 S-表达式的地方添加一个新的情况即可。

```c
if (strstr(t->tag, "qexpr"))  { x = lval_qexpr(); }
```

因为 Q-表达式没有任何求值方式，所以无需改动任何已有的求值函数，我们的 Q-表达式就可以小试牛刀了。尝试输入几个 Q-表达式，看看是否不会被求值。

```shell
lispy> {1 2 3 4}
{1 2 3 4}
lispy> {1 2 (+ 5 6) 4}
{1 2 (+ 5 6) 4}
lispy> {{2 3 4} {1}}
{{2 3 4} {1}}
lispy>
```