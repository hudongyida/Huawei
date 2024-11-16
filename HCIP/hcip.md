# HCIP

## OSPF

一种基于链路转态的内部网关协议

### 核心思想

1. 路由和路由之间先建立连接关系
2. 相互交换自己路由和链路信息【泛洪】
3. 当路由器掌握了**本区域**的路由和链路信息后开始路由计算
4. 将计算结构写入路由表中

### 基础术语/参数/概念

Router ID: 用于标识自制系统中唯一的一台路由器【不是IP地址】，参数格式是一个32位无符号整数
Router ID选举规则：

- 手动指定【在项目中一般使用路由器的管理接口作为router ID】
- 如果没有手动指定则使用Loopback接口中最大的IP地址作为Router ID
- 若没有配置Loopback则使用物理接口中最大的IP地址做完Router ID

配置命令

```
ospf 进程号 Router-id [Router ID]
```

度量值： Cost 开销，每一个接口都有自己的开销
开销的计算方法：100 Mbit/s / 接口带块。若计算出来的cost小于1则按1算
累计cost 就是将链路上的所以出接口的cost值相加
cost值可以自己修改
配置命令【在接口视图下】

```
ospf cost 开销值
```

区域：
在大型园区网络中会有许多的路由，为了减轻路由表的计算压力会将OSPF进行区域划分
ospf Area用于标识一个OSPF区域
在ospf区域标识没有明确的顺序要求，但要求必须存在0区域，【一般也是从0区域开始按照顺序往后划分】
配置命令

```
ospf 1
area 区域号
```

邻居表：
在OSFP进行运算之前需要先建立邻居表，已便于知道哪些路由和自己是相邻的关系
邻居关系通过Hello报文建立和维护，可以使用dis ospf peer 查看

LSDB：
Link state database 链路转态基础数据。用于保存自己产生和从邻居收到的LSA信息
可以使用dis ospf lsdb命令查看

OSPF路由表：
OSPF路由 **不等于** 普通路由表
OSPF路由表时更具LSDB计算出的一份路由表
用于存储OSPF网络的路由信息
可以使用dis ospf routing 查看

```
Destination     Cost    Type    NextHop     AdvRouter   Area
10.0.1.1/32     0       stub    10.0.1.1    10.0.1.1    0.0.0.0
10.0.1.1/30     1       Transit 10.0.12.1   10.0.1.1    0.0.0.0
```

Destination: 目标网络
Cost: 开销
Type: 路由类型
NextHop: 下一跳路由
AdvRouter: 目标路由ID
Area: 区域信息

OSPF报文格式和类型
OSPF报文是封装在IP数据报中的，协议号89

OSPF报头格式

```
*IP头部*
**********************************
* Version * Type * packer Length *
*           Rouser ID            *
*           Area ID              *
*     Checksum   *  Auth Type    *
*        Authentication          *
**********************************
*OSPF数据部分*
*IP尾部*
```

Version：版本
Type： 类型字段
packer Length：整个OSPF报文长度
Rouser ID: 生成改报文的路由器ID
Area ID： 需要告知的区域
Checksum：校验字段，校验包括OSPF报文头部的整个OSPF报文
Auth Type：是否需要认证，0标识不认证，1标识简单的明文密码验证，2标识使用MD5摘要算法认证
Authentication： 认证信息，根据 Auth Type 的值而定

OSPF的5种报文类型

| Type | 报文名称           | 报文功能                |
| ---- | ------------------ | ----------------------- |
| 1    | Hello              | 发现和维护邻居关系      |
| 2    | database           | 交互链路信息摘要【LSA】 |
| 3    | Link state Requset | 请求特定的链路信息      |
| 4    | Link state update  | 发生详细的链路信息      |
| 5    | Link state ACK     | 确认LSA                 |

#### LSA

LSA是OSPF进行链路计算的关键依据
LSA中包含了链路和链路有关的几乎全部信息
LSA 一般存在 OSPF数据报的 OSPF数据部分

OSPF数据部分 LSA报文格式【头部】

```
******************************
* LS Age * Options * LS Type *
*       Link State ID        *
*     Adverising Router      *
*    LS sequence number      *
* LS checksum *   length     *
******************************
```

LS Age ：LSA已经存在的时间
Options: 可选项：OSPF对应的特性
LS Type：LSA类型【1-5 7】
Link State ID：根据LSA类型来定
Adverising Router ：产生LSA的路由ID
LS sequence number：链路转态序列号，当LSA有新的实例产生时，序列号增加1
LS checksum： 校验和
length： 包含LSA头部在内的LSA长度

