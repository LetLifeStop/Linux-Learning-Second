## 线程

linux线程和windows线程是完全不同的

LWP： light weight process 轻量级的进程，本质仍未进程（Linux环境下）

进程：独立地址空间，拥有PCB

**区别：**

在于是否共享地址空间    独居（进程）；合租（线程）

Linux下：

线程：调度的基本单位 

进程：资源分配的基本单位，可以看成是只有一个线程的进程

**CPU按照进程个数分配资源**



**Linux内核线程实现原理**

1. 轻量级进程，也有PCB，创建线程使用的底层函数和进程一样，都是clone
2. 从内核里看进程和线程是一样的，都具有各自的PCB。但是线程的PCB中指向内存资源的三级页表是相同的，所以多个线程共享地址空间，部分相同。

​    三级页表： PCB中的指针 - 》 页目录 - 》 页表  - 》物理内存

3. 线程可以看做寄存器和栈的集合。

​       线程的栈空间是用来分配函数的空间。每一个函数占用栈空间的一块区域，然后这块区域会保存一个栈帧，这个栈帧是用来存放局部变量和临时值（存放当前区域的栈顶和栈底）

4. 进程可以蜕变成线程
5. 在linux下，线程是最小的执行单位；进程是最小的分配资源单位。

查看LWP号（线程号（CPU分配资源的根据），并不是线程ID（在进程内部，区分线程））：ps -Lf pid (pid为进程号，当前进程下有哪些线程)

  线程号和线程ID的区别:线程号是CPU分配时间轮片的依据，线程ID是进程 区分 线程的依据

mmu的工作过程，首先从PCB的三级页表指针逐步往下，找到页表，页表再去对应物理内存



### 线程共享资源

1. 文件描述符表
2. 每种信号的处理方式
3. 当前工作目录
4. 用户ID 和 组ID
5. 内存地址了空间(.text段 data段 .bss段 heap段 共享库 ，（没有栈）)

### 线程非共享资源

1. 线程id
2. 处理器线程和栈指针
3. 独立的栈空间
4. errno变量
5. 信号屏蔽字
6. 调度优先级

### 线程优 ,缺点

优点：

1. 程序的并发性提高
2. 开销小（针对多进程来说，开销小）
3. 数据通信，共享数据方便

缺点：

1. 库函数，不稳定
2. 调试，编写困难，gdb不支持
3. 对信号不好



### 线程控制原语

**pthread_self 函数**

获取线程ID ，对应于进程的getpid函数

编译的时候加 -pthread 

**pthread_creat 函数**

创建一个新进程

int pthread_create(pthread_t *thread , const pthread_attr_t *attr ,void *(*start_routine)(void *) , void *arg);

参数： 

*thread :新建线程的ID

attr : 线程属性

start_routine：指向线程的主控函数，子线程创建成功，该函数自动调用

arg：参数 

```c
/*************************************************************************
	> File Name: thread.c
	> Author: 
	> Mail: 
	> Created Time: 2019年07月29日 星期一 20时49分12秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<pthread.h>

void *cal(void *tmp){
    printf("thread ID = %lu ,PID = %d\n",pthread_self() , getpid());
    return NULL;
}
int main(){
     
      printf("thread ID = %lu , PID = %d\n",pthread_self() , getpid());
     pthread_t  ret ;
    ret = pthread_create( &ret , NULL , cal , NULL );
    if(ret < 0){
        perror("pthread creat error");
        exit(1);
    }
    sleep(1);
      printf("thread ID = %lu , PID = %d\n",pthread_self() , getpid());
    return 0; // 将当前进程退出
}
```

**进程号还是不变的**，但是注意先让父进程睡1s，否则当父进程结束的话，子进程还没来得及运行就over了。

编译方法：  gcc thread.c -pthread -o thread 



**strerror函数**

 把errnum自动转换成对应的字符串



**循环创建多个线程**

