### 线程同步

同步的概念：

同时起步，协调一致

线程同步：

协同步调，按预定的先后次序运行。 一个线程在调用某一个功能时，在没有得到结果之前，该调用不返回。同时其他下线程为了保证数据的一致性，不能调用该功能。

同步的目的，是为了防止数据混乱， 解决与时间有关的错误。

**为了防止数据混乱，可以通过加锁的形式**



#### 互斥量 mutex

 每个线程在对资源进程操作之前都先尝试枷锁，成功加锁之后才能操作，操作结束解锁。资源还是共享的，并且线程还是竞争的，但是通过锁将资源的访问变成了互斥操作。

**ps：**

同一个时刻，只能有一个线程持有该锁。

当A线程对某个全局变量进行加锁访问的时候，B在访问前尝试加锁，拿不到的话就会阻塞等待。如果C线程不去加锁，而是直接访问该全局变量，依然能够访问，从而造成数据混乱。

所以，互斥锁可以看成一把**建议锁**。只有制定好正确的规则才能保证程序正常运行。



主要应用函数：

pthread_mutex_init 函数

pthread_mutex_destroy 函数

pthread_mutex_lock 函数

pthread_mutex_trylock 函数

pthread_mutex_unlock 函数



**pthread_mutex_init 函数**

初始化一个互斥锁，初值看成 1 

int pthread_mutex_init( pthread_mutex_t *restrict mutex , const pthread_mutexattr_t *restrict attr);

**参数一，restrict 关键字**  ，所有对改指针指向内存的修改，只能通过mutex来实现。不能通过该指针以外的其他变量或指针修改

**参数二**，互斥量属性

**pthread_mutex_lock 函数**

加锁，可理解为将mutex - 1 。

**pthread_mutex_unlock函数**

解锁，可理解为将mutex ++ 

**pthread_mutex_trylock 函数**

尝试加锁 ， 和lock的区别是，lock是阻塞加锁；trylock是不阻塞加锁

练习：

模拟两个线程之间冲突

```c
/*************************************************************************
	> File Name: mute.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月01日 星期四 14时39分25秒
 ************************************************************************/

#include<stdio.h>
#include<pthread.h>
#include<stdlib.h>
#include<unistd.h>

void *cal(void *arg){
    srand(time(NULL));
    while(1){
        printf("hello ");
        sleep(rand()%3);
        printf("world\n");
        sleep(rand()%3);

   }
    return NULL;
}
int main(void ){
pthread_t tid;
    srand(time(NULL));
    pthread_create(&tid , NULL ,cal ,NULL);
    while(1){
        printf("HELLO ");
        sleep(rand()%3);
        printf("WORLD\n");
        sleep(rand()%3);
    }
    return 0;
}
```



改正：

```c
/*************************************************************************
	> File Name: mute.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月01日 星期四 14时39分25秒
 ************************************************************************/

#include<stdio.h>
#include<pthread.h>
#include<stdlib.h>
#include<unistd.h>

pthread_mutex_t mutex;
void *cal(void *arg){
    srand(time(NULL));
    while(1){
        pthread_mutex_lock(&mutex); // 上锁
        printf("hello ");
        sleep(rand()%3);
        printf("world\n");
        pthread_mutex_lock(&mutex);
        sleep(rand()%3);

   }
    return NULL;
}
int main(void ){
pthread_t tid;
    srand(time(NULL));
    pthread_mutex_init(&mutex , NULL);
    pthread_create(&tid , NULL ,cal ,NULL);
    while(1){
        pthread_mutex_lock( &mutex);
        printf("HELLO ");
        sleep(rand()%3);
        printf("WORLD\n");
        pthread_mutex_unlock(&mutex);
        sleep(rand()%3);   
    }
    pthread_mutex_destroy(&mutex);
    return 0;
}
```

**结论：**

在访问共享资源前加锁，访问结束后立即解锁。锁的“粒度”越小越好。



### 死锁

1. 线程尝试对同一个互斥量锁两次
2. 线程1拥有A锁，请求获得B锁；线程2拥有B锁，请求获得A锁



   **避免死锁的方法：**

   当拿不到所有的锁的时候，放弃自己拥有的锁

#### 读写锁：

与互斥锁类似，但是允许更高的并行性。特性为：**写独占，读共享**。

**读写锁：**

一把读写锁具有三种状态：

1. 读模式下加锁状态
2. 写。。。。。。。
3. 未加锁

特性：

1. 在写的状态下，写是独占；如果是读的状态下，别的线程读的话，是允许的。

2. 写锁优先级高，如果说有读锁和写锁同时抢夺的话，写锁优先级高。

练习：

##### 主要应用函数：

pthread_rwlock_init 函数

pthread_rwlock_destroy 函数

pthread_rwlock_rdlock 函数

pthread_rwlock_wrlock 函数

pthread_rwlock_tryrdlock 函数

pthread_rwlock_tyyrdlock 函数

pthread_rwlock_trywrlock 函数

pthread_rwlock_unlock 函数



pthread_rwlock_t 类型，用于定义一个读写锁变量

练习：

3个线程不定时“写”全局资源；五个线程不定时“读”同一个全局资源