LSA类型

| 类型 | 名称              | 描述                                                                                                     |
| ---- | ----------------- | -------------------------------------------------------------------------------------------------------- |
| 1    | 路由SLA           | 每个设备都会产生，用于描述自己的链路和开销，产生后会在接口指定的区域内泛洪                               |
| 2    | 网络SLA           | 由DR路由产生，描述MA网络形成的路由关系                                                                   |
| 3    | 网络汇总SLA       | 由ABR产生，用于描述一个区域内的路由信息                                                                  |
| 4    | ASBR汇总LSA       | 由ABR产生，用于描述到达ASBR路由的信息，并且可以跨区域                                                    |
| 5    | AS外部SLA         | 由ASBR，用于描述OSPF的外部路由                                                                           |
| 7    | 非完全末梢区域SLA | 和5类SLA类似，但是只能在NSSA的特殊区域中使用，在跨越区域的时候ABR会将7类SLA转换为5类SLA注入到Area0区域中 |

LSA描述网络的核心思路：
将自己所相接的网络和邻居转化为带权有向图

带权有向图描述网络链路信息的方式/思路

- StubNet:
  用于描述路由器到一个stub网段信息【例如Loopback接口】
  可以理解为，路由器的末端网络。可以直接链接PC，server等业务设备
- p2p:
  用于描述两个路由相连接的点对点网络
  携带开销值为接口的出口开销
- TransNet：
  用于描述多个路由器互联的的关系

  ```
  *******************
  *       R         *
  *       |         *
  *       |         *
  *   R ——S——— R    *
  *       |         *
  *       R         *
  *******************
  ```

  描述过程：

  1. 路由器直接会将假定一个伪节点N1，这N1仅逻辑存在，并不在现实中存在。
  2. 路由器会和这个伪节点N1链接
  3. 对于cost开销的计算，路由器到N1节点的cost开销为路由器自己的接口开销，但是N1伪节点到路由的开销为0

  而对伪节点的描述则为DR路由器的地址【有关DR路由器详细请见下文OSPF路由邻居关系建立】

路由器对LSA的处理机制
但一个路由器收到一个LSA【如果参数相同则按照序号往下比较】

1. 序列号是否相同，否 越大越优
2. 校验和是否相同，否 校验和大优先
3. MAX Age=3600S，否
   比较AGE是否超过15min   是这age小的优先
   否则认为LSA相同，保留先收到的
4. 收到的LSA更优

##### 1类LSA

Router LSA ：
每个路由都会产生的LSA，用于描述该路由器直连的接口信息
router LSA报文格式

```
*OSPF数据部分 LSA报文格式【头部】*
******************************
* 0 * V * E * B * 0 * #links *
*           Link ID          *
*          Link Date         *
* link type * #TOS * metric  *
*              ...           *
******************************
```

字段解释：
V:LSA如果产生的LSA路由是虚链接的端点 值为1 否则为0 【一般默认为0】
E:如果产生此LSA的路由为ASBR 值为1 否则为0
B:如果产生此LSA的路由为ABR 值为1 否则为0
#Lins: LSA中的链接数量
Link ID: 链接参数【根据Link type而定】
Link Data: 链接路由的ID信息【根据Link type而定】
link type: 链接的类型【详细信息见上文 带权有向图】

link type , link data , link ID 和带权有向图的关系
带权有向图的类型是link type的支持，而带权有向图的描述信息则是link data , link ID的参数

| Link Type | Link ID                          | Link Data                        |
| --------- | -------------------------------- | -------------------------------- |
| p2p       | 邻居路由器的Router ID            | 宣告Router LSA路由器接口的IP地址 |
| TransNet  | DR路由器IP地址                   | 宣告Router LSA路由器接口的IP地址 |
| StubNet   | 宣告Router LSA路由器接口的IP地址 | stub网络的网络掩码               |

##### 2类LSA

Network LSA
网络LSA是对MA网络的描述
由DR产生，描述本网段的链路转态【有关DR路由器详细请见下文OSPF路由邻居关系建立】
网络LSA中会描述所有和DR建立了邻居关系的OSPF路由器，并且同时携带该网段的网络掩码

##### 3类LSA

Network Summary LSA
由ABR产生
网络LSA工作原理：