```c
/*************************************************************************
	> File Name: thread.c
	> Author: 
	> Mail: 
	> Created Time: 2019年07月29日 星期一 20时49分12秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<pthread.h>

void *cal(void *tmp){
    int i = (int)tmp;
    sleep(i);
    printf("%dth , thread ID = %lu ,PID = %d\n",i, pthread_self() , getpid());

    return NULL;
}
int main(){
   //   printf("thread ID = %lu , PID = %d\n",pthread_self() , getpid());
    int i ;
    for ( i = 0 ; i < 5 ;i++ ){
    pthread_t  ret ;
    ret = pthread_create( &ret , NULL , cal , (void * )i);
    if(ret < 0){
        perror("pthread creat error");
        exit(1);
    }
    }
    sleep(i);  // 防止父进程提前结束
    //  printf("thread ID = %lu , PID = %d\n",pthread_self() , getpid());
    return 0;
}
```

### 线程与共享

线程之间共享全局变量

练习：编写程序，验证线程之间共享全局变量

**ps：**

为什么要在for循环的下面sleep一秒？ 因为主线程的for循环是比子线程的创建快的多，如果不sleep1s的话，子线程还没来得及创建，flag就被更改了！！！。

```c
/*************************************************************************
	> File Name: judgethread.c
	> Author: 
	> Mail: 
	> Created Time: 2019年07月30日 星期二 09时25分33秒
 ************************************************************************/

#include<stdio.h>
#include<unistd.h>
#include<stdlib.h>
#include<pthread.h>
int flag ;
void *cal(void *tmp){
   int i = (int)tmp;
  //  sleep(i);
    printf(" %d th ,thread ID = %ld , flag = %d\n", i ,pthread_self() , flag);
    return NULL ;
}
int main(){
     int i;
    pthread_t pthread;
    for ( i = 0 ; i < 5; i++ ){
      sleep(1);
     flag = i ;
        int ret;
    ret  = pthread_create( &pthread , NULL , cal , (void *)i);

        if( ret < 0 ){
            perror("creat thread error");
            exit(1);
        }
    }
    sleep(10);
    return 0;
}
```

**exit 是将进程退出， return是返回到调用者的地方，编写多线程的时候，谨慎使用return和exit**

return是将整个进程退出，ex

#### pthread_exit 函数

将单个线程退出

void pthread_exit( void *retval);

无返回值

#### pthread_join 函数

阻塞等待线程的退出，获取线程退出状态，对应进程的waitpid函数，一次只能回收一个

int pthread_join(pthread_t thread , void **retval)

参数：

thread ，线程ID

retval ，存储线程结束状态

```c
/*************************************************************************
	> File Name: pthread_creat.c
	> Author: 
	> Mail: 
	> Created Time: 2019年07月30日 星期二 15时30分47秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<pthread.h>


void *thrd_fuc(void *arg){
    pthread_exit((void *)100);
    // 也可以返回结构体
}
int main(){
  pthread_t tid;
    int ret ;
    int *retval ;
    printf("In main 1: thread id = %lu ,pid = %u\n",pthread_self(),getpid());
    ret = pthread_create( &tid ,NULL , thrd_fuc ,NULL );
    if( ret != 0 ){
       perror("thread fail");
        exit(1);
    }
    pthread_join( tid ,(void **)&retval );
    printf("-----%d\n",(int)retval);
    pthread_exit((void *)1);
}
```

结构体

```c
/*************************************************************************
	> File Name: pthread_creat.c
	> Author: 
	> Mail: 
	> Created Time: 2019年07月30日 星期二 15时30分47秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<pthread.h>
#include<string.h>

typedef struct {
    int num;
    char ch;
    char str[100];
}exit_t;

void *thrd_fuc(void *arg){
    exit_t *ret = (exit_t *)malloc(sizeof(exit_t));
    ret->ch = 'm';
    ret->num = 200;
    strcpy(ret->str , "1111dwadaw\n");
    pthread_exit((void *)ret);
}
int main(){
  pthread_t tid;
    int ret ;
    exit_t *retval ;
    printf("In main 1: thread id = %lu ,pid = %u\n",pthread_self(),getpid());
    ret = pthread_create( &tid ,NULL , thrd_fuc ,NULL );
    if( ret != 0 ){
       perror("thread fail");
        exit(1);
    }
    pthread_join( tid ,(void **)&retval );
    printf(" ch = %c ,num = %d , str = %s \n",retval->ch ,retval->num ,retval->str);
    free(retval); // 回收
    pthread_exit((void *)1);
}
```

