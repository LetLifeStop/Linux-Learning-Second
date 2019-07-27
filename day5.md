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