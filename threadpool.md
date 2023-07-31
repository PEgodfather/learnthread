# Threadpool

有效降低多线程操作中任务申请和释放产生的性能消耗

**线程池通常适合下面的几个场合：**

(1) 单位时间内处理**任务频繁**而且任务**处理时间短**
(2) 对**实时性要求较高**。如果接受到任务后在创建线程，可能满足不了实时要求，因此必须采用线程池进行预创建。

### 首先理解:

#### void*

这是一种任意数据类型的指针。

当你的函数参数中有不确定类型的参数，你就可以使用这个，然后再进行强转。

比如这个：

~~~c++
void say(int type,void* pArgs) {
	switch (type) {
		case 0:{
			double* d = (double*)pArgs;
			break;
		}	
		case 1:{
			int* i = (int*)pArgs;
			break;
		}		
	}
}
~~~

注意：函数外部在接收到void*格式的返回值时，需要强转为自己的数据类型才能使用。

在c语言中用这个方式定义空指针：`#define NULL ((void*)0)`

在c++语言中为：

~~~c++
#ifndef NULL
	#ifdef __cplusplus
		#define NULL 0
	#else 
		#define NULL ((void*)0)
	#endif
#endif
~~~

在C++中**推荐**使用新标准的nullptr来初始化一个空指针。

C语言中NULL代表(void **)0，而在C++中NULL代表的是0，使用nullptr来表示(void* *)0空指针。

___

#### 回调函数

利用一个函数去调用另一个函数。

**普通函数的回调：**

~~~c++
#include <iostream>

void programA_FunA1() 
{ printf("I'am ProgramA_FunA1 and be called..\n"); }
void programA_FunA2() 
{ printf("I'am ProgramA_FunA2 and be called..\n"); }
void programB_FunB1(void (*callback)()) {
  printf("I'am programB_FunB1 and be called..\n");
  callback();
}

int main(int argc, char **argv) {
  programA_FunA1();
  //programA_FunA2是回调函数
  programB_FunB1(programA_FunA2);//programB_FunB1将programA_FunA2作为回调函数
}
~~~

**类中的成员函数回调：**

~~~c++
#include <iostream>

class ProgramA {
 public:
  void FunA1() { printf("I'am ProgramA.FunA1() and be called..\n"); }

  void FunA2() { printf("I'am ProgramA.FunA2() and be called..\n"); }
};

class ProgramB {
 public:
  void FunB1(void (ProgramA::*callback)(), void *context) {
    printf("I'am ProgramB.FunB1() and be called..\n");
    ((ProgramA *)context->*callback)();
  }
};

int main(int argc, char **argv) {
  ProgramA PA;
  PA.FunA1();

  ProgramB PB;
  PB.FunB1(&ProgramA::FunA2, &PA);  // 此处都要加&，必须得这么写
}

~~~

+++

#### static_cast

强制类型转换操作符

举例：

~~~c++
double a = 1.999;
int b = static_cast<double>(a); //相当于a = b ;
~~~

当编译器隐式执行类型转换时，大多数的编译器都会给出一个警告：

~~~c++
e:\vs 2010 projects\static_cast\static_cast\static_cast.cpp(11): warning C4244: “初始化”: 从“double”转换到“int”，可能丢失数据
~~~

使用static_cast可以明确告诉编译器，这种损失精度的转换是在知情的情况下进行的，也可以让阅读程序的其他程序员明确你转换的目的不是由于疏忽。

**使用static_cast可以找回存放在void\*指针中的值。**

~~~c++
double a = 1.999;
void * vptr = & a;
double * dptr = static_cast<double*>(vptr);
cout<<*dptr<<endl;//输出1.999
~~~

---

#### notify_one()与notify_all()

头文件`#include <condition_variable>`

通常配合`#include <mutex>`使用

当一个函数中使用了wait来阻塞线程时，在另一个函数中就可以使用notify_one()与notify_all()来唤醒阻塞线程。

比如：

~~~c++
#include <iostream>           // std::cout
#include <thread>             // std::thread
#include <mutex>              // std::mutex, std::unique_lock
#include <condition_variable> // std::condition_variable
 
std::mutex mtx;
std::condition_variable cv;
bool ready = false;

void print_id(int id) {
	std::unique_lock<std::mutex> lck(mtx);
	while (!ready) cv.wait(lck);
	// ...
	std::cout << "thread " << id << '\n';
}
 
void go() {
	std::unique_lock<std::mutex> lck(mtx);
	ready = true;
	cv.notify_all(); // 这是重点
}
 
