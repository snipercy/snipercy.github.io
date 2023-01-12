# For Understanding Unix/Linux Programming
-----
## `cp` : 实现`bash`中的`cp(copy)`命令
命令用法：`$cp srcfile targetfile`
如果`targetfile`所指定的文件不存在，`cp`就创建这个文件；如果文件已经存在则覆盖，`targetfile`的内容与`srcfile`相同。

###  1. 程序流程：

```c
  open sourcefile for reading
  open copyfile for writing
  +->read from source to buffer -- eof?---+
  |                                       |
  -- write from buffer to copyfile        |
                                          |      
  	  close sourcefile <---------------+
 	  close copyfile
```

用户态的进程是不能访问内核空间的，否则就会出现`segmentfault`，因为没有访问权限，所以处于内核态的用户进程需要将**数据拷贝**到用户空间，这是系统调用开销大原因之一。

扯一句，在网络编程模型中，为什么`epoll`比`select`/`poll`效率高，其中一个原因就是`epoll`减少了`fd`集合的拷贝。(* 先挖两个坑，`syscall`和`epoll`相关知识将将用两篇博文来整理一下*)

另一个导致`syscall`开销大的的原因是：**运行环境的切换**。用户进程(`cp`)位于用户空间，内核位于系统空间，`cp`所要操作的文件是位于磁盘中的，而磁盘等设备只能被处于特权级(`ring 0`)的进程直接访问，即处于内核态的进程才可以访问。用户进程要想获得特权级，需要陷入内核态，可以通过系统调用来实现。具体到`cp`命令就通过调用`read`和`write`等`syscall`使得执行权从用户代码转换到内核代码，这种切换是需要时间的。
当运行内核代码时，`cpu`工作在特权级(`ring 0`)，此时，内核代码使用内核空间中的堆栈和内存环境(内核空间)，这些需要在内核代码执行前准备好，系统调用结束后(`read`返回时)，`cpu`又需要切换到用户模式。
有一个形象的例子来说明这种切换的开销：超人-普通人之间切换：换衣服开销


### 2. 利用缓存提升程序效率

#### a. 应用缓冲技术

先看这样一个生活场景：我们需要煎3个荷包蛋，每次取超市买一个鸡蛋，回家煎好后再去超市买一个。显然，这是很低效的做法，完全可以一次性买3个鸡蛋再煎荷包蛋。

`cp`命令也是做类似的事情：超市里的购物车对应于缓存，鸡蛋对应于源文件内容，荷包蛋对应于目标文件。`cp`要做的就是一次性尽可能多的将源文件中的内容拷贝到缓存文件中，再将缓存中文件拷贝到目标文件中，从而减少`read`和`write`调用的次数。

综上，应用缓存的主要思想是一次读入大量的数据放入缓冲区中，需要的时候从缓冲区中取得数据。


#### b. 内核缓冲技术

上述的**应用缓冲技术**是为了减少系统调用的次数，从而提高程序的效率。`I/O`调用比系统调用还要慢，能直接操作外设的只有内核态程序，所以可以利用内核缓冲技术来较少`I/O`调用的次数。

具体到`cp`程序：内核将磁盘上的数据块复制到内核缓冲区，当一个用户空间的进程需要从磁盘上读取数据时，是将内核缓冲区中的数据块复制到进程的缓冲区中的。

当进程所需的数据块不在内核缓冲区时，该数据块加入请求数据列表中，并挂起进程。一段时间后，内核把相应数据块从磁盘读到内核缓冲区，然后再把数据复制到进程缓冲区中，最后唤醒被挂起的进程。

所以，`read`是把数据从内核缓冲区复制到进程缓冲区，`write`把数据从进程过程区写到内核缓冲区而并非直接写入到磁盘中，这么做的目的是减少I/O提高效率，但也会带来弊端，如,断电导致数据丢失。