回收多个子进程

```c
/*************************************************************************
	> File Name: backthread.c
	> Author: 
	> Mail: 
	> Created Time: 2019年07月30日 星期二 15时56分15秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<pthread.h>

int var = 100;
void *func( void *arg ){
int i = (int)arg;
  sleep(i);
    if(i == 1){
        var = 333;
        printf(" car = %d\n",var);
        return var;
    }
    else if(i == 3){
        var = 777;
        printf("I an %d th pthread ,pthread ID  =%d , var = %d\n" , i ,pthread_self() ,var );
        pthread_exit((void *)var);
    }
    else {
       printf("I am %dth pthread ,pthread ID = %d , var = %d\n",i , pthread_self(), var); 
    }
    return NULL;
}
int main(){
    pthread_t tid[5];
    int i;
    int *ret[5];
    int retval;
    for( i = 0 ; i < 5;i++ ){
    retval = pthread_create(&tid[i] , NULL ,func ,(void *)i );
        if(retval < 0){
           perror("creat error");
            exit(1);
        }
    }
    for( i = 0 ; i< 5;i++ ){
        pthread_join(tid[i] , (void **)&ret[i]);
        printf("---%d th , ret = %d\n",i, (int)ret[i]);
    }
    printf("I am main pthread tid = %lu \n",pthread_self());
    sleep(i);
    return 0;
}
```

**进程之间，由父进程回收子进程；但是线程之间，是互相并行的，线程之间可以互相回收。**



### pthread_detach函数

实现线程分离

int pthread_detach( pthread_t thread); 成功，0；失败，错误号

从状态上，线程与主控线程断开关系。线程结束后，其退出状态不由其他线程获取，而是自己自动释放。

进程如果有该机制，将不会产生僵尸进程。

```c
/*************************************************************************
	> File Name: pthread_detach.c
	> Author: 
	> Mail: 
	> Created Time: 2019年07月30日 星期二 16时42分26秒
 ************************************************************************/

#include<stdio.h>
#include<string.h>
#include<pthread.h>
#include<unistd.h>

void *cal(void *arg){
    int n = 3;
    while(n--){
printf("thread count %d\n",n);
        sleep(1);
    }
    return (void *)2;
}
int main(){
    pthread_t tid;
    void *tret;
    int err;
    pthread_create( &tid ,NULL , cal , NULL );
    
    pthread_detach(tid);

    while(1){
        err = pthread_join(tid ,&tret);
        printf("-----------err = %d\n",err);
        if(err != 0){
            fprintf(stderr , "thread %s\n",strerror(err));
        }
        else fprintf(stderr , "thread exit code%d\n", (int)tret);
        sleep(1);
    }
    return 0;
}
```

如果我们把pthread_detach回收的话，就会发现先thread_count输出完再往下走。

这是因为：**pthread_join 和wait类似，是阻塞等待子线程结束**





### 杀死线程

int pthread_cancel(pthread_t thread) ;

**注意：线程的取消并不是实时的， 而是由一定的延时，需要等到线程到达某个取消点。**

取消点：

1. 执行creat ， open， pause，close等等（man 7 pthreads）

2. 执行系统调用
3. 当没有系统调用的时候，添加pthread_testcancel() ；自己添加取消点

```c
/*************************************************************************
	> File Name: thread_cancel.c
	> Author: 
	> Mail: 
	> Created Time: 2019年07月30日 星期二 17时47分55秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<pthread.h>

void *cal(void *t){
    int i = (int)t;
    while(1)
    { 
printf("dawdad\n");
        sleep(1);
        pthread_testcancel(); // 如果把while里面的，全部删除，就会卡死，无法删掉
    }
   return (void *)666;
}
int main(){
    pthread_t pthread;
    int ret ,ret1 ,ret2;
    ret = pthread_create( &pthread , NULL , cal , NULL );
    if(ret != 0){
        perror("creat error");
        exit(1);
    }

    sleep(3);
     ret1 = pthread_cancel(pthread);
    ret2 = pthread_join(pthread ,(void **)&ret); // 回收一个杀死的进程，会返回 -1 
    printf("%d\n",(int)ret);  // printf("%d\n" , ret);
    sleep(2);
    return 0;
}
```

