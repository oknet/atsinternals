# 基础组件：PluginVC

PluginVC 是以 PluginVCCore 作为核心，其内包含两个 PluginVC 成员，一个是 active_vc，一个是 passive_vc。

PluginVCCore 就像是一个缩小版的 IOCore Net Subsystem，它只负责管理 active_vc 和 passive_vc。

在 TSHttpTxnServerIntercept() 的实现中，HttpSM 会通过 PluginVCCore::connect_re() 获得成员 active_vc，并将其作为 ServerVC。

## 定义

## 参考资料

- [PluginVC.h](http://github.com/apache/trafficserver/tree/master/proxy/PluginVC.h)
- [PluginVC.cc](http://github.com/apache/trafficserver/tree/master/proxy/PluginVC.cc)
