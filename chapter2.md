# Second

## 1. 记录一下我在vim中的问题（与系统共享剪贴板，实现vim中的内容与外面的内容可以相互复制粘贴）
> 打开终端

> 输入“vim --version | grep +clipboard”

> 若有+clipboard说明可用剪贴板(一般最新版vim都下载了的)

> 打开vim 在命令模式下输入:reg查看是否有 "+ 这个寄存器

> 若以上的都有，那么恭喜你，你可以使用 **"+y**和 **"+p**进行共享剪贴板的复制和粘贴了

## 2. 关于函数指针
* **无typedef**

```c
#include <stdio.h>
#include <stdlib.h>

void (*funP)(int);
void (*funA)(int);
void myFun(int x);
int main()
{
    funP=&myFun;
    //深入理解
    printf("sizeof(myFun)=%d\n",sizeof(myFun));
    printf("sizeof(funP)=%d\n",sizeof(funP));
    printf("myFun\t 0x%p=0x%p\n",&myFun,myFun);
    printf("funP\t 0x%p=0x%p\n",&funP,funP);
    printf("funA\t 0x%p=0x%p\n",&funA,funA);
    return 0;
}

void myFun(int x)
{
    printf("myFun: %d\n",x);
}
```

* **有typedef**

```c
#include <stdio.h>
#include <stdlib.h>

typedef void(*FunType)(int);
//前加一个typedef关键字，这样就定义一个名为FunType函数指针类型，而不是一个FunType变量。
//形式同 typedef int* PINT;
void myFun(int x);
void hisFun(int x);
void herFun(int x);
void callFun(FunType fp,int x);
int main()
{
    callFun(myFun,100);//传入函数指针常量，作为回调函数
    callFun(hisFun,200);
    callFun(herFun,300);

    return 0;
}

void callFun(FunType fp,int x)
{
    fp(x);//通过fp的指针执行传递进来的函数，注意fp所指的函数有一个参数
}

void myFun(int x)
{
    printf("myFun: %d\n",x);
}
void hisFun(int x)
{
    printf("hisFun: %d\n",x);
}
void herFun(int x)
{
    printf("herFun: %d\n",x);
}
```
* **仔细观察两者的区别可以发现使用了typedef的自定义函数指针类型，不需要函数名取地址就能赋值函数，而没用的就是声明一个函数指针变量，赋值需要函数名取地址**
