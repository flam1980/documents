## EOS中使用的C++11特性



#### 一、线程支持库

##### 	1.thread



##### 	2.锁和管理器：mutex，lock_guard，unique_lock

**mutex**：是能用于保护共享数据免受从多个线程同时访问的同步原语。`mutex` 提供排他性非递归所有权语义。`std::mutex` 既不可复制亦不可移动。通常不直接使用 `std::mutex` ：使用 std::unique_lock、std::lock_guard 或 std::scoped_lock (C++17 起)以更加异常安全的方式管理锁定。

**lock_guard**： 是互斥体包装器，为在作用域块期间占有互斥提供便利机制。创建 `lock_guard` 对象时，它试图接收给定互斥的所有权。控制离开创建 `lock_guard` 对象的作用域时，销毁 `lock_guard` 并释放互斥。`lock_guard` 类不可复制。

**unique_lock**：是通用互斥包装器，允许延迟锁定、锁定的有时限尝试、递归锁定、所有权转移和与条件变量一同使用。类 unique_lock 可移动，但不可复制。



##### 	3.future：异步任务

类模板 `std::future` 提供访问异步操作结果的机制：

```c++
#include <iostream>
#include <future>
#include <thread>
 
int main()
{
    // 来自 packaged_task 的 future
    std::packaged_task<int()> task([](){ return 7; }); // 包装函数
    std::future<int> f1 = task.get_future();  // 获取 future
    std::thread(std::move(task)).detach(); // 在线程上运行
 
    // 来自 async() 的 future
    std::future<int> f2 = std::async(std::launch::async, [](){ return 8; });
 
    // 来自 promise 的 future
    std::promise<int> p;
    std::future<int> f3 = p.get_future();
    std::thread( [&p]{ p.set_value_at_thread_exit(9); }).detach();
 
    std::cout << "Waiting..." << std::flush;
    f1.wait();
    f2.wait();
    f3.wait();
    std::cout << "Done!\nResults are: "
              << f1.get() << ' ' << f2.get() << ' ' << f3.get() << '\n';
}
```



#### 二、atomic原子操作库

​	https://zh.cppreference.com/w/cpp/atomic/atomic

​	

#### 三、lambda和std::function

##### 	1.lambda

参看： https://zh.cppreference.com/w/cpp/language/lambda

**语法**

|                                                              |      |      |
| ------------------------------------------------------------ | ---- | ---- |
| `**[**` *捕获* `**]**` *<模板形参>*(可选)(C++20) `**(**` *形参* `**)**` *说明符*(可选) *异常说明* *attr* `**->**` *ret* *requires*(可选)(C++20) `**{**` *函数体* `**}**` | (1)  |      |
|                                                              |      |      |
| `**[**` *捕获* `**]**` `**(**` *形参* `**)**` `**->**` *ret* `**{**` *函数体* `**}**` | (2)  |      |
|                                                              |      |      |
| `**[**` *捕获* `**]**` `**(**` *形参* `**)**` `**{**` *函数体* `**}**` | (3)  |      |
|                                                              |      |      |
| `**[**` *捕获* `**]**` `**{**` *函数体* `**}**`              | (4)  |      |
|                                                              |      |      |

 **Lambda 捕获**

*捕获* 是零或更多*捕获符*的逗号分隔列表，可选地以 *默认捕获符* 开始。仅有的默认捕获符是

- `**&**`（以引用隐式捕获被使用的自动变量）和
- `**=** `（以复制隐式捕获被使用的自动变量）。

当出现任一默认捕获符时，都能隐式捕获当前对象（`*this`）。当它被隐式捕获时，始终被以引用捕获，即使默认捕获符是 `=` 也是如此。当默认捕获符为 `=` 时，`*this` 的隐式捕获被弃用。 (C++20 起)



**lambda中引用捕捉的重命名：**

&修饰捕捉变量：声明该变量是一个引用，使用=给该引用捕捉赋值。

```c++
#include <iostream>

int main() {
    int a = 10;
    auto func = [&refa = a]() {
        refa++;
    };
    func();
    std::cout << a << std::endl;
    
    return 0;
}
```



##### 	2.std::function

​	lambda匿名函数打包为一个函数对象，可以如下方式从lambda构造function对象trigger_next_or_finalize：	![image-20200304104556151](images\image-20200304104556151.png)

**函数对象的存储与普通对象一样**：

![image-20200304105023549](images\image-20200304105023549.png)

**函数对象的使用与函数的使用一样：**

![image-20200304105706532](C:\Users\yl\AppData\Roaming\Typora\typora-user-images\image-20200304105706532.png)

#### 四、智能指针

##### 	1.unique_ptr 

##### 	2.shared_ptr和weak_ptr

​	