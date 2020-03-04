##### 一、vs上的设置：

配置管理里面设置Linux-Debug，会在项目目录下产生一个CMakeSettings.json文件，参考 https://docs.microsoft.com/zh-cn/cpp/linux/cmake-linux-project?view=vs-2019 ：

```
"remoteCopySources": true,  //指定是否将本地源复制到远程计算机。如果自行同步代码，这里设置为false，否则保存变更时会将项目目录整个复制到远程计算机上。

```

还会产生一个 .vs\launch.vs.json 文件。编辑如下几项：

      "cwd": "/root/eos/",									// linux上项目目录
      "program": "/root/eos/build/programs/nodeos/nodeos",  // linux上需要调试的文件
      "remoteMachineName": "127.0.0.1",						// gdb host server


##### 二、eosio编译参数设置为debug模式，并编译





##### 三、配置eosio运行参数：

eos运行数据文件目录：

/root/.local/share/eosio/nodeos/data



配置文件：/root/.local/share/eosio/nodeos/config/config.ini

相关配置项：

http-server-address = 127.0.0.1:8888	// 对外提供服务的接口
enable-stale-production = true			// 允许生产块
producer-name = eosio				// 块生产者名称必须为eosio



如果 `enable-stale-production = true` 的情况下 ， 要设置 `producer-name = eosio`  否则无法启动。但如果是设置了多 *BP*的情况下，就要将其配置成其他的值。

**BP：Block Producer，区块生产者**



