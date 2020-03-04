# FC_REFLECT反射宏教程

## 1.背景知识

### 1.1.元语言和目标语言

目标语言：一种描述最终执行任务的编程语言。

元语言：由于目标语言本身也是一种计算机程序，元语言是描述目标语言全部或部分语法规则的语言。

除了从事编译器开发工作，需要将元语言和目标语言进行分离之外，大部分情况下，元语言和目标语言使用的都是同一种编程语言，这在概念上容易混淆，使得对元语言这一块内容难以理解。

在c++中，模板和宏其实属于元语言。

### 1.2.编译器任务中的计算象限

大部分包括c++在内的编程语言，在执行编译和运行的过程中，存在四种计算象限

第一象限：执行期计算

第二象限：编译期计算

第三象限：异构数值计算

第四象限：类型推导计算

发生在第一象限上的计算比较容易理解，因为这是每一个开发者写代码的最终目的。

发生在第二象限上的计算，主要包括一些数值计算的优化，例如定义int a=2+3；那么到第一象限编译的时候，出来的结果是int a=5；其中2+3是在编译期间直接计算出结果，也就是常说的常量折叠。

第三和第四计算象限通常较为抽象，都和模板的推演相关，在理解上有一定的困难。

发生在第三象限上的计算，被称为异构计算，计算使用的是可以存储不同类型的容器对象，例如c++中的tuple。此外使用的函数也是异构函数，这是一种讨论模板函数的复杂方式。用于此类型计算的主要有boost::fusion，如以下例子所示：

```c++
auto to_string=[](auto t){
    std::stringstream ss;
    ss<<t;
    return ss.str();
};

fusion::vector<int,std::string,float> seq{1,"abc",3.4f};
fusion::vector<std::string,std::string,std::string> strings=fusion::transform(seq,to_string);

static_assert(strings==funsion::make_vector("1"s,"abc"s,"3.4"s));
```

发生在第四象限上的计算，是类型计算，使用类型容器，类型函数（也称元含刷）和类型算法。容器存储的是类型或元函数接收的类型作为参数，返回的结果也是类型。用于此类型计算的主要有boost::mpl，如下例子所示：

```c++
template<typename T>
struct add_const_pointer{
    using type=T const*;
};

using types=mpl::vector<int,char,float,void>;
using pointers=mpl::transform<types,add_const_pointer<mpl::_1>>::type;

static_assert(mpl::equal<pointers,
    mpl::vecotr<int const*,char const*,float const*,void const*>>::value,"");
```

如果硬要说存在第五象限的话，那么在c++中，可以把宏的展开也算上。

在编译到执行这个过程，编译器处理的顺序是从高象限到低象限，即宏展开，类型推导，异构数值计算，编译期数值计算与常量折叠，目标程序的执行。

### 1.3.λ演算与编译器发展趋势

由于长期以来，第三象限和第四象限的计算，一直是难以理解又异常神秘的，于是便有人专门将其从编译原理中剥离出来，成为一个独立的课题进行研究，即λ演算。

从c++11开始，c++语言出来扩充基本库之外，便开始在语法层面大量引入和支持λ演算。尽管如此，λ演算对c++来说，依然是弱项，所以我们不得不自己去编写大量底层的代码来实现。

目前，应用于λ演算的语义常见的主要有三种：即公理语义，指称语义和操作语言。而再c++中，使用的是操作语义。

| 语义名称 |             语义介绍             |                 语义优势                 |                          语义劣势                          |                        应用语言                         |
| :------: | :------------------------------: | :--------------------------------------: | :--------------------------------------------------------: | :-----------------------------------------------------: |
| 公理语义 | 基于数学集合论公理推导的处理流程 |  逻辑严密，语法上过了，不太容易出现bug   |         通常不具备图灵完备性，为避免停机问题的论证         | python（不完全），javascript（不完全），haskell（完全） |
| 指称语义 |    基于谓词逻辑推导的处理流程    | 比较容易描述知识结构，适用于知识库的建模 |               比较反人类，使用困难，阅读困难               |                    prolog（较罕见）                     |
| 操作语义 |   基于有限状态机推导的处理流程   |     传统编程语言常用的模式，理解容易     | 如果语义过于复杂，会造成状态库非常庞大，工作量大且难以维护 |               c，c++，java等常见编程语言                |

## 2.c++反射机制原理解析

### 2.1.关键技术简介

由于C++语言，自身并不具备类型反射这个机制，即使在c++中，提供了decltype和type_id这两个简陋的反射功能，但是无法满足我们的需求，主要存在以下几个问题：

1.decltype仅仅能根据变量复制变量的类型，不能获取变量内部的信息