- 若存在Area 0 1 2 三个区域，但由于防环机制，Area 1，区域无法直接和Area 2交换LSA，若要交换LSA则必须要通过Area 0
- 此时Area 1区域中有一条非ABR路由产生的一条路路由信息，并且在Area区域中泛洪。
- 因为OSPF是会根据区域进行路由计算的，所以在Area 1 中非ABR产生的路由条目无法泛洪到其他区域中，所以其他区域无法知道这条产生的新路由
- 为了使得其他区域也能知道这条产生的新路由，此时Area 1的ABR路由就会根据这条路由器产生一条3类LSA，并且告知到Area 0区域中【将LSA 3类报文在Area区域中泛洪】
- 其他区域例如Area 2 ABR在收到 Area 1区域ABR路由产生的LSA后，也会根据这条LSA 重新产生一条新的3类LSA报文，并且告知自己所在的区域Area 2

三类LSA报文格式

```
******************************
* LS Age * Options * LS Type *    //*OSPF数据部分 LSA报文格式【头部】
*       Link State ID        *    //*OSPF数据部分 LSA报文格式【头部】
*     Adverising Router      *    //*OSPF数据部分 LSA报文格式【头部】
*    LS sequence number      *    //*OSPF数据部分 LSA报文格式【头部】
* LS checksum *   length     *    //*OSPF数据部分 LSA报文格式【头部】
*      Network Mask          *
*  0  *      metric          *
*           ....             *
******************************
```

LS Type: 取值为3
Link State ID： 路由器的目标网络
Adverising Router：产生改LSA的路由ID【ABR路由】
Network Mask：目标网络的网络掩码
metric: 到达此路由的开销

##### 5类LSA

在某些情况下OSFP需要引入外部路由
若此时需要将引入的外部路在区域内泛洪就需要5类LSA
此时接入外部路由的这太路由器就是ASBR路路由器，他可以是OSPF中的任意路由器

5类LSA报文格式

```
******************************
* LS Age * Options * LS Type *    //*OSPF数据部分 LSA报文格式【头部】
*       Link State ID        *    //*OSPF数据部分 LSA报文格式【头部】
*     Adverising Router      *    //*OSPF数据部分 LSA报文格式【头部】
*    LS sequence number      *    //*OSPF数据部分 LSA报文格式【头部】
* LS checksum *   length     *    //*OSPF数据部分 LSA报文格式【头部】
*      Network Mask          *
*  E  *  0  *    metric      *
*    Forwarding  address     *
*    External Route Tag      *
*           ...              *
******************************
```

LS Type: 取值为5
Link State ID： 外部路由网段
Adverising Router：产生改LSA的路由ID【ASBR路由】
Network Mask：目标网络的网络掩码
E : 改外部路由使用的度量值类型【见下文】
metric: 到达此路由的开销【需要将E关键纳入考核访问】
Forwarding  address：到所通告的目的地地址的报文将被转发到这个地址，当FA为0.0.0.0时，这到达改外部网段的流量会被引入这跳外部路由的ASBR，若FA不为0.0.0.0则流量会被发往这个转发地址。FA的引入可以使得OSPF在某些特殊场景中的此优问题可以被规避
External Route Tag: 外部路由标记，常被用于部署路由策略

E 类值的标识

- 0 标识度量值类型为Metric-Type-1
- 1 标识度量值类型为Metric-Type-2
  Metric-Type-1 / 2
  的区别在于
  1类会将不仅会急速OSPF内部的cost值，还会将外部路由开销的值一并加上
  2进计算在OSPF内部产生的cost

命令操作

```
ospf 1
import-router 路由类型 路由网络
```

##### 4类LSA

工作模式和3类LSA类似，但是4类LSA是为了解决外部路由引入的通过问题
同3类LSA，若Area 1区域的非ABR设备引入了一条外部路由，并且在Area 1内泛洪5类LSA
Area 1 的ABR路由收到泛洪的5类的LSA，并且根据这条5类LSA产生一条4类的LSA，在0区域泛洪，其他区域同理，收到后也会产生一条4类LSA并且在所属区域内泛洪

报文格式同3类LSA，仅仅只是参数有所改变
LS Typ: 4
Link State ID： ASBR 路由ID
Adverising Router：产生改LSA的路由ID【ABR路由】
Network Mask：保留，无意义
metric: 到达此路由的开销

##### 7类LSA

仅用于解决特殊区域产生的外部路由
由于NSSA区间不允许4类5类LSA泛洪，但又由于NSSA的区域特性，允许外部路由接入，所以添加了7类LSA
但同时7类LSA只能在NSSA特殊区域内泛洪，所以若要将这条LSA告知到其他区域需要ABR路由将根据收到的7类LSA 产生5类的LSA在注入回Area 0区域区域中