#### pthread_equal函数

比较两个线程ID是否相等

#### 控制原语的对比

fork                    pthread_creat

eixt(int)              pthread_exit(void *)

wait(int *)          pthread_join(,void **)  // 注意对比

kill                       pthread_cancel()

getpid                pthread_self()

​                            pthread_detach()

#### 线程属性

 linux下的线程属性是可以根据实际项目需要，进行设置。如果我们要求对程序更高的要求，我们可以通过设置线程栈的大小来降低内存的使用，增加最大线程的个数

typedef struct{

int etachstate;



}pthread_attr_t;

**重点：**

1. 线程分离状态
2. 线程栈大小（默认平均分配）（**默认进程的栈是8102KB == 8M）
3. 线程栈警戒缓冲区大小（位于栈末尾）

##### 线程属性初始化

 注意：应该先初始化线程属性，再pthread_create 创建线程

**初始化线程属性**

Int pthread_attr_init( pthread_attr_t *attr);

**销毁线程属性所占用的资源**

int pthread_attr_destory(pthread_attr_t *attr);

**线程的分离状态**

非分离状态 && 分离状态

非分离状态：等待主线程的结束，当调用pthread_join() 函数返回的时候，创建的线程才算结束，才能释放自己所占的系统资源

分离状态：自己运行结束了，线程也就终止，马上释放资源

**函数：**

设置线程属性  分离 && 非分离

Int pthread_attr_setdetachstate( pthread_attr_t *attr , int detachstate );

获取线程属性 分离 && 非分离

int pthread_attr_getdetachstate(pthread_attr_t *attr ,int *detachstate);

   参数：

attr：

已经初始化的线程属性

分离：

PTHERATE_CREATE_DETACHED

非分离：

PTHERATE_CREATE_JOINABLE

```c
/*************************************************************************
	> File Name: thread_init.c
	> Author: 
	> Mail: 
	> Created Time: 2019年07月30日 星期二 20时21分17秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<pthread.h>
#include<string.h>

void *cal(void *t){
    pthread_exit((void *)77);
}
int main(){
pthread_t tid;
int ret ;
    pthread_attr_t attr;
    ret = pthread_attr_init(&attr);
    if(ret != 0 ){
    fprintf(stderr , "pthread_init error:%s\n",strerror(ret));
        exit(1);
    }
    pthread_attr_setdetachstate( &attr,PTHREAD_CREATE_DETACHED );
    ret = pthread_create( &tid ,&attr ,cal , NULL );
    if(ret != 0){
        fprintf(stderr ,"pthread_create error:%s\n",strerror(ret));
        exit(1);
    }
    ret = pthread_join(tid , NULL);
   // if(ret != 0){
   // perror("join error");
   //     exit(1);
   // }
    printf("--------------%d\n",ret);
    pthread_exit((void*)1);
}
```



注意：

如果设置一个线程为分离线程，有可能在create返回之前就终止了，终止以后就有可能将线程号和系统资源移交给其他的线程使用，这样调用pthread_create就有可能得到错误的信号。

解决方法：

在被创建的线程中调用pthread_cond_timewait函数，让这个线程等一会，留出足够的时间让pthread_create返回



#### 线程的栈地址

##### 设置线程的栈的地址

pthread_attr_setstack(pthread_attr_t *attr , void *stackaddr , size_t stacksize);

**获取线程的栈的地址**

pthread_attr_getstack(pthread_attr_t *attr , void *stackaddr , size_t stacksize);



#### 线程的栈大小

**设置现成的栈的大小**

pthread_attr_setstacksize(pthread_attr_t *attr  , size_t stacksize);

**获取线程的栈的大小**

pthread_attr_getstack(pthread_attr_t *attr , size_t *tacksize);



练习：

查看当前计算机中的一个进程最多能创建多少个线程

