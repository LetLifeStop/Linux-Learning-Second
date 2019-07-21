### exec函数族（无成功返回值，只有失败返回 -1）

 fork 创建子进程之后执行的是和父进程相同的程序，子进程这个时候默认是执行和父进程相同的代码；如果我想让子进程去执行另一个程序，这个时候就需要用到exec函数了，这个进程的代码和数据全部清空，去执行一个新的程序；但是**调用exec函数并不创建新的进程。**

> execlp函数

  l是list ，列表的缩写；p是PATH，路径的缩写；

 这个函数需要配合PATH环境变量来使用，然后第二个参数就是argv[0] ,argv[1] .... 。一般来说argv[0]是没有作用的。注意在最后面加上NULL.

变参函数：参数有多个，但是需要有一个结尾。

  代码示例如下：



```
#include<stdio.h>
#include<unistd.h>
#include<stdlib.h>
int main(  )  {
	  pid_t pid;
	  pid= fork(  ) ;
	  if(pid == -1){
		    perror("  fork erroe!") ;
			exit(1);  
	  }
	  else if(pid > 0){
		    printf("  parent\n");  
	  }
	  else {
		execlp("ls","ls","-l",NULL);  
	  }
	  return 0;
}  
```

> execl函数

   通过路径 +程序名 来加载一个进程

代码示例：

```
#include<stdio.h>
#include<unistd.h>
#include<stdlib.h>
int main(  )  {
	  pid_t pid;
	  pid= fork(  ) ;
	  if(pid == -1){
		    perror("  fork erroe!") ;
			exit(1);  
	  }
	  else if(pid > 0){
		    printf("  parent\n");  
	  }
	  else {
		execl("bin/ls","ls","-l",NULL);  
	  }
	  return 0;
}  
```

1.  l ->list

2. e->environment

3. v-> 数组的形式

   char *argv[] = {"ls","-l",NULL};

​       execv("/bin/ls",argv[]);

4. p ->PATH

练习：将当前系统中的进程信息打印到文件中

首先需要了解文件描述符的构造，从上往下 0 指向stdin，1指向stdout ，2指向stderror ，如果open了一个文件之后3就会指向这个file. 所以，对于这个联系，我们需要将stdout指向我们所需要输出到的那个文件当中去。

然后我们可以首先或者这个文件的描述符，然后通过dup2把stdout复制到文件当中去，这样就可以实现了。

```
#include<stdio.h>
#include<fcntl.h>
#include<stdlib.h>
#include<unistd.h>
int main(  )  {
	  int fd = open ("ps.out" , O_WRONLY|O_CREAT|O_TRUNC, 0664);  
     if(fd < 0 ){
		perror("open ps.out error");
		exit(1);  
	 }
	 dup2(fd, STDOUT_FILENO); // 注意dup2要在execlp之前进行
	 execlp("ps" ," ps" ,"ax" , NULL);
	 return 0;
}
```

因为execlp函数当运行成功的时候是没有返回值的，只有在创建失败的时候才会返回 -1 。那这样的话，如何判断是否创建成功呢？

这个时候就需要看到execlp函数的特点了，当调用exec之类的函数的时候，原来的函数中除了filedescriptor table 之外的所有东西（stack，heap）都被exec的东西所覆盖掉，所以不会执行； 通过这个特点，我们只需要在execlp函数的下面加上：

```
perror("execlp error");
exit(1)
```

就可以了；如果execlp这个函数执行失败的话，他会继续往下走。





### 回收子进程

> **孤儿进程**

父进程先于子进程结束，则子进程成为孤儿进程，子进程的父进程变成了init进程，init进程俗称孤儿院。init进程不一定为1，只是一个正数。

> **僵尸进程**

当进程退出父进程的时候，没有读取到子进程退出的返回代码，这个时候就会产生僵尸进程。僵尸进程会一终止态保存在进程表中，然后等待父进程读取退出状态代码。

**ps：**僵尸进程不能通过kill命令来清除掉。kill命令是用来终止进程的，但是僵尸进程已经结束了，所以没法通过kill命令来清除。

##### 回收子进程的函数

> wait函数

1. 阻塞等待子进程退出

2. 回收子进程残留资源

3. 获取子进程结束状态

   **一次wait函数，只回收一个子进程，哪个先结束，就回收哪个，如果是多个的话，我们可以在父进程中通过while循环来实现**

