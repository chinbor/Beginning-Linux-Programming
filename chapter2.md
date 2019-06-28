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

## 3. 一个完整的终端控制程序

```c

/*=======================================================================================
 * 1.函数原型:
 *		setupterm(char *term,int fd,int *errret);
 *
 * 2.包含头文件:			
 *		#include <term.h>
 *
 * 3.Description:
 *		将当前终端类型设置为term指向的值，如果term为NULL，就是用环境变量TERM的值,参数fd
 *      为一个打开的文件描述符,他用于向终端写数据，如果参数errret不是一个空指针,则函数的
 *		返回值保存在该参数指向的整形变量中,如果errret被设置为空指针,那么函数就会在失败的
 *		时候输出一条诊断信息并导致程序直接退出	
 *=======================================================================================*/

/*=======================================================================================
 * 1.函数原型:
 *		char *tigetstr(char *capname);
 *
 * 2.包含头文件:			
 *		#include <term.h>
 *
 * 3.Description:
 *		返回字符串功能标志的值
 *		例如 tigetstr("cup");返回光标移动功能标志cup的值:\E[%p1%d;%p2%dH
 *=======================================================================================*/

/*=======================================================================================
 * 1.函数原型:
 *		char *tparm(char *cap,long p1,long p2, ... ,long p9);
 *
 * 2.包含头文件:			
 *		#include <term.h>
 *
 * 3.Description:
 *		用实际的数值替换功能标志对应的值中的参数，最多可以替换9个
 *		例如:
 *		char *cursor;
 *		char *esc_sequence;
 *		cursor = tigetstr("cup");
 *		esc_sequence = tparm(cursor,5,30);		构造escape转义序列（可以控制终端）
 *		putp(esc_sequence);			终端控制字符串，作为参数，并且将其发送到标准输出stdout
 *=======================================================================================*/

/*=======================================================================================
 * 1.函数原型:
 *		int *tputs(char *const str,int affcnt,int (*putfunc)(int));
 *
 * 2.包含头文件:			
 *		#include <term.h>
 *
 * 3.Description:
 *		tputs函数是为了不能通过标准输出stdout访问终端的情况准备的,他可以指定一个用于输出字
 *		符的函数。tputs函数的返回值是用户指定的函数putfunc的返回结果。参数affcnt的作用是表
 *		明受这一变化影响的行数，一般设置为1，真正用于输出控制字符串的函数的参数和返回值类型
 *		必须与putfunc函数相同
 *=======================================================================================*/

/*=======================================================================================
 * 1.函数原型:
 *		int fflush(FILE *stream);
 *
 * 2.包含头文件:			
 *		#include <stdio.h>
 *
 * 3.Description:
 *		将文件流中所有未写出的数据立刻写出
 *=======================================================================================*/

/*=======================================================================================
 * 1.函数原型:
 *		int putc(int c,FILE *stream);
 *
 * 2.包含头文件:			
 *		#include <stdio.h>
 *
 * 3.Description:
 *		将单个字符送到输出流
 *=======================================================================================*/

#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <termios.h>
#include <term.h>
#include <curses.h>

static FILE *output_stream = (FILE *)0;


char *menu[] = {
	"a - add new record",
	"d - delete record",
	"q - quit",
	NULL,
};

int getchoice(char *greet, char *choice[],FILE *in,FILE *out);
int char_to_terminal(int char_to_write);

int main()
{
	int choice = 0;
	/* 这里定义两个文件流，与标准输入输出区分开来 */
	FILE *input;
	FILE *output;
	struct termios initial_settings,new_settings;
	/* 函数isatty()通过检测文件描述符(通过fileno(FILE *stream)转换)是否和终端关联,从而判断检测标准输出是否重定向了 */
	if(!isatty(fileno(stdout))){
		fprintf(stderr,"You are not a terminal, OK.\n");
	}
	/* 两个不同的文件输入输出流 */
	input = fopen("/dev/tty","r");
	output = fopen("/dev/tty","w");
	if(!input || !output){
		fprintf(stderr,"Unable to open /dev/tty\n");
		exit(1);
	}
	tcgetattr(fileno(input),&initial_settings);
	new_settings=initial_settings;
	/* 关闭标准输入处理 */
	new_settings.c_lflag &= ~ICANON;
	/* 关闭回显 */
	new_settings.c_lflag &= ~ECHO;
	/* 由于前面设置了非标准模式，所以，可以设置这里的MIN=1，TIME=0，含义是:read调用将一直等待,直到有MIN个字符可以读取时才返回,返回值是读取的字符数量,到达文件尾返回0,这里很重要,因为他这样设置了就只会响应一个字符的读入,不需要回车他就自己响应了(解决了回车换行字符的问题) */
	new_settings.c_cc[VMIN] = 1;
	new_settings.c_cc[VTIME] = 0;
	/* 本地模式下的启用信号，这里是取消启用信号，这里就是通过清楚这个标志来禁用对像ctrl+c(终止程序)这种特殊字符的处理 */
	new_settings.c_lflag &= ~ISIG;
	if(tcsetattr(fileno(input),TCSANOW,&new_settings) != 0){
		fprintf(stderr,"Could not set attributes\n");
	}
	do
	{
		choice = getchoice("Please select an action",menu,input,output);
		/* 会输出到标准输出 stdout，所以这里的输出会被重定向到文件file,如所你执行的是 ./menu5 > file 2>&1 */
		printf("You have chosen: %c\n",choice);
		sleep(1);
	}while(choice != 'q');
	/* 恢复原来的终端接口状态 */
	tcsetattr(fileno(input),TCSANOW,&initial_settings);
	exit(0);
}

int getchoice(char *greet, char *choice[],FILE *in,FILE *out)
{
	int chosen = 0;
	int selected;
	int screenrow,screencol = 10;
	char **option;
	char *cursor,*clear;
	output_stream = out;
	/* 当前终端类型设置为TERM值 */
	setupterm(NULL,fileno(out),(int *)0);
	/* 获得cup光标行动标志的值 */
	cursor = tigetstr("cup");
	/* 获得clear标志的值 */
	clear = tigetstr("clear");
	screenrow = 4;
	/* 将clear的值一个字符一个字符的输出到out流(控制终端的行为，达到清屏作用) */
	tputs(clear,1,&char_to_terminal);
	/* 先用tparm将光标移动到的行和列都给赋值了，然后在用tputs函数将转义序列输出到out流，达到光标的移动 */
	tputs(tparm(cursor,screenrow,screencol),1,&char_to_terminal);
	fprintf(out,"Choice: %s\n",greet);
	screenrow += 2;
	option = choice;
	while(*option){
		tputs(tparm(cursor,screenrow,screencol),1,&char_to_terminal);
		fprintf(out,"%s",*option);
		screenrow++;
		option++;
	}
	fprintf(out,"\n");
	do{
		/* 将out流中未写出的数据全部写出. */
		fflush(out);
		selected = fgetc(in);
		option = choice;
		while(*option){
			if(selected == *option[0]){
				chosen = 1;
				break;
			}
			option++;
		}
		if(!chosen){
			tputs(tparm(cursor,screenrow,screencol),1,&char_to_terminal);
			fprintf(out,"%c is a invalid choice, select again\n",selected);
		}
		screenrow++;
	}while(!chosen);
	/* 加入这个语句进行判断是为了看这个函数是否运行成功了的 */
	if(!tputs(clear,1,&char_to_terminal)) return selected;
}

int char_to_terminal(int char_to_write)
{
	if(output_stream) putc(char_to_write,output_stream);
	return 0;
}
```

* 注意这里可以使用 **.filename**执行,也可以使用 **./filename > file 2>&1**将标准输出和标准错误输出重定向到file文件,这样就可以隐藏一些内容使用户看不见，而只看见能够交互的内容
