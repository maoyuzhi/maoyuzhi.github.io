---
title: linux使用非阻塞I/O的应用程序通常会使用select()
---
参考
https://blog.csdn.net/tayinyinyueyue/article/details/54629341


select使用：

#include <stdio.h> // printf
#include <stdlib.h> // exit
#include <unistd.h> // pipe
#include <string.h> // strlen
#include <pthread.h> // pthread_create
  
using namespace std;
bool threadQuit = false;
void *func(void * fd)
{
    // printf("write fd = %d\n", *(int*)fd);
    // char str[] = "hello everyone!";
    //write( *(int*)fd, str, strlen(str) );
    sleep(10);
    // for (int i = 0 ; i <10 ; i++)
    // {
    //     write( *(int*)fd, str, strlen(str) );
    // }
    sleep(5);
    printf("sleep(15) end write pipe!\n");
    printf("write fd = %d\n", *(int*)fd);
    char str[] = "hello everyone!";
    write( *(int*)fd, str, strlen(str) );
    // close(*(int*)fd);
    return 0;
}
void *func2(void * fd)
{
    printf("func2 threadQuit = %d\n", threadQuit);
    threadQuit = true;
    return 0;
}  
int main()
{
    int fd[2];
    char readbuf[1024];

    if(pipe(fd) < 0)
    {
        printf("pipe error!\n");
    }

    // create a new thread
    pthread_t tid ,t1;
    pthread_create(&tid, NULL, func, &fd[1]);
    //pthread_join(tid, NULL);

    //sleep(3);
    while(!threadQuit){
        fd_set rds;    //声明描述符集合
        FD_ZERO(&rds);   //清空描述符集合
        FD_SET(fd[0], &rds); //设置描述符集合

        struct timeval tv;
        tv.tv_sec = 10;
        tv.tv_usec = 0;
        int ret = select(fd[0] + 1, &rds, NULL, NULL, &tv);//调用select（）监控函数
        printf("ret = %d\n", ret);
        if (ret < 0)  
        {
            printf("select error!\n");
            exit(1);
        } else if ( 0 == ret)  
        {
            printf("no data within 10 seconds!\n");
            continue;
        }
        printf("read pipe start!\n");
        // read buf from child thread
        read( fd[0], readbuf, sizeof(readbuf) );
        printf("read buf = %s\n", readbuf);
        pthread_create(&t1, NULL, func2, NULL);
    }
    printf("abort while!\n");
    return 0;
}

10s select timeout
write,read
10s select timeout
退出循环，结束

g++ threadpipe.cpp -o threadpipe.exe
运行

$ ./threadpipe.exe
ret = 0
no data within 10 seconds!
sleep(15) end write pipe!
write fd = 4
ret = 1
read pipe start!
read buf = hello everyone!
func2 threadQuit = 0
ret = 0
no data within 10 seconds!
abort while!


