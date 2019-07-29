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

start_routine：指向线程的主控函数

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

