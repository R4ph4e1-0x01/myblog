# SSRF_by_SNI_proxy

## 概念梳理

### 何为TLS  SNI

Server Name Indication (SNI)  是HTTPS中TLS协议的一个插件功能。建立安全连接时，会初始化发送一个`ClientHello` 的报文，该报文可能包含SNI字段。这样使得服务器在响应 `ServerHello` 报文时会返回证书用以适配指定域名。

### 何为 SNI proxy

当反向代理服务器使用SNI指定的域名去查询后端时，我们就用到了 SNI proxy 。

一般有两种情况：

- 有 SSL 终端：通过 SNI proxy 建立TLS 连接，使用SNI指定的域名去查找对应后端，用代理传输明文数据流。
- 无 SSL终端：使用SNI指定的域名去查找对应后端，通过 SNI proxy 直接传输数据流，和TCP代理差不多。



### SNI proxy 配置

大部分反代都支持配置文件中配置 SNI proxy；甚至[Kubernetes](https://gist.github.com/kekru/c09dbab5e78bf76402966b13fa72b9d2#choose-upstream-based-on-domain-pattern)也可以。

以下是一个例子：

```
stream {
    map $ssl_preread_server_name $targetBackend {
        test1.example.com backend1:443;
        test2.example.com backend2:9999;
    }

    server {
        listen 443; 
        resolver 127.0.0.11;
        proxy_pass $targetBackend:443;       
        ssl_preread on;
    }
}
```

`$ssl_preread_server_name` 也就是对应`test1.example.com` 作为SNI代理需要查找的域名。`ssl_preread on` 表示开启了 SNI proxy；



### 配置导致的漏洞1

如下配置为例

```
stream {
    server {
        listen 443; 
        resolver 127.0.0.11;
        proxy_pass $ssl_preread_server_name:443;       
        ssl_preread on;
    }
}
```

可以通过用户输入的`$ssl_preread_server_name`随意控制对后端访问的域名；

尽管[RFC 6066](https://www.rfc-editor.org/rfc/rfc6066#page-6)在第三章中明确规定了不允许使用IP地址作为SNI的值，但实际上`$ssl_preread_server_name`确实可以是可以输入IP的，甚至于一些特殊字符或者null字符。



### 配置导致的漏洞2

如下配置为例

```
stream {
    map $ssl_preread_server_name $targetBackend {
        ~^www.example\.com    $ssl_preread_server_name;
    }  

    server {
        listen 443; 
        resolver 127.0.0.11;
        proxy_pass $targetBackend:443;       
        ssl_preread on;
    }
}
```

由于正则表达式的缺陷，没有使用`$`符号强制规定正则开头，可以通过自己的DNS服务器来把 *www.example.com.attacker.com* 这样的域名指向127.0.0.1

### 其他一些潜在利用

SNI 代理仅检查第一条消息，然后代理所有后续流量，即使它不是正确的 TLS 消息。此外，虽然 RFC 规定只能有一个 SNI 字段，但实际上，我们可以发送多个不同的名称（[TLS-Attack](https://github.com/tls-attacker/TLS-Attacker) 在这里是一个方便的工具）。由于 Nginx 只检查第一个值，因此如果后端接受此类消息但随后使用第二个 SNI 值，则（理论上）可能有获得一些额外访问权限的途径。比如连续两个`ClientHello ClientHello`消息。

参考资料：

https://www.xf1433.com/4583.html

https://www.rfc-editor.org/rfc/rfc6066#page-6

https://www.bamsoftware.com/computers/sniproxy/

https://github.com/tls-attacker/TLS-Attacker

https://www.invicti.com/blog/web-security/ssrf-vulnerabilities-caused-by-sni-proxy-misconfigurations/