```c
/*************************************************************************
	> File Name: cont.c
	> Author: 
	> Mail: 
	> Created Time: 2019年07月30日 星期二 20时53分28秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<pthread.h>

void *cal(void *t){
    while(1){sleep(1);} // 防止线程结束，释放栈空间
}
int main(){
    pthread_t ptd;
    int ret;
    int cnt  = 0;
    while(1){
     ret = pthread_create(&ptd ,NULL , cal , NULL);
        if(ret != 0){
            break;
        }
        printf("-----------%d\n", ++cnt);
    }
    return 0;
}
```

调整线程栈空间的大小：

```c
/*************************************************************************
	> File Name: cont.c
	> Author: 
	> Mail: 
	> Created Time: 2019年07月30日 星期二 20时53分28秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<pthread.h>
#include<string.h>

const int SIZE = 0x10000;

void *cal(void *args){ 
    while(1){sleep(1);}
   // pthread_exit((void *)1);
}
int main(void){
    pthread_attr_t  attr;
    pthread_t ptd; 
    int ret ,detach ,cnt= 0 ;
    size_t stacksize ;
    void *stackaddr ;
    ret = pthread_attr_init(&attr);
    if(ret != 0){
        perror("init error");
        exit(1);
    }
    pthread_attr_getstack(&attr ,&stackaddr, &stacksize);
     ret = pthread_attr_getdetachstate( &attr ,&detach );
    if(ret != 0){
        perror("get detachstate error");
        exit(1);
    }
    if(detach ==PTHREAD_CREATE_DETACHED){
        puts("thread detached");
    }
    if(detach == PTHREAD_CREATE_JOINABLE){
        puts("thread joined");
    } 
     pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
  // size_t stacksize;
    while(1){
        stackaddr = malloc(SIZE);
        if(stackaddr == NULL){
            perror("malloc error");
            exit(1);
        }
      //  stacksize = SIZE;
        pthread_attr_setstack( &attr ,stackaddr ,SIZE);
        int rret = pthread_create(&ptd , &attr ,cal , NULL);
        if(rret != 0){
        printf("cnt = %d\n",cnt);
            printf("````%s\n",strerror(rret));
        exit(1);
        }
        cnt++;
    }
   pthread_attr_destroy(&attr);
return 0;
}
```

```c
#include <stdio.h>
#include <pthread.h>
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
 
#define SIZE 0x10000
 
void *th_fun(void *arg)
{
	while (1) 
		sleep(1);
}
 
int main(void)
{
	pthread_t tid;
	int err, detachstate, i = 1;
	pthread_attr_t attr;
	size_t stacksize;   //typedef  size_t  unsigned int 
	void *stackaddr;
 
	pthread_attr_init(&attr);		
	pthread_attr_getstack(&attr, &stackaddr, &stacksize);
	pthread_attr_getdetachstate(&attr, &detachstate);
 
	if (detachstate == PTHREAD_CREATE_DETACHED)   //默认是分离态
		printf("thread detached\n");
	else if (detachstate == PTHREAD_CREATE_JOINABLE) //默认时非分离
		printf("thread join\n");
	else
		printf("thread un known\n");
 
	/* 设置线程分离属性 */
	pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
 
	while (1) {
		/* 在堆上申请内存,指定线程栈的起始地址和大小 */
		stackaddr = malloc(SIZE);
		if (stackaddr == NULL) {
			perror("malloc");
			exit(1);
		}
		stacksize = SIZE;
	 	pthread_attr_setstack(&attr, stackaddr, stacksize);   //借助线程的属性,修改线程栈空间大小
 
		err = pthread_create(&tid, &attr, th_fun, NULL);
		if (err != 0) {
			printf("%s\n", strerror(err));
			exit(1);
		}
		printf("%d\n", i++);
	}
 
	pthread_attr_destroy(&attr);
 
	return 0;
}

```



### NPTL

1. 查看线程库版本：

2. NOTL实现机制（POSIX）
3. 使用线程库时， gcc指定 -lpthread



### 线程使用注意事项

1. 主线程退出，其他线程不退出，主线程应该调用pthread_exit

2. 避免僵尸进程 1. pthread_join 2. pthread_detach 3. pthread_detach 指定分离属性

