# Ryu中的内置APP

## simple_switch_xx.py

两层交换机

* 学习连接到交换机接口的主机MAC地址，并记录在MAC地址表中
* 对于已记录的MAC地址，若再收到该MAC地址的包，则转发到相对应的接口
* 对于没有制定目标地址的包，则执行Flooding

## simple_monitor_13.py

交换机的流量监控功能扩展

## simple_switch_rest_13.py

REST API的使用

1. MAC地址表获取的API
> 取得Switch Hub中存储的MAC地址表内容，以json的形式回传
2. MAC地址表注册的API
> 新增MAC地址，并加入至交换机的流表中