```c
/*************************************************************************
	> File Name: rdwrlock.c
	> Author: 
	> Mail: 
	> Created Time: 2019年08月01日 星期四 16时45分32秒
 ************************************************************************/

#include<stdio.h>
#include<stdlib.h>
#include<pthread.h>
#include<unistd.h>
pthread_rwlock_t rwlock;
int counter;

void *cal_wr(void *arg){
    int t;
    int i = (int)arg;
    while(1){
        t = counter;
        usleep(1000);
        pthread_rwlock_wrlock(&rwlock);
        printf("--------write %d: %lu:counter = %d ++counter == %d\n",i ,pthread_self(),t,++counter );
        pthread_rwlock_unlock(&rwlock);
       usleep(5000);
    }
    return NULL;
}

void *cal_rd(void *arg){
    int i = (int)arg;
    int t;
    while(1){
        t = counter;
        pthread_rwlock_rdlock(&rwlock);
        printf("!!!!!!!!read %d:  %lu:counter = %d counter   == %d\n", i, pthread_self(),t,counter);
        pthread_rwlock_unlock(&rwlock);
        usleep(900);
    }
    return NULL;
} 

int main(){
    pthread_t tid[9];
    pthread_rwlock_init(&rwlock,NULL );
    int i;
    for ( i = 0 ;i < 3 ;i++ ){
        pthread_create( &tid[i] , NULL ,cal_wr ,(void *)i);
    }
    for ( i = 0; i < 5 ;i++ ){
        pthread_create( &tid[i+3] , NULL , cal_rd ,(void *)i);
    }
    for ( i = 0 ;i < 8;i++ ){
        pthread_join(tid[i] , NULL);
    }
    pthread_rwlock_destroy(&rwlock);
    return 0;

}
```

得到的结论：读是可以共享的，但是写不可以。写锁优先级高

### 条件变量

条件变量本身不是锁，但是也可以造成线程阻塞，通常与呼出锁配合。给多线程提供一个共享数据。

**函数：**

pthread_cond_init 

pthread_cond_destroy

**pthread_cond_wait**

pthread_cond_timedwait

限时等待一个环境变量

pthread_cond_signal

唤醒至少一个阻塞在条件变量上的线程

pthread_cond_broadcast

唤醒所有阻塞在条件变量上的线程



**pthread_cond_wait：**

阻塞等待一个条件变量

int pthread_cond_wait(pthread_cond_t *restrict cond ,pthread_mutex_t *restrict mutex);

函数作用：

1. 阻塞等待条件变量cond满足条件。
2. 释放已经掌握的互斥锁。

 **1.2 两步为原子操作**

3. 当被唤醒，pthread_cond_wait函数返回时，接触阻塞并重新申请获取互斥锁。

pthread_cond_timedwait(pthread_cond_t *restrict cond , pthread_mutex_t *restrict mutex , const struct timespec *restrict abstime);

形参：

abstime :绝对时间

使用方法：

time_t cur = time(NULL); 获取当前时间

struct timespec t; 定义timespec 结构体变量t

t.tv_sec = cur +1; 定时1s

pthread_cond_timedwait(&cond , & mutex ,&t);

unix计时元年： 1970/1/1 00:00:00 

### 线程同步中的生产者消费者条件变量模型

两个线程

生产者：

1. 生产食物

2. 上锁，向公共区域投放信息
3. 解锁
4. 传出信号

消费者：

1. 上锁

2. 判断是否有信息

3. 上锁

4. 接受信息

5. 销毁信息

6. 解锁

   

```c
#include<stdlib.h>
#include<stdio.h>
#include<pthread.h>
#include<unistd.h>

pthread_cond_t has_product = PTHREAD_COND_INITIALIZER ;    
pthread_mutex_t lock  = PTHREAD_MUTEX_INITIALIZER;
struct msg{
    int num;
    struct msg *next;
};
struct msg *head;
struct msg *mp;

void *cal_customer(void *arg){
    while(1){
        pthread_mutex_lock(&lock);
        while(head == NULL){
        pthread_cond_wait(&has_product ,&lock);   
        }

        mp = head;
        head = mp->next;
        pthread_mutex_unlock(&lock);
        
        printf("-customer  ---%d\n",mp->num);
        free(mp);
        sleep(rand()%3);
    }
    return NULL;
}

void *cal_productor(void *arg){
    while(1){
        mp = malloc(sizeof(struct msg));
        mp->num = rand() % 100 + 1 ;
        printf("product ---%d\n",mp->num);
        pthread_mutex_lock(&lock);
        mp->next = head;
        head = mp;
        pthread_mutex_unlock(&lock);

       pthread_cond_signal(&has_product);
        sleep(rand()%3);
    }
    return NULL;
}

int main(){
    head = NULL;
    mp = NULL;
    int ret1 , ret2;
    pthread_t fd1 ,fd2;
    ret1 = pthread_create(&fd1 , NULL , cal_customer ,NULL);
    if(ret1 != 0 ){
        perror("create pthread 1 error");
        exit(1);
    }
    ret2 = pthread_create(&fd2 , NULL , cal_productor ,NULL);
    if(ret2 != 0){
        perror("create pthread 2 error");
        exit(1);
    }
    pthread_join(fd1  , NULL);
    pthread_join(fd2 , NULL);

    return 0;
}

```

ps：为了保证正确性，我们应该对公共区域进行上锁，也就是说，对公共区域进行操作的话，必须具有锁。

条件变量的优点：

相对于mutex而言，条件变量可以减少竞争。

如果使用mutex，除了生产者，消费者之间要竞争互斥量之外，消费者之间也需要。有了条件变量机制之后，可以避免这种情况。



### 信号量

进化版的互斥锁 （1 -> N）

**作用：**

提升共同访问共享资源的线程数量(串行运行 - 》并行运行)

**主要应用函数**

sem_init

int sem_init(sem_t *sem , int pshared , unsigned int value);

sem_destroy

sem_wait

**加锁，对信号量进行 --**

sem_trywait

sem_post

**解锁，对信号量进行 ++**

**结论：**

信号量的初值，决定了占用信号量的线程个数



练习：

生产者，消费者模型（队列实现）



