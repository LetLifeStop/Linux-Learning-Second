### 共享内存

  共享打开的文件符也可以在没有fork的前提下，通过文件可以实现进程间通信

> mmap函数

1. 参数，返回值

    void *mmap(void *addr , size_t length , int prot , int flags , int fd ,off_t offset);

   1. addr  建立映射区的首地址，由linux内核指定；使用时直接传NULL

   2. length 欲创建映射区的大小（内核）

   3. prot  映射区权限 PROT_READ , PROT_WRITE , PROT_READ|WRITE.

   4. flags 标志性参数（设定窝里更新区域，设置共享，创建匿名映射区）

       MAP_SHARED ：将映射区所做的修改反映到物理内存中（父进程和子进程共享）

      MAP_PRIVATE   : 映射区所做的修改不会反映到物理内存中（父子进程各自独享）

   5. fd 文件描述符

   6. offset 映射文件的偏移量（部分还是全部）（4k的整数倍）

   返回值：成功，返回创建的映射区首地址；失败 MAP_FAILED 宏

2. 借助共享指针存储磁盘文件（借助指针访问磁盘文件）
3. 父子进程，血缘 关系进程 通信
4. 匿名映射区

练习：通过mmap函数模拟向共享内存中写文件

```
#include<stdio.h>
#include<fcntl.h>
#include<unistd.h>
#include<string.h>
#include<stdlib.h>
#include<sys/mman.h>
#include<sys/types.h>
int main(  )  {
	  char *p = NULL ;
	  int fd =open("dd.txt",O_CREAT|O_RDWR|O_TRUNC, 0644); //首先获得文件的文件表示符
	  if(fd < 0){
	perror("open error:");
	exit(1);  
	  }
	  int len = ftruncate(fd , 4 );  // 将参数指定的文件大小改为4指定的大小
	  if(len == -1){
		perror("ftruncate error") ;
		exit(1);  
	  }
	  p= mmap(NULL,4,PROT_WRITE|PROT_WRITE,MAP_SHARED,fd,0);  //创建一个4kb的映射区
      if(p == MAP_FAILED){
	  perror("mmap error");
	  exit(1);  
	  }
	  strcpy(p ,"abc"); // 模拟向映射区进行写操作
	int ret ; 
	  ret =  munmap(p ,4 );   // 释放空间
	 if(ret == -1){
		perror("munmap error:");
		exit(1);  
	 }
	 close(fd); // 关闭文件

	return 0;
}
```

注意事项

1.  不能创建大小为0的映射区，也就是说第2个参数不能为0
2. p++的话，munmap会失败
3. 当文件只有读的权限时，但是映射区想再获得读和写的操作，会报Permission denied （权限不足）。如果把第四个参数改为私有的，也及时映射区的关系和物理内存的关系断了，会报总线错误（系统错误）。**创建映射区的权限要小于等于打开文件的权限，映射区创建的过程中隐含着对文件的一次读操作**
4. 映射的时候为什么最小单位是4k？ 因为mmu最小的单位就是4k
5. p越界操作会导致munmap失败
6. 文件描述符对mmap映射没有影响

#### mmap父子进程间通信

> unlink函数 

  删除临时文件目录项，是指具备被释放的条件， 具体信息查看man文档

   目录项（本质就是一个目录项），包含两部分 ，文件名和inode编号。

**【Linux 进程】之关于父子进程之间的数据共享分析]**

https://www.cnblogs.com/xuelisheng/p/9361916.html （重要）

在mmap的第三个参数中，如果是shared，这个时候父进程和子进程是共享文件偏移量的，共享建立的映射区；private不会共享。  



#### 匿名映射

我们的目的是为了父子进程之间通信，但是如果按照上述操作的话，还需要打开一个文件进行辅助操作；为了避免这样，我们可以通过匿名映射的方法来实现使用映射区来完成文件读写操作。

使用方法：使用MAP_ANONYMOUS(只在linux中实用)

 int *p = mmap(NULL , 4 ,PROT_READ|PROT_WRITE , MAP_SHARED|MAP_ANONYMOUS ,-1,0)

4随意举例

```c
#include<unistd.h>
#include<sys/wait.h>
#include<stdlib.h>
#include<stdio.h>
int main(  )  {
	pid_t pid,wpid;
	int i;
	for(i = 0 ;i < 3;i++ ) {
	wpid =	fork(  ) ;
		if( wpid ==0)break;
     }  
if( i== 0) {  
	sleep(1);
	printf("%d th\n",i);  
	execlp("ps","ps","-ef",NULL);  
}  
else if(i==1){
	sleep(2);  
	printf("%d th\n",i);  
//	char *p = "1231231\n";
//	p[0]='h';
	printf("1231\n");  
}  
else if(i==2)
{
	sleep(5);  
	printf("%d th\n",i);  
	execl("/bin/ls","ls","-l",NULL);  
}  
int num = 0 ;
if( i== 3){
	num = 0;
     do{
	wpid = waitpid( -1 , NULL , WNOHANG);
	if(wpid >0 )num++;  
	 }while(num<3);
printf("%d\n",num);
}
return 0;
}  
```

