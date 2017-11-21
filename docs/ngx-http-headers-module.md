# ngx_http_headers_module

## 概述

该模块用于设置响应头部，包括 `Expires` 、`Cache-Control` 以及其他任意头部。

## 指令

### add_header

> Syntax: add_header *name* *value* [always];  
> Default: --  
> Context: http, server, location, if in location

增加指定头部字段到响应头部，**只有**以下响应状态才会设置：200,201,204,206,301,302,303,304,307,308。如果设置了 `always` 参数，那么任何响应状态都会设置头部。

可以同时存在多个 `add_header` 。

如果当前上下文**不含**任何 `add_header` 指令，那么会**继承**父级上下文的头部设置；如果当前上下文**包含** `add_header` 指令，那么父级上下文中**使用 `add_header` 设置**的头部对当前上下文无效。（可能不太好理解，后文会有示例说明）

### add_trailer

与 `add_header` 类似，只不过是在**分块传输**的报文实体后面加上字段信息，如 `Expires` 。Nginx 1.13.2 版本开始支持该指令

### expires

> Syntax: expires [modified] *time*; expires epoch | max | off;  
> Default: expires off;  
> Content: http, server, location, if in location  

该指令用于添加或者修改 `Expires` 和 `Cache-Control` 响应头部字段，当且仅当响应码为 200,201,204,206,301,302,303,304,307 和 308。

> HTTP/1.0 只支持 `Expires` ， HTTP/1.1 引入新的缓存头部 `Cache-Control`，为了兼容性，nginx 在设置 `Expires` 时也会自动设置相对应的 `Cache-Control`，开发者可以手动设置（如 `add_header Cache-Control public;`）以覆盖自动设置的值。

*time* 参数可以是正、负时间值。真正的过期时间点为**当前时间**(Date 字段)和 *time* 的总和。如果有 `modified` 参数，那么过期时间点为**文件修改时间**（Last-Modified 字段）和 *time* 的总和。

使用 `@` 设置一天当中的具体时间点，如 `expires @15h30m` 表示每天的 15:30 过期。

`expires epoch` 即 `Expires Thu, 01 Jan 1970 00:00:01 GMT`。`Cache-Control` 会按如下方式自动设置（目前都是设置为 `no-cache`）：

1. 当前时间早于“Thu, 01 Jan 1970 00:00:01 GMT”的话，设置为: `Cache-Control: max-age=t;`，其中 `t` 为时间差
2. 当前时间等于或者晚于“Thu, 01 Jan 1970 00:00:01 GMT”，设置为: `Cache-Control: no-cache;`

> Unix epoch 即 Unix 时间戳，表示从“Thu, 01 Jan 1970 00:00:00 GMT”开始所经过的**时间秒数**

`expires max` 与 `expires epoch` 相对，设置 `Expires: Thu, 31 Dec 2037 23:55:55 GMT` ，并且设置 `Cache-Control: 10y` 。

`expires off` 关闭 `Expires` 头部设置，自然也不会再自动设置 `Cache-Control` 头部。但是还是可以使用 `add_header` 来设置 `Cache-Control` 。

`expires` 也可以**使用变量**，如：

```nginx
map $sent_http_content_type $expires {
  default off;
  application/pdf 42d;
  ~image/ max;
}
expires $expires;
```

## 实践

### `add_header` 继承

如果当前上下文**不含**任何 `add_header` 指令，那么会**继承**父级上下文的头部设置；如果当前上下文**包含** `add_header` 指令，那么父级上下文中**使用 `add_header` 设置**的头部对当前上下文无效。

但是，隐式自动设置的头部还是会继承的。下面的示例中，`expires 30m;` 指令会自动设置 `Cache-Control: max-age=1800`。

```nginx
server {
  expires 30m;
  add_header Access-Control-Allow-Origin *;

  # 继承 server 上下文的头部设置，包括自动设置的头部，因此响应头部含: 
  location = /1 {
    empty_gif;
  }

  # server 上下文中使用 add_header 设置的头部失效，但是自动设置的不会失效，因此响应头部包含：

  location = /2 {
    add_header Access-Control-Allow-Headers 'X-Test';
    empty_gif;
  }
}
```
`server` 上下文中使用 `add_header` 显式设置了 `Access-Control-Allow-Origin` 头部，并且隐式设置了 `Expires` 和 `Cache-Control` 。

`curl --head localhost/1` 时，会继承 `server` 的显式 `add_header` 设置和隐式头部设置，因此返回的头部将包含：

```
Date: Tue, 21 Nov 2017 06:52:05 GMT
Expires: Tue, 21 Nov 2017 07:22:05 GMT
Cache-Control: max-age=1800
Access-Control-Allow-Origin: *
```

`curl --head localhost/2` 时，由于 `location = /2` 上下文使用 `add_header` 显式设置了 `Access-Control-Allow-Headers`，所以不会继承父级上下文 `server` 中显式设置的 `Access-Control-Allow-Origin` 。但是会继承隐式头部设置，因此返回的头部包含：

```
Date: Tue, 21 Nov 2017 06:52:05 GMT
Expires: Tue, 21 Nov 2017 07:22:05 GMT
Cache-Control: max-age=1800
Access-Control-Allow-Headers: X-Test
```

### 重复设置

重复设置相同头部不会合并为一个头部字段，而是会存在多个。

```nginx
server {
  expires 30m;
  location = /3 {
    add_header Cache-Control private;
    add_header Cache-Control max-age=0;
    empty_gif;
  }
}
```

`curl --head localhost/3` 返回的响应头部将包含：

```
Date: Tue, 21 Nov 2017 06:52:05 GMT
Expires: Tue, 21 Nov 2017 07:22:05 GMT
Cache-Control: max-age=1800
Cache-Control: private
Cache-Control: max-age=0
```

当遇到相同的头部时，有的客户端以最开始出现的为准，有的客户端以最后面出现的为准。因此，应尽量避免重复头部字段。

### 根据资源类型设置缓存时间

利用变量方式**实现针对不同类型的资源设置不同的缓存时间**。

```nginx
http {
  map $sent_http_content_type $expires {
    default off;
    text/html 30m;
    text/css max;
    application/javascript 1d;
    ~image/ max;
  }
  expires $expires;
}

```