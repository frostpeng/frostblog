title: 图片下载支持http2.0改造方案关键点
date: 2018-12-01 19:11:12
tags:
	- http2.0
categories: tech
comments: true
---
iOS结合版在手Q6.7.0版本已经上线了http2.0下的图片下载的改造，取得了比较好的效果；Android结合版7.2.0 也针对图片下载做了http2.0的相关改造，简单输出下方案。
###支持版本
相对于iOS必须在9.0以上版本才支持http2.0，Android侧需要5.0以上版本才能很好的支持ALPN(http2.0的协商协议，Android4.4到5.0之间有bug，SPDY使用的是NPN)的实现。
目前手q外网Android 5.0以上占比77.53%，加入版本控制即可。
###网络库的选择
####Httpclient VS HttpUrlConnection VS OKHttp
* Httpclient
目前是结合版downloader使用的网络库，主要是因为历史兼容性的考虑。google官方已经不推荐使用，Android6.0之后HttpClient是不是系统自带的了，不过它在最近的更新中将HttpClient的所有代码copy了一份进来，所以还能使用。目前也不太好支持http2.0/SPDY。

* Httpurlconnection
从Android4.4起内部实现已经变成了OkHttp，但是默认当前版本spdy/http2.0是被禁用掉的。
		https://android.googlesource.com/platform/external/okhttp/+/master/android/main/java/com/squareup/okhttp/HttpsHandler.java
		OkUrlFactory okUrlFactory = HttpHandler.createHttpOkUrlFactory(proxy);
		// All HTTPS requests are allowed.
		okUrlFactory.setUrlFilter(null);
		OkHttpClient okHttpClient = okUrlFactory.client();
		// Only enable HTTP/1.1 (implies HTTP/1.0). Disable SPDY / HTTP/2.0.
		okHttpClient.setProtocols(HTTP_1_1_ONLY);
		okHttpClient.setConnectionSpecs(Collections.singletonList(TLS_CONNECTION_SPEC));
		
* OKHttp 
Android官方实现用于替代HttpUrlConnection和Apache HttpClient,相关下载接口和原有的结合版DownLoader兼容难度比较小，支持spdy/http2.0，本身的连接池复用，gzip压缩，缓存响应数据等技术都比较成熟，目前手Q 已经有使用OKHttp2.5的jar包，可以适当改造，包大小的压力会小很多。

### 当前相册后台方案
#### 只支持Https,之前没有spdy的改造
虽然http2.0可以支持明文传输，但是现在主流的浏览器，客户端，服务端主要还是支持的基于TLS部署的http2.0协议，目前结合版相册图片下载的服务器也只能支持TLS下的http2.0。下载结合版相册图片时必须用https访问才能走http2.0协议。

###主要改造点
####IP直连下的SNI
因为目前的http2.0的实现是基于https的访问，之前Downloader实现的https是不会走IP直连策略的，现在希望提高https访问性能，需要使用IP直连策略。默认情况下IP直连无法通过证书校验，主要是因为传输的domain并没有使用Header中的域名host，而是直接用的IP，所以会出现domain不匹配的情况，导致SSL/TLS握手不成功。这里通过修改OKHttp创建SSLSocket的实现，将IP直接替换成原来的域名，再进行证书校验，就可以实现IP直连下的HTTPS访问。

		//okhttp/src/main/java/com/squareup/okhttp/Connection.java
		// Create the wrapper over the connected socket.
      sslSocket = (SSLSocket) sslSocketFactory.createSocket(
              socket, address.getUriHost(), address.getUriPort(), true /* autoClose */);
      try {
        sslSocket.setEnabledProtocols(new String[]{"TLSv1", "TLSv1.1", "TLSv1.2"});
      }catch(Exception e){
      }
      ConnectionSpec connectionSpec = connectionSpecSelector.configureSecureSocket(sslSocket);
      String currentHost=address.getUriHost();
      if (connectionSpec.supportsTlsExtensions()) {
	  //对比address的header中的host和url中的host，如果发现不一致，即ip直连，则sni host设置为header中的host
        if(address.headerHost!=null && address.headerHost!=""&& !address.headerHost.equals(currentHost)){
          currentHost=address.headerHost;
        }
        //sni host设置，不然无法通过校验
        Platform.get().configureTlsExtensions(
                sslSocket, currentHost, address.getProtocols());
      }

      // Force handshake. This can throw!
      sslSocket.startHandshake();
	  