2.typeid可以根据变量创建一个type_info类型，尽管能获取变量的一些简单信息，但由于type_info尚未形成统一的标准，这对跨平台和跨编译器带来巨大的隐患。

3.在c++标准中，尚不存在一些方法可以访问结构体或者对象内部元素的泛型方法。

所以，在FC_REFLECT中，完全从底层自己实现了一套反射方案，并保证了这套方案具备跨平台有特点。该方案使用了以下几类技术：

1.访问者模式，通过这个模式来获取结构体或对象内部的相关信息

2.模板特例化，为每一个基本类型，编写一个简单的模板特例，该特例返回一个类型名称字符串

3.宏的#运算符，将一个标识符转换成字符串

4.宏的##运算符，将两个标识符拼接成一个新的标识符

5.宏的迭代展开

6.仿函数，通过重载()运算符，是对象具备普通函数的特点

### 2.2.宏的迭代展开

我们先来看以下代码：

```c++
struct Test{
    int a;
    float b;
    char c;
};
FC_REFLECT(Test,(a)(b)(c));
```

这是FC_REFLECT的一个例子，我们可以看到，为了能实现结构体内部元素的访问，首先要将结构体内每一个元素添加到FC_REFLECT中，使其结构体名称和内部元素形成关联性。

接下来，我们可以看一下boost中的一个迭代宏，BOOST_PP_SEQ_FOR_EACH，这是实现反射的关键，我们先来看这个宏的使用方法:

```c++
#include <boost/preprocessor/cat.hpp>
#include <boost/preprocessor/seq/for_each.hpp>

#define SEQ (w)(x)(y)(z)
#define MACRO(r, data, elem) BOOST_PP_CAT(elem, data)
BOOST_PP_SEQ_FOR_EACH(MACRO, _, SEQ);
// 一次展开
MACRO(2,_,w) MACRO(3,_,w) MACRO(4,_,w) MACRO(5,_,w)
// 二次展开
_w _x _y _z
```

依照这个用法，我们可以首先定义一个简单的反射宏，我们把这个反射宏名字叫做REFLECT_V1，具体写法如下：

```c++
template<typename Object>
void visitor_object(Object& obj){}

template<typename Object, typename Type)>
void visitor_elem(Object &obj){
	std::cout<<obj.*Member<<std::endl;
}

#dfefine VISITOR_V1(r,type,elem) \
	void visit_elem<type,decltype(obj.elem),&type::elem>(obj);

#define REFLECT_V1(TYPE, MEN) \
	template<> \
	void visitor_object<TYPE>(TYPE& obj){ \
		BOOST_PP_SEQ_FOR_EACH(VISITOR_V1,TYPE,MEM) \
	}

// 现在我们来用这个宏，展开上面定义的结构体
REFLECT_V1(Test,(a)(b)(c))

// 展开后得到

template<>
void visitor_object<Test>(Test& obj){
	VISITOR_V1(2,Test,a) VISITOR_V1(3,Test,b) VISITOR_V1(4,Test,c)
}

// 二次展开得到
template<>
void visitor_object<Test>(Test& obj){
	visit_elem<Test,decltype(obj.a),&Test::a>(obj);
	visit_elem<Test,decltype(obj.b),&Test::b>(obj);
	visit_elem<Test,decltype(obj.c),&Test::c>(obj);
}

// 最后我们要通过调用visitor_object<Test>的模板特例，遍历打印结构体中的元素
Test t{1,2.0 3};
visitor_object<Test>(t);
```

以上是一个简单的反射实现方式，可以看到，其中BOOST_PP_SEQ_FOR_EACH这个宏，是实现反射的关键。

### 2.3.获取结构体中，每一个元素的变量名称和类型名称

之前的例子，虽然能够遍历结构体中的每一个元素，并且获取其中的值，但是我们很难去判断这个值是那一个变量的，这一节将讲述如何把每一个变量跟值关联起来，以及将每一个类型的名称进行打印。

首先是如何获取类型名称，在第一节的时候，我们讨论到，在c++中有一个typeid的操作符，可以反射类型的相关信息，并且其中有name()这个方法可以获取变量的名称字符串，照理说，我们可以直接使用，像这样：

```
int a=5;
std::cout<<typeid(a).name()<<std::endl;
```

在windows上，可以打印出int这个字符串，但是在linux上却只打印一个i。由于c++尚未制定这个机制的标准，所以在不同编译器不同系统上，输出的信息也不一致，这给我们的跨平台跨编译器开发带来了很大的问题，所以不能使用这个机制去实现它。

FC_REFLECT中，通过对基本类型的硬编码来实现了根据这个功能，具体代码如下所示：

