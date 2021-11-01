# nginx 相关
## 转发
+ 负载均衡有好几种算法  轮转法，散列法， 最少连接法，最低缺失法，最快响应法，加权法。我们这里可以使用加权法来分配请求。

## 静态文件缓存

## 访问日记

## 配置
  nginx path prefix: "/usr/local/nginx"
  nginx binary file: "/usr/local/nginx/sbin/nginx"
  nginx modules path: "/usr/local/nginx/modules"
  nginx configuration prefix: "/usr/local/nginx/conf"
  nginx configuration file: "/usr/local/nginx/conf/nginx.conf"
  nginx pid file: "/usr/local/nginx/logs/nginx.pid"
  nginx error log file: "/usr/local/nginx/logs/error.log"
  nginx http access log file: "/usr/local/nginx/logs/access.log"
  nginx http client request body temporary files: "client_body_temp"
  nginx http proxy temporary files: "proxy_temp"
  nginx http fastcgi temporary files: "fastcgi_temp"
  nginx http uwsgi temporary files: "uwsgi_temp"
  nginx http scgi temporary files: "scgi_temp"

## 常用命令
+ 启动
sudo /usr/local/nginx/sbin/nginx
+ 关闭
sudo /usr/local/nginx/sbin/nginx -s stop
+ 重启
sudo /usr/local/nginx/sbin/nginx -s reload
+ 测试配置文件是否正确
sudo /usr/local/nginx/sbin/nginx -t
+ 想要编译输出debug级别日记
`./configure  --prefix=/Users/andrew_asa/Documents/code/github/lab/nginx/nginx-1.16.0 --with-openssl=./openssl-1.1.0g --with-http_ssl_module --add-module=./nginx-upstream-fair --with-debug`

正常停止   sudo kill -QUIT 主进程号
 
快速停止   sudo kill -TERM 主进程号
nginx -c nginx.conf 指定配置文件




## 参考连接
[nginx 第三方模块网站](https://www.nginx.com/resources/wiki/modules/)
[nginx 配置负载均衡](https://github.com/huangshuwei/blog/issues/10)
[nginx 负载均衡策略之第三方扩展策略](http://rookiezhou.top/nginx-%E8%B4%9F%E8%BD%BD%E5%9D%87%E8%A1%A1%E7%AD%96%E7%95%A5%E4%B9%8B%E7%AC%AC%E4%B8%89%E6%96%B9%E6%89%A9%E5%B1%95%E7%AD%96%E7%95%A5/)
[ngx_http_consistent_hash 项目地址](https://github.com/replay/ngx_http_consistent_hash)
[Nginx教程](https://www.yiibai.com/nginx)
[fair 负载均衡策略](https://github.com/gnosek/nginx-upstream-fair)
[mac编译安装Nginx](http://nullpointer.pw/mac%E7%BC%96%E8%AF%91%E5%AE%89%E8%A3%85nginx.html)
[Nginx Access Log日志统计分析常用命令](https://www.cnblogs.com/coolworld/p/6726538.html)
[https://www.kancloud.cn/digest/understandingnginx/202586](https://www.kancloud.cn/digest/understandingnginx/202586)
[Nginx 中的 upstream 与 subrequest 机制](https://blog.csdn.net/chenhanzhun/article/details/42680343)
[Emiller 的 Nginx 的模块开发指南](http://www.evanmiller.org/nginx-modules-guide.html)
[Emiller 的 Nginx 的模块开发指南 ](https://www.oschina.net/translate/nginx-modules-guide?lang=chs&p=2)
[VS CODE 轻松调试 Nginx](https://juejin.im/entry/5bcd0c4ae51d457ac227d88e)
[NGX的第三方负载均衡模块fair](https://blog.csdn.net/MeRcy_PM/article/details/50662230)
[nginx 健康检查和负载均衡机制分析](https://blog.51cto.com/feihan21/1329885)
[开发指南](https://www.docs4dev.com/docs/zh/nginx/current/reference/dev-development_guide.html)
[理解 Nginx 源码](https://www.kancloud.cn/digest/understandingnginx/202586)