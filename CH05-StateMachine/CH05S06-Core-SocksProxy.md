# 核心组件：SocksProxy

SocksProxy 是ATS作为Socks Proxy接收来自Socks Client请求的状态机。

在 SocksProxy 中，识别出满足下面条件的Socks请求后，会转入 HttpSM 进行处理：

- Socks命令是 CONNECT
- 目标IP是 IPv4
- 目标端口是 80

对于不满足条件的则根据 socks.config 和 records.config 中的配置对 Socks 请求进行透传（Tunnel）

## 定义

## 参考资料

- [SocksProxy.cc](http://github.com/apache/trafficserver/tree/master/proxy/SocksProxy.cc)
