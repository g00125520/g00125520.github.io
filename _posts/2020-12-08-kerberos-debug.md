---
title: Kerberos认证失败问题定位
description: analyze a kerberos authorize failed
categories:
  - micro-service
tags:
  - kerberos 
---

服务器上面需要通过Kerberos认证访问es服务器，在开发环境测试，可以正常获取认证，代码如下：

```java
System.setProperty("java.security.krb5.conf", "/kerberos/krb5.conf");
HttpHeaders httpHeaders = new HttpHeaders();
httpHeaders.add(HttpHeaders.CONTENT_TYPE, "application/json; charset=UTF-8");
HttpEntity httpEntity = new HttpEntity("", httpHeaders);
RestTemplate restTemplate = new KerberosRestTemplate("keytab", "userPrincipal");
ResponseEntity<T> responseEntity = restTemplate.exchange(url, method, httpEntity, clazz, Collections.EMPTY_MAP);
```

上面的代码并无特别之处，开发环境也可以正常通过认证并请求es服务器。但通过容器环境运行，则一直提示401，无法通过认证。检查了容器的/etc/hosts配置及容器时间都没有问题。通过添加：-Xdebug -Xrunjdwp:transport=dt_socket,address=7777,server=y,suspend=n，并通过nginx的stream指令代理，远程debug线上环境，也是报401，看不出什么原因导致。

```nginx
Stream {
  include /etc/nginx/conf.d/*.streamconf;
}
# jdb stream conf
server {
  listen 7777;
  proxy_pass 12.12.12.12:8888;
}
```

添加运行参数：-Dsun.security.krb5.debug=true；查看kerberos认证中间过程详细，发现：KRBError: error message is server not found in kerveros database，而sname中使用的ip而不是域名，认证服务器上面是用域名的，所以通过ip地址是无法找到的，代码中访问的url是ip地址而不是域名，虽然在/etc/hosts中进行了配置，但发送请求时，ip并没有转换为对应的域名，所以导致了错误。具体原因还不清楚。最后添加代码，将exchange方法中的url转换为正确的域名，认证通过。

```java
final InetAddress in = InetAddress.getByName(ip);
final String canonicalServer = in.getCanonicalHostName();
ResponseEntity<T> responseEntity = restTemplate.exchange(canonicalServer , method, httpEntity, clazz, Collections.EMPTY_MAP);
```
