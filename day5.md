### 竟态条件

产生原因：当两个线程竞争同一个资源的时候，如果对资源的访问顺序敏感，就存在时序竟态条件

#### pause函数

   作用：

​    调用该函数会造成该进程主动挂起，等待信号唤醒（收到该进程任意一个信号后终止，处理完信号后，继续往下走，**为了防止收到程序运行终止的信号而使得下面的程序无法继续时，我们可以自定义信号处理函数**）。在信号唤醒没有发生之前，调用该系统调用的进程将处于阻塞状态（主动放弃CPU）

  

  返回值：

  当信号被捕捉，并且信号的捕捉函数返回的时候，pause才会return  -1 .

练习：

使用pause和alarm来实现sleep函数 

```c
/*************************************************************************
    > File Name: com.c
    > Author: ma6174
    > Mail: ma6174@163.com 
    > Created Time: 2019年07月26日 星期五 10时08分19秒
 ************************************************************************/

#include<stdio.h>
#include<unistd.h>
#include<stdlib.h>
#include<sys/time.h>
#include<signal.h>
#include<errno.h>

void cal( int tmp )  {
	printf("catch %d signal right \n " ,tmp );
    return ;
}
void my_sleep( int sec  )  {

struct sigaction now , old ;
now.sa_handler = cal;
now.sa_flags = 0 ;
sigemptyset(&now.sa_mask);  

int ret ;
ret = sigaction (SIGALRM ,&now ,&old); // 对信号执行函数进行自定义
if( ret == -1){
perror("sigaction error") ;
exit(1);
}

alarm(sec);
ret = pause(  ) ; // 收到信号后恢复
if( ret == -1 && errno == EINTR) {
printf("pause right");
}

ret = alarm(0); // 对alarm进行清零
sigaction(SIGALRM , &old ,NULL); // 返回SIGALRM原有的默认执行动作，防止出错
return ;
}
int main(  )  {

while( 1 ) {
	  my_sleep(3); // 自定义的函数
	  printf("  hqx  nb \n") ;
}
return 0;
}
```

**注意：**

在一次调用alarm和pause之间会有一个竞争条件。在一个繁忙的系统当中，可能会发生alarm在调用pause之前就已经超时（alarm设置了 2 s 后运行程序，但是别的优先级更高的模块抢占了CPU，并且这个模块执行的时间很长，导致当alarm都已经完事了，这个模块还没有执行完）。这样就会导致在调用pause时，一直捕捉不到信号。如果没有捕捉到其他信号，调用者将被永久的挂起。

ps：这里的alarm到2s的时候，会被自己的docatch处理掉（还没接触到，等接触到再填坑）。



**alarm和sleep不要混用，调用的都是SIGALARM信号，测试这个样例会搞出问题的。**

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
#include<errno.h>

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
//	sigaddset( &act.sa_mask , SIGQUIT);	
	act.sa_flags = 0;
         
    ret = sigaction(SIGALRM , &act ,NULL);

	if( ret < 0 ){
		perror("sigaction error");
		exit(1);  
	}
	alarm(2);
    // 此处被别的CPU抢占了资源
   int tmp = pause(  );
   if(tmp == -1 && errno == EINTR ){
	     printf("right");
   }
//	 while(1);
	return 0;
}
```



解决方法：

我们可以将信号的接触屏蔽和pause绑起来。

int  sigsuspend(const sigset_t *mask);

功能：

为了防止 “解除信号屏蔽” 和 “挂起等待信号” 这两个操作之间失去CPU ，我们将接触信号屏蔽与挂起等待信号绑在一起就可以解决这个问题了。

参数：

mask是一个表，代表从解除信号屏蔽开始，到接收到信号这短时间遵循的阻塞信号集。调用完毕之后立即恢复到原来的阻塞信号集。

具体使用方法：

1. 设置信号执行函数
2. 屏蔽信号SIGALRM,设置堵塞
3. 设置闹钟
4. 调用sigsuspend函数，注意参数应该解屏蔽SIGALRM信号
5. 收到信号后执行操作
6. 恢复原样

```c
/*************************************************************************
    > File Name: com.c
    > Author: ma6174
    > Mail: ma6174@163.com 
    > Created Time: 2019年07月26日 星期五 10时08分19秒
 ************************************************************************/

#include<stdio.h>
#include<unistd.h>
#include<stdlib.h>
#include<sys/time.h>
#include<signal.h>
#include<errno.h>

