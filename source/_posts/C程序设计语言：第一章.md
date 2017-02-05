---
title: C程序设计语言：第一章
date: 2017-02-05 20:38:48
tags:
- C
- 阅读笔记
category:
- C程序语言设计
---

#### 题记
总算要开始回头看C语言了，太久没用这门语言了，大概有5年了吧。
真是时光荏苒，但是为了进步，以及更高的编程愉悦感，我还是得忍着重新学习的痛苦，再回顾一遍这本经典的《C程序语言设计》了。
这里就记录一下心得，还有一些有趣的习题解答吧。

#### 练习

##### 1.5.2字符计数

```C

#include<stdio.h>

/*
 * 练习1-8 编写一个统计空格、制表符、与换行符个数的程序
 * */

int main(void){
    long nc = 0;
    int ch;
    while( (ch = getchar())!= EOF ){
        if(ch=='\n' || ch=='\t' || ch==' '){
            nc++;
        }
    }
    printf("%ld\n", nc);
    return 0;
}
```

```C
#include<stdio.h>

/*
 * 练习1-9 编写一个将输入复制到输出的程序，并将其中连续的多个空格用一个空格替代
 * */

int main(void){
    int pre;
    int curr;
    curr = getchar();
    pre = -1;

    while(curr!=EOF){
        if(curr!=' '||pre!=' '){
            putchar(curr);
        }
        pre = curr;
        curr = getchar();
    }

    return 0;
}

```



