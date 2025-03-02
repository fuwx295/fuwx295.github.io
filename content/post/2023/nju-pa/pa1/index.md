+++
author = "GreyWind"
title = "PA 1: simple computer"
date = "2023-10-01"
description = "PA 1: simple computer"
tags = [
    "NJU-PA",
]
+++
## Introduction

In lab 1, we will implement a simple debugger. There are three main tasks.

Task 1, Implement single-step execution, Print register status, Scan memory.

Task 2, Implement expression evaluation.

Task 3, Perfect expression evaluation and Implement expression evaluation.

## Implementation

### Correct exit

when using the command `q` to exit, the `nemu` will output an error message.

To fix it, you should debug through `gdb` and know must change the state.

```c
static int cmd_q(char* args) {
    nemu_state.state = NEMU_QUIT;
    return 0;
};
```

### Simple debugger

#### Single-step execution

add `si` cmd to `cmd_table`

```c
static struct {
  const char *name;
  const char *description;
  int (*handler) (char *);
} cmd_table [] = {
  { "help", "Display information about all supported commands", cmd_help },
  { "c", "Continue the execution of the program", cmd_c },
  { "q", "Exit NEMU", cmd_q },

  /* TODO: Add more commands */
  { "si", "Execute N steps, default 1", cmd_si },
};
```

implement `cmd_si` function

```c
static int cmd_si (char *args) {
    int n = 1;
    ...(analytic parameter to n)
    // execution N step
    cpu_exec(n);
    return 0;
};
```

#### Print register status

add `info` cmd to `cmd_table`

```c
{ "info", "Display information about registers or watchpoints", cmd_info },
```

implement `cmd_info` function

the display register function is `isa_reg_display`, you need to modify it.

```c
static int cmd_info(char *args) {
  // analytic parameter
  char *arg = strtok(NULL, " ");
  if (arg == NULL) {
    // error handler
  } else {
    if (strcmp(arg, "r") == 0) {
      // display register
      isa_reg_display()
    } else if (strcmp(arg, "w") == 0) {
      // display watchpoint (need to impl)
    } else {
      // error handler
    }
  }
  return 0;
}
```

#### Scan memory

add `x` cmd to `cmd_table`

```c
{ "x", "Usage: x N EXPR. Scan the memory from EXPR", cmd_x },
```

implement `cmd_x` function

```c
static int cmd_x(char *args) {
  // need to get parameter
  char *arg1, *arg2;
  if (arg1 == NULL || arg2 == NULL) {
    // error handler
    return 0;
  }
  // transfor type
  int n = strtol(arg1, NULL, 10);
  vaddr_t expr = strtol(arg2, NULL, 16);

  // print memory
  ... using vaddr_read(), format like gdb
  return 0;
}
```

### Expression evaluation

To start this, you must read to experiment document [expression evaluation](https://nju-projectn.github.io/ics-pa-gitbook/ics2022/1.5.html) and

[watchpoint](https://nju-projectn.github.io/ics-pa-gitbook/ics2022/1.6.html). Know how to analyze the expression and calculate it.

add `p` cmd to `cmd_table`

```c
{ "p", "Usage: p EXPR. Calculate the experssion", cmd_p },
```

implement `cmd_p` function, and all details in the `expr()` function.

```c
static int cmd_p(char *agrs) {
  bool success;
  word_t res = expr(agrs, &success);
  if (!success) {
    printf("invalid expression\n");
  } else {
    printf("%u\n", res);
  }
  return 0;
}

word_t expr(char *e, bool *success){
  if (!make_token(e)) {
    *success = false;
    return 0;
  }

  return eval(0, nr_token - 1, success);
}
```

finish `make_token` and `eval` function, all hints in the experiment document.

we should categorize all tokens.

```c
enum {
  TK_NOTYPE = 256,

  /* TODO: Add more token types */
  TK_NUM, // number
  TK_REG,

  TK_POS, TK_NEG, TK_DEREF,
  TK_EQ, TK_NEQ, TK_GT, TK_LT, TK_GE, TK_LE,
  TK_AND, TK_OR,
};

static struct rule {
  const char *regex;
  int token_type;
} rules[] = {

  /* TODO: Add more rules.
   * Pay attention to the precedence level of different rules.
   */
  {" +", TK_NOTYPE},    // spaces
  {"\\+", '+'},         // plus
  {"==", TK_EQ},        // equal
  {"-", '-'},
  {"\\(", '('},
  {"\\)", ')'},
  {"\\*", '*'},
  {"/", '/'},
  {"<", TK_LT},
  {"<=", TK_LE},
  {">", TK_GT},
  {">=", TK_GE},
  {"!=", TK_NEQ},
  {"&&", TK_AND},
  {"\\|\\|", TK_OR},

  {"[0-9]+", TK_NUM},
  {"\\$\\w+", TK_REG},
};

static int bound_types[] = {')', TK_NUM, TK_REG};
static int nop_types[] = {'(', ')', TK_NUM, TK_REG};
static int op_types[] = {TK_NEG, TK_POS, TK_DEREF};
```

modify `make_token`

```c
static bool make_token(char *e) {
  int position = 0;
  int i;
  regmatch_t pmatch;

  nr_token = 0;

  while (e[position] != '\0') {
    /* Try all rules one by one. */
    for (i = 0; i < NR_REGEX; i ++) {

      if (regexec(&re[i], e + position, 1, &pmatch, 0) == 0 && pmatch.rm_so == 0) {
        char *substr_start = e + position;
        int substr_len = pmatch.rm_eo;

        Log("match rules[%d] = \"%s\" at position %d with len %d: %.*s",
            i, rules[i].regex, position, substr_len, substr_len, substr_start);

        position += substr_len;

        /* TODO: Now a new token is recognized with rules[i]. Add codes
         * to record the token in the array `tokens'. For certain types
         * of tokens, some extra actions should be performed.
         */

        if (rules[i].token_type == TK_NOTYPE) break;
        // add to token
        tokens[nr_token].type = rules[i].token_type;

        switch (rules[i].token_type) {
          case TK_NUM:
          case TK_REG:
            strncpy(tokens[nr_token].str, substr_start, substr_len);
            tokens[nr_token].str[substr_len] = '\0';
            break;
          case '*':
          case '-':
          case '+':
            if (nr_token == 0 || !type_from(tokens[nr_token-1].type, bound_types, ARRLEN(bound_types))) {
              switch (rules[i].token_type) {
                case '-': tokens[nr_token].type = TK_NEG; break;
                case '+': tokens[nr_token].type = TK_POS; break;
                case '*': tokens[nr_token].type = TK_DEREF; break; 
              }
            }
            break;
            
          //default: TODO();
        }
        nr_token++;

        break;
      }
    }

    if (i == NR_REGEX) {
      printf("no match at position %d\n%s\n%*.s^\n", position, e, position, "");
      return false;
    }
  }

  return true;
}