void cal( int tmp )  {
	printf("catch %d signal right \n " ,tmp );
    return ;
}
void my_sleep( int sec  )  {

sigset_t newmask ,oldmask ,susmask;
struct sigaction now , old ;

now.sa_handler = cal;
now.sa_flags = 0 ;
sigemptyset(&now.sa_mask); 
sigaction(SIGALRM , &now ,&old)
sigemptyset(&newmask);
sigaddset(&newmask , SIGALRM);
sigprocmask(SIG_BLOCK , &newmask , &oldmask);  

alarm(sec);

susmask =  oldmask;
sigdelset(&susmask , SIGALRM);

sigsuspend(&susmask);

alarm(0);
sigaction(SIGALRM , &old ,NULL);

return ;
}
int main(  )  {

while( 1 ) {
	  my_sleep(3);
	  printf("  hqx  nb \n") ;
}
return 0;
}
```

可重入函数 && 不可重入函数

 一个函数在被调用期间（尚未调用结束），由于某种时序又被重复调用。称为重入。

 注意事项：

1. 可重入函数，函数内不能含有**全局变量**及**static变量**，不能使用**malloc，free** 。

2. 信号捕捉函数为可重入函数

3. 信号处理程序判断哪些函数时可重入函数和 不可重入函数man 7 signal

4. 不可重入的原因：

   1. 使用静态数据结构
   2. 调用了malloc 或者 free
   3. 是标准I/O函数

   

#### SIGCHLD信号

信号的产生条件：

  当子进程状态发生变化

练习：借助SIGCHLD信号回收子进程

有10个子进程，子进程结束的时候回发送SIGCHLD信号给父进程，然后父进程调用处理函数。

```c
/*************************************************************************
	> File Name: 27.c
	> Author: 
	> Mail: 
	> Created Time: 2019年07月27日 星期六 20时58分45秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<errno.h>
#include<sys/types.h>
#include<sys/wait.h>
#include<signal.h>

void sig_child(int signo){
    int status;
    pid_t pid;
    // 体会一下while 和 if 的区别
    //  信号不支持排队，所以我们可以通过信号来回收子进程
    while((pid = waitpid(0,&status,WNOHANG)) > 0){
        if(WIFEXITED(status))
        printf("----------child %d exit %d\n",pid ,WEXITSTATUS(status) );
        else if(WIFSIGNALED(status)){
            printf("child %d cancel signal %d \n",pid , WTERMSIG(status));
        }
    }
}
int main(){
    pid_t pid;
    int i;
    for( i = 0 ;i < 10 ; i++ ){
    pid = fork();
        if( pid < 0  ){
            perror("fork error");
            exit(1);
        }
        else if(pid  == 0){
            break;
        }
    }
    if( pid == 0 ){
        int n = 1;
        while( n-- ){
            printf("child ID %d\n",getpid());
            sleep(1);
        }
        return i+1;
    }
    else if( pid > 0){
        
        struct sigaction act;
        act.sa_handler = sig_child;
        sigemptyset(&act.sa_mask);
        act.sa_flags = 0;
        sigaction(SIGCHLD ,&act ,NULL);
        
       while(1){
        printf("Parent ID %d\n",getpid());
        sleep(1);
    }
}
return 0;
}
```





### 信号传参

> 发送信号参数

sigqeue 函数对应kill函数，但是可以向指定进程发送信号的携带参数

int sigqueue( pid_t pid , int sig ,const union sigval value);

参数：

pid         指定进程的ID

sig         信号编号 

sigval    要传出的信息

> 捕捉函数参数

int sigaction( int signum , const struct sigaction *act , struct sigaction *oldac);

struct sigaction{

void (*sa_handler)(int);

void (* sa_sigaction )(int ,siginfo_t  *, void * );

sigset_t sa_mask;

int sa_flags;

void (*sa_restorer)(void);

};

当注册信号捕捉函数，希望获取更多信号相关信息，应该使用sigaction  ,但是这个时候sa_flags 必须指定为SA_SIGINFO。 



##### 终端系统调用

系统调用分为两类：慢速系统调用和其他系统调用

1. 慢速系统调用：可能会是进程永远阻塞的一类 。pause , wait , waitpid , read(从网络读取，从设备读取，从管道等等，可能会出现阻塞)
2. 其他系统调用： 一调用就结束

慢速系统调用被中断的相关行为，实际上就是pause的行为：如read 

1. 想中断pause ，信号不能被屏蔽
2. 信号的处理方式必须是捕捉（默认 ， 忽略都不可以）
3. 中断后返回 -1 ，设置errno为EINTR（表示信号被中断）

可以通过修改sa_flags 参数来设置被信号中断之后系统调用是否重新启动。在执行捕捉函数的时候，不希望自动忽略该信号，同样可以通过修改sa_flags来实现。



### 终端

所有输入，输出设备的总称

终端设备模块

终端设备 - 》 终端设备驱动 -》 **线路规程过滤（内核）**-》系统调用的实现

**ttyname 函数**

当前文件描述符对应的文件名

网络终端的构造

**每一个按键都要在网络上走一遭！！**



#### 进程组

进程组，也称之为作业。 

进程组操作函数

getpgrp函数：

  获取当前进程组ID

getpid函数：

获取指定进程的进程组ID

**setpgid函数：**

  int setpid( pid_t pid ,pit_t pgid); 将参数1对应的进程，加入到参数2对应的进程组中

1. 如果改变子进程为新的进程，应该在fork之后，exec之前
2. 权级问题。非root进程只能改变自己创建的子进程，或有权限操作的进程

PID:   进程ID

PGID：进程组ID



改变进程默认所属的进程

```c
/*************************************************************************

> File Name: setpgif.c
> Author: 
> Mail: 
> Created Time: 2019年07月28日 星期日 17时10分55秒
>  ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<sys/types.h>

int main(){
    pid_t pid;
    pid = fork();
    if(pid < 0){
        perror("fork error");
        exit(1);
    }
    else if(pid == 0 ){
        printf("child PID == %d\n",getpid());
        printf("child group ID == %d\n",getpgid(0)); // 子进程的pid，子进程的组id
        sleep(7);
        printf("child group ID change to ==%d\n",getpid());
        exit(0);
    }
    else if(pid > 0){
        sleep(1); // 先让子进程创建好
        setpgid(pid , pid); 
            sleep(13);
   printf("-------------------\n");
   printf("parent PID == %d\n",getpid());
   printf("parent's parent PID == %d\n",getppid());
   printf("parent Group ID == %d\n",getpgid(0));
  
   sleep(5);
   setpgid(getpid() , getppid());
    printf("\n---group ID of parent is changed to %d\n",getpgid(0));
  while(1); 
}
return 0;
}
```



### 会话

把一堆进程组进行凑组。

创建一个会话注意的条件：

1. 调用进程不能是进程组组长，否则这个进程会变成一个新的会话首领
2. 创建会话的进程，创建之后，变成组长和会长
3. 需要有root权限
4. 新会话丢弃原有的控制终端，该会话没有控制终端（只在后台执行）
5. 该调用是组长进程，则出错返回
6. 建立新会话时，先调用fork，父进程终止，子进程调用setsid

> getsid函数

获取进程所属的会话进程

> setsid函数

 设置会话

练习：

```c
/*************************************************************************
	> File Name: setsid.c
	> Author: 
	> Mail: 
	> Created Time: 2019年07月28日 星期日 17时42分11秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<sys/types.h>
#include<unistd.h>

int main(){
    pid_t pid;
    pid = fork();
    if(pid < 0){
        perror("fork error");
        exit(1);
    }
    else if(pid > 0){
        printf("parent process  ID = %d\n",getpid());
        printf("parent group ID = %d\n",getpgid(0));
        printf("parent session ID = %d\n" ,getsid(0));
    }
    else if(pid == 0){
        printf("child process ID = %d\n",getpid());
      //  printf("child's parent process ID =%d\n",getppid());
        printf("child's group ID = %d\n",getpgid(0));
        printf("child session ID = %d\n",getsid(0)) ;
    sleep(5);
        setsid();

    printf("--------\n\n");

    printf("child group process  ID = %d\n",getpgid(0));
    printf("child's process ID = %d\n",getpid());
    printf("child session ID = %d\n",getsid(0));
}
return 0;
}        
```



### 守护进程

也叫Daemon（精灵）进程，是linux的后台服务进程（没有控制终端，不能直接和用户交互，不受用户登录，注销的影响），独立于控制终端并且周期性执行某种任务或者等待某些时间的发生，一般采用d结尾的名字。

**创建守护进程，最关键的异步就是调用setsid函数创建一个新的session（会话），并且成为会长。**

**创建守护进程模型**

1. 创建子进程，父进程退出
2. 在子进程中创建新的对话（同理setsid的创建过程）

3. 改变当前目录为根目录（也可以变成其他目录，目的就是为了防止占用可卸载的文件系统）

4. 重设文件权限掩码 （umask（）函数），防止继承的文件创建屏蔽字拒绝某些权限。增加守护进程的灵活性

5. 关闭文件描述符 （将0 ，1， 2 重定向 /dev/null）

   继承的打开文件不会用到，浪费系统资源，无法卸载

6. 开始执行守护进程核心工作

7. 退出

基本架构：

```c
/*************************************************************************
	> File Name: fly.c
	> Author: 
	> Mail: 
	> Created Time: 2019年07月28日 星期日 20时37分28秒
 ************************************************************************/

#include<stdio.h>
#include<unistd.h>
#include<stdlib.h>
#include<sys/types.h>
#include<sys/stat.h>
#include<fcntl.h>

int main(){
   pid_t pid;
    pid = fork();
    if(pid < 0 ){
        perror("fork error");
        exit(1);
    }
    else if(pid > 0 ){
        exit(0);
    }
    else {
        setsid();
        int ret = chdir("/huangiqngxiang/home/itcast/");
        if(ret == -1){
            perror("chdir error");
            exit(1);
        }
        umask(0022); 
        // chmod 是文件权限码，0644 。而umask是文件权限码的补码
        close(STDIN_FILENO);
        open("/dev/null",O_RDWR);
        dup2( 0 ,STDOUT_FILENO ); // 把后面的重定向前面的
        dup2( 0 ,STDERR_FILENO );
    }
     while(1);
    return 0;
}
```



练习：

每隔一定的时间，把当前的时间打印，写到文件当中。

在上面的模板的基础上，加上setitimer函数控制时间