7类LSA报文格式同5类LSA

### 工作原理

#### 建立邻居关系

建立邻居关系工作过程概述

1. 路由器直接通过互相发生Hello
2. 协商主从关系
3. 描述各自的LSDB
4. 更新LSA，同步LSDB
5. 计算路由

邻居关系建立：
0. 假设有两台路由器 AB，并且路由器AB上同时运行OSPF

1. 两台路由器每个一定的周期会互相发送Hello报文，但是邻居列表中为空【此时接口转态为Down】
2. B 收到A的Hello报文后如果参数匹配，则将R1加入自己的邻居列表中，再次发生Hello报文，并且【A收到B发出的第一个Hello报文，并且邻居列表中不存在自己时接口模式将从 **Down 改为 Init**】
3. A收到B发出的Hello报文后并且邻居列表中存在自己的时候，则标志着邻居关系建立完成，AB两台路由器的邻居列表中都存在对方的 Router ID【A收到B发出的第一个Hello报文，并且邻居列表中存在自己，接口将从 **Init 改为 2-way**】

Hello报文的作用：

- 发现邻居关系
- 建立邻居关系
- 保持邻居关系

Hello 报文格式

```
*********************************************
*               Network Mask                *
* Hello Interal * Options * Router Priority *
*            RouterDeadinterval             *
*             Designated Router             *
*        Backup Designated Router           *
*               Neighbor                    *
*                  ...                      *
*********************************************
```

参数前面带*的为重要参数
*Network Mask: 接口和子网掩码
*Hello Interal：间隔时间【通常为10s】
Options：E是否支持外部路由 MC 是否支持转发组播数据 N/P 是否为NSSA区域
Router Priority：DR优先级，默认为1，【越小越优先】如果设置为0 则不参加选举
*RouterDeadinterval：失效时间，若在此时间内未收到邻居发来的Hello报文则认为邻居失效【通常为40s】
Designated Router：DR接口地址
Backup Designated Router： BDRR的接口地址
*Neighbor： 邻居列表

#### 建立邻接关系

邻接关系需要再邻居关系的基础上建立
邻接关系建立过程：

1. AB两台路由器会同时发发送DD报文，但改转态下DD报文不包含链路转态信息，并且进行Master选举，【若准备建立邻接关系，接口将从 **2-way 改为 ExStart**，并且开始发送不带链路信息的DD报文】
2. 在计算会后确定Master信息后 *假设是B为主路由*，A会以B发来的DD报文中的seq回复B，并且加上自己的LSDB摘要信息【在收到第一个不携带链路转态信息的DD报文后，接口举将从**ExStart 改为 Exchange** 然后发生带有LSDB摘要信息的DD报文】
3. B在收到A发来的带有LSDB摘要信息的DD报文后会将 seq的值+1并且发生自己带有LSDB摘要信息的报文
4. A收到B发来的DD报文后会对报文基于回复，仅有seq+1的值，不携带任何信息，做为回复的DD报文【A在收到DD报文后会将接口从 **Exchange 改为 Loading**，而B在收到A的回复DD报文后接口也将从**Exchange 改为 Loading**】
   ***注意： 加入双方交互的LSDB摘要信息都是最新的，无许跟新，则接口转态会立马从 Loading 转换为 FULL。或者说直接从Exchange转为FULL***
5. *若交换的LSDB都为最新这不执行这一步* A发送LSR【Link state Requset】，B在收到LSR后会发送对于的LSU【Link state update】。A在收到对于的LSU后会基于LSAck【Link state ACK】确认【完成后接口转态变回从Loading 转换为 FULL】

图示：

```
B..............................A
|  <<DD seq=X I=1 M=1 MS=1     |
|    DD seq=Y I=1 M=1 MS=1>>   |
|                              |
|  <<DD seq=Y LSDB 摘要        |
|    DD seq=Y+1 LSBD摘要 MS=1>>|
|                              |
|  <<DD seq=Y+1                |
```

DD报文格式

```
**********************************************
* Interface MTU * Options * 0 0 0 0 0 I M MS *
*                  DD seq                    *
*                 LSA Header                 *
**********************************************
```

Interface MTU：不片分的情况下可以发出的最大IP报文长度
Options： 同Hello的Options字段
I ：标识是不是第一个DD报文，是则为1，否则为0
M ：标识这是不是最后一个DD报文 是则为0 否则为1
MS：标志是否为主从关系，Router ID大的一方为主，值为1是，标识主
DD seq：确认号

#### DR 和 BDR

