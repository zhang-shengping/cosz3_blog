---
title: F5 Remote Logging
categories:
- F5
tags:
- F5
- self-note
---

![front](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/f5-remote-logging/front.jpg)

# F5 iRule logging 简介

## F5 local logging 方式

### local

iRule local log 方式将 log 信息保存在本地的 `/var/log/ltm` log 文件中。local log 不是被 TMM (`Traffic Management Microkernel`) 处理，而是由 HMS 处理。发出 log 的格式时 `standard syslog format`。

**local logging 语法举例**

```tcl
when HTTP_REQUEST {
log local0. “Requested hostname: [HTTP::host] from IP: [IP::local_addr]”
}
```

## F5 Rmote logging 方式

### Simple Remote Logging

iRule 同样支持将 log 发送到远端服务器上，这些 log 是被 TMM 处理，而不是 HMS (`Host Management Subsystem`) 处理。发出 log 的格式时 `standard syslog format`。log 方式使用的是 `UDP` 连接。

**simple remote logging 语法举例**

```tcl
# 抓取数据包中的 uri
when HTTP_REQUEST {
    set uri [HTTP::uri]
    log 10.10.1.20:20001 "Received Request for uri '$uri' from client.  Sending to server..."
}
```

**测试 fluentd 整理收集到的数据**

```bash
0234-10-28 03:02:10.000000000 -0752 mytag: {"process":"bigip tmm3[16485]:","rule":"Rule /Common/test2 <HTTP_REQUEST>:","message":"Received Request for uri '/' from client.  Sending to server..."}
0234-10-28 03:02:11.000000000 -0752 mytag: {"process":"bigip tmm1[16485]:","rule":"Rule /Common/test2 <HTTP_REQUEST>:","message":"Received Request for uri '/' from client.  Sending to server..."}
```

**kibana 上的现实**

