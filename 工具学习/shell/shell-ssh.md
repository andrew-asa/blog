# ssh 相关
## 免密登录节点
+ 目标节点 ssh-keygen -t rsa -P ""
+ 本机节点 ssh-copy-id root@nodeip

## 设置默认登录名
+ vim 〜/.ssh/config

```
Host example                  // 名称
    HostName example.net      // ip
    User example              // 某人用户名
```
或者

```
Host *                        // 名称
    User example              // 某人用户名
```



## 参考连接
+ [https://blog.csdn.net/feinifi/article/details/78213297](https://blog.csdn.net/feinifi/article/details/78213297)
+ [https://codeday.me/bug/20170409/9747.html](https://codeday.me/bug/20170409/9747.html)