MA网络问题：

```
    *******************
    *       R         *
    *       |         *
    *   R ——S——— R    *
    *       |         *
    *       R         *
    *******************
```

MA网络【Multiple Access多路访问】
MA网络进行邻接关系建立的时候因为关系负责，造成大量的资源浪费【重复的LSA泛洪】
解决问题就需要再MA网络中选出一个指定路由DR，并且由MA网络来建立和维护邻接关系，同步LSA。
为了避免DR路由器出现的单点故障，在选举DR路由的时候还会选举出BDR路由器

DR和BDR的选举规则
选举算法：优先级+Router ID   越大越优先
注意，DR和BDR路由是非抢占式的，优先级为0时不参加DR路由选举

#### SPF路由计算过程

1. 构建SPF树干【非StubNet链路类型】
2. 所有的路由都已自己为中心，然后将自己的邻居纳入候选列表
3. 查看候选列表中的开销，开销小的则被选为干节点
4. 向干节点请求LSA，并且将干节点的邻居添加在自己的候选列表，若邻居发来的LSA中和自己列表中的的路由信息重复，则会进行开销比较，若邻居的开销小，则会将自己候选表中的路由信息替换成新的LSA信息，若自己列表中的开销小则不做处理，若开销相同，则会将邻居的LSA信息添加进来，并且在路由表计算的时候，将两个路由一并计算【负载均衡处理】
5. 在干接点发来的邻居表中会找出开销最短的做为自己的新干节点，一次类推，直到将整个网络的路由信息推出来
6. 干接地算出来后再计算叶子信息【处理StubNet链路类型】

事例：
拓扑如下，并且我们假定从R1路由器开始推算【这里为假定的开销值】
![ospf_SPf](./img/OSPF1%20spf计算.png)

1. R1已自己为根，并且根据自己的LSA将所有的非StubNet类型的网络纳入自己的候选列表，R3【48】TransNetR2【1】【R2 R3，**注意R1和R2之间是以太网线路链接，而以太网线路本身就是广播型网络，所以R1-R2直接的路由类型是 TransNet网络，但R1-R2直接的线路严格上来说并不算是MA网络，但R1和R3直接用的是串口线链接，所以构建的就是为P2P网络**】
2. 从自己的候选列表中开销最短的加入自己的最短路径树，由于R2-R1为TransNet网络所以会先纳入伪节点DR，并且向DR请求网络LSA，但是由于此DR只有R1和R2的路由信息，所以R1收到的网络LSA只有自己和R2的路由信息，R1将R2加入最短路径树中，并且向R2请求LSA信息【邻居信息】。
3. 由于R2和R3R5构建了MA网络，所以类型为TransNet【1+0+1】，和R4构建了P2P网络，类型为P2P【1+0+48】，并且将DR的网络LSA和P2P都纳入候选列表中
4. R1将R2的MA网络中的DR路由纳入最短路劲树中，并且由于R1-R3的cost为48，但从R1-R2-R3的路由为1+0+1,所以候选列表中的p2p2会被移除，并且将R2的MA网到R3路由的路由路径加入候选列表
5. R1分别向R3和R5发送LSA请求，R3的LSA中提供的邻居信息都在候选表中，R3的路由信息已经确定，不再变动
6. 在R5回复的路径的报文中存在P2P链接到的R4【48】，R1会对R4的cost进行比较，从R2过开销为【1+0+48】，从R5过开销为【1+0+1+48】，R2的路由更优秀，所以路由不进行变动
7. 在最短路径树构建完成后，再将各个路由StubNet网络以叶子的形式插入在最短路径树上。【将LSA类型为StubNet的网络全部纳入候选列表中】
8. 根据顺序依次算出OSPF路由表，其他所有路由全部完成后，OSPF计算完成，引入路由表中

#### 区间路由计算过程

区间路由计算过程：
0. 假定Area 1区域已经完成了OSPF路由计算，现在需要将区域内的路由信息泛洪到其他区域中

1. Area 1的ABR路由会根据Area1已经计算完成的ospf路由产生一份3类LSA，并且在Area 0 区域中泛洪，并且Area0 区域中接受该LSA的路由COST值会+1，也包括其他区域的ABR路由器
2. 此时Area 2 3..区域的ARB路由收到Area1 路由的泛洪信息后会将根据这份LSA再产生一份LSA泛洪到自己坐在的区域中，并且将COST值再+1
3. 对于外部路由，计算过程同1-3，只是LSA类型会有所不同

#### OSPF区域的防环机制

