### 信号（也称软中断，借助软件手段实现终端）

###### 信号的概念

> 基本的属性 

特点：

1. 简单 

2. 不能携带大量信息 

3. 需要满足某个特设条件

4. 每个进程收到的所有信号，都是由内核负责发送，内核处理。  

具体过程：信号产生   - >  内核处理  -> 执行信号

阻塞信号集（信号屏蔽字），未决信号集 ：

  类型为set （默认无序，并且无重合）， 位于内存的PCB中。编号从1开始，默认每个信号对应的标记都是0，如果这个信号产生的话，标记变为1.这两个集合之间的关系是，在信号产生的前提下，也就是当前信号在阻塞信号集和未决信号集对于当前的信号都存在时，阻塞信号集能决定未决信号集。当阻塞信号集中该信号成功从1变为0的时候，这个时候未决信号集中该信号也变为0.

二，递达信号

三，未决：产生和递达之间的状态，主要由于阻塞（屏蔽）导致该状态。

信号的处理方式：

1. 执行默认动作
2. 忽略（丢弃）**已经处理，但是没有任何反应，在未决信号集中仍未0**
3. 捕捉（调用自己定义的动作）

> 信号四要素

1. 编号
2. 名称
3. 事件（段错误SIGPIPE（编号11）等）
4. 默认处理动作

查看信号方法 ：第一个，kill -l  （掌握常用的信号及其意义）第二个 man 7 signal 

ps：man 文档中，一个信号会有多个信号，是为了适应多个平台。一般取中间那个数

  **默认动作**

Term：终止进程    Ign ：忽略信号  Core：终止进程（生成core文件，查验进程死亡原因，用于gdb调试 （ **调试方法**，-g gdb a.out run ，停在哪里，哪里就有段错误）（原因：a.out 在卡死的时候，会生成core文件，然后gdb能把他拿出来）

Stop：停止（暂停）进程  Cont： 继续进行进程  

**注意：**9号进程和19号进程不允许捕捉，不允许堵塞，只 能执行默认动作

**产生信号的五中方式**

1. 按键产生，如：ctrl + c. 
2. 系统调用产生 如：kill
3. 软件条件产生 如：定时器 alarm
4. 硬件异常产生 如：段错误
5. 命令产生如：kill命令



#### 信号的产生

> 终端按键产生信号

 ctrl + c   2号信号 SIGINT（终止/中断）

ctrl + z  20号信号 SIGSTP （暂停/停止） 注意区分SIGSTOP

ctrl + \  SIGOUT  (退出)

> 硬件异常产生信号

除0操作  8号进程 SIGFPE

非法访问内存 11号进程SIGSEGV 段错误

总线错误  7号进程 SIGBUS

> 信号集操作函数

kill函数/ kill 命令产生信号

int kill(pid_t pid ,int sig);

sig 最好使用宏名

pid 和waitpid中的pid参数类似

**kill函数和waitpid函数的区别，kill函数当参数为负的组号的时候，会杀死该组内所有的进程，但是waitpid只会回收该组内随机的一个进程！！！**

练习：

简单调用kill函数

```c
/*************************************************************************
    > File Name: kill.c
    > Author: ma6174
    > Mail: ma6174@163.com 
    > Created Time: 2019年07月24日 星期三 19时49分44秒
 ************************************************************************/
#include<unistd.h>
#include<sys/types.h>
#include<stdio.h>
#include<stdlib.h>
#include<sys/types.h>
#include<signal.h>
int main(  )  {
	  
pid_t pid;
pid = getpid(  )  ;
int ans = kill( pid , 9);
int ans = kill(pid , SIGKILL);
 // 相对于第一个来说，第二个更加保险，因为在跨平台的过程中，信号编号可能会有变化
if( ans == -1 ){
	  perror("kill error");
	  exit(1);  
}
printf("Yes\n");
return 0;
}
```

练习：

创建五个子进程，然后通过kill命令，父进程杀掉第三个子进程

这里有两种打法，一种是等到所有的子进程都跑完了然后再去杀死2,；还有一钟就是只要有2的进程就给他整死。可以通过查看ps aux 来检验。

```c
/*************************************************************************
    > File Name: kill.c
    > Author: ma6174
    > Mail: ma6174@163.com 
    > Created Time: 2019年07月24日 星期三 19时49分44秒
 ************************************************************************/
#include<unistd.h>
#include<sys/types.h>
#include<stdio.h>
#include<sys/wait.h>
#include<stdlib.h>
#include<sys/types.h>
#include<signal.h>
int main(  )  {
pid_t tmp;	  
pid_t pid , wpid ; 
int i;
int cnt = 0 ;
for ( i = 0 ; i< 5;i++)  {
	 pid =  fork(  );
	 if(pid == -1){
		perror("fork error");
		exit(1);  
	 }
	 if( i== 2 ){
		   tmp = pid ;
	 }
	  if(pid == 0)  {
		   break;
	 }
}
 
if( i == 2 ){
sleep(3);  
tmp = getpid(  )  ;
int ans = kill( tmp , SIGKILL);
if(ans < 0 ){
perror("kill error");
exit(1);  
}
}  
if( i< 5 ) {
	  while(1){
		    printf( "I am %d th child\n" ,i );
			sleep(2);  
	  }
}
return 0;
}
```