在`Linux`系统中，为了加快数据的读取速度，默认的情况下，某些已经加载内存中的数据将不会直接被写回硬盘，而是先缓存在内存当中，如果一个数据被你重复的改写，那么由于他尚未被写入硬盘中，因此可以直接由内存当中读取出来，在速度上一定是快上相当多的！不过，如此一来也造成些许的困扰，那就是万一你的系统因为某些特殊情况造成不正常关机，由于数据尚未被写入硬盘当中，这个时候就需要sync这个命令来进行数据的写入动作,`shutdown`/`reboot`/`halt`等命令均已经在关机前自动执行了sync命令。


```c
#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>

#define BUFFERSIZE 4096
#define COPYMODE   0644

void oops(char*,char*);

int main(int argc, char* argv[]){
	int in_fd;
	int out_fd;
	int n_chars;
	char buf[BUFFERSIZE];

	if(argc != 3){
		fprintf(stderr,"useage:%s source destination\n",*argv);
		exit(1);
	}

	if( (in_fd = open(argv[1],O_RDONLY)) == -1){
		fprintf(stderr,"Error: connot open %s",argv[1]);
		exit(1);
	}

	if( (out_fd = creat(argv[2],COPYMODE)) == -1){
		fprintf(stderr,"Error: connot creat %s",argv[2]);
		exit(1);
	}

	while((n_chars = read(in_fd,buf,BUFFERSIZE)) > 0){
//fprintf(stdout,"%s",buf);
		if( write(out_fd,buf,n_chars) != n_chars){
			fprintf(stderr,"Write error %s",argv[2]);
			exit(1);
		}
	}

	if( n_chars == -1){
		fprintf(stderr,"Read error from %s",argv[2]);
		exit(1);
	}

	if( close(in_fd) == -1 || close(out_fd) == -1){
		fprintf(stderr,"close error %s");
		exit(1);
	}
}
```



------

## ls : 实现`bash`中的`ls(list files)`命令
### ls1.c
主要步骤:
```c
DIR dir = opendir(const char *)
readdir(dir)
printf(dirent->d_name)
closedir(dir)

```

程序运行结果：

```c
.
..
a.out
fileinfo.c
ls1.c
ls2.c
README.md

```

### ls2.c

增加显示文件的属性，属性信息可以通过stat函数获得。

```c
drwxr-xr-x   7      cy   staff     238Jan 18 15:54.
drwxr-xr-x   9      cy   staff     306Jan 18 15:48..
-rwxr-xr-x   1      cy   staff    9640Jan 18 15:54a.out
-rw-r--r--   1      cy   staff    3010Aug 30 00:34fileinfo.c
-rw-r--r--   1      cy   staff    1735Aug 30 10:08ls1.c
-rw-r--r--   1      cy   staff    3853Aug 30 10:45ls2.c
-rw-r--r--   1      cy   staff     313Jan 17 14:40README.md

```