用于解决可能存在的OSPF区域3层环路问题

1. 所有的Area区域必须和Area 0区域相链接，假如Area1和Area2区域有ABR路由器，但是该ABR路由器不会转发双区域的3类LSA
2. ABR不会将藐视某个区域内的网段再次通过3类LSA注回该区域
3. 从非骨干Area 0 收到的LSA不纳入转发计算

#### 虚链接

会破会OSPF的防环机制，不推荐使用

### OSPF 特殊区域

在OSPF中为了优化网络路由表的规模，和网络性能，所以存在一些特殊区域

#### Stub/ToTally Stub

Stub：
末梢区域，若Area 1为该区域，则Area1为末梢区域，改区域将无法接入外部路由，并且改区域内禁止4类LSA和5类LSA传递，若此时有AS外部路由需要通告，则该区域的ABR路由会生成一个3类LSA在改区域内传输

需要注意：

- 骨干区域不能配置为Stub
- Stub区域内所有的路由都要配置Stub
- 该区域内不能引入AS路由
- 虚链接不能穿越Stub区域

ToTally Stub：
ToTally Stub是在Stub区域上的进行的改进
ToTally Stub除了不允许45类LSA，也将不允许普通的3类LSA区间路由通过进行传递

ToTally Stub 是将AS外部路由和其他区域路由的通过用默认路由代替，通过ABR产生特殊的3类LSA【默认路由信息】进行通告。

配置方法和配置Stub区域相同，值需要再ABR路由器配置Stub时候在后面追加no-summary关键字

#### NAAS/ToTally NSSA

NAAS
不完全末梢区域，和Stub区域类似，但是可以接入AS外部路由。由因为同Stub，所以无法传递45类LSA，但又因为该区域接入了外部路由，所以这边引入了7类LSA。

ToTally NSSA
同ToTally Stub 但是可以接入外部路由，使用7类LSA进行区域内宣告

### OSPF路由特性

1. ABR可以执行路由汇总
2. 可以配置路由的接口为Silent-Interface
   配置成Silent-Interface不会接受和发送OSPF报文，但是仍然可以转发业务数据，并且接口的参数可以参与OSPF的运算
3. OSPF支持报文认证

OSPF认证支持区域认证和接口认证，当两种认证都存在的时候优先使用接口认证
区域认证：在该区域下所有路由器的认证模式和口令必须一致，否则无法加入该区域
接口认证：相邻路由器的直连接口的认证模式必须一致，否则无法连理邻接关系

## IS-IS

IS-IS是OSI中的一种路由链路状态协议，由于在部分场景下OSPF的计算性能可能不够用，所以ISO将ISIS中的部分概念魔改到看TCP/IP体系下

### 核心思想

同OSPF将路由进行分区，然后进行区间路由计算，并且由DIS【OSPF中的DR概念】广播同步链路转态

### 基础术语/参数/概念

NSAP网络服务访问点，类似TCP/IP模型中IP地址的概念

NSAP构成

```
|<-------------------NASP--------------------->|
|<---IDP--->|<----------------DSP------------->|
************************************************
* AFI * IDI * High Order DSD * System ID * SEL *
************************************************
|              1-138         |    6      |  1  |  单位Byte
```

IDP 类似于TCP/IP中的网络号，AFI可以理解为机构，IDI为区域【AFI和IDI 概念不重要】
DSP 类似于TCP/IP中的子网号和主机地址，High Order DSD为区域号， System ID为主机号，SEL为协议号

NET
可以理解为网络中的实体，一个特殊的NSAP，SEL字段为00
例：
49.00001.0000.0000.0001.00

```
          Area ID                 System ID      SEL
           49.00001.             0000.0000.0001.  00
* AFI * IDI * High Order DSD *     System ID   * SEL *
```

区域：
ISIS同OSPF一样存在区域划分，在OSI中设计规划时候划分了3中AS区，但在想TCP/IP迁移的时候做了部分的阉割【修改】，只将区域划分成了2级
骨干区域和非骨干群

路由器的分类
Level-1
若路由器为该级别表示改路由器工作在**非骨干区域**

Level-1-2【默认】
若路由器为该级别表示改路由器工作在骨干区和非骨干区，可以理解为OSPF中的非ABR路由

Level-2
若路由器为该级别表示改路由器工作在**骨干区域**

ISIS支持的网络类型

1. 关播Broadcast 【Ethernet】
2. 点到点p2p

ISIS开销计算:
isis的开销计算有三种计算方式