因为上述方法只能适用linux，为了弥补这个问题，可以使用如下方法：

fd = open("/dev/zero" , O_RDWR); 

   对于这个文件，没有实际的大小，用来创建映射区

p = mmap(NULL,size,PROT_READ|PROT_WRITE,MMAP_SHARED,fd ,0); 

   对于这个文件，也没有实际大小，用来接收

```c
#include<stdio.h>
#include<fcntl.h>
#include<unistd.h>
#include<string.h>
#include<stdlib.h>
#include<sys/mman.h>
#include<sys/types.h>
int main(  )  {
	  char *p = NULL ;
	  int fd =open("/dev/zero",O_CREAT|O_RDWR|O_TRUNC, 0644); //首先获得文件的文件表示符
	  if(fd < 0){
	perror("open error:");
	exit(1);  
	  }
	  int len = ftruncate(fd , 4 );  // 将参数指定的文件大小改为4指定的大小
	  if(len == -1){
		perror("ftruncate error") ;
		exit(1);  
	  }
	  p= mmap(NULL,4,PROT_WRITE|PROT_WRITE,MAP_SHARED,fd,0);  //创建一个4kb的映射区
      if(p == MAP_FAILED){
	  perror("mmap error");
	  exit(1);  
	  }
	  strcpy(p ,"abc"); // 模拟向映射区进行写操作
	int ret ; 
	  ret =  munmap(p ,4 );   // 释放空间
	 if(ret == -1){
		perror("munmap error:");
		exit(1);  
	 }
	 close(fd); // 关闭文件

}
```



mmap无血缘关系进程间通信

示例：

写入：

```c
/*************************************************************************
    > File Name: mmap_w.c
    > Author: ma6174
    > Mail: ma6174@163.com 
    > Created Time: 2019年07月21日 星期日 18时21分37秒
 ************************************************************************/

#include<stdio.h>
#include<sys/stat.h>
#include<fcntl.h>
#include<unistd.h>
#include<stdlib.h>
#include<sys/mman.h>
#include<string.h>

struct STU{
	int id;
	char name[20];
	char sex; 
};
void sys_err(char *str){
	perror(str);
	exit(-1);  
}
int main(int argc ,char *argv[  ])  {
	  int fd;
	  struct STU student={10,"hqx",'m'} ;
	  struct STU *mm;
	  if(argc < 2){
		printf( "./a.out fild_shared\n");
		exit(-1);  
	  }
	  fd = open(argv[1],O_RDWR|O_CREAT ,0664);
          ftruncate(fd ,sizeof(student));
	  if(fd == -1){
		sys_err("open eoor") ; 
	  }
	  mm = mmap(NULL , sizeof(student),PROT_READ|PROT_WRITE,MAP_SHARED,fd,0);
	  if(mm == MAP_FAILED){
		sys_err("mmap error");  
	  }
	  close(fd); // 放开fd
	  while(1){
            memcpy(mm,&student,sizeof(student)); // 赋值
            student.id++;
            sleep(1);
	  }
	  return 0;
}
```

读出：

```c
/*************************************************************************
    > File Name: mmap_r.c
    > Author: ma6174
    > Mail: ma6174@163.com 
    > Created Time: 2019年07月21日 星期日 18时05分36秒
 ************************************************************************/

#include<stdio.h>
#include<sys/stat.h>
#include<fcntl.h>
#include<unistd.h>
#include<stdlib.h>
#include<sys/mman.h>
#include<string.h>

struct STU{
	int id;
	char name[20];
	char sex;
};
void sys_err(char *str){
	perror(str);
	exit(-1);  
}
int main(int argc ,char *argv[  ])  {
	  int fd;
	  struct STU student;
	  struct STU *mm;
	  if(argc < 2){
		printf( "./a.out fild_shared\n");
		exit(-1);  
	  }
	  fd = open(argv[1],O_RDONLY);
	  if(fd == -1){
		sys_err("open eoor") ; 
	  }
	  mm = mmap(NULL , sizeof(student),PROT_READ,MAP_SHARED,fd,0); //注意这里为什么要用指针？因为是一个多重操作，所以通过指针我们可以获取映射区的首地址，这样就可以循环输出了
	  if(mm == MAP_FAILED){
		sys_err("mmap error");  
	  }
	  close(fd);
	  while(1){
		printf("id=%d\tname=%s\t%c\n",mm->id,mm->name,mm->sex);
		sleep(1);  
	  }
	  return 0;
}
```

open文件以后，操作系统进行了一次mmap

strace 程序    可以追踪程序运行的过程中使用了系统调用有哪些

  

> 练习

实现文件多进程的拷贝

对于当前的文件，分成五个部分来进行拷贝

