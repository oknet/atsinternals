# 核心组件：HttpSessionManager

## 定义

单实例，只有一个 HttpSessionManager 类声明的对象 httpSessionManager：

httpSessionManager 是 ServerSessionPool 的管理者，实际上它跟 Processor 有相似之处。

由于 ServerSessionPool 在 ATS 中被设计为两层：

  - 一层是在线程中的，每个线程里有一个本地的 ServerSessionPool
  - 而在 httpSessionManager 中还包含了一个全局的 ServerSessionPool

```
HttpSessionManager httpSessionManager;
```

## 方法

### HttpSessionManager::init

```
void
HttpSessionManager::init()                                                                                                                                                                                                            
{
  m_g_pool = new ServerSessionPool;
}
```


## 参考资料

- [HttpProxyAPIEnums.h](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpProxyAPIEnums.h)
- [HttpSessionManager.h](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpSessionManager.h)
- [HttpSessionManager.cc](http://github.com/apache/trafficserver/tree/master/proxy/http/HttpSessionManager.cc)
