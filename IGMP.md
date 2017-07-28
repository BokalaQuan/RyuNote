# IGMP

IGMP（因特网组管理协议）是用于管理子网之间的多播分组的目的地的协议。

基于组合总所有到的路由器连接的子网的组播路由器，查询该组播组是否加入宿主是本定期（IGMP查询报文）。如果已经加入多播组的主机存在于子网中的短语，并且报告主机是否正在参与任何多播组向组播路由器（IGMP报告消息）。多播路由器存储是否接收到来自任何子网被送往的报告，以确定“其多播组或朝向分组传送到任何子网寻址”。如果与否于该查询，或从从主机接收到的特定的多播组（IGMP离开消息）撤回消息来报告，该子网的组播路由器，全部或分组寻址到所述给定多播组它不会转移。

这种机制将不会是一个多播包被发送到子网中的多播组加入主机不存在，可以减少不必要的通信量。

![IGMP1](https://osrg.github.io/ryu-book/ja/html/_images/fig11.png)



## IGMP侦听

虽然我们可以通过使用IGMP，以减少子网单元上不必要的流量，也就是仍然在子网内产生不必要的流量的可能性。因为多播分组的目的地MAC地址是一个特殊值，它不是由L2交换机的MAC地址表，始终广播对象获知。因此，例如，也可以作为多播组加入宿主已经被连接到只有一个端口，L2交换机将转发组播数据包上的所有端口接收。

![IGMP2](https://osrg.github.io/ryu-book/ja/html/_images/fig2.png)



IGMP侦听，吸取由IGMP报告消息从所述组播组发送加入主机组播路由器L2交换机PEEK（探听）多播包的目的端口，它是一种技术，该技术。这种方法不会是一个组播数据包被发送至端口号多播组，即使在一个子网连接主机，可以减少不必要的流量。

![3](https://osrg.github.io/ryu-book/ja/html/_images/fig3.png)

L2交换机为IGMP侦听，即使它接收到正在参与同一个组播组从多个主机的IGMP报告消息，不转发一次或IGMP报告消息发送到查询器。另外，即使接收到来自所述主机留言一定IGMP，而存在正在参与同一多播组的其他主机，它不IGMP离开消息转发到查询器。其结果是，可以使看起来好像你正在做一台主机和IGMP消息发送到查询的交流，还可以防止不必要的IGMP报文的传输。



## Ryu应用程序执行

IGMP Snooping的功能是使用OpenFlow的实施，然后再尝试运行刘某的IGMP Snooping的应用。

这个程序是您添加IGMP侦听功能，以“交换中心”的应用程序。需要注意的是，在这种程序中，以处理 dpid = 0000000000000001的开关作为一个多播路由器，已经成立，处理与被连接到端口2作为组播服务器的主机。

程序名称：simple_switch_igmp_13.py

```python
class SimpleSwitchIgmp13(simple_switch_13.SimpleSwitch13):
    OFP_VERSIONS = [ofproto_v1_3.OFP_VERSION]
    _CONTEXTS = {'igmplib': igmplib.IgmpLib}

    def __init__(self, *args, **kwargs):
        super(SimpleSwitchIgmp13, self).__init__(*args, **kwargs)
        self.mac_to_port = {}
        self._snoop = kwargs['igmplib']
        self._snoop.set_querier_mode(
            dpid=str_to_dpid('0000000000000001'), server_port=2)

    @set_ev_cls(igmplib.EventPacketIn, MAIN_DISPATCHER)
    def _packet_in_handler(self, ev):
        msg = ev.msg
        datapath = msg.datapath
        ofproto = datapath.ofproto
        parser = datapath.ofproto_parser
        in_port = msg.match['in_port']

        pkt = packet.Packet(msg.data)
        eth = pkt.get_protocols(ethernet.ethernet)[0]

        dst = eth.dst
        src = eth.src

        dpid = datapath.id
        self.mac_to_port.setdefault(dpid, {})

        self.logger.info("packet in %s %s %s %s", dpid, src, dst, in_port)

        # learn a mac address to avoid FLOOD next time.
        self.mac_to_port[dpid][src] = in_port

        if dst in self.mac_to_port[dpid]:
            out_port = self.mac_to_port[dpid][dst]
        else:
            out_port = ofproto.OFPP_FLOOD

        actions = [parser.OFPActionOutput(out_port)]

        # install a flow to avoid packet_in next time
        if out_port != ofproto.OFPP_FLOOD:
            match = parser.OFPMatch(in_port=in_port, eth_dst=dst)
            self.add_flow(datapath, 1, match, actions)

        data = None
        if msg.buffer_id == ofproto.OFP_NO_BUFFER:
            data = msg.data

        out = parser.OFPPacketOut(datapath=datapath, buffer_id=msg.buffer_id,
                                  in_port=in_port, actions=actions, data=data)
        datapath.send_msg(out)

    @set_ev_cls(igmplib.EventMulticastGroupStateChanged,
                MAIN_DISPATCHER)
    def _status_changed(self, ev):
        msg = {
            igmplib.MG_GROUP_ADDED: 'Multicast Group Added',
            igmplib.MG_MEMBER_CHANGED: 'Multicast Group Member Changed',
            igmplib.MG_GROUP_REMOVED: 'Multicast Group Removed',
        }
        self.logger.info("%s: [%s] querier:[%s] hosts:%s",
                         msg.get(ev.reason), ev.address, ev.src,
                         ev.dsts)

```

> 在以后的说明的例子中，使用VLC（http://www.videolan.org/vlc/）来发送和接收组播数据包。 VLC安装，它不会在本文中关于视频可供讨论流。

### 实验环境搭建

要建立一个实验环境来检查IGMP Snooping的应用程序的操作。偏好或登录方法等用于VM图像的使用，请参阅“交换集线器”。

![4](https://osrg.github.io/ryu-book/ja/html/_images/fig4.png)

首先，使用Mininet创建拓扑如示于下图。具体参数如下：

| 参数         | 值            | 说明                     |
| ---------- | ------------ | ---------------------- |
| topo       | linear, 2, 3 | 两个交换机串联（每个交换机分别外接3台主机） |
| mac        | 无            | 自动设定主机的MAC地址           |
| switch     | ovsk         | 使用Open vSwitch         |
| controller | remote       | 使用外部控制器                |
| x          | 无            | 启用xTerm                |

```shell
$ sudo mn --topo linear,2,3 --mac --switch ovsk --controller remote -x
*** Creating network
*** Adding controller
Unable to contact the remote controller at 127.0.0.1:6633
*** Adding hosts:
h1s1 h1s2 h2s1 h2s2 h3s1 h3s2
*** Adding switches:
s1 s2
*** Adding links:
(h1s1, s1) (h1s2, s2) (h2s1, s1) (h2s2, s2) (h3s1, s1) (h3s2, s2) (s1, s2)
*** Configuring hosts
h1s1 h1s2 h2s1 h2s2 h3s1 h3s2
*** Running terms on localhost:10.0
*** Starting controller
*** Starting 2 switches
s1 s2

*** Starting CLI:
mininet>
mininet> net
h1s1 h1s1-eth0:s1-eth1
h1s2 h1s2-eth0:s2-eth1
h2s1 h2s1-eth0:s1-eth2
h2s2 h2s2-eth0:s2-eth2
h3s1 h3s1-eth0:s1-eth3
h3s2 h3s2-eth0:s2-eth3
s1 lo:  s1-eth1:h1s1-eth0 s1-eth2:h2s1-eth0 s1-eth3:h3s1-eth0 s1-eth4:s2-eth4
s2 lo:  s2-eth1:h1s2-eth0 s2-eth2:h2s2-eth0 s2-eth3:h3s2-eth0 s2-eth4:s1-eth4
c0
mininet>
```

### 配置IGMP版本

Ryu的IGMP监听应用只支持IGMP V1 / V2。每个主机将被设置为不使用IGMP V3。该命令输入，请在每个主机的xterm中。

host: h1s1:

```bash
# echo 2 > /proc/sys/net/ipv4/conf/h1s1-eth0/force_igmp_version			
```

host: h1s2:

```shell
# echo 2 > /proc/sys/net/ipv4/conf/h1s2-eth0/force_igmp_version
```

host: h2s1:

```shell
# echo 2 > /proc/sys/net/ipv4/conf/h2s1-eth0/force_igmp_version
```

host: h2s2:

```shell
# echo 2 > /proc/sys/net/ipv4/conf/h2s2-eth0/force_igmp_version
```

host: h3s1:

```shell
# echo 2 > /proc/sys/net/ipv4/conf/h3s-eth0/force_igmp_version
```

host: h3s2:

```shell
# echo 2 > /proc/sys/net/ipv4/conf/h3s2-eth0/force_igmp_version
```

### IP地址设定

在由Mininet自动分配IP地址，所有主机都属于同一个子网。为了建立一个不同的子网，重新分配每个主机上的IP地址。

host: h1s1:

```shell
# ip addr del 10.0.0.1/8 dev h1s1-eth0
# ip addr add 172.16.10.10/24 dev h1s1-eth0
```

host: h1s2:

```shell
# ip addr del 10.0.0.2/8 dev h1s2-eth0
# ip addr add 192.168.1.1/24 dev h1s2-eth0
```

host: h2s1:

```shell
# ip addr del 10.0.0.3/8 dev h2s1-eth0
# ip addr add 172.16.20.20/24 dev h2s1-eth0
```

host: h2s2:

```shell
# ip addr del 10.0.0.4/8 dev h2s2-eth0
# ip addr add 192.168.1.2/24 dev h2s2-eth0
```

host: h3s1:

```shell
# ip addr del 10.0.0.5/8 dev h3s1-eth0
# ip addr add 172.16.30.30/24 dev h3s1-eth0
```

host: h3s2:

```shell
# ip addr del 10.0.0.6/8 dev h3s2-eth0
# ip addr add 192.168.1.3/24 dev h3s2-eth0
```

### 设置默认网关

由于来自主机的IGMP报文能够成功传输，设置默认网关。

host: h1s1:

```shell
# ip route add default via 172.16.10.254
```

host: h1s2:

```shell
# ip route add default via 192.168.1.254
```

host: h2s1:

```shell
# ip route add default via 172.16.20.254
```

host: h2s2:

```shell
# ip route add default via 192.168.1.254
```

host: h3s1:

```shell
# ip route add default via 172.16.30.254
```

host: h3s2:

```shell
# ip route add default via 192.168.1.254
```

### 配置OpenFlow版本

switch: s1 (root):

```shell
# ovs-vsctl set Bridge s1 protocols=OpenFlow13
```

switch: s2 (root):

```shell
# ovs-vsctl set Bridge s2 protocols=OpenFlow13
```

### 交换机控制器的执行

既然准备好了，运行在Ryu应用程序。该命令输入，请继续控制器c0的xterm的。

controller: c0 (root):

```shell
# ryu-manager ryu.app.simple_switch_igmp_13
loading app ryu.app.simple_switch_igmp_13
loading app ryu.controller.ofp_handler
instantiating app None of IgmpLib
creating context igmplib
instantiating app ryu.app.simple_switch_igmp_13 of SimpleSwitchIgmp13
instantiating app ryu.controller.ofp_handler of OFPHandler
...
```

（要发送IGMP查询消息时，它被称为查询器）刚开始后，交换机S1是指示开始操作为将被输出多播路由器日志。

controller: c0 (root):

```shell
...
[querier][INFO] started a querier.
...
```

查询器发送一个IGMP查询消息在60秒内的所有端口，然后注册的流条目，以该多播分组从组播服务器传输到已经返回IGMP报告消息的端口。与此同时，除了查询将启动交换机监听IGMP报文。

controller: c0 (root):

```shell
...
[snoop][INFO] SW=0000000000000002 PORT=4 IGMP received. [QUERY]
...
```

上述日志，表示这是从交换机S1发送IGMP查询消息查询器交换机S2在端口4处被接收。S2将广播接收到的IGMP查询报文。

> 首先从查询到查询报文的可能无意中被发送之前可以窥探准备。在这种情况下，请等待60秒后发送下一个IGMP查询消息。

### 添加一个组播组

然后再加入每个主机的多播组。当您尝试播放流给在VLC特定组播地址，VLC将发送IGMP报告消息。

#### 将h1s2添加到225.0.0.1组

首先在主机h1s2，并设置为播放多播地址“225.0.0.1”的数据流处理。 VLC会从主机h1s2发送IGMP报告消息。

host: h1s2:

```shell
# vlc-wrapper udp://@225.0.0.1
```

S2将认识到，从主机接收h1s2端口1上的IGMP报告消息，接收到多播地址“225.0.0.1”是存在于先前端口1中的基团。

controller: c0 (root):

```shell
...
[snoop][INFO] SW=0000000000000002 PORT=1 IGMP received. [REPORT]
Multicast Group Added: [225.0.0.1] querier:[4] hosts:[]
Multicast Group Member Changed: [225.0.0.1] querier:[4] hosts:[1]
[snoop][INFO] SW=0000000000000002 PORT=1 IGMP received. [REPORT]
[snoop][INFO] SW=0000000000000002 PORT=1 IGMP received. [REPORT]
...
```

上述日志取与S2

- 它已收到的IGMP Report Message的端口1
- 它意识到的多播组的存在以接收多播地址“225.0.0.1”（即查询存在于先前端口4）
- 该基团的参与主机接收一个多播地址“225.0.0.1”是存在于先前端口1

在此之后，查询器继续一旦IGMP查询消息发送到60秒，组播组的加入已经接收到的消息发送每次IGMP报告消息主机h1s2。

controller: c0 (root):

```shell
...
[snoop][INFO] SW=0000000000000002 PORT=4 IGMP received. [QUERY]
[snoop][INFO] SW=0000000000000002 PORT=1 IGMP received. [REPORT]
...
```

尝试检查在该点登记在查询的流程条目。

switch: s1 (root):

```shell
# ovs-ofctl -O openflow13 dump-flows s1
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=827.211s, table=0, n_packets=0, n_bytes=0, priority=65535,ip,in_port=2,nw_dst=225.0.0.1 actions=output:4
 cookie=0x0, duration=827.211s, table=0, n_packets=14, n_bytes=644, priority=65535,ip,in_port=4,nw_dst=225.0.0.1 actions=CONTROLLER:65509
 cookie=0x0, duration=843.887s, table=0, n_packets=1, n_bytes=46, priority=0 actions=CONTROLLER:65535
```

流表分析

- 端口2（组播服务器）从225.0.0.1接收分组寻址到交换机s2的端口4用于转发
- 端口4（交换机s2）
- 类似表错过流条目为“交换集线器”

已注册的流条目中的三个。另外，尽量检查被登记到交换机S2流条目。

switch: s2 (root):

```shell
# ovs-ofctl -O openflow13 dump-flows s2
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=1463.549s, table=0, n_packets=26, n_bytes=1196, priority=65535,ip,in_port=1,nw_dst=225.0.0.1 actions=CONTROLLER:65509
 cookie=0x0, duration=1463.548s, table=0, n_packets=0, n_bytes=0, priority=65535,ip,in_port=4,nw_dst=225.0.0.1 actions=output:1
 cookie=0x0, duration=1480.221s, table=0, n_packets=26, n_bytes=1096, priority=0 actions=CONTROLLER:65535
```

- 对分组的In接收到分组的情况下，寻址从端口1到225.0.0.1（主机h1s2）
- 转移到端口1（主机h1s2），当它接收到一个数据包从端口4寻址到225.0.0.1（查询器）
- 类似表错过流条目为“交换集线器”

#### 将h3s2添加到225.0.0.1组

接下来，将发挥多播地址“225.0.0.1”的数据流给任何主机h3s2。 VLC会从主机h3s2发送IGMP报告消息。

host: h3s2:

```shell
# vlc-wrapper udp://@225.0.0.1
```

S2从主机h3s2收到IGMP报告消息在端口3，加入该组的主机接收一个多播地址“225.0.0.1”，将认识到，还存在提前口3到其它端口1 。

controller: c0 (root):

```shell
...
[snoop][INFO] SW=0000000000000002 PORT=3 IGMP received. [REPORT]
Multicast Group Member Changed: [225.0.0.1] querier:[4] hosts:[1, 3]
[snoop][INFO] SW=0000000000000002 PORT=3 IGMP received. [REPORT]
[snoop][INFO] SW=0000000000000002 PORT=3 IGMP received. [REPORT]
...
```

尝试检查在该点登记在查询的流程条目。

switch: s1 (root):

```shell
# ovs-ofctl -O openflow13 dump-flows s1
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=1854.016s, table=0, n_packets=0, n_bytes=0, priority=65535,ip,in_port=2,nw_dst=225.0.0.1 actions=output:4
 cookie=0x0, duration=1854.016s, table=0, n_packets=31, n_bytes=1426, priority=65535,ip,in_port=4,nw_dst=225.0.0.1 actions=CONTROLLER:65509
 cookie=0x0, duration=1870.692s, table=0, n_packets=1, n_bytes=46, priority=0 actions=CONTROLLER:65535
```

有在查询登记的流条目没有特别的变化。 另外，尽量检查被登记到交换机S2流条目。

switch: s2 (root):

```shell
# ovs-ofctl -O openflow13 dump-flows s2
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=1910.703s, table=0, n_packets=34, n_bytes=1564, priority=65535,ip,in_port=1,nw_dst=225.0.0.1 actions=CONTROLLER:65509
 cookie=0x0, duration=162.606s, table=0, n_packets=5, n_bytes=230, priority=65535,ip,in_port=3,nw_dst=225.0.0.1 actions=CONTROLLER:65509
 cookie=0x0, duration=162.606s, table=0, n_packets=0, n_bytes=0, priority=65535,ip,in_port=4,nw_dst=225.0.0.1 actions=output:1,output:3
 cookie=0x0, duration=1927.375s, table=0, n_packets=35, n_bytes=1478, priority=0 actions=CONTROLLER:65535
```

- 对分组的In接收到分组的情况下，寻址从端口1到225.0.0.1（主机H1s2）
- 对分组的In接收到分组的情况下，寻址从端口3（主机H3s2）到225.0.0.1
- 端口接收到分组时，4寻址从（查询），以225.0.0.1被转发到端口1（主机H1s2）和端口3（主机H3s2）
- 类似Table-miss流表

#### 将h2s2添加到225.0.0.2组

然后，将扮演不同的组播地址“225.0.0.2”的数据流在主机h2s2主机给其他。 VLC会从主机h2s2发送IGMP Report Message。

host: h2s2:

```shell
# vlc-wrapper udp://@225.0.0.2
```

controller: c0 (root):

```shell
...
[snoop][INFO] SW=0000000000000002 PORT=2 IGMP received. [REPORT]
Multicast Group Added: [225.0.0.2] querier:[4] hosts:[]
Multicast Group Member Changed: [225.0.0.2] querier:[4] hosts:[2]
[snoop][INFO] SW=0000000000000002 PORT=2 IGMP received. [REPORT]
[snoop][INFO] SW=0000000000000002 PORT=2 IGMP received. [REPORT]
...
```

switch: s1 (root):

```shell
# ovs-ofctl -O openflow13 dump-flows s1
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=2289.168s, table=0, n_packets=0, n_bytes=0, priority=65535,ip,in_port=2,nw_dst=225.0.0.1 actions=output:4
 cookie=0x0, duration=108.374s, table=0, n_packets=2, n_bytes=92, priority=65535,ip,in_port=4,nw_dst=225.0.0.2 actions=CONTROLLER:65509
 cookie=0x0, duration=108.375s, table=0, n_packets=0, n_bytes=0, priority=65535,ip,in_port=2,nw_dst=225.0.0.2 actions=output:4
 cookie=0x0, duration=2289.168s, table=0, n_packets=38, n_bytes=1748, priority=65535,ip,in_port=4,nw_dst=225.0.0.1 actions=CONTROLLER:65509
 cookie=0x0, duration=2305.844s, table=0, n_packets=2, n_bytes=92, priority=0 actions=CONTROLLER:65535
```

switch: s2 (root):

```shell
# ovs-ofctl -O openflow13 dump-flows s2
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=2379.973s, table=0, n_packets=41, n_bytes=1886, priority=65535,ip,in_port=1,nw_dst=225.0.0.1 actions=CONTROLLER:65509
 cookie=0x0, duration=199.178s, table=0, n_packets=0, n_bytes=0, priority=65535,ip,in_port=4,nw_dst=225.0.0.2 actions=output:2
 cookie=0x0, duration=631.876s, table=0, n_packets=12, n_bytes=552, priority=65535,ip,in_port=3,nw_dst=225.0.0.1 actions=CONTROLLER:65509
 cookie=0x0, duration=199.178s, table=0, n_packets=5, n_bytes=230, priority=65535,ip,in_port=2,nw_dst=225.0.0.2 actions=CONTROLLER:65509
 cookie=0x0, duration=631.876s, table=0, n_packets=0, n_bytes=0, priority=65535,ip,in_port=4,nw_dst=225.0.0.1 actions=output:1,output:3
 cookie=0x0, duration=2396.645s, table=0, n_packets=43, n_bytes=1818, priority=0 actions=CONTROLLER:65535
```

#### 将h3s1添加到225.0.0.1组

也将扮演多播地址“225.0.0.1”的数据流给任何主机h3s1。 VLC会从主机h3s1发送IGMP Report Message。

host: h3s1:

```shell
# vlc-wrapper udp://@225.0.0.1
```

主机h3s1未连接到开关S2。因此，不受IGMP侦听功能，使正常的IGMP数据包，并从查询的交流。尝试检查在该点登记在查询的流程条目。

switch: s1 (root):

```shell
# ovs-ofctl -O openflow13 dump-flows s1
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=12.85s, table=0, n_packets=0, n_bytes=0, priority=65535,ip,in_port=2,nw_dst=225.0.0.1 actions=output:3,output:4
 cookie=0x0, duration=626.33s, table=0, n_packets=10, n_bytes=460, priority=65535,ip,in_port=4,nw_dst=225.0.0.2 actions=CONTROLLER:65509
 cookie=0x0, duration=12.85s, table=0, n_packets=1, n_bytes=46, priority=65535,ip,in_port=3,nw_dst=225.0.0.1 actions=CONTROLLER:65509
 cookie=0x0, duration=626.331s, table=0, n_packets=0, n_bytes=0, priority=65535,ip,in_port=2,nw_dst=225.0.0.2 actions=output:4
 cookie=0x0, duration=2807.124s, table=0, n_packets=46, n_bytes=2116, priority=65535,ip,in_port=4,nw_dst=225.0.0.1 actions=CONTROLLER:65509
 cookie=0x0, duration=2823.8s, table=0, n_packets=3, n_bytes=138, priority=0 actions=CONTROLLER:65535
```

### 开始传递流

host: h2s1:

```shell
# vlc-wrapper sample.mov --sout udp:225.0.0.1 --loop
```

然后，参与该组播组的每个主机上运行的VLC是h1s2，h3s2，“225.0.0.1”的h3s1，视频都是由将播放该组播服务器分发。视频在h2s2参加“225.0.0.2”不能播放。

### 删除组播组

接着，以从多播组脱开每个主机。当您在流播放中退出VLC，VLC将发送IGMP Leave Message。

#### 从225.0.0.1组中删除主机h1s2

在主机上运行h1s2的VLC终止如按Ctrl + C.开关S2从主机h1s2接收到IGMP离开消息在端口1，加入该组的主机接收一个多播地址“225.0.0.1”，将认识到这是不再存在于先前的端口1。

controller: c0 (root):

```shell
...
[snoop][INFO] SW=0000000000000002 PORT=1 IGMP received. [LEAVE]
Multicast Group Member Changed: [225.0.0.1] querier:[4] hosts:[3]
...
```

switch: s1 (root):

```shell
# ovs-ofctl -O openflow13 dump-flows s1
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=1565.13s, table=0, n_packets=1047062, n_bytes=1421910196, priority=65535,ip,in_port=2,nw_dst=225.0.0.1 actions=output:3,output:4
 cookie=0x0, duration=2178.61s, table=0, n_packets=36, n_bytes=1656, priority=65535,ip,in_port=4,nw_dst=225.0.0.2 actions=CONTROLLER:65509
 cookie=0x0, duration=1565.13s, table=0, n_packets=27, n_bytes=1242, priority=65535,ip,in_port=3,nw_dst=225.0.0.1 actions=CONTROLLER:65509
 cookie=0x0, duration=2178.611s, table=0, n_packets=0, n_bytes=0, priority=65535,ip,in_port=2,nw_dst=225.0.0.2 actions=output:4
 cookie=0x0, duration=4359.404s, table=0, n_packets=72, n_bytes=3312, priority=65535,ip,in_port=4,nw_dst=225.0.0.1 actions=CONTROLLER:65509
 cookie=0x0, duration=4376.08s, table=0, n_packets=3, n_bytes=138, priority=0 actions=CONTROLLER:65535
```

switch: s2 (root):

```shell
# ovs-ofctl -O openflow13 dump-flows s2
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=2228.528s, table=0, n_packets=0, n_bytes=0, priority=65535,ip,in_port=4,nw_dst=225.0.0.2 actions=output:2
 cookie=0x0, duration=2661.226s, table=0, n_packets=46, n_bytes=2116, priority=65535,ip,in_port=3,nw_dst=225.0.0.1 actions=CONTROLLER:65509
 cookie=0x0, duration=2228.528s, table=0, n_packets=39, n_bytes=1794, priority=65535,ip,in_port=2,nw_dst=225.0.0.2 actions=CONTROLLER:65509
 cookie=0x0, duration=548.063s, table=0, n_packets=486571, n_bytes=660763418, priority=65535,ip,in_port=4,nw_dst=225.0.0.1 actions=output:3
 cookie=0x0, duration=4425.995s, table=0, n_packets=78, n_bytes=3292, priority=0 actions=CONTROLLER:65535
```

#### 从225.0.0.1组中删除主机h3s2

controller: c0 (root):

```shell
...
[snoop][INFO] SW=0000000000000002 PORT=3 IGMP received. [LEAVE]
Multicast Group Removed: [225.0.0.1] querier:[4] hosts:[]
...
```

switch: s1 (root):

```shell
# ovs-ofctl -O openflow13 dump-flows s1
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=89.891s, table=0, n_packets=79023, n_bytes=107313234, priority=65535,ip,in_port=2,nw_dst=225.0.0.1 actions=output:3
 cookie=0x0, duration=3823.61s, table=0, n_packets=64, n_bytes=2944, priority=65535,ip,in_port=4,nw_dst=225.0.0.2 actions=CONTROLLER:65509
 cookie=0x0, duration=3210.139s, table=0, n_packets=55, n_bytes=2530, priority=65535,ip,in_port=3,nw_dst=225.0.0.1 actions=CONTROLLER:65509
 cookie=0x0, duration=3823.467s, table=0, n_packets=0, n_bytes=0, priority=65535,ip,in_port=2,nw_dst=225.0.0.2 actions=output:4
 cookie=0x0, duration=6021.089s, table=0, n_packets=4, n_bytes=184, priority=0 actions=CONTROLLER:65535
```

switch: s2 (root):

```shell
# ovs-ofctl -O openflow13 dump-flows s2
OFPST_FLOW reply (OF1.3) (xid=0x2):
 cookie=0x0, duration=4704.052s, table=0, n_packets=0, n_bytes=0, priority=65535,ip,in_port=4,nw_dst=225.0.0.2 actions=output:2
 cookie=0x0, duration=4704.053s, table=0, n_packets=53, n_bytes=2438, priority=65535,ip,in_port=2,nw_dst=225.0.0.2 actions=CONTROLLER:65509
 cookie=0x0, duration=6750.068s, table=0, n_packets=115, n_bytes=29870, priority=0 actions=CONTROLLER:65535
```

## 在Ryu上安装IGMP监听应用