```c++
namespace fc {
  class value;
  class exception;
  namespace ip { class address; }

  template<typename... T> struct get_typename;
  template<> struct get_typename<int8_t>   { static const char* name()  { return "int8_t";   } };
  template<> struct get_typename<uint8_t>  { static const char* name()  { return "uint8_t";  } };
  template<> struct get_typename<int16_t>  { static const char* name()  { return "int16_t";  } };
  template<> struct get_typename<uint16_t> { static const char* name()  { return "uint16_t"; } };
  template<> struct get_typename<int32_t>  { static const char* name()  { return "int32_t";  } };
  template<> struct get_typename<uint32_t> { static const char* name()  { return "uint32_t"; } };
  template<> struct get_typename<int64_t>  { static const char* name()  { return "int64_t";  } };
  template<> struct get_typename<uint64_t> { static const char* name()  { return "uint64_t"; } };
  template<> struct get_typename<__int128>          { static const char* name()  { return "int128_t";  } };
  template<> struct get_typename<unsigned __int128> { static const char* name()  { return "uint128_t"; } };

  template<> struct get_typename<double>   { static const char* name()  { return "double";   } };
  template<> struct get_typename<float>    { static const char* name()  { return "float";    } };
  template<> struct get_typename<bool>     { static const char* name()  { return "bool";     } };
  template<> struct get_typename<char>     { static const char* name()  { return "char";     } };
  template<> struct get_typename<void>     { static const char* name()  { return "char";     } };
  template<> struct get_typename<string>   { static const char* name()  { return "string";   } };
  template<> struct get_typename<value>    { static const char* name()   { return "value";   } };
  template<> struct get_typename<fc::exception>   { static const char* name()   { return "fc::exception";   } };
  template<> struct get_typename<std::vector<char>>   { static const char* name()   { return "std::vector<char>";   } };
  template<typename T> struct get_typename<std::vector<T>>
  {
     static const char* name()  {
         static std::string n = std::string("std::vector<") + get_typename<T>::name() + ">";
         return n.c_str();
     }
  };
  template<typename T> struct get_typename<flat_set<T>>
  {
     static const char* name()  {
         static std::string n = std::string("flat_set<") + get_typename<T>::name() + ">";
         return n.c_str();
     }
  };
  template<typename T> struct get_typename< std::deque<T> >
  {
     static const char* name()
     {
        static std::string n = std::string("std::deque<") + get_typename<T>::name() + ">";
        return n.c_str();
     }
  };
  template<typename T> struct get_typename<optional<T>>
  {
     static const char* name()  {
         static std::string n = std::string("optional<") + get_typename<T>::name() + ">";
         return n.c_str();
     }
  };
  template<typename K,typename V> struct get_typename<std::map<K,V>>
  {
     static const char* name()  {
         static std::string n = std::string("std::map<") + get_typename<K>::name() + ","+get_typename<V>::name()+">";
         return n.c_str();
     }
  };

  struct signed_int;
  struct unsigned_int;
  template<> struct get_typename<signed_int>   { static const char* name()   { return "signed_int";   } };
  template<> struct get_typename<unsigned_int>   { static const char* name()   { return "unsigned_int";   } };

}
```

由此，对于普通类型，我们可以直接调用get_typename<类型名称>::name()就能获取类型名称，而对于自定义类型，我们可以在反射宏中添加如下的内容，

```c++
#define REFLECT_V1(TYPE, MEN) \
	template<> \
	void visitor_object<TYPE>(TYPE& obj){ \
		BOOST_PP_SEQ_FOR_EACH(VISITOR_V1,TYPE,MEM) \
	} \
	template<> struct get_typename<TYPE> { static const char* name() { return #TYPE; } };
```

这个关键在于宏运算符#，这个运算符的作用，就是将宏的输入参数编程运算符，我们来看以下例子：

```c++
#define STR(x) #x
STR(1234abcd) //展开结果为"1234abcd"
```

所有，要获取变量名称和参数，我们只需做如下的修改