####30X重定向访问
Downloader的处理逻辑，重定向后header中仍然会保留之前的host域名，服务器会因为header中保留的之前的host域名和url中的host不一致而出错，这里需要修改OKHttp源码来实现。

		//okhttp/src/main/java/com/squareup/okhttp/internal/http/HttpEngine.java
		 case HTTP_MULT_CHOICE:
			  case HTTP_MOVED_PERM:
			  case HTTP_MOVED_TEMP:
			  case HTTP_SEE_OTHER:
				// Does the client allow redirects?
				if (!client.getFollowRedirects()) return null;

				String location = userResponse.header("Location");
				if (location == null) return null;
				HttpUrl url = userRequest.httpUrl().resolve(location);

				// Don't follow redirects to unsupported protocols.
				if (url == null) return null;

				// If configured, don't follow redirects between SSL and non-SSL.
				boolean sameScheme = url.scheme().equals(userRequest.httpUrl().scheme());
				if (!sameScheme && !client.getFollowSslRedirects()) return null;

				// Redirects don't include a request body.
				Request.Builder requestBuilder = userRequest.newBuilder();
				if (HttpMethod.permitsRequestBody(userRequest.method())) {
				  requestBuilder.method("GET", null);
				  requestBuilder.removeHeader("Transfer-Encoding");
				  requestBuilder.removeHeader("Content-Length");
				  requestBuilder.removeHeader("Content-Type");
				}
				if (!sameConnection(url)) {
          //修改原有的直连30X问题，一旦碰到30x的问题，先把header中的host移除
          requestBuilder.removeHeader("Host");
          requestBuilder.removeHeader("Authorization");
        }
		
####线程池并发策略
Downloader因为历史原因一直使用的两个并发下载，http2.0相比http1.1的优势主要体现在多路复用上，这里需要逻辑中的资源锁，将http2.0情况下的并发提高到4。 对比iOS的策略是http1.1并发数系统限制最多为4，http2.0后放开并发数到6。
####多并发情况下OKHttp同一域名创建多个http2.0连接问题
OKHttp 2.5本身存在同一时间发起多个同一域名下的请求时会出现多个socket连接，这里的原因是因为第一个请求发起后，创建相应连接后加入连接池需要很短的时间，第二个请求同时发起，无法在连接池中找到可以复用的连接，所以会创建多个连接。
这里的解决方案是直接给创建连接并加入连接池加锁，避免同一时间同时创建多个连接。

		//okhttp/src/main/java/com/squareup/okhttp/internal/http/HttpEngine.java
		//防止多线程请求http2.0创建多个连接，因为创建connection不一定立马加入了pool，必须获取完立马塞到pool中，别的线程一定能获取到全部最新的连接池
			if(address !=null && address.getSslSocketFactory()!=null &&
					client.getProtocols()!=null && client.getProtocols().contains(Protocol.HTTP_2)) {
			  synchronized (client.getConnectionPool()) {
				connection = createNextConnection();
				Internal.instance.connectAndSetOwner(client, connection, this, networkRequest);
			  }
			}else{
			  connection = createNextConnection();
			  Internal.instance.connectAndSetOwner(client, connection, this, networkRequest);
			}
			