word_t eval(int p, int q, bool *success) {
  *success = true;
  if (p > q) {
    *success = false;
    return 0;
  } else if (p == q) {
    // just one , analyze op_token
  } else if (check_parentheses(p, q)) {
    // in both (), remove ().
  } else {
    // get op posistion
    int op = get_position(p, q);
    if (op < 0) {
      *success = false;
      return 0;
    }
    // eval left and right
    bool ok1, ok2;
    word_t val1 = eval(p, op - 1, &ok1);
    word_t val2 = eval(op + 1, q, &ok2);


    if (!ok2) {
      *success = false;
      return 0;
    }
    word_t ret;
    // two op or one op
    if (ok1) {
      ret = calc2(tokens[op].type, val1, val2, success);
    } else {
      ret = calc1(tokens[op].type,val2, success);
    }
    return ret;
  }
}
```

### Watchpoint

add `w,d` cmd to `cmd_table`

```c
{ "w", "Usage: w EXPR. set watchpoint", cmd_w },
{ "d", "USage: d N. delete watchpoint", cmd_d },
```

implement `cmd_w` and `cmd_d` function

```c
static int cmd_w(char *agrs) {
  if (agrs == NULL) {
    // error handler
    return 0;
  }
  bool success;
  // get expr
  word_t res = expr(agrs, &success);
  if (!success) {
    printf("invalid expression\n");
  } else {
    wp_add(agrs, res);
  }
  return 0;
}

static int cmd_d(char *args) {
  // analyze from args
  int num;
  wp_remove(num);

  return 0;
}
```

finish `wp_remove()`, `wp_add()` and `wp_display()` function

```c
void wp_add(char *expr, word_t res) {
  WP* wp = new_wp();
  strcpy(wp->expr, expr);
  wp->ret = res;
  printf("watchpoint set. %d: %s\n", wp->NO, wp->expr);
}

void wp_remove(int num) {
  assert(num < NR_WP);
  WP* wp = &wp_pool[num];
  free_wp(wp);
  printf("delete watchpoint. %d: %s", wp->NO, wp->expr);
}

void wp_display() {
  WP *wp = head;
  // iterate wp 
}
```

Put the check of watchpoints in `trace_and_difftest()` and use a new macro `CONFIG_WATCHPOINT`

```c
static void trace_and_difftest(Decode *_this, vaddr_t dnpc) {
#ifdef CONFIG_ITRACE_COND
  if (ITRACE_COND) { log_write("%s\n", _this->logbuf); }
#endif
  if (g_print_step) { IFDEF(CONFIG_ITRACE, puts(_this->logbuf)); }
  IFDEF(CONFIG_DIFFTEST, difftest_step(_this->pc, dnpc));
  IFDEF(CONFIG_WATCHPOINT, wp_difftest());
}
```

implement `wp_difftest()` in `watchpoint.c`.

```c
void wp_difftest() {
  WP* wp = head;
  while (wp) {
    bool _;
    //calc expr 
    word_t new = expr(wp->expr, &_);
    if (wp->ret != new) {
      // get to watchpoint.
      wp->ret = new;
    }
    wp = wp->next;
  }
}
```