```c++
template<typename Object>
void visitor_object(Object& obj){}

template<typename Object, typename Type)>
void visitor_elem(const char* type,const char* var,Object &obj){
	std::cout<<"name:"<<type<<std::endl;
	std::cout<<"var:"<<var<<std::endl;
	std::cout<<"value:"<<obj.*Member<<std::endl;
}

#dfefine VISITOR_V2(r,type,elem) \
	void visit_elem<type,decltype(obj.elem),&type::elem>(get_typename<decltype(obj.elem)>::name(),BOOST_PP_STRINGIZE(elem),obj);

#define REFLECT_V2(TYPE, MEN) \
	template<> struct get_typename<TYPE> { static const char* name() { return #TYPE; } }; \
	template<> \
	void visitor_object<TYPE>(TYPE& obj){ \
		BOOST_PP_SEQ_FOR_EACH(VISITOR_V2,TYPE,MEM) \
	}

// 使用方法还是不变
REFLECT_V2(Test,(a)(b)(c))

// 展开后得到

template<>
void visitor_object<Test>(Test& obj){
	VISITOR_V1(2,Test,a) VISITOR_V1(3,Test,b) VISITOR_V1(4,Test,c)
}

// 二次展开得到
template<>
void visitor_object<Test>(Test& obj){
	visit_elem<Test,decltype(obj.a),&Test::a>("int","a",obj);
	visit_elem<Test,decltype(obj.b),&Test::b>("float","b",obj);
	visit_elem<Test,decltype(obj.c),&Test::c>("char","c",obj);
}

// 最后我们要通过调用visitor_object<Test>的模板特例，遍历打印结构体中的元素
Test t{1,2.0,3};
visitor_object<Test>(t);
```

### 2.4.增加可扩展性

前面虽然实现了类型反射的宏，但是具体处理过程是写死的，我们没办法对其进行扩展和定制。但是在具体的项目中，我们需要根据不同的业务去实现不同的功能，为了提高开发效率和模块的复用性，可以使用仿函数的方法来解决这个问题。

所谓仿函数，就是定义一个类，在类中重载括号运算符，使得生成的实例可以像普通函数那样使用，这个叫做仿函数，在这里，我们可以将类型内部元素的过程定义为仿函数，如下所示：

```c++
struct Visitor{
    template<typename Object, typename Type, Type(Object::*Member)>
    operator (const char* type,const char* var,Object &obj){
        // 这里定义具体处理流程
    }
}
```

然后将REFLECT做如下修改：

```c++
template<typename Object，typename Visitor>
void visitor_object(Object& obj,Vistor vistor){}

// visitor_elem这个模板就不需要了

#dfefine VISITOR_V3(r,type,elem) \
	visitor.template operator()<data, decltype(_obj.elem), &data::elem>(get_typename<decltype(_obj.elem)>::name(), BOOST_PP_STRINGIZE(elem), obj);

#define REFLECT_V3(TYPE, MEN) \
	template<> struct get_typename<TYPE> { static const char* name() { return #TYPE; } }; \
	template<typename Visitor> \
	void visitor_object<TYPE>(TYPE& obj, Visitor& visitor){ \
		BOOST_PP_SEQ_FOR_EACH(VISITOR_V2,TYPE,MEM) \
	}
```

这样子，我们如果要访问这个类内部的元素，只需自定义一个类型，然后重载（）运算符，再调用visitor_object函数从参数里面传递进去，就行了，代码的扩展性和复用性都有了。

在FC_REFLECT中，这一过程采用了访问者模式，使得代码更加紧凑，且便于管理，但是具体的原理和上诉的内容无异。

### 2.5.在eos中，反射的应用介绍

eos中，对于反射的应用是十分广泛的，基本上可以说是导出能看代反射。

1.在合约调用过程中，将二进制序列与abi进行匹配

2.将一个类进行序列化和反序列化的转换

3.用于虚拟机与实体机之间的数值传递与处理

4.将一个类转换成json字串返回给客户端

5.虚拟机中，多索引容器对chainbase的访问

6.客户端cleos中，使用get_table访问虚拟机中表

7.node节点和各个plugin之间的数据交换

8.不可逆块二进制文件存储和访问

### 2.6.使用eos中反射的例子：将任意类型转换成json

```c++
// 定义结构体
struct Test{
    char a;
    int b;
    float c;
};
FC_REFLECT(Test,(a)(b)(c));

// 定义访问者类
template<typename T>
class Vistor : fc::reflector_init_visitor<T> {
	public:
		Vistor(T &v) : fc::reflector_init_visitor<T>(v) {}

		//仿函数，重载reflector_init_visitor中的括号操作符
		template<typename Member, class Class, Member(Class::*member)>
		void operator()(const char* name)const {
			result(name, this->obj.*member);
		}

		//获取从Object转换成的json字串
		fc::variant get_result()const {
			return fc::variant(result);
		}

	private:
		//用于保存由Object转换成的json字串，声明为mutable，可被const函数修改
		mutable fc::mutable_variant_object result;
};

// 访问Test中元素
Test t{1,2,3.0};
Vistor<const Test&> v(t);
fc::reflector<Test>::visit(v);
v.get_result();
```

## 3.其他相关技术技术

在eos中，单独的反射例子不多，大部分使用的事复合结构：

1.序列化和反序列化（fc::variant)

2.虚拟机的多索引容器(multi_index)

3.abi文件的解析与输入参数的匹配

这些例子需要结合类型计算（第四象限的推导过程）来实现。

关联文档（multi_index讲解.md）