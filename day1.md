### 进程相关概念

####  程序和进程

> **程序**

   是指编译好的二进制文件，不占用系统资源（CPU，内存等等）。这里拓展一下，磁盘和内存的区别，磁盘存储的内容，当关机后还是存在的。但是对于内存，关机后就消失了。然后磁盘分为硬盘和软盘，软盘相对于硬盘记录的信息比较小，一般装在外面；相对来说，硬盘就会装在电脑里面。

### 进程

 在运行期间是要占用系统资源的。（了解时钟中断（多道程序设计模型）概念，多个程序排队使用， 但是相对于单道程序模式相比，时钟中断可以卡住每个程序的时间，轮换使用CPU）

> **单道程序设计**

> **多道程序设计（并发）**

####  CPU和MMU

> **存储介质**

网络 - > 硬盘  -> 内存 -> cache（缓存） -> 寄存器（在CPU中） 。读取速度越来越快，存储的大小越来越小。

> **CPU**

中央处理器；处理信息的一般步骤，第一步，预取器（预取指令）；第二步，译码器（译码）；第三步，算数逻辑单元 ALU （执行）（只能进行两种处理，第一种 + ，第二种 <<）；第四步，回写数据到寄存器堆。

 当我们从硬盘开始执行一个命令的时候，是先从硬盘一步一步往上走，走到cache，也就是缓存，然后内存到CPU进行处理，处理完之后再回写到cache，然后再一步步的返回到硬盘。

> **MMU**

内存管理单元；位于CPU内部，作为一个硬件存在。功能一，虚拟内存与物理内存的映射，功能二，设置修改内存访问级别（级别越高，越容易访问），功能三，进程之间相互独立。  虚拟地址，可用的地址空间有4G 。空间映射的时候，是按照page（MMU最小单位）来进行申请的。32位的一个page页面大小是4k。

   对于两个不同的进程，两个相同的程序，包括.text , .data等文件的运行内存在物理内存中是相互独立的。 但是，在内核区，两个相同的程序，两个不同的进程，指向的物理内存是相同的，但是PCB（进程描述符）是不同的。  ？？疑问

  **熟悉虚拟内存的具体分层**

> 进程控制块

 按照结构体进行存储，如果查看具体信息通过 
grep -r "task_struct" /usr/include/

命令来进行查找

重点掌握以下部分：

1. 进程id，系统中每个进程有唯一的id.在linux中，通过pid_t类型来进行存储（非负整数）

2. 进程的状态 ，初始化，就绪态（一般前两个简化为一个就绪态）（等待CPU分配时间片），运行态（占用CPU），挂起态（首先主动放弃CPU，等待除CPU以外的其他资源），终止态。

3. 进程切换时需要保存和回复的一些CPU寄存器。（因为是多道程序设计，当再获得时间轮片的时候，能够保证继续按照上一次的进度继续往下存）。

4. 描述虚拟地址空间信息（与物理地址的映射关系，通过mmu维护一张表）。

5. 描述控制终端的信息。

6. 改变当前进程的工作目录。

7. umask掩码 ，改变文件权限（777）

8. 文件描述符表，包含很多指向file结构体的指针。

9. 和信号相关的信息。

10. 用户id和组id。

11. 会话和进程组。

12. 进程可以使用的资源上限。

    ulimit -a 查看进程可以使用的资源上限
    
    

####  环境变量

  **linux操作系统是一个多任务，多用户的开源操作系统。**

 指在操作系统当中指定操作系统环境的一些参数，因为是一个多用户的操作系统，每个人喜欢的环境是不同的，所以可以通过设置环境变量来实现多用户的这个要求。

> 存储形式

与命令号参数类似，char *[]数组，数组名为environ ，内部存储字符串，NULL作为哨兵结尾。

> 特性

1. 以字符串的形式来表达

2. 统一格式，名 = 值[:值] 

3. 值用来描述进程环境信息 

   PATH（可执行文件的路径） （**在执行shell命令的时候，是首先查看当前PATH环境变量从第前面开始找，找到一个可执行文件的路径，再去执行**）

   SHELL（记录当前会用的命令解析器是哪个）

   LANG（当前使用的语言）

   TEAM（当前终端类型）

   HOME（当前用户主目录的路径）

      使用方法： 

   echo &SHELL 

> 加载位置

​    与命令行参数类似。位于用户区，高于stack的位置。

> 引入环境变量表

 必须提前声明环境变量。 extern char **environ ;

ps：如果是date命令，并不是直接执行date命令；而是shell解析器去bin目录下执行了date命令. 

练习：打印当前进程的所有环境变量。 

> 处理环境变量的函数

1. getenv函数

   获取环境变量的值

2. setenv函数 设置环境变量的值 （如果要删除的环境变量在命名合法时，即使不存在也会返回0，也就是返回删除成功；如果是命名非法，返回 -1）

3. unsetenv函数  删除环境变量的值

