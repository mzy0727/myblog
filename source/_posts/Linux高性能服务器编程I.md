---
title: Linux高性能服务器编程I
date: 2023-9-25 17:38:13
tags: 
     - c++
     - c
     - linux
     - 网络编程
categories: 笔记
---

今天的内容包括一些linux网络编程基本api和实现了TCP协议的服务端以及客户端。
<!-- more -->


# Linux高性能服务器编程笔记

### 主机字节序和网络字节序

大端字节序（网络字节序，数据传输，JAVA虚拟机）：高位字节存储在内存的低地址处

小端字节序（主机字节序，现代PC机）：低位字节存储在内存的低地址处

![image-20230925111901112](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230925111901112.png)

地址族类型通常与协议族类型相对应

![image-20230925112244293](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230925112244293.png)

![image-20230925141532173](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230925141532173.png)

```c
#include <arpa/inet.h>
#include <stdio.h>

int main(int argc ,char *argv[]){
    char buf[4] = {192,168,1,2};
    int num = *(int *)buf;
    int sum = htonl(num);
    unsigned char *p = &sum;

    printf("%d %d %d %d\n",*p,*(p+1),*(p+2),*(p+3));

    return 0;
}
```

```c++
#include <iostream>
#include <arpa/inet.h>

int main(int argc, char *argv[]) {
   	//unsigned char buf[4] = {192, 168, 1, 2};
    char buf[4] = {static_cast<char>(192), static_cast<char>(168), static_cast<char>(1), static_cast<char>(2)};
    
    int num = *(int *)buf;
    int sum = htonl(num);
    unsigned char *p = reinterpret_cast<unsigned char *>(&sum);

    std::cout << static_cast<int>(*p) << " " << static_cast<int>(*(p + 1)) << " " << static_cast<int>(*(p + 2)) << " " << static_cast<int>(*(p + 3)) << std::endl;

    return 0;
}
```

```
2 1 168 192
```

c语言代码转c++时，在初始化`buf`数组时，对于大于一个字节的整数，需要将其强制转换为`char`类型，使用`static_cast<char>(value)`进行转换即可。这样做可以避免`narrowing conversion`错误。

**网络里面的数据类型定义都应该用无符号，更加保险**

### 点分十进制转换

```txt
// src为 点分十进制字符串的IPv4地址 或 十六进制字符串表示的IPv6地址 存入dst的内存中 af指定地址族
// 可以为 AF_INET AF_INET6 成功返回1 失败返回-1
int inet_pton(int af, const char * src, void* dst);
// 协议名, 需要转换的ip, 存储地址, 长度(有两个常量 INET_ADDRSTRLEN, INET6_ADDRSTRLEN)
const char* inet_ntop(int af, const void*  src, char* dst, socklen_t cnt);
```

```txt
// 创建 命名 监听 socket
# include <sys/types.h>
# include <sys/socket.h>
// domain指定使用那个协议族 PF_INET PF_INET6
// type指定服务类型 SOCK_STREAM (TCP协议) SOCK_DGRAM(UDP协议)
// protocol设置为默认的0
// 成功返回socket文件描述符(linux一切皆文件), 失败返回-1
int socket(int domain, int type, int protocol);

// socket为socket文件描述符
// my_addr 为地址信息
// addrlen为socket地址长度
// 成功返回0 失败返回 -1
int bind(int socket, const struct sockaddr* my_addr, socklen_t addrlen);

// backlog表示队列最大的长度
int listen(int socket, int backlog);
// 接受连接 失败返回-1 成功时返回socket
int accept(int sockfd, struct sockaddr* addr, socklen_t* addrlen)
```

在Linux内核2.2之后，`socket backlog`参数的形为改变了，**现在它指等待`accept`的`完全建立`的套接字的队列长度**，而不是不完全连接请求的数量。 **不完全连接**的长度可以使用`/proc/sys/net/ipv4/tcp_max_syn_backlog`设置。这意味着当前Linux版本使用上面第二种说法，有两个队列：具有由系统范围设置指定的大小的`SYN队列` 和 应用程序（也就是backlog参数）指定的`accept`队列。

```shell
cat /proc/sys/net/ipv4/tcp_max_syn_backlog
512
```

### **TCP协议**

![image-20230926154633974](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230926154633974.png)

![TCP协议流程](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230927104224676.png)

#### TCP三次握手

![这里写图片描述](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/20170305084830550)

#### 客户端

