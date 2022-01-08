---
title: Linux线程退出方式
---

一：多线程安全退出

在编写多线程代码时，经常面临线程安全退出的问题。
一般情况下，选择检查标志位的方式：
在线程的while循环中，执行完例程后，都对标志位进行检查，
如果标志位指示继续执行则再次执行例程，
如果标志位设置为退出状态，则跳出循环，结束线程的运行。

这个标志位需要主线程（或其他线程）设置，
设置后，主线程调用pthread_join接口进入休眠（接口参数指定了等待的线程控制指针），
子线程退出后，主线程会接收到系统的信号，从休眠中恢复，这个时候就可以去做相关的资源清除动作。

这个方法可以保证子线程完全退出，主线程再去做相关的资源清除操作

时序图如下 
![sequence](thread1.png)

二：子线程阻塞

但是某些应用中，或许会发生下面情况：
子线程阻塞在某个操作无法被唤醒，即使主线程设置了标志位，
由于子线程进入了休眠无法醒过来，也没有办法去检查标志位，
这个时候调用pthread_join进入休眠的主线程等待不到子线程退出的信号，也会一直休眠，系统进入死锁。

为了更安全地使线程退出，
主线程通过pthread_cancel函数来请求取消同一进程中的其他线程，再调用pthread_join等待指定线程退出。
使用pthread_cancel接口，需要了解Linux下线程的两个属性，可取消状态和可取消类型，以及取消点的概念。

可取消状态：包括PTHREAD_CANCEL_ENABLE和PTHREAD_CANCEL_DISABLE。当线程处于PTHREAD_CANCEL_ENABLE，收到cancel请求会使该线程退出运行；反之，若处于PTHREAD_CANCEL_DISABLE，收到的cancel请求将处于未决状态，线程不会退出。线程启动时的默认可取消状态为PTHREAD_CANCEL_ENABLE，可以通过接口pthread_setcancelstate改变可取消状态的属性。

可取消类型：包括PTHREAD_CANCEL_DEFERRED和PTHREAD_CANCEL_ASYNCHRONOUS。当处于PTHREAD_CANCEL_DEFERRED，线程在收到cancel请求后，需要运行到取消点才能退出运行；如果处于PTHREAD_CANCEL_ASYNCHRONOUS，可以在任意时间取消，只要收到cancel请求即可马上退出。线程启动时默认可取消类型为PTHREAD_CANCEL_DEFERRED，可通过pthread_setcanceltype修改可取消类型。

取消点：线程检查是否被取消并按照请求进行动作的一个位置。

采用PTHREAD_CANCEL_DEFERRED取消方式是因为线程可能在获取临界资源后（如获取锁），未释放资源前收到退出信号，如果使用PTHREAD_CANCEL_ ASYNCHRONOUS的方式，无论线程运行到哪个位置，都会马上退出，而占有的资源却得不到释放。
采用PTHREAD_CANCEL_DEFERRED取消方式，线程需要运行到取消点才退出，而主线程在调用pthread_cancel后，不能马上进行线程资源释放，必须调用pthread_join进入休眠，直至等待指定线程退出。

使用PTHREAD_CANCEL_DEFERRED方式并不能完全避免这个问题，因为无法保证在获取临界资源后（比如lock操作）不会进行可以作为取消点的操作（如进行sleep），此时主线程如果对该线程发送cancel信号，线程将会在不释放锁的情况下直接结束运行，即还是会出现在释放资源前线程就退出的问题。
为了避免上述情况，不仅需要设置可取消类型，还需要设置可取消状态。将获取临界资源-释放临界资源之间的代码块都设置成PTHREAD_CANCEL_DISABLE状态，其余的代码块都设置成PTHREAD_CANCEL_ENABLE状态，确保线程在安全的地方退出。如果在可以安全退出的代码块不存在取消点系统调用，可以调用pthread_testcancel函数自己添加取消点。

无论使用哪种方式，核心点就是要保证线程退出的时候不会获取了某些临界资源而无法释放

注意：当主线程调用pthread_cancel接口后，只是将取消请求发送给指定线程，
对接口的成功调用不能保证指定线程已经退出，需要调用pthread_join等待指定线程完全退出，再进行相关资源的释放。
————————————————