- 全局计算，为所有接口设置统一的开销
- 接口开销，为单个设置一个开销
- 自动计算开销，根据接口带宽自动计算开销，开销计算方式同OSPF cost=100/接口带块
  需要注意，由于历史原因，自动计算开销的类型为narrow【最大值为63】但是后来的规定中填了的Wide【最大16777215】类型，而华为路由器的ISIS路由默认采用的narrow类型

ISIS报文格式：
ISIS报文是直接封装在数据链路层中的
ISIS报文头部中可以分为专用头部和通用头部

ISIS通用头部

```
**********************************************
* Intradomain Routing Protocol Discriminator *
*               PDU Huader Length            *
*       Version/Protocol ID Extension        *
*            System ID Length                *
* R * R * R *    PDU type                    *
*                version                     *
*                Reserved                    *
*                MAX Areas                   *
**********************************************
```

Intradomain Routing Protocol Discriminator: 区域路由选择协议*固定 0X83*
PDU Huader Length: ISIS 报文头长度【包括通用和专用头部】
Version/Protocol ID Extension: 版本号*固定0X01*
System ID Length: System ID长度
R: 固定保留为 值为0
version: *固定0X01*
MAX Areas: 支持的最大区域个数

ISIS报文类型：
报头中的PDU type类型
一共9种报文，但是可以分成4中类型
IIH: 类似于OSPF中的Hello报文
    L1 LAN IIH    L2 LAN IIH    P2P IIH
LSP：类似于OSPF中的LSA
    L1LSP   L2LSP
CSNP【SNP】：类似于OSPF中的带链路数据的DD带报文，即链路摘要数据
    L1 CSNP   L2 CSNP
PSNP【SNP】：类似于OSPF中的LSR和LSACK
    L1 PSNP   L2 PSNP

IIH 报文：
同OSPF用于建立和维护邻接关系

IIH报文格式：【P2P和关播类型*LAN IIH*不一样】

```
*************************
*   PDU Common Header   *
* Reserved/Circuit type *
*      Source ID        *
*     Holding Time      *
*      PDU Length       *
* R *   Priority        *
* LAN ID 【仅LAN IIH】/ Local Circuit ID*【仅在P2P IIH】*
* variable Length Fields*
*************************
```

Reserved/Circuit type: 标识路由器类型
Priority：DIS选举优先级0-127越大越优先
LAN ID： DIS的System ID和伪节点ID
Local Circuit ID：本地链路ID

DIS伪节点：
可以理解为OSPF中的DR
DIS负责创建和生成伪节点和生成伪节点的LSP
DIS的选举规则：

1. DIS比较DIS的优先级，最大的直接被选为DIS
2. 若优先级相同，则MAC地址最大的被选择为DIS
   DIS和DR的区别

- DIS优先级为0也会参加选举，但OSPF的优先级为0则不会
- DIS运行抢占，但是OSPF不允许
- DIS中所有路由器都会形成邻接关系，OSPF中只有DR和BDR会建立邻接关系

LSP：
可以理解为OSPF中的LSA
LSP分为两种，Level-1 LSP和Level-2 LSP，对应的是Level-1接口和Level-2接口，而Level-1-2接口则可以同时处理这两种LSP
LSP报文格式

```
**************************
*   PDU Common Header    *
*      PDU Length        *
*   Remaining Lifetime   *
*        LSP ID          *
*     Sequence Number    *
*       Checksum         *
* P * ATT * OL * IS Type *
* variable Length Fields *
**************************
```

Remaining Lifetime： LSP生存时间，以秒为单位
LSP ID：System ID + 伪节点ID + LSP分片后的编号
Sequence Number： LSP序列号
Checksum： 校验和
P：保留字段
ATT： 由level-1-2产生，可以理解为是否为ABR路由器，为1则是ABR路由器，0则不是
OL： 过载位标识，在路由器内存不住是才会设置该表示
IS Type：路由器表示，指明是Level-1还是Level-2路由器

**非**伪节点的LSP
![sisi_LSP_1](./img/sisi_LSP_1.png)
AREA ADDR:该LSP来源的区域号
INTF ADDR:该LSP描述的接口信息
NAR ID ： 该LSP中描述的邻接信息
IP-Internal: 该LSP中描述的网段信息

伪节点的LSP：
![sisi_LSP_2](./img/sisi_LSP_2.png)
伪节点的LSP中只包含了邻接信息，不包含路由信息



ISIS的LSDB
![sisi_LSDB](./img/ISIS-LSBD.png)
伪节点标识：
00标识实节点
非0标识伪节点



