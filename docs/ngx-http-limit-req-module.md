# ngx_http_limit_req_module

## 概述

`ngx_http_limit_req_module` 模块用来限制请求的处理速度，通常是用法是限制单个 IP 的请求数。该限制使用“桶漏算法”（[Leaky Bucket](http://leyew.blog.51cto.com/5043877/860302/)）实现。

## 指令

### limit_req_zone

> Syntax: limit_req_zone *key* zone=*name*:*size* rate=*rate*;  
> Default: --  
> Context: http

设置用来存放多个状态的共享内存区大小，特别是当前过载的请求数量。

*key* 可以是字符串、变量或者字符串和变量的组合，用来指明依据什么来区分请求。例如，*key* 为 `$binary_remote_addr` 时，表明依据 IP 区分请求，此时就会限制单个 IP 的请求数。

*name* 指定共享内存区的名称，用来给 `limit_req` 引用。

*size* 指定共享内存的大小。对于 `$binary_remote_addr` ， 32位系统能存储 16K 条状态记录，64位系统能存 8K 条记录。当共享内存被消耗殆尽时，请求将会返回 503 并中止。返回的错误码使用 `limit_req_status` 定义。

*rate* 指定限速大小，通常是每秒请求数 r/s ，还可以是 r/m 。

### limit_req

> Syntax: limit_req zone=*name* [burst=*number*] [nodelay];  
> Default: --  
> Context: http, server, location

设置共享内存区和请求数量的最大出血范围。同一个上下文环境，可以同时存在多个 `limit_req` 指令。如果当前上下文没有定义 `limit_req`，那么将继承上层环境的指令。例如，

```nginx
limit_req_zone $binary_remote_addr zone=per_ip:10m rate=1r/s;
limit_req_zone $server_name zone=per_server:10m rate=10r/s;
server {
  limit_req zone=per_ip burst=5 nodelay;
  limit_req zone=per_server burst=10;
}
```

该配置会同时从 IP 和 Server name 两个角度限制请求数量。

*name* 设置使用哪个限制规则，值必须是 `limit_req_zone` 中定义的 `zone` 值。

*number* 设置请求数量的出血范围，默认值为 0 。当请求数量超过限制时，超出部分将被延迟处理。如果延迟处理的请求数量又超过出血范围，那么超出出血部分的请求将被中止并返回错误码。

如果不想延迟处理超限的请求，那么可以设置为 `nodelay` 。

### limit_req_status

> Syntax: limit_req_status *code*;   
> Default: limit_req_status 503;  
> Content: http, server, location

*code* 设置请求被拒绝时使用的错误码。

### limit_req_log_level

> Syntax: limit_req_log_level info | notice | warn | error;   
> Default: limit_req_status error;  
> Content: http, server, location

设置当超过限速或者请求被延迟处理时的日志记录等级。对于被延迟处理的请求，其日志记录等级会比当前设定值低一个等级。例如，对于 `limit_req_log_level notice` ，延迟处理的请求会被记录为 `info` 等级。