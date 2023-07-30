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

