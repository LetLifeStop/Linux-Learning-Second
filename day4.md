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

kill函数/ kill 命令产生信号

int kill(pid_t pid ,int sig);

sig 最好使用宏名

pid 和waitpid中的pid参数类似

**kill函数和waitpid函数的区别，kill函数当参数为全组的时候，会杀死该组内所有的进程，但是waitpid只会回收该组内随机的一个进程！！！**

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

信号集操作函数



> 信号屏蔽字



> 信号的捕捉

 

注册信号捕捉函数

 **sigaction函数**