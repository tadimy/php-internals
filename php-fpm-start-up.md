# PHP-FPM 启动分析

上一篇简单介绍了 PHP-FPM 的角色，这一篇将详细解析 PHP-FPM 启动的过程。

它的源代码在 php-src/sapi/fpm 目录中，因为 php 是用 C 语言写的，所以第一步就是找到 main\(\) 函数。fpm\_main.c 文件的 1571 行开始便是 main 函数的主体：

```c
int main(int argc, char *argv[])
{
    int exit_status = FPM_EXIT_OK;
    int cgi = 0, c, use_extended_info = 0;
    zend_file_handle file_handle;

    /* temporary locals */
    int orig_optind = php_optind;
    char *orig_optarg = php_optarg;
    int ini_entries_len = 0;

    int max_requests = 500; // 默认最大请求数
    int requests = 0;
    int fcgi_fd = 0;
    fcgi_request *request;
    char *fpm_config = NULL;
    char *fpm_prefix = NULL;
    char *fpm_pid = NULL;
    int test_conf = 0;
    int force_daemon = -1;
    int force_stderr = 0;
    int php_information = 0;
    int php_allow_to_run_as_root = 0;
    ...
}
```

上面这段是 main 函数变量初始化部分的代码，这里主要关注 max\_requests, requests, fcgi\_fd , \*request 这几个变量。

首先是 max\_requests，配置过 php-fpm 的开发者都应该知道可以在配置文件中配置 pm.max\_requests 的值，在文档中有详细的说明它的作用：

> 设置每个子进程重生之前服务的请求数。对于可能存在内存泄漏的第三方模块来说是非常有用的。如果设置为 '0' 则一直接受请求，等同于 PHP\_FCGI\_MAX\_REQUESTS 环境变量。默认值：0。

可以看到它的默认值是 500，按照文档中所描述的，如果处理的请求数达到 500 之后，会触发改进程的 “重生”，后面我们会介绍这个过程。

继续往下看：

```c
...
zend_signal_startup();

sapi_startup(&cgi_sapi_module);
cgi_sapi_module.php_ini_path_override = NULL;
cgi_sapi_module.php_ini_ignore_cwd = 1;
...
```

这一段主要是初始化 sapi 的代码，调用 sapi\__startup 函数，传入 &cgi\_sapi\_module。因为 fpm 实际上是一个 sapi 的 module, 而 sapi 的 module 是被定义好的一个数据结构，cgi\_sapi\_module 的初始化可以在 fpm\_main.c 中找到：_

```c
static sapi_module_struct cgi_sapi_module = {
	"fpm-fcgi",						/* name */
	"FPM/FastCGI",					/* pretty name */

	php_cgi_startup,				/* startup */
	php_module_shutdown_wrapper,	/* shutdown */

	sapi_cgi_activate,				/* activate */
	sapi_cgi_deactivate,			/* deactivate */

	sapi_cgibin_ub_write,			/* unbuffered write */
	sapi_cgibin_flush,				/* flush */
	NULL,							/* get uid */
	sapi_cgibin_getenv,				/* getenv */

	php_error,						/* error handler */

	NULL,							/* header handler */
	sapi_cgi_send_headers,			/* send headers handler */
	NULL,							/* send header handler */

	sapi_cgi_read_post,				/* read POST data */
	sapi_cgi_read_cookies,			/* read Cookies */

	sapi_cgi_register_variables,	/* register server variables */
	sapi_cgi_log_message,			/* Log message */
	NULL,							/* Get request time */
	NULL,							/* Child terminate */

	STANDARD_SAPI_MODULE_PROPERTIES
};
```

可以看出 cgi_sapi_module 的数据类型是 sapi_module_struct，这个数据类型是 PHP 的 SAPI 中定义的，是类似于 OOP 中 class 的东西。
 