#### TLS完全握手改简短握手
![完全握手](http://i.imgur.com/VrZTzVb.jpg)
![简短握手](http://i.imgur.com/8eoAn5v.png)
* iOS采用的是SSLSession ID的复用策略，Android采用的是SSLSession Ticket的策略。session Ticket的服用方式可以减少TLS握手的RTT次数，减少耗时。
* 两者区别在于Session ticket较之Session ID优势在于服务器使用了负载均衡等技术的时候。Session ID往往是存储在一台服务器上，当我向不同的服务器请求的时候，就无法复用之前的加密参数信息，而Session ticket可以较好的解决此类问题，因为相关的加密参数信息交由客户端管理，服务器只要确认即可。

		 //okhttp/src/main/java/com/squareup/okhttp/internal/Platform.java
		 //通过反射调用setUseSessionTickets为true，启用SSLSession Ticket复用
		 @Override public void configureTlsExtensions(
				SSLSocket sslSocket, String hostname, List<Protocol> protocols) {
			  // Enable SNI and session tickets.
			  if (hostname != null) {
				setUseSessionTickets.invokeOptionalWithoutCheckedException(sslSocket, true);
				setHostname.invokeOptionalWithoutCheckedException(sslSocket, hostname);
			  }

			  // Enable ALPN.
			  if (setAlpnProtocols != null && setAlpnProtocols.isSupported(sslSocket)) {
				Object[] parameters = { concatLengthPrefixed(protocols) };
				setAlpnProtocols.invokeWithoutCheckedException(sslSocket, parameters);
			  }
			}
			
### 优化和测试数据分析
#### http2.0 VS http1.0
####劣势
* https比http慢 空间相册的http2.0是基于https实现的，这里会增加TLS的握手和加解密耗时，可以参考我下图的数据。

####优势
* 并发数增加，连接数减少 http2.0 每个IP只建立一个tcp连接，通过虚拟流的方式来并发发送http请求，RFC7540 建议协议实现者支持最大流并发数不小于 100。 而 http1.1 一般通过创建多个连接来实现并发请求，创建连接对服务器和客户端都有资源消耗，连接数一般都比较小，这里iOS改造前是4个并发，Android改造前是2个并发。iOS调整为http2.0后会使用6个并发，Android在http2.0情况下使用4个并发。
* TLS Session复用
* 头部压缩 索引表算法可以相比gzip等方式更有效率，相册请求头部内容较少，影响比较小。
* 无队首阻塞 http2.0 的流都是通过一个个小的二进制帧在 tcp 连接上发送，不同流的帧互不影响，所以无 http1.1 的队首阻塞。
* TCP连接竞争 多个tcp连接之间会互相竞争，影响对应的tcp慢启动时间，对于拥塞和丢包恢复速度快。
* https更安全，防止数据被挟持破坏。

####未使用的特性
* 优先级设置
* 服务器主动推送

###实验数据
使用腾讯云WeTestWifi下载100张相册图片，对比http,https，http2.0的下载时间。

####时间指标说明
* 总耗时(平均每次) 表示从第一个任务加入队列到最后一个任务下载完成的总时间/下载文件数;
* 等待时长(平均每次) 表示每个任务从加入队列到任务执行的平均等待时间；
* 执行时长(平均每次) 表示每个任务从开始执行到下载完成的时间（写入文件的时间去除）;
* 请求时长(平均每次) 包括创建连接，发出请求，读取response头部的时间；
* response读时长(平均每次) 表示读取response文件流的时间；
####实验数据
https://docs.google.com/spreadsheets/d/1x9gANKCzJkn0lZTg7J1yno2N9uxoSo-zgvPUr94DqbE/edit#gid=0
![实验数据](http://i.imgur.com/GjpuqbW.png)
#### 结论分析：
优化效果图如下
![实验数据](http://i.imgur.com/hbUakan.png)
* 1.https确实影响了下载速度，TLS握手和加解密数据会导致同并发下的https/http2.0比http1.1慢很多； 参考同并发下得https和http1.1的对比可以看出TLS会增加耗时。
* 2.http2.0提高并发后相比https/http1.1优势很明显； 参考图中任意网络延迟下的蓝色区域数据。
* 3.同一并发下，网络延迟高的情况下，http2.0相比https/http1.1的优势反而会减少； 参考同并发下网络延迟为200/400ms的时候，http2.0甚至比https还要慢。
* 4.具有并发数优势的http2.0相比http1.1在网络延迟高的情况下表现更优。 参考200/400ms延迟情况下，http2.0和http1.1并发分别为2,4的对比数据，延迟很高，但是优化的效果越好。
### 外网结合版720数据
![结合版720外网数据](720数据.png)
* 下载总耗时提升比较明显，外网提升11.34%左右；
* 成功率有所下降，分析原因是因为遍历网络策略调整的问题，结合版725已经做了部分优化来提高下载成功率，理论上成功率应该持平甚至有所优化；
* 下载总耗时（包含等待时间）的优化效果和iOS的20%以上有差距，原因暂时定位是iOS的并发数wifi下提高到6，非wifi提高到4，而Android目前是统一从2提高到4，有所差距。
##后续计划
* 1.目前手q的OKhttp版本2.5.0相对较老，对于http2.0的支持存在一些bug，比如刚才的多个socket连接的问题，新版在相关策略上有所优化，后续会考虑升级版本；
* 2.目前是使用的httpclient和OKhttp做的ABtest的方案，OKhttp相对httpclient也有很多细节上的优化，可以考虑全部切换到OKhttp，对于包大小和耗时应该都能有所优化；
* 3.持续分析外网的下载数据，提高对应的成功率和下载耗时；