```
#include<stdio.h>
#include<unistd.h>
#include<sys/wait.h>
#include<stdlib.h>
int main(  )  {
	pid_t pid,wpid;
	pid = fork(  );
	if( pid ==  0 ){
		printf("I am son r,my pid = %d\n  ", getpid(  ));  
	       sleep(10);  
	}
	else if(pid > 0 ){
	    int *t;
		wpid = wait(t);  // t是传出参数，这里是子进程的状态
		if(wpid == -1){
			  perror("wait error");
			  exit(1);  
		}
	//printf("%d\n",*t);    // 不知道为何，这个编译的时候在linux环境下会re
		printf("I am father,my pid = %d my son  pid = %u\n" ,getpid(  ),pid );
		printf("  -------------------------");   
		}
	else {
		  perror("fork \n ") ;
		  return 1;
	}
	return 0;
}
```

我们可以通过wait函数传出的参数判断子进程的结束原因。

WIFEITED , WEXITSTATUS  查看man文档 ，判断退出的状态

1. WIFEITED为非0 ， 代表正常结束 ；然后在正常结束的前提下，通过WEXITSTATUS查看退出的信息（exit的参数）。

2. WIFSINALED为非0，代表进程异常终止；然后在进程异常终止的前提下，通过WTERMSIG查看使得子进程提前终止的信号编码。

   ```c
   #include<stdio.h>
   #include<unistd.h>
   #include<sys/wait.h>
   #include<stdlib.h>
   int main(  )  {
   	pid_t pid,wpid;
   	pid = fork(  );
   	if( pid ==  0 ){
   		printf("I am son ,my pid = %d\n  ", getpid(  ));  
   	       sleep(60);  
   	}
   	else if(pid > 0 ){
   	    int t;
   		wpid = wait(&t);
   		if(wpid == -1){
   			  perror("wait error");
   			  exit(1);  
   		}
   		if(WIFSIGNALED(t)){
   			printf(" signal = %d\n",WTERMSIG(t));  
   		}
   	//printf("%d\n",*t);  
   		//printf("I am son ,my pid = %d my son  pid = %u\n" ,getpid(  ),pid );
   		printf("  -------------------------");   
   		}
   	else {
   		  perror("fork \n ") ;
   		  return 1;
   	}
   	return 0;
   }
   ```

  注意，在wait函数中，参数是一个指针类型的，我一开始是按照*t的方式来进行的，但是在后面判断是否异常终止时。WIFSIGNALED的参数是一个int类型的，如果还是按照 *t 的方法的话，这样会出错的。



> waitpid函数

 

函数原型：pid_t waitpid(pid_t pid , int *wstatus , int options);

第一个参数pid：

1. \> 0 回收指定ID的子进程

2. -1 回收任意子进程（相当于wait）

3. 0 回收和当期调用waitpid一个组的任意一个子进程

4. < -1 回收指定进程组内的任意子进程( ps ajx 可以查看分组信息（PGID）) 

   使用方法 \-组数

第二个参数和wait中的函数用法相同。

第三个参数代表操作方法，WNOHANG 代表不阻塞，也就是父进程不会停下来等待子进程结束，但是会通过类似于CPU分配运行的感觉，就是时间轮片，过一会过来看看，过一会过来看看，直到回收完成。

```c
#include<stdio.h>
#include<unistd.h>
#include<sys/wait.h>
#include<stdlib.h>
int main(  )  {
	pid_t pid,wpid;
    int i ;
	for(i = 0 ;i < 5 ;i++ ) {
        wpid = fork();
        if(wpid == 0 )break;
    }
    if(i== 5){
        do{
           wpid = waitpid(-1 , NULL , WNOHANG);
            if(wpid > 0 )n--;
        }while(n>0);
    }
	return 0;
}
```

出现re的一种操作：

通过字符指针直接指向的字符串是不能被修改的。

```c
char *p = "hello"; p[0]='h'; 
```

将一个字符内容为"hello"的字符数组的首地址赋值给了字符指针，也就是说上述代码是错的。

作业：父进程fork 3 个子进程，三个子进程 一个调用ps命令，一个调用自定义程序1（正常），一个调用自定义程序2（段错误），父进程使用waitpid 对其子进程进行回收

```c
#include<unistd.h>
#include<sys/wait.h>
#include<stdlib.h>
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
//	p[0]='h';   //这里就是会re，但是竟然没报错
	printf("1231\n3");  
}  
else if(i==2)
{
	sleep(5);  
	printf("%d th\n",i);  
	 execl("/bin/ls","ps","-l",NULL);
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

~
一定要注意路径是  /bin/ls 一定要注意bin前面的/ !!!!

**以后编码一定要注意细节**