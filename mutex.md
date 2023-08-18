# 锁

## 互斥锁-mutex

头文件：`#include <mutex>`

`lock()  ` 加锁

`unlock() ` 释放锁

不常用：

`try_lock()` 尝试加锁



**发生了死锁应该如何避免？**

**1.`lock_guard`**

使用`lock_guard` 可以有效避免死锁

使用方式：

~~~c++
#include <list>
#include <mutex>

std::list<int> some_list;
std::mutex some_mutex;

void add_to_list(int new_value)
{
    std::lock_guard<std::mutex> guard(some_mutex);
    some_list.push_back(new_value);
}
~~~

创建锁：`std::mutex some_mutex;`

使用lock_guard： `std::lock_guard<std::mutex> guard(some_mutex);`



**2.`unique_lock`**

unique_lock 可以像上面例子使用lock_guard一样使用，另外，unique_lock相比lock_guard有**更多操作**，例如，使用try_lock()方法尝试lock mutex，若lock失败（即mutex已经被其他线程lock），则直接跳过对临界区操作，而不是阻塞线程。

创建锁：`std::mutex some_mutex;`

使用unique_lock： `std::unique_lock<std::mutex> guard(some_mutex);`

~~~c++
std::mutex mutex_;
print_thread(){
    {
        std::unique_lock<std::mutex> lck(mutex_, std::defer_lock);//延迟加锁
        if(lck.try_lock()){
            std::cout<<"this is a thread"<<std::endl;
        }    
    }
};
~~~



## 自旋锁（不推荐）-spin

从“自旋锁”的名字也可以看出来，如果一个线程想要获取一个被使用的自旋锁，那么它会一致占用CPU请求这个自旋锁使得CPU不能去做其他的事情，直到获取这个锁为止，这就是“自旋”的含义。

当发生阻塞时，互斥锁可以让CPU去处理其他的任务；而**自旋锁让CPU一直不断循环请求获取这个锁**。通过两个含义的对比可以我们知道“自旋锁”是**比较耗费CPU的**。

~~~c++
#include<pthread.h>
#include <stdio.h>

pthread_spinlock_t spinlock;

// 测试
void *func(void *arg){
    int *pcount = (int *)arg;

    int i = 0;

    while (i++ < 100000){
        pthread_spin_lock(&spinlock);
        (*pcount)++;
        pthread_spin_unlock(&spinlock);
        usleep(1);
    }
}
int main(){
    pthread_t thid[THRHEAD_COUNT] = {0};
    int count = 0;
    int i = 0;

    pthread_spin_init(&spinlock, PTHREAD_PROCESS_SHARED);

    for (i = 0; i < THRHEAD_COUNT; i++){
        pthread_create(&thid[i], NULL, func, &count);
    }

    for (i = 0; i < 100; i++){
        printf("count --> %d\n", count);
        sleep(1);
    }
}
~~~



## 原子操作（原子锁）-atomic

当某次操作一旦开始，就一直运行到结束，中间不会有任何中断，这就是原子操作。

**原子操作性能（速度）强于互斥锁**

对于基本类型的临界资源，我们进行访问时可以用原子操作代替互斥锁，来提高性能。

头文件：`#include<atomic>`

使用方式：

创建临界资源（只能单个线程访问的资源）：`atomic<int> i(0)` //初始化i=0

使用简单的运算即可（**暂时不了解支持的操作有哪些**），比如：

~~~c++
void mythread(){
    for (int j = 0; j < 1000000; j++){
        i++;
    }
}
~~~

即使多个线程运行也不会出现共享资源操作问题。

**注意：`atomic` 既不可复制也不可移动。**