3. malloc和mmap申请的内存可被其他线程释放

4. 避免在多线程模型中调动fork ，如果调用，马上exec。子进程中只有调用fork的线程存在，

   其他线程都没有了。

5. 避免在多线程中引入信号机制



##### 多线程拷贝作业

 ```c
#include<stdio.h>
#include<sys/stat.h>
#include<fcntl.h>
#include<unistd.h>
#include<sys/types.h>
#include<sys/wait.h>
#include<stdlib.h>
#include<sys/mman.h>
#include<string.h>
#include<pthread.h>

void sys_err(char *str){
    perror(str);
    exit(-1);
}
int min(int t1 ,int t2  )  {
    if(t1 > t2)return t2;
    return t1;
}
int sto[10];
void  init(int num , int len){ // 分配任务
    int tmp  = len / 5 ;
    while(len > 0 ){
        int t = min(tmp , len);
        sto[ num++ ] = t;
        len -= t;
        if(len<tmp && len != 0 ){
            sto[ num - 1] += len;
            return ;
        }
    }
}
struct sto_wr{
   char *r;
   char *w;
    int id;
};

int len;
void *cal(void *arg){
    struct sto_wr* tmp =(struct sto_wr*) arg;
    printf("%d\n",len);
        memcpy( tmp->w + (tmp->id) * ( len / 5 ), tmp->r + (tmp->id) * ( len / 5 ),sto[tmp->id]);
        //	 printf("%d %d \n",i * 12 , sto[i]);
        printf("%d th sto , from %d to %d byte, %d stored\n", tmp->id, tmp->id * (len/5) ,tmp->id * (len/5) + sto[ tmp->id ] , sto[ tmp->id ] );
    return NULL;
}
int main(int argc ,char *argv[]){
    int fd ;
    struct sto_wr sto[6];

    if(argc < 3) {
       perror("error");
         exit(1);
      }
    struct stat info;     
    // 建立读取文件的映射区
    fd = open (argv[1],O_CREAT|O_RDWR,0777) ; 
    if( fd < 0){
        perror("open file");
        exit(1);
    }
    int tmp ;
    fstat(fd , &info);
    len = info.st_size;
     printf("%d\n" , len);  
    char *p_r = NULL;
    p_r =(char *)mmap(NULL , len , PROT_READ|PROT_WRITE, MAP_SHARED ,fd ,0);  // 建立映射
    if( p_r == MAP_FAILED){
        perror("mmap1 error");
        exit(1);
    }
   char *r = p_r;
    close(fd);
    printf("%d\n",len);
    int fd1;
    // 建立编写文件的映射区
    char *p_w = NULL ;
    fd1 = open (argv[2] , O_RDWR|O_CREAT|O_TRUNC, 0644);

    if(fd1 < 0 ){
        perror("open 2 error");
        exit(1);
    } 
     tmp  = ftruncate(fd1 ,len );
    if(tmp == -1){
        perror("  ftruncate error");
        exit(1);
    }
    p_w =(char *)mmap(NULL , len , PROT_WRITE|PROT_READ, MAP_SHARED , fd1 ,0);
    if(p_w == MAP_FAILED){
        perror("mmap 2 error");
        exit(1);
    }
   char  *w =p_w;
    close(fd1);
    init(0 , len);
    //for(int i = 0 ; i< 5;i++){
     //   printf("  %d %d\n",i,sto[i]);
   // }
    int i;
    int pid ; 
    int cnt = 0;
    pthread_t ptd[5];
 //   pthread_t *ptd = (pthread_t*)malloc(sizeof(pthread_t)*5);
    for (i = 0 ; i< 5; i++) {
      //  sleep(i);
      sto[i].r = r ;
      sto[i].w = w ;
      sto[i].id =i;  // 储存起来使用，直接点用函数即可。
        pid = pthread_create( &ptd[i] , NULL ,cal , (void *)&sto[i]);
      //  printf("%d th , pid == %d\n",i, pid);
        if(pid != 0){
            perror("pthread error");
            exit(1);
        }
    }
    sleep(10);
    return 0;
}
 ```

