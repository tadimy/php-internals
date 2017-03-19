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

这一段主要是初始化 sapi 的代码，调用 sapi\_startup 函数，传入 &cgi\_sapi\_module。因为 fpm 实际上是一个 sapi 的 module, 而 sapi 的 module 是被定义好的一个数据结构，cgi\_sapi\_module 的初始化可以在 fpm\_main.c 中找到：

```c
static sapi_module_struct cgi_sapi_module = {
    "fpm-fcgi",                        /* name */
    "FPM/FastCGI",                    /* pretty name */

    php_cgi_startup,                /* startup */
    php_module_shutdown_wrapper,    /* shutdown */

    sapi_cgi_activate,                /* activate */
    sapi_cgi_deactivate,            /* deactivate */

    sapi_cgibin_ub_write,            /* unbuffered write */
    sapi_cgibin_flush,                /* flush */
    NULL,                            /* get uid */
    sapi_cgibin_getenv,                /* getenv */

    php_error,                        /* error handler */

    NULL,                            /* header handler */
    sapi_cgi_send_headers,            /* send headers handler */
    NULL,                            /* send header handler */

    sapi_cgi_read_post,                /* read POST data */
    sapi_cgi_read_cookies,            /* read Cookies */

    sapi_cgi_register_variables,    /* register server variables */
    sapi_cgi_log_message,            /* Log message */
    NULL,                            /* Get request time */
    NULL,                            /* Child terminate */

    STANDARD_SAPI_MODULE_PROPERTIES
};
```

可以看出 cgi\_sapi\_module 的数据类型是 sapi\_module\_struct，这个数据类型是 PHP 的 SAPI 中定义的，是类似于 OOP 中 class 的东西。而这个 sapi\_startup 函数做的主要事情是分配互斥量\(tsrm\_mutex\_alloc\)。互斥量主要为针对多线程准备的，而 fastcgi 模式运行 PHP 都是单线程，所以不存在多线程中出现临界资源的使用问题。

在此之后的很长一部分代码都是处理命令行模式运行时的输入参数，这一段先略过，直接跳到 cgi\_sapi\_module 的 startup 部分：

```c
/* startup after we get the above ini override se we get things right */
    if (cgi_sapi_module.startup(&cgi_sapi_module) == FAILURE) {
#ifdef ZTS
        tsrm_shutdown();
#endif
        return FPM_EXIT_SOFTWARE;
    }
```

根据上面提到的 sapi\_module\_struct 的定义和 cgi\_sapi\_module 初始化的结果，不难看出 startup 调用的实际上是 fpm\_main.c 中 php\_cgi\_startup 函数。而 php\_cgi\_startup 函数中主要做的事情就是调用 php 的 main.c 中定义的 php\_module\_startup 函数。像这种调用方式在 php 的实现中非常常见，保证了代码的鲁棒性。至于 php\_module\_startup 都干了哪些事情，后面再详细介绍，简而言之，该函数将会读取 php.ini 中的配置初始化 php 解释运行环境，主要包括：

* php 核心配置
* zend 配置
* php 扩展初始化及启动

```c
static int php_cgi_startup(sapi_module_struct *sapi_module) /* {{{ */
{
    if (php_module_startup(sapi_module, &cgi_module_entry, 1) == FAILURE) {
        return FAILURE;
    }
    return SUCCESS;
}
```

到目前为止，需要的东西都初始化过了，该进入 fpm 的正题了：

```c
if (0 > fpm_init(argc, argv, fpm_config ? fpm_config : CGIG(fpm_config), fpm_prefix, fpm_pid, test_conf, php_allow_to_run_as_root, force_daemon, force_stderr)) {

        if (fpm_globals.send_config_pipe[1]) {
            int writeval = 0;
            zlog(ZLOG_DEBUG, "Sending \"0\" (error) to parent via fd=%d", fpm_globals.send_config_pipe[1]);
            zend_quiet_write(fpm_globals.send_config_pipe[1], &writeval, sizeof(writeval));
            close(fpm_globals.send_config_pipe[1]);
        }
        return FPM_EXIT_CONFIG;
    }

    if (fpm_globals.send_config_pipe[1]) {
        int writeval = 1;
        zlog(ZLOG_DEBUG, "Sending \"1\" (OK) to parent via fd=%d", fpm_globals.send_config_pipe[1]);
        zend_quiet_write(fpm_globals.send_config_pipe[1], &writeval, sizeof(writeval));
        close(fpm_globals.send_config_pipe[1]);
    }
    fpm_is_running = 1;

    fcgi_fd = fpm_run(&max_requests);
    parent = 0;

    /* onced forked tell zlog to also send messages through sapi_cgi_log_fastcgi() */
    zlog_set_external_logger(sapi_cgi_log_fastcgi);

    /* make php call us to get _ENV vars */
    php_php_import_environment_variables = php_import_environment_variables;
    php_import_environment_variables = cgi_php_import_environment_variables;

    /* library is already initialized, now init our request */
    request = fpm_init_request(fcgi_fd);
```

