# OpenFlow网络中组播机制的研究



## 系统设计思想

### 组播组成员管理

传统网络中，主机与路由器之间通过运行 **IGMP** 协议来交互，在SDN中， **IGMP** 报文在控制器上实现。

**IGMP** **V3 ** 有两种报文，分别是查询报文与报告报文。控制器接收到查询报文之后，不进行处理，直接转发给 **OF** 交换机；报告报文由主机发出，用于声明加入某个组播组或响应 **IP** 组播路由器的查询。

在组播控制器中，通过解析报告报文的每个 **Group** **Record** 的 **Record** **Type** 字段来确定当前网络中的组播组成员情况。

### 组播转发树

在 **OpenFlow** 网络中生成组播转发树是以边界 **OF** 交换机作为根节点，组播控制器在生成组播转发树之前要获得全局信息，包含整个网络拓扑和各个链路状态等。

* 获取网络拓扑

SDN中使用 **LLDP** 协议作为链路发现协议，该协议提供标准的链路发现方法，可将设备能力、设备标识、接口标识、管理地址等信息组成不同的 **TLV** (Type/Length/Value)，并将这些信息封装在 **LLDPDU** (链路层发现协议数据单元)中，发送给与自己直连的邻居节点，邻居以标准的 **MIB** (管理信息库)的形式将接收到的信息保存，以供网管系统判断查询链路状况。

![LLDP](https://github.com/BokalaQuan/RyuNote/blob/master/LLDP.png)

> 1. 控制器向所有与之相连的 **OF** 交换机 **Packet_Out** 封装  **LLDP** 数据包的消息；
> 2. 该消息命令 **OF** 交换机将此数据包转发到所有端口；
> 3. 交换机 **Packet_In** 数据包给控制器；
> 4. 控制器接收到 **Packet_In** 消息后，分析该数据包并将 **OF** 交换机的连接信息保存到链路发现表中。

* 获取链路状态信息

```c
/* Body of reply to OFPST_PORT request. If a counter is unsupported, 
* set the field to all ones. */
struct ofp_port_stats {
  uint16_t port_no;
  uint8_t pad[6];	/* Align to 64-bits. */
  uint64_t rx_packets;
  uint64_t tx_packets;
  uint64_t rx_bytes;
  uint64_t tx_bytes;
  uint64_t rx_dropped;
  uint64_t tx_dropped;
  uint64_t rx_errors;
  uint64_t tx_errors;
  uint64_t rx_frame_err;
  uint64_t rx_over_err;
  uint64_t rx_crc_err;
  uint64_t collisions;
};
```

其中 *tx_bytes* 字段表示当前该端口发送了多少字节数，可通过定时获取该字段来间接计算链路带宽利用率。

网络拓扑作为图的节点，各个链路利用率作为边的权值，控制器可以利用 **Dijkstra** 算法计算从根节点到目的节点的最优转发路径。

* 有新的组成员加入时：若加入新的组播地址，即当前 **OpenFlow** 网络中还没有该组播，那以直连路由器的 **OF** 交换机作为目的节点，生成最优转发路径；若该组播在网络中已经存在，那以直连路由器的 **OF** 交换机作为根节点，组播转发树中所有的 **OF** 交换机作为目的节点，利用 **Dijkstra** 算法计算得到离根节点最近的目的节点的路径，然后将该路径发现加入组播转发树，形成新枝。
* 有组成员离开时：若该组成员是该组播组的最后一个成员，则直接删除所有的转发路径；若不是，则对转发树进行枝剪，从叶子结点向根节点进行反向删除，一直删除到有分叉的节点。

### 组播服务质量



## 系统设计实现

### 总体设计

![sd](https://github.com/BokalaQuan/RyuNote/blob/master/%E7%B3%BB%E7%BB%9F%E8%AE%BE%E8%AE%A1.png)

### 拓扑发现模块

| 接口名称                      | 调用方法                               | 功能                        |
| :------------------------ | ---------------------------------- | ------------------------- |
| get all the switches      | GET /v1.0/topology/switches        | 获取OpenFlow网络中所有OF交换机的信息   |
| get the switch            | GET /v1.0/topology/switches/<dpid> | 获取编号为dpid的OF交换机信息         |
| get all the links         | GET /v1.0/topology/links           | 获取OpenFlow网络中所有OF交换机的连接信息 |
| get the links of a switch | GET /v1.0/topology/links/<dpid>    | 获取编号为dpid的OF交换机的连接信息      |