```c
#include<stdio.h>
#include<sys/stat.h>
#include<dirent.h>//directory entry
#include<sys/types.h>
#include<string.h>

void do_ls(char[]);
void do_stat(char*);
void show_file_info(char*,struct stat*);
void mode_to_letters(int,char*);
char *uid_to_name(uid_t);
char *gid_to_name(gid_t);

int main(int argc, char* argv[]){
	if(argc == 1){
		do_ls(".");
	}
	else{
		while(--argc){
			/* '*' and '++'优先级相同，由于右结合,
			 故*++argv = *(++argv)*/
			printf("%s:\n",*++argv);
			do_ls(*argv);

		}
	}
}

/*DIR dirent struct defined*/
#if 0
/* structure describing an open directory. */
typedef struct {
	int	__dd_fd;	/* file descriptor associated with directory */
	long	__dd_loc;	/* offset in current buffer */
	long	__dd_size;	/* amount of data returned */
	char	*__dd_buf;	/* data buffer */
	int	__dd_len;	/* size of data buffer */
	long	__dd_seek;	/* magic cookie returned */
	long	__dd_rewind;	/* magic cookie for rewinding */
	int	__dd_flags;	/* flags for readdir */
	__darwin_pthread_mutex_t __dd_lock; /* for thread locking */
	struct _telldir *__dd_td; /* telldir position recording */
} DIR;

struct dirent __DARWIN_STRUCT_DIRENTRY;
#define __DARWIN_STRUCT_DIRENTRY { \
	__uint64_t  d_ino;      /* file number of entry */ \
	__uint64_t  d_seekoff;  /* seek offset (optional, used by servers) */ \
	__uint16_t  d_reclen;   /* length of this record */ \
	__uint16_t  d_namlen;   /* length of string in d_name */ \
	__uint8_t   d_type;     /* file type, see below */ \
	char      d_name[__DARWIN_MAXPATHLEN]; /* entry name (up to MAXPATHLEN bytes) */ \
}
#endif
/*
 * list files in directory called dirname
 */
void do_ls(char dirname[]){
	DIR *dir_ptr;		   /*dir file*/
	struct dirent *direntp;/*each entry*/

	if((dir_ptr = opendir(dirname)) == NULL)
		fprintf(stderr,"cannot open %s\n",dirname);
	else{
		while((direntp = readdir(dir_ptr)) != NULL){
			do_stat(direntp->d_name);
		}
		closedir(dir_ptr);
	}
}

void do_stat(char *filename){
//printf("do_stat hhhhhhhhh\n");
	struct stat info;
	if(stat(filename,&info) == -1)
		perror(filename);
	else
		show_file_info(filename,&info);
}

void show_file_info(char *filename,struct stat *info_p){
	char *uid_to_name(),*ctime(),*gid_to_name(),*filemode();
	void mode_to_letters();
	char modestr[11];

	mode_to_letters(info_p->st_mode,modestr);

	printf("%s",modestr);
	printf("%4d",(int)info_p->st_nlink);
	printf("%8s",uid_to_name(info_p->st_uid));/*左对齐右补空格*/
	printf("%8s",gid_to_name(info_p->st_gid));
	printf("%8ld",(long)info_p->st_size);
	printf("%.12s",4+ctime(&info_p->st_mtime));
	printf("%s\n",filename);
}

void mode_to_letters(int mode,char str[]){
    strcpy(str,"----------")/*10 * '-'*/;
    if(S_ISDIR(mode)) str[0] = 'd';     /*directory*/
    if(S_ISCHR(mode)) str[0] = 'c';     /*char dev*/
    if(S_ISBLK(mode)) str[0] = 'b';     /*block dev:disk dev*/

    if(mode & S_IRUSR) str[1] = 'r';    /*3 bits for user*/
    if(mode & S_IWUSR) str[2] = 'w';
    if(mode & S_IXUSR) str[3] = 'x';

    if(mode & S_IRGRP) str[4] = 'r';    /*3 bits for group*/
    if(mode & S_IWGRP) str[5] = 'w';
    if(mode & S_IXGRP) str[6] = 'x';

    if(mode & S_IROTH) str[7] = 'r';    /*3 bits for other*/
    if(mode & S_IWOTH) str[8] = 'w';
    if(mode & S_IXOTH) str[9] = 'x';
}

#include<pwd.h>
/*get info from /etc/passwd*/
char *uid_to_name(uid_t uid){/*uid_t:unsigned int*/
	//hhhhhhhh
	//struct passwd *getpwuid();
	struct	passwd *getpwuid(), *pw_ptr;
	static char numstr[10];

	if( (pw_ptr = getpwuid(uid) ) == NULL){
		sprintf(numstr,"%d",uid);
		return numstr;
	}
	else
		return pw_ptr->pw_name;
}

#include<grp.h>
/*get info from /etc/group*/
char *gid_to_name(gid_t gid){
	struct group *grp_ptr,*getgrgid();
	static char numstr[10];

	if( (grp_ptr = getgrgid(gid) ) == NULL ){
		sprintf(numstr,"%d",gid);
		return numstr;
	}
	else
		return grp_ptr->gr_name;
}
```

----

## more : 实现`bash`中的`more`命令

read and print 24 lines then pause for a few special commands

```c
+----> show 24 lines form inputs
| +--> print [more?] message
| |    Input Enter,SPACE,or q
| +--- if Enter, advance one line
+----- if SPACE
       if q --> exit
```