int main()
{
	std::thread threads[10];
	// spawn 10 threads:
	for (int i = 0; i < 10; ++i)
		threads[i] = std::thread(print_id, i);
 
	std::cout << "10 threads ready to race...\n";
	go();                       // go!
 
	for (auto& th : threads) th.join();
 
	return 0;
}
~~~

输出结果：

~~~C++
10 threads ready to race...
thread 2
thread 0
thread 9
thread 4
thread 6
thread 8
thread 7
thread 5
thread 3
thread 1
~~~

当使用notify_one()时，结果：

~~~c++
10 threads ready to race...
thread 0
~~~

只会唤醒一个线程。

---

### 线程池构建：

构建线程池首先应该明白线程池的工作原理。

普通的线程池至少包含**任务队列**与**工作线程组**。

<img src="G:\Desktop\vscode-code\git_projecct\thread728\1.png" style="zoom:50%;" />

我在普通线程池上增加了**管理者**，目的是为了调整线程数量，使线程池更灵活与高效。

<img src="G:\Desktop\vscode-code\git_projecct\thread728\2.png" alt="2" style="zoom:50%;" />





#### mythreadpool.h文件

~~~c++
#pragma once
#ifndef __MYTHREADPOOL_H__
#define __MYTHREADPOOL_H__

#include<iostream>
#include<thread>
#include<mutex>
#include<condition_variable>
#include<queue>
#include<vector>
#include<stdlib.h>
using namespace std;

#define MAXADD 2
#define MAXDEL 2

class Task {
public:
	void (*func)(void*);
	void* event;
	Task(void (*f)(void*), void* e) {
		func = f;
		event = e;
	}
};

class MyThreadPool {
private:
	queue<Task> jobs;
	vector<thread> ren;
	thread manager;
	int minren;
	int maxren;
	int workingren;
	int nowren;
	int exitren;
	mutex suo;
	condition_variable baoan;
	bool isStop;
	static void managerRun(void* arg);
	static void findJob(void* arg);


public:
	MyThreadPool(int min, int max);
	~MyThreadPool();
	void addTask(void (*f)(void*), void* e);
};

#endif
~~~



1. `MyThreadPool::managerRun(void* arg)`：这是管理者线程的运行函数。它会定期检查任务队列和线程池的状态，根据任务数量和线程数量的比例，决定是否需要创建新的线程或销毁空闲的线程。

2. `MyThreadPool::findJob(void* arg)`：这是工作线程的运行函数。它会持续寻找任务队列中的任务并执行。如果任务队列为空，线程会进入等待状态，直到有新的任务添加到队列中。

3. `MyThreadPool::MyThreadPool(int min,int max)`：这是线程池的构造函数。它会初始化线程池，并创建指定数量的工作线程和一个管理者线程。

4. `MyThreadPool::~MyThreadPool()`：这是线程池的析构函数。它会设置线程池的停止标志，并回收所有的工作线程和管理者线程。

5. `MyThreadPool::addTask(void (*f)(void*), void* e)`：这是添加任务的函数。它会将新的任务添加到任务队列，并唤醒一个等待的工作线程。

工作流程如下：

1. 创建线程池，指定最小和最大线程数量。
2. 添加任务到线程池，任务会被添加到任务队列中。
3. 工作线程会从任务队列中取出任务并执行。如果任务队列为空，工作线程会进入等待状态。
4. 管理者线程会定期检查任务队列和线程池的状态，根据需要创建新的线程或销毁空闲的线程。
5. 当线程池被销毁时，所有的工作线程和管理者线程会被回收。 

#### mythreadpool.cpp文件

~~~c++
#include "mythreadpool.h"

// 管理者线程运行的函数
void MyThreadPool::managerRun(void* arg) {
	// 将参数转换为线程池对象
	MyThreadPool* pool = static_cast<MyThreadPool*>(arg);
	// 当线程池没有停止时，持续运行
	while (!pool->isStop) {
		// 每两秒钟检查一次
		this_thread::sleep_for(chrono::seconds(2));
		// 锁定线程池
		unique_lock<mutex> lk(pool->suo);

		// 获取当前任务数量，当前线程数量，最大线程数量
		int jobsize = pool->jobs.size();
		int now = pool->nowren;
		int max = pool->maxren;
		int addcount = 0;
		// 解锁线程池
		lk.unlock();

		// 如果任务数量大于当前线程数量且当前线程数量小于最大线程数量
		if (jobsize > now && now < max) {
			// 锁定线程池
			lk.lock();
			// 添加新的线程
			for (int i = 0; i < pool->maxren && addcount < MAXADD && pool->nowren < pool->maxren; ++i) {
				// 如果当前线程为空
				if (pool->ren[i].get_id() == thread::id()) {

					// 创建新的线程
					pool->ren[i] = thread(findJob, pool);
					addcount++;
					pool->nowren++;
				}
			}
			// 解锁线程池
			lk.unlock();
		}
		// 锁定线程池
		lk.lock();
		// 如果工作线程数量的两倍小于当前线程数量且当前线程数量大于最小线程数量
		if (pool->workingren * 2 < pool->nowren && pool->nowren > pool->minren) {
			// 准备销毁线程
			cout << "准备销毁" << " " << pool->workingren * 2 << " " << pool->nowren << " " << endl;
			pool->exitren = MAXDEL;
			// 唤醒所有线程
			pool->baoan.notify_all();
		}
		// 解锁线程池
		lk.unlock();
	}
}