首先便是 **fpm\_init**，然后是 **fpm\_run**，最后是 **fpm\_init\_request**。

```c
int fpm_init(int argc, char **argv, char *config, char *prefix, char *pid, int test_conf, int run_as_root, int force_daemon, int force_stderr) /* {{{ */
{
    fpm_globals.argc = argc;
    fpm_globals.argv = argv;
    if (config && *config) {
        fpm_globals.config = strdup(config);
    }
    fpm_globals.prefix = prefix;
    fpm_globals.pid = pid;
    fpm_globals.run_as_root = run_as_root;
    fpm_globals.force_stderr = force_stderr;

    if (0 > fpm_php_init_main()           ||
        0 > fpm_stdio_init_main()         ||
        0 > fpm_conf_init_main(test_conf, force_daemon) ||
        0 > fpm_unix_init_main()          ||
        0 > fpm_scoreboard_init_main()    ||
        0 > fpm_pctl_init_main()          ||
        0 > fpm_env_init_main()           ||
        0 > fpm_signals_init_main()       ||
        0 > fpm_children_init_main()      ||
        0 > fpm_sockets_init_main()       ||
        0 > fpm_worker_pool_init_main()   ||
        0 > fpm_event_init_main()) {

        if (fpm_globals.test_successful) {
            exit(FPM_EXIT_OK);
        } else {
            zlog(ZLOG_ERROR, "FPM initialization failed");
            return -1;
        }
    }

    if (0 > fpm_conf_write_pid()) {
        zlog(ZLOG_ERROR, "FPM initialization failed");
        return -1;
    }

    fpm_stdio_init_final();
    zlog(ZLOG_NOTICE, "fpm is running, pid %d", (int) fpm_globals.parent_pid);

    return 0;
}
```

不难看出，**fpm\_init** 返回 -1 时，整个程序将会退出，当有错误发生但是 fpm\_globals 中标记了 test\_successful 时，会使用 exit\(0\) 退出，因为配置文件没有错误，而成功时会返回 0。

fpm 初始化的过程分 13 步，分别是:

* fpm\_php\_init\_main\(\) **注册进程清理方法**
* fpm\_stdio\_init\_main\(\) **验证 /dev/null 是否可读写**
* fpm\_conf\_init\_main\(test\_conf, force\_daemon\) **校验并加载配置文件**
* fpm\_unix\_init\_main\(\) **检查 unix 运行环境**
* fpm\_scoreboard\_init\_main\(\) **初始化“进程记分牌”**
* fpm\_pctl\_init\_main\(\) **进程管理相关初始化**
* fpm\_env\_init\_main\(\) **？**
* fpm\_signals\_init\_main\(\) **设置信号处理方式**
* fpm\_children\_init\_main\(\) **初始化子进程，注册进程清理方法**
* fpm\_sockets\_init\_main\(\) **初始化 sockets**
* fpm\_worker\_pool\_init\_main\(\) **注册 worker pool 清理方法**
* fpm\_event\_init\_main\(\) **注册 event 清理方法**

可以说相当复杂的一个过程，只要有一个函数无法返回 0，程序将直接退出。在完成这一系列的操作之后，fpm 会向配置文件中指定的 pid 文件写入 master 进程id。整个过程其实是为了启动 php-fpm 进程做初始化的工作，到这一步 php-fpm 还是无法接收请求的。

```c
if (0 > fpm_conf_write_pid()) {
  zlog(ZLOG_ERROR, "FPM initialization failed");
  return -1;
}
```