raise函数：给当前进程发送指定信号（自己给自己发）

abort函数：给自己发异常终止的信号 6号信号SIGABRT ，终止并产生Core文件

#### 软件条件产生信号

**alarm函数**  

14号信号SIGALRM

alarm（5）-> 3sec -> alarm(4) -> 5sec ->alarm(5) ->alarm(0)

alarm(0)取消闹钟

解释：通过查看man文档，当之前没有调用alarm函数时，返回 0 ；当之前有alarm时，返回上一个alarm函数剩余的时间。 

  每个进程都有且只有唯一定时器

time 命令 统计时间

real  实际

user 用户

sys   内核

实际时间 == 系统时间 + 用户时间 + 等待时间

**setitimer 函数**

设置定时器（闹钟），可代替alarm函数，精度微秒 us

可以设置当前计时和上一次计时相差多长时间

int setitimer(int which ,const struct itimerval *new valude ,struct itimerval *oldvalue);

which : 表示在哪计时

ITIMER_REAL  实际

ITIMER_VIRTUAL  虚拟空间计时 用户空间  

ITIMER_PROF  运行时计时（用户 + 内核）

new valude：当前这次计时的信息

old valude：上一次计时的信息 ，传出参数

   练习：setitimer函数计算一秒内可以数多少个数

```c
*************************************************************************
    > File Name: setitimer.c
    > Author: ma6174
    > Mail: ma6174@163.com 
    > Created Time: 2019年07月25日 星期四 11时36分05秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<sys/time.h>
//typedef struct itimerval{
// typedef  struct timeval{
//    time_t tv_sec;
//	suseconds_t tv_usec;
 // }it_interval ,  itvalue;
//};
unsigned int  my_alarm(unsigned int sec){
   struct itimerval it , oldit;
   int ret;
   
   it.it_interval.tv_sec = 0;
   it.it_interval.tv_usec = 0 ;

   it.it_value.tv_sec = sec;
   it.it_value.tv_usec = 0;
   ret = setitimer(ITIMER_REAL , &it , &oldit);
   if(ret == -1){
	    perror("setitimer error");
		exit(1);  
   }
   return ret ;
}
int main(  )  {
int i = 0 ;
my_alarm(1);
while(1){
printf("%d\n" , i++ );
}
return 0;
}
```

**signal函数** 

 void (*sighandler_t)(int);

sighandler_t signal( int signum  , sighandler_t handler);

```c
/*************************************************************************
    > File Name: setitimer.c
    > Author: ma6174
    > Mail: ma6174@163.com 
    > Created Time: 2019年07月25日 星期四 11时36分05秒
 ************************************************************************/
#include<unistd.h>
#include<stdio.h>
#include<stdlib.h>
#include<sys/time.h>
#include<signal.h>
//typedef struct itimerval{
  
// typedef  struct timeval{
//    time_t tv_sec;
//	suseconds_t tv_usec;
 // }it_interval ,  itvalue;
//};
void mufync(  )  {
	  printf("haahahahahha\n");  
}
int main(  )  {
int i = 0 ;
   struct itimerval it , oldit;
   int ret;
   signal(SIGALRM , mufync);  
   it.it_interval.tv_sec = 5;
   it.it_interval.tv_usec = 0 ;

   it.it_value.tv_sec = 3;
   it.it_value.tv_usec = 0;
    // 这里第一个hahahah是3秒后出现，剩下的就都是5秒了，注意定义方法
   ret = setitimer(ITIMER_REAL , &it , &oldit);
   if(ret == -1){
	    perror("setitimer error");
		exit(1);  
   }
  // my_alarm(1);
    while(1);    
   return 0;
}
```

#### 信号集操作函数   // 只是注册函数，真正捕捉信号的是内核！！！

####  信号集设定

sigset_t set;  // typedef unsigned long sigset_t;

1. int sigemptyset(sigset_t *set); // 将某个信号集清零，注意是指针类型

2. int sigfillset(sigset_t &set); // 将某个信号集全部填充为1

3. int  sigaddset(sigset_t *set ,int signum); 

4. int sigdelset(sigset_t *set ,int signum); 从set中增加或者删除信号signum（ 0->1 ，或者 

   1 -> 0 ）

5. int sigismember(const sigset_t *set,int signum); // 检测是否存在



**sigprocmask函数**

 读取或修改进程的信号屏蔽集（PCB块中）。可用来屏蔽信号，解除屏蔽。

 int sigprocmask( int how ,const sigset_t *set ,sigset_t *oldset);

参数：

 set : 传入参数，保存旧的信号屏蔽集

oldset：传出参数，保存旧的信号屏蔽集

