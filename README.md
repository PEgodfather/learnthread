

# learn thread

I am learning thread in 7.28.



#### 左值-“适合于赋值表达式左侧的值”

变量名、引用、指针等

左值可以取地址和赋值

分配了内存地址。

左值被分为两个子类:可修改的左值(可以更改)和不可修改的左值(const)。

```c++
// p、b、c都是左值
int* p = new int(0);
int b = 1;
const int c = 2;
// rp、rb、rc是上述左值的引用
int*& rp = p;
int& rb = b;
const int& rc = c;`
```



#### 右值

字面常量、表达式返回值、函数返回值

右值不能取地址，因为一般右值都是一些**临时变量**，比如函数返回值，函数执行完毕以后，会将返回值赋值给寄存器或者一个临时变量，我们无法获取一个临时变量的地址。

```c++
// 以下是常见的右值
10			// 常量
x + y		// 表达式的返回值
func(x, y)	// 函数返回值
 
// 以下依次给上述右值起别名
int&& rr1 = 10;
double&& rr2 = x + y;
double&& r33 = func(x, y);
```
##### (1) 左值引用能否引用右值？

举例： `int&  num = 10;`

成立与否：不成立

解决方法：添加`const`关键字修饰

`const int& num = 10;`

##### (2) 右值引用能否引用左值？

举例：`int&& r = a; // a 是一个变量`

成立与否：不成立

解决方法：使用`move`函数将左值转换成右值

`int a = 10;`
`int&& r = move(a);`



#### move()

就是将左值变成右值

```c++
//创建存放string的数组 和 一个字符串
std::vector<std::string> v;
std::string str = "Knock";

// 使用左值引用，将str复制到数组array中
v.push_back(str); 
std::cout << "str: " << str << '\n';    //str:Knock
std::cout << "vector: " << v[0] << '\n';//vector:Knock

// 使用右值引用，将str移动到数组array中
v.push_back(std::move(str)); 

std::cout << "str: " << str << '\n';                  //str:
std::cout << "vector:" << v[0] << ' ' << v[1] << '\n';//vector:Knock Knock
```
上述发现，还可以把变量的值完全移动到另外一个变量上。



### 多线程

- 进程是资源分配的最小单位
- 线程共享进程的栈空间，但是每个线程拥有独立的栈

### 重要：

#### join()

当使用了join()的子线程开始工作后，主线程结束了工作，**主线程会被阻塞，直到这个子线程运行完。**

#### detach()

当使用了detach()的子线程开始工作后，主线程结束了工作，**这个子线程也结束工作。**

join()和detach()由操作系统运行时库负责清理与被调线程相关的资源。

#### this_thread全局函数：

##### this_thread::get_id()

用于获取线程ID。

##### this_thread::sleep_for()

线程休眠。

~~~~~~c++
//方法1
chrono::milliseconds dura(1000);
this_thread::sleep_for(dura);
//方法2
this_thread::sleep_for(chrono::milliseconds(100));
~~~~~~

#### 普通函数创建线程

```c++
void fun(int ret) {
	cout << "我是第" << ret << "个”" << endl;
}
int main() {
	thread my(fun, 10);
	my.join();
}
```

#### lambda匿名函数类创建线程

```		c++
//第一种
auto f = [](int ret) {
	cout << "我是第" << ret << "个”" << endl;
};
thread my(f, 10);
my.join();
```

```c++
//第二种
thread my1([](int ret = 10){
               cout << "我是第" << ret << "个”" << endl;
           },
50000);//<--注意default和传入值

my1.join();
```

#### 类创建线程-仿函数创建线程

```c++
class stu {
public:
	void operator()(int ret);
};
void stu::operator()(int ret)
{
	cout << "我是第" << ret << "个" << endl;
}

int main() {
	stu s1;
	thread name(s1, 20);
	name.join();
	//注意用仿函数类名对象，必须带参数，如果无参只能用上面的不能用stu()
	thread name1(stu(), 30);
	name1.join();
}
```

#### 类的静态成员函数创建线程

```c++
class stu {
public:
	static void fun(int ret);
};
void stu::fun(int ret)//类成员函数类外写
{
	cout << "我是第" << ret << "个" << endl;
}

int main() {
	thread name(stu::fun, 30);
	name.join();
}
```

#### 类的普通成员函数

```c++
class stu {
public:
	void fun(int ret);
};
 void stu::fun(int ret)//类成员函数类外写
{
	cout << "我是第" << ret << "个" << endl;
}
int main() {
	stu s1;//必须先创建对象,生命周期一定要比子线程长
	thread name(&stu::fun, &s1, 300);//注意传进去的是指针
	name.join();
}
```



### 不重要：

#### joinable()

bool类型函数,它会表示当前的线程是否是可执行线程(能被join或者detach)。