```c++
#include <iostream>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>

using namespace std;

int main(){
    // 1. 创建套接字
    int socket_fd;
    socket_fd = socket(AF_INET,SOCK_STREAM,0);
    // 2. 连接服务器
    struct  sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8080);
    inet_pton(AF_INET,"172.18.192.1",&addr.sin_addr.s_addr);
    
    connect(socket_fd,(struct sockaddr*)&addr,sizeof(addr));
    // 3. 读写数据
    char buf[1024] = "";
    while(1){
        int n = read(STDIN_FILENO,buf,sizeof(buf));
        write(socket_fd,buf,n);     //发送数据给服务器
        n = read(socket_fd,buf,sizeof(buf));
        write(STDOUT_FILENO,buf,sizeof(buf));
    }
    // 4. 关闭链接
    close(socket_fd);
    return 0;
}
```

#### 服务器

```c++
#include<iostream>
#include<sys/socket.h>
#include<unistd.h>
#include<arpa/inet.h>
#include<stdio.h>
#include<string.h>

using namespace std;

int main(){

    // 1. 创建套接字
    int socket_fd;
    socket_fd = socket(AF_INET,SOCK_STREAM,0);
    cout<<socket_fd<<endl;
    // 2. 绑定 bind
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8010);
    addr.sin_addr.s_addr = 0;   //如果是0，绑定的是通配地址
   // inet_pton(AF_INET,"192.168.0.102",&addr.sin_addr.s_addr);
    int ret = bind(socket_fd,(struct sockaddr *)&addr,sizeof(addr));
    if(ret < 0){
        perror("");
        exit(0);
    }
    // 3. 监听 listen
    listen(socket_fd,128);  //参数128是等待监听队列和已连接监听队列之和的最大值（linux2.2后，只表示已连接
队列）
    // 4. 提取 accept
    struct sockaddr_in cliaddr; //已建立连接
    socklen_t len = sizeof(cliaddr);
    int Con_fd = accept(socket_fd,(struct sockaddr *)&cliaddr,&len);  //已连接套接字
    char ip[16]="";
    printf("new client ip=%s port=%d\n",inet_ntop(AF_INET, &cliaddr.sin_addr.s_addr,ip,16),ntohs(cliaddr.sin_port));
    // 5. 读写
    char buf[1024] = "";
    while(1){
        bzero(buf,sizeof(buf));     //清空buff
        int n = read(STDIN_FILENO,buf,sizeof(buf));
        write(Con_fd,buf,n);
        n = read(Con_fd,buf,sizeof(buf));
        printf("%s\n",buf);
        
    }
    // 6. 关闭
    close(socket_fd);
    close(Con_fd);
    return 0;
}
```

可以使用命令` nc 127.0.0.1 8008`连接服务器

```txt
new client ip=127.0.0.1 port=56918
```

#### 出错处理封装函数

##### warp.c

```c
#include <stdlib.h>
#include <errno.h>
#include <sys/socket.h>
void perr_exit(const char *s)
{
	perror(s);
	exit(1);
}
int Accept(int fd, struct sockaddr *sa, socklen_t *salenptr)
{
	int n;
	again:
	if ( (n = accept(fd, sa, salenptr)) < 0) {
		if ((errno == ECONNABORTED) || (errno == EINTR))
			goto again;
		else
			perr_exit("accept error");
	}
	return n;
}
int Bind(int fd, const struct sockaddr *sa, socklen_t salen)
{
	int n;
	if ((n = bind(fd, sa, salen)) < 0)
		perr_exit("bind error");
	return n;
}
int Connect(int fd, const struct sockaddr *sa, socklen_t salen)
{
	int n;
	if ((n = connect(fd, sa, salen)) < 0)
		perr_exit("connect error");
	return n;
}
int Listen(int fd, int backlog)
{
	int n;
	if ((n = listen(fd, backlog)) < 0)
		perr_exit("listen error");
	return n;
}
int Socket(int family, int type, int protocol)
{
	int n;
	if ( (n = socket(family, type, protocol)) < 0)
		perr_exit("socket error");
	return n;
}
ssize_t Read(int fd, void *ptr, size_t nbytes)
{
	ssize_t n;
again:
	if ( (n = read(fd, ptr, nbytes)) == -1) {
		if (errno == EINTR)
			goto again;
		else
			return -1;
	}
	return n;
}
ssize_t Write(int fd, const void *ptr, size_t nbytes)
{
	ssize_t n;
again:
	if ( (n = write(fd, ptr, nbytes)) == -1) {
		if (errno == EINTR)
			goto again;
		else
			return -1;
	}
	return n;
}
int Close(int fd)
{
	int n;
	if ((n = close(fd)) == -1)
		perr_exit("close error");
	return n;
}
ssize_t Readn(int fd, void *vptr, size_t n)
{
	size_t nleft;
	ssize_t nread;
	char *ptr;

	ptr = vptr;
	nleft = n;

	while (nleft > 0) {
		if ( (nread = read(fd, ptr, nleft)) < 0) {
			if (errno == EINTR)
				nread = 0;
			else
				return -1;
		} else if (nread == 0)
			break;
		nleft -= nread;
		ptr += nread;
	}
	return n - nleft;
}

ssize_t Writen(int fd, const void *vptr, size_t n)
{
	size_t nleft;
	ssize_t nwritten;
	const char *ptr;

	ptr = vptr;
	nleft = n;

	while (nleft > 0) {
		if ( (nwritten = write(fd, ptr, nleft)) <= 0) {
			if (nwritten < 0 && errno == EINTR)
				nwritten = 0;
			else
				return -1;
		}
		nleft -= nwritten;
		ptr += nwritten;
	}
	return n;
}

static ssize_t my_read(int fd, char *ptr)
{
	static int read_cnt;
	static char *read_ptr;
	static char read_buf[100];

	if (read_cnt <= 0) {
again:
		if ((read_cnt = read(fd, read_buf, sizeof(read_buf))) < 0) {
			if (errno == EINTR)
				goto again;
			return -1;	
		} else if (read_cnt == 0)
			return 0;
		read_ptr = read_buf;
	}
	read_cnt--;
	*ptr = *read_ptr++;
	return 1;
}

ssize_t Readline(int fd, void *vptr, size_t maxlen)
{
	ssize_t n, rc;
	char c, *ptr;
	ptr = vptr;

	for (n = 1; n < maxlen; n++) {
		if ( (rc = my_read(fd, &c)) == 1) {
			*ptr++ = c;
			if (c == '\n')
				break;
		} else if (rc == 0) {
			*ptr = 0;
			return n - 1;
		} else
			return -1;
	}
	*ptr = 0;
	return n;
}

```

