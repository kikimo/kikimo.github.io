---
title: "为什么 DNS 中应该避免 CNAME 记录和 MX 记录共存"
date: 2020-01-31T21:02:45+08:00
draft: false
---

DNS 协议不允许 CNAME 记录和 MX 记录共存。
造成这种约束的主要原因在于：

1. DNS 会对 CNAME 记录走递归解析
2. CNAME 记录的优先级高于 MX 记录

递归 DNS 服务器在查询某个常规域名记录（非 CNAME 记录）时，
如果在本地 cache 中已有该域名有对应的 CNAME 记录，
则会开始用该别名记录来重启查询，
这样 MX 记录会被 CNAME 别名记录的 MX 记录所覆盖。
这个过程，如果我们把 MX 记录替换成 A 记录理解起来也许就更容易了。
实际上，不只是 MX 记录，CNAME 记录和其他非 CNAME 记录都会造成冲突，
除了特殊的 DNSSEC 记录。

以下摘自 [wiki CNAME record](https://en.wikipedia.org/wiki/CNAME_record)

> CNAME records are handled specially in the domain name system, and have several restrictions on their use. When a DNS resolver encounters a CNAME record while looking for a regular resource record, it will restart the query using the canonical name instead of the original name. (If the resolver is specifically told to look for CNAME records, the canonical name (right-hand side) is returned, rather than restarting the query.) The canonical name that a CNAME record points to can be anywhere in the DNS, whether local or on a remote server in a different DNS zone.
For example, if there is a DNS zone as follows:

```txt
NAME                    TYPE   VALUE
--------------------------------------------------
bar.example.com.        CNAME  foo.example.com.
foo.example.com.        A      192.0.2.23
```
> when an A record lookup for bar.example.com is carried out, the resolver will see a CNAME record and restart the checking at foo.example.com and will then return 192.0.2.23.

以下摘自 [cloudxns 的帮助文档](https://www.cloudxns.net/Support/detail/id/6.html)：

> an alias defined in a cname record must have no other resource records of other types (MX, A, etc.). (RFC 1034 section 3.6.2, RFC 1912 section 2.4) The exception is when DNSSec is being used, in which case there can be DNSSEC related records such as RRSIG, NSEC, etc. (RFC 2181 seciton 10.1)