// 寻找任务的函数
void MyThreadPool::findJob(void* arg) {
	// 将参数转换为线程池对象
	MyThreadPool* pool = static_cast<MyThreadPool*>(arg);

	// 持续寻找任务
	while (1) {
		// 锁定线程池
		unique_lock<mutex> lk(pool->suo);

		// 如果任务队列为空且线程池没有停止
		while (pool->jobs.empty() && !pool->isStop) {
			// 等待任务
			pool->baoan.wait(lk);
			// 销毁线程
			if (pool->exitren > 0) {
				pool->exitren--;
				if (pool->nowren > pool->minren) {
					pool->nowren--;
				}
				// 解锁线程池
				lk.unlock();
				return;
			}
		}
		// 如果线程池停止
		if (pool->isStop) {
			// 关闭当前线程
			cout << "线程池即将关闭，当前线程" << this_thread::get_id() << "关闭..." << endl;
			return;
		}

		// 获取任务
		Task task = pool->jobs.front();
		pool->jobs.pop();
		pool->workingren++;
		// 解锁线程池
		lk.unlock();

		// 执行任务
		cout << "thread: " << this_thread::get_id() << " start working..." << endl;
		task.func(task.event);
		free(task.event);
		task.event = nullptr;

		// 任务执行完毕，解锁线程池
		lk.lock();
		pool->workingren--;
		cout << "thread: " << this_thread::get_id() << " end working...当前还有" << pool->workingren << "个线程在工作" << endl;
		lk.unlock();

	}
}

// 线程池构造函数
MyThreadPool::MyThreadPool(int min, int max) {
	minren = min;
	maxren = max;
	workingren = 0;
	nowren = min;
	exitren = 0;
	isStop = false;

	// 创建管理者线程
	manager = thread(managerRun, this);

	// 初始化线程数组
	ren.resize(max);
	for (int i = 0; i < min; ++i) {
		// 创建工作线程
		ren[i] = thread(findJob, this);
	}
}

// 线程池析构函数
MyThreadPool::~MyThreadPool() {
	isStop = true;
	// 回收管理者线程
	if (manager.joinable()) manager.join();
	// 唤醒所有线程
	baoan.notify_all();
	for (int i = 0; i < maxren; ++i)
	{
		// 回收工作线程
		if (ren[i].joinable()) ren[i].join();
	}
}

// 添加任务
void MyThreadPool::addTask(void (*f)(void*), void* e) {
	unique_lock<mutex> lk(suo);
	if (isStop)
	{
		return;
	}
	// 添加任务到任务队列
	jobs.push(Task(f, e));
	// 唤醒一个线程
	baoan.notify_one();
}

~~~

#### 如何使用

首先创建MyThreadPool对象，例如我想要创建5~10个线程在线程池中（线程在线程池中动态切换数量）：

~~~c++
MyThreadPool pool(5,10);
~~~

创建一个任务类，类中创建一个回调函数，如：

~~~c++
class myTask{
  void run(){
      cout<<"任务"<<endl;
  }  
};
~~~

创建一个任务函数func()，如：

~~~c++
void func(void* a) {
	myTask* mt = static_cast<myTask*> (a);
	mt->run();
}
~~~

然后使用添加任务的函数addTask()，例如：

~~~c++
pool.addTask(func,new myTask);
~~~

完成以上操作即可使线程池开始工作。



完整代码：

~~~c++
#include"MyThreadPool.h"

class myTask {
public:
	void run() {
		cout << "任务" << endl;
	}
};
void func(void* a) {
	myTask* mt = static_cast<myTask*> (a);
	mt->run();
}

int main() {
	MyThreadPool pool(5,10);
	
	int N = 100;
	while (N--) {
		this_thread::sleep_for(chrono::milliseconds(100));//等待100ms
		pool.addTask(func, new myTask);
	}

	return 0;
}
~~~