版权声明：本文为CSDN博主「铁桶小分队」的原创文章，遵循CC 4.0 BY-SA版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/keheinash/article/details/50682435


代码如下
/******************************* pthread_kill.c *******************************/
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <errno.h>

void *func1()/*1秒钟之后退出*/
{
    sleep(1);
    printf("线程1（ID：0x%x）退出。\n",(unsigned int)pthread_self());
    pthread_exit((void *)0);
}

void *func2()/*10秒钟之后退出*/
{
    sleep(10);
    //while(1){}
    printf("线程2（ID：0x%x）退出。\n",(unsigned int)pthread_self());
    pthread_exit((void *)0);
}

void test_pthread(pthread_t tid) /*pthread_kill的返回值：成功（0） 线程不存在（ESRCH） 信号不合法（EINVAL）*/
{
    int pthread_kill_err;
    pthread_kill_err = pthread_kill(tid,0);

    if(pthread_kill_err == ESRCH)
        printf("ID为0x%x的线程不存在或者已经退出。\n",(unsigned int)tid);
    else if(pthread_kill_err == EINVAL)
        printf("发送信号非法。\n");
    else
        printf("ID为0x%x的线程目前仍然存活。\n",(unsigned int)tid);
}

int main()
{
    time_t now;    //time_t是一种类型，定义time_t类型的t
    struct tm *tm_now ;
    time(&now) ;
    tm_now = localtime(&now) ;//get date
    printf("start datetime: %d-%d-%d %d:%d:%d\n",tm_now->tm_year+1900, tm_now->tm_mon+1, tm_now->tm_mday, tm_now->tm_hour, tm_now->tm_min, tm_now->tm_sec) ;
    int ret;
    pthread_t tid1,tid2;
    
    pthread_create(&tid1,NULL,func1,NULL);
    pthread_create(&tid2,NULL,func2,NULL);
    
    sleep(3);/*创建两个进程3秒钟之后，分别测试一下它们是否还活着*/
    time(&now) ;
    tm_now = localtime(&now) ;//get date
    printf("sleep 3 datetime: %d-%d-%d %d:%d:%d\n",tm_now->tm_year+1900, tm_now->tm_mon+1, tm_now->tm_mday, tm_now->tm_hour, tm_now->tm_min, tm_now->tm_sec) ;
 
    test_pthread(tid1);/*测试ID为tid1的线程是否存在*/
    test_pthread(tid2);/*测试ID为tid2的线程是否存在*/
    
    time(&now) ;
    tm_now = localtime(&now) ;//get date
    printf("pthread_join start datetime: %d-%d-%d %d:%d:%d\n",tm_now->tm_year+1900, tm_now->tm_mon+1, tm_now->tm_mday, tm_now->tm_hour, tm_now->tm_min, tm_now->tm_sec) ;
    pthread_cancel(tid2);
    pthread_join(tid2,NULL);
    time(&now) ;
    tm_now = localtime(&now) ;//get date
    printf("pthread_join stop datetime: %d-%d-%d %d:%d:%d\n",tm_now->tm_year+1900, tm_now->tm_mon+1, tm_now->tm_mday, tm_now->tm_hour, tm_now->tm_min, tm_now->tm_sec) ;
 
    //exit(0);
}

运行结果
$ ./pthread_kill.exe
start datetime: 2021-11-20 10:43:40
线程1（ID：0x167e0）退出。
sleep 3 datetime: 2021-11-20 10:43:43
ID为0x167e0的线程不存在或者已经退出。
ID为0x168e0的线程目前仍然存活。
pthread_join start datetime: 2021-11-20 10:43:43
pthread_join stop datetime: 2021-11-20 10:43:43

如果不pthread_cancel(tid2);需要等待线程2退出
$ ./pthread_kill.exe
start datetime: 2021-11-20 10:29:31
线程1（ID：0x167e0）退出。
sleep 3 datetime: 2021-11-20 10:29:34
ID为0x167e0的线程不存在或者已经退出。
ID为0x168e0的线程目前仍然存活。
pthread_join start datetime: 2021-11-20 10:29:34
线程2（ID：0x168e0）退出。
pthread_join stop datetime: 2021-11-20 10:29:41
