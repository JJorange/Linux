### 写入管道
```c++
#include<string.h>
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<errno.h>
#include<sys/types.h>
#include<sys/stat.h>
#include<fcntl.h>

#define ERR_EXIT(m) \
	do \
{ \
	perror(m); \
	exit(EXIT_FAILURE); \
}while(0)

int main()
{
	   int infd;
	   int outfd;
	   char buf[1024];
	   int n;

	   mkfifo("tp.c", 0644);
	   infd = open("1.c", O_RDONLY);

	   if(infd == -1)
		   ERR_EXIT("open");

	   outfd = open("tp.c", O_WRONLY);
	   if(outfd == -1)
		   ERR_EXIT("open");

	   while((n = read(infd, buf, 1024)) > 0)
	   {
		      write(outfd, buf, n);
	   }

	   close(infd);
	   close(outfd);

	   return 0;
}

```

### 从管道读
``` c++
#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<errno.h>
#include<unistd.h>
#include<sys/types.h>
#include<sys/stat.h>
#include<fcntl.h>

#define ERR_EXIT(m) \
	do \
{ \
	perror(m); \
	exit(EXIT_FAILURE); \
}while(0)

int main()
{
	int infd;
	int outfd;
	int n;
	char buf[1024];

	outfd = open("1.c.bak",O_WRONLY | O_CREAT | O_TRUNC, 0644);
	if(outfd == -1)
		ERR_EXIT("open");

	infd = open("tp.c",O_RDONLY);
	if(outfd == -1)
		ERR_EXIT("open");

	while((n = read(infd,buf,1024)) > 0)
	{
		   write(outfd,buf,n);
	}

	close(infd);
	close(outfd);
	unlink("tp.c");

	return 0;
}

```
