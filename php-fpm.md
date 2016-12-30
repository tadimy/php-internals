# PHP-FPM 简介

PHP-FPM， 全称 PHP fastcgi process manager \( PHP fastcgi 进程管理器）。

前文介绍 fastcgi 的时候提到过 PHP-FPM，因为 fastcgi 和 传统的 cgi 相比多了常驻进程的特性，为了最大程度发挥服务器的性能，常用的做法是开启很多个常驻的 fastcgi 进程来处理任务，那如何让 Web 服务器能方便的跟所有的进程都能通信？

![](/assets/ngxin-fpm.png)在 PHP 5.4 中其实就已经自带了 php-fpm 来完成这一工作，它相当于 PHP 和 Web 服务器之间的一个中间件，主要的工作就是管理 php 的 fastcgi 进程。php-fpm 启动之后会根据配置

