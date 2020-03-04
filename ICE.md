### ICE笔记

##### 一、编译

两个主要参数：

​		supported-languages：支持的语言设置

​		supported-configs：库类型设置

可以通过修改：config/Make.rules文件或者直接命令行参数的方式修改，默认支持大多数语言，库类型为shared，只编译C++98动态库。

命令行示例：支持c++和java，库版本为支持C++98的动态库和支持C++11的动态库

make supported-languages='cpp java' supported-configs='shared cpp11-shared'



##### 二、slice2cpp

参数--impl-c++11可以直接生成支持C++11的lambda表达式的服务端实现：

./slice2cpp --impl-c++11 Hello.ice

![1569576222904](ICE\1569576222904.png) 

**amd**：支持服务端异步

**ami**：支持客户端异步   //新版好像不需要了，都会产生同步和异步两种调用函数

![1569576360212](ICE\1569576360212.png) 



##### 三、ICE线程池

配置文件

```#
#
# thread pool
#
Ice.ThreadPool.Client.Size=1		#客户端默认线程数
Ice.ThreadPool.Client.SizeMax=1		#客户端最大线程数
Ice.ThreadPool.Server.Size=1		#服务端默认线程数
Ice.ThreadPool.Server.SizeMax=1		#服务端最大线程数
```



##### 四、需要ice的模块编译时要增加编译选项，支持C++11

​	在模块的CMakeList.txt里面增加下面选项：

​	add_definitions(-DICE_CPP11_MAPPING)