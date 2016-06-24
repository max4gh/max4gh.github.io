##关于ThinkPHP大数据量display出现页面空白的bug
###1.问题重现
	操作： 进入订单列表页面，选择每页显示2000条数据，然后点击查询
	期待： 能正常显示数据
	现状： 页面出现空白，无任何输出，亦无错误提示
###2.问题排查
通过 curl 请求：

	 curl 'http://boss.fcuh.com/Order/index.html?order=order_id&order_by=desc&status=&page_size=2000&date_type=create_time&start_date=&end_date=&pay_type=-1&order_type=-1&keyword_type=order_id&keyword=' -H 'Accept-Encoding: gzip, deflate, sdch' -H 'Accept-Language: zh-CN,zh;q=0.8' -H 'Upgrade-Insecure-Requests: 1' -H 'User-Agent: Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/51.0.2704.63 Safari/537.36' -H 'Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8' -H 'Referer: http://boss.fcuh.com/Order/index.html' -H 'Cookie: pgv_pvi=4482610176; PHPSESSID=kme0c5v92qcap4ckgagr2eeoc5' -H 'Connection: keep-alive' --compressed
有数据返回，但数据不完整，最后报一个错误：
 	
	curl: (18) transfer closed with outstanding read data remaining

以上错误指出，是在服务器给客户端浏览器传输数据的过程当中，连接被断开了。
再经过一轮的google发现，问题应该出现在 http header 上，以下是浏览器中看到的 Response Headers

    Cache-control:private
	Connection:keep-alive
	Content-Encoding:gzip
	Content-Type:text/html; charset=utf-8
	Date:Mon, 20 Jun 2016 06:48:14 GMT
	Expires:Thu, 19 Nov 1981 08:52:00 GMT
	Pragma:no-cache
	Server:nginx/1.8.0
	Transfer-Encoding:chunked
	X-Powered-By:ThinkPHP

首页发现的是Response Headers里面少了 Content-Length,接着google了一翻，具体可以参考这篇文章:[Nginx与HTTP协议 content-length](http://blog.csdn.net/sosospicy/article/details/9066547)。
简单来讲就是，因为浏览器未能获取到服务传输数据是真实长度，所以浏览器在接收到部分数据的时候就断开了，导致接收到的数据不完整，不完整的数据，浏览器无法渲染，So, 页面就空白了。

###3.调试
a. 追踪到TP的页面输出代码：

	/**
     * 输出内容文本可以包括Html
     * @access private
     * @param string $content 输出内容
     * @param string $charset 模板输出字符集
     * @param string $contentType 输出类型
     * @return mixed
     */
	private function render($content,$charset='',$contentType=''){
        if(empty($charset))  $charset = C('DEFAULT_CHARSET');
        if(empty($contentType)) $contentType = C('TMPL_CONTENT_TYPE');
        // 网页字符编码
        header('Content-Type:'.$contentType.'; charset='.$charset);
        header('Cache-control: '.C('HTTP_CACHE_CONTROL'));  // 页面缓存控制
        header('X-Powered-By:ThinkPHP');
        // 输出模板文件
        echo $content;
    }

在页面输出前设置header Content-Length,修改如下：

	/**
     * 输出内容文本可以包括Html
     * @access private
     * @param string $content 输出内容
     * @param string $charset 模板输出字符集
     * @param string $contentType 输出类型
     * @return mixed
     */
    private function render($content,$charset='',$contentType=''){
        if(empty($charset))  $charset = C('DEFAULT_CHARSET');
        if(empty($contentType)) $contentType = C('TMPL_CONTENT_TYPE');
        // 网页字符编码
        header('Content-Type:'.$contentType.'; charset='.$charset);
        header('Cache-control: '.C('HTTP_CACHE_CONTROL'));  // 页面缓存控制
        header('Content-Length:'. strlen($content)); //设置输出内容长度
        header('X-Powered-By:ThinkPHP');
        // 输出模板文件
        echo $content;
    }

修改之后，服务器要清空下缓存。本地测试，修改过代码之后，没有出现异常。
浏览器查看Response Headers：
	
	Cache-Control:private
	Connection:Keep-Alive
	Content-Length:20046
	Content-Type:text/html; charset=utf-8
	Date:Mon, 20 Jun 2016 06:12:10 GMT
	Expires:Thu, 19 Nov 1981 08:52:00 GMT
	Keep-Alive:timeout=5, max=100
	Pragma:no-cache
	Server:Apache/2.4.9 (Win32) PHP/5.5.12
	X-Powered-By:ThinkPHP

看到了Content-Length:20046。

但服务器上却页面依然显示空白。再次在浏览器中查看Response Headers，依然没有出现 Content-Length。

b. 因为本地测试Web服务器用的Apache,服务器用的是Nginx,所以考虑到这应该是服务器的区别。
又google后发现Nginx在开启gzip的情况下，默认是使用Transfer-Encoding:chunked这种方式获取传输数据长度的。
既然这样就好办了，把nginx配置文件里
	 
	#gzip on

这一行注释掉，再重启nginx。再次测试下，页面正常了。查看浏览器 Response Headers:
	
	Cache-control:private
	Connection:keep-alive
	Content-Length:2690331
	Content-Type:text/html; charset=utf-8
	Date:Mon, 20 Jun 2016 06:33:09 GMT
	Expires:Thu, 19 Nov 1981 08:52:00 GMT
	Pragma:no-cache
	Server:nginx/1.8.0
	X-Powered-By:ThinkPHP

可见，Content-Length:2690331 。数据量比较大，页面反应比较慢，财务那边使用反馈需要2-3分钟才能显示出来。
查询数据这块，还有优化空间。

###4. 总结
解决这个问题分两步

1.设置http header 
	
	header('Content-Length:'. strlen($content)); //指定输出内容长度

2.关闭nginx gzip
	
	#gzip on

此方案只是解决页面正常显示问题，但为什么会出现这样的问题，还有待研究。
另外就是关闭了gzip后，由于服务器的数据不再压缩导致传输的数据量变大，响应变慢，这个也有待优化。

