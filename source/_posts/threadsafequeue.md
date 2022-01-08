---
title: C++线程安全队列
---
参考
https://www.cnblogs.com/mathyk/p/11572989.html

#include <queue>
#include <mutex>
#include <condition_variable>
#include <initializer_list>
#include <iostream>
#include <stdio.h>
#include <stdlib.h>
#include <pthread.h>
#include <errno.h>
#include <unistd.h>
using namespace std;
        /*
        * 线程安全队列
        * T为队列元素类型
        * 因为有std::mutex和std::condition_variable类成员,所以此类不支持复制构造函数也不支持赋值操作符(=)
        * */
        template<typename T>
        class threadsafe_queue{
        private:
            //data_queue访问信号量
            mutable std::mutex mut;
            mutable std::condition_variable data_cond;
            using queue_type = std::queue<T>;
            queue_type data_queue;
        public:
            using value_type = typename queue_type::value_type;
            using container_type = typename queue_type::container_type;
            threadsafe_queue() = default;
            threadsafe_queue(const threadsafe_queue&) = delete;
            threadsafe_queue& operator=(const threadsafe_queue&) = delete;
            /*
            * 使用迭代器为参数的构造函数,适用所有容器对象
            * */
            template<typename _InputIterator>
            threadsafe_queue(_InputIterator first, _InputIterator last){
                for (auto itor = first; itor != last; ++itor){
                    data_queue.push(*itor);
                }
            }
            explicit threadsafe_queue(const container_type &c) :data_queue(c){}
            /*
            * 使用初始化列表为参数的构造函数
            * */
            threadsafe_queue(std::initializer_list<value_type> list) :threadsafe_queue(list.begin(), list.end()){
            }
            /*
            * 将元素加入队列
            * */
            void push(const value_type &new_value){
                std::lock_guard<std::mutex>lk(mut);
                    data_queue.push(std::move(new_value));
                    data_cond.notify_one();
            }

            /*
            * 从队列中弹出一个元素,如果队列为空就阻塞
            * */
            value_type wait_and_pop(){
                std::unique_lock<std::mutex>lk(mut);
                data_cond.wait(lk, [this]{return !this->data_queue.empty(); });
                auto value = std::move(data_queue.front());
                data_queue.pop();
                return value;
            }
            /*
            * 从队列中弹出一个元素,如果队列为空返回false
            * */
            bool try_pop(value_type& value){
                std::lock_guard<std::mutex>lk(mut);
                if (data_queue.empty())
                    return false;
                value = std::move(data_queue.front());
                data_queue.pop();
                return true;
            }
            /*
            * 返回队列是否为空
            * */
            auto empty() const->decltype(data_queue.empty()) {
                std::lock_guard<std::mutex>lk(mut);
                return data_queue.empty();
            }
            /*
            * 返回队列中元素数个
            * */
            auto size() const->decltype(data_queue.size()){
                std::lock_guard<std::mutex>lk(mut);
                return data_queue.size();
            }
        }; /* threadsafe_queue */
struct readData
{
    char *data;
    int length;
}a,b;
threadsafe_queue<readData> test;
static void *func1(void *)/*1秒钟之后退出*/
{
    while(1){
        sleep(7);
        time_t now;    //time_t是一种类型，定义time_t类型的t
        struct tm *tm_now ;
        time(&now) ;
        tm_now = localtime(&now) ;//get date
        printf("func1 start datetime: %d-%d-%d %d:%d:%d\n",tm_now->tm_year+1900, tm_now->tm_mon+1, tm_now->tm_mday, tm_now->tm_hour, tm_now->tm_min, tm_now->tm_sec) ;

        // readData a;
        a.data = "asdf";
        a.length = 4;
        test.push(a);
        printf("test.size %d \n", test.size());
        time(&now) ;
        tm_now = localtime(&now) ;//get date
        printf("func1 end datetime: %d-%d-%d %d:%d:%d\n",tm_now->tm_year+1900, tm_now->tm_mon+1, tm_now->tm_mday, tm_now->tm_hour, tm_now->tm_min, tm_now->tm_sec) ;

        // printf("线程1（ID：0x%x）退出。\n",(unsigned int)pthread_self());
    }
    pthread_exit((void *)0);
}

static void *func2(void *)/*10秒钟之后退出*/
{
    while(1){
        sleep(4);
        time_t now;    //time_t是一种类型，定义time_t类型的t
        struct tm *tm_now ;
        time(&now) ;
        tm_now = localtime(&now) ;//get date
        printf("func2 start datetime: %d-%d-%d %d:%d:%d\n",tm_now->tm_year+1900, tm_now->tm_mon+1, tm_now->tm_mday, tm_now->tm_hour, tm_now->tm_min, tm_now->tm_sec) ;

        // if (false == test.try_pop(b))
        // {
        //     printf("Hello,func2 false == test.try_pop(b) \n");
        //     continue;
        // }
        // readData b;
        b = test.wait_and_pop();
        printf("b.data %s \n", b.data);
        time(&now) ;
        tm_now = localtime(&now) ;//get date
        printf("func2 end datetime: %d-%d-%d %d:%d:%d\n",tm_now->tm_year+1900, tm_now->tm_mon+1, tm_now->tm_mday, tm_now->tm_hour, tm_now->tm_min, tm_now->tm_sec) ;

        
    }
    
    pthread_exit((void *)0);
}
int main(void)
{
    time_t now;    //time_t是一种类型，定义time_t类型的t
    struct tm *tm_now ;
    time(&now) ;
    tm_now = localtime(&now) ;//get date
    printf("start datetime: %d-%d-%d %d:%d:%d\n",tm_now->tm_year+1900, tm_now->tm_mon+1, tm_now->tm_mday, tm_now->tm_hour, tm_now->tm_min, tm_now->tm_sec) ;
 
    pthread_t tid1,tid2;
    
    pthread_create(&tid1,NULL,&func1,NULL);
    pthread_create(&tid2,NULL,&func2,NULL);
    // time(&now) ;
    // tm_now = localtime(&now) ;//get date
    // printf("start datetime: %d-%d-%d %d:%d:%d\n",tm_now->tm_year+1900, tm_now->tm_mon+1, tm_now->tm_mday, tm_now->tm_hour, tm_now->tm_min, tm_now->tm_sec) ;
    pthread_join(tid1,NULL);
    pthread_join(tid2,NULL);
    readData a;
    a.data = "asdf";
    a.length = 4;
    test.push(a);

    printf("Hello,World， test.size %d \n", test.size());
    readData b;
    b = test.wait_and_pop();
    printf("Hello,World， b.data %s \n", b.data);
    printf("Hello,World， b.length %d \n", b.length);

    a.data = "1111";
    a.length = 5;
    test.push(a);
    printf("Hello,World， test.size %d \n", test.size());

    b = test.wait_and_pop();
    printf("Hello,World， b.data %s \n", b.data);
    printf("Hello,World， b.length %d \n", b.length);

    return 0;

}
运行

./queuethreadsafe.exe
start datetime: 2021-11-29 7:22:24
func2 start datetime: 2021-11-29 7:22:28
func1 start datetime: 2021-11-29 7:22:31
test.size 1
b.data asdf
func1 end datetime: 2021-11-29 7:22:31
func2 end datetime: 2021-11-29 7:22:31