CSNP：同OSPF中的DD报文
CSNP 包含了该设备LSDB中的LSP摘要信息，路由器通过CSNP判断LSDB是否需要同步
- 在广播网络上，CSNP由DIS定期发送【默认10秒】
- 在点对点网络上CSNP只在第一次建立邻接关系时候发送
报文格式
```
**************************
*    PDU Common Header   *
*        PDU Length      *
*         Source ID      *
*        Start LSP ID    *
*        End LSP ID      *
* variable Length Fields *
```
Source ID: 发出CSNP报文的System ID 
Start LSP：CSNP 报文中的第一个LSP ID值
End LSP ID：CSNP 报文中的最后一个LSP ID值


PSNP：同OSPF中的LSR和LSACK
当路由器发现自己的LSDB不同步的时候会发送PSNP请求新的LSP
收到LSP的时候会使用PSNP进行同步
报文格式:
```
**************************
*    PDU Common Header   *
*        PDU Length      *
*         Source ID      *
* variable Length Fields *
```
Source ID: 发出PSNP的路由器的system ID

### 工作原理

#### LSP处理机制
当路由器收到一个或者多个LSP是
1. 检查序列号是否相同       否，序列号大的最优
2. 保活时间=0              是，保活时间为0最优
3. 校验和是否相同          否，校验和大的更优先
                          是，不处理LSP

#### ISIS邻接关系建立过程：
关系建立原则：
- 必须处于同一层级
- Area ID必须一致
- ISIS接口的网络类型必须一致
- ISIS地址接口必须处于同一网段
- 开销计算模式必须一样

广播型网络建立邻接的过程: 
R1和R2同时发送IIH报文
假设R2先收到R1的IIH报文

- R2收到R1发出的IIH报文，但是邻居列表中没有自己【接口进入从Down 改为 Initial】
- 然后R2将R1的System ID添加到自己的邻居列表中，并且再次发出IIH报文
- R1 收到了R2 发出的IIH报文，并且在邻居列表中发现了自己的System ID此时会将R2的System ID添加到自己的邻居列表中并且发出IIH报文【接口从Down直接转为UP转态】
- R2再次收到R1发出的IIH报文，发现邻居列表中存在自己的System ID 【接口转态将从Initial 修改为 UP状态】
- 邻接关系建立完成



点对点型网络建立邻接的过程: 
点对点网络在建立邻接关系时只需要进行两位握手即可，即收到IIH 报文直接进入UP转态
但是2次握手存在明显缺陷，即无法解决单点通讯问题
所在华为路由器中设置，点对点网络中建立通讯依然需要需要3次握手

广播型LSP同步过程：
1. 当有一台新路由器R加入isis时，会先发送IIH报文和链路上的路由器建立邻接关系
2. 链接建立后R路由器会等待LSP计时器刷新，然后将自己的LSP广播发出，以便于网络上的所有路由器都能收到自己的LSP
3. 链路上的DIS会将R的LSP 路由信息添加到自己的LSDB中，并且等待CSNP计算器刷新并且发送CNSP报文
4. 路由器R收到了DIS发来的CNSP报文，并且和自己的LSDB进项比对，然后向DIS请求自己缺少的LSP
5. DIS收到请求后会向R3发送对应的LSP，以来同步LSDB信息
注意：LSP计时器一般是30s一刷新，但是DIS的CNSP信息是LSP计时器的1/3，若链路上的路由器在30s内没收到DIS发出的CNSP则会认为DIS故障，然后重新选举DIS


点对点网络LSP同步过程:
1. R1和R2交换CSNP信息
假设是R1先收到请求并且先想R2发出LSP请求
2. R2收到请求后会发送对应的LSP，并且开启LSP重传倒计时
3. R1收到了R2的LSP信息后会发送PSNP作为确认，R2收到R1的PSNP确定传输完成，若没收到PSNP信息待LSP重传倒计时结束后会再次发送LSP信息


#### 路由计算
Level-1：
计算过程同OSPF的路由计算过程
使用最短路径树SPT进项计算

Level-2:
只负责维护Level-2 的LSDB，并且算出可以到达全网的路由

Level-1-2：
分区域分别计算L1和L2两个区域的路由信息
并且将计算出来的L1的路由信息以LSP的方式发送到L2区域中
同时位于此区域的路由会向L1区域下发默认路由信息，表示自己可以到达其他网段，将ATT标志为改为1


对于区域中可能会存在的默认路由次优问题，可以通过路由渗透解决，即将L2区域的路由信息下发到L1区域中