how参数取值：假设当前的信号屏蔽字为mask

1. SIG_BLOCK : 当how设置为此值，set表示要屏蔽的信号。相当于mask = mask | set 

2. SIG_UNBLOCK : set表示要解除屏蔽的信号，相当于 mask = mask &~mask 

3. SIG_SETMASK: set表示用于替代原始屏蔽及新的屏蔽集。相当于mask = set . 当接触了若干个信号的阻塞，就在sigpromask返回之前，至少将其中一个信号传递

   

**sigpending函数**

读取当前进程的未决信号集

int sigpending( sigset_t *set); set传出参数 

练习：编写程序，把所有常规信号的未决状态打印至屏幕

具体思路：首先创建一个空的，然后初始化，然后对特定的信号进行增删，然后对屏蔽集进行操作，通过屏蔽集就能影响到未决信号集再就是输出检验即可。

```c
/*************************************************************************
    > File Name: printpending.c
    > Author: ma6174
    > Mail: ma6174@163.com 
    > Created Time: 2019年07月25日 星期四 16时35分19秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<signal.h>
#include<unistd.h>
void printped(sigset_t *ped )  {
int i;
for ( i = 1 ; i< 32; i++ ){
	if( sigismember(ped , i) == 1 ){ //判断当前信息的状态
		  putchar('1');  
	}
	else putchar('0');  
}
printf("\n");
}
int main(void )  {
	  
sigset_t myset,oldset,ped;

sigemptyset(&myset);  // 初始化

sigaddset(&myset , SIGQUIT);  // 增加信号

sigprocmask(SIG_BLOCK ,&myset ,&oldset); // 屏蔽3号信号

while(1){
sleep(1);  	
sigpending( &ped );
printped( &ped ) ;
}  
return 0;
}
```

> 信号的捕捉

   

```c
/*************************************************************************
    > File Name: signal2.c
    > Author: ma6174
    > Mail: ma6174@163.com 
    > Created Time: 2019年07月25日 星期四 17时40分57秒
 ************************************************************************/

#include<stdio.h>
#include<signal.h>
#include<stdlib.h>
#include<unistd.h>

typedef void (*sighandler_t) (int); 
void catchsight(int  tmp )  {   // 这里的tmp是信号的编号
	 printf("````````````````%d`\n",tmp);   
}
int main(  )  {
    sighandler_t handler;
	handler = signal(SIGINT , catchsight);  
   if(handler == SIG_ERR){
	    perror("signal error");
		exit(1);  
   }
   while(1);   //不让程序停止
   return 0;
}
```

**sigaction函数**

int sigaction( int signum , const struct sigaction *act , struct sigaction *oldact);

参数：

signum 信号编号

struct sigaction{

void      (*sa_handler)(int) ;  // 函数名 

void      (*sa_sigaction)(int , siginfo_t *, void *);

sigset_t  sa_mask;  //  **用于指定在信号捕捉函数执行期间所屏蔽的信号集（在信号过来的时候，函数处理这个信号的期间，要屏蔽哪些信号）**

int        sa_flags;  

 //一大坨参数，具体查看man文档

void     (*sa_restorer)(void);

 

```c
/*************************************************************************
    > File Name: sigaction.c
    > Author: ma6174
    > Mail: ma6174@163.com 
    > Created Time: 2019年07月25日 星期四 18时36分17秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<signal.h>
#include<unistd.h>

typedef void (*sighandler_t) ( int );

void docatch(int num )  {
	printf( "-----------------%d\n",num);
	return ;
}
int main(  )  {
	int ret ;
	
	struct sigaction act;
	act.sa_handler = docatch;
    sigemptyset( &act.sa_mask);
	sigaddset( &act.sa_mask , SIGQUIT);
	
	act.sa_flags = 0;  // 默认为0 功能就是该信号捕捉函数正在运行时，不再接受相同的信号进入函数

    ret = sigaction(SIGINT , &act ,NULL);
	if( ret < 0 ){
		perror("sigaction error");
		exit(1);  
	}
	while(1);
	return 0;
}
```

#### 信号捕捉特性

1. 对于sigaction函数，在信号执行期间，这个时候阻塞信号集是函数中的sa_mask。当执行完之后会再换回来。

2. xxx信号捕捉函数执行期间，xxx信号自动屏蔽
3. 阻塞的长队信号不支持排队，产生多次的话，只记录一次

#### 内核实现信号捕捉过程

1. 在执行主流程的某条指令的时候因为 出现异常，中断或者系统调用进入内核
2. 在内核中，对于该条指令如果信号的处理动作是默认的话，调用完之后就直接从内核回到主流程了。

如果该条指令的信号处理动作不是默认的话，那么就从内核进入主流程执行自定义的信号处理动作

3. 当信号的处理动作不是默认的前提下，调用完自定义的动作时，通过特殊的系统调用sigreturn 再次进入内核
4. 返回内核之后从上次中断的地方继续往下执行