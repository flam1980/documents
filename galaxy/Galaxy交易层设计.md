### Galaxy交易层设计



#### 一、交易层最主要的几个类型如下：

**交易组织管理相关类型**

transactions_controller         交易管理器
transaction_relativity             交易相关性管理类
transaction_forward_mgr      交易转发管理器
node_transactions                 节点交易管理类

**交易执行、状态相关相关**

transaction_executer             交易执行器
transaction_context               交易执行上下文
ice_apply_context                   交易action执行上下文

**合约执行相关**

contract_manager					合约管理器

database_status						node本地合约接口管理：set_ice_apply_handler，find_ice_apply_handler

​													 node本地合约接口有：	newaccount,setcode,setabi,updateauth,

​													 											deleteauth,linkauth,unlinkauth,canceldelay

galaxy_contract.cpp				  node本地合约相关接口实现



##### 一）交易层对交易的组织方式

**transactions_controller**：是交易层的界面层，对外提供相关接口：

面向输入的接口：

	push_transaction：http服务收到交易
	receive_transaction：net层收到交易

面向block层的接口：

	切换生产状态：
	start_producing //交易层启动生产
	stop_producing	//交易层停止生产
	块生产相关：
	get_irreversible_traces //出块节点：返回已不可逆的交易
	blocked_irreversible_trxs //交易落块成功，交易层同步删除相关交易
	deal_transactions //重播块上的交易

transactions_controller_impl实现类主要作用：

1、实例化相关性管理器和转发管理器。

2、记录所有尚未落块的交易，排序所有已经执行的交易。

3、管理node节点：node_transactions



**transaction_relativity** ：交易相关性管理类

获取一个交易的相关性：

```
get_transacation_relativity
get_proposal_relativity
```

交易间相关性管理、触发的接口：

```
add_transaction //将一个交易加入交易关系管理器
del_transaction //从交易关系管理器中删除一个交易，并触发相关交易
```



**transaction_forward_mgr**：	交易转发管理器





**node_transactions**：	节点交易管理类

node_transactions用于管理某一个节点的所有交易的执行顺序。



##### 	二）交易的异步执行





#####     三）合约管理





#### 二、交易线程池：

1.executer执行线程池（定义在controller中）
2.ice_callback回调线程池（定义在controller中）
3.交易层私有线程池（定义在transactions_controller中）