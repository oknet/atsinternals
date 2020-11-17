# 基础组件：HostDBApplicationInfo

在 HostDBInfo 里，预留了 8 字节空间，用于提供给使用 HostDB 的业务系统存储域名解析结果之外的信息。

目前这 8 字节空间仅用于存储：

- 目标服务器最近一次不可用的时间，以及可以支持的 HTTP 协议的版本号
- Round Robin 数据在 HEAP 区的位置

## 定义

```
//
// This structure contains the host information required by
// the application.  Except for the initial fields it
// is treated as opacque by the database.
//

union HostDBApplicationInfo {
  // 空间占位，既该联合最大长度为 8 字节，HostDBApplicationInfo 是 HostDBInfo 的成员，
  // 而 HostDBInfo 是 MultiCache 的一个 Block，其必须是固定长度的，所以这里使用占位符声明 8 字节空间，
  // HostDBInfo 通过对 application1 和 application2 赋值为 0 来清除该联合所存储的数据。
  struct application_data_allotment {
    unsigned int application1;
    unsigned int application2;
  } allotment;

  /*
   * 下面为各类型应用可能在该 8 字节的空间内存储的数据格式的定义
   */

  //////////////////////////////////////////////////////////
  // http server attributes in the host database          //
  //                                                      //
  // http_version       - one of HttpVersion_t            //
  // pipeline_max       - max pipeline.     (up to 127).  //
  //                      0 - no keep alive               //
  //                      1 - no pipeline, only keepalive //
  // keep_alive_timeout - in seconds. (up to 63 seconds). //
  // last_failure       - UNIX time for the last time     //
  //                      we tried the server & failed    //
  // fail_count         - Number of times we tried and    //
  //                       and failed to contact the host //
  //////////////////////////////////////////////////////////
  // 对于 http_data，实际上只用到了 http_version 和 last_failure 两个成员
  // 仅当 HostDBInfo::round_robin 为 false 时有效
  struct http_server_attr {
    unsigned int http_version : 3;
    unsigned int pipeline_max : 7;
    unsigned int keepalive_timeout : 6;
    unsigned int fail_count : 8;
    unsigned int unused1 : 8;
    unsigned int last_failure : 32;
  } http_data;

  enum HttpVersion_t {
    HTTP_VERSION_UNDEFINED = 0,
    HTTP_VERSION_09 = 1,
    HTTP_VERSION_10 = 2,
    HTTP_VERSION_11 = 3,
  };

  // 当 HostDBInfo::round_robin 为 true 时，app.rr.offset 用于获得 RoundRobin 数据在 HEAP 区的存储位置，
  // 个人认为该成员应该放在 HostDBInfo::data.rr_offset 与 HostDBInfo::data.hostname_offset 并列比较合适。
  struct application_data_rr {
    int offset;
  } rr;
};
```


## 参考资料

- [I_HostDBProcessor.h](http://github.com/apache/trafficserver/tree/6.0.x/iocore/hostdb/I_HostDBProcessor.h)

