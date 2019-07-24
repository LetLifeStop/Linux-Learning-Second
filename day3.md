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

1. 实现文件多进程的拷贝，对于当前的文件，分成五个子进程来进行拷贝

   具体思路：写了一下午，具体是open函数的问题。

   1. 读取要拷贝的文件，读取长度
   2. 创建映射
   3. 建立要写入的文件
   4. 扩展空间
   5. 建立映射
   6. 分配好每个进程都准备哪些内容
   7. 通过映射写入文件
   8. 回收操作

   坑点：

   在open的时候，一定要注意O_TRUNC 的使用

   ![1563802761191](C:\Users\acm506\Desktop\Linux\Linux-Learning-Second\1563802761191.png)

   **这里，当使用O_TRUNC的时候，原来的文件会归零；当归零的时候，mmap的第二个参数就会报错**

   

   ```C
   #include<stdio.h>
   #include<sys/stat.h>
   #include<fcntl.h>
   #include<unistd.h>
   #include<sys/types.h>
   #include<sys/wait.h>
   #include<stdlib.h>
   #include<sys/mman.h>
   #include<string.h>
   
   void sys_err(char *str){
       perror(str);
       exit(-1);
   }
   int min(int t1 ,int t2  )  {
       if(t1 > t2)return t2;
       return t1;
   }
   //int getlen(const char* path){
    //   int filesize = -1;
    //   FILE *fp;
   //   fp = fopen( path , "r");
    //   if(fp == NULL)return filesize;
    //   fseek(fp , 0L ,SEEK_END);
    //   filesize = ftell( fp );
    //   fclose(fp);
    //   return filesize;
   //}
   int sto[10];
   void  cal(int num , int len){ // 分配任务
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
   int main(int argc ,char *argv[]){
       int fd ;
       if(argc < 3) {
          perror("error");
            exit(1);
         }
       int len;
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
       char* w =p_w;
       close(fd1);
       cal(0 , len);
       //for(int i = 0 ; i< 5;i++){
        //   printf("  %d %d\n",i,sto[i]);
      // }
       int i;
       pid_t pid;
       int cnt = 0;
       for (i = 0 ; i< 5; i++) {
           pid = fork();
           if( pid == -1){
               printf("  fork error");
               exit(1);
           }
           if (pid == 0)break;
       }
       if(i != 5){
           sleep(i);
           r += i*(len/5);
           w += i*(len/5);
           memcpy( w, r,sto[i]);
           //	 printf("%d %d \n",i * 12 , sto[i]);
           printf("%d th sto , from %d to %d byte, %d stored\n", i, i * (len/5) ,i * (len/5) + sto[ i ] , sto[ i ] );
       }
       else {
           sleep(8);
           do{
               pid_t pid ;
               pid = waitpid(-1 , NULL , WNOHANG);
               if(pid > 0)cnt++;
           }while(cnt<5);
           printf("back %d process\n",cnt);
       }
       // 按照五个子进程来对文件进行多进程拷贝的操作
   
       int ret = munmap(p_r, len);
       if(ret == -1){
           perror("munmap failed");
           exit(1);
       }
       ret = munmap(p_w , len);
       if(ret == -1){
           perror("munmap failed");
           exit(1);
       }
       return 0;
   }
   ```

2. 实现简易聊天室

   具体实现思路：
   
   1. 首先具有客户端和服务器端，客户端到服务器有一个公共管道，这是为了客户端向服务器端传数据。
   
   2. 服务器端到客户端有一个私有管道，这是为了服务器端向客户端传输数据。
   
   3. 每一次信息传递，客户端通过FIFO公共管道传输到服务器端，然后服务器端处理信息，然后再通过客户端私有的管道传输到客户端
   
      client.h 用来储存客户信息以及头文件
   
      ```c
      #include<stdlib.h>
      #include<stdio.h>
      #include<string.h>
      #include<fcntl.h>
      #include<limits.h>
      #include<sys/types.h>
      #include<sys/stat.h>
      #include<unistd.h>
      #include<ctype.h>
      
      #define SERVER_FIFO_NAME "serve_fifo"  // client -> server
      #define CLIENT_FIFO_NAME "client_%d_fifo" // server ->client
      
      # define BUFFER_SIZE PIPE_BUF
      # define MESSAGE_SIZE 20
      # define NAME_SIZE 256
      
       typedef  struct message{
          pid_t client_pid;   // 客户的文件pid
          char data[MESSAGE_SIZE+10]; 
      }message;
      
      ```
   
      client.c  客户端
   
      ```c
      #include"client.h"
      
      int main(){
          int server_fifo_fd;
          int client_fifo_fd;
          int res;
          message msg;
          char client_fifo_name[1024];
          
          msg.client_pid = getpid();
           // 给client_fifo_name 赋值
          sprintf(client_fifo_name , CLIENT_FIFO_NAME , msg.client_pid);
         
          // 创建管道
        if(mkfifo(client_fifo_name , 0777) == -1){
            perror("creat client_fifo_name error");
            exit(1);
        }
         // 服务器向私有管道中写数据
         server_fifo_fd = open(SERVER_FIFO_NAME ,O_WRONLY);
        if(server_fifo_fd == -1){
            perror("creat server_fifo error");
            exit(1);
        }
      
        sprintf(msg.data , "hello from %d",msg.client_pid);
        printf("%d sent %s",msg.client_pid , msg.data);
         // 客户端向公共管道中写数据
        write(server_fifo_fd , &msg , sizeof(msg));
          
         // 客户端从私有管道中读取数据
         client_fifo_fd = open(client_fifo_name ,O_RDONLY);  
         if(client_fifo_fd == -1){
            perror("creat client fifo fd error");
            exit(1);
        }
        
        res = read(client_fifo_fd , &msg , sizeof(msg));
        if(res > 0){
            printf("received %s\n" , msg.data);
        }
      
        close(client_fifo_fd);
        close(server_fifo_fd);
        unlink(client_fifo_name);
        return 0;
      }
      ```
   
      server.c 服务器端
   
      ```c
      #include"client.h"
      
      int main(){
          int client_fifo_fd;
          int server_fifo_fd;
          char client_fifo_name[1024];
          // 创建公共管道
      	if(mkfifo(SERVER_FIFO_NAME,0777) == -1 ){
              perror("creat SERVER_FIFO error");
              exit(1);
          } 
          //服务器从管道中读取数据 
          server_fifo_fd = open(SERVER_FIFO_NAME , O_RDONLY);
          if( client_fifo_fd < 0 ){
              perror("creat client fifo fd error");
              exit(1);
          }
          sleep(5);
         
      	message msg;
          char *p;
          int res ;
          
      	while(res = read(server_fifo_fd , &msg , sizeof(msg)) > 0 ) {
              p = msg.data;
      	
      		while(*p){
      			*p = toupper(*p);
      			++p;
      		}
      		sprintf(client_fifo_name, CLIENT_FIFO_NAME, msg.client_pid);
              client_fifo_fd = open(client_fifo_name, O_WRONLY);
              if (client_fifo_fd == -1) {
                  perror("creat client_fifo_fd error");
                  exit(1);
              }
              write(client_fifo_fd, &msg, sizeof(msg));
              close(client_fifo_fd);
          }
           close(server_fifo_fd);
           unlink(SERVER_FIFO_NAME);
        return 0;
      }
      
      
      
      ```
   
      
   
      调用方法：(& 代表后台运行)
   
      1. gcc -o server.c server
   
      2. gcc -o client.c client
   
      3.  ./server &
      4. ./client & 

​              **注意**:如果管道本来存在的话，需要先把原来的管道删除