```
#include<stdlib.h>
#include<stdio.h>
#include<string.h>
int main(  )  {
	char *val;
	const char *name = "ABD";
	val = getenv(name);
	printf("1,%s = %s\n" ,name ,val) ;
    setenv(name , "haha", 1);
	val = getenv(name);
	printf("2,%s = %s\n",name,val);
#if 0
	int ret = unsetenv("ABDFGH");
	printf("ret = %d\n", ret) ;

​```
val = getenv(name);
printf("3 %s = %s\n",name,val) ;
​```

#else
	int ret = unsetenv("ABD");
	printf("ret = %d\n",ret);
	val = getenv(name);
    printf("3, %s = %s\n", name, val);
#endif

​```
return 0;
​```

}   
```

ps：第一次，本来程序中是没有ABD这个进程的，所以环境变量肯定为null，然后通过setenv函数，把haha这个进程赋值给了ABD，这样，ABD的环境变量就不是null了。再通过unsetenv函数，把ABD的环境变量给删除掉，然后这个时候再去获得name的环境变量，就又1变成了null。



#### 进程控制原语

> fork函数

​    **返回值有两个：**

      1. 返回子进程的pid （ID一定是 > 0）
      2. 返回0或者 -1 

   ps: UID是用户ID ；PID是进程ID ；PPID是父进程ID

具体运行原理：当到达fork函数的时候，首先创建出一个子进程，然后这个父进程和子进程继续往下走；父进程的fork函数返回的是子进程的pid(>0)，子进程的fork函数返回的为是否创建成功（0 / -1 ）  .

  getpid（）获得当前进程的pid ，getppid（）获得当前进程的父进程的pid

```
#include<stdio.h>
#include<unistd.h>
#include<stdlib.h>
int main(void){
	pid_t pid;
	printf( "xxxxxxxxxx\n");
 	pid =  fork(  );
    if( pid == -1) {
		perror("  fork error");
		exit(1);  
	}
	else if(pid == 0){
		printf("I'am child, pid = %d, ppid = %u\n" , getpid( ) , getppid(  ));
	}
	else {
		printf( "pid = %u , ppid = %u\n",getpid(  ) ,getppid(  ));  
	}
    printf("12334343\n");  
	return 0;
}
```

 

**ps：12334343 会出现两个，但是xxxxxxxx只会出现一次**

作业：写一个有五个进程的c程序

**注意：**不能通过简单的for循环来进行创造，因为fork是按照当前的继续往下走，也就是会创造多于5个进程。 

如果是通过单纯的for循环的话，没有break，最终创造出来的子进程有 2 ^ n - 1个,每一次是创建一个fork之后，另一个是没有被fork的。 

```
#include<stdio.h>
#include<unistd.h>
#include<stdlib.h>
int main(void){
	pid_t pid;
	printf( "xxxxxxxxxx\n");
	for ( int i=0 ; i < 5;i++){
 	pid =  fork(  );
    if( pid == -1) {
		perror("  fork error");
		exit(1);  
	}
	else if(pid == 0){
		printf("I'am child, pid = %d, ppid = %u\n" , getpid( ) , getppid(  ));
	 break; // 当到达子进程时，break卡停
	}
	}
    printf("12334343\n");  
	return 0;
}
```

 

ps：顺序是不一定的，因为需要抢占CPU资源，可以通过sleep函数来实现按照顺序输出。 

##### getgid函数

> 获取当前进程使用用户组ID

  gid_t getgid(void );

> 获取当前进程有效用户组ID

gid_t getegid(void );

 当前进程使用用户组ID与当前进程有效用户组ID的区别：当前进程使用用户组是登录的账户，当前进程有效用户组ID是当前对文件访问的是哪个用户。

#### 进程共享

父子进程相同之处：全局变量，.data，.text, 栈，堆，环境变量，用户ID，宿主目录，进程工作目录，信号处理方式....

父子进程不同之处：1. 进程ID 2. fork返回值 3.父进程ID 4，进程运行时间 5.闹钟（定时器） 6.未决信号集

 注意，父子进程之间，全局变量之间是独享的。

```
#include<stdio.h>
#include<unistd.h>
#include<stdlib.h>
int var = 34;
int main(  )  {
	pid_t pid;
	pid = fork(  );
	if(pid == -1){
	perror("fork");
	exit(1);  
	}
	else if(pid > 0){
		sleep(3);  
		var = 55;
		printf("var = %d\n",var);  
	}
	else if(pid == 0){
		  var  = 100;
		  printf("var = %d\n",var);  
	}
	return 0;
}
```

这里第一次，var == 100 ;然后是 55.

**注意：**

1. 父子进程之间，遵循读时共享，写时复制的原则。

2. 父子进程可以共享文件描述符  （注意和进程控制块task_struct的区别）

3. mmp建立的映射区
4. fork之后父进程先执行还是子进程先执行不确定，取决于内核所使用的的调度算法。

#### gdb调试

使用gdb调试时，gdb只能跟踪一个进程。可以**在fork函数之前**，设置gdb是跟踪父进程还是子进程。

 set follow-fork-mode child  / set follow-fork-mode parent   命令设置跟踪子进程还是父进程。