##### warp.h

```
#ifndef __WRAP_H_
#define __WRAP_H_
void perr_exit(const char *s);
int Accept(int fd, struct sockaddr *sa, socklen_t *salenptr);
int Bind(int fd, const struct sockaddr *sa, socklen_t salen);
int Connect(int fd, const struct sockaddr *sa, socklen_t salen);
int Listen(int fd, int backlog);
int Socket(int family, int type, int protocol);
ssize_t Read(int fd, void *ptr, size_t nbytes);
ssize_t Write(int fd, const void *ptr, size_t nbytes);
int Close(int fd);
ssize_t Readn(int fd, void *vptr, size_t n);
ssize_t Writen(int fd, const void *vptr, size_t n);
ssize_t my_read(int fd, char *ptr);
ssize_t Readline(int fd, void *vptr, size_t maxlen);
#endif

```

![image-20230927104030417](https://glf-1309623969.cos.ap-shanghai.myqcloud.com/img/image-20230927104030417.png)

### 多进程服务端

```
#include <stdio.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <string.h>
#include <unistd.h>
#include <signal.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <errno.h>
 
#define BUF_SIZE 256
 
void do_sigchld(int signo, siginfo_t *siginfo, void *p)
{
    int status;
    pid_t pid;
    while((pid = waitpid(0, &status, WNOHANG)) > 0) //0:回收跟调用进程同组的子进程的资源，WNOHANG:不阻塞
    {
        if (WIFEXITED(status))
            printf("pid = %d, parent pid = %d, child pid = %d, exit status %d\n", pid, getpid(), siginfo->si_pid, WEXITSTATUS(status));
        else if (WEXITSTATUS(status))
            printf("pid = %d, parent pid = %d, child pid = %d, exit by signal %d\n ", pid, getpid(), siginfo->si_pid, WIFSIGNALED(status));
    }
}
 
void sys_err(const char *str, int err)
{
    perror(str);
    exit(err);
}
 
int main(int argc, char *argv[])
{
    if (argc < 2)
    {
        printf("%s port\n", argv[0]);
        exit(1);
    }
    //创建流式套接字
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    if (lfd < 0)
        sys_err("socket", 1);
 
    //绑定ip端口    
    struct sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons((unsigned short)atoi(argv[1]));
    server_addr.sin_addr.s_addr = 0; //0表示将本机所有ip都绑定上
    int ret = bind(lfd, (struct sockaddr *)&server_addr, sizeof(server_addr));
    if (ret < 0)
        sys_err("bind", 1);
    
    //监听
    ret = listen(lfd, 128);
    if (ret < 0)
        sys_err("listen", 1);
    
    //阻塞SIGCHLD信号
    sigset_t set;
    sigemptyset(&set);
    sigaddset(&set, SIGCHLD);
    sigprocmask(SIG_BLOCK, &set, NULL);
    int block = 1;
    //循环提取
    int cfd;
    pid_t pid;
    struct sockaddr_in client_addr; 
    socklen_t len = sizeof(client_addr);
    char buf[BUF_SIZE];
    ssize_t size;
    char ip[INET_ADDRSTRLEN] = "";
    while(1)
    {
        cfd = accept(lfd, (struct sockaddr *)&client_addr, &len);
        if (cfd < 0)
        {
            if (errno == EINTR)
                continue;
 
            sys_err("accept", 1);
        }
 
        printf("client ip = %s, port = %d connect success\n", inet_ntop(AF_INET, &client_addr.sin_addr.s_addr, ip, sizeof(ip)), ntohs(client_addr.sin_port));
        pid = fork();
        if (pid == 0)
        {
            //in child
            close(lfd); //关闭不用了的监听套接字
            //解除阻塞SIGCHLD信号
            sigprocmask(SIG_UNBLOCK, &set, NULL);
            block = 0;
 
            while(1)
            {
                memset(buf, 0, sizeof(buf));
                size = read(cfd, buf, sizeof(buf));
                if (size == 0) //客户端断开连接
                {
                    printf("client close\n");
                    break;
                }
 
                printf("%s\n", buf);
                write(cfd, buf, size);
            }
 
            break;
        }
        else if (pid > 0)
        {
            //in parent
            close(cfd); //关闭不用了的跟客户端通讯的套接字
            if (1 == block)
            {
                //先捕捉
                struct sigaction sa;
                sa.sa_sigaction = do_sigchld;
                sigemptyset(&sa.sa_mask);
                sa.sa_flags = SA_SIGINFO;
                sigaction(SIGCHLD, &sa, NULL);
                //后解除阻塞
                sigprocmask(SIG_UNBLOCK, &set, NULL);
                block = 0;
            }
        }
        else
            sys_err("fork", 1);
    }
    
    //关闭套接字
    if (pid == 0)
        close(cfd);
    else if (pid > 0)
        close(lfd);
 
    return 0;
}
```

```h
new client ip=127.0.0.1 port=53766
w

new client ip=127.0.0.1 port=53768
q
```

`free_process` 函数是用于处理子进程的结束的信号处理函数。当子进程结束时，父进程会收到 `SIGCHLD` 信号，该信号会触发 `free_process` 函数。这个函数使用 `waitpid` 函数来回收子进程，避免它们成为僵尸进程。

#### 多线程服务端

```c
#include <stdio.h>
#include <pthread.h>
#include "wrap.h"

typedef struct c_info
{
    int cfd;
    struct sockaddr_in cliaddr;
}CINFO;

void* client_fun(void *arg);
int main(int argc, char *argv[]){

    if(argc < 2){
        printf("argc < 2  \n ./test 8000\n");
        return 0;
    }
    pthread_attr_t attr;
    pthread_attr_init(&attr);
    pthread_attr_setdetachstate(&attr,PTHREAD_CREATE_DETACHED); //设置线程的分离状态为分离态,线程在退出后会自动释放其资源，而不需要其他线程等待它结束。
    short port = atoi(argv[1]);
    int lfd = tcp4bind(port,NULL);  //创建套接字，绑定
    Listen(lfd,5);

    struct sockaddr_in cliaddr;
    socklen_t len = sizeof(cliaddr);
    CINFO *info;
    while(1){
        int cfd = Accept(lfd,(struct sockaddr *)&cliaddr,&len);
        char ip[16]="";
       // printf("new client ip=%s port=%d\n",inet_ntop(AF_INET,&cliaddr.sin_addr.s_addr,ip,16),ntohs(cliaddr.sin_port));

        pthread_t pthid;
        info = malloc(sizeof(CINFO));
        info->cfd = cfd;
        info->cliaddr = cliaddr;
       // pthread_create(&pthid,NULL,client_fun,info);
         pthread_create(&pthid,&attr,client_fun,info);
    }


    return 0;
}

void* client_fun(void *arg){
    CINFO *info = (CINFO *)arg;
    char ip[16];
    printf("new client ip=%s port=%d\n",inet_ntop(AF_INET,&info->cliaddr.sin_addr.s_addr,ip,16),ntohs(info->cliaddr.sin_port));
    while(1){
        char buf[1024]="";
        int n = read(info->cfd,buf,sizeof(buf));
        if(n < 0){
            perror("");
            break;
        }else if(n == 0){
            printf("client close\n");
            break;
        }else{
            printf("%s\n",buf);
            write(info->cfd,buf,n);
        }
    }
    close(info->cfd);
    free(info);
}
```

#### 端口复用

```c
int opt = 1;
setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
```

