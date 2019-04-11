# OMNET++ 使用

## Windows 安装

安装需要系统环境里有 java sdk 和 mingw64，如果没有请安装

### 安装 omnet++

从 [OMNET++官网](https://www.omnetpp.org) 下载 windows 安装包，如果下载较慢，建议使用 idm 等下载工具下载

解压到安装目录，运行 mingwenv.cmd，使用

```shell
# 配置需要的功能
./configure
# 进行编译安装
make
```

安装完成后，运行程序位于 `$Program/ide/omnetpp.exe`

### 安装 INET 库

这个集成库包含了丰富的仿真模型。

从 [INET官网](https://inet.omnetpp.org) 下载对应版本的 inet 包，解压到安装目录，最好位于 omnet++ 里面。

请先确保 omnet++ 安装成功，使用 mingwenv.cmd 进入解压目录里，运行

```shell
make makefiles
make
```

make 完成后，打开 omnet++ ide，选择导入项目，即选择 `File/Import/General/Existing Projects into Workspace`，选择相应的 INET 文件夹，导入即可

导入完成后，可直接点击运行，选择 OMNeT++ Simulation 即可进行实验

---

# OMNeT++ 教程

## 一个最简单的 OMNeT++ 程序

首先创建一个 OMNeT++ 空项目。打开 IDE，选择 File -> New -> OMNeT++ Project，即可创建一个 OMNeT++ 程序。

在程序中创建一个 NED 文件。选择 File -> New -> Network Description FIle 即可。NED 文件用于定义组件从而实现网络模型

> ned 文件的左下角可以切换 source 和 design 模式

NED 文件内容如下：

```cpp
simple Txc1
{
    gates:
        input in;
        output out;
}

network Tictoc1
{
    submodules:
        tic: Txc1;
        toc: Txc1;
    connections:
        tic.out --> {  delay = 100ms; } --> toc.in;
        tic.in <-- {  delay = 100ms; } <-- toc.out;
}
```

这里我们首先声明了一个简单模块 Txcl，其仅包含两个门逻辑，一个输入，一个输出。下面声明了一个网络 Tictocl，其包含两个子模块 tic 和 toc，两个子模块类型为 Txcl，然后我们连接两个模块的输入输出，让它们可以相互传输，且给定传播延时为 100ms

> 每个网络都必须包含 submodules 和 connections，当然还可以包含一些额外的 types、parameters 等

这里的 Txcl 的模块仅仅只是声明，接下来我们需要使用 C++ 文件来实现其内部功能。首先创建一个 cc 文件，即选择 File -> New -> File，然后开始编写 Txcl 功能

```cpp
#include <string.h>
#include <omnetpp.h>
using namespace omnetpp;

class Txc1 : public cSimpleModule
{
  protected:

    virtual void initialize() override;
    virtual void handleMessage(cMessage *msg) override;
};


// The module class needs to be registered with OMNeT++
Define_Module(Txc1);

void Txc1::initialize()
{
    if (strcmp("tic", getName()) == 0) {
        cMessage *msg = new cMessage("tictocMsg");
        send(msg, "out");
    }
}

void Txc1::handleMessage(cMessage *msg)
{
    send(msg, "out"); // send out the message
}
```

首先定义一个类 Txcl，让它继承自简单模块 cSimpleModule，并使用 `Define_Module` 宏进行注册，如果不注册则会导致在仿真时找不到 Txcl

这我们重写了两个函数， `initialize()` 函数在仿真开始时就会被调用。这里直接让它开始仿真就向外发出一个信息。`handleMessage()` 函数在模块中有消息到达时触发的。这里我们直接让它在有消息时直接向外发出一个消息，网络中的两个模块就可以不断的互传消息

仿真前的准备已经完成了，接下来只需要添加仿真描述文件 pmnetpp.ini 即可，这个文件用于告诉仿真程序网络应该如何仿真。选择 File -> New -> Initialization File(INI) 。其左下角同样有两种模式可以切换。

```cpp
[General]
# nothing here

[Config Tictoc1]
network = Tictoc1
sim-time-limit = 250000s
```

这里omnetpp.ini 有两个必填项，一个是  network ，一个是 sim-time-limit 



## NED

每个 NED 文件内主要包含几种结构：

- simple，简单模块

- module，复合模块，一般由若干个简单模块/复合模块组成

- Network，用于描述网络

- channel，用于描述与连接相关的参数和行为

在每个模块中，所有组件都是可选的，包含如下：

```cpp
connection: // 用于描述节点间的连接关系,注意连接不能跨级，即子模块无法链接到父模块外， ++ 表示使用第一个未连接的门索引
    node1.out --> node2.in; 
    node2.out --> node1.in;
```

```cpp
gates:  // 用于描述模块的连接点，包含三种类型：input，output，inout，每个门必须进行连接，除非设定 @loose 属性，
    input in; 
    output out;
    inout ga[];
```

```cpp
parameters: // 用于定义和赋值参数，可选类型 (volatile) bool, int, double, string, xml
    int count;
    protocol = default("UDP");
```

```cpp
submodules: // 在复合模块中，用于引入子模块，如下引入 Module 和 Queue，使用 `like` 标识类型
    m: Module;
    queue[sizeof(port)]: Queue;
    tcp : TCP if withTCP;
    node[5]: <nodeType> like INode;
```

```cpp
types: // 用于定义仅在本模块使用的模块或通道，防止污染命名空间
```

在 module 和 channel 之间，使用 `extends` 标识继承，如

```cpp
module WirelessHost extends WirelessHostBase
{
    ...
}
```

channel 可以不用在 parameters 下即可定义参数，如：

```cpp
channel Backbone extends ned.DatarateChannel
{
    @backbone; // 这里用于标识定义域

    double cost = default(1);
}
```

在各个模块中，可以使用 `*` 等通配符，如下，则所有包含 timeToLive 的参数会默认设置为 3：

```cpp
parameters:
    **.timeToLive = default(3);
```

在 NED 中，逻辑 XOR 为 `#` 和 `##` ，`^` 为指数，并且 `+` 可用于 string，`@unity()` 用于声明参数的度量单位信息
