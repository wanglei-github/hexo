---
title: HttpClient针对IPV6的支持
date: 2018-08-30 18:13:37
tags: [ipv6,httpclient]
---


ipv6会逐渐代替ipv4协议，对于一些插件使用，若不及时更新版本，会有意想不到的问题。比如java中常用的httplcient组件。
## HttpClient 版本不能低于4.3.6


**原因**：在4.3.6中，httpclient新增了对于异常NoRouteToHostException的处理,低于4.3.6的httpclient会忽视这个异常，导致链接失败。

## NoRouteToHostException的异常处理

### 源码4.3.6

```
class HttpClientConnectionOperator {
    ...省略其他代码
   //关注此方法的异常处理
	public void connect(ManagedHttpClientConnection conn, HttpHost host, InetSocketAddress localAddress, int connectTimeout, SocketConfig socketConfig, HttpContext context) throws IOException {
        Lookup<ConnectionSocketFactory> registry = this.getSocketFactoryRegistry(context);
        ConnectionSocketFactory sf = (ConnectionSocketFactory)registry.lookup(host.getSchemeName());
        if (sf == null) {
            throw new UnsupportedSchemeException(host.getSchemeName() + " protocol is not supported");
        } else {
        	  若host包含ipv4和ipv6，则addresses会包含2个
            InetAddress[] addresses = host.getAddress() != null ? new InetAddress[]{host.getAddress()} : this.dnsResolver.resolve(host.getHostName());
            int port = this.schemePortResolver.resolve(host);

            for(int i = 0; i < addresses.length; ++i) {
                InetAddress address = addresses[i];
                boolean last = i == addresses.length - 1;
                Socket sock = sf.createSocket(context);
                sock.setSoTimeout(socketConfig.getSoTimeout());
                sock.setReuseAddress(socketConfig.isSoReuseAddress());
                sock.setTcpNoDelay(socketConfig.isTcpNoDelay());
                sock.setKeepAlive(socketConfig.isSoKeepAlive());
                int linger = socketConfig.getSoLinger();
                if (linger >= 0) {
                    sock.setSoLinger(linger > 0, linger);
                }

                conn.bind(sock);
                InetSocketAddress remoteAddress = new InetSocketAddress(address, port);
                if (this.log.isDebugEnabled()) {
                    this.log.debug("Connecting to " + remoteAddress);
                }

                try {
                    sock = sf.connectSocket(connectTimeout, sock, host, remoteAddress, localAddress, context);
                    conn.bind(sock);
                    if (this.log.isDebugEnabled()) {
                        this.log.debug("Connection established " + conn);
                    }

                    return;
                } catch (SocketTimeoutException var19) {
                    if (last) {
                        throw new ConnectTimeoutException(var19, host, addresses);
                    }
                } catch (ConnectException var20) {
                    if (last) {
                        String msg = var20.getMessage();
                        if ("Connection timed out".equals(msg)) {
                            throw new ConnectTimeoutException(var20, host, addresses);
                        }

                        throw new HttpHostConnectException(var20, host, addresses);
                    }
                    //此处新增了对于NoRouteToHostException的处理，若访问的host同时拥有ipv4和ipv6，但链接不支持ipv6，会抛出此异常。若不加以处理，则直接会直接报错，退出。
                } catch (NoRouteToHostException var21) {
                    if (last) {
                        throw var21;
                    }
                }

                if (this.log.isDebugEnabled()) {
                    this.log.debug("Connect to " + remoteAddress + " timed out. " + "Connection will be retried using another IP address");
                }
            }

        }
    }
}
```
### 源码4.3.5
```
class HttpClientConnectionOperator {
  ...省略其他代码
  public void connect(
            final ManagedHttpClientConnection conn,
            final HttpHost host,
            final InetSocketAddress localAddress,
            final int connectTimeout,
            final SocketConfig socketConfig,
            final HttpContext context) throws IOException {
        final Lookup<ConnectionSocketFactory> registry = getSocketFactoryRegistry(context);
        final ConnectionSocketFactory sf = registry.lookup(host.getSchemeName());
        if (sf == null) {
            throw new UnsupportedSchemeException(host.getSchemeName() +
                    " protocol is not supported");
        }
         //若host包含ipv4和ipv6，则addresses会包含2个
        final InetAddress[] addresses = this.dnsResolver.resolve(host.getHostName());
        final int port = this.schemePortResolver.resolve(host);
        for (int i = 0; i < addresses.length; i++) {
            final InetAddress address = addresses[i];
            final boolean last = i == addresses.length - 1;

            Socket sock = sf.createSocket(context);
            sock.setSoTimeout(socketConfig.getSoTimeout());
            sock.setReuseAddress(socketConfig.isSoReuseAddress());
            sock.setTcpNoDelay(socketConfig.isTcpNoDelay());
            sock.setKeepAlive(socketConfig.isSoKeepAlive());
            final int linger = socketConfig.getSoLinger();
            if (linger >= 0) {
                sock.setSoLinger(linger > 0, linger);
            }
            conn.bind(sock);

            final InetSocketAddress remoteAddress = new InetSocketAddress(address, port);
            if (this.log.isDebugEnabled()) {
                this.log.debug("Connecting to " + remoteAddress);
            }
            try {
                sock = sf.connectSocket(
                        connectTimeout, sock, host, remoteAddress, localAddress, context);
                conn.bind(sock);
                if (this.log.isDebugEnabled()) {
                    this.log.debug("Connection established " + conn);
                }
                return;
            } catch (final SocketTimeoutException ex) {
                if (last) {
                    throw new ConnectTimeoutException(ex, host, addresses);
                }
            //相对于源码4.3.6 这里缺少了对于 NoRouteToHostException 异常处理，导致链接失败。
            } catch (final ConnectException ex) {
                if (last) {
                    final String msg = ex.getMessage();
                    if ("Connection timed out".equals(msg)) {
                        throw new ConnectTimeoutException(ex, host, addresses);
                    } else {
                        throw new HttpHostConnectException(ex, host, addresses);
                    }
                }
            }
            if (this.log.isDebugEnabled()) {
                this.log.debug("Connect to " + remoteAddress + " timed out. " +
                        "Connection will be retried using another IP address");
            }
        }
    }
}
```
***概括一下：方法connect会对InetAddress[]进行一次循环，源码4.3.5在未对 NoRouteToHostException 进行处理，则方法会直接退出，并不会尝试第二个地址链接。***

## JVM对IPV4和IPV6的选择
***java.net.preferIPv4Stack (default: false)*** If IPv6 is available on the operating system the underlying native socket will be, by default, an IPv6 socket which lets applications connect to, and accept connections from, both IPv4 and IPv6 hosts. However, in the case an application would rather use IPv4 only sockets, then this property can be set to true. The implication is that it will not be possible for the application to communicate with IPv6 only hosts.


***java.net.preferIPv6Addresses (default: false)*** When dealing with a host which has both IPv4 and IPv6 addresses, and if IPv6 is available on the operating system, the default behavior is to prefer using IPv4 addresses over IPv6 ones. This is to ensure backward compatibility, for example applications that depend on the representation of an IPv4 address (e.g. 192.168.1.1). This property can be set to true to change that preference and use IPv6 addresses over IPv4 ones where possible.