![kibana log](https://github.com/zhang-shengping/cosz3_blog/raw/gh-pages/images/f5-remote-logging/kibana-log.jpg)

由于 log 消息是通过 TMM 处理，所以 log server `10.10.1.20` 必须是 TMM route 可达地址（简单来说就是 可以通过 iRule 所绑定的 listener 所在的 VIP 的 interface 到达 log server 的 IP 地址）。

针对不同版本的 BigIP 如何使用 log 函数，我们可以在编辑创建 BigIP iRule 时查看，`BIGIP-13.1.3-0.0.6` 如下：

```tcl
log
Generates and logs the specified message to the Syslog-ng utility. This
command works by performing variable expansion on the message as
defined for the HTTP profile Header Insert setting.
The log command can produce large amounts of output. Use with care in
production environments, especially where disk space is limited.
The syslog facility is limited to logging 1024 bytes per request.
Longer strings will be truncated.
The High Speed Logging feature offers the ability to send TCP or
UDP syslog messages from an iRule with very low CPU or memory overhead.
Consider using HSL instead of the default log command for remote
logging.

Syntax

log <message>

     * Logs the specified message to the syslog-ng utility. Log entries
       are written to the local system log (/var/log/ltm). (See Note below
       about supression.)

log [-noname] <facility>.[<level>] <message>

     * Logs the specified message to the syslog-ng utility at the
       specified facility & log level. The iRule name prefixing the
       message text may optionally suppressed by including the -noname
       option.

log [-noname] <remote_ip>[:<remote_port>] <facility>.[<level>] <message>

     * (LTM only) Logs the specified message directly to the specified IP
       address (and optional alternate port when specified) via UDP.
       Facility and/or level are required. The iRule name prefixing the
       message text may optionally suppressed by including the -noname
       option. <remote_ip> must be a TMM-routed address. If you must route
       specific messages to a remote address via the management interface,
       you must log locally. syslog-ng is able to route messages via both
       TMM and management interfaces using the standard syntax. You can
       define an appropriate filter and remote log destination in LTM's
       syslog-ng service.

   Note: There is a significant behavioral difference when the optional
   <facility>.<level> is specified. When iRule logs messages without the
   facility and/or level, they are rate-limited as a class and
   subsequently logged messages within the rate-limit period may be
   suppressed even though they are textually different. However, when the
   <facility> and/or <level> are specified, the log messages are not
   rate-limited (though syslog-ng will still perform suppression of
   repeated duplicates).

Examples
Log to the local facility with no duplicate message suppression:

log local0. "Found $isCard $type CC# $card_number"

Log in the default message format to a remote syslog server on the
default port:

when CLIENT_ACCEPTED {
log 172.27.31.10 local0.info "Client Connected, IP: [IP::client_addr]"
}
```

### HSL（High speed logging）

HSL 这种方式更高效，速度更快。HSL 也由 TMM 进行管理和发送 log 消息，所以要确保 log server IP 是 TMM interface 可达。HSL 需要和指定的 log server pool 建立连接来发送 log 消息，这种连接方式可以通过直接 open 的方式`HSL::open -proto <UDP|TCP> -pool <poolname>`，也可以使用 publisher 的方式 `HSL::open -proto <UDP|TCP> -pool <poolname>`。

注意这里的 `HSL` 和 `log` 命令是不同的，`HSL` 即可以建立 `TCP` 连接，也可以建立 `UDP` 连接，端口是由创建 log server pool 时定义的，而 `log` 只能使用 `best effort`的 UDP 连接，端口可以通过 `log`命令指定。

`publisher` 和 `pool` 是通过一种或多种 `type` 的 `log destination` 对象建立对应关系。一般情况下通过 `Remote HSL type` 的 `log destination` 发送出去的 log 不是 `standard syslog format`，可以通过 `remote syslog type` 的 `log destination` 进行整形后再将 `standard syslog format` 发给 log server。

**HSL 语法举例 (这个例子中使用的是 publisher 的方法，publisher 事先创建在 /Common partition 下)**

```tcl
# 抓取数据包中的 header 进行判断
when RULE_INIT {
    set static::general_remote_syslog_publisher "/Common/publisher-remote-syslog"
}
 
when CLIENT_ACCEPTED {
    set hsl [HSL::open -publisher $static::general_remote_syslog_publisher]
    HSL::send $hsl "Client connect from [IP::client_addr]:[TCP::client_port]"
}
 
when HTTP_REQUEST {
    if { [HTTP::header exists X-Forwarded-For] } {
        HSL::send $hsl "Client has X-Forwarded-For: [HTTP::header X-Forwarded-For]"
    }
    else {
        HSL::send $hsl "Client has no X-Forwarded-For"
    }
}
```

**测试 fluentd 整理后收到的数据**

```bas
[root@fluentd ~]# tailf /var/log/td-agent/td-agent.log
2019-10-28 03:43:01 -0700 [info]: #0 flushing all buffer forcedly
0234-10-28 06:12:49.000000000 -0752 mytag: {"process":"bigip tmm[16485]:","rule":"Client connect from","message":"10.10.1.19:50680"}
0234-10-28 06:12:49.000000000 -0752 mytag: {"process":"bigip tmm[16485]:","rule":"Client has no","message":"X-Forwarded-For"}
```

针对不同版本的 BigIP 如何使用 log 函数，我们可以在编辑创建 BigIP iRule 时查看，`BIGIP-13.1.3-0.0.6`如下：

```tcl
HSL::open
Open a handle for High Speed Logging communication. After creating the
connection, send data on the connection using HSL::send.

Syntax

HSL::open -proto <UDP|TCP> -pool <poolname>

     * Opens and returns a handle for High Speed Logging communication.
       The handle can be used with HSL::send to send data over a
       particular protocol (TCP or UDP) to a pool comprised of one or more
       logging servers.

HSL::open -publisher <publisher>

     * Opens and returns a handle for High Speed Logging communication for
       a log publisher configured in System->Logs->Configuration->Log
       Publishers. The handle should be used with the HSL::send
       command to send data to the publisher. introduced in v11.3

   Note: The protocol is case sensitive and must be specified in all
   uppercase letters. Prior to 11.1 the protocol value is not validated
   when an iRule is saved, but will cause a run-time error when executed
   for a connection if the protocol is not valid (UDP or TCP).
   The pool name is not validated when an iRule is saved but will cause a
   run-time error when executed if the pool does not exist.

Examples
#1
when CLIENT_ACCEPTED {
    set hsl [HSL::open -proto UDP -pool syslog_server_pool]
}
when HTTP_REQUEST {
   # Log HTTP request via syslog protocol as local7.info; see RFC 3164 for moreinfo
    HSL::send $hsl "<190> [IP::local_addr] [HTTP::uri]\n"
}

#2
when CLIENT_ACCEPTED {
    set hsl [HSL::open -publisher /Common/lpAll]
}
when HTTP_REQUEST {
    HSL::send $hsl "<190> [IP::client_addr]:[TCP::client_port]->[IP::local_addr]:[TCP::local_port]; [HTTP::host][HTTP::uri]"
}
```

# Reference

[Getting Started with iRules: Logging & Comments](https://devcentral.f5.com/s/articles/getting-started-with-irules-logging-comments-20406)

[log iRule events with HSL](https://devcentral.f5.com/s/feed/0D51T00006i7dV0SAI)

[Logging on BIG-IP](https://www.youtube.com/watch?v=dcg9fxwnNTo)


