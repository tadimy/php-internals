# Nginx + PHP-FPM 是怎么完成一次请求的处理的

```
Nginx + PHP-FPM 已经是当前互联网公司最流行的运行 PHP 网站的服务器软件组合，安装的方式各种各样，有根据需求自己下载源码编译安装，也有很成熟的一键安装包提供使用。那么一次请求是如何得到想要的响应的呢？整个流程大概如下图：
```

![](/assets/Nginx-PHP-Workflow.png)

大概流程就是：

* Nginx 监听 80 端口（也可以是其他端口）;
* Nginx 由 master 进程启动多个 Worker 处理 HTTP 请求，master 进程自己不处理请求;
* worker 根据 nginx.conf 的配置和请求内容决定下一步:
  * 有可能将请求发到别的端口上（代理转发proxy\_pass\)
  * 将请求发送给其他提供 socket 接口的进程（如：PHP-FPM\)
  * 直接返回
  * ...
* php-fpm 根据请求调用 php-cgi 执行 php 脚本；
* 将执行结果返回给 Nginx 处理请求的 Worker 进程。



